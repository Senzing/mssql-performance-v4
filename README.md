# mssql-performance

Performance tuning for running **Senzing v4** on **Microsoft SQL Server** under high-throughput load —
companion to
[performance-general-v4](https://github.com/Senzing/performance-general-v4/blob/main/README.md). Apply
everything below **after** the standard Senzing v4 schema and **before** loading; each `ALTER`/index step
is metadata-only on the empty tables (instant).

> [!IMPORTANT]
> The single biggest lever is not any switch here — it is **RAM**. At scale the bottleneck is cache-miss
> read latency, so size the buffer pool to hold as much of the working set as you can (see
> [Key learnings](#key-learnings)).

⭐ marks a key win we measured in our own v4 testing.

## At a glance — the key wins

| Lever | What it does | Measured impact |
|---|---|---|
| ⭐ [Max server memory](#-max-server-memory) | Holds the working set in the buffer pool | **#1 lever** at scale |
| ⭐ [RES_ENT.FEATURES store](#engine-side-settings-that-drive-db-load) _(v4.4+)_ | Engine feature cache — collapses reads/record | **Largest read cut** |
| ⭐ [Optimize for sequential key](#-optimize-for-sequential-key) | Fixes the last-page `PAGELATCH_EX` insert convoy | **Removes the insert ceiling** |
| ⭐ [Cluster the heaps](#-cluster-the-sequential-key-heaps) | Stops `VARCHAR(MAX)` row-forwarding on OBS_ENT / RES_ENT | **+9–13%** · latch 862K→8.5K ms |
| ⭐ [Cover RES_FEAT_EKEY](#-cover-the-res_feat_ekey-by-entity-index) | Kills key-lookups on the by-entity seek | shared-lock wait **−78%** |
| ⭐ [Read-committed snapshot](#-read-committed-snapshot-rcsi) | Eliminates lock-manager waits at ~zero cost | `LCK_M_*` → **0** |
| ⭐ [ODBC fastvalidate](#odbc-client) | Debian unixODBC vs Microsoft's prebuilt driver-manager | **~10× faster** |

---

# Test platform — the hardware behind these numbers

Every measured result on this page came off the rig described here. Read the numbers against it: the
levers that matter most are **ratios** — working set vs. buffer pool, reads per record — so a box with a
different memory-to-dataset ratio will rank them differently.

## Database server (one host, database only — no engine)

| | |
|---|---|
| Chassis | Supermicro `SYS-221H-TN24R` — 2U, 24 × U.2/U.3 NVMe bays |
| CPU | 2 × Intel **Xeon Gold 6438Y+** (Sapphire Rapids) — **64 cores / 128 threads** total, 4.0 GHz max turbo, 60 MiB L3 per socket |
| Memory | **1 TiB** — 16 × 64 GB DDR5-4800 (2R ECC), 16 of 32 slots populated, 2 NUMA nodes |
| Data volume | **22 × Micron 7450 PRO 1.92 TB** U.3 NVMe in **RAID 10** behind a GRAID SupremeRAID controller (GPU-offloaded RAID, driver 1.7.2) → a single 19 TiB (21.1 TB) block device, `ext4`, holding all database files and the log |
| Network | 10 GbE on a dedicated data-plane segment (management traffic is on a separate NIC/subnet) |
| OS | Ubuntu 24.04 LTS, kernel 6.8 |

## Application servers (two hosts, identical)

| | |
|---|---|
| Chassis | Dell **PowerEdge R650xs** |
| CPU | 2 × Intel **Xeon Gold 6326** (Ice Lake) — **32 cores / 64 threads** per host, 2.9 GHz base / 3.5 GHz turbo, 24 MiB L3 per socket |
| Memory | **512 GiB** — 8 × 64 GB DDR4-3200 (2R ECC), 8 of 16 slots populated, 2 NUMA nodes |
| Local storage | Dell BOSS SATA boot + 4–8 TB local NVMe for datasets/scratch — **not** on the database IO path |
| Network | 10 GbE, same data-plane segment as the database server |
| OS | Ubuntu 24.04 LTS, kernel 6.8 |
| Role | Runs the containerized Senzing v4 workers — parallel `add_record` consumers plus redo processors — driving the database server remotely over the data-plane network |

A third small host runs only the message broker that feeds the workers; it is not in the measurement path.

## Database configuration on that hardware

| | |
|---|---|
| Engine | SQL Server 2025, Developer edition, containerized on the database host |
| Buffer pool | `max server memory` = **384 GB**, held constant across arms. The container's budget for the whole `sqlservr` process is **480 GB** — the ~96 GB gap is thread stacks, SQLPAL overhead and dump capture, *not* buffer pool. See [Max server memory](#-max-server-memory). |
| Database files | 8 data files on the one array volume, plus the log — see [Multiple data files](#multiple-data-files) |
| Scale measured | 100M to 1B records; ≈16.9 KB/record all-in for our corpus, so a 1B-record datastore is ≈**16 TiB** |

## What this shape implies

- **The buffer pool is deliberately oversubscribed.** At 1B records ≈16 TiB of datastore sits against a
  384 GB pool — roughly **42×**. That ratio is the whole point of the test: it is what makes cache-miss
  random reads (`PAGEIOLATCH_SH`) the ceiling. A dataset small enough to be cache-resident on this box
  will **not** reproduce any of the read-side results here, which is why results below ~100M records are
  not meaningful.
- **The engine, not the database, owns most of the CPU.** 64 application cores drive 64 database cores,
  and the workload saturates the application side first. Adding database cores is not the scaling lever;
  adding application hosts and cutting reads/record is.
- **Storage is not the bottleneck.** The 22-drive RAID 10 array carried the entire 1B-record load with
  large measured IOPS headroom, which is why every ⭐ above targets *reads per record* or *buffer-pool
  residency* rather than faster disks. Note that on an array like this `iostat %util` is meaningless —
  see [Monitoring](#monitoring--whats-going-on).
- **Nor is the 10 GbE data plane.** The load is an extremely high count of very small round trips, so it
  is sensitive to round-trip *latency* and round-trip *count*, not to bandwidth.
- **Some settings below exist *because* of this core count.** The `-T8904` trace flag removes a
  `LOGFLUSHQ` spinlock regression that only appears on a high-core host at a high commit rate, and the
  `cost threshold for parallelism = 500` result was verified against these 64 database cores. On a much
  smaller box both matter less.

---

# Database settings

## UTF-8 database

```sql
CREATE DATABASE senzing COLLATE Latin1_General_100_CS_AI_SC_UTF8;
```

Required. The `_UTF8` collations require **SQL Server 2019 or later**.

## Batch-commit durability

```sql
ALTER DATABASE senzing SET DELAYED_DURABILITY = FORCED;
```

A large throughput win for bulk load.

## Simple recovery

```sql
ALTER DATABASE senzing SET RECOVERY SIMPLE;
```

For load / perf environments — avoids log-backup management and runaway log growth. Not a production DR
posture.

## Async statistics

```sql
ALTER DATABASE senzing SET AUTO_CREATE_STATISTICS ON;
ALTER DATABASE senzing SET AUTO_UPDATE_STATISTICS_ASYNC ON;
```

Don't stall worker threads on a synchronous statistics refresh.

## ⭐ Read-committed snapshot (RCSI)

```sql
ALTER DATABASE senzing SET ALLOW_SNAPSHOT_ISOLATION ON;
ALTER DATABASE senzing SET READ_COMMITTED_SNAPSHOT ON;
```

Eliminates lock-manager (`LCK_M_*`) waits at effectively zero version-store cost. Keep it on.

## Turn off Query Store

```sql
ALTER DATABASE senzing SET QUERY_STORE = OFF;
```

Pure overhead (`QDS_ASYNC_QUEUE`) for this write-heavy load.

## Leave ADR / Optimized Locking off

> [!CAUTION]
> Do **not** enable `ACCELERATED_DATABASE_RECOVERY` or `OPTIMIZED_LOCKING` for a large cache-bound load.
> ADR's persistent version store steals buffer pool from the hot feature tables (measured net-negative),
> and Optimized Locking (which requires ADR; SQL Server 2025+) only claws back part of that. RCSI already
> covers lock contention.

# Instance & server settings

## Parallelism

```sql
EXEC sp_configure 'max degree of parallelism', 0;
EXEC sp_configure 'cost threshold for parallelism', 500;
```

A high cost threshold keeps the cheap per-record OLTP queries serial without force-serializing
everything. Do **not** pin `MAXDOP 1`. Bonus: a parallel plan appearing on a per-record query is then an
automatic bad-plan alarm.

Verified on a ~1B-record fleet at 500: every engine statement runs at **average DOP 1.00**, and the only
parallel work left is SQL Server's own internal background tasks. The engine workload is fully serial,
which is the intent.

## Fill factor

Set it **per index, by insert pattern** — not server-wide.

```sql
-- Random-key indexes (inserts land mid-tree → real page splits): leave headroom.
--   e.g. LIB_FEAT_SK, keyed on FEAT_HASH
ALTER INDEX LIB_FEAT_SK ON LIB_FEAT REBUILD WITH (FILLFACTOR = 50);

-- Ascending-key clustered PKs (inserts append at the right edge → no mid-tree splits): pack them.
ALTER INDEX LIB_FEAT_PK      ON LIB_FEAT      REBUILD WITH (FILLFACTOR = 100);
ALTER INDEX RES_FEAT_STAT_PK ON RES_FEAT_STAT REBUILD WITH (FILLFACTOR = 100);
```

The old advice here was a server-wide `sp_configure 'fill factor (%)', 50` justified as *"in-page
headroom for hot-row updates."* That is right for randomly-keyed indexes and wrong for the ones keyed
on a monotonic ID, which is most of the hot set. Measured on a 16.4-billion-row load:

| index | key | measured page fullness | verdict |
|---|---|---|---|
| `RES_FEAT_STAT_PK` | `LIB_FEAT_ID`, ascending | **~89% full** | fill factor 50 is *inert* — appended pages fill anyway |
| `LIB_FEAT_PK` | `LIB_FEAT_ID`, ascending | **~74% full** | ~23% of a 3.4 TB index is slack, for updates that never happen |
| `LIB_FEAT_SK` | `FEAT_HASH`, random | ~75% full | **keep 50** — genuinely splitting |

Two things make the blanket 50 a poor default:

* **Fill factor only applies at CREATE/REBUILD.** On an append-mostly table, new right-edge pages fill
  to ~100% regardless of the setting, so it does not deliver the headroom it promises — it just leaves
  whatever slack the last rebuild created.
* **In a cache-bound load, empty page space is the expensive kind.** When the working set is many times
  the buffer pool and `PAGEIOLATCH_SH` is the #1 wait, half-full pages halve effective residency. On
  `LIB_FEAT_PK` alone the slack is ~790 GB.

> [!NOTE]
> Under **RCSI** an updated row grows by a 14-byte version pointer, so genuinely-updated tables *do*
> need some headroom — that, not "hot-row updates" in general, is the real justification. It does not
> apply to write-once tables such as `LIB_FEAT`.

## ⭐ Max server memory

```sql
EXEC sp_configure 'max server memory (MB)', <set explicitly>;
```

Set it explicitly, just under the container/host memory limit to leave non-buffer-pool + OS headroom.
Provision as much as you can — the buffer pool's ability to hold the working set is the dominant factor
at scale.

## Lightweight pooling

```sql
EXEC sp_configure 'lightweight pooling', 1;   -- restart required
```

Helps handle many concurrent workers with less scheduling overhead.

> [!WARNING]
> On **SQL Server on Linux its effect can be limited** — verify `run_value = 1` after restart and
> measure; drop it if it doesn't help. (We set it but did not measure a benefit on Linux.)

## Startup trace flag `-T8904` (many-core hosts)

On high-core boxes at high commit rate, `-T8904` disables inline log-flush, removing the `LOGFLUSHQ`
spinlock scaling regression and cutting system CPU. It is a **startup** trace flag — set it via
`mssql-conf` and restart; **not** `DBCC TRACEON` (that is per-session/runtime and lost on restart).

```bash
# SQL Server on Linux — persist as a startup trace flag, then restart the instance:
/opt/mssql/bin/mssql-conf traceflag 8904 on
#   in a container:  docker exec <mssql-container> /opt/mssql/bin/mssql-conf traceflag 8904 on
#   then restart SQL Server (e.g. docker restart <mssql-container>) for it to take effect.
# On Windows: add -T8904 as a startup parameter in SQL Server Configuration Manager.
```

> [!NOTE]
> `-T8134` is diagnostic-only: it surfaces `SPINLOCK_EXT` as a measurable wait (set the same way, for a
> measurement window; clear it after).

# Indexing & storage

## ⭐ Optimize for sequential key

**Critical for high-concurrency load.** The last-page insert latch (`PAGELATCH_EX`) on the ascending-key
B-trees is the classic insert-throughput ceiling — OFSK (**SQL Server 2019+**) is its purpose-built fix,
converting the raw convoy into managed `BTREE_INSERT_FLOW_CONTROL` and moving the wall off the DB. The v4
schema already sets OFSK on the main hot tables; turn it on for the remaining ascending-key indexes it
leaves off.

```sql
ALTER INDEX RES_FEAT_EKEY_PK ON RES_FEAT_EKEY SET (OPTIMIZE_FOR_SEQUENTIAL_KEY = ON);
ALTER INDEX RES_REL_EKEY_PK  ON RES_REL_EKEY  SET (OPTIMIZE_FOR_SEQUENTIAL_KEY = ON);
ALTER INDEX RES_ENT_OKEY_PK  ON RES_ENT_OKEY  SET (OPTIMIZE_FOR_SEQUENTIAL_KEY = ON);
ALTER INDEX RES_ENT_OKEY_SK  ON RES_ENT_OKEY  SET (OPTIMIZE_FOR_SEQUENTIAL_KEY = ON);
ALTER INDEX DSRC_RECORD_PK   ON DSRC_RECORD   SET (OPTIMIZE_FOR_SEQUENTIAL_KEY = ON);
```

## ⭐ Cluster the sequential-key heaps

The schema ships `OBS_ENT` and `RES_ENT` as heaps; the `VARCHAR(MAX)` feature-blob update forwards heap
rows and makes them the **#1 `PAGELATCH_EX` source**. Clustering fixes it (**+9–13%**). Leave
`DSRC_RECORD` a heap — its PK is a non-sequential varchar key, so clustering gains nothing and scatters
page splits.

```sql
ALTER TABLE OBS_ENT DROP CONSTRAINT OBS_ENT_PK;
ALTER TABLE OBS_ENT ADD CONSTRAINT OBS_ENT_PK PRIMARY KEY CLUSTERED (OBS_ENT_ID)
  WITH (OPTIMIZE_FOR_SEQUENTIAL_KEY = ON);
ALTER TABLE RES_ENT DROP CONSTRAINT RES_ENT_PK;
ALTER TABLE RES_ENT ADD CONSTRAINT RES_ENT_PK PRIMARY KEY CLUSTERED (RES_ENT_ID)
  WITH (OPTIMIZE_FOR_SEQUENTIAL_KEY = ON);
```

## ⭐ Cover the RES_FEAT_EKEY by-entity index

Adding the immutable `FTYPE_ID` stops the by-entity seek doing key lookups back to the clustered PK
(shared-lock wait **−78%**), with no write amplification.

```sql
CREATE NONCLUSTERED INDEX RES_FEAT_EKEY_SK ON RES_FEAT_EKEY (RES_ENT_ID, FTYPE_ID)
  WITH (DROP_EXISTING = ON);
```

<details>
<summary><b>High feature-density data (~100+ features/record) only</b> — fully cover the candidate feature-load</summary>

> [!WARNING]
> A DBA knob, **not** a default — it enlarges the index and raises fragmentation (needs periodic
> `ONLINE REORGANIZE`). Unnecessary below high density.

```sql
CREATE NONCLUSTERED INDEX RES_FEAT_EKEY_SK ON RES_FEAT_EKEY (RES_ENT_ID, FTYPE_ID)
  INCLUDE (SUPPRESSED, OBS_ENT_CNT) WITH (DROP_EXISTING = ON, FILLFACTOR = 50);
```
</details>

## ⭐ Drop IX_EVAL_QUEUE — redundant with the primary key

`SYS_EVAL_QUEUE` has the PK on `MSG_ID` plus `IX_EVAL_QUEUE (ENT_SRC_KEY, DSRC_CODE)`; the latter is
**redundant**. `MSG_ID = fnv1a_hash(encrypted ENT_SRC_KEY) >> 1` is deterministic in `ENT_SRC_KEY`,
and the enqueue's `DSRC_CODE` is the constant `'__REPAIR__'`, so both uniques are **1:1 with the
entity** — one entity targets exactly one row.

On MSSQL the enqueue does a plain `INSERT` and treats the resulting unique-constraint violation as
**success** (a duplicate repair intent is a no-op) — the **PK** violation serves that dedup
identically once `IX_EVAL_QUEUE` is gone. No query seeks on `(ENT_SRC_KEY, DSRC_CODE)`; every read is
by `MSG_ID` (dequeue/count/min-max), and the redo-dequeue lock waits were **100% on the PK, 0 on
`IX_EVAL_QUEUE`**. Dropping it removes one nonclustered index maintained on every redo enqueue on the
hottest table in the system.

```sql
DROP INDEX IX_EVAL_QUEUE ON SYS_EVAL_QUEUE;
```

## Compression

PAGE on the hot tables — **including `LIB_FEAT`**. Rebuild the table **and** all its indexes. Little added
CPU, and it defers the read-IO crossover as the DB grows.

> [!IMPORTANT]
> This previously said *"ROW on `LIB_FEAT` (its `FEAT_HASH` is incompressible, so ROW ≈ PAGE at lower
> decompress cost)."* The premise is correct — `FEAT_HASH` really is incompressible — but the conclusion
> does not follow, because `FEAT_HASH` is only 40 of ~164 bytes. Measured on a 200k-row stratified sample
> at production row density (139.0 avg variable bytes/row vs the corpus's 139.7):
>
> | compression | bytes/row | vs NONE |
> |---|---|---|
> | NONE | 335.2 | — |
> | ROW | 325.4 | **2.9%** |
> | PAGE | 298.8 | **10.9%** |
>
> ROW recovers only ~10 bytes/row because it squeezes *fixed-width* storage, and just 13 of ~164 bytes are
> fixed. It cannot touch the ~139 bytes of `FEAT_HASH` + `FEAT_DESC` + `FELEM_VALUES` that are 85% of the
> row. PAGE's prefix/dictionary compression does reach `FEAT_DESC` and `FELEM_VALUES`. **ROW is not ≈ PAGE
> here; it is 8.2% worse**, which on `LIB_FEAT`'s two indexes (3.4 TB + 1.6 TB) is ~400 GB.
>
> This matters more than the percentage suggests: `LIB_FEAT` cache-miss random reads are the documented
> wall for this workload (`PAGEIOLATCH_SH` #1 at ~67% of wait time; `LIB_FEAT` the top physical-read
> source at 195.7M reads, driven by ~1.4B `FEAT_HASH` seeks → key lookups per 3 hours). Leaving the
> table that *is* the bottleneck on the weaker compression is the wrong trade — and PAGE was already
> measured on the other hot tables here as throughput-positive with little added CPU.

<details>
<summary>Show the compression rebuild block</summary>

```sql
-- PAGE on the integer-key / JSON tables:
ALTER TABLE OBS_ENT       REBUILD WITH (DATA_COMPRESSION = PAGE);
ALTER INDEX ALL ON OBS_ENT REBUILD WITH (DATA_COMPRESSION = PAGE);
-- ...repeat for RES_ENT, RES_FEAT_EKEY, RES_FEAT_STAT, RES_RELATE, RES_ENT_OKEY, RES_REL_EKEY, DSRC_RECORD
-- PAGE on LIB_FEAT too (measured 10.9% vs 2.9% for ROW — see the note above):
ALTER TABLE LIB_FEAT       REBUILD WITH (DATA_COMPRESSION = PAGE);
ALTER INDEX ALL ON LIB_FEAT REBUILD WITH (DATA_COMPRESSION = PAGE);
```
</details>

## Multiple data files

Spreads allocation / free-space (`SPACEMGR_FREESPACE_CACHE`, PFS/GAM) contention across files under many
concurrent inserters. Pre-size for large loads (on Linux ext4/xfs this uses `fallocate` — seconds, not a
zero-fill; the `instant_file_initialization_enabled` DMV false-negatives `N` on Linux, ignore it).

> [!WARNING]
> Applies wherever **you control the database files** — self-managed SQL Server (bare metal, VM, or
> container) and Azure SQL Managed Instance. It does **not** apply to Azure SQL Database **Hyperscale**,
> where page servers manage storage and there are no user data files.

```sql
ALTER DATABASE senzing MODIFY FILE (NAME = senzing, SIZE = <presize>, FILEGROWTH = 32768MB);
ALTER DATABASE senzing ADD FILE (NAME = senzing_2, FILENAME = '.../senzing_2.ndf', SIZE = <presize>, FILEGROWTH = 32768MB)
  TO FILEGROUP [PRIMARY];
-- ...repeat to ~8 equally-sized files...
ALTER DATABASE senzing MODIFY FILEGROUP [PRIMARY] AUTOGROW_ALL_FILES;
```

## Wide feature-value column

Ensure `LIB_FEAT.FELEM_VALUES` is `VARCHAR(MAX)` — long or multibyte (e.g. CJK) feature values otherwise
fail to insert.

# ODBC client

Senzing application containers reach SQL Server through ODBC:

- ⭐ Use **ODBC Driver 18** (`msodbcsql18`) with **Debian's** `unixodbc` (built `--enable-fastvalidate`;
  Microsoft's prebuilt driver-manager is not → **~10× slower**).
- ODBC Driver 18 defaults to `Encrypt=yes`; a lab without TLS certs needs `Encrypt=no` (or
  `TrustServerCertificate=yes`) in the connection string.

> [!NOTE]
> **`AutoTranslate`: leave at the default.** With the UTF-8 collation, UTF-8 text (including CJK and
> supplementary-plane / emoji characters) round-trips byte-exact whether `AutoTranslate` is on or off
> (verified). The old `AutoTranslate = No` workaround was for non-UTF-8 (pre-2019) collations and is not
> required here.

# Engine-side settings that drive DB load

These are Senzing engine settings, not SQL Server knobs, but they dominate the load profile:

- ⭐ **Enable the `RES_ENT.FEATURES` feature store** (**Senzing v4.4+**) — the single largest reduction
  in physical reads/record.
- **On high-name-density data, set the generic NAME behavior to `"sendToRedo":"No"`** — otherwise generic
  name keys generate a self-amplifying redo pile that starves forward progress.

# Monitoring — what's going on?

```sql
-- Live statements
SELECT r.session_id, r.wait_type, t.text
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t;

-- Waits (take a snapshot -> delta, not the cumulative total)
SELECT wait_type, wait_time_ms FROM sys.dm_os_wait_stats ORDER BY wait_time_ms DESC;

-- Row counts (do NOT COUNT(*) the large tables under load — it scans and stalls)
SELECT OBJECT_NAME(object_id) AS tbl, SUM(row_count) AS rows
FROM sys.dm_db_partition_stats WHERE index_id IN (0, 1) GROUP BY object_id;
```

- **The signal at scale:** `PAGEIOLATCH_SH` dominating the waits means cache-miss random reads (working
  set > buffer pool). The answer is fewer reads/record and more RAM — not a faster log or more CPU.
- Read spinlock **backoffs** (not raw spins) from `sys.dm_os_spinlock_stats`.
- `sys.dm_exec_query_stats` is eviction-lossy; truncating a table erases its statistics.
- On GRAID / high-end NVMe arrays, `iostat %util` is meaningless (it saturates at one outstanding IO) —
  use `r_await` and `sys.dm_io_virtual_file_stats`.

# Key learnings

- **At scale the bottleneck is read latency, not CPU / log / network.** Once the working set outgrows the
  buffer pool, `PAGEIOLATCH_SH` (8 KB random reads) dominates. The two levers that move it: reduce
  reads/record (the feature store), and give the buffer pool enough RAM.
- **Per-record throughput drifts down as the DB grows, then flattens.** It tracks the working-set-vs-pool
  ratio, so it is steep while the working set is crossing the pool size and flattens once the DB is much
  larger than the pool — it does not keep accelerating. Provision RAM generously, and measure knob
  comparisons A/B/A to subtract drift.
- **RCSI is the lock-contention fix; `-T8904` removes a many-core scaling regression** but does not raise
  the ceiling — the ceiling is the read-latency wall above.

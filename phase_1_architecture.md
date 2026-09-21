# Phase 1: Snowflake Architecture — Deep Dive

> This is the single most important phase. Everything else in Snowflake (performance tuning, cost control, pipeline design) is really just applying this architecture correctly. Take your time here.

---

## 1.0 Prerequisites Recap

Before diving in, make sure you have:
- A Snowflake trial account (signup at Snowflake's website — 30 days free, ~$400 credits)
- Access to **Snowsight** (the web UI) — this is where you'll run all the code below
- Optionally, SnowSQL CLI installed for command-line practice

Every code block below can be pasted directly into a Snowsight worksheet and run.

---

## 1.1 Multi-Cluster Shared Data Architecture

### The Core Idea

Traditional databases (Oracle, MySQL, on-prem SQL Server) bundle **storage** and **compute** together on the same machine. If you need more query power, you buy a bigger machine — and that bigger machine also has to hold all your data, even data you rarely query.

Snowflake breaks this bundle into **three independent layers**:

```
┌─────────────────────────────────────────────────┐
│              CLOUD SERVICES LAYER                │
│  (Auth, Metadata, Query Parsing/Optimization,    │
│   Security, Infrastructure Management)           │
└─────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
┌───────▼──────┐ ┌───────▼──────┐ ┌──────▼───────┐
│  WAREHOUSE A │ │  WAREHOUSE B │ │  WAREHOUSE C │   ← QUERY PROCESSING LAYER
│   (compute)  │ │   (compute)  │ │   (compute)  │     (Virtual Warehouses)
└───────┬──────┘ └───────┬──────┘ └──────┬───────┘
        │               │               │
        └───────────────┼───────────────┘
                        │
              ┌──────────▼──────────┐
              │   DATABASE STORAGE    │              ← STORAGE LAYER
              │  (S3 / Azure Blob /   │                (Shared, single copy)
              │   Google Cloud Storage)│
              └───────────────────────┘
```

### Layer 1: Database Storage Layer
- All your table data physically lives in cloud object storage (Amazon S3, Azure Blob, or Google Cloud Storage — whichever cloud you picked when creating your account).
- Data is automatically compressed, organized into **micro-partitions**, and stored in a **columnar format**.
- You never manage this storage directly — no disks to provision, no partitioning to design manually.
- **This layer is shared** — every virtual warehouse in your account reads from the *same* single copy of data. No data duplication across compute clusters.

### Layer 2: Query Processing Layer (Virtual Warehouses)
- This is the actual "compute" — clusters of CPU/memory that execute your SQL queries.
- Called **Virtual Warehouses** in Snowflake.
- You can spin up multiple independent warehouses, each isolated from the others, all pointing at the same underlying data.
- Because compute is separate from storage, you can scale compute up/down or add more warehouses **without moving or duplicating data**.

### Layer 3: Cloud Services Layer
- The "brain" that ties everything together: authentication, access control, query parsing, query optimization, transaction management, metadata management.
- This layer is fully managed by Snowflake — you don't provision anything for it.
- Example: when you run a query, this layer decides the execution plan, checks your permissions, and only *then* hands off the actual data-crunching work to a Virtual Warehouse.

### Why This Separation Matters (the actual insight, not just definitions)

| Traditional DB | Snowflake |
|---|---|
| Scale compute = scale storage together | Scale compute and storage independently |
| One workload competing for one set of resources | Multiple warehouses run the same data with zero contention |
| Idle compute still costs money (always-on server) | Compute can auto-suspend to zero cost when idle |
| Adding capacity = downtime/migration | Adding a warehouse = instant, no data movement |

**Real-world example:** Your BI team runs heavy Tableau dashboards all day, and your ETL job runs a 2-hour load every night. In a traditional DB, these compete for the same CPU/disk. In Snowflake, you give BI its own warehouse (`BI_WH`) and ETL its own warehouse (`ETL_WH`) — both querying the *same* tables, with zero resource contention, and each can be sized/scheduled independently.

```sql
-- Two separate warehouses for two separate workloads, same data underneath
CREATE WAREHOUSE bi_wh  WITH WAREHOUSE_SIZE = 'SMALL'  AUTO_SUSPEND = 300 AUTO_RESUME = TRUE;
CREATE WAREHOUSE etl_wh WITH WAREHOUSE_SIZE = 'LARGE'  AUTO_SUSPEND = 60  AUTO_RESUME = TRUE;

-- Both can query the same table with no contention
USE WAREHOUSE bi_wh;
SELECT COUNT(*) FROM sales.public.orders;

USE WAREHOUSE etl_wh;
SELECT COUNT(*) FROM sales.public.orders;
```

**[Interview Hotspot]** If asked "Why is Snowflake's architecture different from a traditional data warehouse?" — always lead with this storage/compute separation, then mention the three layers by name.

---

## 1.2 Virtual Warehouses (Compute Layer, In Depth)

### What Exactly Is a Virtual Warehouse?

A Virtual Warehouse is a named allocation of compute resources (CPU, memory, temp storage) that you use to run SQL queries and DML operations (loading, transforming data). Nothing runs in Snowflake without a running warehouse attached — even a `SELECT 1` needs a warehouse (unless served entirely from result cache, more on that later).

```sql
-- Create your first warehouse
CREATE WAREHOUSE my_wh
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60          -- suspend after 60 seconds idle
  AUTO_RESUME = TRUE         -- auto start when a query is submitted
  INITIALLY_SUSPENDED = TRUE; -- don't start it immediately, save credits
```

### Warehouse Sizes

Sizes double in compute power (and credit consumption) as you go up:

| Size | Servers (roughly) | Credits/hour (approx) |
|---|---|---|
| X-Small | 1 | 1 |
| Small | 2 | 2 |
| Medium | 4 | 4 |
| Large | 8 | 8 |
| X-Large | 16 | 16 |
| 2X-Large | 32 | 32 |
| 3X-Large | 64 | 64 |
| 4X-Large | 128 | 128 |
| 5X-Large / 6X-Large | 256 / 512 | 256 / 512 |

> Important nuance: bigger warehouse size = **more parallelism per query**, not necessarily "handles more concurrent users." For concurrency, you want multi-cluster warehouses (below).

```sql
-- Resize a warehouse anytime (existing running queries finish at old size,
-- new queries use new size — no downtime, no restart needed)
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'MEDIUM';
```

### Single-Cluster vs Multi-Cluster Warehouses

- **Single-cluster** (default): one cluster of compute. Good for steady, predictable workloads.
- **Multi-cluster**: Snowflake automatically spins up *additional clusters of the same size* when concurrent query load increases (e.g., 50 analysts all querying at 9am), then spins them back down when load drops.

```sql
CREATE WAREHOUSE bi_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  WAREHOUSE_TYPE = 'STANDARD'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 4        -- scales OUT (more clusters), not UP (bigger size)
  SCALING_POLICY = 'STANDARD'; -- vs 'ECONOMY'
```

**Key distinction [Interview Hotspot]:**
- **Scaling UP** (bigger `WAREHOUSE_SIZE`) → speeds up a *single* complex/heavy query.
- **Scaling OUT** (multi-cluster, more clusters) → handles more *concurrent* queries/users, doesn't make one query faster.

### Scaling Policies (Multi-cluster only)
- **Standard**: favors starting additional clusters quickly to keep query queuing to a minimum (prioritizes performance over cost).
- **Economy**: waits longer, tries to fully utilize existing clusters before starting a new one (prioritizes cost over performance).

```sql
ALTER WAREHOUSE bi_wh SET SCALING_POLICY = 'ECONOMY';
```

### Auto-Suspend & Auto-Resume — Your #1 Cost Control Lever

- `AUTO_SUSPEND = <seconds>`: warehouse automatically shuts down (stops billing) after N seconds of no query activity.
- `AUTO_RESUME = TRUE`: next incoming query automatically wakes it up again (takes a few seconds, called a "cold start").

```sql
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
```

**Why this matters:** You are billed **per-second of compute usage**, per warehouse, only while it's running (with a 60-second minimum billing per resume). A warehouse that never suspends but is idle 90% of the day is pure wasted cost. This is the single biggest and easiest cost optimization lever for beginners.

**[Interview Hotspot]** "How do you control Snowflake costs?" → Auto-suspend/auto-resume is always the first answer, then warehouse right-sizing, then resource monitors (Phase 5/6 topic).

### Caching — Three Types (Very Commonly Confused, Learn Precisely)

1. **Result Cache** (Cloud Services layer, not warehouse-dependent)
   - If you run the *exact same query* (byte-for-byte SQL text) and underlying data hasn't changed, Snowflake returns cached results **instantly, using zero compute credits** — even if the warehouse is suspended!
   - Persists for 24 hours.
   ```sql
   -- Run this twice in a row — second run returns from Result Cache, near-instant, 0 credits
   SELECT COUNT(*) FROM sales.public.orders;
   ```

2. **Local Disk Cache (Warehouse Cache)**
   - Each running warehouse caches recently-scanned micro-partitions on its local SSD.
   - Speeds up repeated/similar queries hitting the same data while the warehouse is running.
   - **Lost when the warehouse suspends** — this is a hidden cost of aggressive auto-suspend (frequent cold cache = slower repeat queries) — a tradeoff to know about.

3. **Remote Disk Cache (Storage layer itself)**
   - The actual persisted data storage (S3/Blob/GCS) — always durable, this is not really a "cache" in the performance sense, it's just where data lives permanently.

```sql
-- To FORCE bypassing result cache for testing/benchmarking:
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
```

### Practice Exercise
```sql
-- 1. Create a warehouse
CREATE WAREHOUSE practice_wh WAREHOUSE_SIZE='XSMALL' AUTO_SUSPEND=60 AUTO_RESUME=TRUE;
USE WAREHOUSE practice_wh;

-- 2. Run a query, check "Query Profile" in Snowsight for time & credits used
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS LIMIT 1000;

-- 3. Resize up and rerun a heavier query, compare execution time
ALTER WAREHOUSE practice_wh SET WAREHOUSE_SIZE = 'SMALL';
SELECT O_ORDERSTATUS, COUNT(*) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS GROUP BY 1;

-- 4. Check warehouse credit usage history
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE WAREHOUSE_NAME = 'PRACTICE_WH'
ORDER BY START_TIME DESC;
```
> Note: `SNOWFLAKE_SAMPLE_DATA` is a free sample database automatically shared with every Snowflake account — use it throughout this roadmap for practice.

---

## 1.3 Micro-Partitions & Storage (In Depth)

### What Is a Micro-Partition?

When you load data into a Snowflake table, Snowflake automatically (you never do this manually) splits it into **micro-partitions**:
- Each micro-partition holds between **50 MB and 500 MB of uncompressed data** (compressed further on disk, so actual file size is usually smaller).
- Data within a micro-partition is stored in a **columnar** format — meaning each column's values are stored together, not row-by-row. This is why analytical `SELECT col FROM table` type queries are fast — Snowflake only reads the columns you asked for, not entire rows.
- Micro-partitions are **immutable** — once written, they are never edited in place. An `UPDATE` or `DELETE` creates *new* micro-partitions and marks old ones for removal (this connects directly to Time Travel later — old micro-partitions aren't deleted immediately, they're kept for the Time Travel retention window).

### Automatic Partitioning — No DBA Work Required

Traditional data warehouses (Teradata, Redshift, Hive) require you to manually design partition keys (e.g., partition by `order_date`) — get it wrong, and query performance suffers badly, and repartitioning is painful.

Snowflake automatically partitions data as it's loaded, **based on natural ingestion order** (roughly, the order rows arrived in). You don't declare a partition key at table creation the way you would in Redshift.

```sql
-- Notice: no PARTITION BY clause anywhere. This is standard in Snowflake.
CREATE TABLE orders (
    order_id     NUMBER,
    customer_id  NUMBER,
    order_date   DATE,
    amount       NUMBER(10,2)
);
```

### Metadata Per Micro-Partition — The Secret to Speed

For every micro-partition, Snowflake's Cloud Services layer automatically stores metadata, including:
- MIN and MAX value for every column in that micro-partition
- Number of distinct values
- Number of NULLs
- The total number of rows

This metadata is tiny compared to the actual data, so it's cheap to scan.

### Partition Pruning — How This Metadata Makes Queries Fast **[Interview Hotspot]**

When you run a query with a `WHERE` filter, Snowflake first checks the *metadata* of each micro-partition — **without reading the actual data** — to see if that partition could possibly contain matching rows. If a partition's MIN/MAX range doesn't overlap your filter, Snowflake skips it entirely.

**Example:**
```sql
-- Suppose ORDERS has 1000 micro-partitions, and data was loaded in date order
-- (so partitions are naturally sorted by order_date over time)

SELECT * FROM orders WHERE order_date = '2024-06-15';
```
If `order_date` values are naturally clustered by load-time order (which they usually are, since new orders arrive chronologically), Snowflake's metadata might show that only 3 out of 1000 micro-partitions have a MIN/MAX range covering `2024-06-15`. Snowflake reads **only those 3 partitions**, ignoring the other 997 entirely. This is "pruning."

**Why this matters:** the query cost/speed is proportional to partitions *scanned*, not table size. A well-pruned query on a 10 TB table can be as fast as a query on a 10 GB table, if only the relevant partitions are touched.

You can literally see this in action:

```sql
-- Run a query, then check the Query Profile / EXPLAIN output
EXPLAIN SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
WHERE O_ORDERDATE = '1998-01-01';
```
In Snowsight's Query Profile view (after running, not just EXPLAIN), look for the **"Partitions scanned" vs "Partitions total"** stat on the Table Scan step — this is the direct, visible proof of pruning working (or not working).

### When Pruning Breaks Down (why clustering exists — preview of Phase 5)

If the column you filter on is **not correlated with load order** (e.g., you filter on `customer_id` but data was loaded in `order_date` order), then customer IDs are scattered randomly across *all* partitions — MIN/MAX ranges for `customer_id` overlap in nearly every partition, so pruning doesn't help, and Snowflake has to scan almost everything.

This is exactly the problem that **Clustering Keys** (Phase 5) solve — you'll learn that later, but now you understand *why* it's needed: to re-sort data physically so pruning works for the columns you actually filter on.

### Checking Partition/Clustering Health

```sql
-- See how many micro-partitions a table has and clustering depth
SELECT SYSTEM$CLUSTERING_INFORMATION('SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS', '(O_ORDERDATE)');
```

### Storage Cost Note
- You're billed for the *compressed* size of data stored (Snowflake typically achieves 3-5x compression depending on data), plus any historical versions retained for Time Travel/Fail-safe (Phase 6 topic).
- Storage cost is independent of and usually much smaller than compute cost for most workloads — but large historical retention windows (Time Travel set to 90 days on huge, frequently-changing tables) can add up. Keep this in mind for later.

### Practice Exercise
```sql
-- 1. Look at a large sample table's structure
DESCRIBE TABLE SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM;

-- 2. Check its clustering info (this table has ~600M rows, good for a real example)
SELECT SYSTEM$CLUSTERING_INFORMATION('SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM', '(L_SHIPDATE)');

-- 3. Run a filtered query and open Query Profile afterward in Snowsight UI
--    Look at "Table Scan" node -> check partitions scanned vs total
SELECT COUNT(*) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM
WHERE L_SHIPDATE = '1998-01-01';
```

---

## Summary: What You Should Be Able to Explain After This Phase

Test yourself — can you explain each of these out loud, without notes?

1. Draw/describe the 3-layer architecture and what each layer does.
2. Why separating storage and compute is a big deal for cost and scalability (give the BI vs ETL warehouse example).
3. Difference between scaling **up** (bigger warehouse size) vs scaling **out** (multi-cluster) — and when you'd use each.
4. What auto-suspend/auto-resume do and why they're the easiest cost lever.
5. The three types of caching (result cache, local disk/warehouse cache, remote/storage) and what triggers each.
6. What a micro-partition is, and that it's immutable, columnar, and automatically created — no manual partitioning.
7. What metadata Snowflake stores per micro-partition (min/max, nulls, distinct count) and how that enables **pruning**.
8. Why pruning can fail (data not correlated with filter column) — and that this is *why* clustering keys exist (teaser for Phase 5).

---

## Next Step
Once you're comfortable with everything above (ideally by re-running every SQL snippet yourself and checking Query Profile outputs), move to **Phase 2: Core SQL & Data Objects** from `roadmap.md`.


---


# My understandings / Learnings :

1) s3 can store any type od data , be it
   structured : csv , parquet / orc (columnar storage), json with fixed schema
   semi-structured : json with no fixed schema , xml , logs in json format, Avro files
   unstructured :  img, videos, jpg, .mp4 , binary dumps

2) s3 does not enforce any checks on incoming data, dump as data comes, thats why its a **data Lake**

3) 

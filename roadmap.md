# Snowflake Mastery Roadmap for Data Engineering

> Goal: Go from basic SQL knowledge to deep, interview-ready and production-ready mastery of Snowflake as a Data Engineer.
> Approach: Learn concept → practice hands-on in a free Snowflake trial account → note down "why it matters" for real pipelines.

---

## How to Use This Roadmap
- Work top to bottom — each phase builds on the previous one.
- For every topic, don't just read docs — **run it** in a Snowflake trial account (30-day free trial, no card needed).
- Keep a scratchpad of SQL snippets/commands you tried — this becomes your interview prep sheet.
- Anything marked **[Interview Hotspot]** is commonly asked in Snowflake data engineering interviews.

---

## Phase 0: Prerequisites & Setup
- [ ] Create a Snowflake free trial account
- [ ] Understand cloud basics: what is a cloud data warehouse vs traditional DB (Oracle/SQL Server/MySQL)
- [ ] Install SnowSQL CLI and/or Snowsight (web UI)
- [ ] Learn Snowflake's editions: Standard, Enterprise, Business Critical, VPS (differences in features)
- [ ] Understand supported cloud platforms: AWS, Azure, GCP (Snowflake runs on top of these)

---

## Phase 1: Snowflake Architecture (Foundation — Very Important)

### 1.1 Multi-Cluster Shared Data Architecture **[Interview Hotspot]**
- [ ] Three layers of Snowflake architecture:
  - **Database Storage layer** (data stored in compressed, columnar format in cloud storage — S3/Blob/GCS)
  - **Query Processing layer** (Virtual Warehouses — the compute)
  - **Cloud Services layer** (authentication, metadata, query optimization, infra management)
- [ ] Why this decoupling of storage & compute matters (scale independently, pay separately)
- [ ] How this differs from traditional MPP databases (Teradata, Redshift's older architecture)

### 1.2 Virtual Warehouses (Compute)
- [ ] What is a Virtual Warehouse — a cluster of compute resources
- [ ] Warehouse sizes: X-Small to 6X-Large (each size doubles credits/hour)
- [ ] Multi-cluster warehouses (auto-scale for concurrency, not just size)
- [ ] Warehouse scaling policies: **Standard** vs **Economy**
- [ ] Auto-suspend & auto-resume — cost control basics **[Interview Hotspot]**
- [ ] Warehouse caching (result cache, local disk cache, remote disk)
- [ ] `CREATE WAREHOUSE`, `ALTER WAREHOUSE`, resizing warehouses live
- [ ] When to use separate warehouses for ETL vs BI/reporting (workload isolation)

### 1.3 Micro-partitions & Storage
- [ ] What are micro-partitions (50-500MB compressed, immutable, columnar storage units)
- [ ] Automatic partitioning — no manual partitioning needed (unlike Hive/traditional DWs)
- [ ] Metadata Snowflake stores per micro-partition (min/max values, count of distinct values) — enables **pruning**
- [ ] Partition pruning — how Snowflake skips irrelevant micro-partitions during query execution **[Interview Hotspot]**

---

## Phase 2: Core SQL & Data Objects in Snowflake

### 2.1 Databases, Schemas, Tables
- [ ] Object hierarchy: Account → Database → Schema → Table/View
- [ ] Table types **[Interview Hotspot]**:
  - **Permanent** tables (default, full Time Travel support)
  - **Temporary** tables (session-scoped)
  - **Transient** tables (no fail-safe, cheaper storage)
  - **External** tables (query data directly from cloud storage without loading)
- [ ] `CREATE TABLE ... LIKE`, `CREATE TABLE ... CLONE`
- [ ] Views vs Materialized Views vs Secure Views
  - When materialized views auto-refresh, cost implications

### 2.2 Semi-Structured Data Handling **[Interview Hotspot]**
- [ ] `VARIANT`, `OBJECT`, `ARRAY` data types
- [ ] Loading & querying JSON, Parquet, Avro, XML natively
- [ ] `FLATTEN()` function to unnest arrays/objects
- [ ] Dot notation and bracket notation to access nested JSON fields (`data:name.first::string`)
- [ ] `PARSE_JSON()`, `TO_VARIANT()`, `OBJECT_CONSTRUCT()`

### 2.3 Advanced SQL in Snowflake
- [ ] Window functions (RANK, DENSE_RANK, LAG/LEAD, QUALIFY clause — Snowflake-specific!)
- [ ] `QUALIFY` clause (Snowflake's shortcut vs subquery+ROW_NUMBER) **[Interview Hotspot]**
- [ ] CTEs, recursive CTEs
- [ ] `MERGE` statement (upserts) **[Interview Hotspot]**
- [ ] `PIVOT` / `UNPIVOT`
- [ ] Sampling: `SAMPLE`/`TABLESAMPLE`
- [ ] Sequences and `AUTOINCREMENT`

---

## Phase 3: Data Loading & Unloading (Core DE Skill)

### 3.1 Stages **[Interview Hotspot]**
- [ ] Internal stages: User stage, Table stage, Named internal stage
- [ ] External stages: pointing to S3/Azure Blob/GCS
- [ ] `PUT` command (upload local files to internal stage)
- [ ] `LIST @stage_name`, `REMOVE @stage_name`

### 3.2 File Formats
- [ ] `CREATE FILE FORMAT` — CSV, JSON, Parquet, Avro, ORC options
- [ ] Compression types supported (gzip, bzip2, etc.)

### 3.3 Bulk Loading
- [ ] `COPY INTO` command — the primary bulk load mechanism **[Interview Hotspot]**
- [ ] Load history & `VALIDATION_MODE`
- [ ] Error handling options: `ON_ERROR = CONTINUE/SKIP_FILE/ABORT_STATEMENT`
- [ ] File sizing best practices (100-250MB compressed per file recommendation)

### 3.4 Continuous / Streaming Loading **[Interview Hotspot]**
- [ ] **Snowpipe** — automated, continuous loading via event notifications (S3 events, SNS/SQS, Event Grid)
- [ ] Snowpipe vs `COPY INTO` — when to use which (near-real-time vs batch)
- [ ] Snowpipe Streaming API (row-level ingestion, lower latency)
- [ ] Cost model for Snowpipe (serverless compute billed per credit-second)

### 3.5 Unloading Data
- [ ] `COPY INTO <location>` — exporting query results/tables to stage
- [ ] `GET` command to download files from stage

---

## Phase 4: Data Transformation Pipelines (ELT Core)

### 4.1 Streams & Tasks — CDC and Automation **[Interview Hotspot — very commonly asked]**
- [ ] **Streams**: Change Data Capture (CDC) mechanism — track INSERT/UPDATE/DELETE on a table
  - Standard streams vs Append-only streams vs Insert-only streams
  - `METADATA$ACTION`, `METADATA$ISUPDATE`, `METADATA$ROW_ID`
  - Stream offset and consumption (staleness — streams become stale after data retention period)
- [ ] **Tasks**: Scheduled or triggered SQL execution
  - `CREATE TASK` with `SCHEDULE` (cron or interval)
  - Task trees / DAGs (chaining tasks, predecessor tasks)
  - `SYSTEM$STREAM_HAS_DATA()` — conditional task execution based on stream data
  - Task history, error handling, `SUSPEND_TASK_AFTER_NUM_FAILURES`
  - Serverless tasks vs tasks on a dedicated warehouse

### 4.2 Dynamic Tables (Newer, Important) **[Interview Hotspot]**
- [ ] What are Dynamic Tables — declarative, incrementally refreshed tables (Snowflake's answer to dbt-style materialization)
- [ ] `TARGET_LAG` concept
- [ ] When to use Dynamic Tables vs Streams+Tasks vs plain views
- [ ] Refresh modes: incremental vs full refresh, and when Snowflake auto-falls-back to full refresh

### 4.3 Stored Procedures & Scripting
- [ ] Snowflake Scripting (SQL procedural language — `BEGIN...END`, `DECLARE`, `LOOP`, `IF`)
- [ ] Stored Procedures in JavaScript, Python, Java, Scala
- [ ] User-Defined Functions (UDFs) — SQL UDFs vs UDTFs (table functions) vs external functions
- [ ] Snowpark (Python/Java/Scala DataFrame API) for pipeline logic — increasingly asked in DE interviews

---

## Phase 5: Performance & Optimization (What separates juniors from seniors)

### 5.1 Query Performance
- [ ] Reading `EXPLAIN` plans and Query Profile in Snowsight
- [ ] Identifying: full table scans, exploding joins, spillage to local/remote disk
- [ ] Result cache (24 hr) vs warehouse local disk cache vs remote (storage) cache **[Interview Hotspot]**

### 5.2 Clustering **[Interview Hotspot]**
- [ ] Natural clustering vs explicit **Clustering Keys**
- [ ] When a table needs clustering (large tables, frequent filtering on non-ingestion-order column)
- [ ] `SYSTEM$CLUSTERING_INFORMATION()` to check clustering health
- [ ] Automatic Clustering service and its cost

### 5.3 Search Optimization Service
- [ ] When to use it (point lookups, selective equality filters on high-cardinality columns)
- [ ] Cost/benefit vs clustering

### 5.4 Materialized Views vs Dynamic Tables vs Regular Views — performance tradeoffs

### 5.5 Cost Optimization **[Interview Hotspot — DE role always touches cost]**
- [ ] Understanding credit consumption: compute credits vs storage costs vs serverless features (Snowpipe, Search Optimization, Automatic Clustering)
- [ ] Resource Monitors — setting credit quotas and alerts
- [ ] Right-sizing warehouses; scale up vs scale out (multi-cluster) decision
- [ ] Query tagging (`QUERY_TAG`) for cost attribution/chargeback
- [ ] `ACCOUNT_USAGE` and `INFORMATION_SCHEMA` views for cost/usage analysis (`WAREHOUSE_METERING_HISTORY`, `QUERY_HISTORY`)

---

## Phase 6: Data Protection, Governance & Reliability

### 6.1 Time Travel & Fail-safe **[Interview Hotspot]**
- [ ] Time Travel: querying historical data (`AT`/`BEFORE` clause), `UNDROP TABLE`
- [ ] Retention periods by table type and edition (0-1 day Standard, up to 90 days Enterprise+)
- [ ] Fail-safe: 7-day non-configurable recovery window (Snowflake-managed, for disaster recovery only)
- [ ] Difference between Time Travel and Fail-safe (who can access, cost)

### 6.2 Zero-Copy Cloning **[Interview Hotspot]**
- [ ] `CREATE ... CLONE` for databases, schemas, tables
- [ ] Why it's "zero-copy" (metadata pointer, no data duplication until modified)
- [ ] Use cases: dev/test environment refresh, pre-deployment backups

### 6.3 Data Sharing
- [ ] Secure Data Sharing (no data copying, shares metadata pointers across accounts)
- [ ] Reader accounts vs full accounts as data consumers
- [ ] Snowflake Marketplace basics

### 6.4 Security & Governance **[Interview Hotspot]**
- [ ] RBAC model: Users → Roles → Privileges; role hierarchy (SYSADMIN, SECURITYADMIN, ACCOUNTADMIN)
- [ ] Custom roles and least-privilege design
- [ ] Row Access Policies (row-level security)
- [ ] Dynamic Data Masking / Column-level security
- [ ] Object Tagging and Data Classification
- [ ] Network Policies, encryption at rest/in transit (high level, not deep security cert)

---

## Phase 7: Data Engineering Tooling Ecosystem Around Snowflake

- [ ] **dbt (data build tool)** integration with Snowflake — very common in real DE jobs
- [ ] Orchestration tools: Airflow / Snowflake Tasks / Azure Data Factory / Fivetran — where Snowflake fits
- [ ] Snowpark for Python — DataFrame-style transformations, UDFs, ML feature engineering
- [ ] **Snowflake Connectors**: Kafka connector, Spark connector, JDBC/ODBC
- [ ] CI/CD for Snowflake (schemachange, dbt Cloud, Snowflake CLI/DCM, GitHub Actions)
- [ ] Iceberg Tables in Snowflake (open table format support — increasingly relevant)

---

## Phase 8: Real-World Practice Projects (Apply Everything)

1. **Batch ETL pipeline**: Load CSV/JSON files from an external stage (simulate S3) → `COPY INTO` raw tables → transform with views/tasks → load into a star-schema mart.
2. **CDC pipeline**: Use Streams + Tasks to capture incremental changes from a source table and merge into a target table (`MERGE` + Stream + Task combo).
3. **Near real-time pipeline**: Set up Snowpipe to auto-ingest files landing in a stage.
4. **Dynamic Table pipeline**: Rebuild project #2 using Dynamic Tables instead of Streams+Tasks — compare complexity/cost.
5. **Cost governance exercise**: Set up a Resource Monitor, query `ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY`, and produce a cost report per warehouse.
6. **Cloning exercise**: Clone a production schema into a "dev" schema, modify data, verify original is untouched.
7. **Performance tuning exercise**: Take a large table, run a bad query (no pruning), check Query Profile, then add clustering key/rewrite query and compare execution stats.

---

## Phase 9: Interview Prep Checklist

Be ready to explain, in your own words, with examples:
- [ ] Snowflake's architecture and why storage/compute separation matters for cost & scale
- [ ] Difference between `COPY INTO` and Snowpipe — when to pick which
- [ ] How Streams + Tasks implement CDC, and how Dynamic Tables offer a simpler alternative
- [ ] Time Travel vs Fail-safe vs Zero-Copy Cloning — differences and use cases
- [ ] How micro-partitioning and clustering affect query performance (pruning)
- [ ] Warehouse sizing and auto-suspend/resume — how you'd control costs on a real project
- [ ] `MERGE` statement syntax and use case (SCD Type 1/2 implementations)
- [ ] How to handle semi-structured JSON data end-to-end (ingest → flatten → model)
- [ ] RBAC design for a data platform (who gets which role/privilege)
- [ ] A time you diagnosed and fixed a slow query in Snowflake (walk through Query Profile)

---

## Suggested Study Order Summary
1. Architecture (Phase 1) — foundation, don't skip
2. Core SQL + semi-structured data (Phase 2)
3. Data loading: Stages, COPY INTO, Snowpipe (Phase 3)
4. Transformation pipelines: Streams, Tasks, Dynamic Tables (Phase 4)
5. Performance & cost optimization (Phase 5)
6. Governance & reliability features (Phase 6)
7. Ecosystem tools — dbt, Airflow, Snowpark (Phase 7)
8. Build projects (Phase 8) in parallel with above, not after
9. Interview prep (Phase 9) — continuous, revisit weekly

---

## Free Resources to Pair With This Roadmap
- Snowflake official documentation (docs.snowflake.com) — always the source of truth
- Snowflake Hands-On Labs (free guided labs on trial account)
- SnowPro Core Certification study guide (even if not certifying, the exam guide is a great topic checklist)
- Snowflake's own YouTube channel — "Snowflake University" content

---

*Track your progress by checking off boxes above. Revisit Phase 9 weekly once you've covered Phases 1-7.*

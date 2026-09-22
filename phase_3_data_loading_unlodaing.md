# Phase 3: Data Loading & Unloading — Deep Dive

> Prerequisite: Phase 1 (Architecture) and Phase 2 (Core SQL & Data Objects). This phase is where you actually get data INTO and OUT OF Snowflake — the core day-to-day skill of a Data Engineer.

---

## 3.0 The Big Picture: How Data Gets Into Snowflake

There's a fixed mental model to hold onto for this entire phase:

```
Your Files (CSV/JSON/Parquet)
        │
        ▼
   ┌─────────┐
   │  STAGE   │   ← a "waiting area" pointer to files (internal or external)
   └────┬────┘
        │  (uses a FILE FORMAT to know how to parse the files)
        ▼
   ┌───────────┐
   │ COPY INTO  │   ← bulk-loads staged files into a table (batch)
   └────┬──────┘        OR
        │           ┌──────────┐
        │           │ SNOWPIPE  │  ← auto-loads staged files continuously (streaming/event-driven)
        │           └──────────┘
        ▼
   ┌─────────┐
   │  TABLE   │
   └─────────┘
```

Every loading pipeline you ever build follows this same shape: **Stage → File Format → COPY INTO (or Snowpipe) → Table**. Learn each piece well.

---

## 3.1 Stages **[Interview Hotspot]**

A **Stage** is simply a pointer to a location where data files sit before (or after) being loaded/unloaded. Stages don't hold data themselves in the loading sense — they reference file storage, either inside Snowflake (internal) or in your own cloud storage bucket (external).

### 3.1.1 Internal Stages — Three Flavors

Every table and every user automatically gets a built-in stage, plus you can create your own named stages.

**1. User Stage** — `@~`
- Automatically exists for every user. Private to that user, used for files only that user will load, not tied to any one table.
```sql
-- List files sitting in your personal user stage
LIST @~;
```

**2. Table Stage** — `@%table_name`
- Automatically exists for every table. Only usable for loading data into that specific table.
```sql
LIST @%orders_permanent;
```

**3. Named Internal Stage** — created explicitly, most flexible, most commonly used in real pipelines
```sql
CREATE STAGE my_internal_stage
  FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1);
```
Named stages are reusable across multiple tables/pipelines and can have their own default file format and directory table settings — this is what you'll use 90% of the time in real projects.

### 3.1.2 External Stages — Pointing at Your Own Cloud Storage

An external stage references files that live in **your** S3 bucket / Azure Blob container / GCS bucket — Snowflake never takes ownership/copies of the raw files here, it just reads from them (for `COPY INTO` loading) or writes to them (for unloading).

```sql
-- Example: S3-based external stage (requires a Storage Integration for secure, keyless auth
-- — Storage Integrations are covered in the Integrations topic, but here's the simple form)
CREATE STAGE my_s3_stage
  URL = 's3://my-data-bucket/raw/'
  CREDENTIALS = (AWS_KEY_ID = '<key>' AWS_SECRET_KEY = '<secret>')
  FILE_FORMAT = (TYPE = 'JSON');

-- Azure Blob example
CREATE STAGE my_azure_stage
  URL = 'azure://myaccount.blob.core.windows.net/mycontainer/raw/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<sas_token>')
  FILE_FORMAT = (TYPE = 'PARQUET');
```
> Best practice in production: use a **Storage Integration** object instead of embedding raw keys/secrets in the stage DDL — this avoids hardcoding credentials and is the standard approach for real projects (covered under the "Integrations" topic).

### 3.1.3 Uploading Files — the PUT Command

`PUT` uploads a **local file from your machine** into an internal stage. (You cannot `PUT` into an external stage — for external storage, you upload directly via cloud provider tools like AWS CLI, or just drop files into the bucket.)

```sql
-- Run via SnowSQL CLI (not a Snowsight worksheet — PUT needs local filesystem access)
PUT file:///Users/harsh/data/orders_2024_01.csv @my_internal_stage;

-- Common options
PUT file:///Users/harsh/data/*.csv @my_internal_stage
  AUTO_COMPRESS = TRUE     -- gzip compress before upload (default TRUE)
  OVERWRITE = TRUE;
```

### 3.1.4 Listing and Removing Staged Files

```sql
-- See what files are sitting in a stage right now
LIST @my_internal_stage;

-- Filter listing by a path/pattern
LIST @my_internal_stage/orders_2024;

-- Remove files after successful load (good hygiene — avoids re-processing/confusion)
REMOVE @my_internal_stage/orders_2024_01.csv;

-- Remove everything in the stage
REMOVE @my_internal_stage;
```

**[Interview Hotspot]** Be ready to explain the difference between the 3 internal stage types and when you'd pick a Named Internal Stage over relying on the default User/Table stage (answer: reusability across pipelines, shared team access, dedicated file format config).

---

## 3.2 File Formats

A **File Format** object tells Snowflake how to parse the raw bytes in your files — delimiter, header rows, compression, date formats, null handling, etc. You can define it inline in a `COPY INTO`/stage statement, or create it once as a reusable named object (best practice).

```sql
-- Named, reusable CSV file format
CREATE FILE FORMAT my_csv_format
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  NULL_IF = ('NULL', 'null', '')
  EMPTY_FIELD_AS_NULL = TRUE
  COMPRESSION = 'GZIP';

-- Named JSON file format
CREATE FILE FORMAT my_json_format
  TYPE = 'JSON'
  STRIP_OUTER_ARRAY = TRUE   -- unwrap a top-level [ ] array into individual records
  COMPRESSION = 'AUTO';

-- Named Parquet file format
CREATE FILE FORMAT my_parquet_format
  TYPE = 'PARQUET';

-- Attach a file format to a stage as its default
CREATE STAGE my_stage
  FILE_FORMAT = my_csv_format;
```

### Supported Types & Common Options

| Type | Common Options |
|---|---|
| CSV | `FIELD_DELIMITER`, `SKIP_HEADER`, `NULL_IF`, `ESCAPE`, `ENCODING` |
| JSON | `STRIP_OUTER_ARRAY`, `ALLOW_DUPLICATE`, `IGNORE_UTF8_ERRORS` |
| PARQUET | `BINARY_AS_TEXT`, generally needs fewer options — schema is self-describing |
| AVRO | Similar to Parquet — self-describing schema |
| ORC | Rarely used compared to Parquet, but supported |
| XML | `STRIP_OUTER_ELEMENT` |

### Compression Types Supported
`AUTO`, `GZIP`, `BZ2`, `BROTLI`, `ZSTD`, `DEFLATE`, `RAW_DEFLATE`, `NONE`

```sql
-- Snowflake can auto-detect compression on load
CREATE FILE FORMAT auto_compress_format
  TYPE = 'CSV'
  COMPRESSION = 'AUTO';
```

> Tip: Parquet/Avro files are usually preferred over CSV/JSON for large-scale loading — self-describing schema, columnar (for Parquet), and typically smaller file sizes due to better native compression.

---

## 3.3 Bulk Loading — COPY INTO **[Interview Hotspot]**

`COPY INTO` is the primary batch-loading mechanism in Snowflake — it reads files from a stage and loads them into a table.

```sql
-- Basic load
COPY INTO orders_permanent
FROM @my_internal_stage
FILE_FORMAT = (FORMAT_NAME = 'my_csv_format')
PATTERN = '.*orders_2024_.*[.]csv';   -- regex to filter which staged files to load

-- Loading directly with an inline file format (no named object)
COPY INTO raw_events (payload)
FROM @my_json_stage
FILE_FORMAT = (TYPE = 'JSON');
```

### Column Mapping / Transformations During Load

```sql
-- Reorder/transform columns while loading (common when source file column order
-- doesn't match target table, or you need light transformation on the fly)
COPY INTO orders_permanent (order_id, amount, order_date)
FROM (
    SELECT $1, $3, $2   -- $1, $2, $3 = positional column references in the staged file
    FROM @my_internal_stage/orders_2024_01.csv
)
FILE_FORMAT = (FORMAT_NAME = 'my_csv_format');
```

### Load History & Idempotency **[Interview Hotspot]**

Snowflake automatically tracks which files have already been successfully loaded into a table (metadata kept for 64 days by default) — so re-running `COPY INTO` pointed at the same stage will **skip files it already loaded**, preventing accidental duplicate loads.

```sql
-- Force reloading a file already marked as loaded (use with caution)
COPY INTO orders_permanent
FROM @my_internal_stage
FILE_FORMAT = (FORMAT_NAME = 'my_csv_format')
FORCE = TRUE;

-- Check load history for a table
SELECT * FROM INFORMATION_SCHEMA.LOAD_HISTORY
WHERE TABLE_NAME = 'ORDERS_PERMANENT'
ORDER BY LAST_LOAD_TIME DESC;

-- Broader load history across account (via ACCOUNT_USAGE, longer retention)
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE TABLE_NAME = 'ORDERS_PERMANENT'
ORDER BY LAST_LOAD_TIME DESC;
```

### VALIDATION_MODE — Dry Run Before Actually Loading

```sql
-- Test the load without actually inserting rows — returns errors if any
COPY INTO orders_permanent
FROM @my_internal_stage
FILE_FORMAT = (FORMAT_NAME = 'my_csv_format')
VALIDATION_MODE = 'RETURN_ERRORS';

-- Or validate just the first N rows quickly
COPY INTO orders_permanent
FROM @my_internal_stage
FILE_FORMAT = (FORMAT_NAME = 'my_csv_format')
VALIDATION_MODE = 'RETURN_10_ROWS';
```

### Error Handling — ON_ERROR Options **[Interview Hotspot]**

| Option | Behavior |
|---|---|
| `ABORT_STATEMENT` (default) | Stops the entire load on the first error row |
| `CONTINUE` | Skips just the bad row, keeps loading the rest |
| `SKIP_FILE` | Skips the entire file if any row in it errors |
| `SKIP_FILE_<N>` | Skips the file only if error count exceeds N |
| `SKIP_FILE_<N>%` | Skips the file if error percentage exceeds N% |

```sql
COPY INTO orders_permanent
FROM @my_internal_stage
FILE_FORMAT = (FORMAT_NAME = 'my_csv_format')
ON_ERROR = 'CONTINUE';
```
**Real-world guidance:** `CONTINUE` is common for messy source data where losing a few bad rows is acceptable (log them separately); `ABORT_STATEMENT` is safer for critical financial data where you want to know immediately if anything is malformed.

### File Sizing Best Practices

- Aim for **100–250 MB compressed per file** for optimal parallel load performance.
- Too many tiny files = overhead per file dominates (metadata management cost).
- One giant file = can't be parallelized across the warehouse's compute nodes as effectively.
- If you control the file-splitting on the source side (e.g., in an extraction job), split large exports into this size range before staging them.

### Practice Exercise
```sql
CREATE OR REPLACE TABLE stg_orders (
    order_id NUMBER,
    customer_id NUMBER,
    order_date DATE,
    amount NUMBER(10,2)
);

CREATE OR REPLACE FILE FORMAT csv_fmt
  TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1 NULL_IF = ('');

CREATE OR REPLACE STAGE demo_stage FILE_FORMAT = csv_fmt;

-- (Upload a sample orders.csv via PUT using SnowSQL, or use the Snowsight
--  "Load Data" wizard UI which wraps PUT + COPY INTO for you)

COPY INTO stg_orders
FROM @demo_stage
ON_ERROR = 'CONTINUE';

SELECT * FROM stg_orders;
SELECT * FROM INFORMATION_SCHEMA.LOAD_HISTORY WHERE TABLE_NAME = 'STG_ORDERS';
```

---

## 3.4 Continuous / Streaming Loading — Snowpipe **[Interview Hotspot]**

### 3.4.1 What Is Snowpipe?

`COPY INTO` is something *you* run — manually or on a schedule (e.g., via a Task, Phase 4 topic). **Snowpipe** flips this: it automatically loads new files **as soon as they land** in a stage, triggered by cloud storage event notifications, without you running anything.

```
File lands in S3 bucket
        │
        ▼
S3 Event Notification (via SNS/SQS)
        │
        ▼
Snowpipe automatically triggered
        │
        ▼
Data loaded into target table within ~1 minute typically
```

### 3.4.2 Setting Up Snowpipe

```sql
CREATE PIPE my_pipe
  AUTO_INGEST = TRUE   -- listens to cloud storage event notifications automatically
AS
COPY INTO orders_permanent
FROM @my_s3_stage
FILE_FORMAT = (FORMAT_NAME = 'my_csv_format');
```

For `AUTO_INGEST = TRUE` to work, you must also configure your cloud provider's event notification (e.g., an S3 Event Notification pointed at an SQS queue that Snowflake listens to) — Snowflake gives you the SQS ARN to plug into your bucket's event config after pipe creation:

```sql
-- Get the notification channel details needed to configure S3 event notifications
SHOW PIPES LIKE 'my_pipe';
-- Look at the "notification_channel" column in the result — that's the SQS ARN
```

### 3.4.3 Manual/REST-Triggered Snowpipe (Alternative to Auto-Ingest)

If you don't want to wire up cloud event notifications, you can call Snowpipe's REST API (or `ALTER PIPE ... REFRESH`) yourself whenever new files are ready — useful when your orchestrator (e.g., Airflow) already knows exactly when a new file lands.

```sql
-- Manually trigger the pipe to check for new files right now
ALTER PIPE my_pipe REFRESH;
```

### 3.4.4 Monitoring Snowpipe

```sql
-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('my_pipe');

-- See recent Snowpipe load activity
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE PIPE_NAME = 'MY_PIPE'
ORDER BY LAST_LOAD_TIME DESC;

-- Pause/resume a pipe
ALTER PIPE my_pipe SET PIPE_EXECUTION_PAUSED = TRUE;
ALTER PIPE my_pipe SET PIPE_EXECUTION_PAUSED = FALSE;
```

### 3.4.5 Snowpipe vs COPY INTO — When to Use Which **[Interview Hotspot]**

| | `COPY INTO` | Snowpipe |
|---|---|---|
| Trigger | Manual or scheduled (Task) | Automatic, event-driven |
| Latency | Whenever you run it / your schedule interval | Near real-time (~seconds to a couple minutes) |
| Compute | Uses a Virtual Warehouse you specify | Serverless — Snowflake manages compute automatically |
| Billing | Warehouse credits (per-second, based on size) | Serverless compute billed **per credit-second** actually used, independent of any warehouse |
| Best for | Predictable batch loads (nightly ETL) | Files arriving continuously/unpredictably (IoT, event streams, frequent file drops) |

### 3.4.6 Snowpipe Streaming API — Row-Level, Even Lower Latency **[Interview Hotspot]**

Regular Snowpipe still works at the **file** level (waits for a complete file to land). **Snowpipe Streaming** goes further — it lets an application push **individual rows** directly into Snowflake tables via an SDK, without ever writing an intermediate file to a stage at all.

- Used for true low-latency streaming use cases (e.g., a Kafka consumer writing rows directly, IoT sensor data).
- Lower latency than file-based Snowpipe since there's no "wait for file, then notice the file, then load it" cycle.
- Typically integrated via the **Kafka connector** (which can use Snowpipe Streaming under the hood) or directly via the Snowpipe Streaming Java SDK in a custom application.
- Conceptually: `COPY INTO` = batch, file-based Snowpipe = event-driven but still file-based, Snowpipe Streaming = true row-by-row streaming, no files at all.

### 3.4.7 Cost Model for Snowpipe

Snowpipe uses **serverless compute** — meaning Snowflake automatically provisions and manages the compute behind the scenes, and you're billed **per credit-second actually consumed** during each load operation, not for an idle running warehouse. This is generally cost-efficient for irregular/bursty file arrival patterns, but can add up if you have an extremely high volume of very small, frequent files (file-count overhead applies here too, same principle as `COPY INTO` file-sizing best practices).

```sql
-- Check Snowpipe credit usage
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE PIPE_NAME = 'MY_PIPE'
ORDER BY START_TIME DESC;
```

---

## 3.5 Unloading Data — Getting Data OUT of Snowflake

### 3.5.1 COPY INTO <location> — Exporting to a Stage

The same `COPY INTO` command works in reverse — pointing it at a stage location (instead of a table) exports query results or an entire table into files.

```sql
-- Export a full table to an internal stage as compressed CSV
COPY INTO @my_internal_stage/exports/orders_
FROM orders_permanent
FILE_FORMAT = (TYPE = 'CSV' COMPRESSION = 'GZIP' FIELD_DELIMITER = ',')
HEADER = TRUE
SINGLE = FALSE      -- FALSE = split into multiple files (parallelized, recommended for large exports)
MAX_FILE_SIZE = 104857600;  -- ~100 MB per output file

-- Export the result of an arbitrary query, not just a raw table
COPY INTO @my_internal_stage/exports/high_value_
FROM (SELECT * FROM orders_permanent WHERE amount > 1000)
FILE_FORMAT = (TYPE = 'PARQUET');

-- Export directly to an external stage (S3/Blob/GCS) — common for handing data
-- off to another system/team
COPY INTO @my_s3_stage/exports/
FROM orders_permanent
FILE_FORMAT = (TYPE = 'PARQUET');
```

### 3.5.2 GET Command — Downloading Files From an Internal Stage

Just like `PUT` uploads local files to an internal stage, `GET` downloads files from an internal stage back to your local machine (run via SnowSQL CLI, not Snowsight worksheet).

```sql
GET @my_internal_stage/exports/orders_0_0_0.csv.gz file:///Users/harsh/downloads/;
```

For external stages, you don't need `GET` — you just retrieve the files directly from your cloud storage console/CLI (e.g., `aws s3 cp`), since Snowflake wrote them straight there.

### Practice Exercise
```sql
-- Export the practice table you loaded earlier
COPY INTO @demo_stage/export_
FROM stg_orders
FILE_FORMAT = (TYPE = 'CSV' COMPRESSION = 'GZIP')
HEADER = TRUE;

LIST @demo_stage;
-- Then, in SnowSQL CLI: GET @demo_stage/export_0_0_0.csv.gz file:///tmp/;
```

---

## Summary: What You Should Be Able to Explain After This Phase

1. The full loading pipeline shape: Stage → File Format → COPY INTO/Snowpipe → Table.
2. The 3 internal stage types (User, Table, Named) vs external stages, and when to use `PUT`/`LIST`/`REMOVE`.
3. How File Format objects control parsing (CSV/JSON/Parquet options), and why Parquet/Avro are often preferred at scale.
4. `COPY INTO` mechanics: load history/idempotency (won't reload the same file twice unless `FORCE = TRUE`), `VALIDATION_MODE` for dry runs, and all `ON_ERROR` options.
5. File sizing best practice (100–250MB compressed) and why it matters for parallelism.
6. Snowpipe: how `AUTO_INGEST` works with cloud event notifications, how to monitor/pause it, and precisely when to choose Snowpipe over `COPY INTO`.
7. Snowpipe Streaming API — how it differs from file-based Snowpipe (row-level, no staged files at all).
8. Serverless billing model for Snowpipe (credit-seconds, no warehouse needed).
9. How to unload data with `COPY INTO <stage>` (including exporting arbitrary query results) and retrieve files locally with `GET`.

---

## Next Step
Once comfortable, move to **Phase 4: Data Transformation Pipelines** (Streams, Tasks, Dynamic Tables, Stored Procedures/Snowpark) from `roadmap.md` — this is where loaded raw data gets automated and transformed into production-ready tables.

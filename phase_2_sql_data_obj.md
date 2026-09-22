# Phase 2: Core SQL & Data Objects in Snowflake — Deep Dive

> Prerequisite: Phase 1 (Architecture). Everything here builds on the idea that data lives in immutable, columnar micro-partitions managed automatically by Snowflake.

---

## 2.0 Object Hierarchy — Where Everything Lives

Snowflake organizes objects in a strict hierarchy:

```
ACCOUNT
  └── DATABASE
        └── SCHEMA
              ├── TABLE
              ├── VIEW
              ├── STAGE
              ├── FILE FORMAT
              ├── SEQUENCE
              ├── FUNCTION / PROCEDURE
              └── STREAM / TASK
```

Every object (table, view, etc.) is uniquely identified by a **3-part name**: `DATABASE.SCHEMA.OBJECT`.

## VIEWS : 

1) Views are objects which store the sql query as is, they are just abstraction for reuability in complex multi-join queries, helps to avoid giving complete access 

2) when u execute a view , behind the scenes the actual query is executed against the table and all

3) View are like virtual tables, they dont store actual data like table.
```sql
CREATE VIEW high_value_orders AS
SELECT order_id, customer_id, amount
FROM orders
WHERE amount > 10000;
```

- Views are always select only !! u should never use insert, update, delete in views, and while calling views u can select only 

```sql
-- Filter on it
SELECT * FROM my_view WHERE region = 'APAC';

-- Join it with another table or view
SELECT v.*, c.customer_name
FROM my_view v
JOIN customers c ON v.customer_id = c.customer_id;

-- Aggregate on it
SELECT region, COUNT(*) FROM my_view GROUP BY region;

-- Reference it in another view (standard/secure only, not materialized)
CREATE VIEW another_view AS SELECT * FROM my_view WHERE amount > 500;

```


4) In snowflake there are 3 types of views : 

- normal views 
CREATE VIEW my_view AS SELECT ...;
- no restriction on normal views !!

- secure views : the view definition is not visible to non-owner
CREATE SECURE VIEW my_secure_view AS SELECT ...;

- materialized view :
CREATE MATERIALIZED VIEW mv_daily_sales AS SELECT ...;
  - if we are using some dashboard which is querying an expensive aggregation query again and again so its better to compute it once and stored in storage, but if the underlying data changes then its automatically updated in materialized views storage. 
  - we cant use joins in materialized views !! so highly doubtful for dashboard 😅, as they use joins mostly
  - we cant have nested views in materialized views, cant use window func, subquery, udf
- if underlying base table data is changing very frequently then theres no point of using materialized views

```sql
-- Create the hierarchy from scratch
CREATE DATABASE learn_db;
CREATE SCHEMA learn_db.sales;

-- Fully-qualified reference
CREATE TABLE learn_db.sales.orders (
    order_id NUMBER,
    order_date DATE
);

-- Or set context first, then use short names
USE DATABASE learn_db;
USE SCHEMA sales;
CREATE TABLE customers (customer_id NUMBER, name STRING);

-- Check current context anytime
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_WAREHOUSE();
```

Every account also comes with two built-in databases: `SNOWFLAKE` (metadata/usage views like `ACCOUNT_USAGE`) and `SNOWFLAKE_SAMPLE_DATA` (the practice dataset you used in Phase 1). Both of these are Read only DB !

- **SNOWFLAKE DB contains metadata info about ur account, it containes 2 main views : ACCOUNT_USAGE and INFORMATION_SCHEMA** 

1) ACCOUNT_USAGE SCHEMA : (ONLY AVAILABLE IN THIS DB)
contains lot of views
- QUERY_HISTORY : bytes scanned , wh used, spill, etc
- WAREHOUSE_METERING_HISTORY : credit consumption per warehouse over time
- ACCESS_HISTORY : 
- GRANTS_TO_USERS, GRANTS_TO_ROLES :
- TASK_HISTORY, COPY_HISTORY, PIPE_USAGE_HISTORY : 
- Basically: anything you'd want to know about "what happened in my account" — cost, security, performance, access — lives here.
- Scope is account wide , history retained for 1 year , Latency is 45min-3hr 

2) INFORMATION-SCHEMA : (PER DB THIS SCHEMA EXISTS) 
contains lot of views 
- This schema also contains a lot of views around db specific info
- Near real time no delay info !!!

- **In short ACCOUNT_USAGE : account wide history, costing ,etc : latency delay, contains dropped obj**
- **Information_schema : Db level details , near real time metadata, no dropped obj**

### Retention of deleted objects in snowflake :
- only the data holding objects can be time traveled like table(some) , materialized view,etc 

---

## 2.1 Table Types **[Interview Hotspot]**

Snowflake has 4 table types. The differences matter a lot for cost and data-protection guarantees.

### 2.1.1 Permanent Tables (default)
- Default type when you just say `CREATE TABLE`.
- Full **Time Travel** support (1 day on Standard edition, up to 90 days on Enterprise+).
- Full **Fail-safe** support (extra 7-day disaster recovery window after Time Travel expires).
- Costs the most in storage (because of Time Travel + Fail-safe retained versions), used for production data you can't afford to lose.

```sql
CREATE TABLE orders_permanent (
    order_id NUMBER,
    amount NUMBER(10,2)
);
-- Equivalent to:
CREATE OR REPLACE TABLE orders_permanent (
    order_id NUMBER,
    amount NUMBER(10,2)
);
```

### 2.1.2 Temporary Tables
- Exist only for the **duration of the session** that created them. Automatically dropped when the session ends.
- Not visible to other users/sessions at all — even if same table name exists elsewhere.
- No Fail-safe. Time Travel max 1 day (but irrelevant since the table disappears when session ends anyway).
- Use case: intermediate scratch tables inside a single ETL script/session.

```sql
CREATE TEMPORARY TABLE staging_orders (
    order_id NUMBER,
    raw_payload VARIANT
);
-- This table vanishes automatically once you close this session/worksheet
```

### 2.1.3 Transient Tables
- Persist across sessions (unlike Temporary) but have **NO Fail-safe** period at all.
- Time Travel limited to 0 or 1 day only.
- Cheaper to store than Permanent tables (no 7-day Fail-safe storage overhead).
- Use case: staging/ETL intermediate tables that are reproducible from source — you don't need disaster recovery on data you can just reload.

```sql
CREATE TRANSIENT TABLE staging.orders_raw (
    order_id NUMBER,
    payload VARIANT
);
```

### 2.1.4 External Tables
- Data physically stays in your cloud storage (S3/Blob/GCS) — Snowflake does NOT copy it in. Snowflake only stores **metadata** pointing to the files.
- Query the files directly using SQL, without a load/`COPY INTO` step.
- Slower than native tables (no micro-partition metadata optimizations the same way), but useful for data lake scenarios or infrequently-queried archives.

```sql
CREATE EXTERNAL TABLE ext_orders (
    order_id NUMBER AS (value:order_id::NUMBER),
    order_date DATE AS (value:order_date::DATE)
)
LOCATION = @my_s3_stage/orders/
FILE_FORMAT = (TYPE = PARQUET);
```
(You'll build a full working stage + external table example once you hit Phase 3 — Stages.)

### Comparison Table **[Interview Hotspot — asked almost every time]**

| Feature | Permanent | Temporary | Transient | External |
|---|---|---|---|---|
| Lifespan | Until dropped | Session only | Until dropped | Until dropped (metadata only) |
| Time Travel | up to 90 days | up to 1 day | up to 1 day | N/A |
| Fail-safe | 7 days | None | None | N/A |
| Storage cost | Highest | Low (short-lived) | Lower than Permanent | Lowest (data not stored in Snowflake) |
| Use case | Production data | Session scratch work | ETL staging | Data lake / archive |

### Copying Table Structure & Cloning

```sql
-- Copy structure only, no data
CREATE TABLE orders_copy LIKE orders_permanent;

-- Zero-copy clone: copies structure AND data instantly, no extra storage
-- until you modify the clone (covered in depth in Phase 6)
CREATE TABLE orders_clone CLONE orders_permanent;
```

---

## 2.2 Views, Materialized Views, Secure Views

### 2.2.1 Regular Views
- A saved SQL query. No data is stored — every time you query the view, Snowflake re-runs the underlying SQL against current data.

```sql
CREATE VIEW v_high_value_orders AS
SELECT order_id, customer_id, amount
FROM orders_permanent
WHERE amount > 1000;

SELECT * FROM v_high_value_orders;
```

### 2.2.2 Materialized Views **[Interview Hotspot]**
- Unlike a regular view, a Materialized View **physically stores** the query results, and Snowflake **automatically keeps it refreshed** in the background whenever the base table changes.
- Faster to query than a regular view (no re-computation on each query) — but costs extra: storage for the materialized results, plus background compute credits used by Snowflake to keep it in sync.
- Restrictions: only one table in the `FROM` (no joins), no `ORDER BY`, limited aggregate function support.

```sql
CREATE MATERIALIZED VIEW mv_daily_order_totals AS
SELECT order_date, SUM(amount) AS total_amount
FROM orders_permanent
GROUP BY order_date;
```
Use when: the base table changes relatively infrequently but the aggregation is queried very often (e.g., a daily summary dashboard hit thousands of times a day).

**When NOT to use it:** if the base table changes very frequently (every few seconds), the refresh overhead can exceed the benefit — a Dynamic Table (Phase 4) is often a better fit for such cases.

### 2.2.3 Secure Views
- Hides the view's *underlying query definition* from users who have access to query the view but not the base tables.
- Slight performance cost (Snowflake can't apply certain query optimizations across the view boundary), but required for governance/data-sharing scenarios.

```sql
CREATE SECURE VIEW v_customer_summary AS
SELECT customer_id, COUNT(*) AS order_count
FROM orders_permanent
GROUP BY customer_id;
```
Use case: sharing a view with another Snowflake account (Secure Data Sharing, Phase 6) — you don't want the consumer to see your internal SQL logic or unfiltered base table structure via `GET_DDL()`/`SHOW VIEWS`.

---

## 2.3 Semi-Structured Data Handling **[Interview Hotspot — very commonly tested]**

Snowflake can natively store and query JSON, Avro, Parquet, ORC, and XML without forcing you to flatten it into rigid columns first. This is one of Snowflake's standout features vs traditional relational databases.

### 2.3.1 The Semi-Structured Data Types

| Type | Description |
|---|---|
| `VARIANT` | Can hold ANY type of data — a JSON object, array, string, number. The universal semi-structured container. |
| `OBJECT` | Key-value pairs (like a JSON object specifically). |
| `ARRAY` | Ordered list of values. |

```sql
CREATE TABLE raw_events (
    event_id NUMBER,
    payload VARIANT
);

-- Insert JSON directly using PARSE_JSON
INSERT INTO raw_events
SELECT 1, PARSE_JSON('{
    "user": {"id": 101, "name": "Alice"},
    "actions": ["login", "click", "logout"],
    "score": 87.5
}');
```

### 2.3.2 Querying Nested JSON — Dot and Bracket Notation

```sql
SELECT
    payload:user.id::NUMBER          AS user_id,     -- dot notation, cast with ::
    payload:user.name::STRING        AS user_name,
    payload:score::FLOAT             AS score,
    payload:actions[0]::STRING       AS first_action  -- array index access
FROM raw_events;
```
Key rule: `:` accesses a key, `::TYPE` casts the VARIANT value to a concrete SQL type (always do this — without casting, results stay VARIANT type and comparisons/joins behave unexpectedly).

### 2.3.3 FLATTEN() — Turning Arrays/Nested Objects into Rows **[Interview Hotspot]**

`FLATTEN` is a table function that explodes an array or object into one row per element — similar in spirit to `UNNEST` in other SQL dialects.

```sql
-- Explode the "actions" array into one row per action
SELECT
    e.event_id,
    f.value::STRING AS action
FROM raw_events e,
     LATERAL FLATTEN(input => e.payload:actions) f;
```
Result:
```
EVENT_ID | ACTION
1        | "login"
1        | "click"
1        | "logout"
```

`FLATTEN` output columns you'll commonly use:
- `VALUE` — the actual element value
- `KEY` — key name (when flattening an OBJECT, not an ARRAY)
- `INDEX` — position (for arrays)
- `PATH` — full path from root

```sql
-- Flatten a nested OBJECT's key-value pairs
SELECT
    f.key   AS attr_name,
    f.value AS attr_value
FROM raw_events e,
     LATERAL FLATTEN(input => e.payload:user) f;
```

### 2.3.4 Building JSON — The Reverse Direction

```sql
SELECT OBJECT_CONSTRUCT(
    'order_id', order_id,
    'amount', amount
) AS order_json
FROM orders_permanent;

-- ARRAY_CONSTRUCT / ARRAY_AGG to build arrays
SELECT ARRAY_AGG(order_id) AS all_order_ids
FROM orders_permanent;
```

### 2.3.5 Useful Semi-Structured Functions Cheat Sheet

| Function | Purpose |
|---|---|
| `PARSE_JSON(str)` | Convert a JSON string into VARIANT |
| `TO_VARIANT(val)` | Cast a regular value into VARIANT |
| `OBJECT_CONSTRUCT(k1,v1,k2,v2,...)` | Build a JSON object |
| `ARRAY_CONSTRUCT(v1,v2,...)` | Build an array |
| `IS_ARRAY()`, `IS_OBJECT()` | Type-checking on VARIANT values |
| `TYPEOF(val)` | Returns the underlying data type of a VARIANT value |

### Practice Exercise
```sql
CREATE OR REPLACE TABLE raw_orders (id NUMBER, data VARIANT);

INSERT INTO raw_orders
SELECT 1, PARSE_JSON('{"customer":"Bob","items":[{"sku":"A1","qty":2},{"sku":"B2","qty":1}]}');

-- Explode items array and pull sku/qty into columns
SELECT
    r.id,
    r.data:customer::STRING AS customer,
    f.value:sku::STRING     AS sku,
    f.value:qty::NUMBER     AS qty
FROM raw_orders r,
     LATERAL FLATTEN(input => r.data:items) f;
```

---

## 2.4 Advanced SQL in Snowflake

### 2.4.1 Window Functions

Standard SQL window functions all work in Snowflake:

```sql
SELECT
    customer_id,
    order_id,
    amount,
    RANK()       OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS dense_rnk,
    LAG(amount)  OVER (PARTITION BY customer_id ORDER BY order_id)    AS prev_amount,
    LEAD(amount) OVER (PARTITION BY customer_id ORDER BY order_id)    AS next_amount
FROM orders_permanent;
```

### 2.4.2 QUALIFY Clause — Snowflake's Standout Feature **[Interview Hotspot]**

Normally, to filter on a window function result, you'd need a subquery/CTE (since window functions can't be used directly in `WHERE`). Snowflake's `QUALIFY` clause lets you filter on window function results directly, without a wrapping subquery.

```sql
-- WITHOUT QUALIFY (the "old way", needs a subquery)
SELECT * FROM (
    SELECT
        customer_id, order_id, amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rn
    FROM orders_permanent
) WHERE rn = 1;

-- WITH QUALIFY (Snowflake shortcut — cleaner, same result)
SELECT
    customer_id, order_id, amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rn
FROM orders_permanent
QUALIFY rn = 1;
```
**Classic use case:** "get the most recent record per customer" (deduplication) — this pattern appears constantly in real ETL work.

```sql
-- Deduplicate: keep only the latest row per customer_id based on updated_at
SELECT *
FROM customers
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) = 1;
```

### 2.4.3 CTEs and Recursive CTEs

```sql
-- Standard CTE
WITH high_value AS (
    SELECT * FROM orders_permanent WHERE amount > 1000
)
SELECT customer_id, COUNT(*) FROM high_value GROUP BY customer_id;

-- Recursive CTE — e.g., traversing an employee/manager hierarchy
WITH RECURSIVE org_chart AS (
    -- anchor: top-level employees (no manager)
    SELECT employee_id, manager_id, name, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- recursive step: join next level down
    SELECT e.employee_id, e.manager_id, e.name, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart ORDER BY level;
```

### 2.4.4 MERGE — Upserts **[Interview Hotspot — used constantly in real pipelines]**

`MERGE` combines INSERT + UPDATE (+ optionally DELETE) into one atomic statement — the backbone of most incremental/CDC-style loading patterns.

```sql
MERGE INTO orders_permanent AS target
USING staging_orders AS source
    ON target.order_id = source.order_id
WHEN MATCHED THEN
    UPDATE SET target.amount = source.amount,
               target.order_date = source.order_date
WHEN NOT MATCHED THEN
    INSERT (order_id, amount, order_date)
    VALUES (source.order_id, source.amount, source.order_date);
```

This single pattern is exactly how you'll implement **SCD Type 1** (overwrite) updates, and forms the basis of Streams+Tasks CDC pipelines in Phase 4.

**Optional: `WHEN MATCHED ... AND <condition>`** for more nuanced logic:
```sql
MERGE INTO orders_permanent AS t
USING staging_orders AS s
    ON t.order_id = s.order_id
WHEN MATCHED AND s.is_deleted = TRUE THEN DELETE
WHEN MATCHED THEN UPDATE SET t.amount = s.amount
WHEN NOT MATCHED THEN INSERT (order_id, amount) VALUES (s.order_id, s.amount);
```

### 2.4.5 PIVOT / UNPIVOT

```sql
-- PIVOT: turn row values into columns
SELECT *
FROM (SELECT customer_id, order_date, amount FROM orders_permanent)
PIVOT (SUM(amount) FOR order_date IN ('2024-01-01', '2024-01-02', '2024-01-03'))
AS pivoted;

-- UNPIVOT: turn columns back into rows
SELECT customer_id, order_date, amount
FROM monthly_totals
UNPIVOT (amount FOR order_date IN (jan_total, feb_total, mar_total));
```

### 2.4.6 Sampling

```sql
-- Random 10% sample of rows (fast, statistical sample not exact)
SELECT * FROM orders_permanent SAMPLE (10);

-- Sample a fixed number of rows
SELECT * FROM orders_permanent SAMPLE (1000 ROWS);

-- Reproducible sample using a seed
SELECT * FROM orders_permanent SAMPLE (10) SEED (42);
```
Use case: testing a query's logic on a huge table quickly without scanning everything, or generating a representative subset for ML training data.

### 2.4.7 Sequences and AUTOINCREMENT

```sql
-- Explicit sequence object
CREATE SEQUENCE order_id_seq START = 1 INCREMENT = 1;

INSERT INTO orders_permanent (order_id, amount)
VALUES (order_id_seq.NEXTVAL, 250.00);

-- Or use IDENTITY/AUTOINCREMENT directly on a column
CREATE TABLE orders_auto (
    order_id NUMBER AUTOINCREMENT START 1 INCREMENT 1,
    amount NUMBER(10,2)
);
INSERT INTO orders_auto (amount) VALUES (99.99);  -- order_id auto-generated
```

---

## Summary: What You Should Be Able to Explain After This Phase

1. The object hierarchy (Account → Database → Schema → Table/View) and how 3-part naming works.
2. All 4 table types (Permanent, Temporary, Transient, External) — differences in lifespan, Time Travel, Fail-safe, and when to use each.
3. Difference between a regular View, a Materialized View, and a Secure View — and a real use case for each.
4. How to store and query semi-structured JSON: `VARIANT`, dot notation, `::` casting, and `FLATTEN()` to explode arrays/objects into rows.
5. Why `QUALIFY` is a Snowflake-specific shortcut, and how to use it for the very common "latest row per group" deduplication pattern.
6. How `MERGE` implements upserts, and why it's the backbone of incremental data pipelines.
7. Basic PIVOT/UNPIVOT, SAMPLE, and sequence/AUTOINCREMENT usage.

---

## Next Step
Once comfortable, move to **Phase 3: Data Loading & Unloading** (Stages, File Formats, `COPY INTO`, Snowpipe) from `roadmap.md`.

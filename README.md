# Swiggy-Like Data Pipeline – Full Project Breakdown

> **What this document covers:** Every component, concept, and decision used in this end-to-end Snowflake data engineering project — explained with context, purpose, and significance.

---

## Table of Contents

1. [Project Overview & Business Context](#1-project-overview--business-context)
2. [Architecture: Three-Layer Data Flow](#2-architecture-three-layer-data-flow)
3. [Initial Environment Setup](#3-initial-environment-setup)
4. [Data Governance: Tags & Masking Policies](#4-data-governance-tags--masking-policies)
5. [Internal Stage & File Partitioning](#5-internal-stage--file-partitioning)
6. [COPY INTO Command & Audit Columns](#6-copy-into-command--audit-columns)
7. [Streams (CDC)](#7-streams-cdc)
8. [Normal Forms & Star Schema Design](#8-normal-forms--star-schema-design)
9. [Entity Relationship Diagram (ERD)](#9-entity-relationship-diagram-erd)
10. [Surrogate Keys & Hash Keys](#10-surrogate-keys--hash-keys)
11. [MERGE Operation](#11-merge-operation)
12. [Slowly Changing Dimensions (SCD)](#12-slowly-changing-dimensions-scd)
13. [Entity-by-Entity Processing Flow](#13-entity-by-entity-processing-flow)
14. [Fact Table: ORDER_ITEM_FACT](#14-fact-table-order_item_fact)
15. [Date Dimension](#15-date-dimension)
16. [Foreign Key Constraints](#16-foreign-key-constraints)
17. [KPI Views](#17-kpi-views)
18. [Part 2 – Pipeline Automation](#18-part-2--pipeline-automation)
19. [Stored Procedures (SPs)](#19-stored-procedures-sps)
20. [Snowflake Tasks & Task Tree](#20-snowflake-tasks--task-tree)
21. [Streamlit Dashboard](#21-streamlit-dashboard)
22. [Initial vs. Delta Load Strategy](#22-initial-vs-delta-load-strategy)

---

## 1. Project Overview & Business Context

### What is this project?

This project simulates an **end-to-end Snowflake data pipeline** modelled on a food aggregator app like Swiggy. It takes raw source data across multiple business entities and transforms it into an analytics-ready data warehouse with a Streamlit dashboard on top.

### Why food delivery as the domain?

Food delivery apps are familiar to everyone — restaurants, menus, orders, payments, deliveries. That familiarity makes it easier to understand data flows without getting distracted by domain complexity. The entities map cleanly to a data warehouse design.

### Business questions this project answers

- Total revenue and average revenue per item / order
- Top-performing restaurants
- Revenue trends over time (daily, monthly, annually)
- Revenue by customer segment, restaurant, and location
- Delivery performance metrics
- Geographic revenue breakdown

### Entities covered

| Entity | Type | Role in DW |
|---|---|---|
| Location | Master | Dimension |
| Restaurant | Master | Dimension |
| Menu | Master | Dimension |
| Customer | Master | Dimension |
| Customer Address | Master | Dimension |
| Delivery Agent | Master | Dimension |
| Orders | Transactional | Part of Fact |
| Order Items | Transactional | Central Fact |
| Delivery | Transactional | Part of Fact |
| Login Audit | Audit/Event | Supporting |

---

## 2. Architecture: Three-Layer Data Flow

```
CSV Files → Internal Stage → Stage Schema → Clean Schema → Consumption Schema → Streamlit Dashboard
```

This is a **Medallion-style layered architecture**, separating raw ingestion, cleansing, and consumption into distinct schemas.

### End-to-End Data Flow Diagram

<img width="1961" height="980" alt="Food-Aggregator-E2E-Data-Flow" src="https://github.com/user-attachments/assets/dfe69eb7-7db6-48ef-b251-26878f07b6e7" />

### Stage Schema (`stage_sch`)

**What it does:** Acts as the raw landing zone. CSV files are loaded here via COPY INTO with minimal transformation.

**Why it's used:**
- Decouples ingestion from transformation — if anything breaks downstream, raw data is still intact.
- Allows validation of what came in before processing it further.
- Audit columns added here trace every record back to its source file and load timestamp.

### Clean Schema (`clean_sch`)

**What it does:** Receives data from the stage schema via MERGE, applies correct data types, enriches records, and adds surrogate keys.

**Why it's used:**
- The stage layer stores everything as strings (text). The clean layer applies proper types (timestamps, booleans, numbers).
- Data enrichment happens here — e.g., state codes, union territory flags, city tier classification.
- Acts as an OLTP-style intermediate layer before dimensional modeling.

### Consumption Schema (`consumption_sch`)

**What it does:** Holds the final star schema — fact table and all dimension tables. This is what the Streamlit dashboard and BI tools query.

**Why it's used:**
- Optimised for analytical queries with denormalised dimensions.
- Hash keys (surrogate keys) are used instead of source system IDs.
- SCD Type 2 logic preserves historical changes for accurate time-based analysis.

### Common Schema (`common`)

**What it does:** Holds shared, reusable governance objects — tags, masking policies, and utility functions.

**Why it's used:** Centralising governance objects means they can be applied consistently across all schemas without duplication.

### Architecture Objective (Objective ERD)

<img width="1209" height="768" alt="Objective ERD" src="https://github.com/user-attachments/assets/f44b1fdd-c479-447a-885a-a8c8ba0fcd4a" />

The diagram above shows how raw OLTP-style source data (left) is transformed into an analytics-ready star schema (right) — master entities become dimension tables, transactional entities collapse into the central fact table.

---

## 3. Initial Environment Setup

### Role: SYSADMIN

```sql
use role sysadmin;
```

**Why SYSADMIN?** It has the privileges needed to create warehouses, databases, schemas, stages, and tables. In production you'd use more restrictive roles with least-privilege access, but for a dev/sandbox project SYSADMIN is the right starting point.

### Warehouse: `adhoc_wh`

```sql
create warehouse if not exists adhoc_wh
    warehouse_size = 'x-small'
    auto_resume = true
    auto_suspend = 60
    enable_query_acceleration = false
    warehouse_type = 'standard'
    min_cluster_count = 1
    max_cluster_count = 1
    scaling_policy = 'standard'
    initially_suspended = true;
```

**What it is:** The compute layer in Snowflake. All SQL execution, ETL processing, and query runs happen through the warehouse.

**Key settings and why:**

| Setting | Value | Significance |
|---|---|---|
| `warehouse_size = 'x-small'` | X-Small | Sufficient for sandbox/dev. Scaling up is one command away. |
| `auto_resume = true` | ON | Warehouse starts automatically when a query hits it — no manual intervention. |
| `auto_suspend = 60` | 60 seconds | Shuts down after 1 minute of inactivity. Critical for cost control. |
| `initially_suspended = true` | ON | Doesn't consume credits just by being created. |
| `enable_query_acceleration = false` | OFF | Not needed for dev volume. Useful for large analytical workloads. |

### Database and Schemas

```sql
create database if not exists sandbox;

create schema if not exists stage_sch;
create schema if not exists clean_sch;
create schema if not exists consumption_sch;
create schema if not exists common;
```

**Why a sandbox database?** Keeps all development objects isolated from production or other projects. Easy to drop and recreate without impacting anything else.

**Why four schemas?** Each schema has a distinct responsibility (raw / clean / analytical / governance). This prevents accidental mixing of raw and transformed data and makes permission management cleaner.

### Summary of Environment Setup in Snowflake

<img width="1184" height="615" alt="Sandbox environment setup in Snowflake" src="https://github.com/user-attachments/assets/c3b185d6-b078-40f4-864b-79215687bad7" />

---

## 4. Data Governance: Tags & Masking Policies

### Tag Object

```sql
create or replace tag common.pii_policy_tag
    allowed_values = ('PII', 'PRICE', 'SENSITIVE', 'EMAIL')
    comment = 'This is PII policy tag object';
```

**What it is:** A metadata label that can be attached to any column or table in Snowflake.

**Why it's used:**
- Classifies sensitive columns (email, phone, price, etc.) with a standard label.
- Foundation for applying masking policies — you attach the tag to a column, then bind the masking policy to that tag.
- Supports compliance requirements like GDPR and India's DPDP Act.

### Masking Policies

```sql
create or replace masking policy common.pii_masking_policy
    as (pii_text string) returns string -> to_varchar('** PII **');

create or replace masking policy common.email_masking_policy
    as (email_text string) returns string -> to_varchar('** EMAIL **');

create or replace masking policy common.phone_masking_policy
    as (phone string) returns string -> to_varchar('** Phone **');
```

**What it does:** Controls what a user sees when they query a sensitive column. Users without the right role see `** PII **` instead of an actual email address or phone number.

**Why it's significant:**
- Data engineers can build pipelines with real data while analysts and downstream consumers only see masked values.
- The masking happens at query time — the actual data is stored unmasked, so it can be unmasked for authorised users without any data transformation.
- In a production food delivery platform, customer emails and phone numbers are highly sensitive PII. This is non-negotiable.

---

## 5. Internal Stage & File Partitioning

### File Format

```sql
create file format if not exists stage_sch.csv_file_format
    type = 'csv'
    compression = 'auto'
    field_delimiter = ','
    record_delimiter = '\n'
    skip_header = 1
    field_optionally_enclosed_by = '\042'
    null_if = ('\\N');
```

**What it is:** A reusable definition that tells Snowflake how to parse CSV files during ingestion.

**Key options explained:**

| Option | Value | Why |
|---|---|---|
| `skip_header = 1` | ON | Ignores the first row (column headers) so it doesn't get loaded as data. |
| `field_optionally_enclosed_by = '\042'` | `"` (double quote) | Handles fields that have commas inside them (e.g., `"New York, NY"`). |
| `null_if = ('\\N')` | `\N` | Treats the string `\N` as NULL — a common pattern in MySQL CSV exports. |
| `compression = 'auto'` | Auto-detect | Works with gzip or uncompressed files without changing the format definition. |

**Why reusable?** Instead of specifying these options in every COPY INTO command, you reference the file format object once. Consistent parsing across all entities.

### Internal Stage

```sql
create stage stage_sch.csv_stg
    directory = (enable = true)
    comment = 'this is the snowflake internal stage';
```

**What it is:** A Snowflake-managed storage location for uploaded files before they are copied into tables.

**Why internal (not external)?** For this project, files are uploaded directly via Snowflake's UI file loader. An external stage (S3, ADLS) would be used if files were landing in cloud storage automatically.

**Why `directory = true`?** Enables subfolder organisation inside the stage — which is used here to create `initial/` and `delta/` partitions.

### File Partitioning Strategy

```
@stage_sch.csv_stg/
├── initial/
│   ├── Location.csv
│   ├── Customer.csv
│   └── ...
└── delta/
    ├── Location.csv
    ├── Customer.csv
    └── ...
```

**Why partition by load type?**
- Separate `initial/` and `delta/` folders let you run different COPY INTO commands for first-time vs. incremental loads.
- Avoids confusion about which files have already been loaded.
- Mirrors real-world patterns where batch files land in dated or typed folders in cloud storage (S3 / ADLS).

### Step-by-Step: Uploading Files to the Snowflake Stage

**Step 1 — Click "+ Files" in the stage browser**

<img width="1350" height="657" alt="Step 1: Click + Files" src="https://github.com/user-attachments/assets/34096b23-f4ea-43a0-9324-8fab35763925" />

**Step 2 — Select the database, schema, and stage; then Browse and select the files**

<img width="1144" height="666" alt="Step 2: Select database schema stage and browse files" src="https://github.com/user-attachments/assets/9fa57606-4c96-406a-8dc3-76da68a99c6f" />

**Step 3 — Specify the path as `initial/` for initial load files**

<img width="666" height="549" alt="Step 3: Specify initial path" src="https://github.com/user-attachments/assets/73a3a538-c960-4651-b989-639b5fb3736f" />

**Step 4 — Click Upload in the lower right corner**

<img width="701" height="615" alt="Step 4: Click Upload" src="https://github.com/user-attachments/assets/64eee676-46ce-4df6-9915-c2da71cee84b" />

**Step 5 — Repeat the same steps for delta load files, using the `delta/` path**

<img width="691" height="655" alt="Step 5: Delta load into delta path" src="https://github.com/user-attachments/assets/abcfa81d-c5b6-4bc4-ac41-3a4bf53e3d36" />

### Initial and Delta Loads for All Entities (Stage View)

**Initial loads for all entities successfully uploaded:**

<img width="1346" height="628" alt="Initial loads for all entities in stage" src="https://github.com/user-attachments/assets/960ee676-8432-4517-83ef-9444276b3c40" />

**Delta loads for all entities successfully uploaded:**

<img width="1351" height="661" alt="Delta loads for all entities in stage" src="https://github.com/user-attachments/assets/81c38f73-9295-42ca-bfd1-2d4bc2ec4d59" />

### Verifying Files in the Stage

```sql
-- List all files in the stage
list @stage_sch.csv_stg;

-- Preview a specific entity's file content before loading
select
    t.$1 :: text as locationid,
    t.$2 :: text as city,
    t.$3 :: text as state,
    t.$4 :: text as zipcode,
    t.$5 :: text as activeflag,
    t.$6 :: text as createddate,
    t.$7 :: text as modifieddate
from @stage_sch.csv_stg/initial/location
(file_format => 'stage_sch.csv_file_format') t;
```

**Why verify before loading?** It's a validation step — you can preview what's coming in and catch format or column misalignment issues before committing a COPY INTO.

<img width="1348" height="621" alt="Verifying files successfully uploaded in stage" src="https://github.com/user-attachments/assets/23f867be-4392-4428-a81e-18612685a470" />

---

## 6. COPY INTO Command & Audit Columns

### How data gets from Stage files to Stage Tables

After verifying, a COPY INTO command loads data from the stage file into the entity table. During this load, four audit columns are added using Snowflake's file metadata functions.

### Audit Columns

```sql
metadata$filename           as _stg_file_name,
metadata$file_last_modified as _stg_file_load_ts,
metadata$file_content_key   as _stg_file_md5,
current_timestamp           as _copy_data_ts
```

**What each one does and why it matters:**

| Column | Source | Purpose |
|---|---|---|
| `_stg_file_name` | `metadata$filename` | Traces which file this record came from. Critical for debugging batch load issues. |
| `_stg_file_load_ts` | `metadata$file_last_modified` | Tells you the freshness of the data — when the source file was last modified. |
| `_stg_file_md5` | `metadata$file_content_key` | MD5 checksum of the file. Used to verify data integrity and detect if a file was tampered with or re-sent. |
| `_copy_data_ts` | `current_timestamp` | Timestamps when the record entered the pipeline. Used for audits and SLA tracking. |

**Overall significance:** These four columns together give you complete **data lineage** — you know where every row came from, when, and from which version of which file. This is what separates a production-grade pipeline from a basic ETL script.

**Location table loaded into Stage schema with audit columns:**

<img width="1347" height="596" alt="Location table loaded with audit columns" src="https://github.com/user-attachments/assets/405358d1-21ac-4ce5-a69b-2bf0a1846091" />

---

## 7. Streams (CDC)

### What is a Stream?

A Snowflake Stream is a **Change Data Capture (CDC)** object that sits on top of a table and records all inserts, updates, and deletes since the last time the stream was consumed.

```sql
-- Stream on stage layer location table
create stream stage_sch.location_stm on table stage_sch.location;

-- Stream on clean layer location table
create stream clean_sch.location_stm on table clean_sch.restaurant_location;
```

### Why Streams at every layer?

| Layer | Stream on | Purpose |
|---|---|---|
| Stage | `stage_sch.location` | Captures new records that came in via COPY INTO, so only those get merged into clean. |
| Clean | `clean_sch.restaurant_location` | Captures records that changed in clean, so only those get merged into consumption. |

**Why not just reload everything each time?** At scale, reprocessing the entire dataset on every run is expensive and slow. Streams give you only the delta — the rows that actually changed. This is what makes the pipeline efficient and cost-effective.

**Significance for this project:** Swiggy processes millions of orders a day. Reloading the full fact table from scratch daily would be impractical. Streams enable incremental loading — the same pattern used in production CDC pipelines at enterprise scale.

### METADATA$ACTION column in Streams

When you query a stream, it includes `METADATA$ACTION` which tells you whether the row was an `INSERT` or `DELETE`. This is what the MERGE operation uses to decide whether to insert a new record or update an existing one.

---

## 8. Normal Forms & Star Schema Design

### Why normalization matters here

Understanding normalisation helps explain the design decisions in both the source data model (OLTP-style clean schema) and the analytical model (star schema in consumption schema).

### First Normal Form (1NF)

**Rule:** Each column must hold atomic (single) values. No repeating groups.

**Example violation:** Storing `"Address1, Address2"` in a single column.

**How this project handles it:** `Customer_Address` is a separate table — one row per address per customer.

### Second Normal Form (2NF)

**Rule:** Every non-key column must depend on the *entire* primary key, not just part of it.

**Example in this project:** `Order_Items` has a composite key of `(Order_ID, Menu_ID)`. Storing `Restaurant_Name` there would violate 2NF because it depends only on `Menu_ID`. So `Restaurant_Name` lives in the `Menu` or `Restaurant` dimension table.

### Third Normal Form (3NF)

**Rule:** No transitive dependencies — non-key columns should depend only on the primary key, not on another non-key column.

**Example:** If `Customer` stored `City_Name` alongside `Address_ID`, that's a transitive dependency (`Customer_ID → Address_ID → City_Name`). So `City_Name` belongs in the `Address` or `Location` table.

### Why not fully normalise in the consumption layer?

Data warehouses deliberately **denormalise** dimension tables (star schema) for query performance. Joining 10 normalised tables for every dashboard query would be slow. Instead:

- Dimension tables are slightly denormalised (all location attributes in one row).
- This trades some data redundancy for dramatically faster query execution.

### Star Schema

```
                    CUSTOMER_DIM
                         |
RESTAURANT_DIM ——— ORDER_ITEM_FACT ——— DATE_DIM
                         |
                    MENU_DIM
                         |
               DELIVERY_AGENT_DIM
```

**Central fact table:** `ORDER_ITEM_FACT` — holds all measurable events (quantity, price, subtotal).

**Dimension tables:** Descriptive context — who, what, where, when.

**Why star schema?** BI tools like Power BI and Tableau are optimised for star schema queries. Simple joins, fast aggregations, intuitive for analysts.

---

## 9. Entity Relationship Diagram (ERD)

The ERD defines how source entities relate to each other. These relationships directly drive how foreign keys are set up in the fact table.

<img width="1303" height="768" alt="ERD" src="https://github.com/user-attachments/assets/a3644d53-6bd5-4b57-8959-40198dd2f363" />

| Relationship | Type | Significance |
|---|---|---|
| Customer → Customer_Address | One-to-many | One customer can have multiple delivery addresses. |
| Restaurant → Menu | One-to-many | Each restaurant has multiple menu items. |
| Orders → Order_Items | One-to-many | One order can contain many items. |
| Delivery_Agent → Delivery | One-to-many | One agent handles many deliveries over time. |
| Customer → Login_Audit | One-to-many | Each login event is tracked per customer. |

**Why this matters for data engineering:** Understanding source relationships tells you which table is the "many" side (usually the fact or staging table) and which is the "one" side (usually the dimension). This prevents data duplication and incorrect join logic.

---

## 10. Surrogate Keys & Hash Keys

### Surrogate Key (Auto-incremented)

```sql
restaurant_location_sk number autoincrement primary key
```

**What it is:** A system-generated integer that uniquely identifies each row in the warehouse, independent of the source system.

**Why use it instead of source IDs?**
- Source IDs can change if the source system changes.
- Surrogate keys are stable, warehouse-controlled, and consistent.
- Required for SCD Type 2 — when a dimension record changes, the old row stays (with its original SK) and a new row is added with a new SK.

### Hash Key

```sql
md5(location_name || city || state) as location_hash_key
```

**What it is:** A fixed-length value generated by an MD5 hashing function applied to one or more business attributes.

**Why use a hash key in addition to surrogate keys?**
- Hash keys are **deterministic** — the same input always produces the same hash. You can regenerate the key without a lookup table.
- Useful for **deduplication** — if the same record arrives twice, it produces the same hash, so you can detect and skip duplicates.
- Efficient for **change detection** — hash the entire row; if the hash changes, the row changed.
- Acts as a natural **unique identifier** in dimension tables, useful for joining when the source IDs aren't reliable.

---

## 11. MERGE Operation

The MERGE statement is the workhorse of this pipeline. It's used at two points for every entity:

- **Merge 1:** Stage Stream → Clean Table
- **Merge 2:** Clean Stream → Consumption Dimension Table

### How MERGE works

```sql
MERGE INTO target_table t
USING source_stream s
ON t.location_id = s.location_id
WHEN MATCHED AND (changes detected) THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT (...);
```

**WHEN MATCHED:** If the record already exists in the target and something changed, update it.

**WHEN NOT MATCHED:** If it's a new record, insert it.

**Why MERGE instead of INSERT or TRUNCATE+RELOAD?**
- INSERT alone can't handle updates.
- TRUNCATE+RELOAD doesn't preserve history and is expensive at scale.
- MERGE handles both inserts and updates in a single, efficient operation — exactly how production CDC pipelines work.

### Data Enrichment During MERGE

The MERGE into the clean schema isn't just a copy — it also enriches data. For example:

```sql
CASE
    WHEN State = 'Delhi' THEN 'DL'
    WHEN State = 'Maharashtra' THEN 'MH'
    WHEN State = 'Uttar Pradesh' THEN 'UP'
END AS state_code
```

```sql
CASE
    WHEN (State = 'Maharashtra' AND City = 'Mumbai') THEN TRUE
    WHEN (State = 'Karnataka' AND City = 'Bangalore') THEN TRUE
END AS capital_city_flag
```

**Why enrich during MERGE?** The clean layer is the right place for business logic. The stage layer holds raw data as-is; the clean layer applies standards, validations, and classifications that make the data usable downstream.

### After Running the First Merge (Stage Stream → Clean Schema)

<img width="1346" height="632" alt="After running the merge statement into clean schema" src="https://github.com/user-attachments/assets/c599290a-abf9-4b6c-a367-334aa1b9022b" />

---

## 12. Slowly Changing Dimensions (SCD)

### What is SCD?

Slowly Changing Dimensions is a data warehousing technique to handle changes in dimension data over time. The key question is: when a customer changes their address, do you overwrite the old one or keep both?

### Types used in this project

| Type | Behaviour | Used For |
|---|---|---|
| SCD Type 0 | No changes tracked | Fields that should never change (e.g., SSN) |
| SCD Type 1 | Overwrite old value | Correcting errors (e.g., wrong phone number) |
| **SCD Type 2** | Add a new row with effective dates | **Primary method in this project** |
| SCD Type 3 | Add a new column for previous value | When you only need to know the last two values |

### SCD Type 2 Implementation

Every dimension table in the consumption schema includes:

```sql
eff_start_dt   timestamp_tz,   -- When this version became active
eff_end_dt     timestamp_tz,   -- When this version was superseded (NULL if current)
current_flag   boolean         -- TRUE = current record, FALSE = historical
```

**How it works in practice:**

1. Customer lives in Bangalore. Row inserted: `current_flag = TRUE`, `eff_start_dt = today`, `eff_end_dt = NULL`.
2. Customer moves to Mumbai. Old row: `current_flag = FALSE`, `eff_end_dt = today`. New row: `current_flag = TRUE`, `eff_start_dt = today`, `eff_end_dt = NULL`.

**Why SCD Type 2 matters for a food delivery platform:**
- You need to know which city a customer was in at the time of an order — not just where they are now.
- Without SCD2, historical revenue by city would be wrong after a customer moves.
- SCD2 is the industry standard for any dimension that changes over time.

### Second Merge: Clean Stream → Consumption Dimension (with SCD2 applied)

<img width="1356" height="642" alt="Second merge from clean stream into consumption dimension with SCD2" src="https://github.com/user-attachments/assets/9e91dbca-6597-4fc7-bb6d-ba004a7e202d" />

### Dimensions with SCD Type 2 in this project

- `RESTAURANT_LOCATION_DIM`
- `CUSTOMER_DIM`
- `CUSTOMER_ADDRESS_DIM`
- `DELIVERY_AGENT_DIM`

---

## 13. Entity-by-Entity Processing Flow

Every entity follows the same three-step pattern:

```
Stage Schema → Clean Schema → Consumption Schema
(COPY INTO)   (MERGE + Enrich) (MERGE + Hash Key + SCD2)
```

### Location Entity

- **Stage:** Load CSV with audit columns. Create stream on stage table.
- **Clean (`restaurant_location`):** Apply correct types, add `is_union_territory`, `capital_city_flag`, `city_tier`, `state_code`. Surrogate key (auto-increment). Create stream on clean table.
- **Consumption (`RESTAURANT_LOCATION_DIM`):** Hash key as PK. SCD2 columns (`eff_start_dt`, `eff_end_dt`, `current_flag`).

#### Loading Delta Data for Location Entity

After the initial load, the stage stream captures only newly added records from the delta file. Re-running the first merge picks those up and loads them into the clean schema.

<img width="1099" height="554" alt="Delta stream records captured for location entity" src="https://github.com/user-attachments/assets/23cb6220-71f0-49cf-9eae-9322b53048c7" />

After the first merge processes delta records into clean, the clean schema's stream captures those changes. Re-running the second merge pushes them into the consumption dimension table.

<img width="1072" height="549" alt="Delta records merged into clean schema for location" src="https://github.com/user-attachments/assets/94817d50-213c-4dec-acc1-d4a6f4a2ce95" />

<img width="1067" height="530" alt="Delta records pushed into consumption dimension from clean stream" src="https://github.com/user-attachments/assets/1158cdcf-c1cf-4200-8012-cb65dc583557" />

### Restaurant Entity

- Same three-step pattern.
- Initial load: 5 records. Delta: 2 additional files processed incrementally.
- Consumption: `RESTAURANT_DIM` with `restaurant_hk` as hash key.

<img width="1242" height="643" alt="Restaurant entity processing across layers" src="https://github.com/user-attachments/assets/df2411c5-9a85-4fac-b699-504f8892873d" />

### Customer Entity

- Same pattern.
- Clean table: `restaurant_customer`. Surrogate key, `active_flag`, `created_ts/modified_ts`.
- Consumption: `CUSTOMER_DIM` with `customer_hk`, full SCD2 support.

<img width="1275" height="613" alt="Customer entity dimension table in consumption schema" src="https://github.com/user-attachments/assets/b671a153-a578-4370-a713-b844b35dc04f" />

### Customer Address Entity

- Same pattern.
- Consumption: `CUSTOMER_ADDRESS_DIM` with SCD2 — critical because delivery addresses change frequently.

### Menu Entity

- Same pattern.
- Consumption: `MENU_DIM` with `menu_dim_hk`, price, category, availability.

### Delivery Agent Entity

- Same pattern with SCD2 — agents can change zones, status, vehicle, etc.
- Consumption: `DELIVERY_AGENT_DIM` with `delivery_agent_hk`.

### Orders, Order Items, Delivery

- These are transactional entities. They don't get separate standalone dimension tables.
- They feed directly into the `ORDER_ITEM_FACT` table at the finest level of granularity.

---

## 14. Fact Table: ORDER_ITEM_FACT

### Granularity Decision

**Granularity = one row per order item** (not one row per order).

**Why order-item level and not order level?**
- One order can contain multiple items (e.g., biryani + raita + cold drink).
- At order level, you'd lose the ability to analyse item-level revenue, menu popularity, etc.
- From order-item granularity, you can always aggregate up to order level, customer level, or restaurant level. You can't go the other direction.
- This is the lowest granularity, so it's the most flexible.

### Why one fact table for three transactional entities?

Orders, Order Items, and Delivery are all linked. Instead of three separate fact tables at different granularities (which would cause duplicates and join complexity), one `ORDER_ITEM_FACT` table at order-item level is used, with delivery and order attributes linked through foreign keys and measures.

### Fact Table Structure

| Column | Purpose |
|---|---|
| `order_item_fact_sk` | Surrogate key (auto-increment). Primary key in DW. |
| `order_item_id` | Natural key from source system. |
| `order_id` | Links to the order. |
| `customer_dim_key` | FK to CUSTOMER_DIM |
| `customer_address_dim_key` | FK to CUSTOMER_ADDRESS_DIM |
| `restaurant_location_dim_key` | FK to RESTAURANT_LOCATION_DIM |
| `menu_dim_key` | FK to MENU_DIM |
| `delivery_agent_dim_key` | FK to DELIVERY_AGENT_DIM |
| `order_date_dim_key` | FK to DATE_DIM |
| `quantity` | Measure |
| `price` | Measure |
| `subtotal` | Measure |
| `delivery_status` | Measure/attribute |
| `estimated_time` | Measure |

**Why surrogate key instead of using `order_item_id` as PK?**
- The surrogate key is system-generated and independent of the source. If the source system ever resets IDs or changes its key strategy, the DW isn't affected.
- SK also handles SCD2 edge cases where the same source ID might appear in multiple versions.

### Merge Operation for Fact Table

- **Update:** If a matching record exists and changes are detected → update the fact row.
- **Insert:** If the record is new → insert into the fact table.
- **Join:** The merge statement connects the fact table with all related dimension tables using foreign keys to resolve the dimension keys during population.

---

## 15. Date Dimension

```sql
-- CTE-based approach: start from MIN(order_date), recursively generate all dates
```

**What it is:** A pre-built lookup table with one row per calendar date and many attributes derived from that date.

**Typical columns in DATE_DIM:**

| Column | Description |
|---|---|
| `date_dim_hk` | Primary key (NUMBER) |
| `calendar_date` | Unique calendar date |
| `day_of_week` / `day_name` | Day-level attributes |
| `week_number` | ISO week number |
| `month_number` / `month_name` | Month-level attributes |
| `quarter` | Q1–Q4 |
| `year` | Calendar year |
| `is_weekend` | Boolean flag |

**Why build a Date Dimension?**
- Without it, every time-based query requires `EXTRACT(MONTH FROM order_date)` directly on the fact table — expensive at scale.
- With DATE_DIM, you join once and filter/group on pre-calculated columns — fast and simple.
- BI tools like Power BI work best when there's a dedicated date table for time intelligence functions.

**Why start from MIN(order_date)?** You only need dates that your data actually contains. Generating from the earliest order avoids unnecessarily large tables.

**Why CTE/recursive approach?** A recursive CTE generates a series of dates without needing an external calendar file. It's entirely self-contained within Snowflake — no external dependency.

---

## 16. Foreign Key Constraints

```sql
ALTER TABLE consumption_sch.order_item_fact
    ADD CONSTRAINT fk_order_item_fact_customer_dim
    FOREIGN KEY (customer_dim_key)
    REFERENCES consumption_sch.customer_dim (customer_hk);

ALTER TABLE consumption_sch.order_item_fact
    ADD CONSTRAINT fk_order_item_fact_customer_address_dim
    FOREIGN KEY (customer_address_dim_key)
    REFERENCES consumption_sch.customer_address_dim (customer_address_hk);

ALTER TABLE consumption_sch.order_item_fact
    ADD CONSTRAINT fk_order_item_fact_restaurant_dim
    FOREIGN KEY (restaurant_dim_key)
    REFERENCES consumption_sch.restaurant_dim (restaurant_hk);

ALTER TABLE consumption_sch.order_item_fact
    ADD CONSTRAINT fk_order_item_fact_restaurant_location_dim
    FOREIGN KEY (restaurant_location_dim_key)
    REFERENCES consumption_sch.restaurant_location_dim (restaurant_location_hk);

ALTER TABLE consumption_sch.order_item_fact
    ADD CONSTRAINT fk_order_item_fact_menu_dim
    FOREIGN KEY (menu_dim_key)
    REFERENCES consumption_sch.menu_dim (menu_dim_hk);

ALTER TABLE consumption_sch.order_item_fact
    ADD CONSTRAINT fk_order_item_fact_delivery_agent_dim
    FOREIGN KEY (delivery_agent_dim_key)
    REFERENCES consumption_sch.delivery_agent_dim (delivery_agent_hk);

ALTER TABLE consumption_sch.order_item_fact
    ADD CONSTRAINT fk_order_item_fact_delivery_date_dim
    FOREIGN KEY (order_date_dim_key)
    REFERENCES consumption_sch.date_dim (date_dim_hk);
```

**What foreign keys do:** They enforce that every `customer_dim_key` in the fact table must exist in `customer_dim`. Same for restaurant, address, menu, delivery agent, and date.

**Important note about Snowflake:** Snowflake supports foreign key constraints as **metadata** (for documentation and tooling purposes) but does **not enforce them at runtime** by default. This is intentional — Snowflake is built for analytical workloads where enforcement would slow down bulk loads.

**Value of declaring them anyway:**
- BI tools (Tableau, Power BI) read FK metadata to auto-suggest joins.
- Data catalog tools use them to map lineage.
- They document the intended relationships for anyone maintaining the warehouse later.
- They signal intent — any data quality issues that violate these relationships are bugs to be fixed upstream.

---

## 17. KPI Views

After building the star schema, three KPI views are created to simplify reporting:

### `vw_yearly_revenue_kpis`

**Purpose:** Total revenue grouped by year.

**Significance:** Long-term trend analysis. Helps leadership compare year-over-year growth and evaluate strategic goals without writing complex SQL every time.

### `vw_monthly_revenue_kpis`

**Purpose:** Total revenue grouped by year and month.

**Significance:** Identifies seasonal patterns — festival spikes, monsoon slowdowns, etc. Monthly reporting is the standard for most business reviews.

### `vw_daily_revenue_kpis`

**Purpose:** Total revenue grouped by calendar date.

**Significance:** Operational monitoring. A sudden drop on a specific day alerts the team to investigate — could be a technical issue, a city-level problem, or a competitor promotion.

### Why views instead of tables?

- Views query the fact table live — always reflecting the latest data.
- No extra storage cost.
- Business users and BI tools can query the view like a simple table without worrying about joins.
- If the underlying fact table structure changes, you update the view definition in one place — not hundreds of reports.

---

## 18. Part 2 – Pipeline Automation

### The Problem with Manual Scripts

Part 1 built the pipeline manually — running COPY INTO, then MERGE, then MERGE again, in sequence, across 35–40 database objects across four schemas.

At Swiggy's scale this would mean hundreds of tables, thousands of columns — impossible to manage with manual worksheet execution.

**The solution:**
- Wrap all DML (data movement logic) inside **Stored Procedures**.
- Orchestrate Stored Procedures with a **Parent SP** per layer.
- Trigger the whole chain with a **Snowflake Task**.

### Separation of DDL and DML

- **DDL** (CREATE TABLE, CREATE STREAM, ALTER TABLE) — run once during setup. These stay in worksheets.
- **DML** (COPY INTO, MERGE) — run repeatedly on every pipeline execution. These go into Stored Procedures.

**Why separate them?** DDL is infrastructure. DML is the pipeline logic. Mixing them means every time you run the pipeline you risk accidentally recreating or dropping objects.

### Summary of Automation Flow

```
Stage Parent SP
    ├── sp_load_location
    ├── sp_load_restaurant
    ├── sp_load_customer
    ├── sp_load_orders
    └── ...

Clean Parent SP
    ├── sp_merge_location
    ├── sp_merge_restaurant
    └── ...

Consumption Parent SP
    ├── sp_dim_location
    ├── sp_dim_customer
    ├── sp_dim_restaurant
    └── ...

Task Tree
    └── Triggers Stage SP → Clean SP → Consumption SP → Fact Load
```

---

## 19. Stored Procedures (SPs)

### What they do

A Stored Procedure in Snowflake wraps SQL logic into a named, callable object.

```sql
CREATE OR REPLACE PROCEDURE stage_sch.sp_load_location()
RETURNS STRING
LANGUAGE SQL
AS
$$
    COPY INTO stage_sch.location
    FROM @stage_sch.csv_stg/delta/location/
    FILE_FORMAT = stage_sch.csv_file_format;
    RETURN 'Location stage load complete';
$$;
```

### SP Structure per layer

**Stage Layer SPs:**
- One SP per entity for the COPY INTO command.
- A Parent SP calls all entity SPs in the correct sequence: `location → restaurant → customer → orders → ...`

**Clean Layer SPs:**
- One SP per entity for the MERGE from stage stream to clean table.
- Parent SP orchestrates them in sequence.

**Consumption Layer SPs:**
- One SP per entity for the MERGE from clean stream to dimension table.
- Parent SP orchestrates all dimension loads, then triggers the fact table load.

### Why sequential execution?

Data has dependencies. You can't load `ORDER_ITEM_FACT` before `CUSTOMER_DIM` is populated — the foreign keys won't resolve. Sequential execution through a Parent SP guarantees the correct order every time.

### Benefits of Stored Procedures

- **Reusability:** Call the same SP for both manual runs and scheduled task execution.
- **Maintainability:** Change the MERGE logic in one SP — it applies everywhere that SP is called.
- **Error handling:** SPs can include TRY/CATCH-style error handling and logging.
- **Security:** Grant EXECUTE on the SP to a role without giving that role direct table access.

---

## 20. Snowflake Tasks & Task Tree

### What is a Snowflake Task?

A Task is a Snowflake object that schedules the execution of a SQL statement or Stored Procedure on a defined schedule (CRON) or when triggered by a parent task completing successfully.

```sql
CREATE OR REPLACE TASK common.task_run_pipeline
    WAREHOUSE = adhoc_wh
    SCHEDULE = 'USING CRON 0 2 * * * Asia/Kolkata'  -- 2 AM IST daily
AS
    CALL common.sp_parent_orchestrator();
```

### Task Tree

A Task Tree is a directed acyclic graph (DAG) of tasks where each task depends on the successful completion of its parent.

```
Root Task (CRON trigger)
    └── Stage Layer Parent SP Task
            └── Clean Layer Parent SP Task
                    └── Consumption Layer Parent SP Task
                            └── Fact Table Load Task
```

**Why a Task Tree over a single Task?**
- If the Stage layer fails, downstream tasks don't execute — preventing corrupt or incomplete data from reaching the consumption layer.
- Each task can have its own warehouse size.
- Provides visibility — you can monitor each task's success/failure independently in Snowflake's Task History view.

### Challenges with Stored Procedures Alone (without Tasks)

- SPs still require someone to manually call the parent SP.
- No scheduling, no retry logic, no dependency management.
- Tasks solve all three.

### Summary of Automation Stack

| Component | Role |
|---|---|
| Stored Procedures | Wrap the DML logic for each entity per layer |
| Parent Stored Procedures | Orchestrate entity SPs in correct sequence per layer |
| Tasks | Schedule and trigger the parent SPs |
| Task Tree | Ensures correct layer-by-layer execution with dependency management |

---

## 21. Streamlit Dashboard

**What it is:** A Python-based web app framework natively integrated into Snowflake (Streamlit in Snowflake / SiS). It allows you to build interactive dashboards that query Snowflake directly — no external hosting needed.

**What the dashboard shows:**
- Total revenue KPIs
- Revenue trends (daily, monthly, annual)
- Top restaurants by revenue
- Revenue by customer segment
- Delivery performance metrics
- Geographic revenue insights

**Why Streamlit over Power BI or Tableau for this project?**
- Runs entirely inside Snowflake — no external tool license, no data export.
- Python-based — data engineers are comfortable building it.
- Connects directly to the KPI views and fact/dimension tables.
- Demonstrates end-to-end ownership: the same engineer who built the pipeline can build the dashboard.

**Scalability validation:** One of the project's goals was to verify that the Streamlit dashboard continues to perform correctly when data volume scales from sample records to thousands (and eventually millions) of rows. The KPI views abstract the complexity, so the dashboard queries stay simple regardless of data volume.

---

## 22. Initial vs. Delta Load Strategy

### Initial Load

- First-time ingestion of all historical data for an entity.
- Files placed in `@stage_sch.csv_stg/initial/{entity}/` folder.
- Typically a larger batch — all existing records.
- Example: Restaurant entity initial load = 5 records.

### Delta Load

- Incremental changes after the initial load — new records, updated records.
- Files placed in `@stage_sch.csv_stg/delta/{entity}/` folder.
- Only processes what's new or changed — streams ensure this.
- Example: Restaurant entity delta = 2 new files processed incrementally.

### Why this two-phase approach?

This mirrors real-world production patterns:
1. The business team provides sample/historical data to bootstrap the pipeline.
2. After validation, the pipeline runs on real incremental data.
3. Streams ensure only the delta is processed — not the entire history on every run.

### Processing Frequencies

| Mode | Frequency | Use Case |
|---|---|---|
| Batch Processing | Once daily | Standard EDW, business reporting |
| Micro-Batching | Every hour / 15 minutes | Near-real-time dashboards |
| Real-Time Streaming | Continuous | Fraud detection, live dashboards |

This project uses **batch processing** with Task scheduling, which is appropriate for daily business reporting. The architecture can be shifted to micro-batching by simply adjusting the Task CRON schedule — no pipeline logic changes needed.

---

## Quick Reference: Snowflake Objects Used

| Object | Purpose |
|---|---|
| Warehouse | Compute layer for query execution |
| Database | Top-level logical container |
| Schema | Organises objects by layer/purpose |
| Internal Stage | Temporary storage for uploaded CSV files |
| File Format | Defines how CSV files are parsed |
| Table | Stores structured data at each layer |
| Stream | CDC — tracks inserts/updates/deletes on a table |
| Stored Procedure | Wraps DML logic into a callable, reusable object |
| Task | Schedules SP execution; supports dependency-based Task Trees |
| View | Pre-built KPI queries over the fact/dimension tables |
| Tag | Metadata labels for data governance and classification |
| Masking Policy | Controls visibility of sensitive columns based on role |
| Foreign Key Constraint | Documents star schema relationships (informational in Snowflake) |

---

*This document covers every component of the Swiggy-like Snowflake data pipeline project — from sandbox setup to full pipeline automation.*

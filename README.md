# Swiggy End-to-End Data Engineering Project

## Why Swiggy?
To truly learn **end-to-end data engineering**, we need to solve a real-life business problem. Since we all use food delivery apps, it is easier to connect the dots and understand the data flow.

Food aggregator apps (like Swiggy) typically manage these entities:
- Restaurant
- Catalog/Menu
- Promotion
- Order
- Location
- Payment
- Delivery
- Rating

### Business Goals
The leadership team wants insights such as:
- Total revenue
- Average revenue per item
- Average revenue per order
- Top-performing restaurant
- Revenue trends over time
- Revenue by customer segment
- Revenue by restaurant & location
- Delivery performance
- Geographic revenue insights

This project uses **Swiggy’s food ordering subprocess** as an example to design and implement an end-to-end Snowflake data pipeline.

---

##  Project Data
- Synthetic data generated in **CSV format** for different entities.
- Files uploaded using **Snowflake’s File Upload feature**.
- Entities considered: `Location`, `Restaurant`, `Customer`, `Customer-adress`, `Menu`, `Delivery_agent`, `Orders`, `Order-items`, `Delivery`,  `Login-audit` 

---

## End-to-End Data Flow Architecture

CSV Files → Snowflake Stage → Transformation Layer → Fact & Dimension Tables → Streamlit Dashboard


### Entities and Flow
1. **Location Entity** → Define target serviceable locations.  
2. **Restaurant Entity** → Onboard restaurants for each location.  
3. **Menu Entity** → Upload restaurant catalogs.  
4. **Login Entity** → Handle customer login via email/phone.  
5. **Customer Entity** → Maintain customer profiles.  
6. **Address Entity** → Store delivery addresses.  
7. **Order Entity** → Track customer orders.  
8. **Delivery Agent Entity** → Assign agents to orders.  
9. **Delivery Entity** → Record final delivery details.  

At a high level, **9–10 entities (tables)** are required to support the business process.

---

## Breakdown of Sections

### 1. Implement E2E Data Flow
- Create **Database and Schema** in Snowflake.
- Load CSV files into staging tables.
- Process **Location Entity**.
- Handle **Bad Record Processing**.
- Process other **Dimension Entities**.
- Process **Fact Entities** (orders, revenue).
- Build a **Streamlit App** to visualize KPIs.

---

### 2. Automate E2E Data Pipeline Using Stored Procedure
- **Why Automate?** → To remove manual intervention.  
- Create **Stored Procedures** for transformations.  
- Link multiple procedures for sequential processing.  
- Trigger stored procedures using **Snowflake Tasks**.

---

### 3. Automate E2E Data Pipeline Using Task Tree
- **Challenges** with only stored procedure–based approach.  
- Use **Task and Task Tree** for orchestration.  
- Visualize and monitor the overall pipeline in Snowflake.  

---

### 4. Process Large Data Volume
- Ingest **large data volumes** into Snowflake.  
- Validate **processing speed & performance**.  
- Ensure **Streamlit dashboard** scales with bigger datasets.  

---

## End-to-End Data Flow Diagram

<img width="1961" height="980" alt="Food-Aggregator-E2E-Data-Flow" src="https://github.com/user-attachments/assets/dfe69eb7-7db6-48ef-b251-26878f07b6e7" />

---

## ER Diagram Analysis

Understanding entity relationships is crucial in any **data engineering project** because they guide how we design **data warehouse tables across different layers**.

### Entity Relationships

- **Customer → Customer_Address**  
  - One-to-many relationship  
  - A single customer can have one or multiple addresses.  

- **Restaurant → Menu**  
  - One-to-many relationship  
  - Each restaurant can have multiple menu items.  

- **Orders → Order_Items**  
  - One-to-many relationship  
  - For each order placed, the `Orders` table gets one record, while the `Order_Items` table may hold one or more records depending on the number of items in the order.  

- **Delivery_Agent → Delivery**  
  - One-to-many relationship  
  - Each delivery agent can be associated with multiple deliveries.  

- **Customer → Login_Audit**  
  - One-to-many relationship  
  - Each customer may log in multiple times, creating multiple records in the `Login_Audit` table.  

---

## ER Diagram

<img width="1303" height="768" alt="ERD" src="https://github.com/user-attachments/assets/a3644d53-6bd5-4b57-8959-40198dd2f363" />


---

These relationships form the backbone for building **fact and dimension tables** in the data warehouse.

## Source System CSV File Analysis

### Data Sources
The source data for this project includes the following CSV files representing different entities:

- `Location`
- `Restaurant`
- `Customer`
- `Customer_Address`
- `Menu`
- `Delivery_Agent`
- `Orders`
- `Order_Items`
- `Delivery`
- `Login_Audit`

### Initial Load & Delta Load
In this project, I’ll demonstrate how to:
1. Handle **initial loads** – when we get the first set of data.
2. Process **delta loads** – subsequent changes or additions, which is a common pattern in batch processing systems.

For each source, I’ll begin with a handful of records representing both initial and delta loads to build the entire pipeline. Once the solution is validated with sample data, I’ll scale it to thousands of records. This approach mirrors real-world practices where the business team provides sample data to build the pipeline, followed by validation with larger datasets.

---

##  Understanding Overall End-to-End Data Architecture

### Stage Layer
- CSV files from different entities are uploaded to Snowflake using the **file loader** feature.
- These files are stored in a **stage location**.
- A **COPY command** is used to load data from the stage into Snowflake tables.

### Clean Layer
- After loading, the data is moved into the **clean layer**, a schema where:
  - Basic data cleansing
  - Data validation
  - Initial transformations are performed.

### Consumption Layer
- From the clean layer, data is moved to the **consumption layer**, where:
  - Fact and dimension tables are built.
  - The process starts from the `Location` entity and progresses to the `Order_Items` fact table.

---

###  Objective of the Architecture
The main goal of this process is to transform raw source data into a structured data platform that supports analytical queries.

- Source data is ingested and goes through a set of transformations.
- The final data model follows a **star schema dimensional modeling** approach:
  - Master data like `Customer`, `Customer_Address`, `Restaurant`, `Menu`, `Location`, `Delivery_Agent` becomes **dimension tables**.
  - Transactional data like `Order` and `Order_Items` becomes the **fact table**.
- The **clean schema** represents OLTP-style data, while the **consumption schema** provides star-schema structured data for reporting and analysis.

<img width="1209" height="768" alt="Objective ERD" src="https://github.com/user-attachments/assets/f44b1fdd-c479-447a-885a-a8c8ba0fcd4a" />


---

##  Points to Remember

- **Dimensional Modeling / Star Schema**
  - Often referred to as the **star schema** in data warehousing.
  - It adheres to principles like **Second Normal Form (2NF)** in database design.
  - Consists of:
    - A central **fact table** (transactional data)
    - Surrounded by multiple **dimension tables** (master data).
  - Fact tables store measurable business events, while dimension tables store descriptive attributes.

---

###  Notes on Normal Forms (For deeper understanding)

#### First Normal Form (1NF)
- Each column must contain **atomic (indivisible) values**.
- There should be **no repeating groups** or arrays within a column.
- Every row must be **unique** and identifiable by a primary key.

**Example:**  
In the `Customer_Address` table, storing multiple addresses in a single column like this would **violate 1NF**:

| Customer_ID | Address |
|-------------|-----------------------------|
| 1           | "Address1, Address2"       |

Instead, it should be split into separate rows:

| Customer_ID | Address      |
|-------------|--------------|
| 1           | Address1     |
| 1           | Address2     |

---

#### Second Normal Form (2NF)
- Builds on 1NF.
- Ensures that **every non-key column is fully dependent on the entire primary key**, not just part of it.
- Avoids **partial dependencies** where a column depends only on a part of a composite key.

**Example:**  
In the `Order_Items` table, assume the primary key is `(Order_ID, Menu_ID)`.  
If you also store `Restaurant_Name`, which depends only on `Menu_ID`, that’s a violation of 2NF because it’s dependent on only part of the key.

Correct approach: Move `Restaurant_Name` to the `Menu` or `Restaurant` dimension table, where it fully depends on `Menu_ID`.

---

#### Third Normal Form (3NF)
- Builds on 2NF.
- Removes **transitive dependencies**, where non-key columns depend indirectly on the primary key through another column.
- Ensures that **non-key attributes depend only on the primary key**.

**Example:**  
If the `Customer` table has columns like:

| Customer_ID | Address_ID | City_Name  |
|-------------|------------|------------|
| 1           | 101        | New York   |

Here, `City_Name` depends on `Address_ID`, which depends on `Customer_ID`. This is a transitive dependency.

Correct approach:
- Move `City_Name` to an `Address` or `Location` table where it depends directly on `Address_ID`.

---

### Why It Matters
- Normalization helps to **reduce redundancy**, **improve data integrity**, and **streamline updates**.
- In data warehouses, we balance normalization with performance and usability - often using **star schemas** where dimension tables are slightly denormalized for query efficiency.

---
## Setting up the Snowflake Sandbox Environment

In this step, we create a **database sandbox** and all necessary **schemas, file formats, internal stages, tags, and masking policies** to support the end-to-end data pipeline.

---

### 1️⃣ Switch to SYSADMIN Role (SYSADMIN role is used because it has privileges to create warehouses, databases, schemas, and other objects.)
```sql
use role sysadmin;
```
### 2️⃣ Create Warehouse
```sql
create warehouse if not exists adhoc_wh
     comment = 'This is the adhoc-wh'
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
Warehouse: Compute layer in Snowflake. Needed to execute queries and run ETL processes.

auto_resume / auto_suspend ensures cost optimization.

x-small is sufficient for development/sandbox use.

### 3️⃣ Create Sandbox Database and Schemas
```sql
create database if not exists sandbox;
use database sandbox;

create schema if not exists stage_sch;
create schema if not exists clean_sch;
create schema if not exists consumption_sch;
create schema if not exists common;
```
Database: Logical container for all project objects.

Schemas: Organize objects by purpose:

`stage_sch` → raw data staging.

`clean_sch` → cleaned and processed data.

`consumption_sch` → fact and dimension tables for analytics.

`common` → shared objects like tags, masking policies, reusable functions.

### 4️⃣ Create File Format for CSV
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
File format defines how Snowflake reads files in stages.

Important options:

`skip_header = 1` → ignores header row.

`field_optionally_enclosed_by = '\042'` → handles quotes around fields.

`null_if = ('\\N')` → interprets \N as NULL.

### 5️⃣ Create Internal Stage
```sql
create stage stage_sch.csv_stg
    directory = ( enable = true )
    comment = 'this is the snowflake internal stage';
```
Stage: Storage location in Snowflake for your CSV files.

Can be used with COPY INTO commands to load data into tables.

directory = true allows organizing files in subfolders.

### 6️⃣ Create Tag Objects
```sql
create or replace tag 
    common.pii_policy_tag 
    allowed_values = ('PII','PRICE','SENSITIVE','EMAIL')
    comment = 'This is PII policy tag object';
```
Tag: Metadata label that can be attached to tables or columns.

Used for data governance, classification, and compliance.

Example: Mark sensitive columns like `email` or `phone` as `PII`.

### 7️⃣ Create Masking Policies
```sql
create or replace masking policy 
    common.pii_masking_policy as (pii_text string)
    returns string -> to_varchar('** PII **');

create or replace masking policy 
    common.email_masking_policy as (email_text string)
    returns string -> to_varchar('** EMAIL **');

create or replace masking policy 
    common.phone_masking_policy as (phone string)
    returns string -> to_varchar('** Phone **');
```
**Masking Policy** controls how sensitive data is displayed.

- It is **applied to columns** to hide actual values for non-privileged users.
- Helps ensure that sensitive information is not exposed in queries.

###  Examples
- `email_masking_policy` → hides customer email addresses.
- `phone_masking_policy` → hides customer phone numbers.

###  Purpose
- Ensures **data privacy** and compliance with regulations like **GDPR**.
- Works alongside **tags** to help classify and govern sensitive data.

---

## ✅ Summary of Setup

- **Warehouse** → Provides compute resources for running queries and ETL processes.
- **Database + Schemas** → Organizes raw, cleaned, and analytical data into logical containers.
- **File Format** → Defines how CSV files are parsed and interpreted by Snowflake.
- **Internal Stage** → Serves as temporary storage for CSV files before loading them into tables.
- **Tags** → Used to classify sensitive columns for governance and compliance.
- **Masking Policies** → Protect sensitive data by masking it for non-authorized users during queries.

<img width="1184" height="615" alt="image" src="https://github.com/user-attachments/assets/c3b185d6-b078-40f4-864b-79215687bad7" />

##  Uploading Files with Partitions: Initial Load & Delta Load

After uploading the files into the stage location using Snowflake’s file upload feature, I will organize the files based on the type of data load.

###  Types of Data Files
1. **Initial Load Files**
   - Contains the first set of data for each entity.
   - Represents a full snapshot of data at the start.

2. **Delta Load Files**
   - Contains incremental changes or new records after the initial load.
   - Represents updates, inserts, or modifications over time.

---

###  Partitioning in Stage Location
To distinguish between initial and delta loads, I will create **two partitions** inside the internal stage:

- `initial/` → For storing initial load files.
- `delta/` → For storing incremental or delta files.

This structure helps in:
- Easily managing and identifying the type of data.
- Running separate `COPY INTO` commands as per load type.
- Simulating real-world batch processing pipelines where data comes in chunks.

###  Example Stage Structure
@stage_sch.csv_stg/

├── initial/

│ ├── Location.csv

│ ├── Customer.csv

│ └── ...

└── delta/

├── Location.csv

├── Customer.csv

└── ...

###  Notes
- Using partitions mimics how files are organized in cloud storage like AWS S3 or Azure Blob Storage.
- It provides a scalable and structured way to manage multiple data versions.
- This setup allows the pipeline to process full and incremental datasets independently while maintaining a clean workflow.


### Reference Images of data loading into snowflake stage
## 1 <img width="1350" height="657" alt="image" src="https://github.com/user-attachments/assets/34096b23-f4ea-43a0-9324-8fab35763925" />
After clicking the "+ Files"

## 2 <img width="1144" height="666" alt="image" src="https://github.com/user-attachments/assets/9fa57606-4c96-406a-8dc3-76da68a99c6f" />
- Check "Select database, schema abd stage"  
- Goto "Browse" and upload the files"

## 3 <img width="666" height="549" alt="image" src="https://github.com/user-attachments/assets/73a3a538-c960-4651-b989-639b5fb3736f" />
- Specify the path "initial", where the initial load files will be loaded.

## 4 <img width="701" height="615" alt="image" src="https://github.com/user-attachments/assets/64eee676-46ce-4df6-9915-c2da71cee84b" />
- Click "Upload" in the right side lower corner

## 5 <img width="691" height="655" alt="image" src="https://github.com/user-attachments/assets/abcfa81d-c5b6-4bc4-ac41-3a4bf53e3d36" />
- Same steps for Delta loads, into the "Delta" path



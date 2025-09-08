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

<img width="1366" height="768" alt="ERD" src="https://github.com/user-attachments/assets/02c9f01e-9456-4947-9283-206f775623bd" />

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

- **First Normal Form (1NF)**  
  - Ensures that each column contains atomic values and each row is unique.
  - Eliminates repeating groups.

- **Second Normal Form (2NF)**  
  - Builds on 1NF by ensuring that every non-key attribute is fully functionally dependent on the primary key.
  - Avoids partial dependencies.

- **Third Normal Form (3NF)**  
  - Builds on 2NF by removing transitive dependencies.
  - Ensures that non-key attributes depend only on the primary key.

These normal forms help structure data efficiently, reduce redundancy, and improve data integrity. In dimensional modeling, 2NF is commonly used while balancing performance and simplicity.

---


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

## 🗂 ER Diagram Analysis

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

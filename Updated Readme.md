# 🍽️ Swiggy End-to-End Data Engineering Project

## Overview
This project demonstrates a **production-style, end-to-end data engineering pipeline** built entirely using **Snowflake SQL**.  
It models a real-world **food delivery platform (Swiggy-like)** use case, covering data ingestion, transformation, governance, automation, and analytics.

The solution follows a **layered data warehouse architecture** with support for:
- Initial & delta loads
- Change Data Capture (CDC)
- Slowly Changing Dimensions (SCD Type 2)
- Fact table modeling
- Automated pipelines using Stored Procedures and Tasks

---

## Why This Project?
Food delivery platforms are familiar, complex, and data-heavy.  
Using this domain makes it easier to understand real business problems such as:

- Revenue analysis
- Customer behavior tracking
- Restaurant performance
- Delivery efficiency
- Location-based insights

This project focuses on **how data flows from raw CSV files to analytics-ready tables**.

---

## Business Objectives
The leadership team expects insights like:

- Total and average revenue
- Revenue trends (daily, monthly, yearly)
- Top-performing restaurants
- Revenue by location and customer segment
- Delivery performance metrics
- Geographic revenue distribution

---

## Data Sources
Synthetic CSV data is generated for the following entities:

- Location  
- Restaurant  
- Customer  
- Customer Address  
- Menu  
- Delivery Agent  
- Orders  
- Order Items  
- Delivery  
- Login Audit  

Files are uploaded into **Snowflake Internal Stages** and processed using SQL.

---

## High-Level Architecture

```text
CSV Files
↓
Snowflake Internal Stage
↓
Stage Schema (Raw + Audit + Streams)
↓
Clean Schema (Validated & Enriched)
↓
Consumption Schema (Star Schema)
↓
KPI Views / Dashboards
```
## End-to-End Data Flow Diagram

<img width="1961" height="980" alt="Food-Aggregator-E2E-Data-Flow" src="https://github.com/user-attachments/assets/dfe69eb7-7db6-48ef-b251-26878f07b6e7" />
---

## Core Design Principles
- SQL-only implementation
- Layered architecture (Stage → Clean → Consumption)
- Incremental processing using Streams
- SCD Type 2 for historical tracking
- Star schema for analytics
- Data governance with Tags & Masking Policies
- Automation using Stored Procedures and Tasks

---

## Layered Data Architecture

### Stage Schema
- Raw CSV ingestion using `COPY INTO`
- Audit columns added during load
- Streams enabled for CDC
- Initial and delta loads handled separately

### Clean Schema
- Data validation and standardization
- Proper data types applied
- Surrogate keys introduced
- Merge-based incremental updates
- Enrichment using business rules

### Consumption Schema
- Star schema design
- Dimension tables with SCD Type 2
- Central fact table at order-item granularity
- Foreign key enforcement
- Analytics-ready datasets

---
## ER Diagram

<img width="1303" height="768" alt="ERD" src="https://github.com/user-attachments/assets/a3644d53-6bd5-4b57-8959-40198dd2f363" />

## Data Modeling

### Dimension Tables
- CUSTOMER_DIM
- CUSTOMER_ADDRESS_DIM (SCD2)
- RESTAURANT_DIM (SCD2)
- RESTAURANT_LOCATION_DIM (SCD2)
- MENU_DIM
- DELIVERY_AGENT_DIM (SCD2)
- DATE_DIM

### Fact Table
- ORDER_ITEM_FACT  
  - Granularity: **Order Item**
  - Contains revenue, quantity, price, delivery metrics
  - Linked to all dimensions via surrogate/hash keys

---

## Slowly Changing Dimensions (SCD)
This project uses **SCD Type 2** to track historical changes.

Each SCD-enabled dimension includes:
- `eff_start_dt`
- `eff_end_dt`
- `current_flag`

This allows full historical reporting without losing previous values.

---

## Initial Load & Delta Load Strategy

### Stage Partitioning
```text
@stage_sch.csv_stg/
├── initial/
│ ├── location/
│ ├── restaurant/
│ └── ...
└── delta/
├── location/
├── restaurant/
└── ...
```

### Processing Logic
- Initial load: Full snapshot
- Delta load: Incremental inserts/updates
- Streams capture only changed records
- Merge statements apply changes downstream

---

## Data Governance & Security
Implemented using Snowflake-native features:

- **Tags** for classification (PII, Email, Sensitive, Price)
- **Masking Policies** for:
  - Email
  - Phone number
  - PII attributes
- Applied at column level
- Ensures compliance and controlled data access

---

## Automation Strategy

### Why Automation?
Manual SQL execution does not scale.  
This project simulates a production-grade orchestration approach.

### Implementation
- DML logic wrapped inside Stored Procedures
- Separate procedures per entity and layer
- Parent Stored Procedures orchestrate execution order
- Snowflake Tasks trigger pipelines automatically

### Pipeline Flow
```text
Stage SPs
↓
Clean SPs
↓
Consumption SPs
↓
Fact Table
```

---

## KPI & Analytics Views

To simplify analytics, KPI views are created on top of the fact table.

### KPI Views
- `vw_daily_revenue_kpis`
- `vw_monthly_revenue_kpis`
- `vw_yearly_revenue_kpis`

These views:
- Abstract complex joins
- Improve query performance
- Enable direct BI tool consumption

---

## Key Highlights
- End-to-end Snowflake data engineering
- SQL-only implementation
- Realistic enterprise-style architecture
- CDC using Streams
- SCD Type 2 dimensions
- Star schema modeling
- Secure and governed data
- Fully automated pipelines

---

## Tech Stack
- Snowflake (SQL)
- Snowflake Streams & Tasks
- Snowflake Stored Procedures
- Internal Stages
- CSV-based batch ingestion

---

## Final Notes
This project closely mirrors **real-world enterprise data warehouse implementations** used in large-scale platforms like Swiggy.  
It demonstrates not just SQL skills, but also **data modeling, governance, automation, and analytical thinking**.

---

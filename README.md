# NYC TLC Yellow Taxi Lakehouse Pipeline (Azure ADF + Databricks + Delta + Power BI)

End-to-end data engineering project that ingests NYC TLC Yellow Taxi monthly data, performs automated incremental ingestion, standardizes and validates records into a Silver Delta layer (with rejects auditing), generates BI-ready Gold aggregates, and visualizes KPIs in Power BI.

---

## Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Data Source](#data-source)
- [Data Lake Layout](#data-lake-layout)
- [Pipeline Stages](#pipeline-stages)
  - [1) Ingestion (ADF)](#1-ingestion-adf)
  - [2) Silver ETL (Databricks)](#2-silver-etl-databricks)
  - [3) Gold Aggregates (Databricks)](#3-gold-aggregates-databricks)
  - [4) Power BI Dashboard](#4-power-bi-dashboard)
- [Data Quality Rules](#data-quality-rules)
- [How to Run](#how-to-run)
- [Troubleshooting Notes](#troubleshooting-notes)
- [Next Improvements](#next-improvements)

---

## Project Overview

### Goals
- Build a practical, job-ready data engineering pipeline covering:
  - **Data ingestion** from a real-world public source
  - **Automated incremental loading** using a watermark/control table
  - **ETL / standardization** into Delta Lake
  - **Data quality validation** with accepted/rejected routing
  - **BI-ready outputs** and Power BI reporting

### Outcomes
- Ingested monthly Yellow Taxi trip Parquet files from **2016-01 to latest**
- Built a consistent **Silver Delta** table partitioned by year/month
- Captured invalid records into a **rejects** table with reason codes and source tracking
- Produced **Gold Parquet** aggregates suitable for Power BI
- Built a Power BI dashboard with revenue/trips/tips metrics

---

## Architecture

**Source (NYC TLC CloudFront)**  
→ **Azure Data Factory** (parameterized pipeline + incremental loop)  
→ **ADLS Gen2 / Blob Storage** (raw landing)  
→ **Databricks (PySpark)** (Silver Delta + rejects + Gold marts)  
→ **Power BI Desktop** (connect to Gold files in storage)

Watermark/state is stored in **Azure SQL Database**.

---

## Tech Stack
- **Azure Data Factory (ADF)**: orchestration + incremental ingestion
- **Azure Storage (ADLS Gen2 / Blob Storage)**: raw/silver/gold zones
- **Azure Databricks**: PySpark ETL + Delta Lake
- **Delta Lake**: Silver tables + schema drift handling
- **Azure SQL Database**: control/watermark table
- **Power BI Desktop**: dashboard/reporting

---

## Data Source
- NYC TLC monthly Yellow Taxi trip records (public)
- Download endpoint used via CloudFront distribution:
  - Base URL: `https://d37ci6vzurychx.cloudfront.net/`
  - Trip data path pattern: `trip-data/yellow_tripdata_YYYY-MM.parquet`
  - Lookup table: `misc/taxi_zone_lookup.csv`

---

## Data Lake Layout

Container: `rawdata`
rawdata/
yellowtaxi/ # raw landing
yellow_tripdata_YYYY-MM.parquet
control/ # control artifacts (optional)
silver/
tlc_yellow_trips_v2/ # Silver Delta (accepted)
_delta_log/
pickup_year=YYYY/
pickup_month=M/
part-.parquet
tlc_yellow_trips_rejected_v2/ # Silver Delta (rejected)
_delta_log/
part-.parquet
gold/
yellow_monthly/ # BI-ready Parquet aggregate
part-*.parquet


---

## Pipeline Stages

### 1) Ingestion (ADF)
**Purpose:** Automatically download monthly files and store them in the raw zone.

**Key features**
- Linked Services:
  - HTTP source (CloudFront)
  - ADLS Gen2 sink (storage)
  - Azure SQL DB (watermark)
- Parameterized dataset pathing (relative URL)
- Incremental ingestion with **watermark (last_success_ym)**:
  - Read watermark from SQL
  - Compute next month (handles year rollover)
  - Check existence
  - Copy file to lake
  - Update watermark on success

**Notable issues solved**
- 403 errors caused by incorrect dynamic URL expression formatting
- Base URL required trailing `/`
- Correct use of dataset parameters to avoid missing parameter errors
- Loop stop conditions and boolean handling in ADF expressions

---

### 2) Silver ETL (Databricks)
**Purpose:** Standardize schema across years, apply quality checks, write to Delta.

**Silver outputs**
- `silver/tlc_yellow_trips_v2` (accepted)
- `silver/tlc_yellow_trips_rejected_v2` (rejected with reasons)

**Highlights**
- Schema normalization:
  - Add missing columns if absent in older files
  - Safe casting to handle type drift across monthly Parquets
- Data quality validation:
  - Timestamp validity + range checks
  - Dropoff after pickup constraint
  - Money sanity checks with required vs optional fee fields
- Rejection handling:
  - `reject_reason` (`bad_timestamp` / `bad_money`)
  - `source_file` for traceability
- Partitioning:
  - Partition by `pickup_year`, `pickup_month` for pruning and performance

---

### 3) Gold Aggregates (Databricks)
**Purpose:** Produce compact BI-ready datasets.

Example Gold mart: `gold/yellow_monthly`
- monthly trips
- total revenue
- total tips
- avg distance
- avg duration

Gold is written as **Parquet** for easiest Power BI connectivity.

---

### 4) Power BI Dashboard
**Purpose:** Visualize key business KPIs from Gold marts.

**Connection**
- Power BI Desktop → Azure Blob Storage / ADLS Gen2 connector
- Use Power Query to filter to `rawdata/gold/yellow_monthly/` and load Parquet output

**Visuals**
- Revenue trend
- Trips trend
- Tips and KPI cards
- Year/month slicers

---

## Data Quality Rules

### Timestamp Rules
Reject if:
- pickup or dropoff timestamp is null
- timestamps out of valid range (>= 2016-01-01 and <= current time + buffer)
- dropoff < pickup

### Money Rules (robust across historical schema drift)
- Required: `fare_amount`, `total_amount` must be **not null** and **>= 0**
- Optional (allowed null historically): fees like `congestion_surcharge`, `airport_fee`, etc.
  - must be `NULL OR >= 0`

---

## How to Run

### Prerequisites
- Azure subscription with:
  - Storage Account (ADLS Gen2)
  - Data Factory
  - Databricks workspace + cluster
  - Azure SQL Database (for watermark control)
- IAM/RBAC:
  - ADF Managed Identity granted `Storage Blob Data Contributor` to storage/container
  - Power BI identity granted `Storage Blob Data Reader` (if using Entra ID auth)

### Run Steps
1. **ADF**: Trigger ingestion pipeline (downloads new months)
2. **Databricks**: Run Silver ETL notebook (build/update Delta tables)
3. **Databricks**: Run Gold aggregation notebook (write Parquet marts)
4. **Power BI**: Refresh dataset and visuals

---

## Troubleshooting Notes
- **Parquet schema drift** across TLC history is expected (type mismatches).
  - Normalize types before union/write.
- **Decimal overflow** can occur if casting too early.
  - Validate money columns as `double`, filter bad values, then cast accepted rows to decimal.
- **Partial Silver tables** can happen if overwrite+append logic fails mid-run.
  - Use fresh rebuild path (`*_v2`) or delete target folder before a full rebuild.

---

## Next Improvements
- Add dimension tables:
  - `dim_date`, `dim_zone` (taxi_zone_lookup)
- Build zone-level Gold marts:
  - monthly revenue/trips by pickup zone/borough
- Add incremental Silver processing:
  - process only newly ingested months based on watermark
- Implement formal serving layer:
  - Databricks SQL Warehouse or Synapse/Fabric SQL endpoint for enterprise BI access
- Add monitoring:
  - ADF alerts + logging into a dedicated monitoring table

---

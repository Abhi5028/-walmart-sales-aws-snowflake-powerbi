# Walmart Sales Analytics Pipeline — AWS S3 → Snowflake → Power BI

An end-to-end data pipeline built to practice real-world cloud data engineering: cleaning raw retail sales data, staging it in AWS S3, loading it into Snowflake, and visualizing it in Power BI.


## Overview

This project takes the Walmart weekly sales dataset (store-level sales, holiday flags, and economic indicators like fuel price, CPI, and unemployment) through a full pipeline:

**Excel/Google Sheets (cleaning) → AWS S3 (staging) → Snowflake (warehouse) → Power BI (visualization)**

The goal was to build a pipeline that mirrors how a real analytics team would move data from a raw source to a business-ready dashboard, while working entirely within free-tier / trial-credit constraints.

## Architecture

```
Raw CSV
   │
   ▼
Excel / Google Sheets — cleaning & feature engineering
   │  (added Year, Month, WeekOfYear columns)
   ▼
AWS S3 (abhiroop-walmart-pipeline/processed/)
   │
   ▼
Snowflake External Stage → COPY INTO → walmart_sales table
   │
   ▼
Power BI — dashboard with KPIs, trends, and store-level breakdowns
```

## Tech Stack

- **Excel / Google Sheets** — data cleaning and feature engineering
- **AWS S3** — cloud storage / staging layer
- **AWS IAM** — access control for Snowflake-to-S3 connectivity
- **Snowflake** — cloud data warehouse (external stage, `COPY INTO`, SQL transformations)
- **Power BI** — dashboarding and visualization

## Dataset

Walmart weekly store sales data, including:
- `Store`, `Sale_Date`, `Year`, `Month`, `WeekOfYear`
- `Weekly_Sales`, `Holiday_Flag`
- `Temperature`, `Fuel_Price`, `CPI`, `Unemployment`

6,435 rows across multiple stores and a multi-year date range.

## Pipeline Steps

### 1. Data Cleaning (Excel / Google Sheets)
Cleaned the raw dataset manually — handled nulls, standardized data types, and engineered `Year`, `Month`, and `WeekOfYear` columns from the date field to support time-based analysis later in Power BI.

### 2. AWS S3 Staging
Created an S3 bucket with a `raw/` and `processed/` folder structure. Uploaded the cleaned CSV to `processed/`, keeping raw and processed data separated — standard practice for a staging layer.

### 3. Snowflake Connection
Connected Snowflake to S3 using an external stage. Initially attempted this via a Storage Integration + IAM role with cross-account trust (the AWS-recommended, temporary-credential approach), but hit a persistent `sts:AssumeRole` authorization failure despite the trust policy, external ID, and IAM user ARN all being verified correct via CloudTrail. After exhausting standard troubleshooting steps, switched to an IAM user with a scoped-down, read-only access key — a simpler, equally secure approach for a single-developer learning project.

```sql
CREATE STAGE walmart_stage
  URL = 's3://abhiroop-walmart-pipeline/processed/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...')
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
```

### 4. Loading into Snowflake
```sql
COPY INTO walmart_sales
FROM @walmart_stage
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
ON_ERROR = 'CONTINUE';
```
All 6,435 rows loaded successfully.

### 5. Power BI Dashboard
Connected Power BI directly to the Snowflake table and built a dashboard including:
- KPI cards: Total Sales, No. of Weeks, Avg Fuel %, Avg Unemployment %, Total Holiday Weeks, Best Week Sales
- Sales trend by year, split by Holiday Week vs Non-Holiday Week
- Sales vs Temperature scatter plot
- Monthly sales heatmap
- Store-level sales breakdown
- Interactive slicers (Holiday, Store, Year, Month)

## Key Challenge & Troubleshooting

The trickiest part of this project was debugging a cross-account IAM trust relationship between Snowflake and AWS. Despite the trust policy's principal, external ID, and role ARN all matching Snowflake's `DESC INTEGRATION` output exactly (verified multiple times, including a full role recreation), the `AssumeRole` call kept failing. Used AWS CloudTrail Event History to inspect the actual API calls and rule out propagation delay and configuration typos before pivoting to a working alternative (IAM user + access key). This reflects a realistic debugging scenario — cloud IAM configuration issues are common in production data engineering work, and being able to systematically isolate and work around them is a practical skill.

## What I'd Do Differently / Next Steps

- Automate the S3 → Snowflake load with Snowpipe for continuous ingestion
- Revisit the IAM role/Storage Integration approach to fully resolve the trust policy issue for production-grade temporary credentials
- Add data quality checks (e.g., dbt tests) before the load step

## Files in This Repo

- `dashboard.pbix` — Power BI dashboard file
- `dashboard_screenshot.png` — dashboard preview
- `walmart_sales_cleaned.csv` — cleaned dataset used for this project

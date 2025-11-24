# YouTube Data Engineering Project - Complete Breakdown
---

## 🎯 THE BIG PICTURE: What's This Project Really About?

Imagine you're a marketing analyst who needs to answer: "What types of YouTube videos get trending? Which categories are most popular? What makes a video go viral?"

But you've got a problem: The data is messy, scattered across multiple files, in different formats (JSON and CSV), and you can't just open it in Excel because there are thousands of rows.

**Your solution**: Build an automated data pipeline in AWS that:
1. Stores the raw data
2. Cleans and transforms it automatically
3. Makes it easy to query with SQL
4. Visualizes insights in dashboards

**Think of it like**: Building a factory assembly line for data instead of cars.

---

## 📊 THE DATA: What You're Working With

### From Kaggle, you downloaded:

**JSON Files** (one per region):
```
US_category_id.json
GB_category_id.json
CA_category_id.json
etc.
```
These contain category mappings like:
- Category ID 1 = "Film & Animation"
- Category ID 10 = "Music"
- Category ID 24 = "Entertainment"

**CSV Files** (one per region, per day):
```
USvideos.csv
GBvideos.csv
CAvideos.csv
etc.
```
Each contains ~200 trending videos with columns:
- video_id
- title
- channel_title
- category_id (just a number like "24")
- views, likes, dislikes, comments
- publish_time
- tags
- description

**The Problem**: The CSV files only have category_id as a number. You need to JOIN it with the JSON files to get the actual category name.

---

## 🏗️ THE ARCHITECTURE: Step-by-Step Data Flow

### STEP 1: DATA INGESTION (Getting Data into AWS)

**What You Did:**
```bash
# Downloaded data from Kaggle to your laptop
# Used AWS CLI to upload to S3

aws s3 cp USvideos.csv s3://your-bucket/raw/csv/region=US/
aws s3 cp US_category_id.json s3://your-bucket/raw/json/region=US/
```

**What Happened:**
- Your data is now sitting in Amazon S3 (think: cloud Dropbox for big data)
- You organized it into folders:
  - `/raw/csv/` - All the CSV files
  - `/raw/json/` - All the JSON files

**Why S3?**
- It's cheap (pennies per GB)
- Scalable (can store petabytes)
- Other AWS services can directly read from it

---

### STEP 2: DATA CATALOGING (Teaching AWS What Your Data Looks Like)

**The Problem**: AWS doesn't know what's in your files. Is column 1 a date? A number? Text?

**Your Solution: AWS Glue Crawler**

**What is a Crawler?**
Think of it like a robot librarian that:
1. Opens your files in S3
2. Reads the first few rows
3. Figures out the structure (columns, data types)
4. Creates a "table" definition in the Glue Data Catalog

**What You Did:**
```
Created Crawler → Pointed it to s3://your-bucket/raw/json/
Ran Crawler → It created a table called "raw_json" with schema
```

**The Result: Glue Data Catalog**
Now AWS has a metadata database that says:
```
Table: raw_json
Columns:
  - id: string
  - snippet: struct (nested JSON)
    - title: string
    - assignable: boolean
```

**Why This Matters:**
Now you can query your S3 files with SQL as if they were database tables!

---

### STEP 3: QUERYING DATA (First Attempt - It Failed!)

**Tool: AWS Athena**

**What is Athena?**
- It's a serverless SQL query engine
- You write SQL, it reads from S3, returns results
- You only pay per query (per data scanned)

**What You Did:**
```sql
SELECT * FROM raw_json LIMIT 10;
```

**What Happened:** ERROR! 🔥

**Why It Failed:**
The JSON files from Kaggle were formatted like this:
```json
{
  "items": [
    {"id": "1", "snippet": {"title": "Film"}},
    {"id": "10", "snippet": {"title": "Music"}}
  ]
}
```

But Athena expects each JSON object on a separate line (NDJSON format):
```json
{"id": "1", "snippet": {"title": "Film"}}
{"id": "10", "snippet": {"title": "Music"}}
```

**The Fix: You Need to Transform the JSON Files**

---

### STEP 4: DATA TRANSFORMATION (Fixing the JSON Problem)

**Tool: AWS Lambda Function**

**What is Lambda?**
- Serverless computing = you write code, AWS runs it, you don't manage servers
- Perfect for small, quick tasks
- You only pay when the code runs (per millisecond!)

**What You Did:**
Created a Python Lambda function that:

```python
import awswrangler as wr
import pandas as pd

def lambda_handler(event, context):
    # 1. Read the broken JSON from S3
    df = wr.s3.read_json('s3://bucket/raw/json/US_category_id.json')
    
    # 2. Flatten it (convert nested JSON to table)
    # Extract the 'items' array and normalize it
    
    # 3. Write it back as Parquet format to S3
    wr.s3.to_parquet(
        df=df,
        path='s3://bucket/cleansed/json/region=US/',
        dataset=True
    )
```

**Why Parquet Instead of JSON?**
- Parquet is a columnar storage format
- Much faster to query (5-10x)
- Smaller file size (compressed)
- Industry standard for data lakes

**How Lambda Gets Triggered:**
You set up an **S3 Event Notification**:
```
When: New file uploaded to s3://bucket/raw/json/
Do: Automatically run the Lambda function
```

**The Result:**
- Raw JSON in `/raw/json/` (messy, can't query)
- Clean Parquet in `/cleansed/json/` (ready to query!)

---

### STEP 5: PROCESSING CSV FILES (Cleaning the Video Data)

**Tool: AWS Glue Job (PySpark)**

**What is a Glue Job?**
- Managed Apache Spark environment
- For processing large datasets (millions of rows)
- You write PySpark code, AWS runs it on a cluster

**What You Did:**

```python
# Created a Glue Job that:

# 1. Read CSV files from S3
df_csv = glueContext.create_dynamic_frame.from_catalog(
    database="youtube_db",
    table_name="raw_csv"
)

# 2. Clean the data
# - Fix data types (convert strings to integers where needed)
# - Handle missing values
# - Standardize column names

# 3. Write to cleansed layer
glueContext.write_dynamic_frame.from_options(
    frame=df_csv,
    connection_type="s3",
    connection_options={"path": "s3://bucket/cleansed/csv/"},
    format="parquet"
)
```

**Why PySpark Instead of Lambda?**
- Lambda has limits (15 min max, 10GB memory)
- Glue can process huge files in parallel
- Designed for big data ETL

---

### STEP 6: JOINING THE DATA (Creating the Analytics Layer)

**Tool: AWS Glue Studio (Visual ETL)**

**The Goal:** Join CSV data (video stats) with JSON data (category names)

**What You Did:**
Created a visual ETL pipeline:

```
[Source: cleansed_csv] 
        ↓
    [Transform: Cast Types]
        ↓
    [Join] ← [Source: cleansed_json]
        ↓
    [Select Fields]
        ↓
    [Target: s3://bucket/analytics/]
```

**The Join Logic:**
```sql
SELECT 
    csv.video_id,
    csv.title,
    csv.views,
    csv.likes,
    json.category_title,  -- This is what we get from joining!
    csv.region
FROM cleansed_csv csv
LEFT JOIN cleansed_json json
    ON csv.category_id = json.id
    AND csv.region = json.region
```

**The Result:**
Final analytics table with all the data together:
```
video_id | title | views | likes | category_title | region
---------|-------|-------|-------|----------------|-------
abc123   | "..."  | 1.5M  | 50K   | Music          | US
```

---

### STEP 7: QUERYING & VISUALIZATION

**Tool 1: AWS Athena (SQL Queries)**

Now you can run analysis queries:

```sql
-- Which category has most views?
SELECT 
    category_title,
    SUM(views) as total_views,
    COUNT(*) as video_count
FROM analytics_table
GROUP BY category_title
ORDER BY total_views DESC;

-- Top trending videos by region
SELECT 
    region,
    title,
    views,
    likes
FROM analytics_table
WHERE region = 'US'
ORDER BY views DESC
LIMIT 10;
```

**Tool 2: AWS QuickSight (Dashboards)**

Connected QuickSight to Athena and created visualizations:
- Bar chart: Views by Category
- Line chart: Trending over time
- Pie chart: Video distribution by region
- KPIs: Total views, avg likes, etc.

---

## 🔑 KEY AWS SERVICES EXPLAINED

### 1. **S3 (Simple Storage Service)**
**What it is:** Cloud file storage
**Why you used it:** Store all your data files (raw, cleaned, analytics)
**Cost:** ~$0.023 per GB/month
**Think of it as:** Google Drive but for big data

### 2. **IAM (Identity & Access Management)**
**What it is:** Security system for AWS
**Why you used it:** Created roles/permissions so services can talk to each other
**Example:** Lambda needs permission to read S3 and write to S3

### 3. **AWS Glue**
**What it is:** Managed ETL service
**Components you used:**
- **Glue Crawler:** Scans files, creates table schemas
- **Glue Data Catalog:** Metadata database (stores table definitions)
- **Glue Jobs:** Run PySpark code for data transformation
- **Glue Studio:** Visual interface to build ETL pipelines

### 4. **AWS Lambda**
**What it is:** Serverless compute (run code without servers)
**Why you used it:** Quick transformations on small files (JSON to Parquet)
**Limits:** 15 min max runtime, 10GB memory
**Cost:** Free tier = 1M requests/month

### 5. **AWS Athena**
**What it is:** Serverless SQL query engine
**Why you used it:** Run SQL queries on S3 data
**Cost:** $5 per TB scanned
**Speed:** Queries run in seconds, not minutes

### 6. **AWS QuickSight**
**What it is:** Business intelligence / dashboard tool
**Why you used it:** Create visualizations and dashboards
**Alternative:** Tableau, Power BI

---

## 🎓 WHAT ACTUALLY LEARNED

### 1. **Data Lake Architecture**
You built a 3-tier data lake:
- **Landing/Raw Layer:** Original data, untouched
- **Cleansed/Transformed Layer:** Cleaned data
- **Analytics/Curated Layer:** Business-ready data

**Why 3 layers?**
- Keep original data (for audit/debugging)
- Separate transformation logic
- Optimize analytics layer for speed

### 2. **ETL Pipeline Concepts**
- **Extract:** Pull data from sources (Kaggle → S3)
- **Transform:** Clean, join, aggregate data
- **Load:** Write to destination (analytics layer)

### 3. **Event-Driven Architecture**
You set up automation:
- File uploaded to S3 → Triggers Lambda → Transforms data
- Glue Crawler runs → Updates catalog → Athena can query new data

**Why it matters:** No manual intervention, scales automatically

### 4. **Data Formats & Optimization**
- **JSON:** Human-readable, but slow to query
- **CSV:** Simple, but no schema, no compression
- **Parquet:** Columnar, compressed, fast queries (winner!)

### 5. **Partitioning Strategy**
You organized data by region:
```
/analytics/region=US/data.parquet
/analytics/region=GB/data.parquet
```

**Why?**
When you query only US data, Athena doesn't scan GB files = faster + cheaper!

### 6. **IAM Security Best Practices**
- Never use root account for daily work
- Create IAM user with limited permissions
- Use roles (not access keys) for service-to-service access
- Principle of least privilege

### 7. **Serverless vs. Server-based**
**Lambda (Serverless):**
- No server management
- Pay per use
- Auto-scales
- Limited runtime (15 min)

**Glue (Managed Servers):**
- AWS manages cluster
- Can run for hours
- Better for big data

---

### The 2-Minute Pitch:

"I built an end-to-end data pipeline on AWS to analyze YouTube trending video data from Kaggle. The data came in two formats - JSON files with category mappings and CSV files with video statistics across 5 regions.

I designed a three-tier data lake architecture in S3 with raw, cleansed, and analytics layers. For the raw layer, I ingested data using AWS CLI and set up AWS Glue crawlers to automatically discover and catalog the schema.

The JSON files had formatting issues, so I created a Lambda function triggered by S3 events that automatically transforms incoming JSON files to Parquet format for better query performance. For the larger CSV files, I built Glue ETL jobs using PySpark to clean and transform the data.

The key challenge was joining the CSV video data with JSON category data, which I solved using Glue Studio to create a visual ETL pipeline that performs the join and writes to the analytics layer, partitioned by region for query optimization.

Finally, I used AWS Athena for ad-hoc SQL analysis and built QuickSight dashboards to visualize insights like trending categories by region and view patterns over time. The entire pipeline is event-driven and serverless, so it scales automatically and I only pay for what I use."

---

## 🔧 THE ACTUAL TECHNICAL FLOW (TL;DR)

```
KAGGLE DATA
    ↓
LAPTOP (download)
    ↓
AWS CLI (upload)
    ↓
S3 (raw layer)
    ↓
GLUE CRAWLER (scan & catalog)
    ↓
GLUE DATA CATALOG (metadata)
    ↓
┌─────────────────────┬───────────────────┐
│   JSON FILES        │    CSV FILES      │
│   Lambda Function   │    Glue Job       │
│   (small/fast)      │    (big/parallel) │
└─────────────────────┴───────────────────┘
    ↓                        ↓
S3 (cleansed layer - both in Parquet)    ↓
GLUE STUDIO ETL PIPELINE
(join CSV + JSON on category_id)
    ↓
S3 (analytics layer - final joined data)
    ↓
GLUE CRAWLER (catalog analytics table)
    ↓
┌─────────────────────┬───────────────────┐
│   ATHENA            │    QUICKSIGHT     │
│   (SQL queries)     │    (dashboards)   │
└─────────────────────┴───────────────────┘
```



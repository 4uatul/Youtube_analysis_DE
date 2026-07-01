# YouTube Trending Video Data Pipeline

An end-to-end data pipeline I built on AWS to analyze YouTube trending video data. The goal was to answer questions like which categories trend the most, what drives views, and how patterns differ by region, using data that started out messy, scattered across formats, and too large to just open in Excel.

## The data

Source data came from Kaggle, split across two formats:

**JSON files** (one per region) containing category ID to name mappings:
```
US_category_id.json
GB_category_id.json
CA_category_id.json
```

**CSV files** (one per region) with roughly 200 trending videos each:
```
USvideos.csv
GBvideos.csv
CAvideos.csv
```
Columns include video_id, title, channel_title, category_id, views, likes, dislikes, comments, publish_time, tags, and description.

The CSV files only have category_id as a raw number, so I needed to join them against the JSON files to get actual category names.

## Architecture

I built a three-tier data lake in S3:

```
KAGGLE DATA
    |
LAPTOP (download)
    |
AWS CLI (upload)
    |
S3 (raw layer)
    |
GLUE CRAWLER (scan and catalog)
    |
GLUE DATA CATALOG (metadata)
    |
+---------------------+-------------------+
|   JSON FILES        |    CSV FILES      |
|   Lambda Function   |    Glue Job       |
+---------------------+-------------------+
    |                        |
S3 (cleansed layer, both in Parquet)
    |
GLUE STUDIO ETL PIPELINE
(join CSV + JSON on category_id)
    |
S3 (analytics layer, final joined data)
    |
GLUE CRAWLER (catalog analytics table)
    |
+---------------------+-------------------+
|   ATHENA            |    QUICKSIGHT     |
|   (SQL queries)     |    (dashboards)   |
+---------------------+-------------------+
```

**Raw layer**: original data, untouched, kept for audit and debugging.
**Cleansed layer**: transformed into Parquet, schema fixed, ready to query.
**Analytics layer**: joined and business-ready, partitioned by region.

## How I built it

### 1. Ingestion

I downloaded the data from Kaggle and pushed it to S3 with the AWS CLI, organized into `/raw/csv/` and `/raw/json/`:

```bash
aws s3 cp USvideos.csv s3://your-bucket/raw/csv/region=US/
aws s3 cp US_category_id.json s3://your-bucket/raw/json/region=US/
```

I picked S3 because it's cheap, scales to any size, and every other AWS service in this pipeline can read from it directly.

### 2. Cataloging

AWS doesn't know what's inside a raw file by default, so I used a **Glue Crawler** to scan the files in S3, infer the schema, and register a table definition in the **Glue Data Catalog**. Once that ran, I could query the S3 files with SQL as if they were database tables.

### 3. First query attempt, and why it failed

I tried querying the raw JSON with Athena:
```sql
SELECT * FROM raw_json LIMIT 10;
```
It errored out. The Kaggle JSON files were structured as a single object with a nested `items` array, but Athena expects newline-delimited JSON, one object per line. That meant the raw JSON needed to be transformed before it was queryable.

### 4. Fixing the JSON with Lambda

I wrote a Python Lambda function that reads the raw JSON, flattens the nested `items` array into a table, and writes the result back to S3 as Parquet:

```python
import awswrangler as wr

def lambda_handler(event, context):
    df = wr.s3.read_json('s3://bucket/raw/json/US_category_id.json')
    # flatten nested items array
    wr.s3.to_parquet(
        df=df,
        path='s3://bucket/cleansed/json/region=US/',
        dataset=True
    )
```

I chose Parquet over JSON because it's columnar, compresses well, and queries 5 to 10x faster in Athena. The Lambda is triggered automatically by an S3 event notification whenever a new file lands in `/raw/json/`, so there's no manual step involved.

### 5. Processing the CSVs with Glue

The CSV files were larger and needed heavier cleanup (fixing types, handling missing values, standardizing column names), so I used a **Glue Job** running PySpark instead of Lambda. Lambda caps out at 15 minutes and 10GB of memory, which isn't built for this kind of parallel processing at scale.

```python
df_csv = glueContext.create_dynamic_frame.from_catalog(
    database="youtube_db",
    table_name="raw_csv"
)

glueContext.write_dynamic_frame.from_options(
    frame=df_csv,
    connection_type="s3",
    connection_options={"path": "s3://bucket/cleansed/csv/"},
    format="parquet"
)
```

### 6. Joining CSV and JSON

With both datasets cleansed and in Parquet, I used **Glue Studio** to build a visual ETL pipeline that joins the video stats against the category names on `category_id`:

```sql
SELECT
    csv.video_id,
    csv.title,
    csv.views,
    csv.likes,
    json.category_title,
    csv.region
FROM cleansed_csv csv
LEFT JOIN cleansed_json json
    ON csv.category_id = json.id
    AND csv.region = json.region
```

The output lands in the analytics layer, partitioned by region so queries scoped to one region don't scan data for the others.

### 7. Querying and visualization

With the analytics table cataloged, I ran ad-hoc SQL in **Athena**:

```sql
SELECT
    category_title,
    SUM(views) as total_views,
    COUNT(*) as video_count
FROM analytics_table
GROUP BY category_title
ORDER BY total_views DESC;
```

Then connected **QuickSight** to Athena for dashboards: views by category, trending over time, video distribution by region, and top-line KPIs like total views and average likes.

## AWS services used

| Service | Purpose |
|---------|---------|
| S3 | Storage for raw, cleansed, and analytics layers |
| IAM | Roles and permissions between services |
| Glue Crawler | Scans files and infers schema |
| Glue Data Catalog | Metadata store for table definitions |
| Glue Jobs (PySpark) | Large-scale CSV cleaning and transformation |
| Glue Studio | Visual ETL pipeline for the join step |
| Lambda | Event-triggered JSON to Parquet conversion |
| Athena | Serverless SQL queries against S3 |
| QuickSight | Dashboards and visualization |

## What I learned

**Data lake architecture**: splitting storage into raw, cleansed, and analytics layers keeps the original data intact for debugging while letting the analytics layer stay optimized for speed.

**Event-driven pipelines**: an S3 upload triggers Lambda automatically, and crawler runs keep the catalog current, so nothing here needs a manual step to keep flowing.

**File format tradeoffs**: JSON is readable but slow to query, CSV has no schema or compression, Parquet wins on both speed and size for anything going into an analytics layer.

**Partitioning**: organizing the analytics layer by region means a query scoped to one region skips scanning the others entirely, which cuts both cost and query time in Athena.

**IAM practices**: avoided the root account for daily work, used scoped IAM roles for service-to-service access instead of long-lived keys, and kept to least-privilege permissions throughout.

**Serverless vs managed clusters**: Lambda fits small, fast, event-triggered transformations but caps at 15 minutes and 10GB memory. Glue fits the larger PySpark jobs that need to run longer and process data in parallel.

## The pitch

I built an end-to-end data pipeline on AWS to analyze YouTube trending video data from Kaggle, combining JSON category mappings with CSV video statistics across five regions. I designed a three-tier data lake in S3 with raw, cleansed, and analytics layers, using Glue Crawlers to catalog schema automatically. The JSON files had formatting issues that broke Athena queries, so I built a Lambda function triggered by S3 events to convert them to Parquet on the fly. For the larger CSV files, I used Glue ETL jobs with PySpark. The core challenge was joining CSV video data with JSON category data, which I solved with a Glue Studio visual pipeline that writes the joined output to the analytics layer, partitioned by region. I used Athena for ad-hoc SQL analysis and built QuickSight dashboards on top for trend visualization. The whole pipeline is event-driven and serverless, so it scales automatically and I only pay for what runs.

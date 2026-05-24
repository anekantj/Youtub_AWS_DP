# YouTube Trending Videos Data Pipeline

## 1. Project Overview

This is an end-to-end data engineering project that builds a scalable data lake on AWS to analyze YouTube trending videos data. The pipeline processes data from multiple regions, performs ETL transformations, and creates analytics-ready datasets for business intelligence.

---

## 2. Architecture

The project implements a **3-tier data lake architecture**:

1. **Landing Zone (Raw Layer)** - Original data from Kaggle
2. **Cleansed Layer** - Transformed and validated data
3. **Analytics Layer** - Business-ready aggregated data

### Data Flow

```
Kaggle Dataset → S3 Raw → Lambda/Glue ETL → S3 Cleansed → Glue Studio → S3 Analytics → Athena/QuickSight
```

---

## 3. AWS Services Used

| Service | Purpose |
|---------|---------|
| **S3** | Data lake storage (Raw, Cleansed, Analytics buckets) |
| **Lambda** | Serverless ETL for JSON transformation |
| **Glue** | Data cataloging, crawling, and Spark ETL jobs |
| **Athena** | Serverless SQL queries on S3 data |
| **IAM** | Access management and security |
| **CloudWatch** | Monitoring and logging |

---

## 4. Dataset

**Source:** [Kaggle - YouTube Trending Videos Dataset](https://www.kaggle.com/datasets/datasnaek/youtube-new?resource=download)

**Contents:**
- **CSV Files:** Video statistics by region (CA, DE, FR, IN, JP, RU, US, GB)
  - video_id, trending_date, title, channel_title, category_id, views, likes, dislikes, comment_count, etc.
- **JSON Files:** Category mappings for each region
  - Maps category_id to category names (Music, Entertainment, Sports, etc.)

**Regions Processed:** Canada (CA), Germany (DE), Great Britain (GB), United States (US)

---

## 5. Key Features

### 1. **Multi-Tier Data Lake**
- **Raw Zone:** Preserves original data integrity
- **Cleansed Zone:** Validated, typed, and formatted data
- **Analytics Zone:** Business-ready, denormalized datasets

### 2. **Automated ETL Pipeline**
- **Event-Driven:** Lambda automatically triggers on S3 uploads
- **Scalable:** Glue Spark jobs handle large-scale transformations
- **Incremental:** Supports append operations for new data

### 3. **Data Optimization**
- **Format:** Parquet with Snappy compression (columnar storage)
- **Partitioning:** By region and category_id for query performance
- **Schema Evolution:** Proper data typing (BigInt, String, etc.)

### 4. **Query Performance**
- **Predicate Pushdown:** Filters data before processing
- **Partition Pruning:** Queries only relevant partitions
- **Columnar Format:** Efficient analytical queries

---

## 6. S3 Bucket Structure

```
de-on-youtube-raw-useast1-dev/
├── youtube/
│   ├── raw_statistics_reference_data/
│   │   ├── CA_category_id.json
│   │   ├── DE_category_id.json
│   │   
│   └── raw_statistics/
│       ├── region=CA/
│       │   └── CAvideos.csv
│       ├── region=DE/
│       │   └── DEvideos.csv
│       

de-on-youtube-cleansed-useast1-dev/
├── youtube/
│   ├── clean_statistics_reference_data/ (Parquet)
│   └── raw_statistics/
│       ├── region=CA/ (Parquet)
│       ├── region=GB/ (Parquet)
│       └── region=US/ (Parquet)

de-on-youtube-analytics-useast1-dev/
└── final_analytics/
    ├── region=CA/
    │   ├── category_id=1/ (Parquet)
    │   ├── category_id=10/ (Parquet)
    │   
    

de-on-youtube-athena-job-useast1-dev/
└── query_results/
```

---
## 7. Glue Data Catalog

### Databases

| Database | Purpose | Tables |
|----------|---------|--------|
| `db_youtube_raw` | Raw CSV crawler | `raw_statistics` |
| `db_youtube_clean` | Cleansed data | `clean_statistics_reference_data`, `raw_statistics` |
| `db_youtube_analytics` | Final analytics | `final_analytics` |

### Crawlers

1. **de-on-youtube-raw-glue-catalog-01**
   - Target: JSON reference data
   - Output: `db_youtube_raw`

2. **de-on-youtube-raw-csv-crawler-01**
   - Target: Raw CSV statistics
   - Output: `db_youtube_raw`
   - Partitions: Auto-discovered by region

3. **de-on-youtube-cleansed-version-csv-to-parquet-etl**
   - Target: Cleansed Parquet data
   - Output: `db_youtube_clean`

---

## 8. Analytics & Visualization

### Athena Queries

**Sample Join Query:**
```sql
SELECT 
    a.title,
    a.category_id,
    b.snippet_title as category_name,
    a.views,
    a.likes,
    a.comment_count,
    a.region
FROM 
    db_youtube_clean.raw_statistics a
INNER JOIN 
    db_youtube_clean.clean_statistics_reference_data b 
    ON CAST(a.category_id AS BIGINT) = b.id
WHERE 
    a.region = 'ca'
LIMIT 100;
```
---

## 9. Security & IAM

### IAM Roles Created

1. **de-on-youtube-glue-s3-role**
   - Policies: `AmazonS3FullAccess`, `AWSGlueServiceRole`
   - Used by: Glue Crawlers, Glue ETL Jobs

2. **de-on-youtube-raw-s3-lambda-role**
   - Policies: `AmazonS3FullAccess`, `AWSGlueServiceRole`, Lambda execution
   - Used by: Lambda function

### Security Best Practices
- Root account MFA enabled
- IAM admin user for daily operations
- Least privilege principle
- S3 server-side encryption (SSE-S3)
- No public bucket access
- Credential rotation

---

## 10. Troubleshooting

### Common Issues

**1. Lambda JSON Parsing Error**
- **Problem:** "JSON has extra closing braces"
- **Solution:** JSON must be in single-line format (no pretty-printing). Lambda extracts nested "items" array.

**2. Athena Schema Mismatch**
- **Problem:** "HIVE_BAD_DATA: Parquet file incompatible with BigInt"
- **Solution:** 
  1. Delete cleansed data
  2. Update Glue Catalog schema
  3. Re-run Lambda to regenerate Parquet with correct schema

**3. Glue Job Encoding Error**
- **Problem:** UTF-8 encoding errors on non-English data
- **Solution:** Apply predicate pushdown to filter regions: `region IN ('ca', 'gb', 'us')`

**4. S3 Permissions Error**
- **Problem:** "Access Denied" on S3 operations
- **Solution:** Verify IAM role has `AmazonS3FullAccess` policy attached

**5. Lambda Timeout**
- **Problem:** Function times out after 3 seconds
- **Solution:** Increase timeout in Configuration → General → Timeout (set to 3 minutes)
---

## 11. References

Guidance: https://www.youtube.com/watch?v=yZKJFKu49Dk


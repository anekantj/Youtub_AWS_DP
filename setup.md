    
# Setup & Deployment

### Prerequisites
- AWS Account
- AWS CLI installed and configured
- Python 3.8+ (for local testing)
- Dataset downloaded from Kaggle

### Step-by-Step Deployment

#### 1. **Configure AWS CLI**
```bash
aws configure
# Enter Access Key, Secret Key, Region (us-east-1), Output format (json)
```

#### 2. **Create S3 Buckets**
```bash
# Raw bucket
aws s3 mb s3://de-on-youtube-raw-useast1-dev --region us-east-1

# Cleansed bucket
aws s3 mb s3://de-on-youtube-cleansed-useast1-dev --region us-east-1

# Analytics bucket
aws s3 mb s3://de-on-youtube-analytics-useast1-dev --region us-east-1

# Athena output bucket
aws s3 mb s3://de-on-youtube-athena-job-useast1-dev --region us-east-1
```

#### 3. **Upload Data to S3**
```bash
# Navigate to dataset folder
cd /path/to/youtube_data/

# Upload JSON reference data
aws s3 cp . s3://de-on-youtube-raw-useast1-dev/youtube/raw_statistics_reference_data/ \
  --recursive --exclude "*" --include "*.json"

# Upload CSV files by region
aws s3 cp CAvideos.csv s3://de-on-youtube-raw-useast1-dev/youtube/raw_statistics/region=CA/
aws s3 cp DEvideos.csv s3://de-on-youtube-raw-useast1-dev/youtube/raw_statistics/region=DE/
aws s3 cp GBvideos.csv s3://de-on-youtube-raw-useast1-dev/youtube/raw_statistics/region=GB/
aws s3 cp USvideos.csv s3://de-on-youtube-raw-useast1-dev/youtube/raw_statistics/region=US/
# ... repeat for other regions
```

#### 4. **Create IAM Roles**

**Glue Role:**
```bash
# Create trust policy for Glue
cat > glue-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "glue.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create role
aws iam create-role \
  --role-name de-on-youtube-glue-s3-role \
  --assume-role-policy-document file://glue-trust-policy.json

# Attach policies
aws iam attach-role-policy \
  --role-name de-on-youtube-glue-s3-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name de-on-youtube-glue-s3-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
```

**Lambda Role:** (Similar process with Lambda trust policy)

#### 5. **Create Lambda Function**

```python
# lambda_function.py (simplified)
import awswrangler as wr
import pandas as pd
import os

def lambda_handler(event, context):
    # Get S3 event details
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Read JSON
    s3_path = f"s3://{bucket}/{key}"
    df = wr.s3.read_json(path=s3_path)
    
    # Normalize nested structure
    df_normalized = pd.json_normalize(df['items'])
    
    # Write to cleansed bucket
    output_path = os.environ['s3_cleansed_layer']
    wr.s3.to_parquet(
        df=df_normalized,
        path=output_path,
        dataset=True,
        database=os.environ['glue_catalog_db_name'],
        table=os.environ['glue_catalog_table_name'],
        mode=os.environ['write_data_operation']
    )
    
    return {'statusCode': 200}
```

Deploy via AWS Console or CLI.

#### 6. **Create Glue Crawlers**

```bash
# Create JSON crawler
aws glue create-crawler \
  --name de-on-youtube-raw-glue-catalog-01 \
  --role de-on-youtube-glue-s3-role \
  --database-name db_youtube_raw \
  --targets S3Targets=[{Path=s3://de-on-youtube-raw-useast1-dev/youtube/raw_statistics_reference_data/}]

# Create CSV crawler
aws glue create-crawler \
  --name de-on-youtube-raw-csv-crawler-01 \
  --role de-on-youtube-glue-s3-role \
  --database-name db_youtube_raw \
  --targets S3Targets=[{Path=s3://de-on-youtube-raw-useast1-dev/youtube/raw_statistics/}]

# Run crawlers
aws glue start-crawler --name de-on-youtube-raw-glue-catalog-01
aws glue start-crawler --name de-on-youtube-raw-csv-crawler-01
```

#### 7. **Create Glue ETL Jobs**

Use AWS Glue Console or Glue Studio to create visual ETL jobs (see Architecture section for details).

#### 8. **Configure Athena**

```bash
# Set query result location
aws athena create-work-group \
  --name youtube-analytics \
  --configuration "ResultConfigurationUpdates={OutputLocation=s3://de-on-youtube-athena-job-useast1-dev/}"
```

#### 9. **Set Up QuickSight/Athena for querying purposes**

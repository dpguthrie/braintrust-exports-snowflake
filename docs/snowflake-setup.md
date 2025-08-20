# Snowflake and Snowpipe Setup

This guide walks you through setting up Snowflake to automatically ingest Braintrust data from S3 using Snowpipe.

## Prerequisites

- Snowflake account with ACCOUNTADMIN or equivalent privileges
- AWS S3 bucket with Braintrust exports (see [AWS Setup](./aws-setup.md))
- AWS IAM user credentials with S3 read access

## Overview

We'll set up:
1. Snowflake database and schemas
2. Tables to store Braintrust data
3. External stage pointing to S3 bucket
4. File format for JSON/Parquet parsing
5. Snowpipe for automatic ingestion

## Step 1: Initial Database Setup

### Create Database and Schemas

```sql
-- Create database for Braintrust data
CREATE DATABASE BRAINTRUST_DATA;

-- Use the database
USE DATABASE BRAINTRUST_DATA;

-- Create schemas for organization
CREATE SCHEMA RAW_DATA;        -- Raw ingested data
CREATE SCHEMA ANALYTICS;      -- Transformed data for analysis
CREATE SCHEMA MONITORING;     -- Metadata and monitoring tables

-- Set default schema
USE SCHEMA RAW_DATA;
```

### Create User and Role for Snowpipe

```sql
-- Create role for Snowpipe operations
CREATE ROLE SNOWPIPE_ROLE;

-- Grant necessary privileges
GRANT USAGE ON DATABASE BRAINTRUST_DATA TO ROLE SNOWPIPE_ROLE;
GRANT USAGE ON SCHEMA RAW_DATA TO ROLE SNOWPIPE_ROLE;
GRANT CREATE TABLE ON SCHEMA RAW_DATA TO ROLE SNOWPIPE_ROLE;
GRANT CREATE STAGE ON SCHEMA RAW_DATA TO ROLE SNOWPIPE_ROLE;
GRANT CREATE PIPE ON SCHEMA RAW_DATA TO ROLE SNOWPIPE_ROLE;

-- Create user for Snowpipe (optional - can use existing user)
CREATE USER SNOWPIPE_USER
  PASSWORD = 'ChangeThisPassword123!'
  DEFAULT_ROLE = SNOWPIPE_ROLE
  DEFAULT_WAREHOUSE = COMPUTE_WH;

-- Grant role to user
GRANT ROLE SNOWPIPE_ROLE TO USER SNOWPIPE_USER;
```

## Step 2: Create Tables

### Braintrust Traces Table

For high-level trace data:

```sql
CREATE TABLE braintrust_traces (
    id STRING NOT NULL,
    project_id STRING,
    created TIMESTAMP_NTZ,
    metadata VARIANT,
    scores VARIANT,
    metrics VARIANT,
    input STRING,
    output STRING,
    expected STRING,
    tags VARIANT,
    span_id STRING,
    root_span_id STRING,
    -- Additional fields for analysis
    ingestion_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    file_name STRING,
    PRIMARY KEY (id, project_id)
);
```

### Braintrust Spans Table

For detailed span-level data:

```sql
CREATE TABLE braintrust_spans (
    id STRING NOT NULL,
    trace_id STRING,
    parent_span_id STRING,
    project_id STRING,
    created TIMESTAMP_NTZ,
    name STRING,
    type STRING,
    input VARIANT,
    output VARIANT,
    metadata VARIANT,
    metrics VARIANT,
    error STRING,
    -- Additional fields
    ingestion_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    file_name STRING,
    PRIMARY KEY (id, project_id)
);
```

### Raw Data Table (Optional)

For storing complete raw JSON before parsing:

```sql
CREATE TABLE braintrust_raw (
    record_id STRING DEFAULT UUID_STRING(),
    raw_data VARIANT,
    file_name STRING,
    ingestion_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    processed BOOLEAN DEFAULT FALSE
);
```

## Step 3: Create Storage Integration

Create a storage integration to securely connect to S3:

```sql
CREATE STORAGE INTEGRATION s3_braintrust_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR_ACCOUNT_ID:user/SnowflakeS3User'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-company-braintrust-exports/')
  COMMENT = 'Integration for Braintrust S3 exports';
```

**Note**: Replace `YOUR_ACCOUNT_ID` and bucket name with your actual values.

### Get Snowflake's AWS User Info

```sql
DESC STORAGE INTEGRATION s3_braintrust_integration;
```

Note the `STORAGE_AWS_IAM_USER_ARN` and `STORAGE_AWS_EXTERNAL_ID` - you may need these for cross-account access.

## Step 4: Create External Stage

### For JSON Lines Format

```sql
CREATE STAGE s3_braintrust_stage
  STORAGE_INTEGRATION = s3_braintrust_integration
  URL = 's3://your-company-braintrust-exports/braintrust-exports/'
  FILE_FORMAT = (
    TYPE = JSON
    STRIP_OUTER_ARRAY = FALSE
    COMMENT = 'JSON Lines format for Braintrust exports'
  );
```

### For Parquet Format

```sql
CREATE STAGE s3_braintrust_parquet_stage
  STORAGE_INTEGRATION = s3_braintrust_integration
  URL = 's3://your-company-braintrust-exports/braintrust-exports/'
  FILE_FORMAT = (
    TYPE = PARQUET
    COMMENT = 'Parquet format for Braintrust exports'
  );
```

### Test the Stage

```sql
-- List files in the stage
LIST @s3_braintrust_stage;

-- Test reading a file
SELECT $1 FROM @s3_braintrust_stage/2024/01/15/ LIMIT 5;
```

## Step 5: Create File Formats

### JSON Lines File Format

```sql
CREATE FILE FORMAT json_format
  TYPE = JSON
  STRIP_OUTER_ARRAY = FALSE
  DATE_FORMAT = 'AUTO'
  TIMESTAMP_FORMAT = 'AUTO'
  COMMENT = 'File format for JSON Lines Braintrust exports';
```

### Parquet File Format

```sql
CREATE FILE FORMAT parquet_format
  TYPE = PARQUET
  COMMENT = 'File format for Parquet Braintrust exports';
```

## Step 6: Create Snowpipes

### Snowpipe for Traces (JSON)

```sql
CREATE PIPE braintrust_traces_pipe
  AUTO_INGEST = TRUE
  AS
  COPY INTO braintrust_traces (
    id,
    project_id,
    created,
    metadata,
    scores,
    metrics,
    input,
    output,
    expected,
    tags,
    span_id,
    root_span_id,
    file_name
  )
  FROM (
    SELECT 
      $1:id::STRING,
      $1:project_id::STRING,
      $1:created::TIMESTAMP_NTZ,
      $1:metadata,
      $1:scores,
      $1:metrics,
      $1:input::STRING,
      $1:output::STRING,
      $1:expected::STRING,
      $1:tags,
      $1:span_id::STRING,
      $1:root_span_id::STRING,
      METADATA$FILENAME
    FROM @s3_braintrust_stage
  )
  FILE_FORMAT = json_format
  ON_ERROR = CONTINUE;
```

### Snowpipe for Spans (JSON)

```sql
CREATE PIPE braintrust_spans_pipe
  AUTO_INGEST = TRUE
  AS
  COPY INTO braintrust_spans (
    id,
    trace_id,
    parent_span_id,
    project_id,
    created,
    name,
    type,
    input,
    output,
    metadata,
    metrics,
    error,
    file_name
  )
  FROM (
    SELECT 
      $1:id::STRING,
      $1:trace_id::STRING,
      $1:parent_span_id::STRING,
      $1:project_id::STRING,
      $1:created::TIMESTAMP_NTZ,
      $1:name::STRING,
      $1:type::STRING,
      $1:input,
      $1:output,
      $1:metadata,
      $1:metrics,
      $1:error::STRING,
      METADATA$FILENAME
    FROM @s3_braintrust_stage
  )
  FILE_FORMAT = json_format
  ON_ERROR = CONTINUE;
```

### Get Snowpipe SQS Information

```sql
-- Get the SQS queue ARN for S3 notifications
SHOW PIPES LIKE 'braintrust_traces_pipe';

-- Or describe the pipe for detailed info
DESC PIPE braintrust_traces_pipe;
```

Note the `notification_channel` value - this is the SQS queue ARN you'll need to configure in S3.

## Step 7: Configure S3 Event Notifications

Update your S3 bucket to send notifications to Snowpipe's SQS queue:

```json
{
  "QueueConfigurations": [
    {
      "Id": "SnowpipeAutoIngest",
      "QueueArn": "arn:aws:sqs:us-east-1:123456789012:sf-snowpipe-AIDACKCEVSQ6C2EXAMPLE-r8LgUephy2hjS_d7xbZb-A",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "braintrust-exports/"
            },
            {
              "Name": "suffix",
              "Value": ".jsonl"
            }
          ]
        }
      }
    }
  ]
}
```

Use the actual `notification_channel` value from your Snowpipe.

## Step 8: Create Monitoring Views

### Ingestion Monitoring

```sql
-- Switch to monitoring schema
USE SCHEMA MONITORING;

-- Create view to monitor ingestion
CREATE VIEW ingestion_status AS
SELECT 
  pipe_name,
  file_name,
  row_count,
  error_count,
  first_error_message,
  last_received_time,
  last_loaded_time
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -24, current_timestamp())
));

-- Create view for pipe status
CREATE VIEW pipe_status AS
SELECT 
  pipe_name,
  is_stalled,
  stalled_files_count,
  stalled_files_oldest_timestamp,
  pending_files_count
FROM table(information_schema.pipe_usage_history(
  date_range_start => dateadd(days, -7, current_timestamp()),
  date_range_end => current_timestamp()
));
```

### Data Quality Monitoring

```sql
USE SCHEMA RAW_DATA;

-- Create view for data quality checks
CREATE VIEW data_quality_summary AS
SELECT 
  DATE(ingestion_timestamp) as ingestion_date,
  COUNT(*) as total_records,
  COUNT(DISTINCT project_id) as unique_projects,
  COUNT(CASE WHEN id IS NULL THEN 1 END) as null_ids,
  COUNT(CASE WHEN created IS NULL THEN 1 END) as null_timestamps,
  MIN(created) as earliest_record,
  MAX(created) as latest_record
FROM braintrust_traces
GROUP BY DATE(ingestion_timestamp)
ORDER BY ingestion_date DESC;
```

## Step 9: Create Analytics Views

### Switch to Analytics Schema

```sql
USE SCHEMA ANALYTICS;
```

### Daily Metrics Summary

```sql
CREATE VIEW daily_metrics AS
SELECT 
  project_id,
  DATE(created) as date,
  COUNT(*) as total_traces,
  AVG(CAST(metrics:total_cost AS FLOAT)) as avg_cost,
  SUM(CAST(metrics:total_tokens AS INT)) as total_tokens,
  AVG(CAST(scores:Factuality AS FLOAT)) as avg_factuality_score,
  AVG(CAST(scores:Helpfulness AS FLOAT)) as avg_helpfulness_score
FROM RAW_DATA.braintrust_traces
WHERE metrics:total_cost IS NOT NULL
GROUP BY project_id, DATE(created)
ORDER BY date DESC, project_id;
```

### Top Errors View

```sql
CREATE VIEW top_errors AS
SELECT 
  project_id,
  error,
  COUNT(*) as error_count,
  MAX(created) as last_occurrence
FROM RAW_DATA.braintrust_spans
WHERE error IS NOT NULL
GROUP BY project_id, error
ORDER BY error_count DESC;
```

## Step 10: Set Up Alerts

### Create Stored Procedure for Monitoring

```sql
CREATE OR REPLACE PROCEDURE check_ingestion_health()
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
  stalled_pipes INTEGER;
  recent_errors INTEGER;
  result STRING;
BEGIN
  -- Check for stalled pipes
  SELECT COUNT(*) INTO stalled_pipes
  FROM table(information_schema.pipe_usage_history(
    date_range_start => dateadd(hours, -1, current_timestamp()),
    date_range_end => current_timestamp()
  ))
  WHERE is_stalled = TRUE;
  
  -- Check for recent errors
  SELECT COUNT(*) INTO recent_errors
  FROM table(information_schema.copy_history(
    table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
    start_time => dateadd(hours, -1, current_timestamp())
  ))
  WHERE error_count > 0;
  
  -- Build result message
  result := 'Health Check - Stalled Pipes: ' || stalled_pipes || ', Recent Errors: ' || recent_errors;
  
  -- Log results (optional)
  INSERT INTO MONITORING.health_check_log (timestamp, stalled_pipes, recent_errors)
  VALUES (CURRENT_TIMESTAMP(), stalled_pipes, recent_errors);
  
  RETURN result;
END;
$$;
```

### Create Task for Regular Monitoring

```sql
-- Create task to run health checks
CREATE TASK health_check_task
  WAREHOUSE = COMPUTE_WH
  SCHEDULE = 'USING CRON 0 * * * * UTC'  -- Every hour
AS
  CALL check_ingestion_health();

-- Start the task
ALTER TASK health_check_task RESUME;
```

## Step 11: Testing and Validation

### Manual Testing

```sql
-- Test manual copy
COPY INTO braintrust_traces
FROM @s3_braintrust_stage
FILE_FORMAT = json_format
ON_ERROR = CONTINUE;

-- Check results
SELECT COUNT(*) FROM braintrust_traces;
SELECT * FROM braintrust_traces LIMIT 5;
```

### Validate Snowpipe Status

```sql
-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('braintrust_traces_pipe');

-- Check recent pipe history
SELECT *
FROM table(information_schema.pipe_usage_history(
  date_range_start => dateadd(hours, -24, current_timestamp()),
  date_range_end => current_timestamp()
))
WHERE pipe_name = 'BRAINTRUST_TRACES_PIPE';
```

### Monitor Copy History

```sql
-- Check copy history
SELECT *
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -24, current_timestamp())
))
ORDER BY last_loaded_time DESC;
```

## Troubleshooting

### Common Issues

1. **Permission Errors**
   ```sql
   -- Check storage integration status
   DESC STORAGE INTEGRATION s3_braintrust_integration;
   
   -- Verify stage access
   LIST @s3_braintrust_stage;
   ```

2. **File Format Errors**
   ```sql
   -- Test file format with sample data
   SELECT $1 FROM @s3_braintrust_stage/sample_file.jsonl LIMIT 1;
   ```

3. **Snowpipe Not Triggering**
   ```sql
   -- Check if S3 notifications are configured correctly
   SELECT SYSTEM$PIPE_STATUS('braintrust_traces_pipe');
   
   -- Manually refresh pipe
   ALTER PIPE braintrust_traces_pipe REFRESH;
   ```

### Useful Queries

```sql
-- Check recent ingestion activity
SELECT 
  file_name,
  row_count,
  error_count,
  last_loaded_time
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -2, current_timestamp())
))
ORDER BY last_loaded_time DESC;

-- Check pipe queue status
SELECT 
  pipe_name,
  pending_files_count,
  is_stalled,
  stalled_files_count
FROM table(information_schema.pipe_usage_history(
  date_range_start => current_timestamp() - INTERVAL '1 hour',
  date_range_end => current_timestamp()
));
```

## Performance Optimization

### Clustering Keys

```sql
-- Add clustering key for better query performance
ALTER TABLE braintrust_traces CLUSTER BY (project_id, DATE(created));
ALTER TABLE braintrust_spans CLUSTER BY (project_id, DATE(created));
```

### Partition Tables by Date

```sql
-- For very large datasets, consider partitioning
CREATE TABLE braintrust_traces_partitioned (
  -- ... same columns as braintrust_traces
)
PARTITION BY (DATE(created));
```

## Next Steps

After setting up Snowflake:
1. Test the complete pipeline with [Testing Guide](./testing.md)
2. Set up monitoring and alerting
3. Create business intelligence dashboards
4. Configure data retention policies
5. Set up automated data quality checks

## Cost Management

- Use appropriate warehouse sizes
- Set up auto-suspend and auto-resume
- Monitor credit usage
- Consider using multi-cluster warehouses for concurrent workloads
- Optimize table clustering and partitioning
- Use result caching effectively

---

## 📖 Documentation Navigation

**← Previous:** [Braintrust S3 Export Setup](./braintrust-setup.md) | **Next:** [Testing the Complete Pipeline →](./testing.md)

### Complete Setup Guide:
1. [Getting Started](../README.md)
2. [AWS Infrastructure Setup](./aws-setup.md)
3. [Braintrust S3 Export Setup](./braintrust-setup.md)
4. **Snowflake and Snowpipe Setup** (You are here)
5. [Testing the Complete Pipeline](./testing.md)
6. [Troubleshooting Guide](./troubleshooting.md) 
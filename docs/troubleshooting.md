# Troubleshooting Guide

This guide covers common issues and solutions for the Braintrust to Snowflake data pipeline.

## Quick Diagnostics

### Pipeline Health Check

Run these commands to get a quick overview of pipeline health:

```sql
-- Check Snowpipe status
SELECT SYSTEM$PIPE_STATUS('braintrust_traces_pipe');

-- Check recent ingestion activity
SELECT 
  COUNT(*) as recent_records,
  MAX(ingestion_timestamp) as latest_ingestion,
  DATEDIFF(minute, MAX(ingestion_timestamp), CURRENT_TIMESTAMP()) as minutes_since_last
FROM braintrust_traces
WHERE ingestion_timestamp >= CURRENT_TIMESTAMP() - INTERVAL '1 hour';

-- Check for recent errors
SELECT *
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -24, current_timestamp())
))
WHERE error_count > 0
ORDER BY last_loaded_time DESC;
```

### AWS Infrastructure Check

```bash
# Test S3 access with Braintrust role
aws sts get-caller-identity

# List recent files in S3
aws s3 ls s3://your-bucket/braintrust-exports/ --recursive | tail -10

# Check S3 bucket notifications
aws s3api get-bucket-notification-configuration --bucket your-bucket
```

## Common Issues and Solutions

### 1. Braintrust Export Issues

#### Issue: Braintrust automation test fails

**Symptoms:**
- Test automation button shows errors
- No files appearing in S3 bucket
- Error messages about permissions

**Diagnostics:**
```bash
# Check if role can be assumed
aws sts assume-role \
  --role-arn arn:aws:iam::YOUR_ACCOUNT_ID:role/BraintrustExportRole \
  --role-session-name test-session \
  --external-id bt:YOUR_ORG_ID:test

# Check S3 bucket permissions
aws s3api get-bucket-policy --bucket your-bucket
aws s3api get-bucket-acl --bucket your-bucket
```

**Solutions:**
1. **IAM Role Issues:**
   ```json
   // Verify trust policy includes correct external ID
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::024025874120:role/prod-export-automation-role"
         },
         "Action": "sts:AssumeRole",
         "Condition": {
           "StringEquals": {
             "sts:ExternalId": "bt:YOUR_ORG_ID:*"
           }
         }
       }
     ]
   }
   ```

2. **S3 Permissions:**
   ```bash
   # Ensure IAM policy allows necessary S3 operations
   aws iam get-role-policy --role-name BraintrustExportRole --policy-name BraintrustS3ExportPolicy
   ```

3. **Bucket Configuration:**
   - Ensure bucket exists in correct region
   - Verify bucket name matches exactly in automation config
   - Check bucket encryption settings don't block access

#### Issue: Automation runs but no data exported

**Symptoms:**
- Automation shows successful runs
- Empty or no files in S3
- BTQL filter might be too restrictive

**Diagnostics:**
```yaml
# In Braintrust, test your BTQL filter
select: count(1)
from: project_logs('<PROJECT_ID>')
filter: your_filter_conditions

# Check date ranges
select: min(created), max(created)
from: project_logs('<PROJECT_ID>')
```

**Solutions:**
1. **BTQL Filter Too Restrictive:**
   - Remove or broaden the BTQL filter temporarily
   - Check date ranges in filter conditions
   - Verify metadata field names and values

2. **No Data in Time Window:**
   - Check automation interval vs. data creation times
   - Ensure data exists in the project
   - Consider expanding the time window

#### Issue: Partial data export

**Symptoms:**
- Some records missing from exports
- Inconsistent export file sizes
- Timeout errors in Braintrust logs

**Solutions:**
1. **Reduce Batch Size:**
   - Lower the batch size in automation settings
   - Try 500 or fewer records per batch

2. **Adjust Export Schedule:**
   - Increase interval between exports
   - Avoid peak usage times

### 2. AWS S3 Issues

#### Issue: S3 access denied errors

**Symptoms:**
- "Access Denied" errors in Snowflake
- Cannot list S3 objects
- Storage integration test fails

**Diagnostics:**
```sql
-- Test storage integration
DESC STORAGE INTEGRATION s3_braintrust_integration;

-- Test stage access
LIST @s3_braintrust_stage;
```

```bash
# Test S3 access directly
aws s3 ls s3://your-bucket/braintrust-exports/ --profile snowflake
```

**Solutions:**
1. **IAM User Permissions:**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "s3:GetObject",
           "s3:GetObjectVersion",
           "s3:ListBucket",
           "s3:GetBucketLocation"
         ],
         "Resource": [
           "arn:aws:s3:::your-bucket",
           "arn:aws:s3:::your-bucket/*"
         ]
       }
     ]
   }
   ```

2. **Storage Integration Configuration:**
   ```sql
   -- Recreate storage integration with correct settings
   DROP STORAGE INTEGRATION s3_braintrust_integration;
   
   CREATE STORAGE INTEGRATION s3_braintrust_integration
     TYPE = EXTERNAL_STAGE
     STORAGE_PROVIDER = S3
     ENABLED = TRUE
     STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR_ACCOUNT_ID:user/SnowflakeS3User'
     STORAGE_ALLOWED_LOCATIONS = ('s3://your-bucket/');
   ```

#### Issue: S3 event notifications not working

**Symptoms:**
- Files appear in S3 but Snowpipe doesn't trigger
- Manual pipe refresh works
- Notification queue is empty

**Diagnostics:**
```sql
-- Get Snowpipe notification channel
DESC PIPE braintrust_traces_pipe;

-- Check pipe queue status
SELECT *
FROM table(information_schema.pipe_usage_history(
  date_range_start => dateadd(hours, -1, current_timestamp()),
  date_range_end => current_timestamp()
));
```

```bash
# Check S3 bucket notification configuration
aws s3api get-bucket-notification-configuration --bucket your-bucket

# Check SQS queue (if using custom queue)
aws sqs get-queue-attributes --queue-url YOUR_QUEUE_URL --attribute-names All
```

**Solutions:**
1. **Configure S3 Notifications:**
   ```bash
   # Use the notification channel from DESC PIPE output
   aws s3api put-bucket-notification-configuration \
     --bucket your-bucket \
     --notification-configuration file://notification-config.json
   ```

2. **Verify Event Filter:**
   - Check prefix and suffix filters match your file pattern
   - Ensure events include "s3:ObjectCreated:*"

### 3. Snowflake Ingestion Issues

#### Issue: Snowpipe not running

**Symptoms:**
- PIPE_STATUS shows "STOPPED_ON_ERROR" or "PAUSED"
- No recent copy history
- Files accumulating in S3

**Diagnostics:**
```sql
-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('braintrust_traces_pipe');

-- Check for errors
SELECT *
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(days, -1, current_timestamp())
))
WHERE error_count > 0;
```

**Solutions:**
1. **Resume Pipe:**
   ```sql
   ALTER PIPE braintrust_traces_pipe SET PIPE_EXECUTION_PAUSED = FALSE;
   ```

2. **Fix Underlying Issues:**
   - Resolve file format problems
   - Fix table schema mismatches
   - Address permission issues

3. **Refresh Pipe:**
   ```sql
   ALTER PIPE braintrust_traces_pipe REFRESH;
   ```

#### Issue: File format errors

**Symptoms:**
- "JSON parsing error" messages
- High error_count in copy history
- Some files processed, others fail

**Diagnostics:**
```sql
-- Test file format with a specific file
SELECT $1 FROM @s3_braintrust_stage/path/to/specific/file.jsonl LIMIT 1;

-- Check error details
SELECT 
  file_name,
  first_error_message,
  error_count
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -24, current_timestamp())
))
WHERE error_count > 0;
```

**Solutions:**
1. **Update File Format:**
   ```sql
   -- Create more permissive file format
   CREATE OR REPLACE FILE FORMAT json_format_flexible
     TYPE = JSON
     STRIP_OUTER_ARRAY = FALSE
     DATE_FORMAT = 'AUTO'
     TIMESTAMP_FORMAT = 'AUTO'
     ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
   ```

2. **Handle Null Values:**
   ```sql
   -- Update COPY statement to handle nulls
   COPY INTO braintrust_traces (
     id,
     project_id,
     created,
     metadata,
     scores,
     metrics,
     input,
     output
   )
   FROM (
     SELECT 
       COALESCE($1:id::STRING, 'unknown'),
       COALESCE($1:project_id::STRING, 'unknown'),
       TRY_CAST($1:created AS TIMESTAMP_NTZ),
       $1:metadata,
       $1:scores,
       $1:metrics,
       $1:input::STRING,
       $1:output::STRING
     FROM @s3_braintrust_stage
   );
   ```

#### Issue: Data type conversion errors

**Symptoms:**
- Timestamp parsing failures
- Numeric conversion errors
- Variant field access issues

**Solutions:**
1. **Use TRY_CAST Functions:**
   ```sql
   SELECT 
     TRY_CAST($1:created AS TIMESTAMP_NTZ) as created,
     TRY_CAST($1:metrics:total_cost AS FLOAT) as cost,
     TRY_CAST($1:metrics:total_tokens AS INTEGER) as tokens
   FROM @s3_braintrust_stage;
   ```

2. **Handle Different Date Formats:**
   ```sql
   SELECT 
     CASE 
       WHEN $1:created LIKE '%Z' THEN TRY_CAST($1:created AS TIMESTAMP_NTZ)
       WHEN $1:created LIKE '%+%' THEN TRY_CAST($1:created AS TIMESTAMP_TZ)
       ELSE TRY_CAST($1:created AS TIMESTAMP_NTZ)
     END as created
   FROM @s3_braintrust_stage;
   ```

### 4. Performance Issues

#### Issue: Slow query performance

**Symptoms:**
- Queries timeout or take very long
- High warehouse usage
- Memory errors

**Diagnostics:**
```sql
-- Check table size and clustering
SELECT 
  COUNT(*) as row_count,
  COUNT(DISTINCT project_id) as unique_projects,
  MIN(created) as earliest_date,
  MAX(created) as latest_date
FROM braintrust_traces;

-- Check clustering information
SELECT SYSTEM$CLUSTERING_INFORMATION('braintrust_traces');
```

**Solutions:**
1. **Add Clustering Keys:**
   ```sql
   ALTER TABLE braintrust_traces CLUSTER BY (project_id, DATE(created));
   ```

2. **Optimize Queries:**
   ```sql
   -- Use filters that leverage clustering
   SELECT * 
   FROM braintrust_traces 
   WHERE project_id = 'specific_project'
     AND created >= '2024-01-01'
     AND created < '2024-02-01';
   ```

3. **Use Appropriate Warehouse Size:**
   ```sql
   -- Scale up warehouse for large operations
   ALTER WAREHOUSE COMPUTE_WH SET WAREHOUSE_SIZE = 'LARGE';
   
   -- Scale back down when done
   ALTER WAREHOUSE COMPUTE_WH SET WAREHOUSE_SIZE = 'SMALL';
   ```

#### Issue: High storage costs

**Symptoms:**
- Large storage usage in Snowflake
- Rapidly growing table sizes
- Duplicate or unnecessary data

**Solutions:**
1. **Implement Data Retention:**
   ```sql
   -- Archive old data
   CREATE TABLE braintrust_traces_archive AS
   SELECT * FROM braintrust_traces
   WHERE created < CURRENT_DATE() - 365;
   
   DELETE FROM braintrust_traces
   WHERE created < CURRENT_DATE() - 365;
   ```

2. **Remove Duplicates:**
   ```sql
   -- Find and remove duplicates
   DELETE FROM braintrust_traces
   WHERE (id, project_id, ingestion_timestamp) IN (
     SELECT id, project_id, ingestion_timestamp
     FROM (
       SELECT id, project_id, ingestion_timestamp,
              ROW_NUMBER() OVER (PARTITION BY id, project_id ORDER BY ingestion_timestamp DESC) as rn
       FROM braintrust_traces
     ) t
     WHERE rn > 1
   );
   ```

### 5. Monitoring and Alerting Issues

#### Issue: Missing or delayed alerts

**Symptoms:**
- No notifications when issues occur
- Alerts fire too frequently
- Monitoring queries return errors

**Solutions:**
1. **Fix Monitoring Queries:**
   ```sql
   -- Test health check procedure
   CALL check_ingestion_health();
   
   -- Check if monitoring views work
   SELECT * FROM MONITORING.ingestion_status LIMIT 5;
   ```

2. **Adjust Alert Thresholds:**
   ```sql
   -- Update stored procedure with better thresholds
   CREATE OR REPLACE PROCEDURE check_ingestion_health()
   RETURNS STRING
   LANGUAGE SQL
   AS
   $$
   DECLARE
     minutes_since_last_ingestion INTEGER;
     error_rate FLOAT;
     result STRING;
   BEGIN
     -- Check time since last successful ingestion
     SELECT DATEDIFF(minute, MAX(ingestion_timestamp), CURRENT_TIMESTAMP())
     INTO minutes_since_last_ingestion
     FROM braintrust_traces;
     
     -- Check error rate in last hour
     SELECT 
       (SUM(error_count)::FLOAT / NULLIF(SUM(row_count + error_count), 0)) * 100
     INTO error_rate
     FROM table(information_schema.copy_history(
       table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
       start_time => dateadd(hours, -1, current_timestamp())
     ));
     
     -- Build alert message
     IF minutes_since_last_ingestion > 60 THEN
       result := 'ALERT: No data ingested in ' || minutes_since_last_ingestion || ' minutes';
     ELSEIF error_rate > 5 THEN
       result := 'ALERT: High error rate: ' || error_rate || '%';
     ELSE
       result := 'OK: Pipeline healthy';
     END IF;
     
     RETURN result;
   END;
   $$;
   ```

## Diagnostic Queries

### Complete Pipeline Status

```sql
-- Comprehensive pipeline health check
WITH recent_activity AS (
  SELECT 
    COUNT(*) as records_last_hour,
    MAX(ingestion_timestamp) as last_ingestion,
    MIN(created) as earliest_created,
    MAX(created) as latest_created
  FROM braintrust_traces
  WHERE ingestion_timestamp >= CURRENT_TIMESTAMP() - INTERVAL '1 hour'
),
copy_stats AS (
  SELECT 
    SUM(row_count) as total_rows_copied,
    SUM(error_count) as total_errors,
    COUNT(*) as copy_operations
  FROM table(information_schema.copy_history(
    table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
    start_time => dateadd(hours, -24, current_timestamp())
  ))
),
pipe_status AS (
  SELECT SYSTEM$PIPE_STATUS('braintrust_traces_pipe') as pipe_status
)
SELECT 
  r.records_last_hour,
  r.last_ingestion,
  DATEDIFF(minute, r.last_ingestion, CURRENT_TIMESTAMP()) as minutes_since_last,
  c.total_rows_copied as rows_copied_24h,
  c.total_errors as errors_24h,
  c.copy_operations as operations_24h,
  p.pipe_status
FROM recent_activity r, copy_stats c, pipe_status p;
```

### Data Quality Check

```sql
-- Check data quality issues
SELECT 
  'null_ids' as issue_type,
  COUNT(*) as count
FROM braintrust_traces
WHERE id IS NULL

UNION ALL

SELECT 
  'null_timestamps' as issue_type,
  COUNT(*) as count
FROM braintrust_traces
WHERE created IS NULL

UNION ALL

SELECT 
  'future_timestamps' as issue_type,
  COUNT(*) as count
FROM braintrust_traces
WHERE created > CURRENT_TIMESTAMP()

UNION ALL

SELECT 
  'duplicate_ids' as issue_type,
  COUNT(*) as count
FROM (
  SELECT id, project_id
  FROM braintrust_traces
  GROUP BY id, project_id
  HAVING COUNT(*) > 1
);
```

## Emergency Procedures

### Pipeline Recovery

1. **Stop Ingestion:**
   ```sql
   ALTER PIPE braintrust_traces_pipe SET PIPE_EXECUTION_PAUSED = TRUE;
   ```

2. **Identify Issues:**
   - Check recent copy history for errors
   - Verify S3 file contents
   - Test stage and file format

3. **Fix and Resume:**
   ```sql
   -- After fixing issues
   ALTER PIPE braintrust_traces_pipe SET PIPE_EXECUTION_PAUSED = FALSE;
   ALTER PIPE braintrust_traces_pipe REFRESH;
   ```

### Data Corruption Recovery

1. **Backup Current State:**
   ```sql
   CREATE TABLE braintrust_traces_backup AS
   SELECT * FROM braintrust_traces;
   ```

2. **Identify Corrupted Data:**
   ```sql
   -- Find problematic records
   SELECT * FROM braintrust_traces
   WHERE ingestion_timestamp >= 'PROBLEM_START_TIME'
     AND ingestion_timestamp <= 'PROBLEM_END_TIME';
   ```

3. **Remove and Re-ingest:**
   ```sql
   -- Remove corrupted data
   DELETE FROM braintrust_traces
   WHERE ingestion_timestamp >= 'PROBLEM_START_TIME'
     AND ingestion_timestamp <= 'PROBLEM_END_TIME';
   
   -- Re-run copy for specific files
   COPY INTO braintrust_traces
   FROM @s3_braintrust_stage/2024/01/15/
   FILE_FORMAT = json_format
   ON_ERROR = CONTINUE;
   ```

## Getting Help

### Braintrust Support
- Documentation: [Braintrust Docs](https://braintrust.dev/docs)
- Support: support@braintrust.dev
- Check automation logs in Braintrust UI

### AWS Support
- Check CloudTrail for API calls
- Review IAM policy simulator
- AWS Support (if you have a support plan)

### Snowflake Support
- Query history in Snowflake UI
- Information schema functions
- Snowflake Documentation: https://docs.snowflake.com/

### Log Files and Monitoring

Always check these sources when troubleshooting:
- Braintrust automation run history
- AWS CloudTrail logs
- Snowflake query history
- S3 access logs (if enabled)
- Copy history and pipe usage history in Snowflake

---

## 📖 Documentation Navigation

**← Previous:** [Testing the Complete Pipeline](./testing.md) | **🏠 Back to Start:** [Getting Started](../README.md)

### Complete Setup Guide:
1. [Getting Started](../README.md)
2. [AWS Infrastructure Setup](./aws-setup.md)
3. [Braintrust S3 Export Setup](./braintrust-setup.md)
4. [Snowflake and Snowpipe Setup](./snowflake-setup.md)
5. [Testing the Complete Pipeline](./testing.md)
6. **Troubleshooting Guide** (You are here) 
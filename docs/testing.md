# Testing the Complete Pipeline

This guide provides comprehensive testing procedures to validate your Braintrust to Snowflake data pipeline is working correctly.

## Prerequisites

- Completed [AWS Setup](./aws-setup.md)
- Completed [Braintrust Setup](./braintrust-setup.md) 
- Completed [Snowflake Setup](./snowflake-setup.md)
- Test data available in Braintrust

## Testing Strategy

We'll test the pipeline in phases:
1. **Component Testing**: Test each component individually
2. **Integration Testing**: Test the complete end-to-end flow
3. **Performance Testing**: Validate performance under load
4. **Error Handling**: Test error scenarios and recovery

## Phase 1: Component Testing

### 1.1 Test AWS Infrastructure

#### Test S3 Bucket Access

```bash
# Test Braintrust role assumption (replace with your values)
aws sts assume-role \
  --role-arn arn:aws:iam::YOUR_ACCOUNT_ID:role/BraintrustExportRole \
  --role-session-name test-session \
  --external-id bt:YOUR_ORG_ID:test

# Export temporary credentials and test S3 access
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

# Test write access
echo "test content" > test-file.txt
aws s3 cp test-file.txt s3://your-bucket/test/
aws s3 rm s3://your-bucket/test/test-file.txt
```

#### Test Snowflake S3 Access

```bash
# Test with Snowflake IAM user credentials
aws configure set aws_access_key_id YOUR_SNOWFLAKE_ACCESS_KEY --profile snowflake
aws configure set aws_secret_access_key YOUR_SNOWFLAKE_SECRET_KEY --profile snowflake

# Test read access
aws s3 ls s3://your-bucket/ --profile snowflake
```

### 1.2 Test Braintrust Export

#### Manual Trigger Test

1. Go to Braintrust → Configuration → Automations
2. Find your S3 export automation
3. Click **Test automation**
4. Verify the test completes successfully
5. Check S3 bucket for test files

#### Verify Export Configuration

```yaml
# In Braintrust, test your BTQL query if using custom query
select: *
from: project_logs('<PROJECT_ID>')
filter: created >= '2024-01-01'
limit: 10
```

### 1.3 Test Snowflake Components

#### Test Storage Integration

```sql
-- Test storage integration
DESC STORAGE INTEGRATION s3_braintrust_integration;

-- Should show ENABLED = true and no errors
```

#### Test External Stage

```sql
-- List files in stage
LIST @s3_braintrust_stage;

-- Test reading from stage
SELECT $1 FROM @s3_braintrust_stage LIMIT 1;
```

#### Test File Format

```sql
-- Test JSON parsing
SELECT 
  $1:id::STRING as id,
  $1:project_id::STRING as project_id,
  $1:created::TIMESTAMP_NTZ as created
FROM @s3_braintrust_stage 
LIMIT 5;
```

#### Test Manual COPY

```sql
-- Test manual copy into table
COPY INTO braintrust_traces
FROM @s3_braintrust_stage
FILE_FORMAT = json_format
ON_ERROR = CONTINUE;

-- Check results
SELECT COUNT(*) FROM braintrust_traces;
SELECT * FROM braintrust_traces LIMIT 3;
```

## Phase 2: Integration Testing

### 2.1 End-to-End Pipeline Test

#### Step 1: Generate Test Data in Braintrust

Create test data in your Braintrust project:

```python
# Example test script for Braintrust
import braintrust

project = braintrust.init(project="test-pipeline")

# Log some test traces
project.log(
    input="What is the capital of France?",
    output="The capital of France is Paris.",
    metadata={"test": True, "environment": "pipeline-test"},
    scores={"Factuality": 0.95, "Helpfulness": 0.9}
)

project.log(
    input="Explain quantum computing",
    output="Quantum computing uses quantum bits...",
    metadata={"test": True, "environment": "pipeline-test"},
    scores={"Factuality": 0.88, "Helpfulness": 0.85}
)
```

#### Step 2: Trigger Braintrust Export

1. Manually trigger the Braintrust automation
2. Wait for export to complete (check automation status)
3. Verify files appear in S3

```bash
# Check S3 for new files
aws s3 ls s3://your-bucket/braintrust-exports/ --recursive
```

#### Step 3: Verify Snowpipe Ingestion

```sql
-- Check Snowpipe status
SELECT SYSTEM$PIPE_STATUS('braintrust_traces_pipe');

-- Check recent copy history
SELECT *
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -1, current_timestamp())
))
ORDER BY last_loaded_time DESC;

-- Verify test data appears in table
SELECT *
FROM braintrust_traces
WHERE metadata:test = true
ORDER BY ingestion_timestamp DESC;
```

### 2.2 Test Data Quality

#### Verify Data Integrity

```sql
-- Check for null IDs (should be 0)
SELECT COUNT(*) as null_ids
FROM braintrust_traces
WHERE id IS NULL;

-- Check timestamp parsing
SELECT 
  COUNT(*) as total_records,
  COUNT(CASE WHEN created IS NULL THEN 1 END) as null_timestamps,
  MIN(created) as earliest,
  MAX(created) as latest
FROM braintrust_traces
WHERE ingestion_timestamp >= CURRENT_TIMESTAMP() - INTERVAL '1 hour';

-- Verify JSON structure
SELECT 
  COUNT(*) as total,
  COUNT(CASE WHEN metadata IS NOT NULL THEN 1 END) as has_metadata,
  COUNT(CASE WHEN scores IS NOT NULL THEN 1 END) as has_scores
FROM braintrust_traces
WHERE ingestion_timestamp >= CURRENT_TIMESTAMP() - INTERVAL '1 hour';
```

#### Test Data Consistency

```sql
-- Compare record counts between original and ingested data
-- This requires knowing expected counts from Braintrust

-- Check for duplicate records
SELECT id, project_id, COUNT(*)
FROM braintrust_traces
GROUP BY id, project_id
HAVING COUNT(*) > 1;
```

### 2.3 Test Real-time Ingestion

#### Create Continuous Test

```python
# Script to continuously generate test data
import braintrust
import time
import random

project = braintrust.init(project="test-pipeline")

for i in range(10):
    project.log(
        input=f"Test query {i}",
        output=f"Test response {i}",
        metadata={
            "test": True, 
            "batch": "continuous-test",
            "iteration": i
        },
        scores={"Factuality": random.uniform(0.7, 1.0)}
    )
    time.sleep(30)  # Wait 30 seconds between logs
```

#### Monitor Real-time Ingestion

```sql
-- Monitor ingestion in real-time
SELECT 
  COUNT(*) as total_records,
  MAX(ingestion_timestamp) as latest_ingestion
FROM braintrust_traces
WHERE metadata:batch = 'continuous-test';

-- Check ingestion latency
SELECT 
  AVG(DATEDIFF(second, created, ingestion_timestamp)) as avg_latency_seconds,
  MAX(DATEDIFF(second, created, ingestion_timestamp)) as max_latency_seconds
FROM braintrust_traces
WHERE metadata:batch = 'continuous-test'
  AND ingestion_timestamp >= CURRENT_TIMESTAMP() - INTERVAL '1 hour';
```

## Phase 3: Performance Testing

### 3.1 Load Testing

#### High Volume Export Test

Create a large batch of test data in Braintrust:

```python
import braintrust
import concurrent.futures
import threading

def create_batch(batch_id, size):
    project = braintrust.init(project="test-pipeline")
    for i in range(size):
        project.log(
            input=f"Load test query {batch_id}-{i}",
            output=f"Load test response {batch_id}-{i}",
            metadata={
                "test": True,
                "load_test": True,
                "batch_id": batch_id
            },
            scores={"Factuality": 0.9}
        )

# Create 10 batches of 100 records each in parallel
with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(create_batch, i, 100) for i in range(10)]
    concurrent.futures.wait(futures)
```

#### Monitor Performance

```sql
-- Monitor ingestion performance
SELECT 
  DATE_TRUNC('minute', ingestion_timestamp) as minute,
  COUNT(*) as records_per_minute
FROM braintrust_traces
WHERE metadata:load_test = true
GROUP BY DATE_TRUNC('minute', ingestion_timestamp)
ORDER BY minute;

-- Check for any failed ingestions
SELECT *
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -2, current_timestamp())
))
WHERE error_count > 0;
```

### 3.2 Query Performance Testing

```sql
-- Test query performance on large dataset
SELECT COUNT(*) FROM braintrust_traces; -- Baseline count

-- Test aggregation queries
SELECT 
  project_id,
  DATE(created) as date,
  COUNT(*) as daily_count,
  AVG(CAST(scores:Factuality AS FLOAT)) as avg_score
FROM braintrust_traces
WHERE created >= CURRENT_DATE() - 7
GROUP BY project_id, DATE(created)
ORDER BY date DESC, project_id;

-- Test filtering queries
SELECT *
FROM braintrust_traces
WHERE metadata:environment = 'production'
  AND scores:Factuality < 0.8
  AND created >= CURRENT_DATE() - 1;
```

## Phase 4: Error Handling Tests

### 4.1 Test Invalid Data Handling

#### Create Malformed Test File

```bash
# Create a test file with invalid JSON
cat > /tmp/invalid-test.jsonl << 'EOF'
{"id": "test1", "project_id": "proj1", "created": "2024-01-15T10:00:00Z"}
invalid json line
{"id": "test2", "project_id": "proj1", "created": "invalid-date"}
{"id": "test3", "project_id": "proj1", "created": "2024-01-15T10:02:00Z"}
EOF

# Upload to S3
aws s3 cp /tmp/invalid-test.jsonl s3://your-bucket/braintrust-exports/test/
```

#### Verify Error Handling

```sql
-- Check copy history for errors
SELECT *
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -1, current_timestamp())
))
WHERE error_count > 0;

-- Verify good records were still processed
SELECT COUNT(*)
FROM braintrust_traces
WHERE id IN ('test1', 'test3');
```

### 4.2 Test Network/Connection Issues

#### Test S3 Access Failure

```sql
-- Temporarily modify stage URL to test error handling
CREATE OR REPLACE STAGE s3_test_stage
  STORAGE_INTEGRATION = s3_braintrust_integration
  URL = 's3://nonexistent-bucket/test/'
  FILE_FORMAT = json_format;

-- Try to list files (should fail)
LIST @s3_test_stage;

-- Restore original stage
DROP STAGE s3_test_stage;
```

### 4.3 Test Snowpipe Recovery

```sql
-- Check if pipes recover from temporary failures
SELECT SYSTEM$PIPE_STATUS('braintrust_traces_pipe');

-- Manually refresh pipe if needed
ALTER PIPE braintrust_traces_pipe REFRESH;

-- Check pipe queue status
SELECT *
FROM table(information_schema.pipe_usage_history(
  date_range_start => dateadd(hours, -2, current_timestamp()),
  date_range_end => current_timestamp()
))
WHERE pipe_name = 'BRAINTRUST_TRACES_PIPE';
```

## Phase 5: Monitoring and Alerting Tests

### 5.1 Test Monitoring Queries

```sql
-- Test health check procedure
CALL check_ingestion_health();

-- Test monitoring views
SELECT * FROM MONITORING.ingestion_status;
SELECT * FROM MONITORING.pipe_status;
SELECT * FROM RAW_DATA.data_quality_summary;
```

### 5.2 Test Alerting Thresholds

```sql
-- Simulate conditions that should trigger alerts

-- Check for stalled pipes
SELECT *
FROM table(information_schema.pipe_usage_history(
  date_range_start => dateadd(hours, -24, current_timestamp()),
  date_range_end => current_timestamp()
))
WHERE is_stalled = TRUE;

-- Check for high error rates
SELECT 
  file_name,
  error_count,
  row_count,
  (error_count::FLOAT / NULLIF(row_count, 0)) * 100 as error_percentage
FROM table(information_schema.copy_history(
  table_name => 'BRAINTRUST_DATA.RAW_DATA.BRAINTRUST_TRACES',
  start_time => dateadd(hours, -24, current_timestamp())
))
WHERE error_count > 0;
```

## Testing Checklist

### Pre-Test Setup
- [ ] AWS infrastructure configured
- [ ] Braintrust automation created
- [ ] Snowflake tables and pipes created
- [ ] Test data available in Braintrust

### Component Tests
- [ ] S3 bucket accessible by Braintrust role
- [ ] S3 bucket accessible by Snowflake
- [ ] Braintrust automation test passes
- [ ] Snowflake stage can list S3 files
- [ ] Manual COPY command works
- [ ] File format parsing works correctly

### Integration Tests
- [ ] End-to-end data flow works
- [ ] Data appears in Snowflake within expected time
- [ ] Data integrity maintained
- [ ] No duplicate records
- [ ] JSON fields parsed correctly

### Performance Tests
- [ ] High volume ingestion works
- [ ] Query performance acceptable
- [ ] No bottlenecks identified
- [ ] Latency within acceptable limits

### Error Handling Tests
- [ ] Invalid data handled gracefully
- [ ] Connection failures recover
- [ ] Snowpipe resumes after interruption
- [ ] Error logging works correctly

### Monitoring Tests
- [ ] Health checks work
- [ ] Monitoring views return expected data
- [ ] Alerting conditions detected
- [ ] Performance metrics collected

## Cleanup Test Data

After testing, clean up test data:

```sql
-- Remove test records
DELETE FROM braintrust_traces WHERE metadata:test = true;
DELETE FROM braintrust_spans WHERE metadata:test = true;

-- Clean up test files from S3
```

```bash
aws s3 rm s3://your-bucket/braintrust-exports/test/ --recursive
```

## Performance Benchmarks

Document your performance baselines:

| Metric | Expected Value | Actual Value | Status |
|--------|----------------|--------------|---------|
| Export Latency | < 5 minutes | ___ | ✅/❌ |
| Ingestion Latency | < 2 minutes | ___ | ✅/❌ |
| Query Response Time | < 30 seconds | ___ | ✅/❌ |
| Error Rate | < 1% | ___ | ✅/❌ |
| Throughput | > 1000 records/min | ___ | ✅/❌ |

## Next Steps

After successful testing:
1. Set up production monitoring
2. Configure alerting rules
3. Create operational runbooks
4. Schedule regular testing
5. Document any performance optimizations needed

## Troubleshooting Common Test Issues

### Braintrust Export Not Working
- Check IAM role ARN and external ID
- Verify S3 bucket permissions
- Check automation configuration

### Snowpipe Not Ingesting
- Verify S3 event notifications configured
- Check Snowpipe status and queue
- Ensure file format matches data

### Data Quality Issues
- Check JSON structure in source files
- Verify timestamp formats
- Review COPY command column mappings

### Performance Issues
- Check warehouse size in Snowflake
- Review table clustering
- Monitor concurrent operations

---

## 📖 Documentation Navigation

**← Previous:** [Snowflake and Snowpipe Setup](./snowflake-setup.md) | **Next:** [Troubleshooting Guide →](./troubleshooting.md)

### Complete Setup Guide:
1. [Getting Started](../README.md)
2. [AWS Infrastructure Setup](./aws-setup.md)
3. [Braintrust S3 Export Setup](./braintrust-setup.md)
4. [Snowflake and Snowpipe Setup](./snowflake-setup.md)
5. **Testing the Complete Pipeline** (You are here)
6. [Troubleshooting Guide](./troubleshooting.md) 
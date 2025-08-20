# Braintrust S3 Export Setup

This guide walks you through setting up an S3 Export automation in Braintrust to export your project data to an AWS S3 bucket.

## Prerequisites

- Braintrust account with a project containing data
- AWS account with S3 and IAM access
- Automations feature enabled (available starting with v0.0.72 for hybrid deployments)

## Step 1: Prepare AWS Infrastructure

Before setting up the Braintrust automation, you need to:

1. Create an S3 bucket for data exports
2. Create an IAM role that Braintrust can assume
3. Configure proper permissions

See [AWS Setup Guide](./aws-setup.md) for detailed instructions.

## Step 2: Navigate to Automations

1. Log into your Braintrust account
2. Navigate to your project
3. Go to **Configuration → Automations**
4. Click **Add automation**

## Step 3: Configure S3 Export Automation

### Basic Settings

- **Name**: Choose a descriptive name (e.g., "Daily Data Export to S3")
- **Description**: Optional description explaining the automation's purpose
- **Event type**: Select **"BTQL export"**

### Data Export Configuration

Choose what data to export:

#### Option 1: Logs (traces)
- **Use case**: High-level trace data with aggregated metrics
- **Data included**: One row per trace, scores, token counts, cost, and other metrics
- **Recommended for**: Executive dashboards, cost analysis

#### Option 2: Logs (spans)  
- **Use case**: Detailed span-level data
- **Data included**: One row per span (lower level granularity)
- **Recommended for**: Detailed performance analysis, debugging

#### Option 3: Custom BTQL query
- **Use case**: Specific data requirements
- **Configuration**: Write custom BTQL query to define exact data to export
- **Example queries**:
  ```yaml
  # Export only logs with scores below threshold
  select: *
  from: project_logs('<PROJECT_ID>')
  filter: scores.Factuality < 0.8
  
  # Export logs from specific time range
  select: *
  from: project_logs('<PROJECT_ID>')
  filter: created >= '2024-01-01'
  
  # Export logs with specific metadata
  select: *
  from: project_logs('<PROJECT_ID>')
  filter: metadata.environment = 'production'
  
  # Export recent logs with low scores and high token usage
  select: *
  from: project_logs('<PROJECT_ID>')
  filter: 
    created > now() - interval 7 day and
    scores.Factuality < 0.8 and
    metrics.total_tokens > 1000
  
  # Export specific fields for cost analysis
  select:
    metadata.model as model,
    metrics.total_cost as cost,
    metrics.total_tokens as tokens,
    created as timestamp
  from: project_logs('<PROJECT_ID>')
  filter: metadata.environment = 'production'
  sort: created desc
  limit: 10000
  ```

### S3 Configuration

- **S3 export path**: Enter your S3 path (e.g., `s3://your-bucket-name/braintrust-exports/`)
  - ⚠️ **Important**: This path cannot be changed after creation
  - Consider including date/time patterns for organization
- **Role ARN**: Enter the ARN of the IAM role you created (e.g., `arn:aws:iam::123456789012:role/BraintrustExportRole`)

### Export Settings

- **Format**: Choose between:
  - **JSON Lines**: Human-readable, easier to debug
  - **Parquet**: More efficient for large datasets, better for analytics
- **Interval**: Select export frequency:
  - **Hourly**: For real-time analytics
  - **Daily**: Most common choice
  - **Weekly**: For archival purposes
- **Batch size** (Advanced): Number of items per batch (default: 1,000)

## Step 4: Test the Automation

1. Click **Test automation** before saving
2. Braintrust will attempt to write and delete a test file to your S3 bucket
3. Review any error messages and fix configuration issues
4. Ensure the test completes successfully

## Step 5: Save and Monitor

1. Click **Save** to create the automation
2. Monitor the automation runs in the Braintrust UI
3. Check your S3 bucket for exported files

## File Organization

Exported files will be organized in your S3 bucket as:

```
s3://your-bucket-name/braintrust-exports/
├── 2024/01/15/
│   ├── export_2024-01-15_00-00-00.jsonl
│   ├── export_2024-01-15_01-00-00.jsonl
│   └── ...
├── 2024/01/16/
│   └── ...
```

## Data Schema

### Logs (traces) Export Schema

```json
{
  "id": "trace_123",
  "project_id": "proj_456", 
  "created": "2024-01-15T10:30:00.000Z",
  "metadata": {
    "environment": "production",
    "user_id": "user_789"
  },
  "scores": {
    "Factuality": 0.95,
    "Helpfulness": 0.87
  },
  "metrics": {
    "total_tokens": 1250,
    "total_cost": 0.025
  },
  "input": "User query text",
  "output": "Model response text"
}
```

### Logs (spans) Export Schema

```json
{
  "id": "span_123",
  "trace_id": "trace_456",
  "parent_span_id": "span_789",
  "project_id": "proj_456",
  "created": "2024-01-15T10:30:00.000Z",
  "name": "llm_call",
  "type": "llm",
  "input": {"prompt": "..."},
  "output": {"text": "..."},
  "metrics": {
    "tokens": 500,
    "cost": 0.01,
    "latency": 1.2
  }
}
```

## Manual Triggers

You can manually trigger the automation from the Braintrust UI:

1. Go to **Configuration → Automations**
2. Find your automation
3. Click on the status/details
4. Click **Trigger manually**

## Monitoring and Troubleshooting

### Check Automation Status
- View run history in the Braintrust UI
- Monitor for failed runs and error messages
- Check AWS CloudTrail for assume role attempts

### Common Issues
- **IAM Permission Errors**: Verify role ARN and trust policy
- **S3 Access Denied**: Check bucket permissions and policies
- **No Data Exported**: Verify BTQL filter and data availability
- **Large Export Timeouts**: Consider reducing batch size

## Next Steps

After setting up Braintrust export:
1. Verify data is appearing in your S3 bucket
2. Set up [Snowflake and Snowpipe](./snowflake-setup.md) to ingest the data
3. [Test the complete pipeline](./testing.md)

## Security Best Practices

- Use least-privilege IAM policies
- Enable S3 bucket versioning and lifecycle policies
- Monitor access logs
- Regularly rotate access keys if using IAM users
- Consider encrypting S3 bucket with KMS

---

## 📖 Documentation Navigation

**← Previous:** [AWS Infrastructure Setup](./aws-setup.md) | **Next:** [Snowflake and Snowpipe Setup →](./snowflake-setup.md)

### Complete Setup Guide:
1. [Getting Started](../README.md)
2. [AWS Infrastructure Setup](./aws-setup.md)
3. **Braintrust S3 Export Setup** (You are here)
4. [Snowflake and Snowpipe Setup](./snowflake-setup.md)
5. [Testing the Complete Pipeline](./testing.md)
6. [Troubleshooting Guide](./troubleshooting.md) 
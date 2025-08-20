# AWS Infrastructure Setup

This guide covers setting up the AWS infrastructure required for the Braintrust to Snowflake pipeline, including S3 bucket and IAM roles for both Braintrust exports and Snowflake access.

## Prerequisites

- AWS account with administrative access
- AWS CLI installed and configured (optional but recommended)
- Basic understanding of IAM roles and S3

## Step 1: Create S3 Bucket

### Using AWS Console

1. Go to the [S3 Console](https://console.aws.amazon.com/s3/)
2. Click **Create bucket**
3. Configure the bucket:
   - **Bucket name**: Choose a unique name (e.g., `your-company-braintrust-exports`)
   - **Region**: Select the same region as your Snowflake account for optimal performance
   - **Block Public Access**: Keep all settings enabled (recommended)
   - **Versioning**: Enable (recommended for data recovery)
   - **Encryption**: Enable with S3 managed keys or KMS

### Using AWS CLI

```bash
# Create the bucket
aws s3 mb s3://your-company-braintrust-exports --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket your-company-braintrust-exports \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket your-company-braintrust-exports \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

## Step 2: Create IAM Role for Braintrust

Braintrust needs an IAM role it can assume to write data to your S3 bucket.

### Create Trust Policy

Create a file `braintrust-trust-policy.json`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
              "s3:GetObject",
              "s3:GetObjectVersion"
            ],
            "Resource": "arn:aws:s3:::<bucket>/<prefix>/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::<bucket>",
            "Condition": {
                "StringLike": {
                    "s3:prefix": [
                        "<prefix>/*"
                    ]
                }
            }
        }
    ]
}
```

**Important**: Replace `YOUR_ORG_ID` with your actual Braintrust organization ID.

### Create Permission Policy

Create a file `braintrust-s3-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObject",
        "s3:GetObjectAcl",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::your-company-braintrust-exports/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::your-company-braintrust-exports"
    }
  ]
}
```

### Create the Role

Using AWS CLI:

```bash
# Create the role with trust policy
aws iam create-role \
  --role-name BraintrustExportRole \
  --assume-role-policy-document file://braintrust-trust-policy.json

# Create the permission policy
aws iam create-policy \
  --policy-name BraintrustS3ExportPolicy \
  --policy-document file://braintrust-s3-policy.json

# Attach the policy to the role
aws iam attach-role-policy \
  --role-name BraintrustExportRole \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/BraintrustS3ExportPolicy
```

Using AWS Console:

1. Go to [IAM Console](https://console.aws.amazon.com/iam/)
2. Click **Roles** → **Create role**
3. Select **Custom trust policy** and paste the trust policy JSON
4. Click **Next** and create a new policy with the permission JSON
5. Attach the policy to the role
6. Name the role `BraintrustExportRole`

## Step 3: Create IAM User for Snowflake

Snowflake will need access to read from your S3 bucket. You can use either an IAM role or IAM user. This example uses an IAM user.

### Create Permission Policy for Snowflake

Create a file `snowflake-s3-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::your-company-braintrust-exports/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::your-company-braintrust-exports",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["*"]
        }
      }
    }
  ]
}
```

### Create the IAM User

Using AWS CLI:

```bash
# Create the user
aws iam create-user --user-name SnowflakeS3User

# Create the policy
aws iam create-policy \
  --policy-name SnowflakeS3AccessPolicy \
  --policy-document file://snowflake-s3-policy.json

# Attach policy to user
aws iam attach-user-policy \
  --user-name SnowflakeS3User \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/SnowflakeS3AccessPolicy

# Create access keys
aws iam create-access-key --user-name SnowflakeS3User
```

**Important**: Save the access key ID and secret access key - you'll need them for Snowflake configuration.

## Step 4: Configure S3 Event Notifications (Optional)

For real-time Snowpipe ingestion, you can configure S3 to send notifications when new files are uploaded.

### Create SQS Queue for Snowpipe

```bash
# Create SQS queue
aws sqs create-queue \
  --queue-name snowpipe-s3-notifications \
  --attributes '{
    "MessageRetentionPeriod": "1209600",
    "VisibilityTimeoutSeconds": "60"
  }'

# Get queue ARN
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/YOUR_ACCOUNT_ID/snowpipe-s3-notifications \
  --attribute-names QueueArn
```

### Configure S3 Event Notifications

Add this configuration to your S3 bucket:

```json
{
  "QueueConfigurations": [
    {
      "Id": "SnowpipeNotification",
      "QueueArn": "arn:aws:sqs:us-east-1:YOUR_ACCOUNT_ID:snowpipe-s3-notifications",
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

## Step 5: Set Up Cross-Account Access (If Needed)

If your Snowflake account is in a different AWS account, you'll need to set up cross-account access.

### Update S3 Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSnowflakeAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::SNOWFLAKE_ACCOUNT_ID:user/SnowflakeS3User"
      },
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-company-braintrust-exports",
        "arn:aws:s3:::your-company-braintrust-exports/*"
      ]
    }
  ]
}
```

## Step 6: Test Access

### Test Braintrust Role

```bash
# Assume the Braintrust role (for testing)
aws sts assume-role \
  --role-arn arn:aws:iam::YOUR_ACCOUNT_ID:role/BraintrustExportRole \
  --role-session-name test-session \
  --external-id bt:YOUR_ORG_ID:test

# Test S3 write access (using temporary credentials)
aws s3 cp test-file.txt s3://your-company-braintrust-exports/test/
```

### Test Snowflake User Access

```bash
# Configure AWS CLI with Snowflake user credentials
aws configure --profile snowflake

# Test S3 read access
aws s3 ls s3://your-company-braintrust-exports/ --profile snowflake
```

## Security Best Practices

### IAM Policies
- Use least-privilege principle
- Regularly review and audit permissions
- Use conditions to restrict access further
- Enable CloudTrail for audit logging

### S3 Security
- Enable bucket versioning
- Configure lifecycle policies to manage costs
- Use S3 encryption (SSE-S3 or SSE-KMS)
- Monitor access with CloudTrail and S3 access logs
- Consider using S3 Object Lock for compliance

### Key Management
- Rotate access keys regularly
- Use IAM roles instead of users when possible
- Store credentials securely (AWS Secrets Manager)
- Monitor for unused access keys

## Cost Optimization

### S3 Storage Classes
- Use S3 Intelligent-Tiering for automatic cost optimization
- Set up lifecycle policies to transition older data to cheaper storage classes
- Delete incomplete multipart uploads

### Example Lifecycle Policy
```json
{
  "Rules": [
    {
      "Id": "BraintrustDataLifecycle",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "braintrust-exports/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ]
    }
  ]
}
```

## Monitoring and Alerting

Set up CloudWatch alarms for:
- Failed S3 operations
- Unusual access patterns
- Storage costs exceeding thresholds
- SQS queue depth (if using event notifications)

## Next Steps

After completing AWS setup:
1. Note down the following values for later use:
   - S3 bucket name
   - Braintrust IAM role ARN
   - Snowflake IAM user access keys
   - SQS queue ARN (if configured)
2. Proceed to [Braintrust Setup](./braintrust-setup.md)
3. Then configure [Snowflake and Snowpipe](./snowflake-setup.md)

---

## 📖 Documentation Navigation

**← Previous:** [Getting Started](../README.md) | **Next:** [Braintrust S3 Export Setup →](./braintrust-setup.md)

### Complete Setup Guide:
1. [Getting Started](../README.md)
2. **AWS Infrastructure Setup** (You are here)
3. [Braintrust S3 Export Setup](./braintrust-setup.md)
4. [Snowflake and Snowpipe Setup](./snowflake-setup.md)
5. [Testing the Complete Pipeline](./testing.md)
6. [Troubleshooting Guide](./troubleshooting.md) 
# Braintrust to Snowflake Export Pipeline

This repository contains documentation and configuration files for setting up an automated data pipeline that exports data from Braintrust to AWS S3 and then ingests it into Snowflake using Snowpipe.

## Overview

The pipeline consists of two main components:

1. **Braintrust S3 Export Automation**: Automatically exports Braintrust data (logs, traces, spans) to an AWS S3 bucket
2. **Snowflake Snowpipe**: Continuously monitors the S3 bucket and automatically ingests new data into Snowflake tables

## Architecture

```
Braintrust → S3 Bucket → Snowpipe → Snowflake
```

- **Braintrust**: Source system containing logs, traces, and evaluation data
- **S3 Bucket**: Intermediate storage for exported data files (JSON or Parquet format)
- **Snowpipe**: Snowflake's continuous data ingestion service
- **Snowflake**: Target data warehouse for analytics and reporting

## Documentation

- [`docs/braintrust-setup.md`](./docs/braintrust-setup.md) - Complete guide for setting up Braintrust S3 exports
- [`docs/snowflake-setup.md`](./docs/snowflake-setup.md) - Snowflake and Snowpipe configuration
- [`docs/aws-setup.md`](./docs/aws-setup.md) - AWS IAM roles and S3 bucket configuration
- [`docs/testing.md`](./docs/testing.md) - Testing and validation procedures
- [`docs/troubleshooting.md`](./docs/troubleshooting.md) - Common issues and solutions

## Prerequisites

- Braintrust account with access to automations
- AWS account with S3 and IAM permissions
- Snowflake account with appropriate privileges
- Basic understanding of SQL and cloud services

## Support

For issues related to:
- **Braintrust**: [Braintrust Support](mailto:support@braintrust.dev)
- **Snowflake**: [Snowflake Documentation](https://docs.snowflake.com/)
- **AWS**: [AWS Support](https://aws.amazon.com/support/)

## License

This documentation is provided as-is for educational and implementation purposes.

---

## 📖 Documentation Navigation

**Next:** [AWS Infrastructure Setup →](./docs/aws-setup.md)

### Complete Setup Guide:
1. **Getting Started** (You are here)
2. [AWS Infrastructure Setup](./docs/aws-setup.md)
3. [Braintrust S3 Export Setup](./docs/braintrust-setup.md)
4. [Snowflake and Snowpipe Setup](./docs/snowflake-setup.md)
5. [Testing the Complete Pipeline](./docs/testing.md)
6. [Troubleshooting Guide](./docs/troubleshooting.md) 
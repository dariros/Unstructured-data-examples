# Unstructured Data in Snowflake 101 - Setup Guide

This notebook demonstrates how to work with unstructured data in Snowflake, including document processing with AI_EXTRACT, automated pipelines with Tasks and Streams, and interactive PDF viewing with Streamlit.

## 📋 Prerequisites

### Snowflake Account Requirements
- **Snowflake Account** with Cortex AI functions enabled
- **Account Role**: `SYSADMIN` or equivalent with stage and AI function privileges
- **Cortex AI Access**: Ensure your account has access to Snowflake Cortex functions

### Required Packages
Add these packages to your Snowflake notebook environment:
```
pypdfium2
```

## 🏗️ Setup Instructions

### 1. Database and Schema Setup
```sql
-- Create database and schema (adjust names as needed)
CREATE DATABASE IF NOT EXISTS ADVANCED_ANALYTICS;
CREATE SCHEMA IF NOT EXISTS ADVANCED_ANALYTICS.UNSTRUCTURED;
USE SCHEMA ADVANCED_ANALYTICS.UNSTRUCTURED;
```

### 2. Stage Configuration

#### Option A: Internal Stage (Recommended for Demo)
```sql
CREATE OR REPLACE STAGE ADVANCED_ANALYTICS.UNSTRUCTURED.PERMITS_S3
  DIRECTORY = (
    ENABLE = true
    AUTO_REFRESH = true
  )
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  COMMENT = 'Internal stage for unstructured data demo';
```

#### Option B: External S3 Stage (Production Setup)

**Step 1: Create Storage Integration (requires ACCOUNTADMIN role)**
```sql
-- Create storage integration for secure S3 access
CREATE OR REPLACE STORAGE INTEGRATION unstructured_demo_s3_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR_AWS_ACCOUNT:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-bucket-name/unstructured-demo/');

-- Get the IAM user and external ID for AWS setup
DESC INTEGRATION unstructured_demo_s3_integration;
```

**Step 2: AWS IAM Configuration**

Create an IAM role in AWS with the following trust policy (replace `SNOWFLAKE_USER_ARN` and `EXTERNAL_ID` from the DESC command above):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "SNOWFLAKE_USER_ARN"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "EXTERNAL_ID"
        }
      }
    }
  ]
}
```

And attach this permissions policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "arn:aws:s3:::your-bucket-name/unstructured-demo/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::your-bucket-name",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["unstructured-demo/*"]
        }
      }
    }
  ]
}
```

**Step 3: Create External Stage**
```sql
-- Create external S3 stage using the storage integration
CREATE OR REPLACE STAGE ADVANCED_ANALYTICS.UNSTRUCTURED.PERMITS_S3
  URL = 's3://your-bucket-name/unstructured-demo/'
  STORAGE_INTEGRATION = unstructured_demo_s3_integration
  DIRECTORY = (
    ENABLE = true
    AUTO_REFRESH = true
  )
  COMMENT = 'External S3 stage for unstructured data demo';

-- Test the stage setup
LIST @ADVANCED_ANALYTICS.UNSTRUCTURED.PERMITS_S3;
```

### 3. Sample Data Upload

Upload sample PDF files to your stage:

**For Internal Stage:**
```sql
-- Upload files using Snowsight UI or SnowSQL
PUT file://path/to/your/pdfs/*.pdf @ADVANCED_ANALYTICS.UNSTRUCTURED.PERMITS_S3;
```

**For External Stage:**
- Upload PDF files directly to your S3 bucket path
- Run `ALTER STAGE ADVANCED_ANALYTICS.UNSTRUCTURED.PERMITS_S3 REFRESH;` to update directory

### 4. Permissions Setup
```sql
-- Grant necessary privileges to your role
GRANT USAGE ON DATABASE ADVANCED_ANALYTICS TO ROLE YOUR_ROLE;
GRANT USAGE ON SCHEMA ADVANCED_ANALYTICS.UNSTRUCTURED TO ROLE YOUR_ROLE;
GRANT READ ON STAGE ADVANCED_ANALYTICS.UNSTRUCTURED.PERMITS_S3 TO ROLE YOUR_ROLE;

-- Ensure Cortex AI access (may require ACCOUNTADMIN)
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE YOUR_ROLE;
```

## 🎯 Quick Start

1. **Verify Setup**: Run the stage description query in Cell 4 to confirm your stage is configured correctly
2. **Upload Sample Files**: Add 3-5 PDF documents to your stage for testing
3. **Test AI_EXTRACT**: Execute Cell 14 to verify AI functions are working
4. **Explore Interactive Features**: Run the Streamlit app in Cell 12 for PDF viewing

## 📄 Sample Data

For testing, use PDF documents such as:
- Building permits
- Contractor licenses  
- Invoices or contracts
- Any text-based PDF documents

**Optimal File Characteristics:**
- Text-based PDFs (not scanned images)
- Size: Under 10MB each
- Resolution: 150-300 DPI
- Clear, standard fonts

## 🔧 Troubleshooting

### Common Issues

**"Stage not found" error:**
- Verify database/schema names match your setup
- Ensure you have proper permissions on the stage

**"AI_EXTRACT function not found" error:**
- Confirm Cortex AI is enabled in your account
- Check that `CORTEX_USER` role is granted

**"No files found" error:**
- Verify files are uploaded to the stage
- Run `LIST @YOUR_STAGE` to check contents
- For external stages, run `ALTER STAGE ... REFRESH`

**PDF viewer not working:**
- Ensure `pypdfium2` package is installed in notebook environment
- Check that PDF files are accessible from the stage

## 💰 Cost Considerations

- **AI_EXTRACT**: Charges based on input/output tokens (see [Snowflake pricing](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql#cost-considerations))
- **Tasks**: Consume compute credits when running
- **Stage storage**: Internal stages incur storage costs

Start with small document batches to understand your cost patterns.

## 📚 Additional Resources

- [Snowflake Cortex AI Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql)
- [Unstructured Data Guide](https://docs.snowflake.com/en/user-guide/unstructured-intro)
- [Stage Management](https://docs.snowflake.com/en/user-guide/data-load-s3-create-stage)

## 🆘 Support

For questions about this demo:
1. Check the troubleshooting section above
2. Review Snowflake documentation links
3. Verify all prerequisites are met

---
*This demo showcases Snowflake's powerful unstructured data processing capabilities. Happy analyzing! 🎯*

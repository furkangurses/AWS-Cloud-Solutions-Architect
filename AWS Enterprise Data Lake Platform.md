# AWS Enterprise Data Lake Platform

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen?style=for-the-badge&logo=github-actions)](https://github.com)
[![Security Scan](https://img.shields.io/badge/security-passing-brightgreen?style=for-the-badge&logo=shield)](https://github.com)
[![Lake Formation](https://img.shields.io/badge/AWS_Lake_Formation-enabled-FF9900?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com/lake-formation/)
[![Compliance](https://img.shields.io/badge/compliance-GDPR%20%7C%20HIPAA%20%7C%20SOC2-blue?style=for-the-badge)](https://aws.amazon.com)
[![ETL Coverage](https://img.shields.io/badge/ETL_coverage-PII_redacted-critical?style=for-the-badge&logo=awsglue)](https://aws.amazon.com/glue/)
[![License](https://img.shields.io/badge/license-Proprietary-red?style=for-the-badge)](LICENSE)

---

> **⚠️ PRODUCTION CRITICAL:** This repository governs a compliance-regulated, multi-tenant data lake architecture handling PII, financial records, and sensitive operational telemetry. All changes require peer review, passing security gates, and explicit sign-off from the Data Governance team before merging to `main`. Unauthorized schema modifications or permission changes may result in regulatory breach.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Platform Components](#platform-components)
- [Data Ingestion & ETL Pipeline](#data-ingestion--etl-pipeline)
- [PII Detection & Redaction Engine](#pii-detection--redaction-engine)
- [Lake Formation Access Control](#lake-formation-access-control)
- [DataBrew Data Quality Framework](#databrew-data-quality-framework)
- [Workflow Orchestration](#workflow-orchestration)
- [Business Intelligence Layer](#business-intelligence-layer)
- [Environment Configuration](#environment-configuration)
- [IAM Role Matrix](#iam-role-matrix)
- [Security Controls](#security-controls)
- [Deployment Runbook](#deployment-runbook)
- [Operational Verification Checklist](#operational-verification-checklist)
- [Incident Response & Observability](#incident-response--observability)

---

## Architecture Overview

This platform implements a fully managed, serverless data lake on AWS designed for high-throughput ingestion, automated governance, and compliance-aware analytics across multi-region enterprise workloads. It handles structured and semi-structured data at petabyte scale, enforcing row- and column-level security through AWS Lake Formation's fine-grained access control model.
```

┌─────────────────────────────────────────────────────────────────────────┐
│                     ENTERPRISE DATA LAKE ARCHITECTURE                   │
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │  Raw Sources │    │  Ingestion   │    │  AWS Lake Formation      │  │
│  │              │    │  Layer       │    │  (Governance Plane)      │  │
│  │  • CloudTrail│───▶│  • S3 Raw    │───▶│  • Blueprint Workflows   │  │
│  │  • App Logs  │    │    Bucket    │    │  • Data Location Registry│  │
│  │  • CRM/ERP   │    │  • EventBridge    │  • Tag-Based Permissions │  │
│  │  • IoT Feeds │    │    Triggers  │    └──────────────────────────┘  │
│  └──────────────┘    └──────────────┘               │                  │
│                                                      ▼                  │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      AWS GLUE ETL ENGINE                         │  │
│  │                                                                   │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │  │
│  │  │   Extract   │  │  Transform   │  │         Load             │ │  │
│  │  │             │  │              │  │                          │ │  │
│  │  │ • S3 Query  │─▶│ • PII Redact │─▶│ • S3 Curated (Parquet)  │ │  │
│  │  │ • Glue      │  │ • Dedup      │  │ • Glue Data Catalog      │ │  │
│  │  │   Crawler   │  │ • Schema     │  │ • Redshift (OLAP)        │ │  │
│  │  │             │  │   Normalize  │  │                          │ │  │
│  │  └─────────────┘  └──────────────┘  └──────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                         │                               │
│                                         ▼                               │
│  ┌────────────────────────┐   ┌──────────────────────────────────────┐  │
│  │  AWS Glue DataBrew     │   │         Analytics Layer              │  │
│  │  (Data Quality)        │   │                                      │  │
│  │  • DQ Rule Enforcement │   │  • Amazon Athena (Ad-hoc SQL)        │  │
│  │  • PII Profiling       │   │  • Amazon QuickSight (BI Dashboards) │  │
│  │  • Statistical Outlier │   │  • Row/Column-Level Security         │  │
│  └────────────────────────┘   └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Platform Components

| Component | Service | Role | Runtime |
|---|---|---|---|
| **Ingestion Orchestration** | AWS Lake Formation Blueprints | Automates end-to-end data lake provisioning via pre-built workflow templates | Managed |
| **ETL Engine** | AWS Glue (Spark / Python Shell) | Executes extract, transform, and load jobs across all data domains | Serverless |
| **Schema Registry** | AWS Glue Data Catalog | Central metadata store for all databases, tables, and partition schemas | Managed |
| **Data Quality** | AWS Glue DataBrew | 250+ prebuilt transforms; statistical profiling; DQ rule enforcement | Serverless |
| **Governance Plane** | AWS Lake Formation | Fine-grained access control at database, table, column, and row level | Managed |
| **Ad-hoc Query** | Amazon Athena | Serverless SQL over S3; Trino-compatible; Pay-per-query | Serverless |
| **BI & Visualization** | Amazon QuickSight | SPICE-accelerated dashboards; role-aware data sharing | Managed |
| **Raw & Curated Storage** | Amazon S3 | Tiered storage: raw → curated (Parquet) → results | Durable |
| **Audit Trail** | AWS CloudTrail | Source dataset for operational analytics and security forensics | Managed |

---

## Data Ingestion & ETL Pipeline

### Trigger Modes

AWS Glue jobs in this platform support three execution modes, selected based on SLA and data velocity requirements:

| Mode | Trigger Type | Use Case | Example |
|---|---|---|---|
| **Scheduled** | Time-based (cron) | Batch ingestion with known cadence | `cron(0 13 ? * TUE *)` — every Tuesday 13:00 UTC |
| **On-Demand** | Manual / API invocation | Large one-off backfill jobs; ad-hoc remediation | `aws glue start-job-run --job-name <job>` |
| **Conditional (Event-Driven)** | S3 Event → Lambda → Glue trigger | Real-time ingestion on new object arrival | S3 `s3:ObjectCreated:*` → Lambda → `StartJobRun` |

> **⚠️ OPERATIONAL NOTE:** Conditional triggers are the preferred mode for all streaming-adjacent workloads. Ensure Lambda functions invoking Glue jobs implement dead-letter queues (DLQ) and exponential backoff retry logic. Do not poll S3 directly from Glue without an event intermediary.

### Supported Scripting Environments

AWS Glue ETL jobs in this platform are authored in one of the following environments. All scripts are stored in `s3://<glue-assets-bucket>/scripts/` and version-controlled via this repository under `glue/jobs/`.

| Authoring Mode | Engine | Best For |
|---|---|---|
| **Glue Script Editor** | Apache Spark (PySpark) / Python Shell | Full custom transformations; existing migration scripts |
| **Glue Interactive Sessions** | Jupyter Notebook (Spark kernel) | Iterative development; exploratory data engineering |
| **Glue Studio (Visual ETL)** | Auto-generated PySpark | Drag-and-drop DAG authoring with code transparency |

### ETL Job Phases
```

[EXTRACT] [TRANSFORM] [LOAD] ───────────────────── ────────────────────────────── ────────────────────────

-   Query S3 raw prefix • PII field detection • Write Parquet to S3
-   Incremental watermarking • Sensitive data redaction • Update Glue Data Catalog
-   Schema inference via • Deduplication • Notify downstream via SNS Glue Crawler • Date format normalization • Trigger Athena CTAS if needed • Postal/Phone standardization • Null/Sparse field handling

```

---

## PII Detection & Redaction Engine

This platform uses **AWS Glue Studio Visual ETL** to automate the detection and redaction of personally identifiable information without requiring manual code authorship.

### Detected PII Categories

The following entity types are actively scanned in all ingestion pipelines:

| PII Category | Detection Pattern | Redaction Strategy | Regulatory Driver |
|---|---|---|---|
| Social Security Number | Regex + ML classifier | Full field redaction | CCPA, HIPAA |
| Credit Card Number | Luhn + pattern match | Full field redaction | PCI-DSS |
| Email Address | RFC 5322 pattern | Domain-preserving hash | GDPR Art. 17 |
| IP Address (last octet) | IPv4/IPv6 classifier | Last-octet truncation | GDPR Recital 49 |
| Full Name | NER model | Tokenized pseudonym | GDPR Art. 4 |
| Date of Birth | Date entity classifier | Year-only retention | HIPAA Safe Harbor |
| Physical Address | Address NER | City/State only retention | CCPA |
| Tax ID / EIN | Pattern match | Full field redaction | IRS Publication 1075 |

> **⚠️ SECURITY CRITICAL:** Redaction markers (e.g., `[REDACTED]`, `***SUPER-DUPER-SECRET***`) used in development pipelines **must never** be used in production. Production redaction must either drop the column entirely or replace with a deterministic HMAC-SHA256 token to preserve referential integrity across datasets. Audit all redaction configurations before promoting to production.

### Visual ETL Workflow — PII Redaction DAG
```

[Source: S3 Raw CSV] │ ▼ [Detect Sensitive Data Node] ├── Pattern: CREDIT_CARD ├── Pattern: SSN ├── Pattern: EMAIL_ADDRESS └── Action: REDACT (field-level) │ ▼ [Change Schema Node] └── Drop detected_entities metadata columns │ ▼ [Target: S3 Curated JSON/Parquet] └── Update Glue Data Catalog → cloudtraillogs-db

```

The generated PySpark script is committed to `glue/jobs/pii_redaction_etl.py` automatically by the CI pipeline after each Glue Studio save.

---

## Lake Formation Access Control

### Tag-Based Access Control (TBAC)

Lake Formation tags (LF-Tags) are the authoritative permission boundary for this data lake. All IAM-level S3 bucket policies are secondary and must never be the sole access control mechanism.

#### Tag Taxonomy

| Tag Key | Allowed Values | Scope |
|---|---|---|
| `classification` | `public`, `internal`, `confidential`, `restricted` | Database, Table, Column |
| `data_domain` | `finance`, `hr`, `operations`, `security`, `marketing` | Database, Table |
| `job_role` | `analyst`, `developer`, `admin`, `auditor` | Principal-level grant |
| `pii_presence` | `true`, `false` | Column |
| `retention_tier` | `30d`, `1y`, `7y`, `indefinite` | Table |

> **⚠️ GOVERNANCE POLICY:** Default tag assignments at the database level are **inherited** by all child tables and columns unless explicitly overridden. Always tag at the most restrictive level first (i.e., default all new tables to `classification=restricted`) and progressively loosen access. A misconfigured permissive tag is a compliance incident.

#### Permission Grant Flow
```

IAM Principal (User/Role/Group) │ ▼ Lake Formation Tag Permission Check ├── Does principal's granted tags MATCH the resource's LF-Tags? │ YES ─▶ Temporary credential issued (STS AssumeRole) │ NO ─▶ Access denied (no data returned, no error detail leaked) │ ▼ S3 / Glue Data Catalog Resource Access └── Data returned scoped to permitted rows + columns only

```

### Cross-Account Data Sharing

Lake Formation RAM (Resource Access Manager) integration enables controlled cross-account access:

```bash
# Grant cross-account access to a specific LF-Tag combination
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::<TARGET_ACCOUNT_ID>:root \
  --permissions DESCRIBE SELECT \
  --resource '{"LFTagPolicy":{"ResourceType":"TABLE","Expression":[{"TagKey":"classification","TagValues":["internal"]},{"TagKey":"job_role","TagValues":["analyst"]}]}}' \
  --region us-east-1
```

---

## DataBrew Data Quality Framework

AWS Glue DataBrew enforces data quality rules **before** data is admitted into the curated zone of the lake.

### DQ Rule Sets

All rule sets are defined declaratively and stored under `databrew/rulesets/` in this repository.

#### Customer Records Dataset — DQ Rules

```yaml
# databrew/rulesets/customer_records_dq.yaml
RulesetName: customer-records-quality
Dataset: customer_records
Rules:
  - Name: no_duplicate_rows
    CheckExpression: "DUPLICATE_ROWS_COUNT == 0"
    Threshold: 1.0

  - Name: amount_non_zero
    Columns: [Amount, Total_Sales]
    CheckExpression: "NUMERIC_VALUE > 0"
    Threshold: 0.99   # 99% of records must pass

  - Name: pii_presence_flagged
    Columns: [SSN, CreditCard, Email, DateOfBirth]
    CheckExpression: "PII_DETECTED == true"
    Action: FLAG_AND_ALERT
```

> **⚠️ QUALITY GATE:** Any DQ profile job returning a success rate below the defined threshold (`Threshold` field) **must block** the downstream ETL load job. The Glue workflow conditional trigger is configured to evaluate DQ job status before proceeding. A sub-threshold result triggers a PagerDuty P2 alert to the Data Engineering on-call.

### Profiling Run Configuration

DataBrew profile jobs are executed against full datasets on each ingestion cycle:

```bash
aws databrew create-profile-job \
  --name "customer-records-profile-$(date +%Y%m%d)" \
  --dataset-name customer_records \
  --output-location "s3://${DATA_LAKE_BUCKET}/databrew-profiles/" \
  --role-arn "${DATABREW_ROLE_ARN}" \
  --configuration '{"EntityDetectorConfiguration":{"EntityTypes":["CREDIT_CARD","SSN","EMAIL","DRIVER_ID","BANK_ACCOUNT"],"AllowedStatistics":[{"Statistics":["TOP_VALUES","UNIQUE_VALUES_COUNT"]}]}}' \
  --job-sample '{"Mode":"FULL_DATASET"}'
```

---

## Workflow Orchestration

### Lake Formation Blueprint Deployment

Lake Formation blueprints generate complete Glue workflows (crawlers, jobs, triggers) for standard ingestion patterns. This platform uses the **AWS CloudTrail** blueprint as the canonical ingestion template.

```bash
# Deploy Lake Formation blueprint via CLI
aws lakeformation start-data-lake-settings \
  --data-lake-settings '{"DataLakeAdmins":[{"DataLakePrincipalIdentifier":"arn:aws:iam::<ACCOUNT_ID>:role/LakeFormationAdminRole"}]}'

aws lakeformation create-workflow \
  --name lf-cloudtrail-workflow \
  --blueprint-name AwsCloudTrail \
  --blueprint-parameters '{
    "CloudTrailName": "ProductionCloudTrail",
    "TargetDatabase": "cloudtraillogs-db",
    "TargetStorageLocation": "s3://'${DATA_LAKE_BUCKET}'/data/",
    "DataFormat": "Parquet",
    "TablePrefix": "prod"
  }' \
  --role-arn "${LAKEFORMATION_WORKFLOW_ROLE_ARN}"
```

### Glue Workflow DAG Structure

The generated workflow implements the following directed acyclic graph:
```

[ON_DEMAND_TRIGGER] │ ▼ [pre_crawler_job] → Sets workflow state: DISCOVERING │ ▼ [pre_crawler_trigger] │ ▼ [schema_discoverer_crawler] → Crawls S3 raw prefix; updates Data Catalog │ ▼ [post_crawler_trigger] │ ▼ [post_crawler_job] → Sets workflow state: IMPORTING │ ▼ [etl_trigger] │ ▼ [workflow_etl_job] → PySpark transform: dedup, format, PII redaction │ ▼ [post_etl_trigger] │ ▼ [post_etl_job] → Sets workflow state: COMPLETED; SNS notification

```

### Starting a Workflow Run

```bash
# On-demand execution
aws glue start-workflow-run \
  --name lf-cloudtrail-workflow \
  --region "${AWS_REGION}"

# Monitor run status
aws glue get-workflow-run \
  --name lf-cloudtrail-workflow \
  --run-id "${WORKFLOW_RUN_ID}" \
  --query 'Run.Status'
```

> **⚠️ OPERATIONAL NOTE:** Workflow runs for this platform take **15–20 minutes** under normal load. Do not re-trigger a run while the previous run is in `RUNNING` state. Concurrent runs will cause data duplication in the curated zone and will require a full partition repair (`MSCK REPAIR TABLE`) before analytics can resume.

---

## Business Intelligence Layer

Amazon QuickSight serves as the governed analytics frontend for this data lake. All dashboards are published to designated business unit groups with row-level security (RLS) enforced via Lake Formation credential vending.

### Dataset Registration

```bash
# Register curated S3 dataset in QuickSight
aws quicksight create-data-source \
  --aws-account-id "${ACCOUNT_ID}" \
  --data-source-id "cloudtrail-curated-ds" \
  --name "CloudTrail Curated Parquet" \
  --type ATHENA \
  --data-source-parameters '{"AthenaParameters":{"WorkGroup":"primary"}}' \
  --permissions '[{"Principal":"arn:aws:quicksight:'${AWS_REGION}':'${ACCOUNT_ID}':group/default/DataAnalysts","Actions":["quicksight:DescribeDataSource","quicksight:PassDataSource"]}]'
```

### Dashboard Publishing Policy

- Dashboards are published as **read-only shared views** to business unit groups.
- Raw CSV/JSON exports from QuickSight are **disabled** for datasets tagged `classification=confidential` or higher.
- All dashboard interactions are logged to CloudTrail for audit purposes.
- Analysts receive pre-filtered dataset views based on their Lake Formation tag grants — QuickSight never exposes data beyond what Lake Formation permits.

---

## Environment Configuration

All runtime configuration is managed via AWS Systems Manager Parameter Store (encrypted) and injected as environment variables at job execution time. **Never commit credentials or account IDs to this repository.**

| Variable | Description | SSM Path | Required |
|---|---|---|---|
| `DATA_LAKE_BUCKET` | Primary S3 bucket for raw + curated data | `/datalake/prod/bucket-name` | ✅ |
| `RESULTS_BUCKET_PREFIX` | S3 prefix for Athena query results | `/datalake/prod/results-prefix` | ✅ |
| `LAKEFORMATION_WORKFLOW_ROLE_ARN` | IAM role assumed by Lake Formation workflows | `/datalake/prod/workflow-role-arn` | ✅ |
| `DATABREW_ROLE_ARN` | IAM role for DataBrew profile jobs | `/datalake/prod/databrew-role-arn` | ✅ |
| `GLUE_SERVICE_ROLE_ARN` | IAM role for Glue ETL job execution | `/datalake/prod/glue-role-arn` | ✅ |
| `PII_REDACTION_TOKEN` | HMAC secret for deterministic PII tokenization | `/datalake/prod/pii-hmac-secret` | ✅ |
| `SOURCE_DATA_LOCATION` | S3 URI of registered Lake Formation data location | `/datalake/prod/source-data-location` | ✅ |
| `ATHENA_WORKGROUP` | Athena workgroup name with cost controls | `/datalake/prod/athena-workgroup` | ✅ |
| `AWS_REGION` | Target AWS region | `/datalake/prod/aws-region` | ✅ |
| `CLOUDTRAIL_NAME` | Name of the CloudTrail trail used as data source | `/datalake/prod/cloudtrail-name` | ✅ |

---

## IAM Role Matrix

> **⚠️ LEAST PRIVILEGE MANDATE:** No role in this platform is granted `AdministratorAccess` or wildcard resource (`*`) permissions beyond what is explicitly documented below. All role trust policies are scoped to specific AWS service principals. Quarterly access reviews are mandatory per SOC 2 CC6.3.

| Role | Principal | Key Permissions | Scope |
|---|---|---|---|
| `LakeFormationAdminRole` | IAM Admin persona | Full Lake Formation control; all catalog operations | Data lake administrator |
| `LakeFormationServiceRole` | `lakeformation.amazonaws.com` | S3 read on registered locations; Glue catalog write | Service-level ingestion |
| `LakeFormationWorkflowRole` | `glue.amazonaws.com` | Execute Glue jobs/crawlers; S3 read/write on data lake buckets; Catalog write | Workflow execution |
| `DataEngineerRole` | Engineering IAM group | Glue Studio authoring; DataBrew jobs; S3 write to raw prefix; Catalog read | Pipeline development |
| `DataAnalystRole` | Analytics IAM group | Athena query; QuickSight dataset access; S3 read on results prefix only | Read-only analytics |
| `WorkflowExecutionRole` | AWS services (Lambda, EventBridge) | `glue:StartJobRun`, `glue:StartWorkflowRun`; SNS publish | Automation/orchestration |
| `AuditRole` | Security/Compliance team | CloudTrail read; Lake Formation permission read; Athena query on audit tables | Read-only forensics |

---

## Security Controls

### Data-at-Rest

- All S3 buckets use **SSE-KMS** with customer-managed CMKs (AWS KMS).
- KMS key rotation is enabled with a 365-day rotation period.
- S3 bucket policies enforce `aws:SecureTransport: true` (TLS-only).
- S3 Object Lock is enabled on the raw ingestion prefix (COMPLIANCE mode, 7-year retention per data retention policy).

### Data-in-Transit

- All API calls to AWS services traverse TLS 1.2+ endpoints exclusively.
- VPC Endpoints are configured for S3, Glue, Lake Formation, and Athena to prevent data traversal over the public internet.

### Access Governance

- Lake Formation TBAC is the **authoritative** access control layer; S3 bucket policies are secondary.
- All permission grants are logged to CloudTrail (`lakeformation.amazonaws.com` event source).
- MFA delete is enabled on all S3 buckets storing curated or restricted data.
- Temporary STS credentials issued by Lake Formation have a maximum session duration of 1 hour.

### Threat Detection

- AWS GuardDuty S3 protection is enabled on all data lake buckets.
- AWS Macie continuously scans the raw ingestion prefix for undetected PII before DataBrew profiling runs.
- CloudWatch Alarms are configured for anomalous Athena query volumes (potential data exfiltration indicator).

---

## Deployment Runbook

### Prerequisites

Ensure the following are provisioned and validated before executing deployment steps:

- [ ] AWS CLI v2 configured with appropriate credentials (`aws sts get-caller-identity`)
- [ ] Target AWS account has Lake Formation service linked role provisioned
- [ ] S3 buckets created with SSE-KMS and appropriate bucket policies applied
- [ ] CloudTrail trail active and delivering logs to the designated S3 prefix
- [ ] IAM roles (`LakeFormationServiceRole`, `LakeFormationWorkflowRole`) deployed via IaC (Terraform/CDK)
- [ ] AWS Glue Data Catalog encryption enabled (SSE-KMS)
- [ ] VPC endpoints deployed for S3, Glue, Lake Formation, Athena

### Deployment Steps

```bash
# 1. Set runtime environment from SSM Parameter Store
export DATA_LAKE_BUCKET=$(aws ssm get-parameter --name "/datalake/prod/bucket-name" --with-decryption --query "Parameter.Value" --output text)
export SOURCE_DATA_LOCATION=$(aws ssm get-parameter --name "/datalake/prod/source-data-location" --with-decryption --query "Parameter.Value" --output text)
export LAKEFORMATION_WORKFLOW_ROLE_ARN=$(aws ssm get-parameter --name "/datalake/prod/workflow-role-arn" --with-decryption --query "Parameter.Value" --output text)
export AWS_REGION=$(aws ssm get-parameter --name "/datalake/prod/aws-region" --query "Parameter.Value" --output text)

# 2. Register S3 data location with Lake Formation
aws lakeformation register-resource \
  --resource-arn "arn:aws:s3:::${DATA_LAKE_BUCKET}/data/" \
  --role-arn "${LAKEFORMATION_SERVICE_ROLE_ARN}" \
  --use-service-linked-role false \
  --region "${AWS_REGION}"

# 3. Create Glue Data Catalog database
aws glue create-database \
  --database-input '{
    "Name": "cloudtraillogs-db",
    "LocationUri": "'"${SOURCE_DATA_LOCATION}"'",
    "Description": "CloudTrail operational analytics — curated data lake database"
  }' \
  --region "${AWS_REGION}"

# 4. Grant workflow role data location permissions
aws lakeformation grant-permissions \
  --principal "DataLakePrincipalIdentifier=${LAKEFORMATION_WORKFLOW_ROLE_ARN}" \
  --permissions DATA_LOCATION_ACCESS \
  --resource '{"DataLocation":{"ResourceArn":"arn:aws:s3:::'"${DATA_LAKE_BUCKET}"'/data/"}}' \
  --region "${AWS_REGION}"

# 5. Create and start Lake Formation blueprint workflow
aws lakeformation create-workflow \
  --name lf-cloudtrail-workflow \
  --blueprint-name AwsCloudTrail \
  --blueprint-parameters '{
    "CloudTrailName": "'"${CLOUDTRAIL_NAME}"'",
    "TargetDatabase": "cloudtraillogs-db",
    "TargetStorageLocation": "'"${SOURCE_DATA_LOCATION}"'",
    "DataFormat": "Parquet",
    "TablePrefix": "prod"
  }' \
  --role-arn "${LAKEFORMATION_WORKFLOW_ROLE_ARN}" \
  --region "${AWS_REGION}"

# 6. Start workflow run
WORKFLOW_RUN_ID=$(aws lakeformation start-workflow-run \
  --name lf-cloudtrail-workflow \
  --query 'RunId' --output text \
  --region "${AWS_REGION}")

echo "Workflow run initiated: ${WORKFLOW_RUN_ID}"

# 7. Poll for completion (expected: 15–20 minutes)
while true; do
  STATUS=$(aws glue get-workflow-run \
    --name lf-cloudtrail-workflow \
    --run-id "${WORKFLOW_RUN_ID}" \
    --query 'Run.Status' --output text \
    --region "${AWS_REGION}")
  echo "[$(date -u +%H:%M:%S)] Workflow status: ${STATUS}"
  [[ "${STATUS}" == "COMPLETED" || "${STATUS}" == "FAILED" || "${STATUS}" == "STOPPED" ]] && break
  sleep 60
done

[[ "${STATUS}" != "COMPLETED" ]] && { echo "ERROR: Workflow did not complete successfully. Status: ${STATUS}"; exit 1; }
echo "Workflow completed successfully."
```

---

## Operational Verification Checklist

Execute all validation steps after each deployment or workflow run in production.

### Post-Deployment Validation

- [ ] Lake Formation data location is registered: `aws lakeformation list-resources`
- [ ] `cloudtraillogs-db` database visible in Glue Data Catalog
- [ ] `prod_cloudtrail` and `_prod_cloudtrail` tables present in the database
- [ ] `prod_cloudtrail` table data format confirmed as `Parquet` with expected 20+ columns
- [ ] Lake Formation TBAC tags applied to all tables (`classification`, `data_domain`, `job_role`)
- [ ] DataBrew profile job executed; DQ success rate ≥ threshold for all rule sets
- [ ] PII redaction confirmed: no raw SSN, credit card, or email in curated dataset
- [ ] Athena workgroup configured with query result location pointing to `results/` prefix

### Athena Validation Queries

```sql
-- Validate table structure and data availability
SELECT * FROM "cloudtraillogs-db"."prod_cloudtrail" LIMIT 10;

-- Validate PII redaction completeness
SELECT COUNT(*) AS pii_leakage_count
FROM "cloudtraillogs-db"."prod_cloudtrail"
WHERE useridentity_principalid IS NOT NULL
  AND useridentity_principalid NOT LIKE '%[REDACTED]%'
  AND LENGTH(useridentity_principalid) > 20;

-- Operational: Find all CloudTrail error events for incident triage
SELECT eventtime, eventsource, eventname, errorcode, errormessage, sourceipaddress
FROM "cloudtraillogs-db"."prod_cloudtrail"
WHERE NOT errorcode = ''
ORDER BY eventtime DESC
LIMIT 500;

-- Compliance: Identify unauthorized API calls
SELECT useridentity_arn, eventname, sourceipaddress, eventtime
FROM "cloudtraillogs-db"."prod_cloudtrail"
WHERE errorcode IN ('AccessDenied', 'UnauthorizedAccess', 'InvalidClientTokenId')
  AND DATE(from_iso8601_timestamp(eventtime)) = CURRENT_DATE
ORDER BY eventtime DESC;
```

---

## Incident Response & Observability

### CloudWatch Alarms

| Alarm | Metric | Threshold | Response |
|---|---|---|---|
| `GlueETLJobFailure` | `glue.driver.aggregate.numFailedTasks > 0` | Any failure | PagerDuty P2 → Data Engineering on-call |
| `DataQualityThresholdBreach` | DataBrew job output success rate | < 99% | PagerDuty P2 → Data Governance |
| `LakeFormationAccessDeniedSpike` | `lakeformation:GetDataAccess` 4xx errors | > 50/5min | PagerDuty P1 → Security on-call |
| `AthenaQueryVolumeAnomaly` | Athena `DataScannedInBytes` per user | > 500 GB/hour | CloudWatch alert → Security review |
| `S3UnauthorizedAccess` | S3 `4xxErrors` on data lake bucket | > 10/min | GuardDuty auto-block + P1 alert |

### Log Retention

| Log Source | Destination | Retention | Encryption |
|---|---|---|---|
| Glue ETL job logs | CloudWatch Logs | 90 days | SSE-KMS |
| DataBrew profile output | S3 `databrew-profiles/` prefix | 365 days | SSE-KMS |
| Lake Formation audit logs | CloudTrail → S3 | 7 years | SSE-KMS + Object Lock |
| Athena query history | CloudWatch + S3 results bucket | 90 days | SSE-KMS |

---
```

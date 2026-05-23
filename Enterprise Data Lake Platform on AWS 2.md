# 🏗️ Enterprise Data Lake Platform on AWS
### A Production-Grade, Compliance-Regulated Modern Data Architecture

[![CI/CD Pipeline](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)](https://github.com/actions)
[![AWS Lake Formation](https://img.shields.io/badge/AWS-Lake%20Formation-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/lake-formation/)
[![Infrastructure](https://img.shields.io/badge/IaC-Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Security Scan](https://img.shields.io/badge/Security-PASSING-28A745?style=for-the-badge&logo=shield&logoColor=white)]()
[![SIEM Coverage](https://img.shields.io/badge/SIEM-CloudTrail%20Enabled-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)]()
[![Compliance](https://img.shields.io/badge/Compliance-SOC2%20%7C%20HIPAA%20Aligned-red?style=for-the-badge&logo=lock&logoColor=white)]()
[![License](https://img.shields.io/badge/License-Apache%202.0-blue?style=for-the-badge)]()


---

## 📋 Table of Contents

- [Executive Overview](#-executive-overview)
- [Architecture Overview](#-architecture-overview)
- [The Five Pillars of Modern Data Architecture](#-the-five-pillars-of-modern-data-architecture)
- [Data Movement Strategies](#-data-movement-strategies)
- [Data Mesh & Federated Governance Model](#-data-mesh--federated-governance-model)
- [Technology Stack](#-technology-stack)
- [Environment & Configuration Reference](#-environment--configuration-reference)
- [LF-Tag Governance Matrix](#-lf-tag-governance-matrix)
- [Deployment Guide](#-deployment-guide)
- [ETL Pipeline: Glue Job Architecture](#-etl-pipeline-glue-job-architecture)
- [Access Control Implementation (LF-TBAC)](#-access-control-implementation-lf-tbac)
- [Consumer Tier Verification Runbook](#-consumer-tier-verification-runbook)
- [Security Hardening & Compliance Controls](#-security-hardening--compliance-controls)
- [Operational Runbooks](#-operational-runbooks)
- [Pre-Production Checklist](#-pre-production-checklist)
- [Additional Resources](#-additional-resources)

---

## 🎯 Executive Overview

This repository provides the reference implementation for an **enterprise-grade, multi-tenant data lake platform** deployed on AWS. It is architected to ingest, process, govern, and serve structured and semi-structured datasets at petabyte scale while enforcing **fine-grained, attribute-based access control** across internal and external data consumers.

The platform underpins a critical **Media Intelligence & Content Analytics Engine**, processing high-throughput catalog datasets from ingestion through to BI presentation. Consumer access is segmented by subscription tier (`Regular` and `Enterprise`), enforced at the column level using **AWS Lake Formation Tag-Based Access Control (LF-TBAC)** — eliminating error-prone IAM policy sprawl in favor of a declarative, auditable governance model.

> **⚠️ PRODUCTION WARNING:** This platform operates under a **zero-trust data access model**. All IAM-native (`IAMAllowedPrincipals`) passthrough permissions have been explicitly **revoked**. Data access is exclusively governed by Lake Formation LF-tag grants. Any modification to tag assignments or principal grants **must** go through the Change Advisory Board (CAB) process and be peer-reviewed via pull request before applying to the production data catalog.

---

## 🏛️ Architecture Overview
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE DATA LAKE — AWS REFERENCE ARCHITECTURE             │
│                                                                                 │
│  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────────┐  │
│  │   RAW DATA ZONE  │      │  PROCESSING ZONE │      │    CURATED ZONE      │  │
│  │                  │      │                  │      │                      │  │
│  │  Amazon S3       │─────▶│  AWS Glue ETL    │─────▶│  AWS Glue Data       │  │
│  │  (Raw Bucket)    │      │  (transform-job) │      │  Catalog + S3        │  │
│  │                  │      │  PySpark / ML    │      │  (Curated Bucket)    │  │
│  └──────────────────┘      └──────────────────┘      └──────────┬───────────┘  │
│                                                                  │              │
│  ┌───────────────────────────────────────────────────────────────▼───────────┐  │
│  │                    AWS LAKE FORMATION (GOVERNANCE PLANE)                  │  │
│  │                                                                           │  │
│  │   LF-Tag: Environment=[Development, Production]                          │  │
│  │   LF-Tag: Customer=[Regular, Enterprise]                                 │  │
│  │   LF-Tag: Confidential=[True, False]                                     │  │
│  │                                                                           │  │
│  │   Principal: Consumer_A ─── Regular Tier (11/13 columns visible)         │  │
│  │   Principal: Consumer_B ─── Enterprise Tier (13/13 columns visible)      │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                         CONSUMPTION LAYER                                │   │
│  │                                                                          │   │
│  │   Amazon Athena (Trino SQL)  │  Amazon QuickSight  │  SageMaker Studio   │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘

```

---

## 🧱 The Five Pillars of Modern Data Architecture

This platform is designed around AWS's five-pillar framework for modern data architecture. Each pillar maps directly to a set of deployed AWS services and enforced operational policies.

---

### Pillar 1 — Scalable Data Lakes

Secure, scalable object storage is the foundation of the entire platform. Metadata cataloging ensures discoverability at scale.

| Responsibility | AWS Service | Configuration |
|---|---|---|
| Primary Data Storage | **Amazon S3** | Versioning ON, SSE-KMS, Object Lock (Compliance Mode) |
| Data Lake Bootstrap & Registration | **AWS Lake Formation** | S3 locations registered with LF; `IAMAllowedPrincipals` revoked |
| Ad-hoc Interactive Query | **Amazon Athena** | Workgroup isolation per consumer tier; results bucket encrypted |
| Metadata Cataloging | **AWS Glue Data Catalog** | Schema evolution enabled; column-level statistics active |

---

### Pillar 2 — Purpose-Built Analytics Services

A core principle of this architecture is **workload-to-service affinity** — matching the right purpose-built data store to each specific analytical or operational pattern, rather than forcing all workloads through a single, over-generalized engine.

| Workload Pattern | Purpose-Built Service | Justification |
|---|---|---|
| OLTP / Relational Transactions | **Amazon Aurora (PostgreSQL)** | ACID compliance, low-latency reads |
| High-Throughput Key-Value / Session State | **Amazon DynamoDB** | Single-digit ms latency, serverless scaling |
| Full-Text & Faceted Search | **Amazon OpenSearch Service** | Inverted index, aggregations, Kibana dashboards |
| Large-Scale Distributed ETL | **Amazon EMR (Spark/Hive)** | Petabyte-scale Spark processing, transient clusters |
| Enterprise Data Warehousing & BI | **Amazon Redshift** | Columnar storage, RA3 node decoupled storage/compute |
| ML Training & Inference | **Amazon SageMaker** | Feature Store integration, A/B endpoint management |
| Event Streaming / Kafka-Compatible | **Amazon MSK** | Managed Confluent-compatible, schema registry |
| Real-Time Stream Analytics | **Amazon Kinesis Data Analytics** | Apache Flink, sub-second latency, SQL interface |

---

### Pillar 3 — Unified Data Access

Selective, governed data movement is enforced through managed integration services that remove the need for brittle, custom-scripted pipelines.

| Integration Pattern | AWS Service | Use Case in this Platform |
|---|---|---|
| ETL / ELT Orchestration | **AWS Glue** | Schema-on-read transformations, Parquet conversion, ML fill |
| Streaming Data Delivery | **Amazon Data Firehose** | Real-time S3/Redshift delivery with buffering |
| Database Replication | **AWS DMS** | Homogeneous and heterogeneous DB migrations, CDC |
| Managed File Transfer | **AWS Transfer Family** | SFTP/FTPS for third-party data vendor onboarding |
| SaaS Integration | **Amazon AppFlow** | Salesforce, ServiceNow → S3 data ingestion |

---

### Pillar 4 — Unified Governance

> **🔐 CRITICAL SECURITY POSTURE:** Governance is **not optional** and **not additive**. In this platform, Lake Formation is the **sole authorization authority** for all data catalog resources. IAM policies define the upper bound of permissions; Lake Formation grants define the operational, data-level bound. The intersection of both is what a principal can access.

**AWS Lake Formation** is the designated governance service. It handles:

- **Authorization** — Fine-grained column-, table-, and database-level access via LF-TBAC
- **Data Discovery** — Tag-based catalog search without exposing underlying data
- **Auditing** — All data access events logged to AWS CloudTrail and forwarded to centralized SIEM
- **Row-Level & Cell-Level Security** — Available for enterprise tier workloads requiring regulatory segmentation

---

### Pillar 5 — Performance & Cost Effectiveness

Performance optimization is built into the data format and storage strategy, not bolted on post-hoc.

| Optimization Technique | Implementation | Estimated Benefit |
|---|---|---|
| **Columnar Data Format** | Apache Parquet (Snappy compression) | Up to **87% reduction** in Athena bytes scanned |
| **Data Partitioning** | Hive-style partitions (`year=`, `month=`) in S3 | Partition pruning eliminates full-table scans |
| **Predicate Pushdown** | Enforced via Parquet row group min/max statistics | Scans only relevant row groups within a file |
| **Data Compression** | Snappy (balance) or Zstd (max compression) | 3x–7x storage cost reduction vs raw CSV |
| **Billing Alerts** | AWS Budgets → SNS → PagerDuty | Prevents uncontrolled Athena/EMR cost runaway |
| **Lifecycle Policies** | S3 Intelligent-Tiering on raw zone | Automatic cold-tier migration for infrequently accessed objects |

> **💡 ENGINEERING NOTE (Columnar Formats):** Apache Parquet's **row group statistics** (min/max per column chunk) enable query engines to skip entire row groups that don't satisfy a `WHERE` predicate. On a sorted dataset — e.g., `ORDER BY title ASC` — a query such as `SELECT * WHERE title = 'Inception'` will skip all row groups whose max title value is lexicographically less than `'Inception'`. This is not merely a best practice; on datasets exceeding 50M records, this distinction means the difference between a 30-second query and a 300ms query. **All curated zone outputs from this platform are written in Parquet with Snappy compression.**

---

## 🔄 Data Movement Strategies

The platform implements all three canonical data movement patterns. The correct pattern is selected based on the **data residency requirement**, **latency SLA**, and **processing sovereignty** of the consuming workload.

### Pattern 1: Inside-Out (Data Lake → Specialized Stores)

The data lake acts as the **single source of truth**. A curated subset of catalog data is exported downstream to specialized engines for purpose-specific consumption.
```

S3 (Curated Zone) │ ├──▶ Amazon Redshift (Daily reporting, BI dashboards via QuickSight) ├──▶ Amazon OpenSearch (Search analytics, faceted catalog exploration) └──▶ Amazon Neptune (Knowledge graph construction, entity relationship)

```

**Trigger Mechanism:** AWS Glue trigger (cron-scheduled) or EventBridge rule on S3 PutObject event.

---

### Pattern 2: Outside-In (External Sources → Data Lake)

Data **primarily lives in operational systems**. The data lake is engaged only when analytical or ML processing is required. This avoids unnecessary data duplication.
```

Amazon DynamoDB (Operational Game/Session State) Amazon Aurora (Transactional CRM/ERP Data) Third-Party APIs (Market Data, Social Feeds) │ ▼ AWS DMS / AppFlow / Kinesis Firehose │ ▼ S3 Raw Zone ──▶ AWS Glue ETL ──▶ SageMaker (Recommendation / Anomaly Models)

```

**Use Case in this Platform:** Player behavioral data from DynamoDB is periodically exported to the data lake for cohort analysis and churn prediction via SageMaker.

---

### Pattern 3: Around the Perimeter (Service-to-Service)

Data transits **between purpose-built services directly**, without necessarily traversing the central data lake. The data lake may still hold a reference copy, but it is not on the hot path.
```

Amazon Aurora (Customer Profiles) │ └──▶ Amazon DynamoDB (Denormalized Reporting Tables for Dashboard API) │ └──▶ Amazon OpenSearch (Real-time search offload, reducing Aurora IOPS)

```

> **⚠️ OPERATIONAL NOTE:** In around-the-perimeter scenarios, **eventual consistency** between the source system and derived data stores must be explicitly acknowledged in the service SLA. Implement **DMS CDC (Change Data Capture)** to minimize replication lag. Do not rely on batch-scheduled full-table copies for latency-sensitive consumer workloads.

---

## 🕸️ Data Mesh & Federated Governance Model

This platform's governance model is aligned to **Data Mesh principles**, operationalized through Amazon Data Zone and AWS Lake Formation.

| Data Mesh Principle | Platform Implementation |
|---|---|
| **Domain Ownership** | Each business unit (Media Catalog, User Analytics, Ad Revenue) owns its S3 prefix, Glue database, and LF-tag grants |
| **Data as a Product** | Curated datasets are published as discoverable, versioned data products with defined SLAs and ownership metadata in Amazon Data Zone |
| **Self-Serve Data Platform** | Consumers use Amazon Athena + QuickSight with pre-provisioned workgroups; no pipeline request queue needed |
| **Federated Computational Governance** | Lake Formation centrally enforces policy; domain teams manage their own tag assignments within approved taxonomies |

**Amazon Data Zone** serves as the enterprise data catalog and marketplace layer, integrating with:
- AWS Lake Formation (access policy enforcement)
- Amazon Athena (query execution)
- AWS Glue (schema discovery)
- Amazon QuickSight (dashboard publishing)
- Amazon Redshift (warehoused product exposure)

Multi-account strategies (via AWS Organizations) allow data products to span **multiple AWS Regions and organizational units** — enabling resilient, blast-radius-limited architectures.

---

## 🛠️ Technology Stack

| Layer | Technology | Version / Tier |
|---|---|---|
| Cloud Provider | AWS | Multi-AZ, `us-east-1` primary |
| Infrastructure as Code | Terraform | `>= 1.7.0` |
| ETL Runtime | AWS Glue (PySpark) | Glue 4.0 (Spark 3.3) |
| Query Engine | Amazon Athena (Trino) | Engine v3 |
| Governance | AWS Lake Formation | LF-TBAC + Cell-Level Security |
| Object Storage | Amazon S3 | Intelligent-Tiering |
| Columnar Format | Apache Parquet | Snappy Compression |
| ML Platform | Amazon SageMaker | Studio Domain |
| Visualization | Amazon QuickSight | Enterprise Edition |
| Secrets Management | AWS Secrets Manager | Automatic rotation enabled |
| Audit & Compliance | AWS CloudTrail + AWS Config | Organization-level trail |
| SIEM Integration | Amazon Security Lake | OCSF format forwarding |

---

## ⚙️ Environment & Configuration Reference

> **🔐 SECURITY DIRECTIVE:** No credentials, account IDs, or S3 bucket ARNs are to be hardcoded in source code or committed to version control. All sensitive configuration values are injected at runtime via **AWS Secrets Manager** or **AWS Systems Manager Parameter Store** with strict IAM resource-based policies. Non-compliance will trigger an automated PR block via the pre-commit hook.

### Core Platform Variables

| Variable | Description | Example Value | Required |
|---|---|---|---|
| `AWS_REGION` | Primary deployment region | `us-east-1` | ✅ |
| `AWS_ACCOUNT_ID` | Target AWS account identifier | Retrieved from STS | ✅ |
| `TF_VAR_raw_bucket_name` | S3 bucket for raw ingestion zone | `corp-datalake-raw-prod` | ✅ |
| `TF_VAR_curated_bucket_name` | S3 bucket for curated output zone | `corp-datalake-curated-prod` | ✅ |
| `TF_VAR_athena_results_bucket` | S3 bucket for Athena query results | `corp-athena-results-prod` | ✅ |
| `TF_VAR_glue_role_arn` | IAM Role ARN for AWS Glue execution | `arn:aws:iam::ACCOUNT:role/GlueETLRole` | ✅ |
| `TF_VAR_lf_admin_role_arn` | Lake Formation administrator IAM Role ARN | `arn:aws:iam::ACCOUNT:role/LFAdminRole` | ✅ |
| `TF_VAR_environment` | Deployment environment tag | `production` / `development` | ✅ |
| `TF_VAR_glue_job_schedule` | Cron expression for ETL trigger | `cron(0/10 * * * ? *)` | ✅ |
| `TF_VAR_parquet_compression` | Parquet output compression codec | `SNAPPY` | ✅ |
| `TF_VAR_enable_lf_tbac` | Toggle LF-TBAC enforcement | `true` | ✅ |
| `TF_VAR_cloudtrail_bucket` | Dedicated CloudTrail delivery bucket | `corp-cloudtrail-prod` | ✅ |

### Consumer Tier Configuration

| Consumer Tier | IAM Principal | Visible Columns | Accessible Tags |
|---|---|---|---|
| **Standard** | `Consumer_A` role | 11 / 13 (excludes `rank`, `rating_filled`) | `Environment=Production`, `Confidential=False`, `Customer=Regular` |
| **Enterprise** | `Consumer_B` role | 13 / 13 (full column visibility) | `Environment=Production`, `Confidential=False`, `Customer=Regular+Enterprise` |
| **Data Engineer** | `DataEngineer` role | 13 / 13 + schema management | All tags, all values |

---

## 🏷️ LF-Tag Governance Matrix

Lake Formation tag taxonomy is the **sole mechanism** for data access control in the curated zone. The following matrix defines all authorized tag combinations and their associated resource grants.

### Tag Taxonomy Definition

| Tag Key | Authorized Values | Scope |
|---|---|---|
| `Environment` | `Development`, `Production` | Database-level |
| `Confidential` | `True`, `False` | Table-level |
| `Customer` | `Regular`, `Enterprise` | Column-level |

### Resource-to-Tag Mapping

| Resource | Resource Type | Assigned LF-Tags |
|---|---|---|
| `transform-movies-db` | Database | `Environment=Production` |
| `movies` | Table | `Environment=Production` *(inherited)*, `Confidential=False` |
| Columns: `year`, `title`, `directors_0`, `genres_0`, `genres_1`, `running_time_secs`, `actors_0`, `actors_1`, `actors_2`, `directors_1`, `directors_2` | Column Set | `Customer=Regular` |
| Columns: `rank`, `rating_filled` | Column Set | `Customer=Enterprise` |

### Principal Permission Grants

| Principal | Tag Expression | Permission Level |
|---|---|---|
| `Consumer_A` | `Environment=Production` | `Database: Describe` |
| `Consumer_A` | `Confidential=False AND Customer=Regular` | `Table: Select, Describe` |
| `Consumer_B` | `Environment=Production` | `Database: Describe` |
| `Consumer_B` | `Confidential=False AND Customer=(Regular OR Enterprise)` | `Table: Select, Describe` |
| `DataEngineer` | All tags, all values | `Database + Table: Super (DDL + DML)` |

> **⚠️ GOVERNANCE NOTE — AND vs OR Semantics:** When multiple LF-tags are specified within a **single grant**, they are evaluated as a logical **AND**. A principal is only granted access to resources bearing **all** specified tag-value pairs simultaneously. To grant access to resources bearing **either** of two tag values (OR semantics), the values must be included within the **same tag key's value list** in a single grant — as demonstrated in Consumer_B's `Customer=Regular,Enterprise` grant.

---

## 🚀 Deployment Guide

### Prerequisites

Ensure the following tooling is installed and authenticated before initiating deployment:

```bash
# Verify toolchain versions
aws --version          # >= 2.13.0
terraform version      # >= 1.7.0
python --version       # >= 3.11
jq --version           # >= 1.6

# Validate AWS authentication and identity
aws sts get-caller-identity

# Expected output must reflect the Lake Formation admin role or
# a principal with sufficient permissions to register S3 locations
# and bootstrap Lake Formation settings.
```

### Step 1: Repository Initialization

```bash
git clone https://github.com/your-org/enterprise-datalake-platform.git
cd enterprise-datalake-platform

# Install pre-commit security hooks (tfsec, checkov, detect-secrets)
pip install pre-commit --break-system-packages
pre-commit install

# Verify hook installation
pre-commit run --all-files
```

### Step 2: Terraform Infrastructure Provisioning

```bash
cd terraform/

# Initialize providers and remote state backend (S3 + DynamoDB locking)
terraform init \
  -backend-config="bucket=corp-tfstate-prod" \
  -backend-config="key=datalake/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=corp-tfstate-lock"

# Validate configuration
terraform validate

# Security gate: run tfsec static analysis before plan
tfsec . --minimum-severity HIGH

# Generate execution plan
terraform plan \
  -var-file="environments/production.tfvars" \
  -out=tfplan.binary

# Review plan output critically. Pay special attention to:
# - IAM role trust policies
# - S3 bucket ACL and encryption settings
# - Lake Formation location registration

# Apply with explicit approval
terraform apply tfplan.binary
```

> **⚠️ CRITICAL — Bootstrap Order Dependency:** Lake Formation resource registration requires that the `AWSServiceRoleForLakeFormationDataAccess` service-linked role exists in the account **before** Terraform applies LF data location resources. If deploying into a net-new account, execute `aws lakeformation put-data-lake-settings` manually to complete the initial Lake Formation administrator bootstrap prior to running `terraform apply`.

### Step 3: Lake Formation Bootstrap

```bash
# Set the Lake Formation administrator (must match TF_VAR_lf_admin_role_arn)
aws lakeformation put-data-lake-settings \
  --data-lake-settings '{
    "DataLakeAdmins": [
      {"DataLakePrincipalIdentifier": "arn:aws:iam::ACCOUNT_ID:role/LFAdminRole"}
    ],
    "CreateDatabaseDefaultPermissions": [],
    "CreateTableDefaultPermissions": []
  }' \
  --region us-east-1

# CRITICAL: Validate that IAMAllowedPrincipals default permissions
# have been cleared. Non-empty CreateDatabaseDefaultPermissions or
# CreateTableDefaultPermissions will re-enable IAM passthrough,
# bypassing LF-TBAC entirely.
aws lakeformation get-data-lake-settings --region us-east-1 | jq '.DataLakeSettings'
```

### Step 4: Validate ETL Pipeline Execution

```bash
# Trigger the transform-movies Glue job manually for initial validation
aws glue start-job-run \
  --job-name transform-movies \
  --region us-east-1

# Poll for completion (replace JOB_RUN_ID with output of above command)
watch -n 30 "aws glue get-job-run \
  --job-name transform-movies \
  --run-id \$JOB_RUN_ID \
  --region us-east-1 | jq '.JobRun.JobRunState'"

# Expected terminal states: SUCCEEDED | FAILED | STOPPED
# On SUCCEEDED, proceed to Step 5
```

---

## ⚙️ ETL Pipeline: Glue Job Architecture

The `transform-movies` AWS Glue job is a multi-node PySpark pipeline scheduled via a **Glue Trigger** (cron: `every 10 minutes`). It is the primary data transformation layer between the raw ingestion zone and the governed curated zone.

### Node Execution Graph

| Node ID | Type | Description |
|---|---|---|
| `Node-1` | **DataSource (S3)** | Reads `s3://raw-bucket/data/movies_csv/movies.csv` |
| `Node-2` | **DataSource (Glue Catalog)** | Reads existing curated table from `transform-movies-db.movies` |
| `Node-3` | **ML Transform (FillMissingValues)** | Imputes `NULL` values in `rating` column → produces `rating_filled` |
| `Node-4` | **ApplyMapping (Raw)** | Normalizes raw schema to canonical curated schema |
| `Node-5` | **ApplyMapping (Catalog)** | Normalizes catalog schema for merge operation |
| `Node-6` | **Custom PySpark Transform** | Computes delta: appends only **net-new rows** not present in curated table (incremental load pattern) |
| `Node-7` | **SelectFromCollection** | Extracts the `new_rows` DataFrame from the transform collection output |
| `Node-8` | **DataSink (Glue Catalog)** | Writes Parquet output to curated S3 location; updates Glue Data Catalog |

### Key Engineering Decisions

- **Incremental Load (Node-6):** Custom PySpark logic performs a left-anti join between the incoming raw DataFrame and the existing curated catalog DataFrame. Only records absent from the curated dataset are written, preventing duplicate record accumulation on each scheduled run.
- **ML-Assisted Imputation (Node-3):** `awsglueml.transforms.FillMissingValues` uses a trained imputation model to infer missing `rating` values rather than defaulting to mean/median imputation — materially improving data quality for downstream ML recommendation models.
- **Output Format:** All curated writes are **Apache Parquet with Snappy compression** partitioned by ingestion date to enable partition pruning on time-range queries.

---

## 🔐 Access Control Implementation (LF-TBAC)

### Revocation of IAM Passthrough

> **⚠️ MANDATORY FIRST STEP:** Before any LF-tag grants take effect, **all** `IAMAllowedPrincipals` permissions on the `transform-movies-db` database and `movies` table **must be revoked**. Failure to revoke these permissions means IAM policies alone can bypass LF-TBAC grants, rendering the entire column-level access control model non-functional.

```bash
# Identify and revoke IAMAllowedPrincipals database permission
aws lakeformation revoke-permissions \
  --principal '{"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"}' \
  --resource '{"Database": {"Name": "transform-movies-db"}}' \
  --permissions "ALL" \
  --region us-east-1

# Identify and revoke IAMAllowedPrincipals table permission
aws lakeformation revoke-permissions \
  --principal '{"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"}' \
  --resource '{"Table": {"DatabaseName": "transform-movies-db", "Name": "movies"}}' \
  --permissions "ALL" \
  --region us-east-1
```

### Granting LF-TBAC Permissions (Consumer_A — Standard Tier)

```bash
# Step 1: Grant database Describe on Environment=Production
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::ACCOUNT_ID:role/Consumer_A"}' \
  --resource '{
    "LFTagPolicy": {
      "ResourceType": "DATABASE",
      "Expression": [{"TagKey": "Environment", "TagValues": ["Production"]}]
    }
  }' \
  --permissions "DESCRIBE" \
  --region us-east-1

# Step 2: Grant table Select+Describe on Confidential=False AND Customer=Regular
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::ACCOUNT_ID:role/Consumer_A"}' \
  --resource '{
    "LFTagPolicy": {
      "ResourceType": "TABLE",
      "Expression": [
        {"TagKey": "Confidential", "TagValues": ["False"]},
        {"TagKey": "Customer",     "TagValues": ["Regular"]}
      ]
    }
  }' \
  --permissions "SELECT" "DESCRIBE" \
  --region us-east-1
```

### Granting LF-TBAC Permissions (Consumer_B — Enterprise Tier)

```bash
# Step 1: Grant database Describe on Environment=Production (identical to Consumer_A)
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::ACCOUNT_ID:role/Consumer_B"}' \
  --resource '{
    "LFTagPolicy": {
      "ResourceType": "DATABASE",
      "Expression": [{"TagKey": "Environment", "TagValues": ["Production"]}]
    }
  }' \
  --permissions "DESCRIBE" \
  --region us-east-1

# Step 2: Grant table Select+Describe on Confidential=False AND Customer=(Regular OR Enterprise)
# Note: Multi-value Customer tag enables OR semantics within that single key
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::ACCOUNT_ID:role/Consumer_B"}' \
  --resource '{
    "LFTagPolicy": {
      "ResourceType": "TABLE",
      "Expression": [
        {"TagKey": "Confidential", "TagValues": ["False"]},
        {"TagKey": "Customer",     "TagValues": ["Regular", "Enterprise"]}
      ]
    }
  }' \
  --permissions "SELECT" "DESCRIBE" \
  --region us-east-1
```

---

## ✅ Consumer Tier Verification Runbook

Use the following Athena queries to validate LF-TBAC enforcement correctness after any permission change. These queries must be executed under the respective consumer IAM principal context.

### Standard Tier (Consumer_A) — Expected: 11 columns, no `rank` or `rating_filled`

```sql
-- Verify column visibility: MUST NOT contain 'rank' or 'rating_filled'
SELECT * FROM "transform-movies-db"."movies" LIMIT 10;

-- Verify total record count accessible to standard consumer
SELECT COUNT(*) AS total_accessible_records FROM "transform-movies-db"."movies";

-- Attempt to access restricted column (MUST return AccessDeniedException)
SELECT rank FROM "transform-movies-db"."movies" LIMIT 1;
```

**Expected Results:**
- Query 1: Returns 10 rows with **11 columns** (no `rank`, no `rating_filled`)
- Query 2: Returns `4609`
- Query 3: **Fails with AccessDeniedException** — this is the correct and required behavior

---

### Enterprise Tier (Consumer_B) — Expected: 13 columns, full visibility

```sql
-- Verify full column visibility: MUST include 'rank' and 'rating_filled'
SELECT * FROM "transform-movies-db"."movies" LIMIT 10;

-- Validate enterprise-exclusive columns are accessible
SELECT rank, rating_filled, title
FROM "transform-movies-db"."movies"
WHERE rank IS NOT NULL
ORDER BY rank ASC
LIMIT 20;

-- Verify total record count (must match data engineer view)
SELECT COUNT(*) AS total_records FROM "transform-movies-db"."movies";
```

**Expected Results:**
- Query 1: Returns 10 rows with **13 columns** (including `rank` and `rating_filled`)
- Query 2: Returns ranked records with populated `rating_filled` values
- Query 3: Returns `4609`

---

## 🔒 Security Hardening & Compliance Controls

### Security Control Inventory

| Control | Implementation | Compliance Mapping |
|---|---|---|
| Encryption at Rest | S3 SSE-KMS (Customer Managed Key) | SOC2 CC6.1, HIPAA §164.312(a)(2)(iv) |
| Encryption in Transit | TLS 1.2+ enforced via S3 bucket policy | SOC2 CC6.7 |
| Column-Level Access Control | LF-TBAC `Customer` tag on `rank`, `rating_filled` | SOC2 CC6.3, PCI-DSS 7.1 |
| Least Privilege IAM | No wildcard `*` actions; resource-scoped policies only | CIS AWS Benchmark 1.x |
| Immutable Audit Trail | CloudTrail organization-level trail, S3 Object Lock | SOC2 CC7.2, GDPR Art. 5(2) |
| IAM Passthrough Disabled | All `IAMAllowedPrincipals` grants revoked | Zero-Trust Data Access |
| Secrets Rotation | Secrets Manager 30-day auto-rotation | SOC2 CC6.1 |
| Athena Workgroup Isolation | Per-consumer-tier workgroups with result encryption | SOC2 CC6.6 |
| Data Residency | Explicit S3 bucket region lock via `aws:RequestedRegion` condition | GDPR Art. 44 |
| Vulnerability Scanning | `tfsec` + `checkov` in CI gate; fail on HIGH/CRITICAL | DevSecOps Shift-Left |

### Threat Model & Mitigation

| Threat Vector | Attack Scenario | Mitigation |
|---|---|---|
| **Privilege Escalation** | Consumer_A assumes Consumer_B role to access `rank` column | IAM permission boundary + CloudTrail anomaly alert |
| **Data Exfiltration** | Athena results written to attacker-controlled S3 bucket | Workgroup output location locked; S3 bucket policy denies cross-account PutObject |
| **LF-Tag Tampering** | Unauthorized removal of `Confidential=True` tag to expose sensitive data | LF admin role restricted to automated pipeline; manual changes require MFA + CAB approval |
| **ETL Job Injection** | Malicious PySpark code injected via compromised source CSV | Glue job script stored in version-controlled, access-restricted S3 prefix; script hash verified at runtime |
| **Catalog Poisoning** | Attacker registers a rogue S3 location to the data lake | Lake Formation location registration restricted to `LFAdminRole` only |

---

## 📓 Operational Runbooks

### Runbook: Add a New Data Consumer

```bash
# 1. Create IAM role for new consumer (use Terraform module)
# terraform/modules/consumer_role/main.tf

# 2. Register consumer in Lake Formation permissions
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::ACCOUNT_ID:role/NEW_CONSUMER_ROLE"}' \
  --resource '{"LFTagPolicy": {"ResourceType": "DATABASE", "Expression": [...]}}' \
  --permissions "DESCRIBE"

# 3. Validate access with dry-run Athena query under new principal
aws athena start-query-execution \
  --query-string "SELECT COUNT(*) FROM \"transform-movies-db\".\"movies\"" \
  --work-group "consumer-standard-wg" \
  --result-configuration "OutputLocation=s3://corp-athena-results-prod/validation/"

# 4. Record grant in CMDB (mandatory for SOC2 access review evidence)
```

### Runbook: Rotate LF-Tag Taxonomy (Adding New Tag Key)

```bash
# Step 1: Define new tag in Lake Formation
aws lakeformation create-lf-tag \
  --tag-key "DataClassification" \
  --tag-values "Public" "Internal" "Restricted" "TopSecret"

# Step 2: Apply to target resources (databases, tables, or columns)
aws lakeformation add-lf-tags-to-resource \
  --resource '{"Table": {"DatabaseName": "transform-movies-db", "Name": "movies"}}' \
  --lf-tags '[{"TagKey": "DataClassification", "TagValues": ["Internal"]}]'

# Step 3: Update existing principal grants to include new tag if required
# Step 4: Update Terraform state to reflect new tag (avoid configuration drift)
# Step 5: Run compliance validation script
```

---

## ✅ Pre-Production Checklist

Use this checklist for every deployment to production. **All items must be checked before the change window closes.**

### Infrastructure

- [ ] Terraform plan reviewed and approved by ≥2 senior engineers
- [ ] `tfsec` scan returns zero `HIGH` or `CRITICAL` findings
- [ ] `checkov` scan returns zero `FAILED` checks on security policies
- [ ] Remote state backend S3 bucket versioning and MFA-delete confirmed enabled
- [ ] DynamoDB state lock table verified operational

### Lake Formation Governance

- [ ] All `IAMAllowedPrincipals` grants confirmed revoked on target database and tables
- [ ] LF-Tag taxonomy matches approved governance schema in this README
- [ ] Consumer_A column visibility confirmed: **11 columns** (no `rank`, no `rating_filled`)
- [ ] Consumer_B column visibility confirmed: **13 columns** (full schema)
- [ ] `AccessDeniedException` confirmed when Consumer_A queries `rank` column
- [ ] Data Engineer role confirmed to have super-admin data catalog permissions

### ETL Pipeline

- [ ] `transform-movies` Glue job last run status: **SUCCEEDED**
- [ ] Row count post-ETL matches expected baseline: **4609 records**
- [ ] Parquet output files confirmed in curated S3 bucket (`s3://curated-bucket/movies/`)
- [ ] Glue trigger `transform-movies-trigger` cron expression validated: `cron(0/10 * * * ? *)`
- [ ] Incremental load logic confirmed: no duplicate rows on re-execution

### Security & Compliance

- [ ] CloudTrail organizational trail active and delivering to dedicated audit bucket
- [ ] S3 audit bucket Object Lock (Compliance Mode) verified
- [ ] AWS Config rules enabled: `lakeformation-data-lake-settings-check`, `glue-job-s3-kms-encryption-enabled`
- [ ] Athena workgroup per-query data scanned limit configured (cost + exfiltration control)
- [ ] KMS key rotation enabled on all customer-managed keys
- [ ] Secrets Manager rotation confirmed scheduled for all service credentials

### Operational Readiness

- [ ] Runbooks reviewed and accessible in on-call wiki
- [ ] PagerDuty integration for AWS Budgets billing alert confirmed
- [ ] Rollback procedure documented and tested in staging environment
- [ ] CMDB updated with new consumer grants (SOC2 access review evidence)

---

## 📚 Additional Resources

| Resource | URL |
|---|---|
| AWS Lake Formation Developer Guide | https://docs.aws.amazon.com/lake-formation/latest/dg/what-is-lake-formation.html |
| LF-TBAC Permissions Model | https://docs.aws.amazon.com/lake-formation/latest/dg/TBAC-permissions-model.html |
| Assigning LF-Tags to Data Catalog Resources | https://docs.aws.amazon.com/lake-formation/latest/dg/tagging-catalog-resource.html |
| AWS Glue Triggers Reference | https://docs.aws.amazon.com/glue/latest/dg/trigger-job.html |
| Apache Parquet Specification | https://parquet.apache.org/docs/file-format/ |
| Amazon Data Zone — Data Mesh | https://docs.aws.amazon.com/datazone/latest/userguide/what-is-datazone.html |
| CIS AWS Foundations Benchmark | https://www.cisecurity.org/benchmark/amazon_web_services |
| AWS Well-Architected Framework — Data Analytics Lens | https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/analytics-lens.html |

---
```

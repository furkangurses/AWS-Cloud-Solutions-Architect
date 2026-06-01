# aws-lake-formation-tbac

![AWS](https://img.shields.io/badge/AWS-Lake%20Formation-FF9900?logo=amazonaws&logoColor=white)
![Glue](https://img.shields.io/badge/AWS-Glue%20ETL-8A2BE2?logo=amazonaws&logoColor=white)
![Athena](https://img.shields.io/badge/AWS-Athena-1A73E8?logo=amazonaws&logoColor=white)
![IaC](https://img.shields.io/badge/Access%20Control-LF--TBAC-green)
![Status](https://img.shields.io/badge/status-reference--implementation-blue)

> **Tag-Based Access Control (LF-TBAC) for a governed data lake on AWS.**  
> This repo documents the architecture, implementation, and operational notes for applying Lake Formation LF-tag policies to a multi-tier data consumer model — separating standard-tier and premium-tier access to a shared curated dataset without duplicating data or managing per-column IAM policies.

---

## Table of Contents

- [Architecture](#architecture)
- [Data Pipeline](#data-pipeline)
- [LF-Tag Taxonomy](#lf-tag-taxonomy)
- [Tag-to-Resource Mapping](#tag-to-resource-mapping)
- [Consumer Permission Matrix](#consumer-permission-matrix)
- [Glue Job Reference](#glue-job-reference)
- [Athena Validation Queries](#athena-validation-queries)
- [Deployment Checklist](#deployment-checklist)
- [Engineering Notes](#engineering-notes)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Data Producer Layer                        │
│                                                                     │
│   S3 (raw zone)           AWS Glue ETL           S3 (curated zone) │
│  s3://datalake/raw/   ──► transform-movies ──►  s3://datalake/out/ │
│                                  │                                  │
│                           Glue Data Catalog                         │
│                         transform-movies-db                         │
│                               movies (table)                        │
└─────────────────────────┬───────────────────────────────────────────┘
                          │  Lake Formation TBAC
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
  Data Engineer       Consumer_A      Consumer_B
  (full access)     (standard tier)  (enterprise tier)
  Athena queries    11 columns        13 columns
```

**Key design principle:** A single physical table backs all consumers. Column-level visibility is enforced by Lake Formation at query time via LF-tags — no materialized views, no duplicate tables, no per-user IAM policies on S3 prefixes.

---

## Data Pipeline

### Glue Job: `transform-movies`

The ETL job runs on a 10-minute cron schedule via a Glue Trigger (`transform-movies-trigger`) and performs idempotent incremental loads.

**Node graph summary:**

| Node | Type | Description |
|------|------|-------------|
| 1 | S3 Source | Reads `data/movies_csv/movies.csv` from raw S3 bucket |
| 2 | Glue Data Catalog Source | Reads existing curated table for delta detection |
| 3 | FillMissingValues | Imputes nulls in `rating` → writes to `rating_filled` column |
| 4–5 | ApplyMapping | Schema normalization and column type casting |
| 6 | Custom PySpark Transform | Computes net-new rows (`new_df = raw_df.subtract(catalog_df)`) |
| 7 | SelectFromCollection | Extracts new-rows dataframe from transform output |
| 8 | Glue Data Catalog Sink | Appends to `transform-movies-db.movies` |

**Cron schedule (Glue Trigger):**

```
cron(0/10 * * * ? *)   # every 10 minutes, all hours, every day
```

**Schema after transformation:**

```
year             bigint
title            string
directors_0      string
directors_1      string
genres_0         string
genres_1         string
running_time_secs bigint
actors_0         string
actors_1         string
actors_2         string
rank             bigint      ← enterprise only
rating_filled    double      ← enterprise only (imputed)
```

> **Note on `rating_filled`:** The `FillMissingValues` node from `awsglueml.transforms` uses ML-based imputation, not a simple mean fill. This column is considered derived/enriched and is therefore gated behind the enterprise tier.

---

## LF-Tag Taxonomy

Three tag keys are defined at the Lake Formation catalog level. All tags are centrally managed and must not be applied ad hoc outside this taxonomy.

```
Lake Formation → Permissions → LF-Tags and Permissions
```

| Key | Allowed Values | Purpose |
|-----|---------------|---------|
| `Environment` | `Development`, `Production` | Prevents dev/staging resources from being queried in production workflows |
| `Customer` | `Regular`, `Enterprise` | Controls column-level visibility by subscription tier |
| `Confidential` | `True`, `False` | Flags PII or commercially sensitive attributes; consumers without `False` access cannot reach those resources |

> **Governance rule:** New catalog resources (databases, tables, columns) must have all three tag keys assigned before being granted to any consumer principal. Untagged resources default to no-access under this IAM-revoked model.

---

## Tag-to-Resource Mapping

### Database: `transform-movies-db`

| Tag Key | Value |
|---------|-------|
| `Environment` | `Production` |

### Table: `movies`

| Tag Key | Value | Source |
|---------|-------|--------|
| `Environment` | `Production` | Inherited from database |
| `Confidential` | `False` | Explicitly assigned |

### Column-level tags: `movies`

| Columns | `Customer` Value |
|---------|-----------------|
| `year`, `title`, `directors_0`, `directors_1`, `genres_0`, `genres_1`, `running_time_secs`, `actors_0`, `actors_1`, `actors_2` | `Regular` |
| `rank`, `rating_filled` | `Enterprise` |

> **Inheritance behaviour:** LF-tag values set on a database propagate to child tables and columns unless explicitly overridden at the lower level. The `Environment=Production` tag on the database is therefore inherited by the `movies` table — no need to re-assign it at the table level.

---

## Consumer Permission Matrix

IAM-based (`IAMAllowedPrincipals`) access was revoked from all catalog resources prior to creating TBAC grants. All access is now exclusively governed by LF-tag expressions.

| Permission Grant | Principal | LF-Tag Expression | Permission Type |
|-----------------|-----------|-------------------|-----------------|
| DB describe | `Consumer_A` | `Environment=Production` | `Database:Describe` |
| Table/column read | `Consumer_A` | `Confidential=False AND Customer=Regular` | `Table:Select,Describe` |
| DB describe | `Consumer_B` | `Environment=Production` | `Database:Describe` |
| Table/column read | `Consumer_B` | `Confidential=False AND (Customer=Regular OR Customer=Enterprise)` | `Table:Select,Describe` |

**Resulting column visibility per consumer:**

| Column | Data Engineer | Consumer_A (Standard) | Consumer_B (Enterprise) |
|--------|:---:|:---:|:---:|
| `year` | ✅ | ✅ | ✅ |
| `title` | ✅ | ✅ | ✅ |
| `directors_0` | ✅ | ✅ | ✅ |
| `directors_1` | ✅ | ✅ | ✅ |
| `genres_0` | ✅ | ✅ | ✅ |
| `genres_1` | ✅ | ✅ | ✅ |
| `running_time_secs` | ✅ | ✅ | ✅ |
| `actors_0` | ✅ | ✅ | ✅ |
| `actors_1` | ✅ | ✅ | ✅ |
| `actors_2` | ✅ | ✅ | ✅ |
| `rank` | ✅ | ❌ | ✅ |
| `rating_filled` | ✅ | ❌ | ✅ |

> **Multi-value tag grants:** When a single permission grant specifies multiple values for the same key (e.g., `Customer=Regular,Enterprise`), Lake Formation evaluates this as an **OR** across values for that key. When two *different* keys appear in the same grant (e.g., `Confidential=False` + `Customer=Regular`), they are evaluated as an **AND**. This distinction is critical when designing the permission model.

---

## Glue Job Reference

### Manual trigger (one-off run)

```bash
aws glue start-job-run \
  --job-name transform-movies \
  --region us-east-1
```

### Check last 5 run statuses

```bash
aws glue get-job-runs \
  --job-name transform-movies \
  --max-results 5 \
  --query 'JobRuns[*].{RunId:Id,Status:JobRunState,Started:StartedOn,Duration:ExecutionTime}' \
  --output table
```

### Describe the trigger

```bash
aws glue get-trigger \
  --name transform-movies-trigger \
  --query 'Trigger.{Schedule:Schedule,State:State,Actions:Actions}'
```

Expected output:

```json
{
  "Schedule": "cron(0/10 * * * ? *)",
  "State": "ACTIVATED",
  "Actions": [{ "JobName": "transform-movies" }]
}
```

---

## Athena Validation Queries

All queries target the `primary` workgroup. Output location is pre-configured via workgroup settings.

### Baseline row count (admin/engineer view)

```sql
SELECT COUNT(*) AS total_rows
FROM "transform-movies-db"."movies";
-- Expected: 4609
```

### Spot-check full schema (engineer view)

```sql
SELECT *
FROM "transform-movies-db"."movies"
LIMIT 10;
-- Expected: 13 columns returned
```

### Verify standard-tier column restriction

```sql
-- Run as Consumer_A (standard subscription IAM user)
SELECT *
FROM "transform-movies-db"."movies"
LIMIT 10;
-- Expected: 11 columns — rank and rating_filled must NOT appear
```

### Verify enterprise-tier full access

```sql
-- Run as Consumer_B (enterprise subscription IAM user)
SELECT rank, rating_filled, title
FROM "transform-movies-db"."movies"
WHERE rating_filled IS NOT NULL
ORDER BY rank ASC
LIMIT 20;
-- Expected: Results returned without permission error; 13 columns visible
```

### Confirm imputed column coverage

```sql
SELECT
  COUNT(*)                                          AS total_rows,
  COUNT(rating_filled)                              AS non_null_rating_filled,
  ROUND(COUNT(rating_filled) * 100.0 / COUNT(*), 2) AS fill_rate_pct
FROM "transform-movies-db"."movies";
```

---

## Deployment Checklist

Use this when replicating this pattern to a new catalog database or AWS account.

- [ ] Register S3 data lake location with Lake Formation (`Register location`)
- [ ] Revoke `IAMAllowedPrincipals` from all target databases and tables
- [ ] Define LF-tag taxonomy (keys + allowed values) at account level
- [ ] Apply `Environment` tag to database; confirm child tables inherit it
- [ ] Apply `Confidential` tag to tables; apply `Customer` tag at column granularity
- [ ] Grant `Database:Describe` to each consumer principal using `Environment` tag
- [ ] Grant `Table:Select,Describe` using multi-key AND expressions per tier
- [ ] Validate with per-consumer Athena queries — confirm column counts match expected matrix
- [ ] Confirm Glue Trigger is `ACTIVATED`; verify at least one successful job run before opening consumer access

---

## Engineering Notes

### Why TBAC over resource-based Lake Formation permissions

The alternative — granting permissions directly on named databases/tables/columns — does not scale in a data mesh or data product model. With TBAC:

- A new table added to the catalog inherits appropriate access **automatically** if tags are applied correctly at creation time.
- Adding a new consumer tier requires a single new permission grant against tag expressions, not N grants across N catalog resources.
- Tags survive table renames and schema evolutions; named resource grants do not.

### IAM vs. Lake Formation: the handoff

Before Lake Formation can enforce column-level access, IAM-based `IAMAllowedPrincipals` must be fully revoked. As long as `IAMAllowedPrincipals` grants exist, any IAM principal with `glue:GetTable` can bypass Lake Formation filters. This is the most common misconfiguration in Lake Formation deployments.

### FillMissingValues and the `rating_filled` column

The `awsglueml.transforms.FillMissingValues` transform invokes an AutoML-based imputation model under the hood. For production workloads this has cost and latency implications — each Glue job run that processes new rows will incur a short ML inference step. For high-throughput pipelines, evaluate replacing this with a deterministic statistical fill and removing the ML dependency.

### Tag inheritance — caveat

Lake Formation tag inheritance is **one-directional** (parent → child) and **read-only** from the child's perspective. If you re-tag a database after tables already exist, the existing tables do **not** retroactively update; inheritance only applies at the time a child resource is created or when the parent tag is explicitly propagated. Always verify effective tags on child resources after any parent re-tagging event.

### Column-level grants and `SELECT *`

Athena consumers who issue `SELECT *` will silently receive only the columns their LF-tag grants permit. There is no error thrown for restricted columns — they simply do not appear in the result set. This is the intended behaviour but has implications for application code that relies on a fixed column index or `*` expansion. Downstream consumers should always select columns by name.

---

<details>
<summary>📋 LF-Tag CLI reference (expand)</summary>

### Create an LF-tag

```bash
aws lakeformation create-lf-tag \
  --tag-key "Customer" \
  --tag-values '["Regular","Enterprise"]' \
  --region us-east-1
```

### List all LF-tags

```bash
aws lakeformation list-lf-tags \
  --resource-share-type LOCAL \
  --region us-east-1
```

### Apply LF-tag to a table

```bash
aws lakeformation add-lf-tags-to-resource \
  --resource '{"Table":{"DatabaseName":"transform-movies-db","Name":"movies"}}' \
  --lf-tags '[{"TagKey":"Confidential","TagValues":["False"]}]' \
  --region us-east-1
```

### Apply LF-tag to specific columns

```bash
aws lakeformation add-lf-tags-to-resource \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "transform-movies-db",
      "Name": "movies",
      "ColumnNames": ["rank","rating_filled"]
    }
  }' \
  --lf-tags '[{"TagKey":"Customer","TagValues":["Enterprise"]}]' \
  --region us-east-1
```

### Grant TBAC table permissions to a principal

```bash
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier":"arn:aws:iam::123456789012:user/Consumer_B"}' \
  --resource '{
    "LFTagPolicy": {
      "ResourceType": "TABLE",
      "Expression": [
        {"TagKey":"Confidential","TagValues":["False"]},
        {"TagKey":"Customer","TagValues":["Regular","Enterprise"]}
      ]
    }
  }' \
  --permissions '["SELECT","DESCRIBE"]' \
  --region us-east-1
```

</details>

---

## Related Resources

- [Lake Formation TBAC permissions model](https://docs.aws.amazon.com/lake-formation/latest/dg/TBAC-overview.html)
- [Assigning LF-Tags to Data Catalog resources](https://docs.aws.amazon.com/lake-formation/latest/dg/TBAC-assigning-tags.html)
- [AWS Glue Triggers documentation](https://docs.aws.amazon.com/glue/latest/dg/trigger-job.html)
- [Column-level security in Lake Formation](https://docs.aws.amazon.com/lake-formation/latest/dg/column-level-security.html)

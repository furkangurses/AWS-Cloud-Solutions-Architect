# AWS Enterprise Landing Zone — Multi-Account Governance Framework

[![Architecture: Multi-Account](https://img.shields.io/badge/Architecture-Multi--Account%20Landing%20Zone-orange?style=flat-square&logo=amazonaws)](https://aws.amazon.com/organizations/)
[![Security: SCPs Enforced](https://img.shields.io/badge/Security-SCPs%20Enforced-critical?style=flat-square&logo=awslambda)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)
[![IAM: Identity%20Center%20SSO](https://img.shields.io/badge/IAM-Identity%20Center%20SSO-blue?style=flat-square&logo=amazoniam)](https://aws.amazon.com/iam/identity-center/)
[![Logging: CloudTrail%20Centralized](https://img.shields.io/badge/Logging-CloudTrail%20Centralized-green?style=flat-square&logo=amazonaws)](https://aws.amazon.com/cloudtrail/)
[![IaC: CloudFormation](https://img.shields.io/badge/IaC-CloudFormation-yellow?style=flat-square&logo=amazonaws)](https://aws.amazon.com/cloudformation/)
[![Compliance: Well-Architected](https://img.shields.io/badge/Compliance-Well--Architected%20Framework-lightgrey?style=flat-square)](https://aws.amazon.com/architecture/well-architected/)
[![Drift Prevention: Control Tower](https://img.shields.io/badge/Drift%20Prevention-Control%20Tower-purple?style=flat-square&logo=amazonaws)](https://aws.amazon.com/controltower/)

---

> **⚠️ PRODUCTION CRITICAL — OPERATIONAL CONTEXT**
> This document governs the design, implementation, and operational governance of a multi-tenant AWS enterprise environment serving a digital marketing agency with **isolated per-customer AWS accounts**. Every architectural decision herein is driven by the requirements of **blast-radius containment, compliance boundary enforcement, centralized observability, and zero-trust identity access management**. This is not a sandbox reference. Deviations from this specification must be reviewed by the Cloud Center of Excellence (CCoE) and documented via RFC.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Target Architecture Overview](#target-architecture-overview)
- [Account Topology & OU Hierarchy](#account-topology--ou-hierarchy)
- [Core Services Reference Matrix](#core-services-reference-matrix)
- [Identity & Access Management](#identity--access-management)
  - [Cross-Account IAM Roles](#cross-account-iam-roles)
  - [IAM Identity Center — Single Sign-On](#iam-identity-center--single-sign-on)
- [AWS Organizations & Service Control Policies](#aws-organizations--service-control-policies)
  - [SCP: Instance Type Enforcement](#scp-instance-type-enforcement)
  - [SCP: Root User Restriction](#scp-root-user-restriction)
  - [SCP: CloudTrail Tamper Prevention](#scp-cloudtrail-tamper-prevention)
  - [SCP: IP-Based Console Access Restriction](#scp-ip-based-console-access-restriction)
  - [Tag Policies: Uniform Tagging Enforcement](#tag-policies-uniform-tagging-enforcement)
- [Centralized Logging Architecture](#centralized-logging-architecture)
  - [AWS CloudTrail — Organization Trail](#aws-cloudtrail--organization-trail)
  - [AWS Config — Configuration Compliance](#aws-config--configuration-compliance)
  - [VPC Flow Logs](#vpc-flow-logs)
  - [Amazon GuardDuty](#amazon-guardduty)
- [Automated Account Provisioning Pipeline](#automated-account-provisioning-pipeline)
  - [AWS Control Tower — Account Vending Machine](#aws-control-tower--account-vending-machine)
  - [AWS Service Catalog — Infrastructure Vending Machine](#aws-service-catalog--infrastructure-vending-machine)
  - [AWS CloudFormation — IaC Engine](#aws-cloudformation--iac-engine)
- [Billing & Cost Governance](#billing--cost-governance)
- [Security Hardening Checklist](#security-hardening-checklist)
- [Operational Runbook Checklist](#operational-runbook-checklist)
- [Migration Roadmap](#migration-roadmap)
- [Further Reading](#further-reading)

---

## Problem Statement

The agency currently operates **all customer workloads from a single AWS account**, authenticated via the **root user**. This topology violates fundamental AWS security best practices and creates the following critical risks:

| Risk Category | Description | Business Impact |
|---|---|---|
| **Blast Radius** | A misconfiguration or compromise in one customer workload can affect all others | Data breach across multiple clients; SLA violations |
| **Billing Opacity** | No per-customer cost attribution exists | Inability to invoice accurately; budget overruns undetected |
| **IAM Sprawl** | All teams share a flat permission model within one account | Privilege escalation vectors; no least-privilege enforcement |
| **Regulatory Non-Compliance** | Customer data co-residency violates data isolation requirements | Legal liability; loss of enterprise contracts |
| **No Audit Trail Isolation** | A single CloudTrail (if configured) cannot isolate per-customer API activity | Forensic analysis failure during incident response |
| **Service Quota Contention** | All workloads compete for the same AWS service limits | Production degradation from unrelated development activity |

---

## Target Architecture Overview

The target state is a **AWS Well-Architected Multi-Account Landing Zone** anchored by a **Shared Services Account** that orchestrates all governance, identity, logging, and provisioning for the entire organization.
```

┌─────────────────────────────────────────────────────────────────┐ │ AWS Organizations │ │ (Management Account) │ │ │ │ ┌──────────────────────────────────────────────────────────┐ │ │ │ Shared Services Account │ │ │ │ ┌──────────────┐ ┌──────────────┐ ┌───────────────┐ │ │ │ │ │ AWS Control │ │ IAM Identity│ │ AWS │ │ │ │ │ │ Tower │ │ Center(SSO) │ │ CloudTrail │ │ │ │ │ └──────────────┘ └──────────────┘ │ (Org Trail) │ │ │ │ │ ┌──────────────┐ ┌──────────────┐ └───────────────┘ │ │ │ │ │ AWS Service │ │ CloudWatch │ ┌───────────────┐ │ │ │ │ │ Catalog │ │ Logs (Agg.) │ │ S3 Logs │ │ │ │ │ └──────────────┘ └──────────────┘ │ Bucket │ │ │ │ └──────────────────────────────────────└───────────────┘──┘ │ │ │ │ ┌──────────────────────────────────────────────────────────┐ │ │ │ OU: Customer-A │ │ │ │ ┌────────────────┐ ┌──────────────┐ ┌────────────────┐ │ │ │ │ │ OU: Dev │ │ OU: QA │ │ OU: Production │ │ │ │ │ │ ┌────────────┐ │ │ ┌──────────┐ │ │ ┌────────────┐ │ │ │ │ │ │ │Dev Workload│ │ │ │QA Account│ │ │ │Prod Account│ │ │ │ │ │ │ │Account │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │SCP: t2. │ │ │ └──────────┘ │ │ └────────────┘ │ │ │ │ │ │ │micro only │ │ └──────────────┘ └────────────────┘ │ │ │ │ │ └────────────┘ │ │ │ │ │ └────────────────┘ │ │ │ └──────────────────────────────────────────────────────────┘ │ │ ┌──────────────────────────────────────────────────────────┐ │ │ │ OU: Customer-B (Identical structure) │ │ │ └──────────────────────────────────────────────────────────┘ │ └─────────────────────────────────────────────────────────────────┘

```

---

## Account Topology & OU Hierarchy

The Organizational Unit (OU) hierarchy enforces **workload isolation at the account boundary level**. Each customer and each environment is a hard-isolated AWS account with independent IAM, billing, and service quota allocations.
```

Root (Management Account) │ ├── OU: Shared Services │ └── Account: shared-services-prod │ (AWS Organizations, IAM Identity Center, │ CloudTrail Org Trail, Control Tower, │ Service Catalog, CloudWatch Logs Aggregator) │ ├── OU: Security (Recommended) │ └── Account: security-audit │ (GuardDuty Aggregator, AWS Config Aggregator, │ Security Hub, Centralized S3 Logs [Immutable]) │ ├── OU: Customer-A │ ├── OU: Dev │ │ └── Account: customer-a-dev-workload-[uid] │ │ (SCP: DenyNonT2Micro, Service Catalog) │ ├── OU: QA │ │ └── Account: customer-a-qa-[uid] │ └── OU: Production │ └── Account: customer-a-prod-[uid] │ (Restrictive SCPs, CCoE Access Only) │ ├── OU: Customer-B │ └── (Identical structure to Customer-A) │ └── OU: Sandbox (Optional) └── Account: sandbox-[engineer-alias]-[uid] (Disconnected from corporate services, time-bounded)

```

> **🔒 SECURITY NOTE — OU Movement Restriction**
> An SCP **must** be applied at the Root OU level to restrict the `organizations:MoveAccount` API action to CCoE-authorized IAM principals only. Without this control, a developer could move a Dev account into the Production OU — temporarily bypassing stricter production SCPs — to launch unauthorized resources, then move it back. This is a known privilege escalation vector.

---

## Core Services Reference Matrix

| AWS Service | Scope | Primary Role | Governed By |
|---|---|---|---|
| **AWS Organizations** | Org-wide | Account lifecycle management, SCP enforcement | Management Account |
| **IAM Identity Center** | Org-wide | Workforce SSO, Permission Set assignment | Shared Services Account |
| **AWS Control Tower** | Org-wide | Landing Zone, account vending, drift detection | Shared Services Account |
| **AWS CloudFormation** | Per-account | IaC engine for all resource provisioning | Control Tower StackSets |
| **AWS Service Catalog** | Per-account | Pre-approved product portfolio for developers | Shared Services Account |
| **AWS CloudTrail** | Org-wide | Immutable API audit trail (Organization Trail) | Shared Services S3 Bucket |
| **AWS Config** | Per-account | Resource configuration state & compliance rules | Config Aggregator (Security Account) |
| **VPC Flow Logs** | Per-VPC | Network traffic capture for forensics | CloudWatch Logs / S3 |
| **Amazon GuardDuty** | Org-wide | Threat detection & ML-based anomaly identification | Security Account (Delegated Admin) |
| **IAM Roles** | Cross-account | Temporary credential vending via AssumeRole | Per-account, Trust Policy governed |
| **CloudWatch** | Per-account + Aggregated | Billing alarms, application metrics, log streams | Shared Services Account |

---

## Identity & Access Management

### Cross-Account IAM Roles

AWS IAM Roles are the **foundational cross-account authentication primitive**. They eliminate the anti-pattern of replicating IAM Users across every account. A role provides **temporary, scoped credentials** via the `sts:AssumeRole` API call — no long-lived access keys, no shared passwords.

**How AssumeRole Works (Cross-Account Flow):**
```

Account B (Trusted Account) Account A (Trusting Account) ┌─────────────────────┐ ┌─────────────────────────────┐ │ IAM User: rafael │ │ IAM Role: AdminRole │ │ │ │ ┌─────────────────────────┐ │ │ Attached Policy: │ ──AssumeRole─▶│ │ Trust Policy: │ │ │ sts:AssumeRole │ │ │ { │ │ │ on arn:aws:iam:: │ │ │ "Principal": { │ │ │ <AccountA>:role/ │ │ │ "AWS": "arn:aws:iam │ │ │ AdminRole │ │ │ ::<AccountB>:user/ │ │ │ │◀─Temp Creds──│ │ rafael" │ │ └─────────────────────┘ │ │ } │ │ │ │ } │ │ │ └─────────────────────────┘ │ │ Permission Policy: │ │ AdministratorAccess │ └─────────────────────────────┘

```

**Key IAM Role Taxonomy:**

| Role Type | Description | Common Use Case |
|---|---|---|
| **Cross-Account Role** | Trusted by a principal in a different AWS account | CCoE admin access to child accounts |
| **Service Role** | Assumed by an AWS service to act on your behalf | EC2 instance profile, Lambda execution role |
| **Service-Linked Role** | Pre-defined by AWS, auto-created, cannot be modified | GuardDuty, Config, Control Tower |
| **OrganizationAccountAccessRole** | Auto-created by AWS Organizations in new member accounts | Management account → child account break-glass access |

**Trust Policy JSON — Scoped to Specific IAM User:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_B_ID>:user/rafael"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

> **⚠️ SECURITY HARDENING — MFA on AssumeRole**
> Production cross-account roles **must** include the `aws:MultiFactorAuthPresent: true` condition in the Trust Policy. A role without this condition allows any compromised long-term credential in the trusted account to laterally move into any trusting account. This is a Tier-1 security control.

---

### IAM Identity Center — Single Sign-On

IAM Identity Center (successor to AWS Single Sign-On) is the **recommended workforce identity and access management solution** for multi-account environments. It eliminates the need for users to manually perform `AssumeRole` by providing a unified access portal backed by Permission Sets.

**Integration Flow:**
```

Corporate IdP (Microsoft AD / Okta / Azure AD) │ │ SAML 2.0 / OIDC Federation ▼ IAM Identity Center ┌─────────────────────────────────────────┐ │ User: [engineer@company.com](mailto:engineer@company.com) │ │ Permission Sets: │ │ - AdministratorAccess → Dev Accounts │ │ - ReadOnlyAccess → Prod Accounts │ └─────────────────────────────────────────┘ │ │ Provisions STS temporary credentials │ and auto-configures IAM Roles in accounts ▼ AWS Access Portal (https://<org>.awsapps.com/start) ├── Customer-A-Dev-Workload → [Console | CLI Credentials] ├── Customer-A-Production → [Console | CLI Credentials] └── Shared-Services → [Console | CLI Credentials]

```

**Permission Set Configuration Principles:**

| Permission Set Name | Accounts Assigned | Underlying IAM Policy | Consumer |
|---|---|---|---|
| `PlatformAdministratorAccess` | Shared Services | `AdministratorAccess` | CCoE Engineers |
| `DeveloperAccess` | Customer-*/Dev | Custom: EC2, S3, Lambda, SQS (no IAM write) | Application Developers |
| `SecurityAuditAccess` | All accounts | `SecurityAudit` (AWS Managed) | SecOps, Auditors |
| `ReadOnlyAccess` | Customer-*/Production | `ReadOnlyAccess` (AWS Managed) | Support tier, QA leads |
| `BillingReadOnly` | Management Account | `Billing` (AWS Managed) | FinOps team |

> **🔒 MANDATORY CONTROL — Enable MFA in IAM Identity Center**
> MFA must be enforced at the Identity Center level for all workforce users. Configure the MFA enforcement mode to **`Required`** for every user. Acceptable authenticator types are TOTP applications (e.g., Okta Verify, Microsoft Authenticator) or FIDO2/WebAuthn hardware keys. SMS-based MFA must **not** be used in production environments due to SIM-swap attack vulnerability.

---

## AWS Organizations & Service Control Policies

AWS Organizations is the **control plane for your entire AWS estate**. SCPs are IAM policy documents applied at the OU or Account boundary that define the **maximum permission ceiling** — they cannot grant permissions, only restrict them. An explicit `Deny` in an SCP overrides any `Allow` in any identity-based policy, including the account root user.

### SCP: Instance Type Enforcement

Prevents unauthorized (cost-prohibitive) EC2 instance types from being launched in development OUs. Developers may only use `t2.micro` (or an equivalent burstable family as per your cost governance policy).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyT2MicroInstanceType",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:InstanceType": "t2.micro"
        }
      }
    }
  ]
}
```

**Attachment Target:** OU `Customer-*/Dev`

---

### SCP: Root User Restriction

Prevents the account root user from performing any API actions within member accounts. This is critical because new accounts created by AWS Organizations have a recoverable root password.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserAllActions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:root"
          ]
        }
      }
    }
  ]
}
```

**Attachment Target:** Root OU (applies to ALL member accounts)

---

### SCP: CloudTrail Tamper Prevention

Prevents any principal in any member account from deleting, stopping, or modifying the Organization Trail configuration or its backing S3 objects. This preserves the integrity of the centralized audit trail.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyCloudTrailModification",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail",
        "cloudtrail:PutEventSelectors",
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "s3:PutBucketPolicy",
        "s3:DeleteBucketPolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

**Attachment Target:** Root OU (applies to ALL member accounts)

---

### SCP: IP-Based Console Access Restriction

Restricts AWS Management Console and API access to known corporate IP ranges (office egress IPs and corporate VPN endpoints). This mirrors standard on-premises network access control models.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyConsoleAccessFromNonCorpIP",
      "Effect": "Deny",
      "Action": [
        "signin:*",
        "*"
      ],
      "Resource": "*",
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": [
            "203.0.113.0/24",
            "198.51.100.128/25"
          ]
        },
        "Bool": {
          "aws:ViaAWSService": "false"
        }
      }
    }
  ]
}
```

> **⚠️ OPERATIONAL WARNING — Before Attaching IP SCPs**
> Validate your complete corporate IP address range (including all VPN gateway egress IPs and any remote-worker CIDR blocks) **before** attaching this policy. An incorrect CIDR block will lock all users out of affected accounts. Test on a single non-production account first. Ensure the Management Account is explicitly excluded or that a break-glass access path exists via AWS Support.

---

### Tag Policies: Uniform Tagging Enforcement

Tag Policies enforce a standardized tagging schema across all accounts to enable accurate cost allocation, resource governance, and automated compliance reporting.

**Example Tag Policy — Enforce `Environment` tag on EC2:**

```json
{
  "tags": {
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": [
          "Production",
          "QA",
          "Development",
          "Sandbox"
        ]
      },
      "enforced_for": {
        "@@assign": [
          "ec2:instance",
          "ec2:volume",
          "rds:db",
          "s3:bucket"
        ]
      }
    }
  }
}
```

**Mandatory Tag Keys (Enforced via SCP — deny resource creation without these tags):**

| Tag Key | Expected Values | Enforcement Mechanism | Purpose |
|---|---|---|---|
| `Environment` | `Production`, `QA`, `Development`, `Sandbox` | Tag Policy + SCP Deny | Environment classification |
| `CostCenter` | `CC-001`, `CC-002`, … (from FinOps registry) | SCP Deny on create | Financial chargeback |
| `Project` | Customer name or internal project code | SCP Deny on create | Billing attribution |
| `Owner` | `team-platform`, `team-customer-a`, … | Tag Policy | Operational ownership |
| `DataClassification` | `Public`, `Internal`, `Confidential`, `Restricted` | Tag Policy | Data governance |

---

## Centralized Logging Architecture

All log data — infrastructure, application, network, and security — flows into the **Shared Services Account** (or a dedicated Security Account for higher-compliance environments). This creates a single, tamper-resistant pane of glass for forensic analysis, threat hunting, and compliance auditing.

### AWS CloudTrail — Organization Trail

An **Organization Trail** is created once in the Management/Shared Services Account and automatically propagates to every current and future member account. Member accounts can view but **cannot modify or delete** the trail.

**Configuration Requirements:**

| Parameter | Required Value | Rationale |
|---|---|---|
| `IsOrganizationTrail` | `true` | Covers all current and future member accounts automatically |
| `IsMultiRegionTrail` | `true` | Captures API activity in all AWS Regions, not just the home region |
| `EnableLogFileValidation` | `true` | SHA-256 hash chain validation; detects log tampering |
| `IncludeGlobalServiceEvents` | `true` | Captures IAM, STS, and other global service API calls |
| `EventSelector.DataResources` | S3 object-level events | Required for detecting unauthorized data exfiltration |
| `S3BucketName` | `aws-cloudtrail-logs-<org-id>-<random>` | Centralized bucket in Shared Services / Security Account |
| `KMSKeyId` | `arn:aws:kms:…` | Encrypt logs at rest with a CMK; deny key use to member accounts |

**S3 Prefix Structure (per Organization Trail):**
```

s3://aws-cloudtrail-logs-<org-id>/ ├── AWSLogs/ │ └── o-<org-id>/ ← Organization-level prefix │ ├── <account-id-shared-svc>/ │ │ └── CloudTrail/ │ │ └── us-east-1/ │ │ └── 2024/01/15/ │ │ └── <account-id>_CloudTrail_us-east-1_*.json.gz │ ├── <account-id-customer-a-dev>/ │ │ └── CloudTrail/… │ └── <account-id-customer-a-prod>/ │ └── CloudTrail/…

```

---

### AWS Config — Configuration Compliance

AWS Config records **point-in-time configuration snapshots** of all AWS resources and evaluates them against compliance rules. Unlike CloudTrail (which records *who did what and when*), Config records *what the resource looks like and how it changed over time*.

| Use Case | CloudTrail | AWS Config |
|---|---|---|
| "Who deleted the S3 bucket?" | ✅ API caller, time, source IP | ❌ |
| "What were the security group rules on Jan 1st?" | ❌ | ✅ Historical configuration state |
| "Is this EC2 instance compliant with our AMI policy?" | ❌ | ✅ Config Rules + Remediation |
| "Did any IAM policy change grant `*:*` permissions?" | ✅ (what changed) | ✅ (was the resource ever non-compliant?) |

**Recommended Managed Config Rules (Priority Set):**

- `restricted-ssh` — Detects security groups with unrestricted SSH (0.0.0.0/0) ingress
- `s3-bucket-public-read-prohibited` — Detects public S3 buckets
- `iam-root-access-key-check` — Alerts if root account has active access keys
- `mfa-enabled-for-iam-console-access` — Validates MFA enforcement for console IAM users
- `cloudtrail-enabled` — Validates CloudTrail is active and logging
- `ec2-instances-in-vpc` — Ensures no EC2 instances are launched outside a VPC

---

### VPC Flow Logs

VPC Flow Logs capture L3/L4 metadata for all network interfaces in a VPC. They are **non-intrusive** (collected out-of-band, no latency impact) and are essential for network forensics, security group validation, and unexpected egress detection.

**Recommended Destination:** Amazon S3 (cost-optimized for long-term retention with S3 Intelligent-Tiering) and/or CloudWatch Logs (for near-real-time alerting via Metric Filters).

---

### Amazon GuardDuty

GuardDuty performs near-continuous **ML-driven threat detection** by analyzing CloudTrail management events, CloudTrail data events (S3), DNS logs, VPC Flow Logs, EKS audit logs, and EBS volume data. It requires no agents and no infrastructure.

> **🔒 DEPLOYMENT MANDATE — GuardDuty Multi-Account**
> GuardDuty must be enabled in **every AWS Region** across **every member account**, with a designated **Delegated Administrator** account (typically the Security Account). Findings from all member accounts aggregate into the Delegated Administrator account for centralized investigation and SIEM ingestion. Disabling GuardDuty in any member account must be blocked via an SCP.

---

## Automated Account Provisioning Pipeline

### AWS Control Tower — Account Vending Machine

AWS Control Tower is the **orchestration layer** that sits above AWS Organizations. It automates the creation of new accounts with pre-configured, compliant landing zones — eliminating manual, error-prone account bootstrap procedures and preventing **configuration drift** across the estate.

**Control Tower Provisions (Automatically, On New Account Creation):**

- Creates the account within the correct OU in AWS Organizations
- Applies all inherited SCPs from the OU hierarchy
- Deploys baseline CloudFormation StackSets (Config, CloudTrail enrollment, VPC baseline)
- Configures the account as a GuardDuty member under the Delegated Admin
- Enrolls the account into AWS Security Hub (if configured)
- Makes the account immediately available for IAM Identity Center assignment

**Relationship to Other Services:**
```

New Account Request (via Account Factory) │ ▼ AWS Control Tower ┌──────────────────────────────────────────┐ │ Orchestrates: │ │ - AWS Organizations (account create) │ │ - CloudFormation StackSets (baseline) │ │ - IAM Identity Center (enrollment) │ │ - Service Catalog (portfolio deploy) │ │ - GuardDuty member enrollment │ └──────────────────────────────────────────┘ │ ▼ Fully Configured, Compliant AWS Account (Ready for workload deployment in < 30 minutes)

```

---

### AWS Service Catalog — Infrastructure Vending Machine

Service Catalog provides a **self-service infrastructure marketplace** of CCoE-approved CloudFormation-backed products organized into Portfolios. Developers consume pre-validated infrastructure building blocks without requiring direct CloudFormation or IAM expertise.

**Example Portfolio: `Developer Environments`**

| Product Name | Description | Resource Stack | Constraints |
|---|---|---|---|
| `Dev-IDE-Cloud9-T2Micro` | Cloud IDE with Git integration | AWS Cloud9, EC2 t2.micro, IAM Instance Profile | Instance type locked; auto-stop after 4h idle |
| `Serverless-API-Scaffold` | API Gateway + Lambda + SQS baseline | API GW, 2x Lambda, SQS Queue, CloudWatch Dashboard | Lambda memory cap: 512MB; no VPC bypass |
| `RDS-Dev-MySQL-Single-AZ` | Non-HA MySQL instance for development | RDS MySQL t3.micro, Security Group, Parameter Group | No Multi-AZ; deletion protection disabled; auto-stop nightly |
| `S3-Private-App-Bucket` | Hardened private S3 bucket | S3 Bucket, Bucket Policy (deny public ACL), SSE-S3 | Public access block enforced; versioning enabled |

> **⚠️ GOVERNANCE NOTE — Service Catalog Launch Constraints**
> Every Service Catalog product must use a **Launch Constraint IAM Role** to provision resources. This means the end-user's own IAM permissions are **not** used to provision the stack — a dedicated, least-privilege provisioning role is assumed instead. This prevents privilege escalation through Service Catalog.

---

### AWS CloudFormation — IaC Engine

All infrastructure provisioned within this Landing Zone must be defined as **CloudFormation templates**. Direct console-based resource creation is prohibited in QA and Production OUs via SCP.

**Template Authoring Standards:**

- All templates must include a `Metadata` block with owner, cost center, and data classification tags
- All resources must propagate `aws:cloudformation:stack-name` and mandatory business tags via `PropagateAtLaunch` or resource-level `Tags`
- Stack drift detection must be scheduled (weekly minimum) via EventBridge + Lambda automation
- Templates must be stored in a version-controlled repository with peer review requirements; direct S3 uploads to Service Catalog without Git history are prohibited

---

## Billing & Cost Governance

| Control | Implementation | Owner |
|---|---|---|
| **Billing Separation** | Each AWS account is a native billing boundary | Automatic via multi-account structure |
| **Consolidated Billing** | AWS Organizations consolidates all accounts under one invoice | Management Account |
| **Per-Account Billing Alarms** | CloudWatch Billing Metric + SNS Topic per child account, monitored from Shared Services | FinOps + CCoE |
| **Cost Allocation Tags** | `CostCenter`, `Project`, `Environment` activated in Management Account | FinOps |
| **AWS Cost Anomaly Detection** | Enabled per account and at organization level; SNS alert on >15% anomaly | FinOps |
| **Service Quota Isolation** | Each account receives its own service quota allocations; no quota contention between customers | Automatic via multi-account structure |
| **SCP: Restrict Expensive Instance Types** | Block `p4d`, `x2`, `u-` class instances outside approved production accounts | Applied at Dev/QA OUs |

> **💰 COST GOVERNANCE — Developer Billing Alarms**
> A CloudWatch billing alarm must be configured for every developer account with a threshold appropriate to the team's approved monthly budget. The alarm SNS topic must notify both the developer and their team lead. This provides real-time feedback on runaway resources (e.g., forgotten large instances) before end-of-month billing surprises.

---

## Security Hardening Checklist

Use this checklist as a pre-production readiness gate before any customer environment goes live.

**Identity & Access**
- [ ] Root user access keys deleted in all accounts (verified via Config Rule `iam-root-access-key-check`)
- [ ] Root user MFA enabled in Management Account (hardware key required)
- [ ] IAM Identity Center MFA enforcement set to `Required` for all users
- [ ] No long-term IAM User access keys exist in any member account (all access via Identity Center SSO)
- [ ] `OrganizationAccountAccessRole` access to child accounts restricted to break-glass use only; audited quarterly
- [ ] Cross-account role Trust Policies require `aws:MultiFactorAuthPresent: true`

**SCP Enforcement**
- [ ] Root User restriction SCP attached to Root OU
- [ ] CloudTrail tamper-prevention SCP attached to Root OU
- [ ] Dev OU instance type restriction SCP verified (attempt to launch non-`t2.micro` returns `Unauthorized`)
- [ ] OU movement restriction SCP applied; only CCoE principals in `arn:aws:iam::*:role/OrgAdminRole` can call `organizations:MoveAccount`
- [ ] Resource creation without mandatory tags is denied by SCP in QA and Production OUs

**Logging & Detection**
- [ ] Organization Trail enabled, multi-region, with log file validation; KMS CMK encryption verified
- [ ] CloudTrail S3 bucket has Object Lock (Governance mode, minimum 365-day retention) enabled
- [ ] AWS Config enabled in all regions across all accounts; Config Aggregator configured in Security Account
- [ ] GuardDuty enabled in all regions across all accounts; findings aggregating to Security Account
- [ ] VPC Flow Logs enabled for all VPCs; destination S3 bucket has lifecycle policy (90-day transition to Glacier)

**Network**
- [ ] No default VPC exists in any member account (`ec2:DeleteVpc` logged; default VPC removal scripted via Control Tower customization)
- [ ] All VPCs use non-overlapping RFC 1918 CIDR blocks (consult the IP Address Management registry before VPC creation)
- [ ] Security Groups deny all inbound by default; no `0.0.0.0/0` ingress on port 22 or 3389 in production accounts

**Provisioning & Drift**
- [ ] All production resources deployed via CloudFormation; no manually created resources (validated via Config drift detection)
- [ ] Service Catalog products use Launch Constraint IAM Roles (not end-user permissions)
- [ ] Control Tower Account Factory Customizations (AFCAs) tested on a fresh account post-provisioning

---

## Operational Runbook Checklist

**Onboarding a New Customer (New OU + Accounts):**
- [ ] Request submitted via internal ticketing system with customer name, project code, and CostCenter
- [ ] CCoE creates Customer OU under Root via AWS Organizations (naming convention: `customer-<client-id>`)
- [ ] Dev, QA, and Production child OUs created within the Customer OU
- [ ] Relevant SCPs attached to each child OU (Dev: permissive; Prod: restrictive)
- [ ] Accounts provisioned via Control Tower Account Factory (not manual Organizations account creation)
- [ ] IAM Identity Center Permission Sets assigned to the new accounts for relevant teams
- [ ] CloudTrail Organization Trail automatically covers new accounts — verify S3 prefix appears within 15 minutes
- [ ] GuardDuty member invitation sent to and accepted by new account (verify in Security Account)
- [ ] Customer tag (`Project: customer-<client-id>`) propagated to all accounts; verified in Cost Explorer

**Onboarding a New Developer (Maria Scenario):**
- [ ] Identity created in IAM Identity Center (or synced from corporate IdP)
- [ ] User assigned to relevant Group (e.g., `group-customer-a-developers`)
- [ ] Group linked to `DeveloperAccess` Permission Set on `customer-a-dev` account
- [ ] Developer launches `Dev-IDE-Cloud9-T2Micro` product from Service Catalog self-service portal
- [ ] Cloud9 environment provisioned within developer account; logs stream to CloudWatch Logs in Shared Services

---

## Migration Roadmap

The transition from single-account to multi-account is a **phased, multi-sprint program**, not a single-weekend migration. Below is the high-level execution sequence.

| Phase | Duration | Key Activities | Exit Criteria |
|---|---|---|---|
| **Phase 0: Foundation** | 2–4 weeks | Enable Organizations, deploy Control Tower Landing Zone, configure IAM Identity Center, establish Organization CloudTrail | All foundational services operational; CCoE can SSO into Management Account |
| **Phase 1: Workload Classification** | 2 weeks | Inventory all workloads in the monolithic account; assign customer/project ownership; define target OU structure | Tagged inventory complete; OU design approved by CCoE |
| **Phase 2: IaC Authoring** | 4–8 weeks | Translate existing infrastructure into CloudFormation templates; build Service Catalog product library; define all SCPs | All target-state infrastructure expressible as versioned IaC |
| **Phase 3: Dev Environment Migration** | 2–4 weeks | Provision new customer Dev accounts via Control Tower; migrate dev workloads; validate SCP enforcement | Developers working from new accounts; zero resource usage in legacy account |
| **Phase 4: QA & Production Migration** | 4–8 weeks | Migrate QA environments; conduct regression + load testing; migrate production workloads with AWS DMS for database layer | Production traffic fully served from new accounts; legacy account in read-only/drain state |
| **Phase 5: Legacy Account Decommission** | 1–2 weeks | Audit legacy account for missed resources; export final CloudTrail logs; close account via Organizations | Legacy account closed; AWS Support notified |

> **⚠️ DATABASE MIGRATION NOTE**
> For migrating stateful database workloads (Phase 4), use **AWS Database Migration Service (DMS)** with Change Data Capture (CDC) enabled to perform near-zero-downtime migrations. DMS maintains a live replication stream from the source database to the target until the cutover window. Plan cutover windows during off-peak hours and communicate to customers in advance. Do not attempt database migrations without a validated rollback procedure.

---

## Further Reading

| Resource | Type | Description |
|---|---|---|
| [AWS Organizations Best Practices for SCPs](https://aws.amazon.com/blogs/industries/best-practices-for-aws-organizations-service-control-policies-in-a-multi-account-environment/) | Blog | Deep-dive on SCP strategy for complex OU hierarchies |
| [Organizing Your AWS Environment Using Multiple Accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html) | AWS Whitepaper | Comprehensive multi-account organization guidance |
| [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/) | Framework | Five pillars: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization |
| [AWS Control Tower — What is it?](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html) | Official Docs | Control Tower architecture, account factory, guardrails |
| [IAM Identity Center — How it works](https://docs.aws.amazon.com/singlesignon/latest/userguide/how-it-works.html) | Official Docs | SSO flows, Permission Sets, SCIM provisioning |
| [Building a Secure Landing Zone](https://aws.amazon.com/blogs/security/building-a-secure-landing-zone/) | Security Blog | Security-focused landing zone implementation patterns |
| [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html) | Prescriptive Guidance | Reference architecture for security-focused multi-account topology |

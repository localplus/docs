# 🥚🐔 **Bootstrap Guide**
## *LOCAL-PLUS Platform Initialization*

---

> **Retour vers** : [Architecture Overview](../EntrepriseArchitecture.md)

---

# 📋 **Table of Contents**

1. [Architecture Overview](#architecture-overview)
2. [Layer 0 — Control Tower](#layer-0--control-tower)
3. [Layer 1 — Terraform Foundation](#layer-1--terraform-foundation)
4. [Account Factory](#account-factory)
5. [Platform Provisioning](#platform-provisioning)
6. [Bootstrap Repository Structure](#bootstrap-repository-structure)

---

# 🏗️ **Architecture Overview**

> **Voir [ADR-001](../adr/ADR-001-LANDING-ZONE-APPROACH.md) pour le rationale complet.**

## Principe

| Layer | Géré par | Responsabilité |
|-------|----------|----------------|
| **Layer 0** | Control Tower | Org, SCPs, Logging, Audit |
| **Layer 1** | Terraform | SSO, Custom Controls, Account Factory |
| **Layer 2+** | Terraform | VPC, EKS, RDS, Platform |

> *"Control Tower comme fondation, Terraform comme langage."*

## Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 0 — CONTROL TOWER                          │
│                    (Managed, Immutable, Audit)                       │
├─────────────────────────────────────────────────────────────────────┤
│  • AWS Organizations        • CloudTrail org-level                  │
│  • OUs (Security, Infra,    • AWS Config                            │
│    Workloads, Suspended)    • Security Hub                          │
│  • Guardrails (400+)        • Log Archive + Audit accounts          │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 1 — TERRAFORM                               │
│                    (GitOps, GitHub Actions)                          │
├─────────────────────────────────────────────────────────────────────┤
│  • SSO (groups, permission sets)                                     │
│  • Custom Controls (aws_controltower_control)                       │
│  • Account Factory (baseline: OIDC, KMS, S3 state)                  │
│  • Shared Services Account                                           │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 2+ — PLATFORM                               │
│                    (platform-application-provisioning/)              │
├─────────────────────────────────────────────────────────────────────┤
│  • VPC, Subnets, Transit Gateway                                     │
│  • EKS Clusters                                                      │
│  • RDS, Kafka, Cache (Aiven)                                        │
│  • Observability stack                                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

# 🔧 **Layer 0 — Control Tower**

> **Setup via Console AWS (one-time)**

## Prerequisites

| Requirement | Status |
|-------------|--------|
| AWS Account (Management) | Required |
| Email domain for accounts | Required |
| Region: eu-west-1 | Required (GDPR) |

## Steps

| Step | Action | Duration |
|------|--------|----------|
| 1 | Console → Control Tower → Set up landing zone | 45 min |
| 2 | Home region: eu-west-1 | Included |
| 3 | Additional regions: eu-central-1 (DR) | Included |
| 4 | Log Archive account created | Automatic |
| 5 | Audit account created | Automatic |
| 6 | IAM Identity Center enabled | Included |

## What Control Tower creates

> **Voir [ADR-001 Resource Inventory](../adr/ADR-001-LANDING-ZONE-APPROACH.md#control-tower-resource-inventory) pour la liste complète.**

| Account | Resources créées |
|---------|------------------|
| **Management** | Organizations, Service Roles, CloudTrail, Config |
| **Log Archive** | S3 Buckets (logs), KMS Key, Lifecycle Policies |
| **Audit** | Config Aggregator, Security Hub, GuardDuty, IAM Access Analyzer |
| **Workload (baseline)** | CloudTrail local, Config local, CT-managed roles |

---

# 🏗️ **Layer 1 — Terraform Foundation**

> **Repo: `bootstrap/`** — GitHub Actions CI/CD

## What we manage in Terraform

| Component | Module | Description |
|-----------|--------|-------------|
| **SSO** | `sso/` | Groups, Permission Sets |
| **Custom Controls** | `control-tower/` | Additional guardrails via Terraform |
| **Account Factory** | `account-factory/` | AFT module via GitHub Actions |
| **Shared Services** | `core-accounts/` | ECR, Transit Gateway |

## SSO Groups

| Group | Permission Set | Access |
|-------|----------------|--------|
| PlatformAdmins | AdministratorAccess | Full access |
| Developers | PowerUserAccess | No IAM changes |
| ReadOnly | ViewOnlyAccess | Read only |
| SecurityAuditors | SecurityAudit | Security review |
| OnCall | IncidentResponder | Break-glass |

## Custom Controls

| Control | Purpose | Target OU |
|---------|---------|-----------|
| Require IMDSv2 | EC2 metadata security | Workloads |
| Deny Public S3 | Data protection | All |
| Enforce EU Regions | GDPR compliance | All |
| Require Encryption | Data at rest | Workloads |

---

# 🏭 **Account Factory**

> Self-service account provisioning via PR

## Workflow

| Step | Action | Actor |
|------|--------|-------|
| 1 | Run `task account:create` | Developer |
| 2 | Fill account request YAML | Developer |
| 3 | Create PR | Developer |
| 4 | Review request | Platform Team |
| 5 | Merge PR | Platform Team |
| 6 | GitHub Actions applies Terraform | Automated |
| 7 | Account ready with baseline | Automated |

## What's created per account

| Resource | Purpose |
|----------|---------|
| AWS Account | In appropriate OU |
| S3 Bucket | Terraform state |
| GitHub OIDC | CI/CD authentication |
| KMS Keys | Encryption (terraform, secrets, eks) |
| Security Baseline | EBS encryption, S3 block public |

## Request fields

| Field | Description | Example |
|-------|-------------|---------|
| account_name | Unique identifier | localplus-backend-dev |
| environment | dev / staging / prod | dev |
| owner_email | Team contact | backend@localplus.io |
| team | Owning team | backend |
| purpose | Business justification | Backend services development |

---

# 📦 **Platform Provisioning**

> **Repo: `platform-application-provisioning/`**

## Order of operations

| Order | Resource | Dependencies |
|-------|----------|--------------|
| 1 | VPC + Subnets | Account created |
| 2 | KMS Keys | Account created |
| 3 | EKS Cluster | VPC, KMS |
| 4 | IRSA | EKS |
| 5 | VPC Peering (Aiven) | VPC, Aiven project |
| 6 | ArgoCD | EKS |

## Providers

| Provider | Resources | Frequency |
|----------|-----------|-----------|
| AWS | VPC, EKS, KMS | 1x per environment |
| Aiven | PostgreSQL, Kafka, Valkey | 1x per environment |
| Cloudflare | DNS, WAF, Tunnel | 1x per zone |

---

# 📋 **Bootstrap Repository Structure**

| Directory | Purpose |
|-----------|---------|
| `.github/workflows/` | CI/CD pipelines (plan, apply, account-request) |
| `control-tower/` | Data sources + custom controls |
| `sso/` | Groups, Permission Sets |
| `account-factory/` | Account creation + baseline |
| `tests/checkov-policies/` | Custom compliance policies |
| `tests/compliance-bdd/` | BDD audit tests |
| `docs/` | Runbook |

---

# 🔒 **Policy as Code**

| Tool | Purpose | Format |
|------|---------|--------|
| **Trivy** | Security scanning (IaC + secrets) | Built-in |
| **Checkov** | 2000+ compliance policies | YAML |
| **terraform-compliance** | Audit-readable policies | BDD/Gherkin |

---

# 🔗 **Related Documentation**

| Topic | Link |
|-------|------|
| **ADR Landing Zone** | [ADR-001](../adr/ADR-001-LANDING-ZONE-APPROACH.md) |
| **CI/CD & Delivery** | [Platform Engineering](../platform/PLATFORM-ENGINEERING.md) |
| **Security Setup** | [Security Architecture](../security/SECURITY-ARCHITECTURE.md) |
| **Networking** | [Networking Architecture](../networking/NETWORKING-ARCHITECTURE.md) |

---

*Document maintenu par : Platform Team*  
*Dernière mise à jour : Janvier 2026*

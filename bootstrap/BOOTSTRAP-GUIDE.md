# 🥚🐔 **Bootstrap Guide**
## *LOCAL-PLUS Platform Initialization*

---

> **Retour vers** : [Architecture Overview](../EntrepriseArchitecture.md)

---

# 📋 **Table of Contents**

1. [Layer 0 — Manual Bootstrap](#layer-0--manual-bootstrap)
2. [Account Factory — Self-Service](#account-factory--self-service)
3. [Platform Application Provisioning](#platform-application-provisioning)
4. [Workload Provisioning](#workload-provisioning)
5. [Layer 2 — Platform Bootstrap](#layer-2--platform-bootstrap)
6. [Layer 3+ — Application Services](#layer-3--application-services)
7. [Bootstrap Repository Structure](#bootstrap-repository-structure)

---

# 🔧 **Layer 0 — Manual Bootstrap (1x per AWS Organization)**

> **Principe :** Point d'entrée unique pour chaque cloud provider.  
> Ces étapes sont manuelles car elles créent les fondations pour toute l'automatisation future.

## Étapes

| Étape | Action | Outil | Durée |
|-------|--------|-------|-------|
| 1 | Créer compte Management | Console AWS | 10 min |
| 2 | Activer AWS Organizations | Console | 5 min |
| 3 | Activer Control Tower | Console | 45 min |
| 4 | Configurer IAM Identity Center (SSO) | Console | 30 min |
| 5 | Créer OUs (Security, Infrastructure, Workloads) | Control Tower | 15 min |
| 6 | Appliquer SCPs | Console Organizations | 15 min |
| 7 | Créer Core Accounts | Control Tower | 15 min/compte |

## AWS Multi-Account Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AWS CONTROL TOWER (Organization)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  MANAGEMENT     │  │  SECURITY       │  │  LOG ARCHIVE    │             │
│  │  ACCOUNT        │  │  ACCOUNT        │  │  ACCOUNT        │             │
│  │  • Control Tower│  │  • GuardDuty    │  │  • CloudTrail   │             │
│  │  • Organizations│  │  • Security Hub │  │  • Config Logs  │             │
│  │  • SCPs         │  │  • IAM Identity │  │  • VPC Flow Logs│             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    WORKLOAD ACCOUNTS (OU: Workloads)                │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│  │  │ DEV Account │  │ STAGING     │  │ PROD Account│                  │   │
│  │  │ VPC + EKS   │  │ VPC + EKS   │  │ VPC + EKS   │                  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    SHARED SERVICES ACCOUNT (OU: Infrastructure)     │   │
│  │  • Transit Gateway Hub     • Container Registry (ECR)              │   │
│  │  • VPC Endpoints           • Artifact Storage (S3)                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# 🏭 **Account Factory — Self-Service**

> **Principe :** Les équipes demandent un AWS account via PR dans `bootstrap/account-factory/requests/`

## Ce qui est créé automatiquement

| Ressource | Description |
|-----------|-------------|
| **AWS Account** | Dans l'OU appropriée (Workloads/Dev, Staging, Prod) |
| **S3 Bucket** | Pour Terraform state |
| **GitHub OIDC** | Pour CI/CD sans credentials statiques |
| **Baseline IAM Roles** | Admin, Developer, ReadOnly |

## Workflow

1. **Équipe** crée un fichier YAML dans `bootstrap/account-factory/requests/`
2. **PR Review** par Platform Team
3. **Merge** déclenche Terraform via CI/CD
4. **Account créé** avec baseline automatique

---

# 📦 **Platform Application Provisioning**

> **Repo :** `platform-application-provisioning`  
> Contient les modules Terraform pour provisionner les services applicatifs.

## Providers

| Provider | Ce qui est provisionné | Fréquence |
|----------|------------------------|-----------|
| **Cloudflare** | Zone DNS, WAF, Tunnel | 1x par zone |
| **Aiven** | Projet, VPC peering | 1x par environment |
| **AWS** | VPC, EKS, KMS | 1x par environment |

## Modules disponibles

| Module | Description |
|--------|-------------|
| `database/` | Aiven PostgreSQL |
| `kafka/` | Aiven Kafka |
| `cache/` | Aiven Valkey |
| `vpc/` | AWS VPC |
| `eks/` | AWS EKS Cluster |
| `eks-namespace/` | Namespace + RBAC + NetworkPolicy |

---

# 🖥️ **Workload Provisioning**

> Ordre de provisionnement pour un nouvel environnement.

| Ordre | Ressource | Dépendances |
|-------|-----------|-------------|
| 1 | VPC + Subnets | Account créé |
| 2 | KMS Keys | Account créé |
| 3 | EKS Cluster | VPC, KMS |
| 4 | IRSA | EKS |
| 5 | VPC Peering (Aiven) | VPC, Aiven projet |
| 6 | Outputs → Platform repos | Tous |

---

# 🚀 **Layer 2 — Platform Bootstrap**

> Installation des composants platform sur le cluster EKS.

| Ordre | Action | Dépendance |
|-------|--------|------------|
| 1 | Install ArgoCD via Helm | EKS ready |
| 2 | Apply App-of-Apps ApplicationSet | ArgoCD running |
| 3 | ArgoCD syncs `platform-*` repos | Reconciliation auto |

**ArgoCD : Instance centralisée unique** gérant tous les environnements.

---

# 📱 **Layer 3+ — Application Services**

> ArgoCD ApplicationSets découvrent automatiquement les services.

## Fonctionnement

1. **Git Generator** scanne les répertoires de services
2. **Matrix Generator** croise avec les clusters (dev/staging/prod)
3. **Applications créées** automatiquement pour chaque combinaison
4. **Sync** selon la politique (auto pour dev, manual pour prod)

## Flux de déploiement

```
Git push → ArgoCD détecte → Sync (dev: auto, prod: manual) → Deployed
```

→ **CI/CD détaillé** : voir [Platform Engineering](../platform/PLATFORM-ENGINEERING.md)

---

# 📋 **Bootstrap Repository Structure**

```
bootstrap/
├── .mise.toml                    # Tool versions
├── Taskfile.yaml                 # Task orchestration
│
├── aws-landing-zone/
│   ├── organization/             # OUs definition
│   ├── control-tower/            # Control Tower setup
│   ├── sso/                      # SSO groups, permission sets
│   ├── scps/                     # Service Control Policies
│   └── core-accounts/            # Core accounts config
│
├── account-factory/
│   ├── main.tf                   # Account creation
│   ├── templates/                # Baseline resources
│   └── requests/                 # Account requests (PR)
│
├── tests/
│   ├── unit/                     # terraform test
│   ├── compliance/               # OPA/Conftest
│   └── security/                 # Trivy
│
└── docs/
    ├── RUNBOOK-BOOTSTRAP.md
    └── ACCOUNT-FACTORY.md
```

---

# 🔗 **Related Documentation**

| Topic | Link |
|-------|------|
| **CI/CD & Delivery** | [Platform Engineering](../platform/PLATFORM-ENGINEERING.md) |
| **Security Setup** | [Security Architecture](../security/SECURITY-ARCHITECTURE.md) |
| **Networking** | [Networking Architecture](../networking/NETWORKING-ARCHITECTURE.md) |

---

*Document maintenu par : Platform Team*  
*Dernière mise à jour : Janvier 2026*

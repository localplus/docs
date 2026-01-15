# 🏗️ **LOCAL-PLUS — Architecture Définitive**
## *Gift Card & Loyalty Platform*
### *Version 1.0 — Janvier 2026*

---

# 📋 **PARTIE I — CONTEXTE & CONTRAINTES**

## **1.1 Paramètres Business**

| Paramètre | Valeur | Impact architectural |
|-----------|--------|---------------------|
| **RPO** | 1 heure | Backups horaires minimum, réplication async acceptable |
| **RTO** | 15 minutes | Failover automatisé, pas de procédure manuelle |
| **TPS** | 500 transactions/sec | Pas de sharding nécessaire, single Postgres suffit |
| **RPS** | 1500 requêtes/sec | Load balancer + HPA standard |
| **Durée de vie** | 5+ ans | Design pour évolutivité, pas de shortcuts |
| **Équipe on-call** | 5 personnes | Runbooks exhaustifs, alerting structuré |

## **1.2 Contraintes Compliance**

| Standard | Exigences clés | Impact |
|----------|---------------|--------|
| **GDPR** | Droit à l'oubli, consentement, data residency EU | Logs anonymisés, data retention policies, EU region |
| **PCI-DSS** | Pas de stockage PAN, encryption at rest/transit, audit logs | mTLS, Vault pour secrets, audit trail immutable |
| **SOC2** | Contrôle d'accès, monitoring, incident response | RBAC strict, observabilité complète, runbooks documentés |

## **1.3 Contraintes Techniques**

| Contrainte | Choix | Rationale |
|------------|-------|-----------|
| **Cloud primaire** | AWS | Décision business |
| **Région initiale** | eu-west-1 (Ireland) | GDPR, latence Europe |
| **Multi-région** | Prévu, pas immédiat | Design pour, implémente plus tard |
| **Database** | Aiven PostgreSQL | Managed, multicloud-ready, PCI compliant |
| **Messaging** | Aiven Kafka | Managed, multicloud-ready |
| **Observabilité** | Self-hosted, coût minimal | Prometheus/Loki/Tempo + CloudWatch Logs (tier gratuit) |

---

# 🏛️ **PARTIE II — ARCHITECTURE LOGIQUE**

## **2.1 Vue d'ensemble**

### **2.1.1 AWS Multi-Account Strategy (Control Tower)**

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
│  │                 │  │    Center       │  │                 │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    WORKLOAD ACCOUNTS (OU: Workloads)                │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│  │  │ DEV Account │  │ STAGING     │  │ PROD Account│                  │   │
│  │  │             │  │ Account     │  │             │                  │   │
│  │  │ VPC + EKS   │  │ VPC + EKS   │  │ VPC + EKS   │                  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    SHARED SERVICES ACCOUNT (OU: Infrastructure)     │   │
│  │  • Transit Gateway Hub                                              │   │
│  │  • Centralized VPC Endpoints                                        │   │
│  │  • Container Registry (ECR)                                         │   │
│  │  • Artifact Storage (S3)                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### **2.1.2 Architecture EKS par Environnement**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WORKLOAD ACCOUNT (PROD) — eu-west-1                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                        VPC — 10.0.0.0/16                              │ │
│  │                                                                       │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │  │                    EKS CLUSTER                                  │ │ │
│  │  │                                                                 │ │ │
│  │  │   ┌─────────────────────────────────────────────────────────┐  │ │ │
│  │  │   │ NODE POOL: platform (taints: platform=true:NoSchedule)  │  │ │ │
│  │  │   │ Instance: m6i.xlarge (dedicated resources)              │  │ │ │
│  │  │   ├─────────────────────────────────────────────────────────┤  │ │ │
│  │  │   │ PLATFORM NAMESPACE                                      │  │ │ │
│  │  │   │ • ArgoCD (centralisé)                                   │  │ │ │
│  │  │   │ • Cilium (CNI + Gateway API)                            │  │ │ │
│  │  │   │ • Vault Agent Injector                                  │  │ │ │
│  │  │   │ • External-Secrets Operator                             │  │ │ │
│  │  │   │ • Kyverno                                               │  │ │ │
│  │  │   │ • OTel Collector                                        │  │ │ │
│  │  │   │ • Prometheus + Loki + Tempo + Grafana                   │  │ │ │
│  │  │   └─────────────────────────────────────────────────────────┘  │ │ │
│  │  │                                                                 │ │ │
│  │  │   ┌─────────────────────────────────────────────────────────┐  │ │ │
│  │  │   │ NODE POOL: application (default, auto-scaling)          │  │ │ │
│  │  │   │ Instance: m6i.large (cost-optimized)                    │  │ │ │
│  │  │   ├─────────────────────────────────────────────────────────┤  │ │ │
│  │  │   │ APPLICATION NAMESPACES                                  │  │ │ │
│  │  │   │ • svc-ledger                                            │  │ │ │
│  │  │   │ • svc-wallet                                            │  │ │ │
│  │  │   │ • svc-merchant                                          │  │ │ │
│  │  │   │ • svc-giftcard                                          │  │ │ │
│  │  │   │ • svc-notification                                      │  │ │ │
│  │  │   └─────────────────────────────────────────────────────────┘  │ │ │
│  │  │                                                                 │ │ │
│  │  └─────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                       │ │
│  │                           │ VPC Peering / Transit Gateway             │ │
│  │                           ▼                                           │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │  │                    AIVEN VPC                                    │ │ │
│  │  │  • PostgreSQL (Primary + Read Replica)                         │ │ │
│  │  │  • Kafka Cluster                                               │ │ │
│  │  └─────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    EXTERNAL SERVICES                                  │ │
│  │  • AWS S3 (Terraform state, backups, artifacts)                      │ │
│  │  • AWS KMS (Encryption keys)                                         │ │
│  │  • AWS Secrets Manager (bootstrap secrets only)                      │ │
│  │  • HashiCorp Vault (self-hosted on EKS — runtime secrets)            │ │
│  │  • AWS CloudWatch Logs (tier gratuit, fallback)                      │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### **2.1.3 Node Pool Strategy**

|| Node Pool | Taints | Usage | Instance Type | Scaling |
||-----------|--------|-------|---------------|---------|
|| **platform** | `platform=true:NoSchedule` | ArgoCD, Monitoring, Security tools | m6i.xlarge | Fixed (2-3 nodes) |
|| **application** | None (default) | Domain services | m6i.large | HPA (2-10 nodes) |
|| **spot** (optionnel) | `spot=true:PreferNoSchedule` | Batch jobs, non-critical | m6i.large (spot) | Auto (0-5 nodes) |

## **2.2 Domain Services**

| Service | Responsabilité | Pattern | Criticité |
|---------|---------------|---------|-----------|
| **svc-ledger** | Earn/Burn transactions, ACID ledger | Sync REST + gRPC | P0 — Core |
| **svc-wallet** | Balance queries, snapshots | Sync REST + gRPC | P0 — Core |
| **svc-merchant** | Onboarding, configuration | Sync REST | P1 |
| **svc-giftcard** | Catalog, rewards | Sync REST | P1 |
| **svc-notification** | SMS/Email dispatch | Async (Kafka consumer) | P2 |

## **2.3 Data Flow**

```
┌─────────────┐     gRPC      ┌─────────────┐
│ svc-ledger  │◄─────────────►│ svc-wallet  │
└──────┬──────┘               └──────┬──────┘
       │                             │
       │ Outbox                      │ Read
       ▼                             ▼
┌─────────────┐              ┌─────────────┐
│   Kafka     │              │ PostgreSQL  │
│  (Aiven)    │              │  (Aiven)    │
└──────┬──────┘              └─────────────┘
       │
       │ Consume
       ▼
┌─────────────────────┐
│  svc-notification   │
│  svc-analytics      │
└─────────────────────┘
```

---

# 🌿 **PARTIE II.B — GIT STRATEGY**

## **Trunk-Based Development avec Cherry-Pick**

```
                    main (trunk)
                        │
        ┌───────────────┼───────────────┐
        │               │               │
    feature/A       feature/B       feature/C
        │               │               │
        └───────────────┼───────────────┘
                        │
                    merge to main
                        │
            ┌───────────┴───────────┐
            │                       │
            ▼                       ▼
    maintenance/v1.x.x      maintenance/v2.x.x
    (cherry-pick avec       (cherry-pick avec
     label: backport-v1)     label: backport-v2)
```

### **Règles Git**

|| Branche | Usage | Politique |
||---------|-------|-----------|
|| `main` | Trunk principal | Tous les PRs mergent ici |
|| `maintenance/v1.x.x` | Maintenance version 1 | Cherry-pick depuis main uniquement |
|| `maintenance/v2.x.x` | Maintenance version 2 | Cherry-pick depuis main uniquement |
|| `feature/*` | Développement | Short-lived, merge to main |

### **Workflow Cherry-Pick**

1. **Développeur** crée un PR vers `main`
2. **Développeur** ajoute le label `backport-v1` si le fix doit aller dans v1.x.x
3. **CI** (après merge dans main) détecte le label et crée automatiquement un PR cherry-pick vers `maintenance/v1.x.x`
4. **Reviewer** valide le cherry-pick PR

> **Principe :** Tout passe par `main` d'abord. Les branches de maintenance reçoivent uniquement des cherry-picks validés.

---

# 🗂️ **PARTIE III — ORGANISATION DES REPOSITORIES**

## **3.1 Structure Complète**

```
github.com/localplus/

══════════════════════════════════════════════════════════════════════════════
TIER 0 — FOUNDATION (Platform Team ownership)
══════════════════════════════════════════════════════════════════════════════

bootstrap/
├── layer-0/
│   └── aws/
│       └── README.md                    # Runbook: Create bootstrap IAM role
├── layer-1/
│   ├── foundation/
│   │   ├── main.tf
│   │   ├── networking.tf                # VPC, Subnets, NAT, VPC Peering Aiven
│   │   ├── eks.tf                       # EKS cluster
│   │   ├── iam.tf                       # IRSA, Workload Identity
│   │   ├── kms.tf                       # Encryption keys
│   │   └── outputs.tf
│   ├── tests/
│   │   ├── unit/                        # Terraform unit tests (terraform test)
│   │   ├── compliance/                  # Checkov, tfsec, Regula
│   │   └── integration/                 # Terratest
│   └── backend.tf                       # S3 native locking
└── docs/
    └── RUNBOOK-BOOTSTRAP.md

══════════════════════════════════════════════════════════════════════════════
TIER 1 — PLATFORM (Platform Team ownership)
══════════════════════════════════════════════════════════════════════════════

platform-gitops/
├── argocd/
│   ├── install/                         # Helm values for ArgoCD
│   └── applicationsets/
│       ├── platform.yaml                # Sync platform-* repos
│       └── services.yaml                # Sync svc-* repos (Git + Cluster generators)
├── projects/                            # ArgoCD Projects (RBAC)
└── README.md

platform-networking/
├── cilium/
│   ├── values.yaml                      # Cilium Helm config
│   └── policies/                        # ClusterNetworkPolicies
├── gateway-api/
│   ├── gateway-class.yaml
│   ├── gateways/
│   └── httproutes/
└── README.md

platform-observability/
├── otel-collector/
│   ├── daemonset.yaml                   # Node-level collection
│   ├── deployment.yaml                  # Gateway collector
│   └── config/
│       ├── receivers.yaml
│       ├── processors.yaml              # Cardinality filtering, PII scrubbing
│       ├── exporters.yaml
│       └── sampling.yaml                # Tail sampling config
├── prometheus/
│   ├── values.yaml
│   ├── rules/                           # AlertRules, RecordingRules
│   └── serviceMonitors/
├── loki/
│   ├── values.yaml
│   └── retention-policies.yaml          # GDPR: 30 days max
├── tempo/
│   └── values.yaml
├── pyroscope/                           # Continuous Profiling (APM)
│   ├── values.yaml
│   └── scrape-configs.yaml
├── sentry/                              # Error Tracking (APM)
│   ├── values.yaml
│   ├── dsn-config.yaml
│   └── alert-rules.yaml
├── grafana/
│   ├── values.yaml
│   ├── dashboards/
│   │   ├── platform/
│   │   ├── services/
│   │   └── apm/                         # APM-specific dashboards
│   │       ├── service-overview.json
│   │       ├── dependency-map.json
│   │       ├── database-performance.json
│   │       └── profiling-flamegraphs.json
│   └── datasources/
└── README.md

platform-cache/
├── valkey/
│   ├── values.yaml                      # Helm config for Valkey
│   ├── cluster-config.yaml
│   └── monitoring/
│       ├── servicemonitor.yaml
│       └── alerts.yaml
├── sdk/
│   ├── python/                          # Cache SDK helpers
│   │   ├── cache_client.py
│   │   └── patterns.py                  # Cache-aside, write-through
│   └── go/
│       └── cache/
└── README.md

platform-security/
├── vault/
│   ├── policies/                        # Per-service policies
│   ├── auth-methods/                    # Kubernetes auth
│   └── secret-engines/
├── external-secrets/
│   ├── operator/
│   └── cluster-secret-stores/
├── kyverno/
│   ├── cluster-policies/
│   │   ├── require-labels.yaml
│   │   ├── require-probes.yaml
│   │   ├── require-resource-limits.yaml
│   │   ├── restrict-privileged.yaml
│   │   ├── require-image-signature.yaml # Supply chain
│   │   └── mutate-default-sa.yaml
│   └── policy-reports/
├── supply-chain/
│   ├── cosign/                          # Image signing config
│   └── sbom/                            # Syft config
├── audit/
│   └── audit-policy.yaml                # K8s audit logging
└── README.md

══════════════════════════════════════════════════════════════════════════════
TIER 2 — CONTRACTS (Shared ownership)
══════════════════════════════════════════════════════════════════════════════

contracts-proto/
├── buf.yaml
├── buf.gen.yaml
├── localplus/
│   ├── ledger/v1/
│   │   ├── ledger.proto
│   │   └── ledger_service.proto
│   ├── wallet/v1/
│   │   ├── wallet.proto
│   │   └── wallet_service.proto
│   └── common/v1/
│       ├── money.proto
│       └── pagination.proto
└── README.md

sdk-python/
├── localplus/
│   ├── clients/                         # Generated gRPC clients
│   ├── telemetry/                       # OTel instrumentation helpers
│   ├── testing/                         # Fixtures, factories
│   └── security/                        # Vault client wrapper
├── pyproject.toml
└── README.md

sdk-go/
├── clients/
├── telemetry/
└── go.mod

══════════════════════════════════════════════════════════════════════════════
TIER 3 — DOMAIN SERVICES (Product Team ownership)
══════════════════════════════════════════════════════════════════════════════

svc-ledger/                              # TON LOCAL-PLUS ACTUEL
├── src/
│   └── app/
│       ├── api/
│       ├── domain/
│       ├── infrastructure/
│       └── main.py
├── tests/
│   ├── unit/                            # pytest, mocks
│   ├── integration/                     # testcontainers
│   ├── contract/                        # pact / grpc-testing
│   └── conftest.py
├── perf/
│   ├── k6/
│   │   ├── smoke.js
│   │   ├── load.js
│   │   └── stress.js
│   └── scenarios/
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── prod/
├── migrations/                          # Alembic
├── Dockerfile
├── Taskfile.yml
└── README.md

svc-wallet/                              # Même structure
svc-merchant/                            # Même structure
svc-giftcard/                            # Même structure
svc-notification/                        # Même structure (+ Kafka consumer)

══════════════════════════════════════════════════════════════════════════════
TIER 4 — QUALITY ENGINEERING (Shared ownership)
══════════════════════════════════════════════════════════════════════════════

e2e-scenarios/
├── scenarios/
│   ├── earn-burn-flow.spec.ts
│   ├── merchant-onboarding.spec.ts
│   └── giftcard-purchase.spec.ts
├── fixtures/
├── playwright.config.ts
└── README.md

chaos-experiments/
├── litmus/
│   └── chaosengine/
├── experiments/
│   ├── pod-kill/
│   ├── network-partition/
│   ├── db-latency/
│   └── kafka-broker-kill/
└── README.md

══════════════════════════════════════════════════════════════════════════════
TIER 5 — DOCUMENTATION (Shared ownership)
══════════════════════════════════════════════════════════════════════════════

docs/
├── adr/                                 # Architecture Decision Records
│   ├── 001-modular-monolith-first.md
│   ├── 002-aiven-managed-data.md
│   ├── 003-cilium-over-calico.md
│   └── ...
├── runbooks/
│   ├── incident-response.md
│   ├── database-failover.md
│   ├── kafka-recovery.md
│   └── secret-rotation.md
├── platform-contracts/
│   ├── deployment-sla.md
│   ├── observability-requirements.md
│   └── security-baseline.md
├── compliance/
│   ├── gdpr/
│   │   ├── data-retention-policy.md
│   │   ├── right-to-erasure.md
│   │   └── consent-management.md
│   ├── pci-dss/
│   │   ├── cardholder-data-flow.md
│   │   └── encryption-requirements.md
│   └── soc2/
│       ├── access-control-policy.md
│       └── incident-response-policy.md
├── threat-models/
│   ├── svc-ledger-stride.md
│   └── platform-attack-surface.md
└── onboarding/
    ├── new-developer.md
    └── new-service-checklist.md
```

---

# 🥚🐔 **PARTIE IV — BOOTSTRAP STRATEGY**

## **4.1 Layer 0 — Manual Bootstrap (1x per AWS account)**

| Action | Commande/Outil | Output |
|--------|---------------|--------|
| Créer IAM Role pour Terraform CI/CD | AWS CLI | `arn:aws:iam::xxx:role/TerraformCI` |
| Configurer OIDC pour GitHub Actions | AWS Console/CLI | GitHub peut assumer le role |

**C'est TOUT. Le S3 backend est auto-créé par Terraform 1.10+**

## **4.1.1 GitHub Actions — Reusable & Composite Workflows**

> **Note :** Utiliser des **reusable workflows** et **composite actions** pour standardiser les pipelines CI/CD.

- **Reusable workflows** : `.github/workflows/` partagés entre repos (build, test, deploy)
- **Composite actions** : `.github/actions/` pour encapsuler des steps communs (setup-python, terraform-plan, etc.)

## **4.2 Layer 1 — Foundation (Terraform)**

| Ordre | Ressource | Dépendances |
|-------|-----------|-------------|
| 1 | VPC + Subnets | Aucune |
| 2 | KMS Keys | Aucune |
| 3 | EKS Cluster | VPC, KMS |
| 4 | IRSA (IAM Roles for Service Accounts) | EKS |
| 5 | VPC Peering avec Aiven | VPC, Aiven créé manuellement d'abord |
| 6 | Outputs → Platform repos | Tous |

## **4.3 Layer 2 — Platform Bootstrap**

| Ordre | Action | Dépendance |
|-------|--------|------------|
| 1 | Install ArgoCD via Helm (1x) | EKS ready |
| 2 | Apply App-of-Apps ApplicationSet | ArgoCD running |
| 3 | ArgoCD syncs platform-* repos | Reconciliation automatique |

**ArgoCD : Instance centralisée unique** (comme demandé)

## **4.4 Layer 3+ — Application Services**

ArgoCD ApplicationSets avec **Git Generator + Matrix Generator** découvrent automatiquement les services.

---

# 🧪 **PARTIE V — TESTING STRATEGY COMPLÈTE**

## **5.1 Terraform Testing**

| Type | Outil | Quand | Bloquant |
|------|-------|-------|----------|
| **Format/Lint** | `terraform fmt`, `tflint` | Pre-commit | Oui |
| **Security scan** | `tfsec`, `checkov` | PR | Oui |
| **Compliance** | `regula`, `opa conftest`, [terraform-compliance](https://terraform-compliance.com/) | PR | Oui |
| **Policy as Code** | HashiCorp Sentinel | PR | Oui |
| **Unit tests** | `terraform test` (native 1.6+) | PR | Oui |
| **Integration** | `terratest` | Nightly | Non |
| **Drift detection** | `terraform plan` scheduled | Daily | Alerte |

## **5.2 Application Testing**

| Type | Localisation | Outil | Trigger | Bloquant |
|------|--------------|-------|---------|----------|
| **Unit** | `svc-*/tests/unit/` | pytest | Pre-commit, PR | Oui |
| **Integration** | `svc-*/tests/integration/` | pytest + testcontainers | PR | Oui |
| **Contract** | `svc-*/tests/contract/` | pact, grpc-testing | PR | Oui |
| **Performance** | `svc-*/perf/` | k6 | Nightly, Pre-release | Non |
| **E2E** | `e2e-scenarios/` | Playwright | Post-merge staging | Oui pour prod |
| **Chaos** | `chaos-experiments/` | Litmus | Weekly | Non |

## **5.3 TNR (Tests de Non-Régression)**

| Catégorie | Contenu | Fréquence |
|-----------|---------|-----------|
| **Critical Paths** | Earn → Balance Update → Notification | Nightly |
| **Golden Master** | Snapshot des réponses API | Nightly |
| **Compliance** | GDPR data retention, PCI encryption checks | Nightly |
| **Security** | Kyverno policy audit, image signature verification | Nightly |

## **5.4 Compliance Testing**

| Standard | Test | Outil |
|----------|------|-------|
| **GDPR** | PII not in logs | OTel Collector scrubbing + log audit |
| **GDPR** | Data retention < 30 days | Loki retention policy check |
| **PCI-DSS** | mTLS enforced | Cilium policy audit |
| **PCI-DSS** | Encryption at rest | AWS KMS audit |
| **SOC2** | Audit logs present | CloudTrail + K8s audit logs check |
| **SOC2** | Access control | Kyverno policy reports |

---

# 🔐 **PARTIE VI — SECURITY ARCHITECTURE**

## **6.1 Defense in Depth**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAYER 1: PERIMETER                                                          │
│ • AWS WAF on ALB (si exposé internet)                                       │
│ • AWS Shield (DDoS)                                                         │
│ • Cilium Gateway API (TLS termination)                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAYER 2: NETWORK                                                            │
│ • VPC isolation (private subnets only for workloads)                        │
│ • Cilium NetworkPolicies (default deny, explicit allow)                     │
│ • VPC Peering Aiven (no public internet for DB/Kafka)                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAYER 3: IDENTITY & ACCESS                                                  │
│ • IRSA (IAM Roles for Service Accounts) — no static credentials            │
│ • Cilium mTLS (WireGuard) — pod-to-pod encryption                          │
│ • Vault dynamic secrets — DB credentials rotated                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAYER 4: WORKLOAD                                                           │
│ • Kyverno policies (no privileged, resource limits, probes required)       │
│ • Image signature verification (Cosign)                                    │
│ • Read-only root filesystem                                                │
│ • Non-root containers                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAYER 5: DATA                                                               │
│ • Encryption at rest (AWS KMS, Aiven native)                               │
│ • Encryption in transit (mTLS)                                             │
│ • PII scrubbing in logs (OTel processor)                                   │
│ • Audit trail immutable (CloudTrail, K8s audit logs)                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## **6.2 Réponse à : "Commence simple, mais la dette technique ?"**

**Le paradoxe :** Tu veux commencer simple mais avec GDPR/PCI-DSS/SOC2, tu ne peux PAS ignorer la sécurité.

**La solution : Security Baseline dès Day 1, évolution par phases**

| Phase | Ce qui est en place | Ce qui vient après |
|-------|---------------------|-------------------|
| **Day 1** | Cilium mTLS (zero config), Kyverno basic policies, Vault pour secrets | - |
| **Month 3** | Image signing (Cosign), SBOM generation | - |
| **Month 6** | SPIRE (si multi-cluster), Confidential Computing évaluation | - |

**Pas de dette technique SI :**
- mTLS dès le début (Cilium = zero effort)
- Secrets dans Vault dès le début (pas de migration douloureuse)
- Policies Kyverno dès le début (culture sécurité)

**La vraie dette technique serait :**
- Commencer sans mTLS → Migration massive plus tard
- Secrets en ConfigMaps → Rotation impossible
- Pas d'audit logs → Compliance failure

---

# 📊 **PARTIE VII — OBSERVABILITY ARCHITECTURE**

## **7.1 Stack Self-Hosted (Coût Minimal)**

| Composant | Outil | Coût | Retention |
|-----------|-------|------|-----------|
| **Metrics** | Prometheus | 0€ (self-hosted) | 15 jours local |
| **Metrics long-term** | Thanos Sidecar → S3 | ~5€/mois S3 | 1 an |
| **Logs** | Loki | 0€ (self-hosted) | 30 jours (GDPR) |
| **Traces** | Tempo | 0€ (self-hosted) | 7 jours |
| **Dashboards** | Grafana | 0€ (self-hosted) | N/A |
| **Fallback logs** | CloudWatch Logs | Tier gratuit 5GB | 7 jours |

**Coût estimé : < 50€/mois** (principalement S3 pour Thanos)

## **7.2 Telemetry Pipeline**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Applications   │     │  OTel Collector │     │   Backends      │
│                 │     │                 │     │                 │
│  • SDK Python   │────►│  • Receivers    │────►│  • Prometheus   │
│  • Auto-instr   │     │  • Processors   │     │  • Loki         │
│                 │     │  • Exporters    │     │  • Tempo        │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               │ Scrubbing
                               ▼
                        ┌─────────────────┐
                        │ GDPR Compliant  │
                        │ • No user_id    │
                        │ • No PII        │
                        │ • No PAN        │
                        └─────────────────┘
```

## **7.3 Cardinality Management**

| Label | Action | Rationale |
|-------|--------|-----------|
| `user_id` | DROP | High cardinality, use traces |
| `request_id` | DROP | Use trace_id instead |
| `http.url` | DROP | URLs uniques = explosion |
| `http.route` | KEEP | Templated, low cardinality |
| `service.name` | KEEP | Essential |
| `http.method` | KEEP | Low cardinality |
| `http.status_code` | KEEP | Low cardinality |

## **7.4 SLI/SLO/Error Budgets**

| Service | SLI | SLO | Error Budget |
|---------|-----|-----|--------------|
| **svc-ledger** | Availability | 99.9% | 43 min/mois |
| **svc-ledger** | Latency P99 | < 200ms | N/A |
| **svc-wallet** | Availability | 99.9% | 43 min/mois |
| **Platform (ArgoCD, Prometheus)** | Availability | 99.5% | 3.6h/mois |

## **7.5 Alerting Strategy**

| Severity | Exemple | Notification | On-call |
|----------|---------|--------------|---------|
| **P1 — Critical** | svc-ledger down | PagerDuty immediate | Wake up |
| **P2 — High** | Error rate > 5% | Slack + PagerDuty 15min | Within 30min |
| **P3 — Medium** | Latency P99 > 500ms | Slack | Business hours |
| **P4 — Low** | Disk usage > 80% | Slack | Next day |

## **7.6 APM (Application Performance Monitoring)**

### **7.6.1 Stack APM**

| Composant | Outil | Intégration | Usage |
|-----------|-------|-------------|-------|
| **Distributed Tracing** | Tempo + OTel | Auto-instrumentation Python/Go | Request flow, latency breakdown |
| **Profiling** | Pyroscope (Grafana) | SDK intégré | CPU/Memory profiling continu |
| **Error Tracking** | Sentry (self-hosted) | SDK Python/Go | Exception tracking, stack traces |
| **Database APM** | pg_stat_statements | Prometheus exporter | Query performance |
| **Real User Monitoring** | Grafana Faro | JavaScript SDK | Frontend performance (si applicable) |

### **7.6.2 APM Pipeline**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ OTel SDK     │  │ Pyroscope    │  │ Sentry SDK   │  │ pg_stat      │    │
│  │ (Traces)     │  │ (Profiles)   │  │ (Errors)     │  │ (DB metrics) │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                 │                 │                 │             │
└─────────┼─────────────────┼─────────────────┼─────────────────┼─────────────┘
          │                 │                 │                 │
          ▼                 ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         COLLECTION LAYER                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    OTel Collector (Gateway)                          │   │
│  │  • Receives: traces, metrics, logs                                   │   │
│  │  • Processes: sampling, enrichment, PII scrubbing                    │   │
│  │  • Exports: Tempo, Prometheus, Loki                                  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STORAGE & VISUALIZATION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Tempo    │  │ Pyroscope  │  │   Sentry   │  │  Grafana   │            │
│  │  (Traces)  │  │ (Profiles) │  │  (Errors)  │  │ (Unified)  │            │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### **7.6.3 Instrumentation Standards**

| Language | Auto-instrumentation | Manual Instrumentation | Frameworks supportés |
|----------|---------------------|------------------------|---------------------|
| **Python** | `opentelemetry-instrumentation` | `@tracer.start_as_current_span` | FastAPI, SQLAlchemy, httpx, grpcio |
| **Go** | OTel contrib packages | `tracer.Start()` | gRPC, net/http, pgx |

### **7.6.4 Sampling Strategy**

| Environment | Head Sampling | Tail Sampling | Rationale |
|-------------|---------------|---------------|-----------|
| **Dev** | 100% | N/A | Full visibility pour debug |
| **Staging** | 50% | Errors: 100% | Balance cost/visibility |
| **Prod** | 10% | Errors: 100%, Slow: 100% (>500ms) | Cost optimization |

### **7.6.5 APM Dashboards**

| Dashboard | Métriques clés | Audience |
|-----------|---------------|----------|
| **Service Overview** | RPS, Error rate, Latency P50/P95/P99 | On-call |
| **Dependency Map** | Service topology, inter-service latency | Platform team |
| **Database Performance** | Query time, connections, deadlocks | Backend devs |
| **Error Analysis** | Error count by type, affected users | Product team |
| **Profiling Flame Graphs** | CPU hotspots, memory allocations | Performance team |

### **7.6.6 Trace-to-Logs-to-Metrics Correlation**

```
┌─────────────────┐     trace_id     ┌─────────────────┐
│     TRACES      │◄────────────────►│      LOGS       │
│     (Tempo)     │                  │     (Loki)      │
└────────┬────────┘                  └────────┬────────┘
         │                                    │
         │ Exemplars (trace_id in metrics)    │
         │                                    │
         ▼                                    ▼
┌─────────────────────────────────────────────────────────┐
│                    GRAFANA                               │
│  • Click trace → See logs for that request              │
│  • Click metric spike → Jump to exemplar trace          │
│  • Click error log → Navigate to full trace             │
└─────────────────────────────────────────────────────────┘
```

### **7.6.7 APM Alerting**

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| **High Error Rate** | Error rate > 1% for 5min | P2 | Investigate errors in Sentry |
| **Latency Degradation** | P99 > 2x baseline for 10min | P2 | Check traces for slow spans |
| **Database Slow Queries** | Query time P95 > 100ms | P3 | Analyze pg_stat_statements |
| **Memory Leak Detected** | Memory growth > 10%/hour | P3 | Check Pyroscope profiles |

---

# 💾 **PARTIE VIII — DATA ARCHITECTURE**

## **8.1 Aiven Configuration**

| Service | Plan | Config | Coût estimé |
|---------|------|--------|-------------|
| **PostgreSQL** | Business-4 | Primary + Read Replica, 100GB | ~300€/mois |
| **Kafka** | Business-4 | 3 brokers, 100GB retention | ~400€/mois |
| **Valkey (Redis)** | Business-4 | 2 nodes, 10GB, HA | ~150€/mois |

**Coût total Aiven estimé : ~850€/mois**

## **8.2 Database Strategy**

| Aspect | Choix | Rationale |
|--------|-------|-----------|
| **Replication** | Aiven managed (async) | RPO 1h acceptable |
| **Backup** | Aiven automated hourly | RPO 1h |
| **Failover** | Aiven automated | RTO < 15min |
| **Connection** | VPC Peering (private) | PCI-DSS, no public internet |
| **Pooling** | PgBouncer (Aiven built-in) | Connection efficiency |

## **8.3 Schema Ownership**

| Table | Owner Service | Access pattern |
|-------|---------------|----------------|
| `transactions` | svc-ledger | CRUD |
| `ledger_entries` | svc-ledger | CRUD |
| `wallets` | svc-wallet | CRUD |
| `balance_snapshots` | svc-wallet | CRUD |
| `merchants` | svc-merchant | CRUD |
| `giftcards` | svc-giftcard | CRUD |

**Règle : 1 table = 1 owner. Cross-service = gRPC ou Events, jamais JOIN.**

## **8.4 Kafka Topics**

| Topic | Producer | Consumers | Retention |
|-------|----------|-----------|-----------|
| `ledger.transactions.v1` | svc-ledger (Outbox) | svc-notification, svc-analytics | 7 jours |
| `wallet.balance-updated.v1` | svc-wallet | svc-analytics | 7 jours |
| `merchant.onboarded.v1` | svc-merchant | svc-notification | 7 jours |

## **8.5 Cache Architecture (Valkey/Redis)**

### **8.5.1 Stack Cache**

| Composant | Outil | Hébergement | Coût estimé |
|-----------|-------|-------------|-------------|
| **Cache primaire** | Valkey (Redis-compatible) | Aiven for Caching | ~150€/mois |
| **Cache local (L1)** | Python `cachetools` / Go `bigcache` | In-memory | 0€ |

> **Note :** Valkey est le fork open-source de Redis, maintenu par la Linux Foundation. Aiven supporte Valkey nativement.

### **8.5.2 Cache Topology**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MULTI-LAYER CACHE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ L1 — LOCAL CACHE (per pod)                                          │    │
│  │ • TTL: 30s - 5min                                                   │    │
│  │ • Size: 100MB max per pod                                           │    │
│  │ • Use case: Hot data, config, user sessions                         │    │
│  └───────────────────────────────┬─────────────────────────────────────┘    │
│                                  │ Cache miss                               │
│                                  ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ L2 — DISTRIBUTED CACHE (Valkey cluster)                             │    │
│  │ • TTL: 5min - 24h                                                   │    │
│  │ • Size: 10GB                                                        │    │
│  │ • Use case: Shared state, rate limits, session store                │    │
│  └───────────────────────────────┬─────────────────────────────────────┘    │
│                                  │ Cache miss                               │
│                                  ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ L3 — DATABASE (PostgreSQL)                                          │    │
│  │ • Source of truth                                                   │    │
│  │ • Write-through pour updates                                        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### **8.5.3 Cache Strategies par Use Case**

| Use Case | Strategy | TTL | Invalidation |
|----------|----------|-----|--------------|
| **Wallet Balance** | Cache-aside (read) | 30s | Event-driven (Kafka) |
| **Merchant Config** | Read-through | 5min | TTL + Manual |
| **Rate Limiting** | Write-through | Sliding window | Auto-expire |
| **Session Data** | Write-through | 24h | Explicit logout |
| **Gift Card Catalog** | Cache-aside | 15min | Event-driven |
| **Feature Flags** | Read-through | 1min | Config push |

### **8.5.4 Cache Patterns Implementation**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CACHE-ASIDE PATTERN                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Application checks cache                                                 │
│  2. If HIT → return cached data                                             │
│  3. If MISS → query database                                                │
│  4. Store result in cache with TTL                                          │
│  5. Return data to caller                                                   │
│                                                                              │
│  ┌─────────┐    GET     ┌─────────┐                                         │
│  │   App   │───────────►│  Cache  │                                         │
│  └────┬────┘            └────┬────┘                                         │
│       │                      │ MISS                                         │
│       │    SELECT            ▼                                              │
│       └─────────────────►┌─────────┐                                        │
│                          │   DB    │                                        │
│                          └─────────┘                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         WRITE-THROUGH PATTERN                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Application writes to cache AND database atomically                     │
│  2. Cache is always consistent with database                                │
│                                                                              │
│  ┌─────────┐   SET+TTL   ┌─────────┐                                        │
│  │   App   │────────────►│  Cache  │                                        │
│  └────┬────┘             └─────────┘                                        │
│       │                                                                      │
│       │   INSERT/UPDATE                                                      │
│       └─────────────────►┌─────────┐                                        │
│                          │   DB    │                                        │
│                          └─────────┘                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### **8.5.5 Cache Invalidation Strategy**

| Trigger | Méthode | Use Case |
|---------|---------|----------|
| **TTL Expiry** | Automatic | Default pour toutes les clés |
| **Event-driven** | Kafka consumer | Wallet balance après transaction |
| **Explicit Delete** | API call | Admin actions, config updates |
| **Pub/Sub** | Valkey PUBLISH | Real-time invalidation cross-pods |

### **8.5.6 Cache Key Naming Convention**

```
{service}:{entity}:{id}:{version}

Exemples:
  wallet:balance:user_123:v1
  merchant:config:merchant_456:v1
  giftcard:catalog:category_active:v1
  ratelimit:api:user_123:minute
  session:auth:session_abc123
```

### **8.5.7 Cache Metrics & Monitoring**

| Metric | Seuil alerte | Action |
|--------|--------------|--------|
| **Hit Rate** | < 80% | Revoir TTL, préchargement |
| **Latency P99** | > 10ms | Check network, cluster size |
| **Memory Usage** | > 80% | Eviction analysis, scale up |
| **Evictions/sec** | > 100 | Augmenter cache size |
| **Connection Errors** | > 0 | Check connectivity, pooling |

## **8.6 Queueing & Background Jobs**

### **8.6.1 Queueing Architecture Overview**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         QUEUEING ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ TIER 1 — EVENT STREAMING (Kafka)                                    │    │
│  │ • Use case: Event-driven architecture, CDC, audit logs              │    │
│  │ • Pattern: Pub/Sub, Event Sourcing                                  │    │
│  │ • Retention: 7 jours                                                │    │
│  │ • Ordering: Per-partition                                           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ TIER 2 — TASK QUEUE (Valkey + Python Dramatiq/ARQ)                  │    │
│  │ • Use case: Background jobs, async processing                       │    │
│  │ • Pattern: Producer/Consumer, Work Queue                            │    │
│  │ • Features: Retries, priorities, scheduling                         │    │
│  │ • Durability: Redis persistence                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ TIER 3 — SCHEDULED JOBS (Kubernetes CronJobs)                       │    │
│  │ • Use case: Batch processing, reports, cleanup                      │    │
│  │ • Pattern: Time-triggered execution                                 │    │
│  │ • Managed: K8s native                                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### **8.6.2 Kafka vs Task Queue — Decision Matrix**

| Critère | Kafka | Task Queue (Valkey) |
|---------|-------|---------------------|
| **Message Ordering** | ✅ Per-partition | ❌ Best effort |
| **Message Replay** | ✅ Retention-based | ❌ Non |
| **Priority Queues** | ❌ Non natif | ✅ Oui |
| **Delayed Messages** | ❌ Non natif | ✅ Oui |
| **Dead Letter Queue** | ✅ Configurable | ✅ Intégré |
| **Exactly-once** | ✅ Avec idempotency | ❌ At-least-once |
| **Throughput** | 🚀 Très élevé | 📈 Élevé |
| **Use Case** | Events, CDC, Streaming | Jobs, Tasks, Async work |

### **8.6.3 Task Queue Stack**

| Composant | Outil | Rôle |
|-----------|-------|------|
| **Task Framework** | Dramatiq (Python) / Asynq (Go) | Task definition, execution |
| **Broker** | Valkey (Redis-compatible) | Message storage, routing |
| **Result Backend** | Valkey | Task results, status |
| **Scheduler** | APScheduler / Dramatiq-crontab | Periodic tasks |
| **Monitoring** | Dramatiq Dashboard / Prometheus | Task metrics |

### **8.6.4 Task Queue Patterns**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TASK PROCESSING FLOW                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Producer                    Broker                    Workers              │
│  ┌─────────┐               ┌─────────┐               ┌─────────┐            │
│  │ svc-*   │──── enqueue ──►│ Valkey  │◄── poll ─────│ Worker  │            │
│  │ API     │               │         │               │ Pods    │            │
│  └─────────┘               │ Queues: │               └────┬────┘            │
│                            │ • high  │                    │                 │
│                            │ • default│                   │ execute         │
│                            │ • low   │                    ▼                 │
│                            │ • dlq   │              ┌─────────┐             │
│                            └─────────┘              │  Task   │             │
│                                                     │ Handler │             │
│                                                     └─────────┘             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### **8.6.5 Queue Definitions**

| Queue | Priority | Workers | Use Cases |
|-------|----------|---------|-----------|
| **critical** | P0 | 5 | Transaction rollbacks, fraud alerts |
| **high** | P1 | 10 | Email confirmations, balance updates |
| **default** | P2 | 20 | Notifications, analytics events |
| **low** | P3 | 5 | Reports, cleanup, batch exports |
| **scheduled** | N/A | 3 | Cron-like scheduled tasks |
| **dead-letter** | N/A | 1 | Failed tasks investigation |

### **8.6.6 Retry Strategy**

| Retry Policy | Configuration | Use Case |
|--------------|---------------|----------|
| **Exponential Backoff** | base=1s, max=1h, multiplier=2 | API calls, external services |
| **Fixed Interval** | interval=30s, max_retries=5 | Database operations |
| **No Retry** | max_retries=0 | Idempotent operations |

```
Retry Timeline (Exponential):
  Attempt 1: immediate
  Attempt 2: +1s
  Attempt 3: +2s
  Attempt 4: +4s
  Attempt 5: +8s
  ...
  Attempt N: move to DLQ
```

### **8.6.7 Dead Letter Queue (DLQ) Handling**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DLQ WORKFLOW                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Task fails after max retries                                            │
│  2. Task moved to DLQ with metadata:                                        │
│     • Original queue                                                        │
│     • Failure reason                                                        │
│     • Stack trace                                                           │
│     • Attempt count                                                         │
│     • Timestamp                                                             │
│  3. Alert sent to Slack (P3)                                                │
│  4. On-call investigates                                                    │
│  5. Options:                                                                │
│     a) Fix bug → Replay task                                                │
│     b) Manual resolution → Delete from DLQ                                  │
│     c) Archive for audit                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### **8.6.8 Scheduled Jobs (CronJobs)**

| Job | Schedule | Service | Description |
|-----|----------|---------|-------------|
| **balance-reconciliation** | `0 2 * * *` | svc-wallet | Daily balance verification |
| **expired-giftcards** | `0 0 * * *` | svc-giftcard | Mark expired cards |
| **analytics-rollup** | `0 */6 * * *` | svc-analytics | 6-hourly aggregation |
| **log-cleanup** | `0 3 * * 0` | platform | Weekly log rotation |
| **backup-verification** | `0 4 * * *` | platform | Daily backup integrity check |
| **compliance-report** | `0 6 1 * *` | platform | Monthly compliance export |

### **8.6.9 Task Queue Monitoring**

| Metric | Seuil alerte | Action |
|--------|--------------|--------|
| **Queue Depth** | > 1000 tasks | Scale workers |
| **Processing Time P95** | > 30s | Optimize task, check resources |
| **Failure Rate** | > 5% | Investigate DLQ, check dependencies |
| **DLQ Size** | > 10 tasks | Immediate investigation |
| **Worker Availability** | < 50% | Check pod health, scale up |

---

# 🌐 **PARTIE IX — NETWORKING ARCHITECTURE**

## **9.1 VPC Design**

| CIDR | Usage |
|------|-------|
| 10.0.0.0/16 | VPC Principal |
| 10.0.0.0/20 | Private Subnets (Workloads) |
| 10.0.16.0/20 | Private Subnets (Data) |
| 10.0.32.0/20 | Public Subnets (NAT, LB) |

## **9.2 Traffic Flow**

| Flow | Path | Encryption |
|------|------|------------|
| Internet → Services | ALB → Cilium Gateway → Pod | TLS + mTLS |
| Service → Service | Pod → Pod (Cilium) | mTLS (WireGuard) |
| Service → Aiven | VPC Peering | TLS |
| Service → AWS (S3, KMS) | VPC Endpoints | TLS |

## **9.3 Gateway API Configuration**

| Resource | Purpose |
|----------|---------|
| **GatewayClass** | Cilium implementation |
| **Gateway** | HTTPS listener, TLS termination |
| **HTTPRoute** | Routing vers services (path-based) |

## **9.4 Network Policies (Default Deny)**

| Policy | Effect |
|--------|--------|
| Default deny all | Aucun trafic sauf explicite |
| Allow intra-namespace | Services même namespace peuvent communiquer |
| Allow specific cross-namespace | svc-ledger → svc-wallet explicite |
| Allow egress Aiven | Services → VPC Peering range only |
| Allow egress AWS endpoints | Services → VPC Endpoints only |

---

# ⚡ **PARTIE X — RESILIENCE & DR**

## **10.1 Failure Modes**

| Failure | Detection | Recovery | RTO |
|---------|-----------|----------|-----|
| **Pod crash** | Liveness probe | K8s restart | < 30s |
| **Node failure** | Node NotReady | Pod reschedule | < 2min |
| **AZ failure** | Multi-AZ detect | Traffic shift | < 5min |
| **DB primary failure** | Aiven health | Automatic failover | < 5min |
| **Kafka broker failure** | Aiven health | Automatic rebalance | < 2min |
| **Full region failure** | Manual | DR procedure (future) | 4h (target) |

## **10.2 Backup Strategy**

| Data | Method | Frequency | Retention | Location |
|------|--------|-----------|-----------|----------|
| **PostgreSQL** | Aiven automated | Hourly | 7 jours | Aiven (cross-AZ) |
| **PostgreSQL PITR** | Aiven WAL | Continuous | 24h | Aiven |
| **Kafka** | Topic retention | N/A | 7 jours | Aiven |
| **Terraform state** | S3 versioning | Every apply | 90 jours | S3 |
| **Git repos** | GitHub | Every push | Infini | GitHub |

## **10.3 Disaster Recovery (Future)**

| Scenario | Current | Future (Multi-region) |
|----------|---------|----------------------|
| Single AZ failure | Automatic (multi-AZ) | Automatic |
| Region failure | Manual restore from backup | Automatic failover |
| Data corruption | PITR restore | PITR restore |

---

# 🛠️ **PARTIE XI — PLATFORM ENGINEERING**

## **11.1 Platform Contracts**

| Contrat | Garantie Platform | Responsabilité Service |
|---------|-------------------|------------------------|
| **Deployment** | Git push → Prod < 15min | Manifests K8s valides |
| **Secrets** | Vault dynamic, rotation auto | Utiliser External-Secrets |
| **Observability** | Auto-collection traces/metrics/logs | Instrumentation OTel |
| **Networking** | mTLS enforced, Gateway API | Déclarer routes dans HTTPRoute |
| **Scaling** | HPA disponible | Configurer requests/limits |
| **Security** | Policies enforced | Passer les policies |

## **11.2 Golden Path (New Service Checklist)**

| Étape | Action | Validation |
|-------|--------|------------|
| 1 | Créer repo depuis template | Structure conforme |
| 2 | Définir protos dans contracts-proto | buf lint pass |
| 3 | Implémenter service | Unit tests > 80% |
| 4 | Configurer K8s manifests | Kyverno policies pass |
| 5 | Configurer External-Secret | Secrets résolus |
| 6 | Ajouter ServiceMonitor | Metrics visibles Grafana |
| 7 | Créer HTTPRoute | Trafic routable |
| 8 | PR review | Merge → Auto-deploy dev |

## **11.3 On-Call Structure (5 personnes)**

| Rôle | Responsabilité | Rotation |
|------|---------------|----------|
| **Primary** | First responder, triage | Weekly |
| **Secondary** | Escalation, expertise | Weekly |
| **Incident Commander** | Coordination si P1 | On-demand |

---

# 📊 **PARTIE XII — MAPPING TERMINOLOGIE**

| Terme | Application concrète Local-Plus |
|-------|--------------------------------|
| **Reconciliation loop** | ArgoCD sync, Kyverno background scan |
| **Desired state store** | Git repos |
| **Drift detection** | ArgoCD diff, `terraform plan` scheduled |
| **Blast radius** | Namespace isolation, PDB, Resource Quotas |
| **Tenant isolation** | Vault policies per service, Network Policies |
| **Paved road / Golden path** | Template service, checklist onboarding |
| **Guardrails** | Kyverno policies (not gates) |
| **Ephemeral credentials** | Vault dynamic DB secrets (TTL) |
| **SLI/SLO/SLA** | Prometheus recording rules, Error budgets |
| **Cardinality** | OTel Collector label filtering |
| **Circuit breaker** | Cilium timeout policies |
| **Outbox pattern** | svc-ledger → Kafka transactional |
| **Control plane vs Data plane** | platform-* repos vs svc-* repos |
| **Progressive delivery** | Argo Rollouts (canary) — future |
| **Idempotency** | Idempotency-Key header (SYSTEM_CONTRACT.md) |
| **Pessimistic locking** | SELECT FOR UPDATE (SYSTEM_CONTRACT.md) |
| **Error budget** | 43 min/mois pour 99.9% SLO |
| **MTTR** | Target < 15min (RTO) |
| **Runbook** | docs/runbooks/*.md |
| **Postmortem** | docs/postmortems/*.md (blameless) |
| **APM (Application Performance Monitoring)** | Tempo + Pyroscope + Sentry |
| **Distributed Tracing** | OTel → Tempo, trace_id correlation |
| **Profiling** | Pyroscope (CPU/Memory flame graphs) |
| **Cache-aside pattern** | Valkey lookup, DB fallback, cache on miss |
| **Write-through cache** | Sync write to cache + DB |
| **Cache invalidation** | TTL + Event-driven (Kafka) + Pub/Sub |
| **L1/L2 Cache** | L1=In-memory (pod), L2=Valkey (distributed) |
| **Task Queue** | Dramatiq + Valkey (background jobs) |
| **Dead Letter Queue (DLQ)** | Failed tasks après max retries |
| **Exponential Backoff** | Retry avec délai croissant (1s, 2s, 4s...) |
| **Priority Queue** | critical > high > default > low |
| **CronJob** | K8s scheduled tasks (batch, cleanup) |
| **Rate Limiting** | Valkey sliding window counter |

---

# 🚀 **PARTIE XIII — SÉQUENCE DE CONSTRUCTION**

| Phase | Focus | Livrables | Estimation |
|-------|-------|-----------|------------|
| **1** | Bootstrap Layer 0-1 | IAM, VPC, EKS, Aiven setup (PG, Kafka, Valkey) | 3 semaines |
| **2** | Platform GitOps | ArgoCD, ApplicationSets | 1 semaine |
| **3** | Platform Networking | Cilium, Gateway API | 1 semaine |
| **4** | Platform Security | Vault, External-Secrets, Kyverno | 2 semaines |
| **5** | Platform Observability | OTel, Prometheus, Loki, Tempo, Grafana | 2 semaines |
| **5b** | Platform APM | Pyroscope, Sentry, APM Dashboards | 1 semaine |
| **6** | Platform Cache | Valkey setup, SDK integration | 1 semaine |
| **7** | Contracts | Proto definitions, SDK Python | 1 semaine |
| **8** | svc-ledger | Migrate ton local-plus, full tests | 3 semaines |
| **9** | svc-wallet | Second service, gRPC integration | 2 semaines |
| **10** | Kafka + Outbox | Event-driven patterns | 2 semaines |
| **10b** | Task Queue | Dramatiq setup, background workers | 1 semaine |
| **11** | Testing complet | TNR, Perf, Chaos | 2 semaines |
| **12** | Compliance audit | GDPR, PCI-DSS, SOC2 checks | 2 semaines |
| **13** | Documentation | Runbooks, ADRs, Onboarding | 1 semaine |

**Total : ~25 semaines**

---

# ✅ **PARTIE XIV — CHECKLIST FINALE**

## **Avant de commencer :**

- [ ] Compte AWS créé, billing configuré
- [ ] Compte Aiven créé
- [ ] Organisation GitHub créée
- [ ] Décision : HashiCorp Vault self-hosted sur EKS
- [ ] Domaine DNS acquis

## **Décisions architecturales validées :**

- [ ] RPO 1h, RTO 15min — OK
- [ ] AWS eu-west-1 — OK
- [ ] Aiven pour Kafka + PostgreSQL — OK
- [ ] Self-hosted observability — OK
- [ ] ArgoCD centralisé — OK
- [ ] Cilium + Gateway API — OK
- [ ] Kyverno — OK
- [ ] GDPR + PCI-DSS + SOC2 — OK

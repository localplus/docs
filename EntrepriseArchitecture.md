# 🏗️ **LOCAL-PLUS — Architecture Overview**
## *Gift Card & Loyalty Platform*
### *Version 1.0 — Janvier 2026*

---

> **Ce document est la porte d'entrée de l'architecture LOCAL-PLUS.**  
> Il fournit une vue d'ensemble et des liens vers la documentation détaillée.

---

# 📋 **PARTIE I — EXECUTIVE SUMMARY**

## **1.1 Scope**

LOCAL-PLUS est une plateforme de gestion de cartes cadeaux et fidélité, conçue pour :
- **Scalabilité** : 500 TPS, 1500 RPS
- **Résilience** : RPO 1h, RTO 15min
- **Compliance** : GDPR, PCI-DSS, SOC2
- **Durée de vie** : 5+ ans

### **Non-Goals (Phase 1)**
- Multi-région active-active
- API Gateway/APIM dédié (évaluation future)
- Mobile apps natives

## **1.2 Paramètres Clés**

| Paramètre | Valeur | Impact |
|-----------|--------|--------|
| **RPO** | 1 heure | Backups horaires, réplication async |
| **RTO** | 15 minutes | Failover automatisé |
| **TPS** | 500 transactions/sec | Single Postgres suffit |
| **RPS** | 1500 requêtes/sec | Load balancer + HPA standard |
| **Équipe on-call** | 5 personnes | Runbooks exhaustifs |

## **1.3 Compliance Summary**

| Standard | Exigences clés | Documentation |
|----------|---------------|---------------|
| **GDPR** | Data residency EU, droit à l'oubli | → [docs/compliance/gdpr/](compliance/gdpr/) |
| **PCI-DSS** | Pas de stockage PAN, encryption, audit | → [docs/compliance/pci-dss/](compliance/pci-dss/) |
| **SOC2** | RBAC, monitoring, incident response | → [docs/compliance/soc2/](compliance/soc2/) |

## **1.4 Tech Stack Overview**

| Catégorie | Choix | Rationale |
|-----------|-------|-----------|
| **Cloud** | AWS (eu-west-1) | Décision business, GDPR |
| **Orchestration** | EKS + ArgoCD | GitOps, cloud-native |
| **Database** | Aiven PostgreSQL | Managed, PCI compliant |
| **Messaging** | Aiven Kafka | Event-driven, managed |
| **Cache** | Aiven Valkey | Redis-compatible, managed |
| **Edge/CDN** | Cloudflare | WAF, DDoS, Zero Trust |
| **Observability** | Prometheus/Loki/Tempo | Self-hosted, coût minimal |
| **Secrets** | HashiCorp Vault | Dynamic secrets, rotation |
| **CNI** | Cilium | mTLS, Gateway API |
| **Policies** | Kyverno | Admission control |

---

# 🏛️ **PARTIE II — ARCHITECTURE**

## **2.1 Context Diagram (C4 Level 1)**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              END USERS                                       │
│                    (Merchants, Consumers, Partners)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLOUDFLARE EDGE                                      │
│              (DNS, WAF, DDoS, CDN, Zero Trust, Tunnel)                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LOCAL-PLUS PLATFORM                                  │
│                              (AWS EKS)                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Domain Services: svc-ledger, svc-wallet, svc-merchant, svc-giftcard │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AIVEN DATA LAYER                                     │
│              (PostgreSQL, Kafka, Valkey — VPC Peering)                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

## **2.2 Container Diagram (C4 Level 2)**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AWS WORKLOAD ACCOUNT — eu-west-1                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                           EKS CLUSTER                                   │ │
│  │                                                                         │ │
│  │   ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │   │ PLATFORM NODE POOL (taints: platform=true:NoSchedule)           │  │ │
│  │   │ • ArgoCD        • Cilium         • Vault Agent                  │  │ │
│  │   │ • OTel Collector • Prometheus    • Grafana                      │  │ │
│  │   │ • Loki          • Tempo          • Kyverno                      │  │ │
│  │   └─────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                         │ │
│  │   ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │   │ APPLICATION NODE POOL (default, auto-scaling)                   │  │ │
│  │   │ • svc-ledger    • svc-wallet     • svc-merchant                 │  │ │
│  │   │ • svc-giftcard  • svc-notification                              │  │ │
│  │   └─────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    │ VPC Peering                             │
│                                    ▼                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                          AIVEN VPC                                      │ │
│  │  • PostgreSQL (Primary + Read Replica)                                 │ │
│  │  • Kafka Cluster (3 brokers)                                           │ │
│  │  • Valkey Cluster (HA)                                                 │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## **2.3 Domain Services**

| Service | Responsabilité | Pattern | Criticité |
|---------|---------------|---------|-----------|
| **svc-ledger** | Earn/Burn transactions, ACID ledger | Sync REST + gRPC | P0 — Core |
| **svc-wallet** | Balance queries, snapshots | Sync REST + gRPC | P0 — Core |
| **svc-merchant** | Onboarding, configuration | Sync REST | P1 |
| **svc-giftcard** | Catalog, rewards | Sync REST | P1 |
| **svc-notification** | SMS/Email dispatch | Async (Kafka consumer) | P2 |

## **2.4 Data Flow**

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

# 🌿 **PARTIE III — DELIVERY MODEL**

## **3.1 Git Strategy**

**Trunk-Based Development avec Cherry-Pick**

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

| Branche | Usage | Politique |
|---------|-------|-----------|
| `main` | Trunk principal | Tous les PRs mergent ici |
| `maintenance/v*.x.x` | Maintenance versions | Cherry-pick depuis main uniquement |
| `feature/*` | Développement | Short-lived, merge to main |

## **3.2 GitOps Flow (ArgoCD)**

- **ArgoCD centralisé** : Instance unique gérant tous les environnements
- **App-of-Apps pattern** : ApplicationSets avec Git + Matrix generators
- **Sync automatique** : Dev auto-sync, Staging/Prod manual approval

## **3.3 Environments**

| Environment | Account | Cluster | Sync Policy |
|-------------|---------|---------|-------------|
| **dev** | localplus-dev | eks-dev | Auto-sync |
| **staging** | localplus-staging | eks-staging | Manual |
| **prod** | localplus-prod | eks-prod | Manual + Approval |

## **3.4 CI/CD**

→ **Documentation détaillée** : [docs/bootstrap/BOOTSTRAP-GUIDE.md](bootstrap/BOOTSTRAP-GUIDE.md)

---

# 🗂️ **PARTIE IV — REPOSITORY & OWNERSHIP MODEL**

## **4.1 Repository Tiers**

| Tier | Repos | Description | Owner |
|------|-------|-------------|-------|
| **T0 — Foundation** | `bootstrap/` | AWS Landing Zone, Account Factory | Platform Team |
| **T1 — Platform** | `platform-*` | GitOps, Networking, Security, Observability | Platform Team |
| **T2 — Contracts** | `contracts-proto`, `sdk-*` | APIs, SDKs partagés | Platform + Backend |
| **T3 — Domain** | `svc-*` | Services métier | Product Teams |
| **T4 — Quality** | `e2e-scenarios`, `chaos-*` | Tests E2E, Chaos engineering | QA + Platform |
| **T5 — Documentation** | `docs/` | Documentation centralisée | All Teams |

## **4.2 Ownership Matrix**

| Tier | Owner Team | Approvers | Change Process |
|------|------------|-----------|----------------|
| **T0 — Foundation** | Platform | Platform Lead + Security | ADR + RFC obligatoire |
| **T1 — Platform** | Platform | Platform Team (2 reviewers) | ADR si breaking change |
| **T2 — Contracts** | Platform + Backend | Tech Lead | Buf breaking detection |
| **T3 — Domain** | Product Teams | Team Lead | Standard PR review |
| **T4 — Quality** | QA + Platform | QA Lead | Standard PR review |
| **T5 — Documentation** | All | Tech Lead | Standard PR review |

## **4.3 Repository Index**

### Tier 0 — Foundation
| Repo | Description | README |
|------|-------------|--------|
| `bootstrap/` | AWS Landing Zone, Control Tower, Account Factory | → [bootstrap/README.md](../bootstrap/README.md) |

### Tier 1 — Platform
| Repo | Description | README |
|------|-------------|--------|
| `platform-gitops/` | ArgoCD, ApplicationSets | → [platform-gitops/README.md](../platform-gitops/README.md) |
| `platform-networking/` | Cilium, Gateway API | → [platform-networking/README.md](../platform-networking/README.md) |
| `platform-observability/` | OTel, Prometheus, Loki, Tempo, Grafana | → [platform-observability/README.md](../platform-observability/README.md) |
| `platform-security/` | Vault, External-Secrets, Kyverno | → [platform-security/README.md](../platform-security/README.md) |
| `platform-cache/` | Valkey configuration, SDK | → [platform-cache/README.md](../platform-cache/README.md) |
| `platform-gateway/` | APISIX (future), Cloudflare config | → [platform-gateway/README.md](../platform-gateway/README.md) |
| `platform-application-provis/` | Terraform modules (DB, Kafka, Cache, EKS) | → [platform-application-provis/README.md](../platform-application-provis/README.md) |

### Tier 2 — Contracts
| Repo | Description | README |
|------|-------------|--------|
| `contracts-proto/` | Protobuf definitions | → [contracts-proto/README.md](../contracts-proto/README.md) |
| `sdk-python/` | Python SDK (clients, telemetry) | → [sdk-python/README.md](../sdk-python/README.md) |
| `sdk-go/` | Go SDK | → [sdk-go/README.md](../sdk-go/README.md) |

### Tier 3 — Domain Services
| Repo | Description | README |
|------|-------------|--------|
| `svc-ledger/` | Earn/Burn transactions | → [svc-ledger/README.md](../svc-ledger/README.md) |
| `svc-wallet/` | Balance queries | → [svc-wallet/README.md](../svc-wallet/README.md) |
| `svc-merchant/` | Merchant onboarding | → [svc-merchant/README.md](../svc-merchant/README.md) |
| `svc-giftcard/` | Gift card catalog | → [svc-giftcard/README.md](../svc-giftcard/README.md) |
| `svc-notification/` | Notifications (Kafka consumer) | → [svc-notification/README.md](../svc-notification/README.md) |

### Tier 4 — Quality
| Repo | Description | README |
|------|-------------|--------|
| `e2e-scenarios/` | Playwright E2E tests | → [e2e-scenarios/README.md](../e2e-scenarios/README.md) |
| `chaos-experiments/` | Litmus chaos tests | → [chaos-experiments/README.md](../chaos-experiments/README.md) |

---

# 🔐 **PARTIE V — PLATFORM BASELINES**

## **5.1 Security Baseline**

**Defense in Depth** : 6 couches de sécurité

| Layer | Composant | Protection |
|-------|-----------|------------|
| **Edge** | Cloudflare | WAF, DDoS, Bot protection |
| **Gateway** | APISIX (future) | JWT, Rate limiting |
| **Network** | Cilium | NetworkPolicies, default deny |
| **Identity** | IRSA + Vault | Dynamic secrets, mTLS |
| **Workload** | Kyverno | Pod security, image signing |
| **Data** | KMS + Aiven | Encryption at rest/transit |

→ **Documentation détaillée** : [docs/security/SECURITY-ARCHITECTURE.md](security/SECURITY-ARCHITECTURE.md)

## **5.2 Observability Baseline**

| Signal | Outil | Retention | Coût |
|--------|-------|-----------|------|
| **Metrics** | Prometheus + Thanos | 15j local, 1an S3 | ~5€/mois |
| **Logs** | Loki | 30 jours (GDPR) | Self-hosted |
| **Traces** | Tempo | 7 jours | Self-hosted |
| **Profiling** | Pyroscope | 7 jours | Self-hosted |
| **Errors** | Sentry (self-hosted) | 30 jours | Self-hosted |

→ **Documentation détaillée** : [docs/observability/OBSERVABILITY-GUIDE.md](observability/OBSERVABILITY-GUIDE.md)

## **5.3 Networking Baseline**

| Composant | Rôle | Configuration |
|-----------|------|---------------|
| **Cloudflare** | Edge, WAF, Tunnel | Free tier |
| **Cilium** | CNI, mTLS, Gateway API | WireGuard encryption |
| **VPC Peering** | Aiven connectivity | Private, no internet |
| **Route53** | Private DNS, backup | Internal zones |

→ **Documentation détaillée** : [docs/networking/NETWORKING-ARCHITECTURE.md](networking/NETWORKING-ARCHITECTURE.md)

## **5.4 Data Baseline**

| Service | Provider | Plan | Coût estimé |
|---------|----------|------|-------------|
| **PostgreSQL** | Aiven | Business-4 | ~300€/mois |
| **Kafka** | Aiven | Business-4 | ~400€/mois |
| **Valkey** | Aiven | Business-4 | ~150€/mois |

**Règle d'or** : 1 table = 1 owner. Cross-service = gRPC ou Events, jamais JOIN.

→ **Documentation détaillée** : [docs/data/DATA-ARCHITECTURE.md](data/DATA-ARCHITECTURE.md)

---

# ⚡ **PARTIE VI — RESILIENCE & DR**

## **6.1 Failure Modes**

| Failure | Detection | Recovery | RTO |
|---------|-----------|----------|-----|
| Pod crash | Liveness probe | K8s restart | < 30s |
| Node failure | Node NotReady | Pod reschedule | < 2min |
| AZ failure | Multi-AZ detect | Traffic shift | < 5min |
| DB primary failure | Aiven health | Automatic failover | < 5min |
| Kafka broker failure | Aiven health | Automatic rebalance | < 2min |
| Full region failure | Manual | DR procedure | 4h (target) |

## **6.2 Backup Strategy**

| Data | Method | Frequency | Retention |
|------|--------|-----------|-----------|
| PostgreSQL | Aiven automated | Hourly | 7 jours |
| PostgreSQL PITR | Aiven WAL | Continuous | 24h |
| Kafka | Topic retention | N/A | 7 jours |
| Terraform state | S3 versioning | Every apply | 90 jours |

→ **Documentation détaillée** : [docs/resilience/DR-GUIDE.md](resilience/DR-GUIDE.md)

---

# 🛠️ **PARTIE VII — PLATFORM CONTRACTS**

## **7.1 Golden Path (New Service Checklist)**

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

## **7.2 SLI/SLO/Error Budgets**

| Service | SLI | SLO | Error Budget |
|---------|-----|-----|--------------|
| **svc-ledger** | Availability | 99.9% | 43 min/mois |
| **svc-ledger** | Latency P99 | < 200ms | N/A |
| **svc-wallet** | Availability | 99.9% | 43 min/mois |
| **Platform** | Availability | 99.5% | 3.6h/mois |

## **7.3 On-Call Structure**

| Rôle | Responsabilité | Rotation |
|------|---------------|----------|
| **Primary** | First responder, triage | Weekly |
| **Secondary** | Escalation, expertise | Weekly |
| **Incident Commander** | Coordination si P1 | On-demand |

→ **Documentation détaillée** : [docs/platform/PLATFORM-ENGINEERING.md](platform/PLATFORM-ENGINEERING.md)

---

# 🚀 **PARTIE VIII — ROADMAP**

## **8.1 Séquence de Construction**

| Phase | Focus | Estimation |
|-------|-------|------------|
| **1** | Bootstrap Layer 0-1 (IAM, VPC, EKS, Aiven) | 3 semaines |
| **2** | Platform GitOps (ArgoCD) | 1 semaine |
| **3** | Platform Networking (Cilium, Gateway API) | 1 semaine |
| **3b** | Edge & CDN (Cloudflare) | 1 semaine |
| **4** | Platform Security (Vault, Kyverno) | 2 semaines |
| **5** | Platform Observability | 2 semaines |
| **5b** | Platform APM | 1 semaine |
| **6** | Platform Cache (Valkey) | 1 semaine |
| **7** | Contracts (Proto, SDK) | 1 semaine |
| **8** | svc-ledger | 3 semaines |
| **9** | svc-wallet | 2 semaines |
| **10** | Kafka + Outbox | 2 semaines |
| **10b** | Task Queue | 1 semaine |
| **11** | Testing complet | 2 semaines |
| **12** | Compliance audit | 2 semaines |
| **13** | Documentation | 1 semaine |

**Total estimé : ~25 semaines**

## **8.2 Checklist avant démarrage**

### Comptes & Accès
- [ ] Compte AWS créé, billing configuré
- [ ] Compte Aiven créé
- [ ] Compte Cloudflare créé (Free tier)
- [ ] Organisation GitHub créée
- [ ] Domaine DNS acquis et transféré vers Cloudflare

### Décisions validées
- [ ] RPO 1h, RTO 15min
- [ ] AWS eu-west-1
- [ ] Aiven pour Kafka + PostgreSQL + Valkey
- [ ] Cloudflare pour DNS + WAF + CDN
- [ ] Self-hosted observability
- [ ] ArgoCD centralisé
- [ ] Cilium + Gateway API
- [ ] Kyverno
- [ ] HashiCorp Vault self-hosted

---

# 📚 **APPENDIX**

## **A. Glossaire**

→ [docs/GLOSSARY.md](GLOSSARY.md)

## **B. ADR Index**

| ADR | Titre | Statut |
|-----|-------|--------|
| 001 | Modular Monolith First | Accepted |
| 002 | Aiven Managed Data | Accepted |
| 003 | Cilium over Calico | Accepted |
| ... | ... | ... |

→ [docs/adr/](adr/)

## **C. Change Management Process**

### Architecture Changes
1. **ADR Required** : Toute décision impactant >1 service
2. **Review** : Platform Team + Tech Lead
3. **Communication** : Slack #platform-updates

### Breaking Changes
1. RFC obligatoire (`docs/rfc/`)
2. Migration path documenté
3. Annonce 2 sprints avant

### Emergency Changes
1. Incident Commander approval
2. Post-mortem obligatoire
3. ADR rétroactif sous 48h

---

# 📖 **Documentation Index**

| Document | Description | Path |
|----------|-------------|------|
| **Bootstrap Guide** | AWS setup, Account Factory | → [bootstrap/BOOTSTRAP-GUIDE.md](bootstrap/BOOTSTRAP-GUIDE.md) |
| **Security Architecture** | Defense in depth, policies | → [security/SECURITY-ARCHITECTURE.md](security/SECURITY-ARCHITECTURE.md) |
| **Observability Guide** | Metrics, logs, traces, APM | → [observability/OBSERVABILITY-GUIDE.md](observability/OBSERVABILITY-GUIDE.md) |
| **Networking Architecture** | VPC, Cloudflare, Gateway API | → [networking/NETWORKING-ARCHITECTURE.md](networking/NETWORKING-ARCHITECTURE.md) |
| **Data Architecture** | PostgreSQL, Kafka, Cache, Queues | → [data/DATA-ARCHITECTURE.md](data/DATA-ARCHITECTURE.md) |
| **Testing Strategy** | Unit, Integration, E2E, Chaos | → [testing/TESTING-STRATEGY.md](testing/TESTING-STRATEGY.md) |
| **Platform Engineering** | Contracts, Golden Path, On-Call | → [platform/PLATFORM-ENGINEERING.md](platform/PLATFORM-ENGINEERING.md) |
| **DR Guide** | Backup, Recovery, Runbooks | → [resilience/DR-GUIDE.md](resilience/DR-GUIDE.md) |
| **Glossary** | Terminologie | → [GLOSSARY.md](GLOSSARY.md) |

---

*Document maintenu par : Platform Team*  
*Dernière mise à jour : Janvier 2026*

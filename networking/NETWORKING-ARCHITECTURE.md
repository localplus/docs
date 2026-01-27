# 🌐 **Networking Architecture**
## *LOCAL-PLUS VPC, Edge, CDN & Gateway*

---

> **Retour vers** : [Architecture Overview](../EntrepriseArchitecture.md)

---

# 📋 **Table of Contents**

1. [VPC Design](#vpc-design)
2. [Traffic Flow](#traffic-flow)
3. [Gateway API Configuration](#gateway-api-configuration)
4. [Network Policies](#network-policies)
5. [Cloudflare Architecture](#cloudflare-architecture)
6. [DNS Configuration](#dns-configuration)
7. [Route53 — DNS Interne & Backup](#route53--dns-interne--backup)
8. [API Gateway / APIM (Future)](#api-gateway--apim-future)
9. [Multi-Cloud Vision](#multi-cloud-vision)

---

# 🏗️ **VPC Design**

## CIDR Allocation

| CIDR | Usage | Subnets |
|------|-------|---------|
| 10.0.0.0/16 | VPC Principal | - |
| 10.0.0.0/20 | Private Subnets (Workloads) | 3 AZs |
| 10.0.16.0/20 | Private Subnets (Data) | 3 AZs |
| 10.0.32.0/20 | Public Subnets (NAT, LB) | 3 AZs |

## Architecture EKS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                       │
│                           (End Users)                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLOUDFLARE EDGE (Global)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  • DNS (localplus.io)          • WAF (OWASP rules)                         │
│  • DDoS Protection (L3-L7)     • SSL/TLS Termination                       │
│  • CDN (static assets)         • Bot Protection                            │
│  • Cloudflare Tunnel           • Zero Trust Access                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Cloudflare Tunnel (encrypted)
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
│  │  │   │ • ArgoCD, Cilium, Vault, Kyverno, OTel, Grafana        │  │ │ │
│  │  │   └─────────────────────────────────────────────────────────┘  │ │ │
│  │  │                                                                 │ │ │
│  │  │   ┌─────────────────────────────────────────────────────────┐  │ │ │
│  │  │   │ NODE POOL: application (default, auto-scaling)          │  │ │ │
│  │  │   │ Instance: m6i.large (cost-optimized)                    │  │ │ │
│  │  │   ├─────────────────────────────────────────────────────────┤  │ │ │
│  │  │   │ APPLICATION NAMESPACES                                  │  │ │ │
│  │  │   │ • svc-ledger, svc-wallet, svc-merchant, etc.           │  │ │ │
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
│  │  │  • Valkey (Redis-compatible)                                   │ │ │
│  │  └─────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Node Pool Strategy

| Node Pool | Taints | Usage | Instance Type | Scaling |
|-----------|--------|-------|---------------|---------|
| **platform** | `platform=true:NoSchedule` | ArgoCD, Monitoring, Security tools | m6i.xlarge | Fixed (2-3 nodes) |
| **application** | None (default) | Domain services | m6i.large | HPA (2-10 nodes) |
| **spot** (optionnel) | `spot=true:PreferNoSchedule` | Batch jobs, non-critical | m6i.large (spot) | Auto (0-5 nodes) |

---

# 🔄 **Traffic Flow**

| Flow | Path | Encryption |
|------|------|------------|
| Internet → Services | Cloudflare → Tunnel → Cilium Gateway → Pod | TLS + mTLS |
| Service → Service | Pod → Pod (Cilium) | mTLS (WireGuard) |
| Service → Aiven | VPC Peering | TLS |
| Service → AWS (S3, KMS) | VPC Endpoints | TLS |

---

# 🚪 **Gateway API Configuration**

## Resources

| Resource | Purpose |
|----------|---------|
| **GatewayClass** | Cilium implementation |
| **Gateway** | HTTPS listener, TLS termination |
| **HTTPRoute** | Routing vers services (path-based) |

## Gateway Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| **GatewayClass** | `cilium` | Utilise le controller Cilium |
| **Listener** | HTTPS:443 | TLS termination |
| **TLS Mode** | Terminate | Certificat géré par External-Secrets |
| **Allowed Routes** | All namespaces | Services peuvent déclarer leurs routes |

## HTTPRoute Routing

| Pattern | Exemple | Backend |
|---------|---------|---------|
| Path prefix | `/v1/ledger/*` | svc-ledger:8080 |
| Path prefix | `/v1/wallet/*` | svc-wallet:8080 |
| Path prefix | `/v1/merchant/*` | svc-merchant:8080 |
| Exact path | `/health` | Tous les services |

---

# 🔒 **Network Policies**

## Default Deny Strategy

| Policy | Effect |
|--------|--------|
| Default deny all | Aucun trafic sauf explicite |
| Allow intra-namespace | Services même namespace peuvent communiquer |
| Allow specific cross-namespace | svc-ledger → svc-wallet explicite |
| Allow egress Aiven | Services → VPC Peering range only |
| Allow egress AWS endpoints | Services → VPC Endpoints only |

## Cilium Network Policy Rules

### Ingress Rules

| From | To | Port | Protocol |
|------|----|------|----------|
| Gateway (platform) | All services | 8080, 50051 | TCP |
| svc-ledger | svc-wallet | 50051 | gRPC |
| Prometheus | All services | 8080 | metrics |

### Egress Rules

| From | To | Port | Description |
|------|----|------|-------------|
| All services | Aiven PostgreSQL | 5432 | Database |
| All services | Aiven Kafka | 9092 | Messaging |
| All services | Aiven Valkey | 6379 | Cache |
| All services | AWS VPC Endpoints | 443 | S3, KMS, etc. |
| OTel Collector | Tempo, Loki | 4317, 3100 | Telemetry |

---

# ☁️ **Cloudflare Architecture**

## Pourquoi Cloudflare ?

| Critère | Cloudflare | AWS CloudFront + WAF | Verdict |
|---------|------------|---------------------|---------|
| **Coût** | Free tier généreux | Payant dès le début | ✅ Cloudflare |
| **WAF** | Gratuit (règles de base) | ~30€/mois minimum | ✅ Cloudflare |
| **DDoS** | Inclus (unlimited) | AWS Shield Standard gratuit | ≈ Égal |
| **SSL/TLS** | Gratuit, auto-renew | ACM gratuit | ≈ Égal |
| **CDN** | 300+ PoPs, gratuit | Payant au GB | ✅ Cloudflare |
| **DNS** | Gratuit, très rapide | Route53 ~0.50€/zone | ✅ Cloudflare |
| **Zero Trust** | Gratuit jusqu'à 50 users | Cognito + ALB payant | ✅ Cloudflare |

> **Décision :** Cloudflare en front, AWS en backend. Best of both worlds.

## Edge Layers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLOUDFLARE EDGE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  LAYER 1: DNS                                                                │
│  • Authoritative DNS (localplus.io)                                         │
│  • DNSSEC enabled                                                           │
│  • Geo-routing (future multi-region)                                        │
│                                                                              │
│  LAYER 2: DDoS Protection                                                   │
│  • Layer 3/4 DDoS mitigation (automatic, unlimited)                         │
│  • Layer 7 DDoS mitigation                                                  │
│                                                                              │
│  LAYER 3: WAF                                                               │
│  • OWASP Core Ruleset                                                       │
│  • Custom rules (rate limit, geo-block, bot score)                          │
│                                                                              │
│  LAYER 4: SSL/TLS                                                           │
│  • Edge certificates (auto-issued)                                          │
│  • Full (strict) mode → Origin certificate                                  │
│  • TLS 1.3 only, HSTS enabled                                               │
│                                                                              │
│  LAYER 5: CDN & Caching                                                     │
│  • Static assets caching                                                    │
│  • Tiered caching                                                           │
│                                                                              │
│  LAYER 6: Cloudflare Tunnel                                                 │
│  • No public IP needed on origin                                            │
│  • Encrypted tunnel to EKS                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Cloudflare Services

| Service | Plan | Configuration | Coût |
|---------|------|---------------|------|
| **DNS** | Free | Authoritative, DNSSEC, proxy enabled | 0€ |
| **CDN** | Free | Cache everything, tiered caching | 0€ |
| **SSL/TLS** | Free | Full (strict), TLS 1.3, edge certs | 0€ |
| **WAF** | Free | Managed ruleset, 5 custom rules | 0€ |
| **DDoS** | Free | L3/L4/L7 protection, unlimited | 0€ |
| **Bot Management** | Free | Basic bot score, JS challenge | 0€ |
| **Rate Limiting** | Free | 1 rule (10K req/month free) | 0€ |
| **Tunnel** | Free | Unlimited tunnels, cloudflared | 0€ |
| **Access** | Free | Zero Trust, 50 users free | 0€ |

**Coût Cloudflare total : 0€** (Free tier suffisant pour démarrer)

## WAF Rules Strategy

| Rule Set | Type | Action | Purpose |
|----------|------|--------|---------|
| **OWASP Core** | Managed | Block | SQLi, XSS, LFI, RFI protection |
| **Cloudflare Managed** | Managed | Block | Zero-day, emerging threats |
| **Geo-Block** | Custom | Block | Block high-risk countries (optional) |
| **Rate Limit API** | Custom | Challenge | > 100 req/min per IP on /api/* |
| **Bot Score < 30** | Custom | Challenge | Likely bot traffic |

## SSL/TLS Configuration

| Setting | Value | Rationale |
|---------|-------|-----------|
| **SSL Mode** | Full (strict) | Origin has valid cert |
| **Minimum TLS** | 1.2 | PCI-DSS compliance |
| **TLS 1.3** | Enabled | Performance + security |
| **HSTS** | Enabled (max-age=31536000) | Force HTTPS |
| **Always Use HTTPS** | On | Redirect HTTP → HTTPS |
| **Origin Certificate** | Cloudflare Origin CA | 15-year validity, free |

## Cloudflare Tunnel

| Composant | Rôle | Déploiement |
|-----------|------|-------------|
| **cloudflared daemon** | Agent tunnel | 2+ replicas, namespace platform |
| **Tunnel credentials** | Secret d'authentification | Vault / External-Secrets |
| **Tunnel config** | Routing rules | ConfigMap |
| **Health checks** | Vérification disponibilité | Cloudflare dashboard |

**Avantages :**
- Pas d'IP publique exposée sur l'origin
- Connexion outbound uniquement (pas de firewall inbound)
- Encryption de bout en bout
- Failover automatique entre replicas

## Cloudflare Access (Zero Trust)

| Resource | Policy | Authentication |
|----------|--------|----------------|
| **grafana.localplus.io** | Team only | GitHub SSO |
| **argocd.localplus.io** | Team only | GitHub SSO |
| **api.localplus.io/admin** | Admin only | GitHub SSO + MFA |
| **api.localplus.io/*** | Public | No auth (application handles) |

---

# 🌍 **DNS Configuration**

## DNS Records — localplus.io

| Type | Name | Content | Proxy | TTL |
|------|------|---------|-------|-----|
| A | @ | Cloudflare Tunnel | ☁️ ON | Auto |
| CNAME | www | @ | ☁️ ON | Auto |
| CNAME | api | tunnel-xxx.cfargotunnel.com | ☁️ ON | Auto |
| CNAME | grafana | tunnel-xxx.cfargotunnel.com | ☁️ ON | Auto |
| CNAME | argocd | tunnel-xxx.cfargotunnel.com | ☁️ ON | Auto |
| TXT | @ | SPF record | ☁️ OFF | Auto |
| TXT | _dmarc | DMARC policy | ☁️ OFF | Auto |
| MX | @ | Mail provider | ☁️ OFF | Auto |

---

# 🛣️ **Route53 — DNS Interne & Backup**

| Use Case | Solution | Configuration |
|----------|----------|---------------|
| **DNS Public (Primary)** | Cloudflare | Authoritative pour `localplus.io` |
| **DNS Public (Backup)** | Route53 | Secondary zone, sync via AXFR |
| **DNS Privé (Internal)** | Route53 Private Hosted Zones | `*.internal.localplus.io` |
| **Service Discovery** | Route53 + Cloud Map | Résolution services internes |
| **Health Checks** | Route53 Health Checks | Failover automatique |

## Architecture DNS Hybride

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DNS ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  EXTERNAL TRAFFIC                          INTERNAL TRAFFIC                  │
│                                                                              │
│  ┌─────────────────┐                       ┌─────────────────┐              │
│  │ Cloudflare DNS  │                       │ Route53 Private │              │
│  │  (Primary)      │                       │ Hosted Zone     │              │
│  │                 │                       │                 │              │
│  │ localplus.io    │                       │ internal.       │              │
│  │ api.localplus.io│                       │ localplus.io    │              │
│  └────────┬────────┘                       └────────┬────────┘              │
│           │                                         │                        │
│           │ Failover                                │ VPC DNS                │
│           ▼                                         ▼                        │
│  ┌─────────────────┐                       ┌─────────────────┐              │
│  │ Route53 Public  │                       │ EKS CoreDNS     │              │
│  │  (Backup)       │                       │ + Cloud Map     │              │
│  │ Health checks   │                       │ svc-*.svc.      │              │
│  │ Failover ready  │                       │ cluster.local   │              │
│  └─────────────────┘                       └─────────────────┘              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Route53 Features

| Feature | Use Case Local-Plus |
|---------|---------------------|
| **Private Hosted Zones** | Résolution DNS interne VPC |
| **Health Checks** | Failover automatique |
| **Alias Records** | Pointage vers ALB/NLB |
| **Geolocation Routing** | Future multi-région |
| **Failover Routing** | Backup si Cloudflare down |
| **Weighted Routing** | Canary deployments |

---

# 🚪 **API Gateway / APIM (Future)**

> **Statut :** À définir ultérieurement. Pour le moment : Cloudflare → Cilium Gateway → Services.

## Options à évaluer

| Solution | Type | Coût | Notes |
|----------|------|------|-------|
| **AWS API Gateway** | Managed | Pay-per-use | Simple, intégré AWS |
| **Gravitee CE** | APIM complet | Gratuit | Portal, Subscriptions inclus |
| **Kong OSS** | Gateway | Gratuit | Populaire, plugins riches |
| **APISIX** | Gateway | Gratuit | Cloud-native, performant |

**Décision reportée à Phase 2+ selon les besoins :**
- Si besoin B2B/Partners → APIM (Gravitee)
- Si juste rate limiting/auth → AWS API Gateway
- Si multi-cloud requis → APISIX ou Kong

## Architecture Actuelle (Phase 1)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE SIMPLIFIÉE — PHASE 1                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Internet                                                                    │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ CLOUDFLARE (DNS, WAF, DDoS, TLS)                                    │    │
│  └──────────────────────────────┬──────────────────────────────────────┘    │
│                                 │                                            │
│                                 │ Tunnel                                     │
│                                 ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ AWS EKS — Cilium Gateway API (Routing interne, mTLS)                │    │
│  │                                                                      │    │
│  │  Services : svc-ledger, svc-wallet, svc-merchant, ...               │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Pas d'API Gateway dédié pour le moment — Cilium Gateway API suffit.       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# 🌍 **Multi-Cloud Vision**

> **Objectif :** L'architecture edge (Cloudflare) est **cloud-agnostic** et peut router vers plusieurs cloud providers.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MULTI-CLOUD ARCHITECTURE (Future)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         CLOUDFLARE EDGE                                      │
│                    (Global Load Balancing)                                   │
│                              │                                               │
│              ┌───────────────┼───────────────┐                              │
│              │               │               │                              │
│              ▼               ▼               ▼                              │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                   │
│  │  AWS (Primary)│  │  GCP (Future) │  │ Azure (Future)│                   │
│  │  eu-west-1    │  │  europe-west1 │  │ westeurope    │                   │
│  │               │  │               │  │               │                   │
│  │  Gateway +    │  │  Gateway +    │  │  Gateway +    │                   │
│  │  Services     │  │  Services     │  │  Services     │                   │
│  └───────────────┘  └───────────────┘  └───────────────┘                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    AIVEN (Multi-Cloud Data Layer)                   │    │
│  │  • PostgreSQL avec réplication cross-cloud                         │    │
│  │  • Kafka avec MirrorMaker cross-cloud                              │    │
│  │  • Valkey avec réplication                                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Multi-Cloud Readiness

| Composant | Multi-Cloud Ready | Comment |
|-----------|-------------------|---------|
| **Cloudflare** | ✅ Oui | Load balancing global, health checks multi-origin |
| **APISIX** | ✅ Oui | Déployable sur tout K8s (EKS, GKE, AKS) |
| **Aiven** | ✅ Oui | PostgreSQL, Kafka, Valkey disponibles sur AWS/GCP/Azure |
| **ArgoCD** | ✅ Oui | Peut gérer des clusters multi-cloud |
| **Vault** | ✅ Oui | Réplication cross-datacenter |
| **OTel** | ✅ Oui | Standard ouvert, backends interchangeables |

## Phases Multi-Cloud

| Phase | Scope | Timeline |
|-------|-------|----------|
| **Phase 1 (Actuelle)** | AWS uniquement, architecture cloud-agnostic | Now |
| **Phase 2** | DR sur GCP (read replicas, failover) | +12 mois |
| **Phase 3** | Active-Active multi-cloud | +24 mois |

---

*Document maintenu par : Platform Team*  
*Dernière mise à jour : Janvier 2026*

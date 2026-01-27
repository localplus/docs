# 🔐 **Security Architecture**
## *LOCAL-PLUS Defense in Depth*

---

> **Retour vers** : [Architecture Overview](../EntrepriseArchitecture.md)

---

# 📋 **Table of Contents**

1. [Defense in Depth](#defense-in-depth)
2. [Layer 0 — Edge (Cloudflare)](#layer-0--edge-cloudflare)
3. [Layer 1 — API Gateway](#layer-1--api-gateway)
4. [Layer 2 — Network](#layer-2--network)
5. [Layer 3 — Identity & Access](#layer-3--identity--access)
6. [Layer 4 — Workload](#layer-4--workload)
7. [Layer 5 — Data](#layer-5--data)
8. [Supply Chain Security](#supply-chain-security)
9. [Security Roadmap](#security-roadmap)

---

# 🛡️ **Defense in Depth**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAYER 0: EDGE (Cloudflare)                                                  │
│ • Cloudflare WAF (OWASP Core Ruleset, custom rules)                        │
│ • Cloudflare DDoS Protection (L3/L4/L7, unlimited)                         │
│ • Bot Management (JS challenge, CAPTCHA)                                   │
│ • TLS 1.3 termination, HSTS enforced                                       │
│ • Cloudflare Tunnel (no public origin IP)                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAYER 1: API GATEWAY (Cilium Gateway API)                                   │
│ • JWT/API Key validation                                                   │
│ • Rate limiting (fine-grained, per user/tenant)                            │
│ • Request validation                                                       │
│ • Circuit breaker                                                          │
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
│ • IRSA / Workload Identity (no static credentials)                         │
│ • Cilium mTLS (WireGuard) — pod-to-pod encryption                          │
│ • Vault dynamic secrets — DB credentials rotated                           │
│ • PAM — Privileged Access Management                                       │
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

---

# 🌐 **Layer 0 — Edge (Cloudflare)**

## Protection Components

| Component | Protection | Configuration |
|-----------|------------|---------------|
| **WAF** | OWASP Core Ruleset | Managed + Custom rules |
| **DDoS** | L3/L4/L7 mitigation | Unlimited, automatic |
| **Bot Protection** | JS challenge, CAPTCHA | Bot score threshold |
| **TLS** | 1.3 only, HSTS | Full (strict) mode |
| **Tunnel** | No public origin IP | Encrypted connection |

## WAF Rules Strategy

| Rule Set | Type | Action | Purpose |
|----------|------|--------|---------|
| **OWASP Core** | Managed | Block | SQLi, XSS, LFI, RFI protection |
| **Cloudflare Managed** | Managed | Block | Zero-day, emerging threats |
| **Geo-Block** | Custom | Block | Block high-risk countries (optional) |
| **Rate Limit API** | Custom | Challenge | > 100 req/min per IP on /api/* |
| **Bot Score < 30** | Custom | Challenge | Likely bot traffic |

---

# 🚪 **Layer 1 — API Gateway**

## Cilium Gateway API (Phase 1)

| Feature | Configuration | Purpose |
|---------|---------------|---------|
| **TLS Termination** | Cloudflare Origin cert | Encryption |
| **Path-based routing** | HTTPRoute resources | Traffic routing |
| **mTLS** | Cilium automatic | Service authentication |

## APISIX (Future Phase 2+)

| Feature | Configuration | Purpose |
|---------|---------------|---------|
| **JWT Validation** | RS256, JWKS endpoint | Authentication |
| **API Key** | Header-based | Partner authentication |
| **Rate Limiting** | Per user/tenant | Abuse prevention |
| **Request Validation** | JSON Schema | Input validation |
| **Circuit Breaker** | Timeout + failure threshold | Resilience |

---

# 🔒 **Layer 2 — Network**

## VPC Isolation

| Subnet Type | CIDR | Usage | Internet Access |
|-------------|------|-------|-----------------|
| **Private (Workloads)** | 10.0.0.0/20 | EKS nodes, pods | NAT Gateway only |
| **Private (Data)** | 10.0.16.0/20 | VPC Endpoints | None |
| **Public** | 10.0.32.0/20 | NAT Gateway, LB | Direct |

## Cilium Network Policies

| Policy | Effect |
|--------|--------|
| Default deny all | Aucun trafic sauf explicite |
| Allow intra-namespace | Services même namespace peuvent communiquer |
| Allow specific cross-namespace | svc-ledger → svc-wallet explicite |
| Allow egress Aiven | Services → VPC Peering range only |
| Allow egress AWS endpoints | Services → VPC Endpoints only |

---

# 🔑 **Layer 3 — Identity & Access**

## Vue d'ensemble — Modèle Zero Static Credentials

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    IDENTITY & ACCESS ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    WORKLOADS (Pods)                                 │    │
│  │                                                                     │    │
│  │  Pod svc-ledger                 Pod svc-wallet                     │    │
│  │  ├── ServiceAccount             ├── ServiceAccount                 │    │
│  │  └── JWT Token (auto)           └── JWT Token (auto)               │    │
│  │                                                                     │    │
│  └───────────────────────────┬─────────────────────────────────────────┘    │
│                              │                                               │
│          ┌───────────────────┼───────────────────┐                          │
│          │                   │                   │                          │
│          ▼                   ▼                   ▼                          │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                    │
│  │    IRSA      │   │    Vault     │   │   Cilium     │                    │
│  │ (AWS Access) │   │  (Secrets)   │   │   (mTLS)     │                    │
│  └──────┬───────┘   └──────┬───────┘   └──────────────┘                    │
│         │                  │                                                │
│         │ AssumeRole       │ Dynamic creds                                  │
│         ▼                  ▼                                                │
│  ┌──────────────┐   ┌──────────────┐                                       │
│  │   AWS IAM    │   │  PostgreSQL  │                                       │
│  │  S3, KMS...  │   │    Kafka     │                                       │
│  └──────────────┘   └──────────────┘                                       │
│                                                                              │
│  ZERO STATIC CREDENTIALS — Tout est éphémère                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## IRSA — IAM Roles for Service Accounts

### Comment ça marche ?

**IRSA** permet à un pod Kubernetes d'assumer un rôle IAM AWS **sans credentials statiques**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         IRSA — FLOW                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. POD DÉMARRE                                                             │
│     ├── Kubernetes injecte un JWT token (ServiceAccount)                   │
│     └── Le token contient: namespace, service account, issuer              │
│                                                                              │
│  2. POD VEUT ACCÉDER À S3                                                   │
│     ├── AWS SDK détecte le token IRSA                                      │
│     └── SDK appelle STS AssumeRoleWithWebIdentity                          │
│                                                                              │
│  3. AWS STS VALIDE                                                          │
│     ├── Vérifie le JWT via OIDC Provider (EKS)                             │
│     ├── Vérifie que le ServiceAccount match le Trust Policy                │
│     └── Retourne des credentials temporaires (15min-12h)                   │
│                                                                              │
│  4. POD ACCÈDE À S3                                                         │
│     ├── Utilise les credentials temporaires                                │
│     └── AWS SDK renouvelle automatiquement                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Configuration

| Composant | Configuration |
|-----------|---------------|
| **OIDC Provider** | Créé automatiquement avec EKS, URL: `oidc.eks.region.amazonaws.com/id/CLUSTER_ID` |
| **IAM Role** | Trust policy qui autorise le ServiceAccount spécifique |
| **ServiceAccount** | Annotation `eks.amazonaws.com/role-arn` |
| **Pod** | Utilise le ServiceAccount, reçoit le token automatiquement |

### Trust Policy — Principe

| Élément | Description |
|---------|-------------|
| **Principal** | `arn:aws:iam::ACCOUNT:oidc-provider/oidc.eks...` |
| **Condition** | `sub` = `system:serviceaccount:NAMESPACE:SA_NAME` |
| **Action** | `sts:AssumeRoleWithWebIdentity` |

### Mapping Services → Roles

| Service Account | IAM Role | Permissions AWS |
|-----------------|----------|-----------------|
| `svc-ledger` | `role-svc-ledger` | S3 read (specific bucket), KMS decrypt |
| `svc-notification` | `role-svc-notification` | SES send email |
| `external-secrets` | `role-external-secrets` | Secrets Manager read |
| `otel-collector` | `role-otel-collector` | CloudWatch Logs write |

---

## Workload Identity Federation — Concept Général

> **IRSA est une implémentation AWS de Workload Identity Federation.**

### Qu'est-ce que Workload Identity Federation ?

| Concept | Description |
|---------|-------------|
| **Définition** | Mécanisme permettant à une workload (pod, VM, CI job) d'obtenir des credentials cloud **sans secret statique** |
| **Principe** | Le workload prouve son identité via un token (JWT), le cloud provider échange contre des credentials temporaires |
| **Standard** | OIDC (OpenID Connect) — standard ouvert |

### Implémentations par Cloud

| Cloud | Nom | Comment ça marche |
|-------|-----|-------------------|
| **AWS** | IRSA (EKS) | Pod → JWT → STS AssumeRoleWithWebIdentity → IAM Role |
| **GCP** | Workload Identity | Pod → JWT → GCP Token Service → Service Account |
| **Azure** | Workload Identity | Pod → JWT → Azure AD → Managed Identity |
| **Multi-cloud** | SPIRE/SPIFFE | Standard open-source, fédération cross-cloud |

### Pourquoi c'est mieux que les credentials statiques ?

| Critère | Credentials Statiques | Workload Identity |
|---------|----------------------|-------------------|
| **Rotation** | Manuelle, risquée | Automatique (15min-12h) |
| **Blast radius** | Si leak → accès permanent | Si leak → expire rapidement |
| **Audit** | Difficile à tracer | Chaque assume est loggé |
| **Gestion** | Secrets à distribuer | Zero secret management |
| **Compliance** | SOC2/PCI problématique | SOC2/PCI friendly |

---

## Vault — Dynamic Secrets

### Comment Vault génère des credentials dynamiques

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      VAULT DYNAMIC SECRETS                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. POD DEMANDE UN SECRET                                                   │
│     ├── Pod s'authentifie à Vault (Kubernetes auth)                        │
│     └── Vault vérifie le ServiceAccount JWT                                │
│                                                                              │
│  2. VAULT GÉNÈRE LES CREDENTIALS                                            │
│     ├── Vault se connecte à PostgreSQL                                     │
│     ├── CREATE ROLE "svc-ledger-abc123" WITH PASSWORD '...' VALID UNTIL... │
│     └── Retourne username/password au pod                                  │
│                                                                              │
│  3. POD UTILISE LES CREDENTIALS                                             │
│     ├── Connexion à PostgreSQL                                             │
│     └── TTL: 1 heure (renouvelable)                                        │
│                                                                              │
│  4. EXPIRATION                                                              │
│     ├── Vault révoque automatiquement                                      │
│     └── PostgreSQL: DROP ROLE "svc-ledger-abc123"                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Auth Methods

| Method | Use Case | Identity Source |
|--------|----------|-----------------|
| **Kubernetes** | Pods EKS | ServiceAccount JWT |
| **AWS IAM** | Lambda, EC2 | Instance metadata |
| **AppRole** | CI/CD | Role ID + Secret ID |
| **OIDC** | GitHub Actions | GitHub JWT |

### Secret Engines

| Engine | Path | Purpose | TTL |
|--------|------|---------|-----|
| **Database** | `database/` | PostgreSQL dynamic credentials | 1h (renouvelable) |
| **KV v2** | `secret/` | Static secrets (API keys externes) | N/A |
| **Transit** | `transit/` | Encryption as a service | N/A |
| **PKI** | `pki/` | Certificats TLS | 24h |

---

## PAM — Privileged Access Management

### Pourquoi PAM ?

| Problème | Solution PAM |
|----------|--------------|
| SSH keys partagées | Accès éphémère, certificat SSH signé |
| Admin accounts permanents | Just-in-Time access |
| Pas d'audit | Session recording, audit complet |
| Blast radius élevé | Least privilege, time-bound |

### Architecture PAM

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PAM ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  UTILISATEUR VEUT ACCÉDER À UN SYSTÈME                                      │
│                                                                              │
│  ┌──────────────┐                                                           │
│  │  Engineer    │                                                           │
│  │  (Browser)   │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                    │
│         │ 1. Request access                                                  │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    PAM SOLUTION                                      │   │
│  │  • Teleport (open-source) ou                                        │   │
│  │  • HashiCorp Boundary ou                                            │   │
│  │  • AWS SSM Session Manager                                          │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │                                                                      │   │
│  │  2. AUTHENTICATION                                                   │   │
│  │     ├── SSO (GitHub, Okta, Google)                                  │   │
│  │     └── MFA required                                                │   │
│  │                                                                      │   │
│  │  3. AUTHORIZATION                                                    │   │
│  │     ├── Check RBAC (role-based)                                     │   │
│  │     ├── Check time restrictions                                     │   │
│  │     └── Approval workflow (si P1 incident)                          │   │
│  │                                                                      │   │
│  │  4. CREDENTIAL VENDING                                               │   │
│  │     ├── Generate short-lived SSH cert (10min-8h)                    │   │
│  │     └── Or create temporary DB user                                 │   │
│  │                                                                      │   │
│  │  5. SESSION                                                          │   │
│  │     ├── Proxied connection                                          │   │
│  │     └── Full session recording (audit)                              │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         │ 6. Access granted (time-limited)                                   │
│         ▼                                                                    │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │  EKS Node    │     │  Database    │     │  Bastion     │                │
│  │  (kubectl)   │     │  (psql)      │     │  (SSH)       │                │
│  └──────────────┘     └──────────────┘     └──────────────┘                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Options PAM pour Local-Plus

| Solution | Type | Coût | Features |
|----------|------|------|----------|
| **AWS SSM Session Manager** | Managed | Gratuit | SSH/RDP sans bastion, audit CloudTrail |
| **Teleport** | Open-source | Gratuit (Community) | SSH, K8s, DB, session recording |
| **HashiCorp Boundary** | Open-source | Gratuit (Community) | Session brokering, Vault integration |

### Recommandation Phase 1

| Access Type | Solution | Justification |
|-------------|----------|---------------|
| **EKS kubectl** | IRSA + AWS SSO | Native, zero config |
| **Database** | Vault dynamic creds | Already planned |
| **SSH nodes** | SSM Session Manager | Gratuit, no bastion needed |
| **Emergency access** | Break-glass with MFA | Documented procedure |

---

## mTLS — Cilium WireGuard

| Aspect | Configuration |
|--------|---------------|
| **Activation** | Automatique avec Cilium |
| **Protocol** | WireGuard (kernel-level) |
| **Certificate management** | Géré par Cilium |
| **Application changes** | Aucun — transparent |
| **Performance** | Minimal overhead (kernel crypto) |

---

# 🛡️ **Layer 4 — Workload**

## Kyverno Policies

| Policy | Effect | Enforcement |
|--------|--------|-------------|
| `require-labels` | Pods must have required labels | Enforce |
| `require-probes` | Liveness + Readiness required | Enforce |
| `require-resource-limits` | CPU/Memory limits required | Enforce |
| `restrict-privileged` | No privileged containers | Enforce |
| `require-image-signature` | Cosign signature required | Enforce |
| `mutate-default-sa` | Auto-mount SA token disabled | Enforce |

## Container Security Settings

| Setting | Value | Rationale |
|---------|-------|-----------|
| `runAsNonRoot` | true | Prevent root execution |
| `readOnlyRootFilesystem` | true | Prevent filesystem writes |
| `allowPrivilegeEscalation` | false | Prevent privilege escalation |
| `capabilities.drop` | ALL | Minimal capabilities |

---

# 💾 **Layer 5 — Data**

## Encryption

| Data State | Method | Key Management |
|------------|--------|----------------|
| **At rest (PostgreSQL)** | AES-256 | Aiven managed |
| **At rest (Kafka)** | AES-256 | Aiven managed |
| **At rest (S3)** | AES-256 | AWS KMS |
| **In transit** | TLS 1.3 + mTLS | Cilium + Aiven |

## PII Protection

| Data Type | Protection | Implementation |
|-----------|------------|----------------|
| **User ID** | Anonymized in logs | OTel processor |
| **Email** | Masked in logs | OTel processor |
| **PAN** | Never stored | Application validation |
| **IP Address** | Hashed in logs | OTel processor |

## Audit Trail

| Source | Destination | Retention | Immutability |
|--------|-------------|-----------|--------------|
| **AWS CloudTrail** | S3 (Log Archive) | 1 year | S3 Object Lock |
| **K8s Audit Logs** | CloudWatch Logs | 90 days | CloudWatch retention |
| **Application Audit** | PostgreSQL | 1 year | Append-only table |

---

# 🔗 **Supply Chain Security**

## Image Signing — Cosign

| Étape | Description |
|-------|-------------|
| **Build** | CI build l'image Docker |
| **Sign** | Cosign signe l'image avec une clé privée |
| **Push** | Image + signature pushées vers registry |
| **Deploy** | Kyverno vérifie la signature avant d'admettre le pod |
| **Reject** | Si signature invalide → pod refusé |

## SBOM — Software Bill of Materials

| Étape | Outil | Output |
|-------|-------|--------|
| **Generate** | Syft | SBOM en format SPDX-JSON |
| **Attach** | Cosign | SBOM attaché à l'image |
| **Scan** | Grype | Vulnérabilités dans les dépendances |
| **Policy** | Kyverno | Reject si vulnérabilités critiques |

---

# 📅 **Security Roadmap**

## Phase 1 — Day 1 (Current)

| Component | Status | Effort |
|-----------|--------|--------|
| Cilium mTLS | ✅ Zero config | Included |
| IRSA (Workload Identity) | ✅ Ready | 1 day |
| Kyverno basic policies | ✅ Ready | 2 days |
| Vault for secrets | ✅ Ready | 1 week |
| External-Secrets Operator | ✅ Ready | 2 days |
| SSM Session Manager | ✅ Ready | 1 day |

## Phase 2 — Month 3

| Component | Status | Effort |
|-----------|--------|--------|
| Image signing (Cosign) | 🔜 Planned | 1 week |
| SBOM generation (Syft) | 🔜 Planned | 2 days |
| Supply chain verification | 🔜 Planned | 1 week |
| Teleport (full PAM) | 🔜 Evaluation | 1 week |

## Phase 3 — Month 6

| Component | Status | Effort |
|-----------|--------|--------|
| SPIRE (if multi-cluster) | 📋 Evaluation | TBD |
| Confidential Computing | 📋 Evaluation | TBD |

---

*Document maintenu par : Platform Team + Security Team*  
*Dernière mise à jour : Janvier 2026*

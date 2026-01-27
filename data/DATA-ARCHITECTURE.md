# 💾 **Data Architecture**
## *LOCAL-PLUS Database, Kafka, Cache & Queues*

---

> **Retour vers** : [Architecture Overview](../EntrepriseArchitecture.md)

---

# 📋 **Table of Contents**

1. [Aiven Configuration](#aiven-configuration)
2. [Database Strategy](#database-strategy)
3. [Schema Ownership](#schema-ownership)
4. [Kafka Topics](#kafka-topics)
5. [Kafka Monitoring](#kafka-monitoring)
6. [Cache Architecture (Valkey)](#cache-architecture-valkey)
7. [Queueing & Background Jobs](#queueing--background-jobs)

---

# 🗄️ **Aiven Configuration**

## Services Overview

| Service | Plan | Config | Coût estimé |
|---------|------|--------|-------------|
| **PostgreSQL** | Business-4 | Primary + Read Replica, 100GB | ~300€/mois |
| **Kafka** | Business-4 | 3 brokers, 100GB retention | ~400€/mois |
| **Valkey (Redis)** | Business-4 | 2 nodes, 10GB, HA | ~150€/mois |

**Coût total Aiven estimé : ~850€/mois**

---

# 🐘 **Database Strategy**

## Configuration

| Aspect | Choix | Rationale |
|--------|-------|-----------|
| **Replication** | Aiven managed (async) | RPO 1h acceptable |
| **Backup** | Aiven automated hourly | RPO 1h |
| **Failover** | Aiven automated | RTO < 15min |
| **Connection** | VPC Peering (private) | PCI-DSS, no public internet |
| **Pooling** | PgBouncer (Aiven built-in) | Connection efficiency |

## Connection Best Practices

| Paramètre | Valeur recommandée | Rationale |
|-----------|-------------------|-----------|
| **pool_size** | 20 | Nombre de connexions par pod |
| **max_overflow** | 10 | Connexions supplémentaires en pic |
| **pool_timeout** | 30s | Attente max pour une connexion |
| **pool_recycle** | 1800s | Recycler connexions toutes les 30min |
| **ssl** | require | Obligatoire pour PCI-DSS |

---

# 📊 **Schema Ownership**

| Table | Owner Service | Access pattern |
|-------|---------------|----------------|
| `transactions` | svc-ledger | CRUD |
| `ledger_entries` | svc-ledger | CRUD |
| `wallets` | svc-wallet | CRUD |
| `balance_snapshots` | svc-wallet | CRUD |
| `merchants` | svc-merchant | CRUD |
| `giftcards` | svc-giftcard | CRUD |

**Règle d'or : 1 table = 1 owner. Cross-service = gRPC ou Events, jamais JOIN.**

---

# 📨 **Kafka Topics**

## Topic Configuration

| Topic | Producer | Consumers | Retention |
|-------|----------|-----------|-----------|
| `ledger.transactions.v1` | svc-ledger (Outbox) | svc-notification, svc-analytics | 7 jours |
| `wallet.balance-updated.v1` | svc-wallet | svc-analytics | 7 jours |
| `merchant.onboarded.v1` | svc-merchant | svc-notification | 7 jours |

## Outbox Pattern avec Debezium

> **Implementation** : On utilise **Debezium** avec **PostgreSQL Logical Replication** (publication + replication slot), pas le polling.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OUTBOX PATTERN (Debezium CDC)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Application writes to DB + Outbox table in same transaction             │
│  2. Debezium reads WAL via replication slot                                 │
│  3. Events published to Kafka                                               │
│  4. Consumers process events                                                │
│                                                                              │
│  ┌─────────┐    ┌─────────────┐    ┌──────────┐    ┌─────────────┐         │
│  │ svc-*   │───►│ PostgreSQL  │───►│ Debezium │───►│   Kafka     │         │
│  │         │    │ (WAL/Slot)  │    │  (CDC)   │    │             │         │
│  └─────────┘    └─────────────┘    └──────────┘    └──────┬──────┘         │
│                                                           │                 │
│                 Publication + Replication Slot            ▼                 │
│                                                  ┌─────────────────┐        │
│                                                  │   Consumers     │        │
│                                                  └─────────────────┘        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Debezium Configuration

| Composant | Description |
|-----------|-------------|
| **Publication** | `CREATE PUBLICATION outbox_pub FOR TABLE outbox;` |
| **Replication Slot** | Créé automatiquement par Debezium |
| **Connector** | Debezium PostgreSQL Connector |
| **Output** | Kafka topic par table (ou SMT pour routing) |

## Outbox Table Structure

| Colonne | Type | Description |
|---------|------|-------------|
| `id` | UUID | Primary key |
| `aggregate_type` | VARCHAR(255) | Type d'entité (Transaction, Wallet...) |
| `aggregate_id` | VARCHAR(255) | ID de l'entité |
| `event_type` | VARCHAR(255) | Type d'événement |
| `payload` | JSONB | Données de l'événement |
| `created_at` | TIMESTAMPTZ | Timestamp création |

---

# 📊 **Kafka Monitoring**

## Métriques Essentielles

| Métrique | Description | Seuil Alerte | Sévérité |
|----------|-------------|--------------|----------|
| **Consumer Lag** | Messages non traités | > 1000 | P2 |
| **Partition Lag** | Lag par partition | > 500 | P3 |
| **Under-replicated Partitions** | Partitions sans réplicas | > 0 | P1 |
| **Active Controller Count** | Controllers actifs | ≠ 1 | P1 |
| **Offline Partitions** | Partitions inaccessibles | > 0 | P1 |
| **Bytes In/Out Rate** | Débit Kafka | Anomalie > 50% | P3 |
| **Request Latency P99** | Latence requêtes | > 100ms | P2 |
| **ISR Shrink Rate** | Réduction In-Sync Replicas | > 0/min sustained | P2 |

## Consumer Lag Monitoring

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CONSUMER LAG                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Producer Offset:     1000  ────────────────────────────────►               │
│  Consumer Offset:      800  ──────────────────────►                         │
│                              │◄───── LAG = 200 ─────►│                      │
│                                                                              │
│  LAG = Producer Offset - Consumer Offset                                    │
│                                                                              │
│  Causes de Lag élevé:                                                       │
│  • Consumer lent (processing time)                                          │
│  • Consumer crashé                                                          │
│  • Pic de trafic                                                            │
│  • Problème de partition rebalancing                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Dashboard Kafka Recommandé

| Panel | Métrique | Type |
|-------|----------|------|
| **Total Consumer Lag** | `kafka_consumergroup_lag` | Gauge |
| **Lag par Consumer Group** | `kafka_consumergroup_lag` by group | Gauge |
| **Messages In/sec** | `kafka_server_brokertopicmetrics_messagesin_total` | Counter → Rate |
| **Bytes In/Out** | `kafka_server_brokertopicmetrics_bytesin_total` | Counter → Rate |
| **Request Latency** | `kafka_network_requestmetrics_requestqueuetimems` | Histogram |
| **Partition Count** | `kafka_server_replicamanager_partitioncount` | Gauge |
| **Under-replicated** | `kafka_server_replicamanager_underreplicatedpartitions` | Gauge |

---

# 🚀 **Cache Architecture (Valkey)**

## Stack Cache

| Composant | Outil | Hébergement | Coût estimé |
|-----------|-------|-------------|-------------|
| **Cache primaire** | Valkey (Redis-compatible) | Aiven for Caching | ~150€/mois |
| **Cache local (L1)** | Python `cachetools` / Go `bigcache` | In-memory | 0€ |

> **Note :** Valkey est le fork open-source de Redis, maintenu par la Linux Foundation. Aiven supporte Valkey nativement.

## Cache Topology

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

## Cache Strategies par Use Case

| Use Case | Strategy | TTL | Invalidation |
|----------|----------|-----|--------------|
| **Wallet Balance** | Cache-aside (read) | 30s | Event-driven (Kafka) |
| **Merchant Config** | Read-through | 5min | TTL + Manual |
| **Rate Limiting** | Write-through | Sliding window | Auto-expire |
| **Session Data** | Write-through | 24h | Explicit logout |
| **Gift Card Catalog** | Cache-aside | 15min | Event-driven |
| **Feature Flags** | Read-through | 1min | Config push |

## Cache Patterns

### Cache-Aside Pattern

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
```

### Write-Through Pattern

```
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

## Cache Invalidation Strategy

| Trigger | Méthode | Use Case |
|---------|---------|----------|
| **TTL Expiry** | Automatic | Default pour toutes les clés |
| **Event-driven** | Kafka consumer | Wallet balance après transaction |
| **Explicit Delete** | API call | Admin actions, config updates |
| **Pub/Sub** | Valkey PUBLISH | Real-time invalidation cross-pods |

## Cache Key Naming Convention

```
{service}:{entity}:{id}:{version}

Exemples:
  wallet:balance:user_123:v1
  merchant:config:merchant_456:v1
  giftcard:catalog:category_active:v1
  ratelimit:api:user_123:minute
  session:auth:session_abc123
```

## Cache Metrics & Monitoring

| Metric | Seuil alerte | Action |
|--------|--------------|--------|
| **Hit Rate** | < 80% | Revoir TTL, préchargement |
| **Latency P99** | > 10ms | Check network, cluster size |
| **Memory Usage** | > 80% | Eviction analysis, scale up |
| **Evictions/sec** | > 100 | Augmenter cache size |
| **Connection Errors** | > 0 | Check connectivity, pooling |

---

# 📋 **Queueing & Background Jobs**

## Architecture Overview

> **Clarification** : La Task Queue est **interne** aux services, pas en frontal comme RabbitMQ.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TASK QUEUE vs MESSAGE BROKER (RabbitMQ)                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ❌ Pattern RabbitMQ (frontal) - PAS ce qu'on fait:                         │
│                                                                              │
│     Client → RabbitMQ → Worker → Response to Client (synchrone)            │
│                                                                              │
│  ✅ Notre pattern (Task Queue interne):                                     │
│                                                                              │
│     Client → API (svc-*) → Response immédiate (< 200ms)                    │
│                    │                                                        │
│                    └──► enqueue task → Valkey → Worker (async, background)  │
│                                                                              │
│  Différence clé:                                                            │
│  • L'API répond IMMÉDIATEMENT au client                                     │
│  • Le worker traite en BACKGROUND (fire-and-forget ou avec callback)       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Queueing Tiers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         QUEUEING ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ TIER 1 — EVENT STREAMING (Kafka)                                    │    │
│  │ • Use case: Event-driven architecture, CDC, audit logs              │    │
│  │ • Pattern: Pub/Sub, Event Sourcing                                  │    │
│  │ • Ordering: Per-partition guaranteed                                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ TIER 2 — TASK QUEUE (Valkey + Dramatiq)                             │    │
│  │ • Use case: Background jobs, async processing                       │    │
│  │ • Pattern: Producer/Consumer, Work Queue                            │    │
│  │ • Features: Retries, priorities, scheduling                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ TIER 3 — SCHEDULED JOBS (Kubernetes CronJobs)                       │    │
│  │ • Use case: Batch processing, reports, cleanup                      │    │
│  │ • Pattern: Time-triggered execution                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Kafka vs Task Queue — Quand utiliser quoi ?

| Critère | Kafka | Task Queue (Valkey) |
|---------|-------|---------------------|
| **Message Ordering** | ✅ Per-partition | ❌ Best effort |
| **Message Replay** | ✅ Retention-based | ❌ Non |
| **Priority Queues** | ❌ Non natif | ✅ Oui |
| **Delayed Messages** | ❌ Non natif | ✅ Oui |
| **Dead Letter Queue** | ✅ Configurable | ✅ Intégré |
| **Exactly-once** | ✅ Avec idempotency | ❌ At-least-once |
| **Use Case** | Events entre services | Jobs internes async |

## Task Queue Stack

| Composant | Outil | Rôle |
|-----------|-------|------|
| **Task Framework** | Dramatiq (Python) / Asynq (Go) | Task definition, execution |
| **Broker** | Valkey (Redis-compatible) | Message storage, routing |
| **Result Backend** | Valkey | Task results, status |
| **Scheduler** | APScheduler / Dramatiq-crontab | Periodic tasks |
| **Monitoring** | Dramatiq Dashboard / Prometheus | Task metrics |

## Task Processing Flow

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
│       │                    │ • high  │                    │                 │
│       │ Response           │ • default│                   │ execute         │
│       │ immédiate          │ • low   │                    ▼                 │
│       ▼                    │ • dlq   │              ┌─────────┐             │
│   Client                   └─────────┘              │  Task   │             │
│   (n'attend pas)                                    │ Handler │             │
│                                                     └─────────┘             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Queue Definitions

| Queue | Priority | Workers | Use Cases |
|-------|----------|---------|-----------|
| **critical** | P0 | 5 | Transaction rollbacks, fraud alerts |
| **high** | P1 | 10 | Email confirmations, balance updates |
| **default** | P2 | 20 | Notifications, analytics events |
| **low** | P3 | 5 | Reports, cleanup, batch exports |
| **scheduled** | N/A | 3 | Cron-like scheduled tasks |
| **dead-letter** | N/A | 1 | Failed tasks investigation |

## Retry Strategy

| Retry Policy | Configuration | Use Case |
|--------------|---------------|----------|
| **Exponential Backoff** | base=1s, max=1h, multiplier=2 | API calls, external services |
| **Fixed Interval** | interval=30s, max_retries=5 | Database operations |
| **No Retry** | max_retries=0 | Idempotent operations |

## Dead Letter Queue (DLQ) Handling

| Étape | Action |
|-------|--------|
| 1 | Task fails après max retries |
| 2 | Task moved to DLQ avec metadata (reason, stack trace, attempts) |
| 3 | Alert Slack (P3) |
| 4 | On-call investigate |
| 5 | Options: Fix → Replay, Manual resolution, Archive |

## Scheduled Jobs (CronJobs)

| Job | Schedule | Service | Description |
|-----|----------|---------|-------------|
| **balance-reconciliation** | `0 2 * * *` | svc-wallet | Daily balance verification |
| **expired-giftcards** | `0 0 * * *` | svc-giftcard | Mark expired cards |
| **analytics-rollup** | `0 */6 * * *` | svc-analytics | 6-hourly aggregation |
| **log-cleanup** | `0 3 * * 0` | platform | Weekly log rotation |
| **backup-verification** | `0 4 * * *` | platform | Daily backup integrity check |
| **compliance-report** | `0 6 1 * *` | platform | Monthly compliance export |

## Task Queue Monitoring

| Metric | Seuil alerte | Action |
|--------|--------------|--------|
| **Queue Depth** | > 1000 tasks | Scale workers |
| **Processing Time P95** | > 30s | Optimize task, check resources |
| **Failure Rate** | > 5% | Investigate DLQ, check dependencies |
| **DLQ Size** | > 10 tasks | Immediate investigation |
| **Worker Availability** | < 50% | Check pod health, scale up |

---

*Document maintenu par : Platform Team + Backend Team*  
*Dernière mise à jour : Janvier 2026*

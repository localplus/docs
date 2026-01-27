# 📖 **Glossary**
## *LOCAL-PLUS Platform Terminology*

---

> **Retour vers** : [Architecture Overview](EntrepriseArchitecture.md)

---

# 🧩 **1. Core Software Architecture Terms**

| Term | Definition |
|------|------------|
| **Monolith** | Single deployable unit containing all application functionality |
| **Microservices** | Architecture where application is composed of small, independent services |
| **Service boundaries** | Clear interfaces and responsibilities defining where one service ends and another begins |
| **Tight coupling** | Strong dependencies between components making them hard to change independently |
| **Loose coupling** | Minimal dependencies between components allowing independent evolution |
| **Cohesion** | Degree to which elements of a module belong together |
| **Separation of concerns** | Design principle for separating a program into distinct sections |
| **Scalability (vertical)** | Adding more power to existing machines (scale up) |
| **Scalability (horizontal)** | Adding more machines to the pool (scale out) |
| **Fault tolerance** | System's ability to continue operating when components fail |
| **Resilience** | System's ability to recover from failures and continue to function |
| **High availability** | System designed to be operational for a high percentage of time |
| **Latency budget** | Maximum acceptable delay for an operation across the system |
| **Throughput** | Number of operations a system can handle per unit of time |
| **Concurrency** | Multiple computations executing during overlapping time periods |
| **Rate limiting** | Controlling the rate of requests to protect system resources |
| **Backpressure** | Mechanism to resist and control upstream load when overwhelmed |
| **Stateless** | Component that doesn't retain client state between requests |
| **Stateful** | Component that maintains state across requests |
| **Idempotency** | Operation that produces same result regardless of how many times executed |
| **Eventual consistency** | Data will become consistent across replicas given enough time |
| **Strong consistency** | All nodes see the same data at the same time |
| **CAP theorem** | Distributed system can only provide 2 of 3: Consistency, Availability, Partition tolerance |
| **Data locality** | Keeping data close to where it's processed |
| **ACID** | Atomicity, Consistency, Isolation, Durability — transaction guarantees |
| **BASE** | Basically Available, Soft state, Eventually consistent — alternative to ACID |
| **CQRS** | Command Query Responsibility Segregation — separate read and write models |
| **Retry + Exponential backoff** | Retry failed operations with increasing delays |
| **Circuit breaker** | Pattern to prevent cascading failures by failing fast |
| **Bulkhead isolation** | Isolating components to prevent failure propagation |
| **Canary deployment** | Gradual rollout to a subset of users before full deployment |
| **Blue/Green deployment** | Two identical environments, switch traffic between them |
| **Progressive delivery** | Gradual rollout with automated checks and rollback |
| **Feature flags** | Toggles to enable/disable features without deployment |

---

# 🚢 **2. DevOps Core Concepts**

| Term | Definition |
|------|------------|
| **CI/CD** | Continuous Integration / Continuous Delivery — automated build, test, deploy |
| **Fail-Fast** | Design principle to detect and report failures immediately |
| **Deployment pipeline** | Automated sequence of stages from code to production |
| **GitOps** | Infrastructure and application management using Git as source of truth |
| **Pull-based delivery** | Agents pull desired state from Git (vs push-based) |
| **Infrastructure as Code (IaC)** | Managing infrastructure through code rather than manual processes |
| **Configuration drift** | Divergence between actual and intended configuration state |
| **Desired state vs actual state** | What should be vs what currently is |
| **Convergence loop** | Process that continuously moves actual state toward desired state |
| **Immutability** | Resources are replaced rather than modified |
| **Artifact registry** | Repository for storing build artifacts (images, packages) |
| **Environment parity** | Keeping dev, staging, prod as similar as possible |
| **Supply chain security** | Protecting the software delivery pipeline from attacks |
| **Build reproducibility** | Ability to recreate identical builds from same inputs |
| **Trunk-based development** | All developers work on a single branch (main/trunk) |
| **Shift left** | Moving testing and security earlier in the development process |
| **Continuous compliance** | Automated compliance checks integrated into pipeline |
| **Golden pipeline** | Standardized, pre-approved CI/CD pipeline |
| **Self-service delivery** | Teams can deploy without manual intervention |
| **Release automation** | Automated release process with minimal human intervention |
| **Promotion** | Moving artifacts from one environment to the next |

---

# 🛠️ **3. Platform Engineering Vocabulary**

| Term | Definition |
|------|------------|
| **Paved road** | Recommended path that's easy to follow and well-supported |
| **Golden path** | Opinionated, supported way to accomplish common tasks |
| **Developer experience (DevEx)** | Quality of developers' interactions with tools and processes |
| **Self-service portals** | Interfaces for teams to provision resources without tickets |
| **Platform boundaries** | Clear interfaces between platform and application teams |
| **Internal Developer Platform (IDP)** | Set of tools and services that enable self-service |
| **Tenant isolation** | Separation of resources between different users/teams |
| **Blast radius** | Scope of impact when something fails |
| **Multi-tenancy** | Single instance serving multiple isolated tenants |
| **Platform contracts** | Agreements about what the platform provides and expects |
| **Declarative everything** | Describing what you want, not how to achieve it |
| **Reconciliation loop** | Controller pattern that continuously aligns actual with desired state |
| **Policy as Code** | Expressing policies in code for automated enforcement |
| **Control plane vs data plane** | Management layer vs traffic/data processing layer |
| **Standardization** | Consistent patterns across the organization |
| **Opinionated defaults** | Pre-configured choices that work for most cases |
| **Guardrails** | Constraints that guide without blocking |
| **Drift detection** | Identifying when actual state differs from desired |
| **Day-2 operations** | Ongoing operations after initial deployment |
| **Platform lifecycle** | Stages from creation through deprecation |
| **Operational excellence** | Running workloads effectively and gaining insights |
| **Infra product thinking** | Treating infrastructure as a product with users |

---

# 🐳 **4. Container & Kubernetes Terminology**

| Term | Definition |
|------|------------|
| **Control plane** | Components that manage the cluster (API server, scheduler, etc.) |
| **Data plane** | Worker nodes where application workloads run |
| **Pod** | Smallest deployable unit in Kubernetes, one or more containers |
| **Deployment** | Declarative updates for Pods and ReplicaSets |
| **StatefulSet** | Manages stateful applications with stable identities |
| **DaemonSet** | Ensures a Pod runs on all (or some) nodes |
| **Service** | Abstract way to expose an application running on Pods |
| **Ingress** | API object managing external access to services |
| **Gateway API** | Next-generation Ingress, more expressive routing |
| **CRD (Custom Resource Definition)** | Extends Kubernetes API with custom resources |
| **Operator** | Controller that manages complex applications using CRDs |
| **Controller** | Control loop that watches state and makes changes |
| **Reconciliation loop** | Controller pattern comparing desired vs actual state |
| **Desired state store (etcd)** | Key-value store holding cluster state |
| **Horizontal Pod Autoscaler** | Scales Pods based on CPU/memory or custom metrics |
| **Vertical Pod Autoscaler** | Adjusts resource requests/limits automatically |
| **KEDA** | Kubernetes Event-Driven Autoscaling |
| **Knative** | Platform for serverless workloads on Kubernetes |
| **Service mesh** | Infrastructure layer for service-to-service communication |
| **Admission controller** | Intercepts requests before persistence |
| **Mutating webhook** | Modifies resources during admission |
| **Validating webhook** | Rejects invalid resources during admission |
| **Secrets** | Objects for sensitive data (passwords, tokens) |
| **ConfigMaps** | Objects for non-sensitive configuration data |
| **Namespace tenancy** | Using namespaces to isolate workloads |
| **Sidecar pattern** | Helper container running alongside main container |
| **Init containers** | Containers that run before app containers start |
| **Pod disruption budget** | Limits voluntary disruptions to Pods |
| **Resource requests vs limits** | Minimum guaranteed vs maximum allowed resources |
| **OOMKilled / throttling** | Container killed for memory / slowed for CPU |
| **Node pool** | Group of nodes with same configuration |
| **Taints/Tolerations** | Mechanism to repel/accept Pods on nodes |
| **Affinity rules** | Scheduling preferences for Pod placement |
| **kro** | Kubernetes Resource Orchestrator |

---

# 🔄 **5. GitOps Deep Vocabulary**

| Term | Definition |
|------|------------|
| **Declarative manifests** | YAML/JSON files describing desired state |
| **Single source of truth** | Git as the authoritative source for system state |
| **Drift** | When actual state differs from Git-defined state |
| **Convergence** | Process of moving actual state toward desired state |
| **Pull reconciliation** | Agent pulls changes from Git (vs push deployment) |
| **Progressive sync** | Gradual application of changes with health checks |
| **Rollback via Git revert** | Undoing changes by reverting Git commits |
| **Commit-driven deployments** | Deployments triggered by Git commits |
| **Audit trail** | Git history as immutable record of all changes |
| **Policy enforcement** | Automated checks before sync |
| **Drift remediation** | Automatic correction of drift |
| **Secret sealing** | Encrypting secrets for safe Git storage (ex: Sealed Secrets, pas SOPS) |
| **Environments as branches** | Different branches for different environments |
| **Kustomize overlays** | Environment-specific customizations |

---

# ☁️ **6. Cloud Architecture Concepts**

| Term | Definition |
|------|------------|
| **Shared responsibility model** | Division of security responsibilities between cloud and customer |
| **Multi-AZ** | Deployment across multiple Availability Zones |
| **Multi-region** | Deployment across multiple geographic regions |
| **Zonal vs regional resources** | Resources in one zone vs replicated across zones |
| **Edge caching** | Caching content at edge locations near users |
| **Network peering** | Direct network connection between VPCs |
| **Private service connect** | Private connectivity to managed services |
| **NAT gateway** | Network address translation for outbound traffic |
| **Egress costs** | Charges for data leaving cloud provider |
| **Ingress filtering** | Controlling inbound traffic |
| **Cloud IAM** | Cloud Identity and Access Management |
| **Workload identity federation** | Federating external identities with cloud IAM |
| **Service accounts** | Identity for non-human principals |
| **Service perimeter** | Boundary controlling access to resources |
| **Threat modeling** | Systematic analysis of potential threats |
| **Cloud Armor / WAF** | Web Application Firewall services |
| **Autoscaling** | Automatic adjustment of resources based on demand |
| **Rehydration** | Recreating immutable resources from scratch |
| **Blue/Green infra provisioning** | Two environments for zero-downtime infrastructure changes |
| **PAM** | Privileged Access Management |

---

# 🔧 **7. Infrastructure as Code Vocabulary**

## Terraform-specific

| Term | Definition |
|------|------------|
| **Providers** | Plugins that interact with APIs (AWS, GCP, etc.) |
| **Resources** | Infrastructure components managed by Terraform |
| **Data sources** | Read-only queries to existing resources |
| **Modules** | Reusable, encapsulated Terraform configurations |
| **State** | Record of resources Terraform manages |
| **State locking** | Preventing concurrent state modifications |
| **Workspaces** | Separate state files for different environments |
| **Drift** | Difference between state and actual infrastructure |
| **Lifecycle ignore_changes** | Ignoring specific attribute changes |
| **Outputs** | Values exported from modules |
| **Variable validation** | Rules for valid variable values |
| **Sentinel** | HashiCorp's policy as code framework |

## Platform IaC

| Term | Definition |
|------|------------|
| **Composability** | Building complex systems from simpler parts |
| **Reusable patterns** | Standardized infrastructure blueprints |
| **Module registries** | Centralized storage for shared modules |
| **Abstraction leaks** | When implementation details break through abstractions |
| **Snowflake infrastructure** | Unique, non-reproducible configurations |

---

# 🧮 **8. Observability (SRE Vocabulary)**

## Three Pillars + Modern Additions

| Term | Definition |
|------|------------|
| **Logs** | Time-stamped records of discrete events |
| **Metrics** | Numeric measurements aggregated over time |
| **Traces** | Records of request paths through distributed systems |
| **Profiles** | CPU/memory usage patterns over time |
| **Events** | Significant occurrences in the system |
| **Span attributes** | Metadata attached to trace spans |
| **Telemetry pipelines** | Collection, processing, and routing of telemetry |

## Methods & Signals

| Term | Definition |
|------|------------|
| **RED metrics** | Rate, Errors, Duration — for services |
| **USE method** | Utilization, Saturation, Errors — for resources |
| **Golden signals** | Latency, Traffic, Errors, Saturation |
| **Histogram buckets** | Distribution of values in ranges |
| **Sampling** | Recording only a subset of data |
| **Correlation IDs** | Identifiers linking related events |
| **Distributed tracing** | Following requests across service boundaries |
| **Log enrichment** | Adding context to log entries |
| **Span propagation** | Passing trace context between services |
| **Telemetry context** | Shared context for correlated telemetry |
| **P50/P95/P99 Latency** | Percentile latency measurements |

## Advanced Observability

| Term | Definition |
|------|------------|
| **Cardinality** | Number of unique label combinations |
| **Dimensionality** | Number of labels/attributes |
| **Retention policies** | Rules for how long data is kept |
| **Aggregation windows** | Time periods for aggregating data |
| **Exemplars** | Links from metrics to specific traces |
| **Structured logs (JSON)** | Machine-parseable log format |
| **High-cardinality labels** | Labels with many unique values (avoid!) |
| **Traceparent / tracestate** | W3C trace context headers |
| **Baggage propagation** | Passing custom context through requests |
| **Span links** | Connecting related but non-parent spans |
| **Tail-based sampling** | Sampling based on complete trace |
| **Head-based sampling** | Sampling decision at trace start |
| **Adaptive sampling** | Dynamic sampling based on conditions |
| **Context propagation** | Passing trace context between services |
| **Semantic conventions** | OpenTelemetry standard naming |
| **Continuous profiling** | Always-on performance profiling |
| **Flamegraphs** | Visualization of call stacks and time |
| **Log correlation** | Linking logs to traces and metrics |

## Alerting & Incidents

| Term | Definition |
|------|------------|
| **Alert fatigue** | Desensitization from too many alerts |
| **Multi-window burn rates** | Error budget consumption over multiple time windows |
| **Error budgets** | Allowable unreliability before action required |
| **Burn-rate alerts** | Alerts based on error budget consumption speed |
| **SLO/SLA/SLI** | Objective/Agreement/Indicator for service levels |
| **Availability vs reliability** | Uptime vs consistent correct behavior |
| **Thundering herd** | Many clients retrying simultaneously |
| **Retry storms** | Cascading retries overwhelming systems |
| **Cascading failures** | Failure spreading through dependencies |
| **Deadman's switch** | Alert when expected signal is absent |
| **Synthetic monitoring** | Artificial requests to test availability |
| **Service dependency graphs** | Visualization of service relationships |
| **Load shedding** | Dropping requests to protect system |
| **Health probes** | Liveness, readiness, startup checks |
| **Blameless postmortems** | Learning from incidents without blame |
| **MTTR/MTTA/MTBF/MTTD** | Mean Time To Recovery/Acknowledge/Between Failures/Detect |
| **Alert silencing** | Temporarily suppressing alerts |
| **Dead-letter queues (DLQ)** | Queue for failed messages |
| **Observability debt** | Accumulated lack of observability |

## Prometheus Metric Types

| Type | Description | Usage | Exemple |
|------|-------------|-------|---------|
| **Counter** | Valeur qui ne fait qu'augmenter (jamais diminuer) | Comptage d'événements cumulatifs | `http_requests_total`, `errors_total` |
| **Gauge** | Valeur qui peut monter ET descendre | Valeurs instantanées | `temperature`, `queue_size`, `active_connections` |
| **Histogram** | Distribution de valeurs dans des buckets prédéfinis | Latences, tailles de requêtes | `http_request_duration_seconds` |
| **Summary** | Comme Histogram mais calcule les percentiles côté client | Percentiles précis (mais plus coûteux) | `request_latency` |

### Counter vs Gauge

```
Counter (cumulatif):         Gauge (instantané):
     ▲                            ▲
  100│     ●                   50│  ●     ●
   80│   ●                     40│    ●
   60│  ●                      30│●     ●
   40│ ●                       20│        ●
   20│●                        10│
     └────────►                  └────────►
       time                        time
```

### Histogram Buckets

```
http_request_duration_seconds_bucket{le="0.1"}   → Requests < 100ms
http_request_duration_seconds_bucket{le="0.5"}   → Requests < 500ms
http_request_duration_seconds_bucket{le="1.0"}   → Requests < 1s
http_request_duration_seconds_bucket{le="+Inf"}  → All requests (total)

Calcul P99: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### Quand utiliser quoi ?

| Besoin | Type | Pourquoi |
|--------|------|----------|
| Comptage d'événements | Counter | Ne fait qu'augmenter, rate() pour débit |
| Valeur actuelle | Gauge | Peut monter/descendre |
| Latences (P50, P95, P99) | Histogram | Buckets permettent percentiles |
| Taille de queue | Gauge | Valeur instantanée |
| Nombre de requêtes | Counter | Cumulatif, rate() pour RPS |

---

# 🔥 **9. Reliability Engineering Vocabulary**

| Term | Definition |
|------|------------|
| **SLO (Service Level Objective)** | Target reliability level |
| **SLI (Service Level Indicator)** | Metric measuring service behavior |
| **SLA (Service Level Agreement)** | Contractual reliability commitment |
| **Error budget** | Allowable unreliability (100% - SLO) |
| **Budget burn** | Rate of error budget consumption |
| **Reliability targets** | Goals for system reliability |
| **Failure domains** | Scope where failures are isolated |
| **Blast radius** | Impact area of a failure |
| **Incident commander** | Person coordinating incident response |
| **Postmortem (blameless)** | Analysis of incidents without blame |
| **MTTR** | Mean Time To Recovery |
| **MTTD** | Mean Time To Detection |
| **MTTF** | Mean Time To Failure |
| **Runbook** | Step-by-step guide for operational tasks |
| **Playbook** | Guide for responding to specific scenarios |
| **On-call rotation** | Schedule for incident response duty |
| **Escalation path** | Chain for escalating issues |
| **Severity levels** | Categories of incident impact (SEV-1, SEV-2...) |

---

# 🔐 **10. Security Terminology**

| Term | Definition |
|------|------------|
| **Zero Trust** | Never trust, always verify |
| **Principle of least privilege** | Grant minimum necessary access |
| **RBAC** | Role-Based Access Control |
| **ABAC** | Attribute-Based Access Control |
| **Ephemeral credentials** | Short-lived, automatically rotated credentials |
| **Dynamic secrets** | Secrets generated on-demand with TTL |
| **Secret rotation** | Regular replacement of credentials |
| **Time-bound access** | Access that expires automatically |
| **Vault Agent** | Sidecar for secret injection |
| **Token minting** | Creating authentication tokens |
| **Policy boundaries** | Limits on what policies can grant |
| **Just-in-time access** | Access granted only when needed |
| **SBOM** | Software Bill of Materials |
| **Supply chain attacks** | Compromising software delivery pipeline |
| **Secret scanning** | Detecting exposed credentials |
| **Threat modeling** | Systematic security analysis |
| **Attack surface** | All points where attacker could enter |
| **Posture management** | Continuous security state assessment |
| **Vulnerability hygiene** | Keeping systems patched and secure |

---

# 🧵 **11. Networking Vocabulary**

## Core Networking

| Term | Definition |
|------|------------|
| **CIDR** | Classless Inter-Domain Routing notation |
| **Subnets** | Logical subdivisions of a network |
| **VPC peering** | Direct connection between VPCs |
| **VPC Service Controls** | Perimeter around GCP resources |
| **Route table** | Rules for directing network traffic |
| **NAT gateway** | Network Address Translation for outbound traffic |
| **Public vs private subnet** | Internet-accessible vs internal-only |
| **Load balancer (L4 vs L7)** | Transport vs application layer balancing |
| **Reverse proxy** | Proxy that handles client requests for backend servers |
| **TLS termination** | Decrypting TLS at a proxy/load balancer |
| **mTLS** | Mutual TLS — both sides authenticate |
| **VPN tunnels** | Encrypted connections over public networks |
| **Egress control** | Controlling outbound traffic |
| **DNS resolution** | Translating names to IP addresses |
| **Split-horizon DNS** | Different DNS responses internal vs external |
| **Service discovery** | Finding service endpoints dynamically |
| **Latency vs jitter** | Delay vs variation in delay |

## Network Security

| Term | Definition |
|------|------------|
| **Network ACLs** | Stateless firewall rules for subnets |
| **Security groups** | Stateful firewall rules for instances |
| **Firewall rules** | Rules controlling network traffic |
| **Ingress vs egress** | Inbound vs outbound traffic |
| **East-west vs north-south** | Internal vs external traffic |
| **Overlay networks** | Virtual networks on top of physical |
| **Underlay networks** | Physical network infrastructure |
| **Zero trust networking** | Verify every request regardless of source |
| **Network segmentation** | Dividing network into zones |
| **Micro-segmentation** | Fine-grained network isolation |

## DNS

| Term | Definition |
|------|------------|
| **DNS TTL** | Time-To-Live for DNS records |
| **DNS cache poisoning** | Attack corrupting DNS cache |
| **Anycast vs unicast** | Same IP multiple locations vs single location |
| **GSLB** | Global Server Load Balancing |
| **CNAMES vs ANAMEs** | Canonical names vs ALIAS records |
| **DNS SRV records** | Service location records |
| **Weighted DNS records** | Traffic distribution via DNS |
| **DNS failover** | Automatic DNS-based failover |

## Load Balancing

| Term | Definition |
|------|------------|
| **Round robin** | Distributing requests in rotation |
| **Least connections** | Sending to server with fewest connections |
| **Weighted** | Distribution based on server capacity |
| **IP hash** | Consistent routing based on client IP |
| **Sticky sessions** | Routing same client to same server |
| **Connection draining** | Completing requests before removing server |
| **Health checks (active/passive)** | Probing vs observing server health |

## Advanced Networking

| Term | Definition |
|------|------------|
| **Service mesh** | Infrastructure for service communication |
| **Sidecar proxy (Envoy)** | Proxy container alongside application |
| **Policy-based routing** | Routing based on policies not just destination |
| **BGP** | Border Gateway Protocol |
| **ASN** | Autonomous System Number |
| **Peering vs transit** | Direct connection vs paying for routing |
| **PrivateLink / VPC Endpoints** | Private connectivity to services |
| **MTU** | Maximum Transmission Unit |
| **QoS** | Quality of Service |
| **Bandwidth vs throughput** | Capacity vs actual data transfer rate |

## Kubernetes Networking

| Term | Definition |
|------|------------|
| **kube-proxy** | Network proxy on each node |
| **ClusterIP** | Internal-only service IP |
| **NodePort** | Service exposed on node ports |
| **LoadBalancer service** | Service with external load balancer |
| **Ingress controller** | Implementation of Ingress API |
| **Gateway API** | Next-generation ingress specification |
| **NetworkPolicies** | L3/L4 firewall for pods |
| **PodCIDR** | IP range allocated to pods |
| **CNI** | Container Network Interface |
| **Calico / Cilium** | Popular CNI implementations |
| **Pod-to-pod encryption** | Encrypting traffic between pods |

---

# 🗄️ **12. Database Reliability Vocabulary**

| Term | Definition |
|------|------------|
| **RPO/RTO** | Recovery Point/Time Objective |
| **Replication lag** | Delay between primary and replica |
| **Write amplification** | Extra writes from indexing/replication |
| **Connection pooling** | Reusing database connections |
| **Hot standby** | Replica ready for immediate failover |
| **Warm standby** | Replica needing some preparation |
| **Cold failover** | Failover requiring significant setup |
| **Partitioning (sharding)** | Splitting data across databases |
| **Read replicas** | Copies for read-only queries |
| **Transaction boundaries** | Scope of ACID guarantees |
| **Isolation levels** | Degree of transaction isolation |
| **Backfill** | Populating data retroactively |
| **pg_bouncer** | PostgreSQL connection pooler |
| **Vacuum** | PostgreSQL maintenance for dead tuples |
| **Dead tuple accumulation** | Buildup of deleted row versions |
| **Failover election** | Process of choosing new primary |

---

# ✅ **13. Platform Anti-Patterns**

| Anti-Pattern | Description |
|--------------|-------------|
| **Configuration drift** | Actual state diverging from intended |
| **Snowflake servers** | Unique, non-reproducible configurations |
| **Tight coupling** | Components that can't change independently |
| **Hidden dependencies** | Undocumented relationships between systems |
| **Mutating production manually** | Direct changes bypassing automation |
| **Silent failure** | Failures without alerts or logs |
| **Shadow ops** | Unofficial processes outside standard tooling |
| **Orphan secrets** | Unused but still valid credentials |
| **Credential sprawl** | Credentials scattered across systems |
| **Static long-lived passwords** | Credentials that never expire |
| **Single-tenant-by-accident** | Unintended tight coupling to one tenant |

---

# 🧠 **14. Architecture Trade-Off Terminology**

| Trade-Off | Description |
|-----------|-------------|
| **Latency vs throughput** | Response time vs capacity |
| **Cost vs durability** | Expense vs data safety |
| **Consistency vs availability** | Data correctness vs uptime |
| **Security vs convenience** | Protection vs ease of use |
| **Performance vs maintainability** | Speed vs code clarity |
| **Complexity vs control** | Features vs simplicity |
| **Abstraction leakage** | When implementation details break through abstractions |

---

# 🎛️ **15. Control Plane Vocabulary**

| Term | Definition |
|------|------------|
| **Declarative specification** | Describing what you want, not how |
| **Controller manager** | Component running controllers |
| **Watch loops** | Controllers watching for changes |
| **Reconciliation** | Aligning actual with desired state |
| **Drift remediation** | Correcting drift automatically |
| **Desired state store** | Where desired state is persisted |
| **Operator SDK** | Framework for building operators |
| **Custom resources** | User-defined Kubernetes resources |

---

# 🐍 **16. FastAPI Vocabulary**

## FastAPI Core

| Term | Definition |
|------|------------|
| **Path operations** | HTTP method + path combinations |
| **Path operation function** | Function handling a path operation |
| **Dependency injection** | Automatic provision of dependencies |
| **Dependencies (Depends)** | FastAPI's DI mechanism |
| **Request state** | Data attached to request lifecycle |
| **Background tasks** | Tasks executed after response |
| **Middleware** | Code running before/after requests |
| **Routers** | Grouping of path operations |
| **Sub-applications** | Mounting apps within apps |
| **Exception handlers** | Custom error handling |
| **Response models** | Pydantic models for responses |
| **Startup/shutdown events** | Lifecycle hooks |
| **Lifespan protocol** | Modern async context manager for lifecycle |
| **OpenAPI schema generation** | Automatic API documentation |

## Pydantic

| Term | Definition |
|------|------------|
| **BaseModel** | Base class for data models |
| **Field validators** | Validation functions for fields |
| **Model config** | Configuration for model behavior |
| **Strict types** | Types that don't coerce |
| **Alias generation** | Automatic field name aliases |
| **Model inheritance** | Extending models |
| **ORM mode** | Compatibility with ORM objects |

## Async/Concurrency

| Term | Definition |
|------|------------|
| **Event loop** | Core of async execution |
| **Coroutine** | Async function |
| **Context switching** | Switching between coroutines |
| **Async DB engines** | Non-blocking database drivers |

## API Integration

| Term | Definition |
|------|------------|
| **Clients (httpx)** | Async HTTP client library |
| **Session reuse** | Reusing HTTP connections |
| **Circuit breakers** | Preventing cascading failures |
| **Retries with jitter** | Randomized retry timing |
| **Backoff** | Increasing delay between retries |
| **Timeout budgets** | Allocating latency across operations |

---

# 🤖 **17. Modern AI Platform Terms**

| Term | Definition |
|------|------------|
| **RAG** | Retrieval-Augmented Generation |
| **Vector embeddings** | Numerical representations of content |
| **Chunking strategies** | Methods for splitting documents |
| **Hallucination rate** | Frequency of incorrect AI outputs |
| **Prompt injection** | Attack via malicious prompts |
| **Safety guardrails** | Controls preventing harmful outputs |
| **Structured tool calling** | AI invoking tools with typed parameters |
| **Agent orchestration** | Managing multi-step AI workflows |
| **Agent handoff** | Transferring between specialized agents |
| **Latency budget (LLM)** | Acceptable delay for AI responses |
| **Function calling** | AI calling predefined functions |
| **Streaming response** | Incremental output delivery |
| **Semantic caching** | Caching based on meaning similarity |
| **Evaluation metrics (RAGAS)** | Framework for RAG evaluation |
| **Tracing (Langfuse)** | Observability for LLM applications |
| **Observability of prompts** | Tracking prompt performance |
| **Agentic** | AI that can take autonomous actions |
| **vLLM** | High-performance LLM inference engine |

---

# 🧱 **18. How to Use These Terms**

## In PR Reviews

- *"This increases blast radius"*
- *"We risk configuration drift here"*
- *"Can we enforce immutability?"*
- *"Retries need idempotency guarantees"*

## In Meetings

- *"What's the rollback path?"*
- *"What's our boundary for tenant isolation?"*

## In Documentation

- *"We apply progressive delivery to reduce risk"*

---

# 📚 **19. Learning Practice**

For any term, ask:

1. **Define** — What is it?
2. **When to use** — Appropriate scenarios
3. **When NOT to use** — Anti-patterns
4. **Trade-offs** — What you gain/lose
5. **Real-world example** — Concrete usage
6. **Sentence** — How to use it naturally

---

# 🌿 **20. Git Vocabulary**

| Term | Definition |
|------|------------|
| **Cherry-pick** | Apply specific commits to another branch |
| **Backport** | Apply fix from newer to older version |
| **Forwardport** | Apply fix from older to newer version |

---

# 📨 **21. Messaging & Event-Driven Systems**

| Term | Definition |
|------|------------|
| **Outbox Pattern** | Writing events to DB table, then to message broker atomically |
| **Event sourcing** | Storing events as source of truth |
| **CDC (Change Data Capture)** | Capturing database changes as events |
| **Exactly-once semantics** | Guarantee of processing exactly once |
| **At-least-once delivery** | Guarantee of delivery (may have duplicates) |
| **Consumer group** | Group of consumers sharing workload |
| **Partition** | Ordered subset of topic messages |
| **Dead Letter Queue (DLQ)** | Queue for failed messages |
| **Saga pattern** | Distributed transaction via event choreography |
| **Compensating transaction** | Undoing previous transaction on failure |

---

*Document maintenu par : Platform Team*  
*Dernière mise à jour : Janvier 2026*

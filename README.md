# End-to-End DevOps Project

A comprehensive study of modern DevOps architecture patterns — covering distributed systems, microservices, Kubernetes, observability, security, and GitOps. This document serves as both a learning reference and architecture guide.

---

## Table of Contents

1. [Distributed Systems](#1-distributed-systems)
2. [Monolith vs Microservices](#2-monolith-vs-microservices)
3. [API Communication](#3-api-communication)
4. [Service Discovery](#4-service-discovery)
5. [Load Balancing](#5-load-balancing)
6. [High Availability](#6-high-availability)
7. [Auto Scaling](#7-auto-scaling)
8. [Reliability with K8s](#8-reliability-with-k8s)
9. [Security](#9-security)
10. [Observability](#10-observability)
11. [Deployment Strategies](#11-deployment-strategies)
12. [GitOps / CI-CD](#12-gitops--ci-cd)

---

## 1. Distributed Systems

**Core principle:** Scalability, fault tolerance, and load distribution.

- A service can run on multiple servers simultaneously — traffic is distributed across them
- If one server goes down, others continue serving requests (no single point of failure)
- Distributed systems introduce **consistency challenges**: you must decide how nodes stay in sync

**Types of distributed systems:**
- Distributed databases (e.g., Cassandra, CockroachDB)
- Distributed file systems (e.g., HDFS, S3)
- Distributed compute (e.g., Kubernetes, Spark)
- Message queues (e.g., Kafka, RabbitMQ)

**Key challenges:**
- Network partitions
- Data consistency (CAP theorem tradeoffs)
- Clock synchronization
- Partial failure handling

---

## 2. Monolith vs Microservices

### Monolithic Architecture

Everything is deployed as a single unit.

- Works well when starting out — simple to develop, test, and deploy
- Becomes painful at scale: one bug in the payments page requires redeploying the entire app
- Scaling means scaling **everything**, even parts that don't need it

### Microservices Architecture

**Core principle:** Scalability, modularity, independent deployability.

Each capability runs as its own independent service with its own codebase and deployment lifecycle.

| Aspect | Monolith | Microservices |
|---|---|---|
| Deployment | Single unit | Per-service |
| Scaling | All-or-nothing | Independent |
| Fault isolation | Low | High |
| Team ownership | Shared | Per-service |
| Debugging | Simpler | Harder (distributed tracing needed) |

**Trade-offs of microservices:**
- More moving parts, more network calls
- Harder to debug across service boundaries
- Teams can own and deploy their services independently
- Different services can use different tech stacks

**Microservices in this project:** The project architecture decomposes into independently deployable services, each with its own scaling and failure profile.

---

## 3. API Communication

### Synchronous (REST / HTTP)
- One service calls another and **waits** for a response
- Simple and widely understood
- Creates **tight coupling** — caller is blocked until the callee responds
- Failure propagates immediately

### Asynchronous (Message Queues)
- Services communicate via an event/message broker (e.g., Kafka, SQS)
- Caller does not wait — fire and forget
- Better for decoupling, resilience, and throughput
- Adds operational complexity (broker management, message ordering, idempotency)

### gRPC
- Binary protocol (Protocol Buffers) over HTTP/2
- Much faster than REST for internal service-to-service calls
- Strongly typed contracts via `.proto` files
- Used heavily in Kubernetes-native environments

---

## 4. Service Discovery

**Problem:** In a dynamic environment, IP addresses of services change constantly (pods get rescheduled, scaled up/down). Services need a reliable way to find each other.

### Types of Service Discovery

**Client-side discovery:**
- Client queries a service registry (e.g., Consul, Eureka)
- Gets a list of available instances
- Client performs its own load balancing

**Server-side discovery:**
- Client sends request to a fixed endpoint (load balancer / proxy)
- The proxy handles discovery and routing
- Client doesn't care about service instances

### Kubernetes Approach

Kubernetes uses **server-side discovery** natively:

- Every service gets a stable DNS name (`my-service.namespace.svc.cluster.local`)
- `kube-dns` / CoreDNS resolves these names to the ClusterIP
- The kube-proxy handles routing to healthy pods
- When pods scale or restart, the DNS record automatically updates

**Key Kubernetes Service types:**
- `ClusterIP` — internal only (default)
- `NodePort` — exposes on each node's IP at a static port
- `LoadBalancer` — provisions a cloud load balancer
- `ExternalName` — maps to an external DNS name

---

## 5. Load Balancing

**Goal:** High availability and performance — distribute traffic across all healthy instances.

Multiple layers of load balancing exist:

### Layer 4 (Transport Layer)
- Operates on TCP/UDP
- Very fast — does not understand HTTP
- Routes based on IP + port only

### Layer 7 (Application Layer)
- Understands HTTP/HTTPS
- Can route based on URL path, headers, cookies, request content
- Supports SSL termination, content-based routing, intelligent routing rules

### Load Balancing Algorithms

| Algorithm | How it works | Best for |
|---|---|---|
| Round Robin | Requests cycle through servers in order | Stateless, uniform workloads |
| Least Connections | Routes to server with fewest active connections | Long-lived or variable-length requests |
| Weighted Round Robin | Servers get traffic proportional to assigned weight | Mixed capacity server pools |
| IP Hash | Same client IP always hits same server | Sessions requiring stickiness |

### Load Balancing in Kubernetes

**In-cluster:**
- `Service` (ClusterIP) is the default internal load balancer
- `kube-proxy` distributes traffic across healthy pod endpoints
- No external component needed for east-west traffic

**External (Ingress):**
- `Ingress` resource defines HTTP routing rules
- An **Ingress Controller** (e.g., NGINX, Traefik) implements those rules
- Routes external traffic to the correct service based on host/path
- Handles TLS termination

**Cloud-native:**
- `Service` of type `LoadBalancer` provisions a cloud load balancer (ALB, NLB on AWS)
- Advanced application-level routing via AWS ALB Ingress Controller

---

## 6. High Availability

**Goal:** Fault tolerance and redundancy — ensure the system survives failures without downtime.

### Key Concepts

**Fault Tolerance** — designing so that a failure does **not** cause downtime.

**Redundancy** — running multiple instances; if one fails, others take over.

**HA Definition:**
- 99% uptime = ~87 hours downtime/year
- 99.9% = ~8.7 hours downtime/year
- 99.99% = ~52 minutes downtime/year
- 99.999% = ~5 minutes downtime/year

### Achieving HA

- Run **multiple replicas** across multiple nodes and availability zones
- Use **health checks** — load balancer continuously checks if instances are healthy; unhealthy ones get removed from rotation automatically
- **RTO** (Recovery Time Objective) — how long can the system stay down before business impact is unacceptable?
- **RPO** (Recovery Point Objective) — how much data loss is acceptable?

### HA in Kubernetes

- Set `replicas: N` in your Deployment — Kubernetes keeps N healthy pods running at all times
- Spread pods across nodes using `podAntiAffinity` rules
- Use `PodDisruptionBudgets` to enforce minimum availability during node maintenance
- Multi-AZ node groups ensure regional zone failures don't take down all pods

---

## 7. Auto Scaling

**Goal:** Availability, reliability, cost efficiency — scale capacity to match demand automatically.

### Horizontal vs Vertical Scaling

| Type | Approach | Trade-off |
|---|---|---|
| Vertical (Scale Up) | Add more CPU/RAM to existing node | Has hardware ceiling; causes downtime |
| Horizontal (Scale Out) | Add more instances/pods | Preferred for cloud-native; requires stateless services |

### Kubernetes Autoscaling

**HPA — Horizontal Pod Autoscaler**
- Scales the number of pod replicas based on metrics (CPU, memory, custom metrics)
- Configured with `minReplicas`, `maxReplicas`, and target utilization
- Evaluates metrics every 15 seconds

**VPA — Vertical Pod Autoscaler**
- Adjusts CPU/memory requests and limits on pods automatically
- Based on actual observed usage
- Trade-off: VPA always restarts pods to apply new resource values

**Cluster Autoscaler**
- Scales the **node pool** itself (adds/removes EC2 instances)
- Triggered when pods cannot be scheduled due to insufficient node capacity
- When a node is underutilized, it drains pods and removes the node

**KEDA — Kubernetes Event Driven Autoscaler**
- Scales pods based on external event sources (SQS queue depth, Kafka lag, Redis list length, Prometheus metrics)
- Scales to zero when idle — cost-efficient for batch/event-driven workloads
- More intelligent than CPU-based HPA for async workloads

---

## 8. Reliability with K8s

**Goal:** Resilience, self-healing.

### Traditional Deployment Problems
- Deploying too many pods on one node exhausts its resources
- No enforcement of fair resource distribution across teams/workloads

### Kubernetes Reliability Primitives

**Resource Requests and Limits**
- `requests`: minimum guaranteed resources (used for scheduling decisions)
- `limits`: maximum allowed resources (enforced at runtime)
- Always set both — prevents noisy-neighbor problems

**Pod Disruption Budgets (PDB)**
- Ensures a minimum number of pods always stay running during voluntary disruptions (node drains, upgrades)
- Example: `minAvailable: 2` guarantees at least 2 replicas are always up

**ReplicaSets**
- Ensure a desired number of pod replicas are always running
- If a pod crashes, the ReplicaSet controller immediately starts a replacement

**Liveness Probe**
- Kubernetes checks if the app is alive
- If it fails, the container is restarted automatically
- Prevents zombie processes from serving traffic

**Readiness Probe**
- Kubernetes checks if the pod is **ready to receive traffic**
- Pod is only added to the Service endpoints after it passes
- Prevents routing to pods that are starting up or temporarily overloaded

**KEDA (self-healing through scaling)**
- Scales down to zero idle pods, scales up immediately when events arrive
- The platform handles self-healing automatically — there are no fixed idle pods

---

## 9. Security

**Core principle:** Security, zero trust, defense in depth.

Security is a layered concern — it must be addressed at every level of the stack.

### Core Security Concepts

- **Authentication** — who are you?
- **Authorization** — what are you allowed to do?
- **Encryption** — is data protected in transit and at rest?
- **Audit** — can you prove what happened?

### Kubernetes Security

**RBAC (Role-Based Access Control)**
- Controls which users and service accounts can call which Kubernetes API operations
- `Role` / `ClusterRole` — defines allowed actions on resources
- `RoleBinding` / `ClusterRoleBinding` — assigns roles to subjects
- Principle of least privilege: grant only what is needed

**Secrets Management**
- Never hardcode credentials — use Kubernetes `Secrets` or external secret stores (Vault, AWS Secrets Manager)
- Secrets are base64-encoded by default — use encryption at rest for production
- Avoid storing secrets in container image layers

**Pod Security**
- Avoid running containers as root
- Use `readOnlyRootFilesystem: true`
- Drop unnecessary Linux capabilities
- Use non-privileged ports (>1024)

**Network Policies**
- By default, all pods can communicate with all other pods
- Network Policies restrict ingress/egress between pods — zero-trust networking
- Define which services can talk to which

**Zero Trust Model**
- Default deny: trust nothing, verify everything
- Every service must authenticate to every other service
- Use service meshes (Istio, Linkerd) for mutual TLS between services

---

## 10. Observability

**Goal:** Operational excellence — understand what your system is doing and why.

### The Three Pillars

**Logs**
- Timestamped record of events
- Capture: requests, errors, warnings, state changes
- Structured logging (JSON) makes logs machine-queryable
- Tools: Fluentd, Loki, CloudWatch Logs, ELK stack

**Metrics**
- Numerical measurements over time
- The RED method for services:
  - **R** — Rate (requests per second)
  - **E** — Errors (error rate as %)
  - **D** — Duration (latency distribution)
- Tools: Prometheus + Grafana

**Traces**
- Track a single request as it flows through multiple services
- Show exactly where time was spent and where failures occurred
- Tools: Jaeger, Zipkin, AWS X-Ray

### Standard Way to Think About Metrics: USE Method
- **U** = Utilization
- **S** = Saturation
- **E** = Errors

Applied to infrastructure resources (CPU, memory, disk, network).

### Kubernetes Observability

- **Prometheus** scrapes metrics from pods via `/metrics` endpoint
- **Grafana** visualizes dashboards from Prometheus data
- Use **ServiceMonitor** CRDs to tell Prometheus which services to scrape
- **Alertmanager** routes alerts based on rules (PagerDuty, Slack, email)

---

## 11. Deployment Strategies

**Goal:** Continuous delivery, rollback safety.

Different strategies trade off speed, risk, and resource cost.

| Strategy | Approach | Rollback | Resource Cost |
|---|---|---|---|
| Recreate | Stop old, start new | Slow (downtime) | Low |
| Rolling Update | Gradually replace old pods with new | Automatic in K8s | Low |
| Blue/Green | Two identical envs; flip traffic switch | Instant | 2x |
| Canary | Route small % to new version; expand gradually | Fast | Low |

### Rolling Update (Kubernetes default)

- Kubernetes replaces pods one-by-one (or in batches)
- `maxSurge`: how many extra pods can exist during update
- `maxUnavailable`: how many pods can be down during update
- Zero-downtime by default if health checks are configured correctly

### Blue / Green

- Blue = current production
- Green = new version deployed but not serving traffic
- Flip the load balancer to point to green
- Instant rollback: flip back to blue
- Requires maintaining two full environments simultaneously

### Canary

- Route a small % (e.g., 5%) of users to the new version
- Monitor error rates and latency
- Gradually increase traffic if metrics look good
- Roll back by redirecting 100% traffic back to stable
- Progressive delivery tools: Argo Rollouts, Flagger

### Feature Flags

- Decouple deployment from feature release
- Code ships to production but features are toggled on/off via config
- Allows A/B testing and gradual rollout without redeployment

---

## 12. GitOps / CI-CD

**Goal:** Infrastructure as code, availability, and consistency.

**Core problem GitOps solves:** Traditionally, a senior engineer configures the system manually — if they leave, institutional knowledge leaves with them. GitOps codifies all configuration in Git.

### CI/CD Pipeline

```
Code Push → Git → CI Pipeline (build, test, lint) → Container Image → Registry → CD Pipeline → Cluster
```

**CI (Continuous Integration):**
- Automatically builds and tests every commit
- Fails fast if tests break — prevents broken code from reaching production
- Tools: GitHub Actions, GitLab CI, Jenkins, CircleCI

**CD (Continuous Delivery/Deployment):**
- Automatically deploys verified builds to staging or production
- GitOps approach: the cluster always reflects what's in Git
- Tools: ArgoCD, Flux

### GitOps Principles

- **Git as single source of truth** — all cluster state is declared in a Git repo
- **Declarative configuration** — describe desired state, not imperative steps
- **Automated reconciliation** — a controller continuously ensures cluster state matches Git
- **Pull-based** — the cluster pulls changes from Git (not CI pushes to the cluster)

### ArgoCD (GitOps Controller)

- Watches a Git repo for changes to Kubernetes manifests
- Continuously syncs cluster state to match the desired state in Git
- Provides a UI to visualize what's deployed and whether it's in sync
- If someone changes something in the cluster manually (drift), ArgoCD detects and corrects it

### Infrastructure as Code (IaC)

- Kubernetes manifests, Helm charts, and Terraform configs live in Git
- Every infrastructure change goes through PR review
- Changes are auditable, reversible, and reproducible
- Tools: Terraform, Pulumi, AWS CDK, Helm

---

## Architecture Summary

```
                        ┌─────────────────────────────────┐
                        │         GitOps Repo (Git)        │
                        │    K8s Manifests / Helm Charts   │
                        └────────────┬────────────────────┘
                                     │ ArgoCD sync
                        ┌────────────▼────────────────────┐
                        │        Kubernetes Cluster        │
                        │                                  │
         ┌──────────────┤  Ingress (NGINX / ALB)          │
         │              │  ├── Service A (3 replicas)      │
  External│              │  ├── Service B (2 replicas)      │
  Traffic │              │  └── Service C (5 replicas)      │
         └──────────────┤                                  │
                        │  HPA / KEDA (auto-scaling)       │
                        │  Prometheus + Grafana (metrics)  │
                        │  Network Policies (zero trust)   │
                        └─────────────────────────────────┘
```

---

## Key Takeaways

- **Microservices** give you independent scalability and fault isolation — at the cost of operational complexity
- **Kubernetes** is the control plane for everything: scheduling, scaling, self-healing, service discovery
- **Observability** (logs + metrics + traces) is non-negotiable in distributed systems — you cannot debug what you cannot see
- **GitOps** treats infrastructure like software: version-controlled, reviewed, and automatically reconciled
- **Security** must be layered — RBAC, network policies, secrets management, and zero-trust networking together
- **Deployment strategies** exist on a risk/speed tradeoff — canary and blue/green deployments protect production stability

---

## Tools & Technologies Referenced

| Category | Tools |
|---|---|
| Container Orchestration | Kubernetes, Minikube |
| Service Mesh | Istio, Linkerd |
| Ingress | NGINX Ingress Controller, AWS ALB |
| Autoscaling | HPA, VPA, Cluster Autoscaler, KEDA |
| Observability | Prometheus, Grafana, Jaeger, Loki |
| GitOps / CD | ArgoCD, Flux |
| CI | GitHub Actions, GitLab CI |
| IaC | Terraform, Helm |
| Secret Management | HashiCorp Vault, AWS Secrets Manager |
| Load Balancing | NGINX, HAProxy, AWS ALB/NLB |
| Deployment | Argo Rollouts, Flagger |

---

*Based on the End-to-End DevOps Project series. Part 1 covers systems design fundamentals through GitOps.*

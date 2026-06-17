# DevOps & Infrastructure — Complete Deep-Dive Revision Guide
## System Design Interview Preparation | Product-Based Companies

---

**Prepared for:** Yash | AI/GenAI Engineer transitioning to Product Company System Design Interviews
**Coverage:** Containers & Kubernetes · CI/CD & Blue-Green Deployments · Monitoring, Logging & Distributed Tracing · AWS Core Services (S3, EC2, RDS, SQS)

---

## Table of Contents

1. **Containers & Kubernetes** — Namespaces/cgroups, Docker layers, K8s architecture, Pods/Deployments/Services, HPA, Ingress
2. **CI/CD & Blue-Green Deployments** — CI pipeline stages, rolling/blue-green/canary strategies, GitOps, Argo CD
3. **Monitoring, Logging & Distributed Tracing** — RED/USE methods, Prometheus, SLOs/error budgets, structured logs, OTel traces, watermarks
4. **AWS Core Services** — S3 (storage classes, presigned URLs), EC2 (instance families, pricing), RDS Aurora (Multi-AZ, read replicas, RDS Proxy), SQS (Lambda integration, fan-out)
5. **Appendix** — Cross-topic reference, complete production pipeline, BFSI-specific tips

---

# DevOps & Infrastructure — Deep-Dive System Design Notes
### For Product-Based Company Interviews | Beginner → Advanced

---

> **How to use these notes:** Same structure as all previous guides.
> What is it → Why does it exist → How it works step by step → Diagrams → Internals
> → Tradeoffs → Real-world → Failures → Interview tips.
> Every concept is explained from scratch — no prior DevOps knowledge assumed.

---

# TOPIC 1: Containers & Kubernetes

---

## 1. What Problem Do Containers Solve?

The oldest problem in software deployment: **"It works on my machine."**

```
THE CLASSIC NIGHTMARE (pre-containers):

Developer laptop:
  Python 3.10, Django 4.2, PostgreSQL 14, OpenSSL 1.1.1
  App works perfectly ✅

Staging server:
  Python 3.8 (OS default), Django 3.0 (old install), OpenSSL 1.0.2
  App crashes with dependency conflicts ❌

Production server:
  Python 3.11 (different ops team), Django 4.1, OpenSSL 3.0
  Some features work, some silently fail ❌

CI build server:
  Python 3.9... different again ❌

The ENVIRONMENT is part of the application. But traditionally,
you shipped only CODE — and hoped the environment matched.
```

**Containers** package the APPLICATION and ALL ITS DEPENDENCIES (runtime, libraries, config) into a single portable unit that runs IDENTICALLY everywhere — developer laptop, CI server, staging, production.

```
WITH CONTAINERS:

┌─────────────────────────────────────────────────────────┐
│  Container Image: "my-django-app:v2.4.1"                  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ App code: /app/                                      │  │
│  │ Python 3.10.12 (exact version, always)              │  │
│  │ Django 4.2.3 (pinned, always)                       │  │
│  │ All pip packages at exact versions                   │  │
│  │ OpenSSL 1.1.1w (exact, always)                      │  │
│  │ System libs: libc, libssl, etc. (exact versions)    │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

Run this image on:
  Developer laptop → IDENTICAL environment ✅
  CI server → IDENTICAL environment ✅
  Staging → IDENTICAL environment ✅
  Production → IDENTICAL environment ✅
  Another cloud provider → IDENTICAL environment ✅
```

---

## 2. Containers vs Virtual Machines — The Critical Difference

```
VIRTUAL MACHINE:
┌─────────────────────────────────────────────────────────┐
│ PHYSICAL HARDWARE                                         │
│   ┌─────────────────────────────────────────────────┐    │
│   │ HYPERVISOR (VMware, VirtualBox, AWS Xen/KVM)     │    │
│   │  ┌──────────────┐  ┌──────────────┐             │    │
│   │  │  VM 1         │  │  VM 2         │             │    │
│   │  │  Guest OS     │  │  Guest OS     │             │    │
│   │  │  (Linux full) │  │  (Linux full) │             │    │
│   │  │  App + Deps   │  │  App + Deps   │             │    │
│   │  └──────────────┘  └──────────────┘             │    │
│   └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘

Each VM: full OS kernel, gigabytes of disk, minutes to boot.
Isolation: strong (separate kernel), Heavy: yes.

CONTAINERS:
┌─────────────────────────────────────────────────────────┐
│ PHYSICAL HARDWARE                                         │
│   HOST OS KERNEL (Linux — shared by ALL containers)      │
│   CONTAINER RUNTIME (Docker/containerd)                   │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│   │Container│ │Container│ │Container│ │Container│       │
│   │  App A  │ │  App B  │ │  App C  │ │  App D  │       │
│   │ +libs   │ │ +libs   │ │ +libs   │ │ +libs   │       │
│   └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
└─────────────────────────────────────────────────────────┘

Each container: shares the HOST OS KERNEL, only packages
its own application + user-space libraries.
Startup: milliseconds (no OS boot needed).
Size: megabytes (vs gigabytes for VMs).
Isolation: process-level (Linux namespaces + cgroups).

┌───────────────────┬──────────────────────┬──────────────────────┐
│ Property           │ Virtual Machines       │ Containers            │
├───────────────────┼──────────────────────┼──────────────────────┤
│ OS                │ Full OS per VM        │ Shared host OS kernel │
│ Startup time      │ Minutes               │ Milliseconds          │
│ Image size        │ GB                    │ MB                    │
│ Isolation         │ Strong (kernel-level) │ Process-level         │
│ Resource overhead │ High                  │ Very low              │
│ Density           │ ~10s per host         │ 100s-1000s per host   │
│ Security boundary │ Hypervisor            │ Namespace/cgroup      │
└───────────────────┴──────────────────────┴──────────────────────┘
```

---

## 3. How Containers Work Internally — Linux Primitives

Containers aren't magic — they're clever use of two Linux kernel features:

### Namespaces — Isolation

```
Linux namespaces give a process its OWN VIEW of system resources,
making it think it's the only process on the machine:

PID namespace:    Container sees its own PID 1 (init process).
                  Can't see host's PIDs or other containers' PIDs.

Network namespace: Container gets its OWN network interface,
                   IP address, routing table, port space.
                   Container can bind to port 80 without
                   conflicting with another container also
                   binding to port 80 (different namespaces!).

Mount namespace:  Container sees its OWN filesystem root (/).
                  The host's /etc, /home, /var are invisible.

User namespace:   Container's "root" user maps to an unprivileged
                  user on the host (security isolation).

IPC namespace:    Isolated inter-process communication.
UTS namespace:    Container has its own hostname.
```

### cgroups — Resource Limits

```
Control Groups (cgroups) enforce RESOURCE LIMITS on a process group:

Container A: max 2 CPU cores, max 512MB RAM
Container B: max 0.5 CPU cores, max 256MB RAM

WITHOUT cgroups: Container A could consume ALL CPU/RAM,
  starving Container B and the host OS.
WITH cgroups: Container A is HARD LIMITED to its allocation.

ALSO PROVIDES: CPU throttling, I/O limits, network bandwidth
limits, memory accounting, OOM (Out-of-Memory) killing of the
container (not the entire host!) when a container exceeds its
memory limit.
```

---

## 4. Docker — The Container Runtime

```
DOCKER ARCHITECTURE:

Developer CLI:  docker build / docker run / docker push
                    │
                    ▼
           Docker Daemon (dockerd)
                    │
                    ▼
           Container Runtime (containerd → runc)
                    │
                    ▼
           Linux Kernel (namespaces + cgroups)

DOCKER IMAGE: A read-only, layered template for creating containers.
DOCKER CONTAINER: A running instance of an image (writable layer on top).

IMAGE LAYERS (Union Filesystem):
Layer 5: [Your app code]          ← changes frequently, top layer
Layer 4: [pip packages installed] ← changes on dependency updates
Layer 3: [Python 3.10 installed]  ← rarely changes
Layer 2: [Ubuntu 22.04 base OS]   ← almost never changes
Layer 1: [Scratch/base layer]

EACH LAYER IS CACHED separately. If only Layer 5 changes (your
code), Docker reuses layers 1-4 from cache → BUILD IS FAST.
Only the changed layer and above are rebuilt.

DOCKERFILE BEST PRACTICES (ordering matters for cache):
# Put rarely-changing things FIRST (most cache hits):
FROM python:3.10-slim
COPY requirements.txt .          # only changes when deps change
RUN pip install -r requirements.txt  # expensive — cache it!
COPY . .                         # changes every commit — last!
CMD ["gunicorn", "app:app"]

# BAD ORDER (invalidates pip cache on every code change):
FROM python:3.10-slim
COPY . .                         # every code change → cache miss
RUN pip install -r requirements.txt  # re-runs expensively EVERY BUILD
```

---

## 5. What Problem Does Kubernetes Solve?

Containers solve "packaging and running one app." But in production you have:
- 500 containers across 50 microservices
- Containers crashing (need automatic restart)
- Traffic spikes (need more containers, then fewer when quiet)
- Rolling deployments without downtime
- Service-to-service routing
- Health checking, secret management, config management

**Kubernetes (K8s)** is a **container orchestration platform** — it manages WHERE containers run, ensures they're always running (self-healing), scales them up/down, routes traffic between them, and manages deployments. Think of it as the operating system for a cluster of machines.

```
WITHOUT KUBERNETES (managing 500 containers manually):
"Container for payment-service crashed on server 12"
→ Someone must notice, SSH to server 12, restart container
→ Maybe 5 minutes of downtime before anyone notices

"Black Friday — need 50 more payment-service instances NOW"
→ Someone must manually start 50 containers, on which servers?
→ Manual work takes 30 minutes. Black Friday traffic is NOW.

WITH KUBERNETES:
Container crashes → K8s detects (health check fails) →
  AUTOMATICALLY restarts on a healthy node → 5 second recovery

Traffic spike → auto-scaler detects CPU > 70% →
  AUTOMATICALLY starts 50 more containers → 2 minutes, hands-off
```

---

## 6. Kubernetes Architecture — Every Component Explained

```
KUBERNETES CLUSTER:

┌─────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                              │
│                                                               │
│  ┌────────────┐ ┌──────────────┐ ┌──────────────────────┐   │
│  │ API Server  │ │    etcd       │ │  Scheduler            │   │
│  │ (the front  │ │ (distributed  │ │  (decides WHERE to   │   │
│  │  door to K8s│ │  config/state │ │   run each Pod)      │   │
│  │  — REST API)│ │  store)       │ │                      │   │
│  └────────────┘ └──────────────┘ └──────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Controller Manager                                    │    │
│  │  (runs control loops: "desired state vs actual state"  │    │
│  │   ReplicaSet controller, Deployment controller, etc.)  │    │
│  └──────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   NODE 1     │  │   NODE 2     │  │   NODE 3     │
│ (worker)     │  │ (worker)     │  │ (worker)     │
│              │  │              │  │              │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │  kubelet  │ │  │  kubelet  │ │  │  kubelet  │ │
│ │(node agent│ │  │           │ │  │           │ │
│ │talks to   │ │  │           │ │  │           │ │
│ │API server)│ │  │           │ │  │           │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │kube-proxy│ │  │ kube-proxy│ │  │ kube-proxy│ │
│ │(iptables │ │  │           │ │  │           │ │
│ │/networking│ │  │           │ │  │           │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
│              │  │              │  │              │
│ [Pod][Pod]   │  │ [Pod][Pod]   │  │ [Pod][Pod]   │
└─────────────┘  └─────────────┘  └─────────────┘

CONTROL PLANE COMPONENTS:

API Server: The single entry point for all K8s operations.
  kubectl commands, the controller manager, the scheduler —
  ALL talk to the API server. It validates and persists
  resource definitions to etcd.

etcd: A distributed key-value store (uses Raft consensus —
  recall from the Replication topic in Databases notes!) that
  stores ALL cluster state — which pods exist, what nodes are
  available, all configurations. If etcd is lost, the cluster
  state is lost. ALWAYS back up etcd.

Scheduler: When a new Pod needs to run, the scheduler picks
  WHICH node to place it on. Considers: node resource
  availability (CPU/RAM), taints/tolerations, affinity rules,
  pod anti-affinity (don't put all replicas on the same node!).

Controller Manager: Runs dozens of control loops, each
  continuously reconciling "desired state" vs "actual state":
  - ReplicaSet Controller: "I want 3 pods. I see 2. Start 1 more."
  - Deployment Controller: manages rolling updates
  - Node Controller: detects and responds to node failures

WORKER NODE COMPONENTS:

kubelet: Agent on every node. Receives pod specs from API server,
  tells the container runtime (containerd) to start/stop
  containers, reports node status back, runs liveness probes.

kube-proxy: Manages network rules (iptables/IPVS) on each node
  to implement Kubernetes Services — ensuring traffic to a
  Service's ClusterIP gets routed to one of its healthy Pods.

Container Runtime: containerd (the default) or CRI-O — the
  low-level engine that actually pulls images and runs containers.
```

---

## 7. Kubernetes Objects — The Core Abstractions

### Pod — The Smallest Deployable Unit

```
A Pod is one or more containers that:
- Share a NETWORK NAMESPACE (same IP address, same port space)
- Share STORAGE VOLUMES
- Are always scheduled TOGETHER on the same node

USUALLY: one container per Pod.
EXCEPTION: "sidecar" pattern — a main container + a helper
container (e.g., a log-shipping agent, or an Envoy proxy
sidecar for service mesh — relevant to your microservices work).

Pod lifecycle:
Pending → Running → Succeeded/Failed
(K8s schedules it) (container running) (completed/crashed)

Pods are EPHEMERAL — they're meant to die and be replaced.
NEVER address a Pod directly by IP — use a Service.
```

### Deployment — Manage Pod Replicas + Rolling Updates

```
DEPLOYMENT YAML:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3                    ← I want 3 pods running always
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment
        image: company/payment:v2.4.1
        resources:
          requests:              ← guaranteed minimum
            memory: "256Mi"
            cpu: "250m"          ← 250 millicores = 0.25 CPU cores
          limits:                ← hard maximum
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:           ← "is this container healthy?"
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3    ← restart after 3 consecutive failures
        readinessProbe:          ← "is this container ready for traffic?"
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3

LIVENESS vs READINESS PROBES (critical distinction):
Liveness: "Is the container still alive?" If fails → RESTART container.
  Use for: detecting deadlocks, infinite loops, zombie processes.

Readiness: "Is the container ready to receive traffic?" If fails →
  REMOVE from Service's load balancing pool (but DON'T restart).
  Use for: startup time (app takes 30s to warm up), dependency
  unavailability (DB is down — don't crash, just stop getting traffic).

A container can be ALIVE but not READY (e.g., loading ML models).
Readiness ensures users never hit a container that's not ready.
This connects to the Load Balancer health checks from Scalability
notes — Kubernetes uses the SAME concept internally.
```

### Service — Stable Network Endpoint

```
Pods are ephemeral (die and get new IPs). A Service provides a
STABLE ClusterIP and DNS name that load-balances across all
matching Pods:

SERVICE TYPES:
ClusterIP (default): Internal-only IP. Only reachable WITHIN
  the cluster. Use for: service-to-service communication.
  payment-service.default.svc.cluster.local:8080

NodePort: Exposes the service on a PORT on EVERY node.
  Access from outside: any_node_ip:30080
  Use for: development, simple testing. Not for production.

LoadBalancer: Provisions a CLOUD LOAD BALANCER (AWS ELB, GCP
  Load Balancer) pointing to NodePorts on all nodes.
  Use for: exposing services to the internet.
  Gets a real public IP/DNS.

ExternalName: Maps a Service to an external DNS name (e.g.,
  my-database.example.com) — for services outside the cluster.
```

### ConfigMap and Secret — Externalizing Configuration

```
Inject configuration into Pods without baking it into the image:

ConfigMap (non-sensitive config):
  DATABASE_HOST: postgres.prod.svc.cluster.local
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"

Secret (sensitive config — base64 encoded, encrypted at rest):
  DATABASE_PASSWORD: <base64-encoded value>
  API_KEY: <base64-encoded value>
  TLS_CERT: <base64-encoded cert>

WHY THIS MATTERS:
The SAME container image can run in DEV, STAGING, PROD —
with DIFFERENT configs injected via ConfigMap/Secrets.
No secrets baked into images (a security anti-pattern that
would expose them in image registries).

Secrets are mounted as environment variables OR volume files:
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

### Horizontal Pod Autoscaler (HPA)

```
Automatically scales the number of Pod replicas based on metrics:

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  ← scale when avg CPU > 70%

This IS the auto-scaling concept from Scalability notes applied
WITHIN Kubernetes. HPA handles "more pods" scaling; Cluster
Autoscaler (a separate component) handles "more nodes" scaling
(when HPA wants more pods but there's no room on existing nodes).
```

---

## 8. Kubernetes Networking

```
POD-TO-POD COMMUNICATION:
Every Pod gets a unique IP address within the cluster.
Any Pod can communicate with any other Pod by IP.
Implemented by CNI plugins: Calico, Flannel, Cilium.
These use VXLAN tunneling (recall from OSI Model topic,
Networking Fundamentals!) to create a virtual network overlay
across physical nodes.

SERVICE DISCOVERY:
K8s runs CoreDNS in the cluster.
payment-service.default.svc.cluster.local → resolves to ClusterIP
(Same SRV record concept from DNS topic, Networking Fundamentals!)

INGRESS — HTTP Routing INTO the Cluster:

External Traffic
    │
    ▼
┌───────────────────────────────────────────────────────┐
│  INGRESS CONTROLLER (Nginx Ingress, AWS ALB Ingress)   │
│  Routes based on HOST and PATH:                         │
│  api.company.com/payments → payment-service:8080        │
│  api.company.com/orders   → order-service:8080          │
│  app.company.com          → frontend-service:80         │
└───────────────────────────────────────────────────────┘

Ingress = the Kubernetes equivalent of an L7 Reverse Proxy
(recall Reverse Proxy from Scalability notes) — same concept,
implemented as a K8s-native resource.
```

---

## 9. Real-World Usage

**Google:** Invented Kubernetes (open-sourced Borg, their internal cluster manager, as K8s). Google runs billions of containers per week across their infrastructure.

**Shopify:** Runs their entire platform on Kubernetes. During Black Friday 2021, they auto-scaled to handle >40 million requests per minute using K8s HPA and Cluster Autoscaler — exactly the "schedule scale before predictable spike" pattern from the Auto-scaling topic.

**Airbnb:** Migrated from a monolith to microservices running on K8s. Uses custom K8s operators for managing stateful services (databases, Kafka clusters within K8s). Their "Apollo" internal PaaS is built on top of K8s.

**Relevant to your Deloitte/GenAI work:** LLM inference services (your LangGraph-based systems) run on K8s with GPU node pools. K8s GPU scheduling (`nvidia.com/gpu: 1` resource request) assigns specific GPU nodes to inference pods, and HPA scales inference replicas based on request queue depth — connecting auto-scaling and Kubernetes in your actual domain.

---

## 10. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ All pods scheduled on ONE node; │ No pod anti-affinity rules;      │ Set podAntiAffinity rules to      │
│ node fails → total service outage│ scheduler placed all replicas    │ spread pods across nodes/AZs;    │
│                                  │ on the same node                 │ use topologySpreadConstraints    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Pod OOM-killed repeatedly         │ Memory limits too low for actual │ Profile actual memory usage;      │
│ (OOMKilled status)               │ workload; limits set too          │ set limits higher; investigate   │
│                                  │ aggressively                      │ memory leaks in the app          │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Rolling update causes brief       │ Old pods terminated before new    │ Set proper readinessProbe;        │
│ errors (requests fail during      │ pods pass readiness check         │ configure minReadySeconds;       │
│ deployment)                      │                                  │ use maxUnavailable=0 in           │
│                                  │                                  │ deployment strategy               │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ etcd data loss → entire cluster   │ etcd cluster fails with no        │ Regular etcd backups; run etcd   │
│ state lost                        │ backup; nodes with etcd data      │ on 3 or 5 nodes (odd number      │
│                                  │ all fail simultaneously           │ for Raft quorum); multi-AZ etcd  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Liveness probe too aggressive:    │ App needs 60s to start; liveness │ Set initialDelaySeconds > app    │
│ container restarted before it     │ probe fires at 10s → fails →      │ startup time; use separate       │
│ can even start (CrashLoopBackOff) │ restart → never gets to start     │ startupProbe for slow starts     │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 11. Interview Quick-Fire Answers

**Q: What's the difference between a container and a virtual machine?**
A: A VM includes a FULL OS kernel (gigabytes, minutes to boot, strong isolation via hypervisor). A container shares the HOST OS kernel and only packages the application's user-space libraries and code (megabytes, milliseconds to start, process-level isolation via Linux namespaces and cgroups). Containers are far more lightweight and dense — hundreds can run on a single host — but have a smaller security boundary (a kernel exploit could affect all containers on a host, unlike VMs).

**Q: What is the difference between a liveness probe and a readiness probe in Kubernetes?**
A: A liveness probe answers "is this container still alive and healthy?" — failure causes Kubernetes to RESTART the container. A readiness probe answers "is this container ready to receive traffic?" — failure causes Kubernetes to REMOVE the pod from the Service's load balancing pool but NOT restart it. A pod can be alive but not ready (e.g., during startup, or when a dependency like a database is temporarily unavailable). Using both prevents traffic from reaching unhealthy pods AND avoids unnecessary restarts for temporarily unavailable pods.

**Q: How does Kubernetes ensure a deployment has zero downtime during updates?**
A: Kubernetes Deployments use a rolling update strategy by default. It gradually replaces old pods with new ones: starts new pods, waits for their readiness probes to pass, then terminates an equal number of old pods. Configuring `maxUnavailable: 0` ensures old pods are never terminated until new pods are confirmed ready (readiness probe passing). Combined with proper readiness probes, Service load balancing (which automatically excludes non-ready pods), and connection draining, this achieves near-zero-downtime deployments.

**Q: What is etcd and why is it critical in Kubernetes?**
A: etcd is a distributed key-value store (using Raft consensus for consistency) that stores ALL Kubernetes cluster state — pod definitions, service configurations, secrets, node status, deployment specs. It is the "source of truth" for the entire cluster. If etcd data is lost, the cluster loses all knowledge of what should be running. This is why etcd backup and multi-node (3 or 5 instances for Raft quorum) configuration is critical for production Kubernetes clusters.

---
---

# TOPIC 2: CI/CD & Blue-Green Deployments

---

## 1. What Problem Does CI/CD Solve?

In the early days of software, code was written by large teams over months, then released in massive "big bang" deployments — often quarterly or annually. Each release was a high-risk, all-hands-on-deck event.

```
THE WATERFALL / BIG BANG DEPLOYMENT PROBLEMS:

Developer A writes feature X (2 weeks)
Developer B writes feature Y (2 weeks)
Developer C fixes bug Z (3 weeks)
... (all separately, not integrated)

RELEASE DAY (3 months later):
"Let's merge everything and deploy"
→ 47 merge conflicts
→ Feature X breaks Feature Y (never tested together!)
→ Bug Z fix introduced regression in Feature W
→ 8-hour deployment window with rollback plan
→ 3am production fix required
→ Users affected for hours

THESE PROBLEMS SCALE WITH: team size × time between integrations
```

**CI/CD** (Continuous Integration / Continuous Delivery / Continuous Deployment) is the practice of integrating code changes frequently, automatically testing them, and releasing to production often — ideally many times per day.

```
THE CI/CD PHILOSOPHY:

"If something is painful, do it MORE OFTEN so you get better at it
and the individual instances become less painful."

Integration painful? → Integrate EVERY commit (CI)
Testing painful? → Test AUTOMATICALLY on every commit (CI)
Deployment painful? → Deploy CONTINUOUSLY (CD)

Amazon deploys to production THOUSANDS of times per day.
Netflix deploys HUNDREDS of times per day.
Google deploys MILLIONS of times per year (across all services).

These companies deploy MORE often than traditional companies
deploy LESS often — and have FEWER outages as a result,
because each change is small and its blast radius is limited.
```

---

## 2. Continuous Integration (CI) — Build, Test, Validate

```
CI PIPELINE — triggers on EVERY push to any branch:

Developer pushes code to GitHub/GitLab
         │
         ▼
    [TRIGGER: CI Pipeline starts]
         │
         ▼
┌────────────────────────────────────────────────────────┐
│  STAGE 1: BUILD                                         │
│  - Pull code from repo                                   │
│  - Install dependencies (from cache if possible)         │
│  - Compile / build application                           │
│  - Build Docker image                                    │
│  Duration: 1-3 minutes                                  │
├────────────────────────────────────────────────────────┤
│  STAGE 2: STATIC ANALYSIS / LINTING                     │
│  - Code style checks (flake8, eslint, gofmt)            │
│  - Security scanning (bandit for Python, Snyk)          │
│  - Dependency vulnerability scan                         │
│  Duration: 30 seconds - 2 minutes                       │
├────────────────────────────────────────────────────────┤
│  STAGE 3: UNIT TESTS                                    │
│  - Run all unit tests                                    │
│  - Generate coverage report                              │
│  - Fail if coverage drops below threshold (e.g., 80%)   │
│  Duration: 1-5 minutes                                  │
├────────────────────────────────────────────────────────┤
│  STAGE 4: INTEGRATION TESTS                             │
│  - Spin up dependencies (DB, Redis, external services   │
│    via Docker Compose or K8s ephemeral namespaces)       │
│  - Run integration/API tests                             │
│  - Tear down test environment                            │
│  Duration: 5-15 minutes                                  │
├────────────────────────────────────────────────────────┤
│  STAGE 5: BUILD & PUSH IMAGE                            │
│  - Tag Docker image with commit SHA:                    │
│    company/payment-service:a3f8b2c1                     │
│  - Push to container registry (ECR, GCR, Docker Hub)   │
│  Duration: 1-3 minutes (if layers cached)               │
└────────────────────────────────────────────────────────┘
         │
         ▼
    RESULT: ✅ "This commit is tested and safe to deploy"
         OR  ❌ "FAILED: unit test 'test_payment_idempotency'"
               → Developer IMMEDIATELY notified (Slack, email)
               → NO broken code reaches main branch
               → Fast feedback loop (minutes, not days)
```

---

## 3. Continuous Delivery vs Continuous Deployment

```
CONTINUOUS DELIVERY:
Pipeline runs automatically through CI AND staging deployment.
PRODUCTION deployment requires a HUMAN APPROVAL STEP.

CI → (auto) → Staging Deploy → (auto tests) → [HUMAN CLICKS "APPROVE"] → Production

Use when: Compliance requirements mandate human sign-off on
production releases (common in BFSI!), or business reasons
require scheduled release windows.

CONTINUOUS DEPLOYMENT:
Pipeline runs automatically ALL THE WAY to production.
NO human approval step.

CI → (auto) → Staging Deploy → (auto tests pass) → Production (automatic!)

Use when: High confidence in automated tests, small-scope
deployments, fast rollback capability is robust.

Most mature tech companies (Netflix, Spotify, Amazon product
teams) use Continuous Deployment for their core services.
Most BFSI/regulated companies use Continuous Delivery with
human approval gates and change management processes.
```

---

## 4. Deployment Strategies — How to Ship Without Breaking Things

### Rolling Deployment (Kubernetes default)

```
Current: v1 v1 v1 v1 v1  (5 pods running v1)
Deploy v2:
Step 1: v1 v1 v1 v1 v2   (1 pod updated to v2, health checked)
Step 2: v1 v1 v1 v2 v2   (2 pods v2)
Step 3: v1 v1 v2 v2 v2   (3 pods v2)
Step 4: v1 v2 v2 v2 v2   (4 pods v2)
Step 5: v2 v2 v2 v2 v2   (all pods v2, done)

PROS: Simple, no extra infrastructure needed, always has
  capacity (never drops below desired replicas during update)
CONS: During transition, BOTH v1 and v2 are serving traffic
  simultaneously — can cause issues if v1 and v2 are
  INCOMPATIBLE (e.g., v2 requires a new DB schema that v1 also
  writes to — mixed schema state during rollout). Not instant
  rollback (must roll forward or roll back slowly).
```

### Blue-Green Deployment — Zero Risk, Instant Rollback

```
CONCEPT: Run TWO IDENTICAL PRODUCTION ENVIRONMENTS simultaneously.
Only ONE is "live" (receiving real traffic) at any time.

SETUP:
┌─────────────────────────────────────────────────────────┐
│                    LOAD BALANCER / DNS                   │
│                    (currently points to BLUE)            │
└───────────────┬──────────────────────────────────────────┘
                │ ALL traffic
                ▼
┌────────────────────────┐    ┌────────────────────────────┐
│    BLUE ENVIRONMENT     │    │    GREEN ENVIRONMENT        │
│    (LIVE — v1)          │    │    (IDLE — deploying v2)    │
│    5 pods running v1    │    │    5 pods running v2        │
│    DB: production       │    │    DB: production (shared)  │
└────────────────────────┘    └────────────────────────────┘

DEPLOYMENT STEPS:
1. Deploy v2 to GREEN environment (idle — no real traffic)
2. Run smoke tests and full test suite against GREEN
3. "Warm up" GREEN (cache warming — recall Caching notes!)
4. SWITCH TRAFFIC: Load balancer now points to GREEN
   → This is usually a single DNS change or LB rule update
   → Takes seconds to propagate
5. MONITOR GREEN (new production) for 15-30 minutes
6. SUCCESS? → Keep GREEN as live. Tear down BLUE (or keep
     as "previous production" for fast rollback).
   FAILURE? → INSTANT ROLLBACK: point LB back to BLUE.
     Full rollback in seconds!

AFTER SUCCESSFUL SWITCH:
┌────────────────────────────┐    ┌────────────────────────┐
│    BLUE ENVIRONMENT         │    │    GREEN ENVIRONMENT    │
│    (now IDLE — old v1)      │    │    (now LIVE — v2)      │
│    (kept for fast rollback  │    │    5 pods running v2    │
│     for ~1 hour, then       │    │                        │
│     repurposed for v3)      │    │                        │
└────────────────────────────┘    └────────────────────────┘

NEXT DEPLOYMENT: Deploy v3 to BLUE. Switch. Repeat.
(BLUE and GREEN alternate as the "live" environment)

PROS:
✅ INSTANT ROLLBACK — seconds, not minutes
✅ NO MIXED VERSION TRAFFIC — all traffic on one version
✅ GREEN fully tested before ANY real traffic hits it
✅ Easy to test GREEN with production traffic on a % basis
   before full cutover

CONS:
❌ DOUBLE THE INFRASTRUCTURE COST during deployment
   (two full environments running simultaneously)
❌ DATABASE MIGRATIONS are hard — both environments share
   the same production database. Migrations must be
   BACKWARD COMPATIBLE with v1 (add columns, never remove them
   until v1 is fully gone). This is the hardest part of
   blue-green deployments in practice.
```

### Canary Deployment — Gradual, Risk-Controlled Rollout

```
CONCEPT: Route a SMALL PERCENTAGE of real traffic to the new
version, observe for errors/performance issues, then gradually
increase to 100%.

┌────────────────────────┐
│    LOAD BALANCER        │
│    Routing rules:       │
│    95% → v1 (stable)   │
│     5% → v2 (canary)   │
└────────────────────────┘
         │        │
         ▼        ▼
┌────────────┐ ┌────────────┐
│  v1 pods    │ │  v2 pods    │
│ (stable,    │ │ (canary,   │
│  most users)│ │  5% users) │
└────────────┘ └────────────┘

PROGRESSIVE ROLLOUT:
Day 1: 1% canary → monitor (error rate, latency, business metrics)
Day 2: 5% canary → monitor
Day 3: 25% canary → monitor
Day 4: 50% canary → monitor
Day 5: 100% → v2 is now fully live

AUTOMATED ANALYSIS (Spinnaker, Argo Rollouts):
Compare error rate of v2 vs v1 automatically.
If v2's error rate > v1's error rate by threshold:
  → Automatic ROLLBACK (traffic moved back to 100% v1)
  → Alert engineers

USE CASES:
✅ High-risk changes (new payment flow, checkout redesign)
✅ Performance-sensitive changes (new algorithm, new DB schema)
✅ When you want REAL USER FEEDBACK on a small subset before
   committing to the full rollout

CONS:
❌ Complexity (need sophisticated traffic splitting — K8s
   Ingress with weight annotations, or a service mesh like Istio)
❌ Mixed versions serve traffic simultaneously (same DB schema
   compatibility concern as rolling deployments)
❌ Need good observability to detect problems in canary
   (covered in Topic 3: Monitoring)
```

### A/B Testing vs Canary — The Distinction

```
CANARY: Route small % to v2 to DETECT BUGS/REGRESSIONS before
  full rollout. The 5% of users are not intentionally different —
  they're just getting v2 first. Goal: VALIDATE NEW CODE IS SAFE.

A/B TESTING: Route specific USER SEGMENTS to different versions
  to MEASURE WHICH VERSION PERFORMS BETTER on business metrics.
  Intentional: "Group A sees button color Blue; Group B sees Red."
  Goal: MEASURE BUSINESS IMPACT (conversion rate, click-through).

Both use traffic splitting, but the PURPOSE is different.
A/B testing is a PRODUCT DECISION tool; canary is a DEPLOYMENT
SAFETY tool.
```

---

## 5. CI/CD Tooling Landscape

```
SOURCE CONTROL / TRIGGER:
GitHub, GitLab, Bitbucket → code push triggers pipeline

CI SERVERS / PIPELINE RUNNERS:
GitHub Actions (most popular today — YAML-defined workflows
  running in GitHub's own infrastructure)
GitLab CI/CD (tightly integrated with GitLab)
Jenkins (older, self-hosted, highly customizable — common in
  enterprise/BFSI)
CircleCI, Buildkite, TeamCity

ARTIFACT / IMAGE REGISTRIES:
Docker Hub (public)
AWS ECR (Elastic Container Registry)
Google GCR / Artifact Registry
GitHub Container Registry

DEPLOYMENT / GITOPS:
Argo CD (most popular Kubernetes GitOps tool — "desired state
  in Git, Argo CD reconciles cluster to match")
Flux CD (alternative GitOps tool)
Spinnaker (Netflix's open-source CD platform — strong canary
  analysis, multi-cloud, enterprise-grade)
Helm (Kubernetes package manager — templates for deploying
  complex multi-resource applications)

GITHUB ACTIONS EXAMPLE (simplified):
name: CI/CD Pipeline
on:
  push:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: docker build -t myapp:$GITHUB_SHA .
      - name: Run tests
        run: docker run myapp:$GITHUB_SHA pytest
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS ...
          docker push myapp:$GITHUB_SHA

  deploy-staging:
    needs: build-and-test
    steps:
      - name: Deploy to staging K8s
        run: kubectl set image deployment/myapp myapp=myapp:$GITHUB_SHA

  deploy-production:
    needs: deploy-staging
    environment:
      name: production     ← requires manual approval in GitHub UI
    steps:
      - name: Deploy to production K8s
        run: kubectl set image deployment/myapp myapp=myapp:$GITHUB_SHA
```

---

## 6. GitOps — The Modern Deployment Paradigm

```
GITOPS PRINCIPLE: "Git is the single source of truth for
infrastructure and application state."

TRADITIONAL CD: CI pipeline runs kubectl commands directly
  → Pipeline has cluster credentials → Security risk
  → "What's actually running in production?" — need to check
    the cluster (not the git repo) to be sure

GITOPS:
Developer merges PR to main →
Git repo now contains the DESIRED STATE of the cluster
          │
          ▼
    ArgoCD / Flux (running INSIDE the cluster)
    continuously compares:
    DESIRED STATE (git repo) vs ACTUAL STATE (cluster)
          │
          ▼
    Any DRIFT detected? → Argo CD reconciles:
    applies missing resources, removes extra resources,
    updates changed resources

BENEFITS:
✅ AUDIT TRAIL: every infrastructure change is a Git commit
   (who changed what, when, why — PR description)
✅ ROLLBACK: git revert → Argo CD applies the old state
✅ SECURITY: cluster credentials stay IN the cluster —
   no external pipeline needs cluster access
✅ SELF-HEALING: if someone manually modifies the cluster
   (kubectl edit in production — a bad practice), Argo CD
   reverts it to match git
```

---

## 7. Real-World Usage

**Netflix (Spinnaker):** Netflix open-sourced Spinnaker, their CD platform built for multi-cloud deployments. It powers automated canary analysis — comparing error rates and latency between canary and baseline using statistical significance. Netflix deploys to production hundreds of times per day with minimal human intervention.

**Shopify:** Uses GitHub Actions for CI, Kubernetes with Argo CD for GitOps deployments. During Flash Sales, they use pre-emptive scaling (scheduled scaling in K8s) combined with canary deployments — new code is validated on 1% traffic before ramping up, just before a major sale event.

**BFSI / Deloitte context:** Regulated industries commonly use Continuous Delivery (not Deployment) with mandatory change advisory board (CAB) approvals. Jenkins pipelines are still dominant in many banks. Blue-green is preferred over rolling updates because of the database migration complexity and the need for instant rollback — regulators expect outage windows to be minimal and rollbacks to be rehearsed.

---

## 8. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Blue-green: DB migration breaks  │ v2 runs a migration that removes │ Expand-contract pattern: first    │
│ v1 (during rollback window)      │ a column v1 still reads          │ deploy migration that ADDS new    │
│                                  │                                  │ column (v1 and v2 both work);    │
│                                  │                                  │ after v2 fully live, deploy       │
│                                  │                                  │ cleanup migration to remove old   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Canary bad metric not caught;    │ Canary analysis not configured;   │ Define SLOs (error rate, latency) │
│ bad version promoted to 100%     │ monitoring alert thresholds too   │ for automatic canary analysis;    │
│                                  │ loose; slow-burn failure not      │ extend canary observation period  │
│                                  │ caught in short window            │ for high-risk changes             │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ CI pipeline is "green" but app   │ Tests pass in CI but don't cover  │ Shift-left testing: add           │
│ broken in production             │ real-world conditions; mocked     │ integration/contract tests;       │
│                                  │ dependencies hide issues          │ test in staging with prod-like   │
│                                  │                                  │ data; smoke tests post-deploy     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Slow CI pipeline discourages      │ No parallelism; sequential stages;│ Parallelize test stages; cache    │
│ frequent commits ("CI takes 45   │ no caching of dependencies        │ Docker layers and pip/npm deps;   │
│ minutes")                        │                                  │ split tests into fast unit and    │
│                                  │                                  │ slower integration stages         │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 9. Interview Quick-Fire Answers

**Q: What's the difference between blue-green and canary deployments?**
A: Blue-green maintains two complete production environments — traffic is switched ALL AT ONCE from old (blue) to new (green), enabling instant rollback by simply switching back. It gives zero mixed-version traffic but costs double the infrastructure during the switch. Canary gradually routes a SMALL PERCENTAGE of traffic (1-5%) to the new version, observing for issues before slowly ramping to 100%. Canary is better for high-risk changes where you want real-user validation; blue-green is better when you need instant rollback capability and can't tolerate mixed-version traffic (e.g., incompatible API changes).

**Q: What is GitOps and how is it different from traditional CD?**
A: In traditional CD, a pipeline runs kubectl commands or scripts directly against the cluster — the pipeline has cluster credentials and PUSHES changes. In GitOps, the desired cluster state is stored IN GIT, and a tool INSIDE the cluster (Argo CD, Flux) continuously reconciles the actual state to match the desired state in Git. Git is the single source of truth — every infrastructure change is a Git commit with a full audit trail, rollbacks are git reverts, and the cluster self-heals any manual drift.

**Q: What is the "expand-contract" (or parallel change) pattern in blue-green deployments?**
A: When a database migration is needed alongside an application change, you can't simply migrate the schema if the old version (blue) is still running and using the old schema. Expand-contract solves this in two phases: First, deploy a migration that ADDS the new column (or table) but doesn't remove the old one — now both v1 (blue) and v2 (green) work. Switch traffic to v2. Once v1 is fully decommissioned, deploy a second "contract" migration to remove the old column. This avoids any window where the running application is incompatible with the database schema.

**Q: What is Continuous Integration and why is it important?**
A: CI is the practice of integrating code changes into a shared repository FREQUENTLY (ideally every commit), with each integration automatically triggering a build and test suite. This catches integration bugs IMMEDIATELY (when the code is fresh and the developer remembers what they changed) rather than weeks later at release time. The "integration hell" of merging months of divergent work disappears when everyone integrates daily. CI also ensures the main branch is ALWAYS in a releasable state — deployments can happen at any time, not only after a painful merge/integration phase.


---
---

# TOPIC 3: Monitoring, Logging & Distributed Tracing

---

## 1. What Problem Does Observability Solve?

You've deployed 50 microservices across a Kubernetes cluster. A user reports: "My checkout is slow." Without observability, you're completely blind — which of the 50 services is the problem? Is it slow? Throwing errors? Is the database the bottleneck?

**Observability** is the ability to understand what a system is doing INTERNALLY by examining its EXTERNAL OUTPUTS. The three pillars are:

```
THE THREE PILLARS OF OBSERVABILITY:

┌──────────────────────────────────────────────────────────────────┐
│  METRICS                                                           │
│  "What is happening, quantitatively, right now and over time?"    │
│  Numeric measurements aggregated over time.                       │
│  Examples: CPU 72%, p99 latency 340ms, 1,247 errors/min           │
│  Answers: Is the system healthy? Is performance degrading?        │
├──────────────────────────────────────────────────────────────────┤
│  LOGS                                                              │
│  "What exactly happened, line by line?"                           │
│  Timestamped records of discrete events.                          │
│  Examples: "2026-06-14 10:32:15 ERROR payment failed:             │
│             card declined for user_123, order_456"                │
│  Answers: What did the system do? What was the exact error?       │
├──────────────────────────────────────────────────────────────────┤
│  TRACES                                                            │
│  "How did THIS specific request flow through ALL services?"       │
│  End-to-end path of a single request across microservices.        │
│  Examples: request_id=abc followed across 8 services              │
│  Answers: WHERE is the latency? Which service in the chain        │
│           is slow or failing?                                      │
└──────────────────────────────────────────────────────────────────┘

ANALOGY:
Metrics = the dashboard gauges in your car (speed, fuel, engine temp)
  → Tell you something is wrong at a glance
Logs = the mechanic's diagnostic report, event by event
  → Tell you WHAT happened in sequence
Traces = GPS tracking of the EXACT JOURNEY your car took
  → Tell you WHERE in the journey things went wrong
```

---

## 2. Metrics — Deep Dive

### Types of Metrics

```
COUNTER: A value that only INCREASES monotonically. Reset to 0 on restart.
  Examples:
  - Total HTTP requests received: 14,392,821
  - Total errors: 4,231
  - Total bytes sent: 892,342,123,456

  USE FOR: Calculating RATES (requests/second, errors/minute):
  rate(http_requests_total[5m]) → requests per second over 5 min

GAUGE: A value that can go UP or DOWN at any point.
  Examples:
  - Current active connections: 1,247
  - Current memory usage: 3.2GB
  - Current queue depth: 4,891 messages
  - CPU utilization: 72%

  USE FOR: Current state / levels.

HISTOGRAM: Distributes values into BUCKETS, enabling percentile
  calculations (p50, p95, p99 latency).
  Examples:
  http_request_duration_seconds_bucket{le="0.1"} = 4,231
  http_request_duration_seconds_bucket{le="0.5"} = 8,902
  http_request_duration_seconds_bucket{le="1.0"} = 9,987
  http_request_duration_seconds_bucket{le="+Inf"} = 10,001

  FROM THIS: p99 ≈ 0.5s (99% of requests complete within 0.5s)
  USE FOR: Latency distributions, response size distributions.

SUMMARY: Similar to histogram but calculates percentiles
  AT THE APPLICATION SIDE (pre-computed) rather than in the
  monitoring system. Less flexible for aggregation.
```

### The RED Method — Metrics That Matter for Services

```
For every service, track these THREE metrics religiously:

R — RATE: How many requests per second is this service handling?
   → Alert if drops unexpectedly (traffic routing problem?)

E — ERRORS: What fraction of requests are failing?
   → Alert if error rate > 1% (or whatever SLO threshold)

D — DURATION: How long do requests take? (p50, p95, p99)
   → Alert if p99 latency exceeds SLO (e.g., 500ms)

THESE THREE ANSWER: "Is my service healthy right now?"
```

### The USE Method — Metrics That Matter for Infrastructure

```
For every resource (CPU, memory, disk, network):

U — UTILIZATION: How busy is the resource? (percentage of time busy)
   CPU: 72% utilized (warning if > 80% sustained)
   Disk: 61% full (alert if > 85%)

S — SATURATION: How much EXTRA WORK is queued? (can't be handled immediately)
   CPU run queue length (tasks waiting for CPU)
   Memory swap usage (RAM full, using disk — very bad!)
   Disk I/O wait (processes waiting for disk)

E — ERRORS: Is the resource reporting errors?
   Disk read/write errors
   Network packet drops
   Memory ECC errors
```

### Prometheus — The Standard Metrics System

```
PROMETHEUS ARCHITECTURE:

┌──────────────────────────────────────────────────────────────┐
│               INSTRUMENTED APPLICATIONS                        │
│  Your app exposes a /metrics HTTP endpoint:                    │
│  GET /metrics                                                  │
│  # TYPE http_requests_total counter                            │
│  http_requests_total{method="GET",status="200"} 14392821       │
│  http_requests_total{method="POST",status="500"} 4231          │
│  # TYPE process_memory_bytes gauge                             │
│  process_memory_bytes 3456789012                               │
└──────────────────────────────────────────────────────────────┘
                         ▲ Prometheus SCRAPES (pulls) every 15s
                         │
┌──────────────────────────────────────────────────────────────┐
│               PROMETHEUS SERVER                                │
│  - Scrapes /metrics from all services on schedule             │
│  - Stores time-series data in its TSDB                        │
│    (recall Time-series DBs topic, Databases notes!)           │
│  - Runs alert rules: if rate(errors[5m]) > 0.01 → FIRE        │
└──────────────────────────────────────────────────────────────┘
                         │
              ┌──────────┴──────────┐
              ▼                      ▼
┌────────────────┐          ┌─────────────────┐
│  ALERTMANAGER   │          │   GRAFANA        │
│  Routes alerts  │          │   Dashboards &   │
│  to PagerDuty,  │          │   visualization  │
│  Slack, email   │          │   (PromQL queries│
└────────────────┘          └─────────────────┘

PROMQL (Prometheus Query Language) examples:
# Request rate over 5 minutes
rate(http_requests_total[5m])

# Error rate (fraction of requests that are errors)
rate(http_requests_total{status=~"5.."}[5m])
  / rate(http_requests_total[5m])

# p99 latency
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket[5m]))

PULL vs PUSH MODEL:
Prometheus PULLS (scrapes) metrics — services expose an endpoint,
Prometheus polls it on schedule. This is different from push-based
systems (Graphite, StatsD) where apps push metrics to a collector.
PULL benefits: Prometheus can detect if a service is DOWN (scrape
fails), and the service doesn't need to know where to send metrics.
```

### SLOs, SLAs, and SLIs — The Reliability Framework

```
SLI (Service Level INDICATOR): A specific METRIC you measure.
  "p99 latency of the checkout API"
  "error rate of the payment service"

SLO (Service Level OBJECTIVE): The TARGET value for an SLI.
  "p99 latency < 300ms, 99.9% of the time"
  "error rate < 0.1% over any 30-day window"
  An INTERNAL goal your team commits to.

SLA (Service Level AGREEMENT): A CONTRACT with customers/partners.
  "If uptime drops below 99.9% in a month, customer gets a credit"
  An EXTERNAL commitment with financial/legal consequences.

ERROR BUDGET:
If your SLO is 99.9% uptime → you have 0.1% error budget.
In 30 days: 30 × 24 × 60 × 0.001 = 43.2 minutes of allowed downtime.

Error budgets enable:
- If budget is ample: can deploy riskier changes, experiment more
- If budget is nearly exhausted: freeze deployments, focus on reliability
- Makes reliability a QUANTITATIVE, team-shared goal (not vague)

SRE PRINCIPLE: "SLOs are the contract between product and SRE teams.
Error budgets are the mechanism for balancing reliability and velocity."
```

---

## 3. Logging — Deep Dive

### Structured vs Unstructured Logs

```
UNSTRUCTURED LOG (plain text — hard to query at scale):
2026-06-14 10:32:15 ERROR Payment failed for user 123, order 456, card declined

STRUCTURED LOG (JSON — machine-queryable):
{
  "timestamp": "2026-06-14T10:32:15.234Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "abc123xyz",      ← connects to distributed traces!
  "span_id": "def456",
  "user_id": 123,
  "order_id": 456,
  "error": "card_declined",
  "error_code": "INSUFFICIENT_FUNDS",
  "amount": 4999,
  "duration_ms": 342,
  "environment": "production",
  "version": "v2.4.1"
}

WHY STRUCTURED LOGS MATTER AT SCALE:
With 500 microservices logging millions of lines/hour, you need
to QUERY logs: "Show me all card_declined errors in the last hour
where amount > 10000, grouped by error_code."

This query is IMPOSSIBLE on plain text logs without grep+awk
gymnastics. On JSON logs indexed in Elasticsearch: instant.

ALWAYS log in JSON in production. Always include:
- timestamp (ISO 8601 with milliseconds)
- level (DEBUG/INFO/WARN/ERROR)
- service name + version
- trace_id and span_id (for correlation with traces!)
- request_id (from HTTP headers — recall Networking Fundamentals)
- user_id or tenant_id (for multi-tenant filtering)
- duration_ms (how long the operation took)
```

### Log Aggregation Pipeline

```
PROBLEM: 500 services × 20 pods each = 10,000 pod log streams.
Logs on pod's local disk → ephemeral (pod dies → logs gone!).
Need to CENTRALIZE logs durably.

THE STANDARD PIPELINE:

┌──────────────────────────────────────────────────────────────┐
│  APPLICATION PODS (10,000 pods)                               │
│  Each pod writes logs to STDOUT (not to file!)               │
│  Kubernetes captures stdout/stderr per container              │
└──────────────────────────────────────────────────────────────┘
                         │ (read by)
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  LOG COLLECTOR (DaemonSet — one per K8s node)                 │
│  Fluent Bit / Fluentd / Vector                                │
│  - Reads container logs from /var/log/containers/ on the node │
│  - Parses, enriches (adds pod name, namespace, node name)    │
│  - Ships to central log store                                  │
└──────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  LOG AGGREGATOR / BUFFER                                       │
│  Kafka (recall Messaging notes!) — absorbs log bursts,        │
│  buffers during log store outages, enables replay             │
│  OR direct to log store (simpler, less resilient)            │
└──────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  LOG STORE + SEARCH                                            │
│                                                               │
│  ELK Stack: Elasticsearch (index/search) + Logstash          │
│    (processing) + Kibana (UI)                                 │
│                                                               │
│  OR: OpenSearch (AWS fork of Elasticsearch)                  │
│                                                               │
│  OR: Loki (Grafana's lightweight log aggregation —           │
│    indexes LABELS not full text; much cheaper storage;       │
│    trades rich search for cost efficiency)                    │
│                                                               │
│  OR: Datadog Logs, Splunk (SaaS — no self-hosting)           │
└──────────────────────────────────────────────────────────────┘

WHY WRITE LOGS TO STDOUT (not files)?
- Kubernetes captures all stdout/stderr automatically
- No log rotation logic needed in the app
- Pod dies? Kubernetes keeps the last N lines accessible
  via `kubectl logs pod-name --previous`
- Consistent collection path for ALL services regardless
  of language/framework (Python, Java, Go, Node.js all
  just write to stdout)
```

### Log Levels — When to Use Each

```
DEBUG: Extremely verbose. Every function call, every variable.
  NEVER enable in production (volume is enormous — 100x more
  than INFO; will fill disk instantly and cost a fortune in
  centralized logging SaaS).
  Enable temporarily for debugging a SPECIFIC issue.

INFO: Normal operational events. Service started. Request
  received. Job completed. "User 123 logged in."
  PRODUCTION DEFAULT: yes, keep INFO in prod.

WARN: Unexpected but handled situation. "DB connection pool
  at 80% — nearing limit." "Retry attempt 2 of 3 for
  external API call." Not an error yet, but worth knowing.

ERROR: Something failed that shouldn't have. "Payment processing
  failed." "DB query timed out." Always warrant investigation.
  ALERT threshold: if ERROR rate spikes → PagerDuty.

FATAL/CRITICAL: System cannot continue. About to crash or in
  an unrecoverable state. Extremely rare; should ALWAYS alert.
```

---

## 4. Distributed Tracing — Deep Dive

### The Problem: Request Flows Through Many Services

```
User checkout request:
→ API Gateway (1ms)
→ Auth Service (12ms)
→ Cart Service (5ms)
→ Inventory Service (3ms)
→ Payment Service (340ms ← HERE IS THE BOTTLENECK)
→ Order Service (8ms)
→ Notification Service (async)
Total: ~370ms

WITHOUT TRACING: You see "checkout is slow (370ms)" in metrics.
But WHERE? You'd have to grep logs across 6 services, correlate
timestamps, hope they're all in sync — takes hours.

WITH TRACING: One trace shows the ENTIRE journey with exact
timing at each hop — Payment Service took 340ms, everything
else was fine. Investigation takes seconds.
```

### How Distributed Tracing Works

```
CORE CONCEPTS:

TRACE: The entire journey of ONE request through the system.
  Identified by a globally unique TRACE ID (128-bit hex string).

SPAN: One unit of work within a trace — corresponds to one
  operation in one service. A trace is a TREE of spans.

SPAN CONTEXT: The trace_id + span_id + flags that are
  propagated between services (via HTTP headers or Kafka
  message headers) so each service knows it's part of a trace.

PROPAGATION:

Request arrives at API Gateway:
→ Gateway generates TRACE ID: "abc123xyz"
→ Gateway creates Span 1 (span_id: "s001")
→ Gateway calls Auth Service, INJECTING headers:
   X-B3-TraceId: abc123xyz
   X-B3-SpanId: s001
   X-B3-Sampled: 1

Auth Service:
→ EXTRACTS trace context from incoming headers
→ Creates Span 2 (span_id: "s002", parent_id: "s001")
→ Calls Cart Service, injecting context again

...continues through all services...

RESULTING TRACE (a Gantt chart of spans):

Span 1 [API Gateway]     |████████████████████████████| 370ms
  Span 2 [Auth]          |██| 12ms
  Span 3 [Cart]          |█| 5ms
  Span 4 [Inventory]     |█| 3ms
  Span 5 [Payment]       |████████████████████████████| 340ms
    Span 5a [DB query]   |█████████████████████| 250ms ← DB SLOW!
    Span 5b [Card API]   |████| 90ms
  Span 6 [Order]         |██| 8ms

NOW YOU CAN SEE: Payment is slow because the DB query takes 250ms.
The DB query is the root cause. Investigation complete in seconds.
```

### OpenTelemetry — The Standard

```
HISTORICALLY: Each tracing vendor had its own SDK.
Using Datadog? Instrument your code with Datadog SDK.
Switching to Jaeger? Re-instrument everything.
Problem: vendor lock-in at the instrumentation level.

OPENTELEMETRY (OTel): Vendor-neutral, open-source standard for
traces, metrics, AND logs. One SDK, any backend.

COMPONENTS:
OTel SDK: Language libraries (Python, Java, Go, Node.js...) that
  auto-instrument common frameworks (Django, Spring, Express, Gin)
  and provide APIs for manual instrumentation.

OTel Collector: A middleware agent that receives telemetry from
  apps (via OTLP protocol), processes/enriches/samples it, and
  exports to MULTIPLE backends simultaneously:
  → Jaeger (trace visualization)
  → Prometheus (metrics)
  → Elasticsearch (logs)
  OR → Datadog, New Relic, Honeycomb (commercial SaaS)

┌──────────────┐    OTLP     ┌──────────────┐   export   ┌──────────┐
│  Your Service │ ──────────▶ │ OTel Collector│ ─────────▶ │  Jaeger  │
│  (OTel SDK)   │             │              │            │ (traces) │
└──────────────┘             └──────────────┘   export   ┌──────────┐
                                                ─────────▶ │Prometheus│
                                                           │(metrics) │
                                                           └──────────┘

SAMPLING:
100% trace sampling = every request generates a full trace.
At 10,000 req/sec, that's 10,000 traces/sec — expensive to store!

SAMPLING STRATEGIES:
Head-based: Make sampling decision at the START of a trace
  (random % — e.g., sample 1% of all traces). Simple but may
  miss rare important traces.

Tail-based: Collect ALL spans for a trace, THEN decide whether
  to keep it based on the OUTCOME (error? slow? always keep!).
  Better but requires buffering all spans until trace completes.

RULE: Always sample 100% of:
  - Errors and exceptions
  - Slow requests (p99 outliers)
  - Specific high-value flows (checkout, payment)
  Sample 1-10% of normal, fast, successful requests.
```

---

## 5. The Observability Stack — Putting It Together

```
OPEN SOURCE STACK (common in K8s environments):

Metrics:   Prometheus (collection) + Grafana (dashboards + alerts)
Logs:      Fluent Bit (collection) + Loki (storage) + Grafana (UI)
Traces:    OTel SDK (instrumentation) + Jaeger/Tempo (storage/UI)
Alerts:    Alertmanager → PagerDuty / Slack / OpsGenie

COMMERCIAL SaaS (simpler ops, higher cost):

Datadog:   Metrics + Logs + Traces + APM, all integrated, excellent UX
New Relic: Similar to Datadog
Dynatrace: Strong auto-discovery and AI-powered alerting
Grafana Cloud: Managed Grafana + Loki + Tempo + Prometheus

ALERT ROUTING (Alertmanager / PagerDuty):
Severity: CRITICAL → PagerDuty (wake someone up, 24/7 on-call)
Severity: WARNING  → Slack #alerts channel (investigate next business day)
Severity: INFO     → Log to alerting DB (weekly review)

GOOD ALERT DESIGN:
❌ BAD: "CPU > 80% for 1 minute" — too noisy, triggers too often
✅ GOOD: "p99 latency > 500ms for 5 consecutive minutes" (symptom-based)
❌ BAD: Alert on every possible metric (alert fatigue → ignoring alerts)
✅ GOOD: Alert ONLY on symptoms that violate user-facing SLOs
         "SLO BURN RATE: error budget consumed 14x faster than expected"
```

---

## 6. Real-World Usage

**Netflix (Atlas + Edgar):** Netflix built Atlas (their in-house time-series metrics system, handling billions of metrics per minute) and Edgar (distributed tracing). At Netflix's scale, commercial tooling becomes cost-prohibitive. Their tracing system helps engineers identify exactly which microservice in their 700+ service ecosystem caused a given user-visible failure.

**Uber (M3 + Jaeger):** Uber built M3 (open-sourced) for metrics at massive scale, and contributed heavily to Jaeger (the open-source distributed tracing system, now under CNCF). Jaeger was invented at Uber because they needed to trace requests across their large microservice fleet to diagnose latency issues in ride-matching and payment processing.

**Google (Dapper + Cloud Ops):** Google's internal distributed tracing system "Dapper" (2010 paper) pioneered the concepts that became Zipkin, Jaeger, and eventually OpenTelemetry. Google Cloud's operations suite (formerly Stackdriver) provides managed observability for GCP workloads.

**BFSI relevance:** RBI guidelines require maintaining audit logs of all financial transactions for a minimum period (7-10 years depending on transaction type). Structured logging with long-term archival (Elasticsearch → S3 Glacier) is a regulatory requirement, not just a best practice. Distributed tracing is essential for diagnosing payment failures that span multiple systems (API gateway → payment processor → core banking → ledger).

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Log pipeline backed up; critical │ Kafka log buffer full; Fluent   │ Size Kafka log topic with enough  │
│ error logs not arriving in time  │ Bit exerting backpressure;       │ retention; alert on collector     │
│ for incident response            │ Elasticsearch write latency      │ lag; size Elasticsearch for       │
│                                  │ spiked                           │ ingest rate                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Alert fatigue — team ignores     │ Too many noisy alerts on causes  │ Alert only on SLO violations;     │
│ alerts ("cry wolf")              │ not symptoms; low threshold;     │ implement alert severity tiers;   │
│                                  │ no ownership                     │ regular alert review sessions     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Traces show high latency but     │ Sampling rate too low (1%) —     │ Always sample errors 100%;        │
│ the slow requests aren't sampled │ the slow outlier requests        │ use tail-based sampling or        │
│                                  │ happened to be in the 99%        │ adaptive sampling that increases  │
│                                  │ that weren't sampled             │ rate when anomalies detected      │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Metrics cardinality explosion     │ Label with high cardinality      │ Never use high-cardinality        │
│ crashes Prometheus                │ (user_id, request_id as label)   │ values as metric labels; use      │
│                                  │ creates millions of time series  │ logs/traces for per-request data  │
│                                  │                                  │ (recall TSDB cardinality topic,   │
│                                  │                                  │ Databases notes)                  │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: What are the three pillars of observability and how do they differ?**
A: Metrics are numeric measurements aggregated over time (CPU%, request rate, p99 latency) — they tell you IF something is wrong and show trends. Logs are timestamped records of discrete events (structured JSON) — they tell you WHAT specifically happened. Traces follow a single request's journey across all services — they tell you WHERE in the distributed system the problem occurred. You need all three: metrics for alerting and dashboards, logs for root cause investigation, traces for pinpointing which service or operation in a distributed request is the bottleneck.

**Q: What is an SLO and how does the error budget concept work?**
A: An SLO (Service Level Objective) is an internal reliability target — e.g., "p99 latency < 300ms, 99.9% of requests over any 30-day window." The error budget is the allowed "failure budget" — 0.1% of requests can exceed 300ms or fail. If you're consuming the budget faster than expected, you slow down risky deployments and focus on reliability. If budget is ample, you can accept more risk with deployments and experiments. It transforms reliability from a vague goal into a quantified, team-shared constraint.

**Q: How does distributed tracing work across microservices?**
A: The first service that receives a request generates a globally unique Trace ID and creates the first Span. When it calls downstream services, it injects the Trace ID and its Span ID into the request headers (HTTP) or message metadata (Kafka). Each downstream service extracts this context, creates a child Span (recording its own work with start/end timestamps), and propagates the context further. At the end, a tracing backend (Jaeger, Tempo) assembles all spans by Trace ID into a timeline showing the complete end-to-end request journey — a Gantt chart revealing exactly where latency is spent.

**Q: Why is it bad to use high-cardinality values (like user_id) as Prometheus metric labels?**
A: Each unique combination of label values creates a SEPARATE time series in Prometheus. If you label `http_requests_total` with `user_id`, and you have 10 million users, Prometheus must store and query 10 million separate time series — causing memory exhaustion and query timeouts ("cardinality explosion," directly recalling the TSDB topic in Databases notes). High-cardinality per-entity data belongs in LOGS (where you can query "all errors for user_id=123") or TRACES (the trace carries the user_id as an attribute, not a label), not in metrics.

---
---

# TOPIC 4: AWS Core Services — S3, EC2, RDS, SQS

---

## 1. The AWS Mental Model

AWS (Amazon Web Services) is the world's largest cloud provider. Rather than buying and managing physical servers, databases, and storage systems, AWS lets you provision these as on-demand services — pay for what you use, scale up and down instantly, and avoid the capital expense and operational burden of physical infrastructure.

```
THE CLOUD VALUE PROPOSITION:

TRADITIONAL (on-premise):
- Buy servers: ₹50 lakh upfront
- Wait 4-8 weeks for delivery and rack installation
- Provision storage, networking, OS
- Hire ops team to maintain
- Capacity planning: guess 3 years in advance
- Traffic spike? Can't scale quickly — order more servers!

AWS CLOUD:
- Launch 100 servers: 30 seconds, pay by the hour
- Launch 0 servers: costs ₹0
- Unexpected spike? Auto-scale to 1000 servers in minutes
- No physical maintenance, no hardware procurement
- Global infrastructure in 30+ regions — deploy globally in hours

KEY MENTAL MODEL: AWS services are BUILDING BLOCKS.
Your system design answers should specify WHICH AWS service
solves WHICH specific problem — not just "we'll use the cloud."
```

---

## 2. Amazon S3 — Simple Storage Service

### What It Is

S3 is AWS's object storage service. We covered object storage in depth in the Databases notes — S3 IS the canonical example. Everything there applies directly here. This section adds the AWS-specific operational details.

```
S3 CORE MODEL:
BUCKET: A globally unique namespace container (like a domain name).
  Bucket names are globally unique across ALL AWS accounts.
  "my-company-prod-assets-20260614" (uniqueness requires specificity)

OBJECT: Any file/data (up to 5TB per object) stored in a bucket.
  Key: the object's "path" (e.g., "images/product/789/main.jpg")
  Value: the actual bytes
  Metadata: content-type, custom tags, storage class

S3 URL STRUCTURE:
https://my-bucket.s3.amazonaws.com/images/product/789/main.jpg
OR:  https://s3.amazonaws.com/my-bucket/images/product/789/main.jpg
```

### S3 Storage Classes (Cost Optimization)

```
S3 INTELLIGENT-TIERING: Auto-moves objects between tiers based
  on access patterns. Small monitoring fee per object.
  Best for: unpredictable access patterns.

S3 STANDARD: Frequently accessed objects.
  Latency: milliseconds. Cost: highest per GB.

S3 STANDARD-IA (Infrequent Access): Objects accessed monthly.
  Same millisecond latency. Lower storage cost. Per-GET fee.

S3 GLACIER INSTANT RETRIEVAL: Archives accessed quarterly.
  Millisecond retrieval (unlike old Glacier). Much cheaper.

S3 GLACIER FLEXIBLE RETRIEVAL: Archives. Minutes to hours retrieval.
  Very cheap storage. Use for: compliance archives, disaster recovery.

S3 GLACIER DEEP ARCHIVE: Cheapest option. 12-48 hour retrieval.
  Use for: regulatory long-term retention (7-10 year BFSI records).

LIFECYCLE POLICIES (automated tiering — recall Object Storage topic):
{
  "Rules": [{
    "Status": "Enabled",
    "Transitions": [
      {"Days": 30,  "StorageClass": "STANDARD_IA"},
      {"Days": 365, "StorageClass": "GLACIER"},
      {"Days": 2555,"StorageClass": "DEEP_ARCHIVE"}
    ],
    "Expiration": {"Days": 3650}  ← delete after 10 years
  }]
}
```

### S3 Key Features for System Design

```
VERSIONING: Keep all versions of every object.
  Protection against accidental deletion/overwrite.
  Combined with MFA Delete for extra protection.

S3 PRESIGNED URLS: Temporary authenticated URLs for direct
  client access to S3 objects — without routing through your
  application servers:

  # Your backend generates this (Python boto3):
  url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'user/123/doc.pdf'},
    ExpiresIn=3600  # valid for 1 hour
  )
  # → https://my-bucket.s3.amazonaws.com/user/123/doc.pdf?
  #     X-Amz-Algorithm=...&X-Amz-Signature=...&X-Amz-Expires=3600

  Client uses this URL to directly download/upload from/to S3.
  YOUR SERVERS NEVER HANDLE THE FILE BYTES — massive bandwidth saving!

  For UPLOADS (PUT presigned URLs):
  Client uploads directly to S3 → no file going through your
  app server → server just creates the presigned URL (tiny op)
  → S3 stores the file → client confirms upload to your API.
  USED BY: Dropbox, Instagram, Slack file uploads.

S3 EVENTS / NOTIFICATIONS: Trigger Lambda on S3 events:
  Object uploaded → Lambda runs image resize → stores thumbnails
  Object uploaded → Lambda validates → moves to "processed" bucket
  (Directly connects to event-driven architecture, Messaging notes!)

S3 + CLOUDFRONT (CDN): S3 is frequently the ORIGIN for CloudFront
  CDN distributions — exactly the CDN architecture from
  Networking Fundamentals notes, with S3 as the origin store.

CORS CONFIGURATION: If browsers need to directly access S3
  (presigned URLs for file downloads/uploads), configure CORS
  on the S3 bucket to allow requests from your app's domain.

MULTIPART UPLOAD: For objects > 100MB, split into parts (5MB min)
  and upload in parallel — better throughput, resilient to failures
  (failed parts retry independently). Required for objects > 5GB.
```

---

## 3. Amazon EC2 — Elastic Compute Cloud

### What It Is

EC2 provides virtual machines (instances) — virtual servers in the cloud. You choose the hardware profile (CPU, RAM, network, storage type), the OS (Amazon Linux 2, Ubuntu, Windows), and pay by the hour (or second for some instance types).

```
EC2 INSTANCE FAMILIES (key ones for interviews):

t-series (t3, t4g): "Burstable" general purpose.
  Normal load: runs on baseline (e.g., 20% CPU)
  Burst: can burst to 100% CPU using "CPU credits"
  USE FOR: Dev/test, low-traffic apps, microservices with
  variable load. t3.micro is often "free tier."
  
m-series (m6i, m7g): "Balanced" general purpose.
  Equal CPU and memory ratio. No burst limitations.
  USE FOR: Application servers, most production workloads.
  m7g = ARM-based Graviton3 (30-40% cheaper than x86 for same perf)

c-series (c7i, c7g): "Compute optimized." High CPU, less RAM.
  USE FOR: Compute-heavy work (ML inference WITHOUT GPU,
  media encoding, high-traffic web servers, scientific computing)

r-series (r7i, r7g): "Memory optimized." High RAM, moderate CPU.
  USE FOR: In-memory databases, large caches, in-memory analytics.
  r6i.32xlarge: 128 vCPU, 1024 GB RAM — the "big machine" for
  vertical scaling before going horizontal.

p-series / g-series: GPU instances.
  USE FOR: ML training (p4d.24xlarge: 8 × A100 GPUs!), GPU inference,
  rendering. Directly relevant to your LLM/GenAI workloads.

i-series: "Storage optimized." NVMe SSDs, very high IOPS.
  USE FOR: Databases needing local high-speed storage (Cassandra
  nodes, Elasticsearch, high-throughput database servers).
```

### EC2 Pricing Models — Critical for Cost Optimization

```
ON-DEMAND: Pay full hourly rate, no commitment.
  Use for: unpredictable workloads, short-term, dev/test.
  MOST EXPENSIVE. Most flexible.

RESERVED INSTANCES (1 or 3 year commitment):
  Up to 72% cheaper than on-demand.
  Use for: stable, predictable baseline workloads.
  Standard Reserved: specific instance type, AZ.
  Convertible Reserved: can change type, still up to 54% off.

SAVINGS PLANS: Flexible commitment ($/hour spend for 1-3 years).
  Apply across any EC2 instance, Lambda, Fargate.
  Simpler than Reserved Instances. Up to 66% off.
  RECOMMENDED: Most companies use Savings Plans for baseline +
  Spot for burst capacity.

SPOT INSTANCES: Bid on AWS's spare capacity.
  Up to 90% cheaper than on-demand.
  CATCH: AWS can TERMINATE your instance with 2-minute notice
  when they need the capacity back.
  Use for: stateless workloads, batch jobs, ML training
  (with checkpointing!), fault-tolerant distributed workloads.
  NEVER use Spot for: databases, stateful services, anything
  that can't handle sudden termination gracefully.

REAL-WORLD STRATEGY:
Baseline load: Reserved Instances or Savings Plans (predictable)
Normal auto-scaling: On-Demand (predictable burst)
Cost-optimized burst: Spot Instances (with interruption handling)
```

### EC2 Key Concepts for System Design

```
AMI (Amazon Machine Image): A pre-built machine template.
  Your base OS + installed software + configuration.
  Launch 100 identical instances from one AMI in minutes.
  Custom AMIs built in CI/CD (Packer tool) enable immutable
  infrastructure (recall GitOps from CI/CD topic).

SECURITY GROUPS: Virtual firewalls at the instance level.
  Stateful (return traffic automatically allowed).
  Rules: allow TCP port 443 from 0.0.0.0/0 (anyone)
         allow TCP port 5432 from my-app-security-group
         deny everything else by default (implicit deny)

KEY PAIRS: SSH access. Private key on your laptop, public key
  on the instance. NO PASSWORD SSH.

USER DATA: Script that runs on FIRST BOOT of an instance.
  Used for: software installation, config setup, registering
  with monitoring. For production: prefer AMI baking over
  user data (faster launch, consistent state).

PLACEMENT GROUPS:
  Cluster: pack instances CLOSE TOGETHER (same rack) for
    low-latency network (<10μs). Risk: rack failure hits all.
    Use for: HPC, distributed databases needing fast node comms.
  Spread: place instances on DIFFERENT hardware.
    Risk tolerance: individual instance failure. Max 7 per AZ.
    Use for: critical small groups (primary + replicas).
  Partition: logical partitions across different hardware.
    Use for: large distributed systems (Cassandra, Kafka, HDFS)
    where partition failure shouldn't affect other partitions.

AUTO SCALING GROUP (ASG): Manage a fleet of EC2 instances.
  Exactly the Auto-scaling concept from Scalability notes,
  implemented as an AWS service. Integrates with:
  - ALB (Application Load Balancer) for traffic distribution
  - CloudWatch metrics for scaling triggers
  - Launch Templates for instance configuration
  - Multiple AZs for high availability
```

---

## 4. Amazon RDS — Relational Database Service

### What It Is

RDS is a MANAGED relational database service. AWS handles: OS patching, database engine upgrades, backups, Multi-AZ failover, read replica creation, storage scaling. You focus on schema design and queries, not database administration.

```
RDS SUPPORTED ENGINES:
- PostgreSQL (most popular for new workloads)
- MySQL
- MariaDB
- Oracle (enterprise, expensive license)
- SQL Server (enterprise, expensive)
- Amazon Aurora (AWS's own — see below)
```

### Amazon Aurora — The RDS Flagship

```
Aurora is NOT just "managed PostgreSQL/MySQL" — it's a
REARCHITECTED database with a custom storage layer:

AURORA ARCHITECTURE:
┌───────────────────────────────────────────────────────────┐
│                    COMPUTE LAYER                            │
│  Primary Instance (write)    Read Replicas (up to 15)      │
│  (handles writes and reads)  (handle reads only)           │
└───────────────────────────────────────────────────────────┘
         │ writes to                    │ reads from
         ▼                              ▼
┌───────────────────────────────────────────────────────────┐
│              DISTRIBUTED STORAGE LAYER                      │
│  6 copies of data across 3 Availability Zones              │
│  Writes: quorum of 4/6 copies (durable immediately)        │
│  Reads: quorum of 3/6 copies                               │
│  SELF-HEALING: automatically repairs corrupt data blocks   │
│  from remaining copies                                      │
└───────────────────────────────────────────────────────────┘

WHY AURORA IS DIFFERENT FROM STANDARD RDS:
Standard RDS Primary-Replica: primary writes to primary's disk,
  then ships WAL to replicas → replication lag of seconds.
Aurora: ALL instances (primary + replicas) read FROM THE SAME
  distributed storage layer → replica lag: typically <10ms!
  A read replica "immediately" sees writes (near-zero lag).

AURORA SERVERLESS v2:
Aurora capacity that auto-scales in fine-grained increments
(0.5 ACU units at a time, where 1 ACU ≈ 2GB RAM).
Scale from minimum to maximum based on actual load — no
pre-provisioning. Scales up in under a second.
PERFECT FOR: Variable workloads, dev/test (can scale to zero!),
unpredictable traffic (flash sales, startup growth).
```

### RDS Key Features for System Design

```
MULTI-AZ DEPLOYMENT (High Availability):

┌──────────────────┐              ┌──────────────────┐
│   PRIMARY RDS     │  synchronous │   STANDBY RDS     │
│   Availability    │─ replication▶│   Availability    │
│   Zone A          │              │   Zone B          │
│   (handles all   │              │   (hot standby,   │
│    reads+writes) │              │    NOT for reads) │
└──────────────────┘              └──────────────────┘

Automatic failover if primary fails: ~60-120 seconds.
The ENDPOINT DNS (my-db.cluster-xyz.us-east-1.rds.amazonaws.com)
AUTOMATICALLY points to the new primary after failover —
application reconnects to the same hostname, now the standby.

CRITICAL: Multi-AZ standby is NOT a read replica — it's a
hot standby FOR FAILOVER ONLY. To add read capacity, create
separate Read Replicas.

READ REPLICAS (Read Scaling):
Up to 5 read replicas per RDS instance (15 for Aurora).
Asynchronous replication (small lag).
Each has its OWN endpoint — your app must explicitly route
read queries to a read replica endpoint.
Can be promoted to independent primary (disaster recovery).
Can be in DIFFERENT REGIONS (cross-region read replicas) —
same Geo-distribution pattern from Scalability notes.

RDS PROXY:
A managed connection pooler that sits between your app
and RDS:

App (Lambda functions, many EC2 instances)
  │  (hundreds of short-lived connections)
  ▼
RDS Proxy
  │  (maintains ~10 long-lived connections)
  ▼
RDS Instance

WHY ESSENTIAL FOR LAMBDA + RDS:
Lambda functions are stateless and may run in hundreds of
concurrent instances. Each Lambda instance opening a DB
connection directly → could exhaust RDS connection limit
(PostgreSQL: typically 100-500 max connections).
RDS Proxy: Lambda connects to Proxy, Proxy maintains a
POOL of RDS connections — your 1000 Lambda instances share
20 actual DB connections via the pool.

AUTOMATED BACKUPS + POINT-IN-TIME RECOVERY (PITR):
RDS takes daily snapshots + continuous WAL archiving.
PITR: Restore database to ANY SECOND within retention period
(1-35 days). Essential for: "Oops, someone ran DELETE without
WHERE at 2:17:43pm yesterday — restore to 2:17:42pm."

PARAMETER GROUPS: Database configuration as code.
  max_connections: 200
  shared_buffers: 8GB
  work_mem: 64MB
  log_min_duration_statement: 1000  ← log queries > 1 second
Apply parameter group to multiple RDS instances for consistency.
```

### RDS vs Aurora vs Self-Managed on EC2

```
┌──────────────────────┬─────────────────────┬─────────────────────┬──────────────────────┐
│ Factor                │ Self-managed on EC2  │ RDS                  │ Aurora                │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Operational burden    │ Full (you do it all) │ Low (AWS manages OS, │ Low (same as RDS)     │
│                      │                      │ backups, failover)   │                       │
│ Customization         │ Maximum              │ Moderate             │ Less than RDS for     │
│                      │                      │                      │ some params           │
│ Cost                  │ Lower (no managed    │ Higher than EC2       │ Higher than RDS        │
│                      │ premium) but you pay  │ (typically 20-30%    │ (typically 20% more   │
│                      │ ops labor             │ more than EC2)       │ than RDS)             │
│ Replication lag       │ Configurable         │ Seconds (async)       │ < 10ms (shared storage│
│ (read replicas)       │                      │                      │ architecture)          │
│ Failover time         │ Manual/scripted      │ 60-120 seconds        │ < 30 seconds          │
│ Max read replicas     │ Unlimited             │ 5                    │ 15                    │
│ Serverless scaling    │ No                   │ No (Aurora Serverless│ Yes (Aurora Serverless│
│                      │                      │ for Aurora only)     │ v2)                   │
│ Best for              │ Extreme customization│ Standard relational  │ High-performance,     │
│                      │ or open-source tools  │ workloads, familiar  │ variable load, or     │
│                      │ not in RDS            │ managed experience   │ very high read scale  │
└──────────────────────┴─────────────────────┴─────────────────────┴──────────────────────┘
```

---

## 5. Amazon SQS — Simple Queue Service

### What It Is

We covered SQS in detail in the Messaging & Event Streaming notes (Topic 1: Message Queues). SQS IS the primary example used there. This section adds the AWS-specific operational details and integration patterns.

### SQS Architecture in AWS

```
PRODUCER (Lambda, EC2, ECS, EventBridge)
    │
    │  SendMessage (up to 256KB body)
    ▼
┌────────────────────────────────────────────────────────────┐
│                    SQS QUEUE                                 │
│  Distributed across multiple AWS AZs (high availability)    │
│  Messages replicated to 3 AZs before ACKing producer        │
│  Standard: virtually unlimited throughput                    │
│  FIFO: 3,000 msg/sec with batching (300 without)            │
└────────────────────────────────────────────────────────────┘
    │
    │  ReceiveMessage (up to 10 messages per call)
    ▼
CONSUMER (Lambda trigger, EC2 workers, ECS tasks)
    │
    │  DeleteMessage (explicit ACK after processing)
    ▼
(message deleted from queue)
```

### SQS + Lambda — The Most Common Serverless Pattern

```
The most common AWS pattern: SQS triggers Lambda automatically.

┌─────────────────┐      ┌─────────────────┐      ┌─────────────┐
│  Order Service   │      │   SQS Queue       │      │   Lambda     │
│  (EC2/ECS/Lambda)│ ─── │  "order-events"   │ ─── │  Function    │
│  sends messages  │      │  (buffer + retry) │      │  (processes  │
└─────────────────┘      └─────────────────┘      │  each batch) │
                                                    └─────────────┘

Lambda Event Source Mapping:
- Lambda polls SQS automatically (you don't write polling code)
- Delivers batches of up to 10 (standard) or 10,000 (FIFO) messages
- If Lambda function throws an exception → entire batch retried
- After maxReceiveCount failures → batch moved to DLQ
- Lambda concurrency scales automatically with queue depth
  (up to a configured maximum) — automatic consumer scaling!

BATCH WINDOW: Lambda waits up to N seconds to accumulate a
  full batch before invoking (reduces Lambda invocation cost
  for high-volume, latency-tolerant processing).

IMPORTANT: If Lambda throws on ANY message in a batch,
  ALL messages in the batch are retried. To avoid retrying
  good messages with bad ones, use "reportBatchItemFailures"
  — report ONLY the specific message IDs that failed:

def handler(event, context):
    failures = []
    for record in event['Records']:
        try:
            process(record)
        except Exception:
            failures.append({'itemIdentifier': record['messageId']})
    return {'batchItemFailures': failures}  # only retry these
```

### SQS Integration Patterns

```
PATTERN 1: SQS + Auto Scaling Group
  Queue depth → CloudWatch metric → ASG scaling policy
  Queue depth > 1000 messages → scale out workers
  Queue depth < 100 messages → scale in workers
  (Exact "auto-scale based on queue depth" from Scalability notes!)

PATTERN 2: SNS → SQS Fan-Out (pub/sub)
  (Exactly the SNS+SQS pattern from Messaging notes Topic 4)
  OrderPlaced → SNS Topic → multiple SQS queues (one per consumer)
  Each consumer: email, inventory, analytics each get own queue
  Each queue has its own DLQ, visibility timeout, retry policy

PATTERN 3: SQS → Lambda → SQS (pipeline chaining)
  Raw data → SQS1 → Lambda (parse) → SQS2 → Lambda (enrich)
    → SQS3 → Lambda (store)
  Each stage decoupled; each has its own DLQ; independently scalable.

PATTERN 4: SQS for request/response (async API)
  Client → POST /jobs → Server creates job, sends to SQS, returns job_id
  Background Worker reads SQS → processes → stores result
  Client polls: GET /jobs/{job_id}/result → returns when done
  Use for: long-running operations (ML inference, report generation,
  video encoding) that can't complete within HTTP timeout (30s).
  Client gets immediate response, polls for completion.
```

---

## 6. How These Four Services Work Together — Reference Architecture

```
COMPLETE AWS ARCHITECTURE FOR A PRODUCTION WEB APPLICATION:

                    USERS
                      │
                      ▼
            ┌──────────────────┐
            │  Route 53 (DNS)   │
            │  + CloudFront CDN │  ← Static assets served from
            └──────────┬───────┘    CloudFront + S3
                       │
                       ▼
            ┌──────────────────┐
            │  ALB (Load Bal.)  │  ← L7 load balancer distributes
            └──────────┬───────┘    traffic across EC2/ECS/Lambda
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │  EC2     │  │  EC2     │  │  EC2     │  ← Auto Scaling Group
    │  (app    │  │  (app    │  │  (app    │    scales 3-50 instances
    │  server) │  │  server) │  │  server) │    on CPU/request metrics
    └────┬────┘  └────┬────┘  └────┬────┘
         │             │             │
         └─────────────┴─────────────┘
                       │
              ┌────────┴────────┐
              │                  │
              ▼                  ▼
    ┌──────────────────┐  ┌──────────────────┐
    │  RDS Aurora       │  │  ElastiCache      │
    │  (Primary +       │  │  (Redis)          │
    │   Read Replicas)  │  │  for caching      │
    └──────────────────┘  └──────────────────┘
              │
              ▼
    ┌──────────────────┐      ┌──────────────────┐
    │  SQS Queue(s)     │ ───▶ │  Lambda           │
    │  for async work   │      │  (background      │
    │  (email, reports, │      │   workers)        │
    │   image resize)   │      └──────────────────┘
    └──────────────────┘
              │
    ┌─────────┴─────────┐
    │                    │
    ▼                    ▼
┌──────────────┐  ┌──────────────┐
│  S3 Bucket    │  │  S3 Bucket    │
│  (user uploads│  │  (processed   │
│   raw files)  │  │   output)     │
└──────────────┘  └──────────────┘

OBSERVABILITY (Topic 3) overlaid on this entire architecture:
CloudWatch: metrics from EC2, RDS, SQS, Lambda (built-in)
CloudWatch Logs: centralized log aggregation (all services auto-send)
CloudWatch Alarms: trigger Auto Scaling, PagerDuty alerts
X-Ray: AWS's distributed tracing service (OTel-compatible)
```

---

## 7. Real-World Usage

**Airbnb:** Uses a comprehensive AWS stack — EC2 with Reserved Instances for baseline compute, Auto Scaling Groups for variable load, RDS Aurora for core transactional data (bookings, payments), S3 for all media (listing photos — petabytes), SQS for booking confirmation emails and notifications, CloudFront in front of S3 for global photo delivery.

**Netflix:** Heavy S3 user — their entire video content catalog (petabytes) lives in S3, distributed to custom CDN appliances (Open Connect) nightly. RDS Aurora for content metadata. EC2 Spot Instances for encoding new video content (fault-tolerant, checkpointed batch job — ideal for Spot).

**Stripe:** Uses EC2 with significant Reserved Instance coverage for predictable baseline load, with Spot for stateless, fault-tolerant batch processing. SQS for payment webhook delivery queues. S3 for compliance document storage with Glacier archival.

**BFSI context:** Indian BFSI companies using AWS must comply with RBI data localization requirements — the ap-south-1 (Mumbai) region is the default, with ap-south-2 (Hyderabad) available for multi-AZ within India. RDS with Multi-AZ within the same region satisfies data residency while providing HA. S3 with lifecycle policies to Glacier satisfies long-term transaction record retention requirements. SQS for async payment processing workflows between core banking and payment gateways.

---

## 8. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ S3 object not found after a     │ Eventual consistency edge case    │ Modern S3 has strong consistency  │
│ recent write (rare, now fixed)  │ (pre-December 2020 S3 behavior)  │ for new objects — architect with  │
│                                  │ OR object key typo/prefix issue  │ content-hash filenames to avoid   │
│                                  │                                  │ key-not-found issues at all        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ RDS failover takes 2+ minutes    │ Multi-AZ failover + DNS TTL;     │ Shorten DNS TTL for RDS endpoint; │
│ → app gets DB connection errors  │ application not handling         │ implement retry logic with         │
│                                  │ transient connection failures    │ exponential backoff in app;        │
│                                  │                                  │ Aurora failover is much faster     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Lambda + RDS: "too many          │ Each Lambda invocation opens a   │ Use RDS Proxy between Lambda and  │
│ connections" error under load    │ new DB connection; 1000 Lambdas  │ RDS — Proxy pools connections,    │
│                                  │ = 1000 connections > RDS max     │ Lambda connects to Proxy instead  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ EC2 Spot Instance termination    │ AWS reclaims Spot capacity with  │ Use Spot only for stateless,       │
│ causes data loss or job failure  │ 2-minute warning                 │ checkpointed, or interruption-     │
│                                  │                                  │ tolerant workloads; handle the    │
│                                  │                                  │ termination notice to checkpoint   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ S3 "hot prefix" throttling        │ Many objects with same key       │ Randomize key prefixes for        │
│ under high request rate           │ prefix; S3 partitions by prefix; │ high-throughput use cases;        │
│                                  │ one partition overwhelmed         │ S3 auto-scales partitions after   │
│                                  │                                  │ ~30 minutes but initial burst     │
│                                  │                                  │ can hit limits                     │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 9. Interview Quick-Fire Answers

**Q: When would you use S3 vs EBS vs EFS for storage on AWS?**
A: S3 (object storage) for: unstructured files, images/videos, backups, data lakes, static website hosting, anything accessed via HTTP. No filesystem interface. EBS (Elastic Block Store) for: the root volume of an EC2 instance, databases running on EC2 (block device, like an SSD attached to one instance at a time). EFS (Elastic File System) for: shared filesystem across MULTIPLE EC2 instances simultaneously (NFS protocol) — useful for shared config, or stateful applications that need a shared mount point across instances.

**Q: What's the difference between RDS Multi-AZ and Read Replicas?**
A: Multi-AZ is for HIGH AVAILABILITY — a standby instance in a different AZ receives synchronous replication from the primary. It takes over automatically on primary failure (DNS failover in ~60-120s). The standby does NOT serve read traffic. Read Replicas are for READ SCALING — asynchronous replicas that CAN serve read queries, with a small replication lag. They don't provide automatic failover but can be manually promoted. For HA: use Multi-AZ. For read scaling: use Read Replicas. Production: use BOTH.

**Q: What is RDS Proxy and why is it important for Lambda-based architectures?**
A: Lambda functions are stateless and may scale to hundreds or thousands of concurrent instances. Each instance opening a direct database connection could exhaust RDS's connection limit (PostgreSQL typically supports 100-500 max connections). RDS Proxy sits between Lambda and RDS, maintaining a pool of long-lived connections to RDS and efficiently multiplexing many Lambda connections through fewer actual DB connections. It also provides faster failover (maintains connections during Multi-AZ failover) and IAM authentication for database access.

**Q: When should you use SQS Standard vs SQS FIFO?**
A: SQS Standard when: order doesn't matter and consumers are idempotent, you need high throughput (unlimited), and occasional duplicate delivery is acceptable. SQS FIFO when: strict message ordering is required (financial transaction sequences, state machine transitions where order changes the outcome), exactly-once processing is needed (within the 5-minute deduplication window), and throughput fits within limits (3,000/sec with batching). FIFO costs more and has lower throughput — don't use it unless ordering is a hard requirement.

---
---

# APPENDIX: Cross-Topic Quick Reference

## Master Comparison — All DevOps & Infra Concepts

```
┌──────────────────────────┬───────────────────────────────────────────────────────────┐
│ Topic                     │ Core Question It Answers                                    │
├──────────────────────────┼───────────────────────────────────────────────────────────┤
│ Containers & Kubernetes    │ "How do I package applications consistently and manage     │
│                           │ hundreds of them across a cluster automatically?"           │
│ CI/CD & Blue-Green         │ "How do I ship code changes to production frequently,       │
│                           │ safely, with fast rollback when things go wrong?"           │
│ Monitoring, Logging,       │ "How do I KNOW what my distributed system is doing —       │
│ Distributed Tracing        │ when it's healthy, when it's broken, and WHY?"             │
│ AWS Core Services          │ "Which managed cloud services solve which specific          │
│                           │ infrastructure problems without building from scratch?"     │
└──────────────────────────┴───────────────────────────────────────────────────────────┘
```

## How All Four Topics Interconnect

```
A COMPLETE PRODUCTION SYSTEM DELIVERY PIPELINE:

1. DEVELOPER writes code → pushes to GitHub

2. CI/CD (Topic 2):
   GitHub Actions triggers CI pipeline:
   Build Docker image → run tests → push to ECR
   GitOps: Argo CD detects new image tag in Git → deploys to K8s

3. KUBERNETES (Topic 1):
   K8s rolling update / blue-green / canary deployment
   Liveness + readiness probes validate new pods
   HPA auto-scales based on CPU/request metrics
   Service routes traffic only to ready pods

4. AWS INFRASTRUCTURE (Topic 4):
   EC2 nodes (Auto Scaling Group) host K8s worker nodes
   RDS Aurora stores persistent data (with Multi-AZ + Read Replicas)
   S3 stores user uploads + static assets (served via CloudFront CDN)
   SQS decouples async work (email, image processing)
   Lambda processes SQS messages serverlessly

5. OBSERVABILITY (Topic 3):
   Prometheus scrapes metrics from K8s pods
   Fluent Bit ships pod logs → Loki → Grafana
   OTel SDK instruments app → Jaeger traces
   Alertmanager: SLO burn rate alert → PagerDuty
   Engineer: checks Grafana dashboard → Jaeger trace →
     "Payment Service DB query taking 250ms, investigate index"

6. BACK TO CI/CD: Engineer creates PR → back to step 1
   (The entire loop completes — ideally many times per day)
```

## Final Study Tips

```
1. DRAW the Kubernetes architecture (control plane + worker nodes)
   and the complete AWS reference architecture from memory.
   These two diagrams cover the majority of "how would you
   design the infrastructure for X?" questions.

2. ALWAYS CONNECT infrastructure choices to the system design
   requirements they serve:
   "We need high availability for the database → RDS Multi-AZ"
   "We need read scaling → Aurora Read Replicas"
   "We need fast rollback for high-risk deploys → Blue-Green"
   "p99 latency is spiking → check Jaeger traces to find which
    service is slow before jumping to solutions"

3. CONNECT this module to prior notes:
   - K8s HPA = Auto-scaling (Scalability notes) in a container context
   - K8s Services + Ingress = Load Balancing + Reverse Proxy (Scalability)
   - K8s Liveness/Readiness = Health Checks (Scalability — LB topic)
   - S3 Object Storage = Object Storage topic (Databases notes)
   - SQS = Message Queues topic (Messaging notes) — same concepts
   - Prometheus TSDB = Time-series DBs topic (Databases notes)
   - Distributed Tracing propagation = X-Request-Id header
     (Networking Fundamentals — HTTP topic)
   - RDS replication = Replication topic (Databases notes)

4. FOR BFSI/FINTECH (relevant to your Deloitte/interviews):
   - DEPLOYMENT: Blue-green preferred (instant rollback, no mixed
     versions serving transactions)
   - COMPLIANCE: CI/CD with mandatory human approval gate (CAB
     process), full audit trail (Git + pipeline logs)
   - DATA: RDS Aurora in ap-south-1 with Multi-AZ for RBI data
     localization; S3 + Glacier for 7-10 year log retention
   - OBSERVABILITY: Structured logs with trace_id correlation
     mandatory for transaction audit; alert on payment error
     rate SLO breach within 5 minutes
   - SECURITY: Secrets in AWS Secrets Manager (not K8s Secrets
     or environment variables); VPC with private subnets for RDS
     (never publicly accessible); Security Groups as micro-segmentation

5. THE GOLDEN INTERVIEW ANSWER STRUCTURE for "how would you
   deploy/operate X?":
   a) Container: Dockerfile + ECR image
   b) Orchestration: Kubernetes Deployment + Service + Ingress
   c) Deployment strategy: Rolling/Blue-Green/Canary (justify why)
   d) Infrastructure: EC2 ASG or ECS Fargate + RDS + S3 + SQS
   e) Observability: Prometheus metrics + structured logs +
      distributed traces with OTel
   f) CI/CD: GitHub Actions → ECR → Argo CD → K8s
   This structure shows full-stack infrastructure thinking.
```

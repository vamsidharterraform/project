# Real-Time EKS Monitoring Project
## Multi-Cluster Observability with Splunk, Prometheus, Grafana, and AppDynamics

> **Purpose:** Interview preparation for a Senior DevOps / Platform Engineer role. This guide describes a realistic production-style architecture with four Amazon EKS clusters and centralized observability.
>
> **Important:** The cluster names, node counts, workloads, and component placement below are illustrative examples for learning and interview discussion. They are not verified details of any particular production environment.
>
> **Scope:** Splunk, Prometheus, Grafana, and AppDynamics. **OpenTelemetry is intentionally excluded.**

---

## 1. Project Overview

Imagine an enterprise running employee, payment, order, and reporting microservices on AWS EKS. Different teams deploy workloads into different clusters, while the operations team needs a common way to investigate incidents across all clusters.

### Project goals

- Collect application and container logs centrally in **Splunk**.
- Monitor Kubernetes, worker-node, and application metrics using **Prometheus**.
- Build shared dashboards in **Grafana**.
- Investigate application performance and business-transaction latency using **AppDynamics**.
- Troubleshoot incidents across clusters without manually logging in to every worker node.
- Detect missing telemetry as well as application failures.

### Example cluster inventory

| EKS cluster | Purpose | Example worker nodes | Example workloads |
|---|---|---:|---|
| `dev-eks` | Development | 3 | Developer testing and integration |
| `staging-eks` | Pre-production | 3 | Release validation and performance testing |
| `prod-eks-a` | Production workloads | 6 | Employee and order services |
| `prod-eks-b` | Additional production capacity or workload isolation | 3 | Payment and reporting services |
| **Total** | | **15** | |

Each EKS cluster has its own Kubernetes control plane and worker nodes. The example uses 15 worker nodes in total; real sizing depends on application demand, pod requests, availability requirements, and autoscaling policy.

---

## 2. High-Level Architecture

```text
                    AWS ACCOUNT / ORGANIZATION
 ┌──────────────────────────────────────────────────────────────────────┐
 │                                                                      │
 │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
 │  │ dev-eks          │  │ staging-eks     │  │ prod-eks-a          │  │
 │  │                  │  │                 │  │                     │  │
 │  │ Business apps    │  │ Business apps   │  │ Business apps       │  │
 │  │ Splunk collector │  │ Splunk collector│  │ Splunk collector    │  │
 │  │ Node Exporter    │  │ Node Exporter   │  │ Node Exporter       │  │
 │  │ kube-state-mets  │  │ kube-state-mets │  │ kube-state-mets     │  │
 │  │ Prometheus agent │  │ Prometheus agent│  │ Prometheus agent    │  │
 │  │ AppDynamics      │  │ AppDynamics     │  │ AppDynamics         │  │
 │  └────────┬─────────┘  └────────┬────────┘  └──────────┬──────────┘  │
 │           │                     │                      │             │
 │           └─────────────────────┼──────────────────────┘             │
 │                                 │                                    │
 │  ┌─────────────────┐            │                                    │
 │  │ prod-eks-b      │            │                                    │
 │  │                 │            │                                    │
 │  │ Business apps   │            │                                    │
 │  │ Splunk collector├────────────┼───────────┐                        │
 │  │ Node Exporter   ├────────────┼───────┐   │                        │
 │  │ Prometheus agent├────────────┼───┐   │   │                        │
 │  │ AppDynamics     │            │   │   │   │                        │
 │  └─────────────────┘            │   │   │   │                        │
 │                                 │   │   │   │                        │
 │                CENTRAL OBSERVABILITY PLATFORM                        │
 │  ┌──────────────────────────────▼───▼───▼───▼─────────────────────┐  │
 │  │                                                                  │  │
 │  │  ┌──────────────┐  ┌─────────────────────┐  ┌────────────────┐  │  │
 │  │  │ Splunk       │  │ Central metrics      │  │ AppDynamics   │  │  │
 │  │  │ Logs/search  │  │ backend / Prometheus │  │ Controller     │  │  │
 │  │  │ Alerts       │  │ compatible storage   │  │ APM analysis   │  │  │
 │  │  └──────────────┘  └──────────┬──────────┘  └────────────────┘  │  │
 │  │                               │                                  │  │
 │  │                      ┌────────▼────────┐                         │  │
 │  │                      │ Grafana         │                         │  │
 │  │                      │ Shared dashboards│                        │  │
 │  │                      └─────────────────┘                         │  │
 │  └──────────────────────────────────────────────────────────────────┘  │
 └──────────────────────────────────────────────────────────────────────┘
```

**Architecture recommendation:** Collect telemetry close to the workloads in each cluster, then send it securely to centralized services. For larger environments, local Prometheus-compatible collection can remote-write to a centralized durable metrics backend. Grafana queries the backend; it does not normally store the metrics itself.

A central Prometheus server scraping every cluster directly is another option, but it requires reliable cross-cluster connectivity, authentication, service discovery, and network controls.

### Data flow

| Data | Collection method | Destination |
|---|---|---|
| Container logs | Splunk-compatible log collector DaemonSet | Splunk |
| Node CPU, memory, disk, and network | Node Exporter DaemonSet | Prometheus |
| Kubernetes object state | `kube-state-metrics` Deployment | Prometheus |
| Container CPU and memory | Kubelet / cAdvisor metrics | Prometheus |
| Application metrics | Application instrumentation or metrics endpoint | Prometheus |
| Application transactions and performance | AppDynamics application agents and cluster-level integration | AppDynamics Controller |
| Dashboards | Grafana queries configured data sources | Grafana |

---

## 3. DaemonSet vs Deployment vs StatefulSet

This distinction is central to understanding what runs on every node and what does not.

### DaemonSet — node-level coverage

A DaemonSet runs a pod on every **eligible** node, subject to node selectors, affinity, taints, tolerations, architecture, and other scheduling rules.

Typical examples:

- Splunk-compatible log collector
- Prometheus Node Exporter
- Some node-level AppDynamics Machine Agent configurations

### Deployment — replicated service

A Deployment manages a desired number of interchangeable pod replicas. The scheduler places those replicas on eligible nodes; it does not guarantee one pod per node.

Typical examples:

- `kube-state-metrics`
- AppDynamics Cluster Agent
- Grafana
- Application microservices
- Prometheus, depending on the installation design

### StatefulSet — stable identity or storage

A StatefulSet is useful when an application needs stable pod identity, persistent storage, or ordered lifecycle behavior.

Example: a self-hosted Prometheus installation using persistent storage may be deployed as a StatefulSet, depending on the Helm chart and architecture.

### Quick comparison

| Resource | Placement model | Common use |
|---|---|---|
| DaemonSet | One pod per eligible node | Node-level logs and metrics |
| Deployment | Desired number of replicas | Stateless services and agents |
| StatefulSet | Stable pod identity and storage patterns | Stateful services or monitoring servers requiring persistent volumes |
| Job / CronJob | One-off or scheduled execution | Maintenance and batch tasks |

---

## 4. What Runs in Each EKS Cluster?

The recommended baseline is similar across the four clusters. Production clusters may have more replicas, stricter resource limits, or dedicated node groups.

| Component | Typical resource | Scope | Main purpose |
|---|---|---|---|
| Splunk log collector | DaemonSet | Each eligible worker node | Read and forward container logs |
| Node Exporter | DaemonSet | Each eligible worker node | Expose host/OS metrics |
| `kube-state-metrics` | Deployment | One or more replicas per cluster | Expose Kubernetes object-state metrics |
| Prometheus | Deployment, StatefulSet, or agent-style collector | Per the metrics architecture | Scrape, query, and/or forward metrics |
| AppDynamics Cluster Agent | Deployment | Per monitored cluster | Kubernetes-aware monitoring integration |
| AppDynamics Machine Agent | Depends on mode; sometimes DaemonSet | Depends on monitoring requirements | Infrastructure metrics |
| AppDynamics application agent | Runs with/in the supported application runtime | Selected applications | Transaction-level performance |
| Business applications | Usually Deployments | Per application | Serve business requests |
| Grafana | Usually central; optional cluster-local Deployment | Central or per cluster | Dashboards and visualization |

### Example workload layout per cluster

```text
dev-eks
├── Application namespace(s)
│   ├── frontend Deployment
│   ├── backend Deployment
│   └── supporting services
├── Monitoring namespace
│   ├── Splunk log collector DaemonSet
│   ├── Node Exporter DaemonSet
│   ├── kube-state-metrics Deployment
│   └── Prometheus collector/server (architecture dependent)
└── AppDynamics namespace
    └── Cluster Agent Deployment (if enabled)
```

Repeat the same general pattern in `staging-eks`, `prod-eks-a`, and `prod-eks-b`, with differences based on workload, scale, and monitoring policy.

---

## 5. Component Details

### 5.1 Splunk log collector — DaemonSet

**Purpose:** Collect container logs from each eligible worker node and forward them to Splunk.

Common implementation choices include Splunk Connect for Kubernetes or a Fluent Bit-based collector configured for Splunk ingestion.

**How it works:**

1. Application containers write logs to standard output and standard error.
2. The container runtime stores the logs on the worker node.
3. The collector reads the relevant container log files.
4. It enriches events with Kubernetes metadata such as cluster, namespace, pod, and container.
5. It forwards events securely to Splunk.
6. Engineers search for exceptions, errors, request IDs, and deployment-related failures.

Example metadata:

```text
cluster=prod-eks-a
namespace=employee-backend
pod=employee-backend-abc123
container=backend
level=ERROR
```

**Why a DaemonSet?** Each node hosts logs for containers running on that node. A node-level collector can gather them without requiring every application team to deploy its own log-forwarding sidecar.

**Operational caveat:** Collection is not guaranteed simply because the pod is running. Buffering, permissions, filters, network failures, backpressure, and ingestion configuration must also be monitored.

### 5.2 Prometheus Node Exporter — DaemonSet

**Purpose:** Expose operating-system and host-level metrics from worker nodes.

Typical metrics include:

- CPU utilization and CPU time
- Available memory and memory pressure
- Disk space and disk I/O
- Network throughput and errors
- Filesystem utilization

**Example incident:** A node begins running out of disk space. Node Exporter metrics show filesystem usage rising. Prometheus triggers an alert, Grafana displays the affected node, and the engineer investigates container logs and ephemeral-storage usage.

**Why a DaemonSet?** Host metrics are needed from every eligible node, not just from one arbitrarily scheduled pod.

### 5.3 `kube-state-metrics` — Deployment

**Purpose:** Expose metrics about Kubernetes API objects such as pods, Deployments, ReplicaSets, and nodes.

Examples:

- Desired versus available Deployment replicas
- Pod phase and readiness information
- Object state and scheduling-related metrics
- Declared resource requests and limits

**Example:** A Deployment wants three replicas but only two are available. `kube-state-metrics` exposes object-state metrics, Prometheus evaluates an alert, and Grafana displays the replica shortfall.

**Why not a DaemonSet?** It watches Kubernetes API objects and does not need one pod per worker node. One or more replicas per cluster are generally enough, depending on scale and availability requirements.

### 5.4 Prometheus — metrics collection and query engine

Prometheus scrapes metric endpoints, stores time-series data, evaluates alert rules, and supports PromQL queries.

It can collect:

- Node Exporter metrics
- `kube-state-metrics` metrics
- Kubelet and cAdvisor metrics
- Application metrics exposed by endpoints
- Metrics from additional exporters

Possible designs:

| Pattern | How it works | Typical fit |
|---|---|---|
| Prometheus per cluster | Scrapes targets in that cluster | Smaller or isolated environments |
| Prometheus Agent / agent-style collection | Scrapes local targets and remote-writes metrics | Centralized metrics platforms |
| Central Prometheus | Scrapes targets across clusters | Suitable connectivity and manageable scale |
| HA Prometheus pairs plus remote storage | Multiple collectors with durable backend | Larger, availability-sensitive environments |

A Prometheus server is not normally a DaemonSet. It is commonly a Deployment or StatefulSet, depending on packaging and storage requirements.

### 5.5 AppDynamics Cluster Agent — Deployment

**Purpose:** Provide Kubernetes-aware monitoring integration and collect cluster-level information for AppDynamics, depending on the installed version and configuration.

It can help provide visibility into:

- Kubernetes workloads and namespaces
- Application-to-container or workload relationships
- Deployment and cluster context
- Kubernetes metadata associated with monitored applications

It is typically a Deployment rather than one pod per worker node.

### 5.6 AppDynamics Machine Agent — optional node-level component

The Machine Agent can collect infrastructure-level metrics. Its placement depends on the supported installation and monitoring mode. Some Kubernetes designs deploy it as a DaemonSet to monitor every worker node; other designs use a different deployment model.

Do not assume every AppDynamics installation requires both a Cluster Agent and a Machine Agent DaemonSet. Confirm the deployed architecture and supported configuration for the AppDynamics version in use.

### 5.7 AppDynamics application agent — application runtime

The application agent instruments a supported application runtime to capture application performance and transaction details.

For example, a Java backend may use a Java agent to monitor HTTP transactions, database calls, and downstream service calls.

This is **not the same as a DaemonSet**. The agent generally runs as part of the application process/container configuration or another supported application-specific arrangement.

It can help answer:

- Which business transaction is slow?
- Is the delay inside the application or in a downstream dependency?
- Which database call contributes most to response time?
- What are transaction response-time percentiles such as p95?

The detail available depends on instrumentation, configuration, framework support, and licensing.

### 5.8 Business applications — Deployments

Example service names:

```text
employee-frontend       Deployment
employee-backend        Deployment
payment-service         Deployment
order-service           Deployment
reporting-service       Deployment
```

These names are illustrative. Each application has its own replica count, resource requests and limits, health probes, configuration, and scaling policy.

Monitoring components observe the business workloads; they do not replace the workloads themselves.

---

## 6. Example: How Many Monitoring Pods Across the Clusters?

Assume the example has 15 worker nodes total and all are eligible for both the Splunk collector and Node Exporter.

| Component | Placement model | Illustrative total |
|---|---|---:|
| Splunk collector | DaemonSet, one per eligible node | Up to 15 pods |
| Node Exporter | DaemonSet, one per eligible node | Up to 15 pods |
| `kube-state-metrics` | One replica per cluster | About 4 pods |
| AppDynamics Cluster Agent | One replica per cluster, if configured that way | About 4 pods |
| Grafana | Centralized deployment | Depends on HA/replica design |
| Central metrics backend | Centralized service | Depends on product and HA design |

These counts are examples, not fixed requirements. If a DaemonSet is not eligible for a node, its pod will not run there. If `kube-state-metrics` has two replicas per cluster, the total would be about eight rather than four.

### What runs on one worker node?

Assume `prod-eks-a` has six worker nodes. One example node might host:

```text
Worker node: prod-node-01
├── Splunk log collector          [DaemonSet pod]
├── Node Exporter                 [DaemonSet pod]
├── employee-backend replica A    [Deployment pod]
├── employee-backend replica B    [Deployment pod]
└── employee-frontend replica     [Deployment pod]
```

Other system pods may also run there, such as networking, DNS, storage, and security agents. Their presence depends on the cluster design and scheduling configuration.

The node does **not** need to run all Splunk, Prometheus, Grafana, and AppDynamics services. It runs the pods scheduled to it. Some components run once per eligible node, some have a few replicas per cluster, and others run centrally outside EKS.

---

## 7. Centralized vs Cluster-Local Components

| Component | Typical location | Responsibility |
|---|---|---|
| Splunk log collectors | Every EKS cluster | Read and forward node-local container logs |
| Splunk indexing/search platform | Central platform | Store, search, correlate, and alert on logs |
| Node Exporter | Every EKS cluster | Expose node metrics |
| `kube-state-metrics` | Every EKS cluster | Expose Kubernetes object-state metrics |
| Prometheus collectors | Every EKS cluster or central scraper | Scrape and evaluate metrics |
| Central metrics backend | Central platform | Retain and query metrics from multiple clusters |
| Grafana | Usually central | Shared dashboards across clusters |
| AppDynamics Cluster Agent | Monitored EKS clusters | Kubernetes-aware monitoring integration |
| AppDynamics application agents | Selected application runtimes | Application and transaction performance data |
| AppDynamics Controller | Central platform or vendor-managed service | Central APM UI and analysis |

### Centralization does not mean central collection

- A centralized Grafana instance does not automatically store metrics. Grafana normally queries data sources.
- A centralized Splunk platform does not mean log collectors run centrally. Node-level collectors run in each cluster and forward logs to Splunk.
- Central Prometheus scraping is possible, but remote-write from local collectors can be operationally easier at larger scale.
- Central services need secure connectivity, authentication, capacity planning, and availability design.

---

## 8. Commands to Inspect the Actual Cluster

Use commands to confirm the deployed resources rather than assuming that the architecture diagram matches the real cluster.

### A. List EKS clusters

```bash
aws eks list-clusters --region us-west-2
```

This lists clusters in the selected region that the AWS caller can access.

### B. Connect to the intended cluster

```bash
aws eks update-kubeconfig \
  --region us-west-2 \
  --name prod-eks-a

kubectl config current-context
kubectl get nodes -o wide
```

Repeat for each cluster as needed. Verify the current context before making changes.

### C. List DaemonSets

```bash
kubectl get daemonsets -A
```

Inspect a specific DaemonSet:

```bash
kubectl describe daemonset <daemonset-name> -n <namespace>
```

Check desired and ready pod counts:

```bash
kubectl get daemonsets -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,DESIRED:.status.desiredNumberScheduled,CURRENT:.status.currentNumberScheduled,READY:.status.numberReady'
```

This helps identify whether a collector is running on all intended nodes.

### D. List Deployments and StatefulSets

```bash
kubectl get deployments -A
kubectl get statefulsets -A
```

Inspect replicas and availability:

```bash
kubectl describe deployment <deployment-name> -n <namespace>
```

### E. See pods and the nodes hosting them

```bash
kubectl get pods -A -o wide
```

Find pods running on a particular node:

```bash
kubectl get pods -A -o wide \
  --field-selector spec.nodeName=<node-name>
```

This is useful when investigating a busy node or a missing collector.

### F. Find monitoring components by name

```bash
kubectl get pods -A | grep -Ei \
  'splunk|fluent-bit|prometheus|node-exporter|kube-state-metrics|grafana|appdynamics'
```

Names vary by Helm chart, operator, namespace, and product version. This is a starting point, not proof that all expected components are present.

### G. Check resource consumption

```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

`kubectl top` requires a functioning metrics source, usually Metrics Server. It is useful for current resource usage; Prometheus is more suitable for historical time-series analysis and alerting.

---

## 9. Real-World Incident Scenarios

### Scenario 1: Application latency is high

**Symptoms:** Users report slow API responses, but the pods are running.

1. Open Grafana and examine CPU, memory, restarts, node pressure, and request metrics.
2. Use AppDynamics to identify slow business transactions, time spent in the application, and slow downstream calls.
3. Search Splunk for errors, timeouts, correlation IDs, and exceptions.
4. Check affected pods and nodes with `kubectl get pods -A -o wide` and `kubectl top pods -A`.
5. Determine whether the issue is application code, database latency, networking, resource saturation, or an overloaded dependency.
6. Apply the appropriate fix and verify recovery using the same metrics and transaction measurements.

**Interview answer:** “I correlate metrics, logs, and application-performance data instead of assuming high latency is caused by CPU alone.”

### Scenario 2: A pod repeatedly restarts

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Investigate:

- `OOMKilled` and memory limits
- Liveness and readiness probe failures
- Application exceptions
- Missing configuration or secrets
- Database/API connectivity
- Scheduling and node pressure

Prometheus helps show resource trends, Splunk helps search logs, and AppDynamics may provide application-level context.

### Scenario 3: Logs are missing from Splunk

Check the collector before assuming the application stopped logging.

```bash
kubectl get daemonsets -A
kubectl get pods -A | grep -Ei 'splunk|fluent-bit'
kubectl logs <collector-pod> -n <namespace>
kubectl describe pod <collector-pod> -n <namespace>
```

Investigate:

- Collector readiness and restarts
- File access and log-path configuration
- Filters that might drop events
- Network connectivity and authentication
- Buffer saturation or backpressure
- Index, source, and metadata configuration in Splunk

### Scenario 4: A worker node is under pressure

```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu

kubectl get pods -A -o wide \
  --field-selector spec.nodeName=<node-name>

kubectl describe node <node-name>
```

Use Prometheus and Grafana to inspect CPU, memory, disk, and network trends. Check pod requests and limits, ephemeral storage, scheduling distribution, and node-level logs.

Identify whether the problem is a resource-heavy workload, excessive pod density, or insufficient node capacity before deciding whether to scale pods, rebalance workloads, or add worker capacity.

### Scenario 5: One cluster cannot send telemetry centrally

Check the local collectors first, then the connection to the destination.

- Are collector pods ready?
- Are credentials valid?
- Can the cluster reach the central endpoint?
- Are security groups, routing, DNS, and egress policies correct?
- Are there TLS or certificate errors?
- Is the central ingestion endpoint healthy?

This separates a central-platform outage from a cluster-specific telemetry-delivery issue.

---

## 10. Common Interview Questions and Answers

### Q1. Why do we use a DaemonSet for log collection?

Each worker node can host container log files for the pods running on it. A DaemonSet provides a collector on every eligible node without requiring a separate collector Deployment for every application.

### Q2. Why isn't Prometheus a DaemonSet?

Prometheus is usually a metrics scraper and time-series database or collector. It does not normally need one full server per node. A Prometheus server or agent can scrape metrics from many targets.

### Q3. Does Grafana collect metrics?

Grafana primarily queries data sources and visualizes their results. Prometheus or another metrics backend stores and serves the metrics.

### Q4. What is the difference between Node Exporter and `kube-state-metrics`?

Node Exporter exposes operating-system and host metrics. `kube-state-metrics` exposes metrics about Kubernetes objects and their declared or observed state.

### Q5. Does the AppDynamics Cluster Agent replace the application agent?

No. The Cluster Agent provides Kubernetes-aware monitoring integration. The application agent instruments supported application runtimes to provide transaction-level performance details.

### Q6. If we have 15 worker nodes, will we have 15 Prometheus pods?

Not necessarily. If Prometheus is deployed as one server per cluster, there could be four server instances in this example. With agent mode, the design may differ. Node Exporter and a node-level log collector would each have up to 15 pods if all 15 nodes are eligible.

### Q7. If a pod is healthy but the application is slow, what do you check?

I compare request latency and application transaction data with resource metrics, downstream dependency latency, and logs. A healthy pod means its health checks pass; it does not prove that the user-facing request is fast.

### Q8. How do you reduce monitoring overhead?

Tune scrape intervals and metric retention, avoid excessive high-cardinality labels, control log volume, configure sensible resource requests and limits, and use sampling or filtering where appropriate. Preserve enough telemetry to troubleshoot production incidents.

---

## 11. Production Design Considerations

- **High availability:** Replicate critical central services and avoid single points of failure in metrics and log ingestion.
- **Security:** Use least-privilege IAM, TLS, secure credential management, and controlled network paths between clusters and central services.
- **Resource management:** Give collectors appropriate requests and limits. Monitor their CPU, memory, disk, and dropped-event metrics.
- **Retention and cost:** Set log and metrics retention according to incident investigation and compliance needs.
- **Alerting:** Alert on missing telemetry as well as application symptoms. A monitoring system that silently stops collecting data is itself an operational risk.
- **Cluster upgrades:** Verify DaemonSet compatibility, collector health, and telemetry continuity during node rotations and Kubernetes upgrades.
- **Cardinality:** Avoid unbounded metric labels such as unique request IDs, which can cause Prometheus storage and query-performance problems.
- **Ownership:** Define which team maintains cluster agents, central backends, dashboards, alerts, and application instrumentation.
- **Network resilience:** Design for temporary loss of connectivity, buffering, retry behavior, and controlled recovery without losing large volumes of telemetry.
- **Access control:** Restrict access to sensitive logs, application traces, and dashboards according to role and environment.

---

## 12. Final Interview Summary

Use this answer as a concise explanation of the architecture. Adjust it to match your actual experience.

> “In this example architecture, we have four EKS clusters for development, staging, and production workloads. Each cluster runs node-level monitoring components such as a Splunk-compatible log collector and Prometheus Node Exporter as DaemonSets, so each eligible worker node gets a collector and host-metrics exporter.
>
> We deploy `kube-state-metrics` as a Deployment to expose Kubernetes object-state metrics. Prometheus collects node, Kubernetes, and application metrics, with metrics forwarded to a centralized backend. Grafana provides common dashboards across clusters.
>
> For application performance, AppDynamics uses its Kubernetes integration and application agents to correlate workload information with transaction latency and downstream dependencies. Splunk centralizes logs for searching and alerting.
>
> When troubleshooting, I correlate Grafana metrics, Splunk logs, and AppDynamics transaction data, then validate the issue using Kubernetes commands such as `kubectl describe`, `kubectl logs --previous`, `kubectl top`, and `kubectl get pods -o wide`. The exact pod counts depend on node counts, replicas, and scheduling configuration.”

### Three rules to remember

1. **DaemonSet = one pod per eligible node.**
2. **Deployment = a desired number of replicas scheduled across eligible nodes.**
3. **Centralized monitoring = centralized visibility, not necessarily centralized collection.**

### Interview integrity

Be explicit about which parts you personally implemented and which parts are a proposed architecture. Do not present illustrative cluster counts or components as verified production facts.

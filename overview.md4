Real-Time EKS Monitoring Project — Interview Preparation Guide
We’ll frame this as a realistic production DevOps project with 4 EKS clusters, centralized Splunk, Prometheus, Grafana, and AppDynamics — no OpenTelemetry.
The important interview distinction is between what runs in each Kubernetes cluster, what runs on each worker node, and what is centralized outside the clusters. Not every component runs on every node.
The cluster counts and node counts below are an illustrative production design for interview preparation, not a claim about your actual environment.
1. Project overview

Production Kubernetes Observability
AWS EKS · Multi-cluster · Centralized monitoring


4
EKS clusters


15
Example worker nodes


4
Core observability tools


1
Central observability platform




Business scenario
Assume a company runs employee, payment, order, and reporting microservices on AWS EKS. Different teams deploy applications into different clusters, but the operations team needs a single place to investigate incidents.
The goals are to:
- Collect application and container logs centrally in Splunk.
- Monitor Kubernetes, node, and application metrics using Prometheus.
- Build centralized dashboards in Grafana.
- Trace application performance problems and identify slow business transactions using AppDynamics.
- Troubleshoot incidents across all four clusters without logging in to every node individually.
Example cluster inventory
EKS cluster	Purpose	Example worker nodes	Example workloads
dev-eks	Development	3	Developer testing, integration
staging-eks	Pre-production	3	Release validation, performance testing
prod-eks-a	Production workloads	6	Employee and order services
prod-eks-b	Additional production capacity / isolation	3	Payment and reporting services
Total		15	
Important: These are illustrative node counts. Actual production sizing depends on application load, pod requests, availability requirements, and scaling policies. Each EKS cluster has its own Kubernetes control plane and worker nodes.

2. High-level architecture
AWS environment
4 independent EKS clusters

Cluster 1
dev-eks

Applications
Log collectors (DaemonSet)
Node metrics (DaemonSet)
Metrics collector (local)


Cluster 2
staging-eks

Applications
Log collectors (DaemonSet)
Node metrics (DaemonSet)
Metrics collector (local)


Cluster 3
prod-eks-a

Applications
Log collectors (DaemonSet)
Node metrics (DaemonSet)
Metrics collector (local)


Cluster 4
prod-eks-b

Applications
Log collectors (DaemonSet)
Node metrics (DaemonSet)
Metrics collector (local)



Secure connectivity · authentication · TLS · network policies

Central observability platform
Splunk
Logs, search, alerts


Prometheus / metrics backend
Metrics, queries, alert rules


Grafana
Dashboards, visualization


AppDynamics
Application performance, transactions





How the data flows
Data	Collection method	Destination
Container logs	Splunk-compatible log collector DaemonSet	Splunk
Node CPU, memory, disk, network	Node Exporter DaemonSet	Prometheus
Kubernetes object state	kube-state-metrics Deployment	Prometheus
Container CPU and memory	Kubelet / cAdvisor metrics	Prometheus
Application metrics	Application instrumentation or metrics endpoint	Prometheus
Application performance and business transactions	AppDynamics application agents and cluster-level components	AppDynamics Controller
Dashboards	Grafana queries configured data sources	Grafana UI
Architecture recommendation: Deploy local metrics collection in each cluster and send metrics to a centralized, durable metrics backend. Grafana can query that backend centrally. For larger environments, this may be a Prometheus-compatible remote-write backend such as Amazon Managed Service for Prometheus or a horizontally scalable Prometheus-compatible system.
A single central Prometheus server scraping every cluster directly is also possible, but it requires reliable cross-cluster connectivity, authentication, service discovery, and appropriate network controls.

3. The most important interview concept: DaemonSet vs Deployment vs StatefulSet

DaemonSet — node-level coverage
Runs a pod on every eligible node, subject to selectors, taints, tolerations, and other scheduling rules.
Examples: Splunk log collector, Node Exporter.




Deployment — replicated application services
Runs a desired number of interchangeable pod replicas. Those pods can be scheduled across eligible nodes.
Examples: kube-state-metrics, AppDynamics Cluster Agent, Grafana.




StatefulSet — stable identity or persistent state
Manages pods with stable identities and is useful when an application needs persistent storage or ordered lifecycle behavior.
Example: a self-hosted Prometheus deployment using persistent storage may use a StatefulSet, depending on the chart and architecture.



What runs on each node?
For a simple example, assume prod-eks-a has six worker nodes.
Component	Kubernetes resource	Approximate pod placement
Splunk log collector	DaemonSet	1 per eligible node → up to 6
Node Exporter	DaemonSet	1 per eligible node → up to 6
Application pods	Deployment	Depends on replica count and scheduling
kube-state-metrics	Deployment	Usually 1 or more replicas per cluster, not one per node
AppDynamics Cluster Agent	Deployment	Usually one or a small number of replicas per cluster
Prometheus	Deployment or StatefulSet, or agent configuration	Depends on metrics architecture
Grafana	Deployment	Central platform, or optional cluster-local instance
The counts above assume each worker node is eligible and the relevant DaemonSet has no additional scheduling restrictions. System nodes, taints, architecture differences, and tolerations can change the actual count.
Interview answer: A DaemonSet gives node-level coverage; a Deployment gives replica-based application or service coverage. A Deployment does not automatically create one pod on every node.

4. What runs inside each of the four EKS clusters?
The recommended baseline is similar across all four clusters. Production may have more replicas, stronger resource limits, or separate node groups.
4.1 Splunk log collector — DaemonSet









Purpose: Collect container logs from each eligible worker node and forward them to Splunk.
Common implementation choices include Splunk Connect for Kubernetes or a Fluent Bit-based collector configured for Splunk ingestion.


How it works
1. Application containers write logs to standard output and standard error.
2. Kubernetes container runtime stores the logs on the worker node.
3. The log collector pod reads the relevant container log files.
4. It enriches records with Kubernetes metadata, such as cluster, namespace, pod, and container.
5. It forwards the records securely to Splunk.
6. Operations engineers search for errors, exceptions, request IDs, and deployment-related failures.
Example labels to include in log events:
cluster=prod-eks-a
namespace=employee-backend
pod=employee-backend-abc123
container=backend
level=ERROR


Why a DaemonSet? Each node hosts logs from the containers running on that node. A node-level collector can collect those logs without requiring every application team to install a separate log-forwarding sidecar.
Important: A collector is not guaranteed to receive every log. Buffering, permissions, filters, network failures, and backpressure must be configured and monitored.
4.2 Prometheus Node Exporter — DaemonSet









Purpose: Expose operating-system metrics from worker nodes.
Node Exporter exposes host metrics; it does not replace Kubernetes object metrics or application instrumentation.


Typical metrics:
- CPU utilization and CPU time.
- Available memory and memory pressure.
- Disk space and disk I/O.
- Network throughput and errors.
- Filesystem utilization.
Example incident: A node starts running out of disk space. Node Exporter shows the filesystem usage rising. Prometheus triggers an alert, and Grafana displays the affected node. The engineer checks node logs, container logs, and ephemeral-storage usage to identify the cause.
Interview detail: Node Exporter is generally a DaemonSet because you want host metrics from each eligible node, rather than from a single pod on an arbitrary node.
4.3 kube-state-metrics — Deployment
Purpose: Expose metrics about Kubernetes API objects. It reports what Kubernetes knows about objects such as pods, Deployments, ReplicaSets, and nodes.
Examples include:
- Desired versus available Deployment replicas.
- Pod phase and readiness information.
- Restart and scheduling-related object state.
- Resource requests and limits declared on workloads.
Example: A Deployment wants three replicas, but only two are available. kube-state-metrics exposes the relevant object-state metrics, Prometheus evaluates an alert, and Grafana shows the replica shortfall.
Why not a DaemonSet? kube-state-metrics watches Kubernetes API objects. It does not need to run once per worker node. One or more replicas per cluster are generally sufficient, depending on scale and deployment configuration.
4.4 Prometheus — metrics collection and query engine
Prometheus scrapes metric endpoints, stores time-series data, evaluates alert rules, and exposes PromQL queries.
It can collect:
- Node Exporter metrics.
- kube-state-metrics metrics.
- Kubelet and cAdvisor metrics.
- Application metrics exposed by application endpoints.
- Additional exporter metrics.
Possible deployments
Pattern	How it works	Best fit
Prometheus per cluster	Each cluster scrapes local targets	Smaller deployments or isolated clusters
Prometheus Agent / agent-style collection	Scrapes local targets and remote-writes metrics	Centralized metrics platforms
Central Prometheus	Scrapes targets across clusters	Environments with suitable connectivity and manageable scale
HA Prometheus pairs plus remote storage	Multiple collectors and a durable backend	Larger, availability-sensitive environments
A Prometheus server is not normally installed as a DaemonSet. It is usually a Deployment or StatefulSet, depending on its packaging and storage needs.
4.5 AppDynamics Cluster Agent — Deployment
Purpose: Provide Kubernetes-aware monitoring and collect cluster-level information for AppDynamics, depending on the installed version and configuration.
It can help provide visibility into:
- Kubernetes workloads and namespaces.
- Application-to-container or workload relationships.
- Deployment and cluster context for application performance.
- Kubernetes metadata associated with monitored applications.
It is typically installed as a Deployment, not one pod per node.
4.6 AppDynamics Machine Agent — optional node-level component
The Machine Agent can collect infrastructure-level metrics. Its placement depends on the supported installation and monitoring mode. In some Kubernetes designs it is deployed as a DaemonSet to monitor each worker node; other designs use different deployment models.
Do not assume that every AppDynamics installation requires both a Cluster Agent and a Machine Agent DaemonSet. Confirm the actual deployment and supported architecture for the AppDynamics version in use.
4.7 AppDynamics application agent — application runtime
The application agent instruments a supported application runtime to capture application performance and transaction details.
For example, a Java backend may use a Java agent configured to monitor HTTP transactions, database calls, and downstream service calls.
This is not the same as a DaemonSet. The agent generally runs as part of the application process/container configuration or in another supported application-specific arrangement.
It can help answer questions such as:
- Which business transaction is slow?
- Is the delay inside the application or a downstream dependency?
- Which database call contributes most to the response time?
- What are the transaction response-time percentiles, such as p95?
The level of detail depends on instrumentation, agent configuration, application framework, and licensing.
4.8 Application workloads — Deployments
Your actual business services run as Deployments or other workload resources:
employee-frontend       Deployment
employee-backend        Deployment
payment-service         Deployment
order-service           Deployment
reporting-service       Deployment


These names are examples. Each service has its own replica count, resource requests and limits, health probes, configuration, and scaling policy.
The monitoring components observe these workloads; they do not replace the application workloads themselves.

5. Per-cluster inventory: what would we expect?
This is an example of the expected workload layout, not a literal inventory of your clusters.
Component	dev-eks	staging-eks	prod-eks-a	prod-eks-b
Splunk log collector	DaemonSet	DaemonSet	DaemonSet	DaemonSet
Node Exporter	DaemonSet	DaemonSet	DaemonSet	DaemonSet
kube-state-metrics	Deployment	Deployment	Deployment	Deployment
Prometheus collector	Local server or agent	Local server or agent	Local server or agent	Local server or agent
AppDynamics Cluster Agent	Deployment, if enabled	Deployment, if enabled	Deployment	Deployment
AppDynamics Machine Agent	Optional	Optional	Per monitoring design	Per monitoring design
Application agents	As needed	As needed	As needed	As needed
Business applications	Development services	Release candidates	Production services	Production services
Grafana	Usually centralized	Usually centralized	Usually centralized	Usually centralized
What does this mean in practice?
If there are 15 worker nodes across all four clusters and both the Splunk collector and Node Exporter run on every eligible worker node:
- Splunk collector: up to 15 pods across the four clusters.
- Node Exporter: up to 15 pods across the four clusters.
These are not 15 pods in every cluster. The total is calculated from the eligible nodes across all clusters.
If kube-state-metrics has one replica per cluster, that would be approximately four pods total. A Deployment with two replicas in each cluster would instead have approximately eight pods total.
The actual counts depend on replica settings, scheduling, and whether additional collectors are installed.

6. What runs centrally versus inside EKS?
Tool/component	Typical location	Responsibility
Splunk log collectors	Each EKS cluster	Read and forward node-local container logs
Splunk indexing/search platform	Central platform	Store, search, correlate, and alert on logs
Node Exporter	Each EKS cluster	Expose node metrics
kube-state-metrics	Each EKS cluster	Expose Kubernetes object-state metrics
Prometheus collectors	Each EKS cluster, or central scraper	Scrape and evaluate metrics according to architecture
Central metrics backend	Central platform	Retain and query metrics from multiple clusters
Grafana	Usually central platform	Dashboards and visualizations across clusters
AppDynamics Cluster Agent	Monitored EKS clusters	Kubernetes-aware monitoring integration
AppDynamics application agents	Application runtime	Application and transaction performance data
AppDynamics Controller	Central platform or vendor-managed service	Central application-performance UI and analysis
Important centralization distinction
A centralized Grafana instance does not automatically mean all metrics are stored in Grafana. Grafana normally queries data sources.
Likewise, a centralized Splunk platform does not mean the log collector needs to run centrally. Node-level collectors can run inside each EKS cluster and send logs to the central platform.

7. Example: how many pods are running on each node?
Assume one worker node is running three application pods and two monitoring DaemonSets.
Worker node: prod-node-01
Example only

Splunk collector
DaemonSet pod


Node Exporter
DaemonSet pod


Backend replica A
Deployment pod


Backend replica B
Deployment pod


Frontend replica
Deployment pod



Other system pods may also run on the node, including networking, DNS, storage, and security agents, depending on the cluster design.

The node is not responsible for running all Prometheus, Grafana, Splunk, and AppDynamics services. It runs the pods scheduled to that node. Some components run once per eligible node, some have a few replicas per cluster, and others run centrally outside the cluster.

8. Commands to inspect the actual cluster
Use these commands during an interview or a real incident to prove what is deployed rather than guessing from the architecture diagram.
A. List clusters
aws eks list-clusters --region us-west-2


This lists EKS clusters in the selected AWS region that the caller can access.
B. Connect to the intended cluster
aws eks update-kubeconfig \
  --region us-west-2 \
  --name prod-eks-a

kubectl config current-context
kubectl get nodes -o wide


Repeat for each cluster when necessary. Verify the context before making changes.
C. List DaemonSets
kubectl get daemonsets -A


Inspect details:
kubectl describe daemonset <daemonset-name> -n <namespace>


Check the desired and available pod counts:
kubectl get daemonsets -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,DESIRED:.status.desiredNumberScheduled,CURRENT:.status.currentNumberScheduled,READY:.status.numberReady'


This is a useful way to check whether a collector is running on all intended nodes.
D. List Deployments and StatefulSets
kubectl get deployments -A
kubectl get statefulsets -A


Inspect replicas and availability:
kubectl get deployments -A
kubectl describe deployment <deployment-name> -n <namespace>


E. See pods and the nodes hosting them
kubectl get pods -A -o wide


Find pods running on a particular node:
kubectl get pods -A -o wide \
  --field-selector spec.nodeName=<node-name>


This command is particularly useful when investigating a busy node.
F. Find monitoring components
kubectl get pods -A | grep -Ei \
  'splunk|fluent-bit|prometheus|node-exporter|kube-state-metrics|grafana|appdynamics'


Names vary by Helm chart, operator, namespace, and product version. A search by name is a starting point, not proof that all monitoring components are present.
G. Check resource consumption
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory


kubectl top requires a functioning metrics source, usually Metrics Server. It is useful for short-term resource usage, while Prometheus is better suited to time-series monitoring and historical investigation.

9. Real-world incident scenarios
Scenario 1: Application latency is high
Symptoms: Users report slow API responses, but pods are running.
1. Open Grafana and examine CPU, memory, restarts, node pressure, and request metrics.
2. Use AppDynamics to identify the slow business transaction, time spent in the application, and slow downstream calls.
3. Search Splunk for errors, timeouts, correlation IDs, and exceptions.
4. Check the affected pods and nodes using kubectl get pods -A -o wide and kubectl top pods -A.
5. Determine whether the issue is application code, database latency, network problems, saturation, or an overloaded dependency.
6. Apply the appropriate fix and verify recovery using the same metrics and transaction measurements.
Interview answer: “I correlate metrics, logs, and application performance data rather than assuming high latency is caused by CPU alone.”
Scenario 2: A pod is repeatedly restarting
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.lastTimestamp


Investigate:
- OOMKilled and memory limits.
- Liveness and readiness probe failures.
- Application exceptions.
- Missing configuration or secrets.
- Downstream database/API connectivity.
- Scheduling and node pressure.
Prometheus helps show resource trends, Splunk helps search logs, and AppDynamics may provide application-level context.
Scenario 3: Logs are missing from Splunk
Check the collector before assuming the application stopped logging.
kubectl get daemonsets -A
kubectl get pods -A | grep -Ei 'splunk|fluent-bit'
kubectl logs <collector-pod> -n <namespace>
kubectl describe pod <collector-pod> -n <namespace>


Investigate:
- Collector pod readiness and restarts.
- File access and log path configuration.
- Filters that might drop events.
- Network connectivity and authentication.
- Buffer saturation or backpressure.
- Index, source, and metadata configuration in Splunk.
Scenario 4: A worker node is under pressure
kubectl top nodes
kubectl top pods -A --sort-by=cpu

kubectl get pods -A -o wide \
  --field-selector spec.nodeName=<node-name>

kubectl describe node <node-name>


Use Prometheus and Grafana to inspect CPU, memory, disk, and network trends. Check pod requests and limits, ephemeral storage, scheduling distribution, and node-level logs.
If the node is overloaded, identify whether the problem is a resource-heavy workload, excessive pod density, or insufficient node capacity before deciding whether to scale pods, rebalance workloads, or add worker capacity.
Scenario 5: One cluster cannot send telemetry centrally
Check the local collectors first, then the connection to the destination.
- Are collector pods ready?
- Are their credentials valid?
- Can the cluster reach the central endpoint?
- Are security groups, routing, DNS, and egress policies correct?
- Are there TLS or certificate errors?
- Is the central ingestion endpoint healthy?
This is a good interview example because a failure in the central platform and a failure in a single cluster require different troubleshooting paths.

10. Common interview questions and strong answers
Q1. Why do we use a DaemonSet for log collection?
Because each worker node can host container log files for the pods running on it. A DaemonSet provides a collector on every eligible node without requiring a separate collector Deployment for every application.

Q2. Why isn't Prometheus a DaemonSet?
Prometheus is usually a metrics scraper and time-series database or collector. It does not normally need one full server per node. A Prometheus server or agent can scrape metrics from many targets.

Q3. Does Grafana collect metrics?
Grafana primarily queries data sources and visualizes their results. Prometheus or another metrics backend stores and serves the metrics.

Q4. What is the difference between Node Exporter and kube-state-metrics?
Node Exporter exposes operating-system and hardware-related metrics. kube-state-metrics exposes metrics about Kubernetes objects and their declared or observed state.

Q5. Does the AppDynamics Cluster Agent replace the application agent?
No. The Cluster Agent provides Kubernetes-aware monitoring integration. The application agent instruments supported application runtimes to provide transaction-level performance details.

Q6. If we have 15 worker nodes, will we have 15 Prometheus pods?
Not necessarily. If Prometheus is deployed as one server per cluster, there could be four server instances in this example. If using agent mode, the design may differ. Node Exporter and a node-level log collector would each have up to 15 pods if all 15 nodes are eligible.

Q7. If a pod is healthy but the application is slow, what do you check?
I compare request latency and application transaction traces with resource metrics, downstream dependency latency, and logs. A healthy pod only means its health checks pass; it does not prove that the user-facing request is fast.

Q8. How do you reduce monitoring overhead?
Tune scrape intervals and metric retention, avoid excessive high-cardinality labels, control log volume, configure sensible resource requests and limits, and use sampling or filtering where appropriate. Maintain enough telemetry to troubleshoot production incidents.


11. Production design considerations
Before calling this a production-ready architecture, account for the following:
- High availability: Replicate critical central services and avoid a single point of failure in the metrics and log ingestion paths.
- Security: Use least-privilege IAM, TLS, secure credential management, and controlled network paths between clusters and central services.
- Resource management: Give collectors appropriate requests and limits; monitor their own CPU, memory, disk, and dropped-event metrics.
- Retention and cost: Set log and metrics retention based on incident investigation and compliance requirements.
- Alerting: Alert on missing telemetry as well as application symptoms. A monitoring system that silently stops collecting data is itself an operational risk.
- Cluster upgrades: Verify DaemonSet compatibility, collector health, and metrics/log continuity during node rotations and Kubernetes upgrades.
- Cardinality: Avoid unbounded metric labels such as unique request IDs, which can cause Prometheus storage and query performance problems.
- Ownership: Define which team maintains cluster agents, central backends, dashboards, alerts, and application instrumentation.

12. Final interview summary
A concise answer you can give in an interview:
“In our example architecture, we have four EKS clusters for development, staging, and production workloads. Each cluster runs node-level monitoring components such as a Splunk-compatible log collector and Prometheus Node Exporter as DaemonSets, so each eligible worker node gets a collector and host-metrics exporter.
We deploy kube-state-metrics as a Deployment to expose Kubernetes object-state metrics. Prometheus collects node, Kubernetes, and application metrics, with metrics forwarded to a centralized backend. Grafana provides a common dashboard across clusters.
For application performance, AppDynamics uses its Kubernetes integration and application agents to correlate workload information with transaction latency and downstream dependencies. Splunk centralizes logs for searching and alerting.
When troubleshooting, I correlate Grafana metrics, Splunk logs, and AppDynamics transaction data, then validate the issue using Kubernetes commands such as kubectl describe, kubectl logs --previous, kubectl top, and kubectl get pods -o wide. The exact pod counts depend on node counts, replicas, and scheduling configuration.”

Remember these three rules:
1. DaemonSet = one pod per eligible node.
2. Deployment = a desired number of replicas, scheduled across eligible nodes.
3. Centralized monitoring = centralized visibility, not necessarily centralized collection.
For a real interview, be explicit about which parts you personally implemented and which parts are the proposed architecture. Never present illustrative cluster counts or components as verified production facts.
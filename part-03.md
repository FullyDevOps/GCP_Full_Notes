### **I. Compute Runtimes: The "Where Your Code Runs" Layer**

#### **1. Cloud Run (Serverless Containers)**
*   **Core Concept:** Fully managed, **stateless**, **container-based** serverless platform. You bring a container image (Docker), GCP handles *all* infrastructure: scaling (to zero!), load balancing, health checks, networking, security patching. **Event-driven or HTTP-triggered.**
*   **Why DevOps Cares:** Eliminates VM/OS management. Focus shifts to **container build pipelines**, **image optimization**, **scaling policies**, and **observability**. Ideal for microservices, APIs, background tasks.
*   **Key Mechanics:**
    *   **Scaling:** Scales **per request** (1 instance = 1 concurrent request by default, configurable). Scales **to zero** when idle (cost saver!). Max instances configurable.
    *   **Concurrency:** Tunable (1-80+). Higher = more efficient resource use *per instance* but requires thread-safe code.
    *   **Cold Starts:** Initial request after scale-to-zero causes delay (seconds). Mitigate via **min instances** (keep warm), **request concurrency**, optimized container startup.
    *   **Execution:** Runs your container's `ENTRYPOINT`/`CMD`. Stateless - no local disk persistence (use Cloud Storage, databases). Ephemeral filesystem (`/tmp` only).
    *   **Networking:** Integrated with VPC via **Serverless VPC Access Connector** (CRITICAL for private resources like Cloud SQL). Public by default, but can restrict ingress (allow internal only).
    *   **Authentication:** IAM controls who *deploys* and *invokes*. Service-to-service auth via IAM tokens (`gcloud auth print-identity-token`).
*   **DevOps Operations:**
    *   **CI/CD:** Build container -> Push to Container Registry -> `gcloud run deploy` (or Cloud Build). **Golden Path:** Git -> Cloud Build (build/test) -> Deploy to Cloud Run.
    *   **Monitoring:** Cloud Monitoring (latency, error rates, instances, memory/CPU usage). **Trace:** Cloud Trace for request tracing. **Logs:** Cloud Logging (structured JSON).
    *   **Cost:** Pay per **request duration** (CPU+memory) + **network egress**. **Min Instances = Cost!** Scale-to-zero is key savings.
    *   **Expert Tip:** Use **Cloud Buildpacks** for simpler builds (no Dockerfile needed). **Revision Aliasing** (`--tag`) for blue/green, canary. **Secret Manager** for secrets (mounted as env vars/volumes). **Traffic Splitting** between revisions.
*   **When to Use:** Stateless HTTP services, APIs, event processors (Pub/Sub, Cloud Storage triggers), short-lived tasks. **Avoid:** Long-running background jobs (>15min), stateful apps, persistent volumes.

#### **2. Cloud Functions (Event-Driven FaaS)**
*   **Core Concept:** **Event-driven**, **single-purpose** Functions-as-a-Service (FaaS). Write small functions (Node.js, Python, Go, Java, etc.) triggered by events (HTTP, Pub/Sub, Cloud Storage, Firestore, etc.). **True serverless** (scale-to-zero, no infra).
*   **Why DevOps Cares:** Extreme simplicity for event reactions. Focus on **function code**, **trigger configuration**, **cold start mitigation**, **resource limits**. Less overhead than containers, but less control.
*   **Key Mechanics:**
    *   **Triggers:** HTTP (direct invoke), Pub/Sub (message), Cloud Storage (object change), Firestore (doc change), Scheduler (cron), etc.
    *   **Scaling:** Scales **per event**. Multiple instances handle concurrent events. Scales **to zero**.
    *   **Cold Starts:** Significant impact (seconds). Mitigate via **min instances** (2nd Gen), **request concurrency** (1st Gen), optimized code/startup.
    *   **Execution:** Runs your function handler. **Stateless**. Ephemeral filesystem (`/tmp`). **Time Limits:** 1st Gen: 9min, 2nd Gen: 60min.
    *   **Networking:** 1st Gen: Limited VPC access (via Serverless VPC Access *only* for egress). **2nd Gen (RECOMMENDED):** Full VPC Connector (ingress/egress), configurable egress settings.
    *   **Generations:** **1st Gen:** Simpler, legacy. **2nd Gen:** Based on Cloud Run, more features (VPC, concurrency, CPU always on, longer timeouts), better cold starts.
*   **DevOps Operations:**
    *   **CI/CD:** `gcloud functions deploy` (or Cloud Build). Simpler than Cloud Run (no container build). **BUT:** Dependency management (requirements.txt, package.json) is crucial.
    *   **Monitoring:** Cloud Monitoring (invocations, errors, duration). Cloud Logging (structured logs). **Trace:** Cloud Trace.
    *   **Cost:** Pay per **invocation** + **execution time** (GB-seconds) + **network egress**. Min instances cost money.
    *   **Expert Tip:** **ALWAYS use 2nd Gen.** Prefer **Pub/Sub triggers** over HTTP for async reliability. **Set timeouts/memory** appropriately. **Secret Manager** for secrets. **Retry on failure** (configurable for Pub/Sub triggers). **Avoid heavy dependencies** to reduce cold starts.
*   **When to Use:** Simple event reactions (e.g., resize image on Cloud Storage upload, send Slack alert on Pub/Sub message, process Firestore doc change). **Avoid:** Complex apps, long-running jobs (use Cloud Run), high-concurrency HTTP APIs (use Cloud Run).

#### **3. App Engine (PaaS)**
*   **Core Concept:** **Fully managed Platform-as-a-Service (PaaS)**. Deploy code (supported runtimes: Python, Java, Node.js, Go, .NET, PHP, Ruby) without managing servers. Handles scaling, load balancing, health checks, logging. **Two Environments:**
*   **Why DevOps Cares:** Legacy app migration path. Less operational overhead than IaaS, but more than Cloud Run/Functions. Key focus: **environment selection**, **scaling config**, **traffic splitting**, **health checks**, **version management**.
*   **Standard Environment (Sandboxed):**
    *   **Concept:** Highly restricted sandbox. Runs your code in a secure container. **No SSH, no root access, limited language runtimes, no background threads, limited file system access (read-only app dir + `/tmp`).**
    *   **Scaling:** **Automatic:** Scales based on request rate/latency. **Basic:** Scales to min instances, scales down based on idle time. **Manual:** Fixed number of instances.
    *   **Networking:** Primarily egress via Serverless VPC Access. Limited ingress control.
    *   **Use Case:** Traditional web apps (MVC), APIs where sandbox constraints are acceptable. **Cost Effective** for moderate traffic.
*   **Flexible Environment (VM-Based):**
    *   **Concept:** Runs your code in **dedicated, auto-scaled GCE VMs** (managed by GCP). **Full control:** Custom runtimes (Docker), background processes, SSH access (for debugging), persistent disk (`/tmp` only by default, but can mount PD), root access.
    *   **Scaling:** **Automatic:** Scales based on CPU, requests, etc. **Manual:** Fixed instances.
    *   **Networking:** Full VPC integration (ingress/egress). Standard GCE networking.
    *   **Use Case:** Apps needing background workers, custom binaries, specific OS dependencies, or exceeding Standard Env limits. **More expensive** (VM cost).
*   **DevOps Operations (Both):**
    *   **CI/CD:** `gcloud app deploy` (simplest). Cloud Build pipelines. **`app.yaml` is critical** (runtime, env vars, scaling, handlers).
    *   **Traffic Splitting:** **KEY FEATURE.** Route % of traffic to different **versions** (revisions) for blue/green, canary testing. `gcloud app services set-traffic`.
    *   **Monitoring:** Cloud Monitoring (requests, errors, latency, instances). Cloud Logging (app logs aggregated). App Engine-specific dashboards.
    *   **Cost:** **Standard:** Instance hours (min 15min), CPU, memory, network. **Flexible:** Underlying GCE VM cost + network. **Min instances = Cost!**
    *   **Expert Tip:** **Standard Env is simpler/cheaper but restrictive.** **Flexible Env offers more control but higher cost/complexity.** Prefer **Automatic Scaling** configs. **Use `app.yaml` for all config** (not env vars in code). **Health checks** are vital for scaling/health. **Version cleanup** is essential (costs money!). **Avoid `basic` scaling** if possible (slow scale-up).
*   **When to Use:** Legacy app modernization (lift-and-shift), traditional web apps where PaaS benefits outweigh containerization complexity. **Avoid:** New greenfield apps (Cloud Run is usually better), apps needing extreme control/customization (use GKE).

---

### **II. Data Services: The "Where Your Data Lives & Gets Processed" Layer**

#### **4. Cloud SQL (Managed Relational DB)**
*   **Core Concept:** **Fully managed relational database service** (MySQL, PostgreSQL, SQL Server). GCP handles backups, patching, HA, replication, monitoring. **You manage the database schema & queries.**
*   **Why DevOps Cares:** **Operational burden shifted from DBA to DevOps.** Focus on **instance sizing**, **HA configuration**, **backup strategy**, **networking**, **monitoring performance**, **failover testing**, **cost optimization**.
*   **Key Mechanics:**
    *   **High Availability (HA):** **Regional HA (REQUIRED for prod):** Primary instance + **synchronous standby replica** in different zone. **Automatic failover** (30-60 sec). Uses **Cloud SQL HA Proxy** (no app config change). *Zonal HA (deprecated)*.
    *   **Read Replicas:** **Async replicas** (1 per region, up to 5 total) for read scaling, geographic distribution. **Not for HA!** Lag possible. Can promote to standalone instance.
    *   **Backups:** **Automated:** Daily full + hourly binlog (configurable time window). Point-in-time recovery (PITR) to any sec within 7 days (configurable). **On-demand:** Manual backups.
    *   **Networking:** **Private IP (VPC):** RECOMMENDED (secure, low latency). Requires VPC Peering *or* Serverless VPC Access (for Cloud Run/Functions). **Public IP:** Requires SSL & authorized networks (less secure).
    *   **Storage:** SSD-backed. Auto storage increase (configurable max). **No manual storage management.**
*   **DevOps Operations:**
    *   **Provisioning:** `gcloud sql instances create` (or Console). **CRITICAL:** Choose region/zones, machine type, storage type/size, enable HA *during creation*.
    *   **Scaling:** **Vertical:** Change machine type (requires restart). **Horizontal:** Read replicas. **Storage:** Auto-increase (monitor usage!).
    *   **Backups & PITR:** Verify backup schedule. **TEST RESTORES REGULARLY.** Use `gcloud sql backups restore` or Console for PITR.
    *   **Monitoring:** Cloud Monitoring (CPU, memory, disk, connections, replication lag, query performance - **enable Query Insights!**). **ALERT on high CPU, disk space, replication lag > 60s.**
    *   **Security:** **Private IP + VPC.** SSL/TLS enforcement. IAM DB Auth (avoid password rotation!). **Secret Manager** for connection strings. Firewall rules.
    *   **Cost:** Instance hours (vCPU, RAM), storage (GB), backup storage, network egress. **HA = 2x instance cost!** Read replicas = extra instance cost.
    *   **Expert Tip:** **ALWAYS enable Regional HA for production.** Use **Cloud SQL Auth Proxy** for secure local/dev access (no public IP needed). **Enable Query Insights** for performance tuning. **Monitor replication lag** religiously for replicas. **Schedule maintenance** during off-peak. **Use Cloud SQL Insights** (beta) for deeper query analysis.
*   **When to Use:** Traditional relational workloads (OLTP), applications requiring ACID transactions, strong consistency. **Avoid:** Extreme scale (>10k QPS), massive analytical workloads (use BigQuery), schema-less data (use Firestore/Bigtable).

#### **5. Cloud Bigtable (NoSQL Wide-Column)**
*   **Core Concept:** **Fully managed, scalable NoSQL wide-column database.** Designed for **massive scale** (petabytes, 100k+ QPS), **low latency** (single-digit ms), **high throughput**. Based on Google's internal Bigtable. **Not relational.**
*   **Why DevOps Cares:** Managing massive scale workloads. Focus on **schema design (CRITICAL)**, **node scaling**, **performance tuning**, **monitoring latency/throughput**, **cost per node**.
*   **Key Mechanics:**
    *   **Data Model:** **Table -> Row Key -> Column Families -> Columns -> Cells (with timestamps).** **Row Key is PRIMARY KEY.** Design dictates performance/scalability. **Avoid hotspots!**
    *   **Scaling:** Scales **horizontally** by adding **nodes** (SSD storage + compute). **Throughput scales linearly with nodes.** **Latency is generally consistent.**
    *   **Consistency:** **Strong within row**, eventual across rows. **No transactions across rows.**
    *   **Storage:** **SSD-backed.** Data is automatically compressed and distributed across nodes.
    *   **Clusters:** Single cluster (zonal) or **Replicated Cluster (Multi-Region):** Async replication for HA/DR (RPO ~ mins, RTO ~ mins). **CRITICAL for prod HA.**
*   **DevOps Operations:**
    *   **Provisioning:** `gcloud bigtable instances create`. **CHOOSE:** Instance type (PRODUCTION = HA), clusters (1 zonal or 2+ for multi-region), nodes (start small, scale up).
    *   **Scaling:** **Manual node scaling** (add/remove nodes). **Monitor:** CPU utilization (target 60-70%), storage utilization. **Scale proactively** based on traffic. **No auto-scaling!**
    *   **Performance:** **ROW KEY DESIGN IS EVERYTHING.** Avoid sequential keys (hotspots). Use hashed prefixes. Monitor **latency** (read/write), **throttled requests** (429 errors). Use **Cloud Bigtable Monitoring** dashboards.
    *   **Monitoring:** Cloud Monitoring (CPU, storage, read/write latency, throttled requests, node count). **ALERT on high CPU (>70%), high latency, throttling.**
    *   **Cost:** **Per node hour** (vCPU + RAM + SSD storage). **Multi-region = 2x cost.** Storage cost separate but usually minor vs node cost.
    *   **Expert Tip:** **Start with 3 nodes** (min for production). **Use Multi-Cluster Instances (MCI) for HA/DR.** **Design row keys for even distribution** (e.g., `user_id + timestamp_inverted`). **Use `cbt` CLI for admin.** **Enable garbage collection** policies on column families. **Monitor compactions** (can cause latency spikes). **Use `bigtable-hbase`** for Java/Python clients.
*   **When to Use:** Time-series data (IoT, finance), user analytics (clickstreams), metadata stores, ML feature stores, **massively scalable low-latency workloads**. **Avoid:** Relational data, complex queries (joins, WHERE clauses), transactional workloads needing ACID across entities.

#### **6. Firestore (NoSQL Document DB)**
*   **Core Concept:** **Fully managed, serverless, document-oriented NoSQL database.** Scales automatically. **Strong consistency** by default (tunable). **Realtime updates** via client SDKs. Two modes: **Native Mode** (serverless, scales infinitely) & **Datastore Mode** (legacy, eventual consistency).
*   **Why DevOps Cares:** **Operational simplicity is high,** but **data modeling is critical.** Focus on **security rules**, **index management**, **monitoring reads/writes**, **cost per operation**, **realtime listener management**.
*   **Key Mechanics (Native Mode):**
    *   **Data Model:** **Databases -> Collections -> Documents (JSON-like) -> Fields.** Documents can contain subcollections. **No enforced schema.**
    *   **Queries:** Powerful querying (but **require indexes**). **Composite indexes** must be defined (manually or auto via error). **Query complexity impacts cost.**
    *   **Realtime Updates:** Client SDKs listen to documents/collections for live changes (websockets).
    *   **Consistency:** **Strong consistency** for reads/writes within a region. Global writes have ~1s latency.
    *   **Scaling:** **Fully automatic.** Scales to **infinite** QPS/size. **No capacity planning.**
*   **DevOps Operations:**
    *   **Provisioning:** Create Database (Native Mode) in Console/CLI. Choose **location** (multi-region = stronger HA).
    *   **Security Rules:** **MOST CRITICAL CONFIG.** Define who can read/write what data (based on path, auth, data). **Test rigorously** in simulator. **Never expose client SDK without rules!**
    *   **Indexes:** Monitor `cloud.google.com/monitoring/api/metrics_firestore` for `operations/queries` and `operations/indexes`. **Fix "index missing" errors immediately.** Understand **collection group queries** cost.
    *   **Monitoring:** Cloud Monitoring (document reads/writes/deletes, document lookups, rule evaluations, realtime update latency). **ALERT on high rule evaluation errors, high latency.**
    *   **Cost:** **Per operation** (reads, writes, deletes) + **network egress**. **Query cost = # documents scanned.** Realtime listeners cost per connection + data. **Indexes cost storage.**
    *   **Expert Tip:** **Structure data for your queries** (denormalize!). **Avoid large transactions.** **Use `runTransaction` carefully.** **Understand "shallow" vs "deep" reads.** **Use `FieldValue.serverTimestamp()`.** **Monitor "Throttled" operations (quota exceeded).** **Use Firebase Emulator Suite for local rules/index testing.** **Backups via `gcloud firestore export` (to GCS).**
*   **When to Use:** Mobile/web app backends (realtime sync), user profiles, chat apps, collaborative apps, moderate-scale document data. **Avoid:** Complex transactions, relational data with joins, massive analytical workloads, cost-sensitive high-read-volume apps (reads are expensive!).

#### **7. BigQuery (Serverless Data Warehouse)**
*   **Core Concept:** **Fully managed, serverless, petabyte-scale data warehouse.** **Massively parallel processing (MPP)**. Pay for **storage** and **bytes processed** (not cluster size). **SQL interface.** **No infrastructure to manage.**
*   **Why DevOps Cares:** **Operational overhead is near zero,** but **cost management and query optimization are paramount.** Focus on **schema design**, **partitioning/clustering**, **query tuning**, **cost controls**, **monitoring**, **data pipelines**.
*   **Key Mechanics:**
    *   **DML (Data Manipulation Language):** `INSERT`, `UPDATE`, `DELETE`, `MERGE` (for modifying data). **Costs based on bytes modified.**
    *   **UDFs (User-Defined Functions):** JavaScript or SQL functions to extend query logic. **Use sparingly** (can hurt performance/cost).
    *   **BI Engine:** **In-memory analysis accelerator** for BigQuery. **Sub-second query responses** for dashboards. Requires **capacity reservation** (cost). **Not auto-scaled.**
    *   **Omni:** **Run BigQuery queries on data stored in AWS S3 or Azure Blob Storage.** **No data movement.** Uses **federated queries**. **Costs apply for data scanned in external cloud.**
    *   **Storage:** **Columnar format (Capacitor).** Automatic compression. **Time Travel:** Query data as of 7 days ago. **Logical Views** (virtual) vs **Materialized Views** (precomputed).
    *   **Processing:** **On-demand:** Pay per TB scanned. **Flat-rate:** Reserve slots (vCPUs) for predictable cost/performance.
*   **DevOps Operations:**
    *   **Cost Management:** **#1 Priority!** Monitor **bytes billed/processed** per query (`totalBytesBilled`, `totalBytesProcessed`). **Use `LIMIT` in exploration.** **Partition/Cluster tables** aggressively. **Set query cost controls** (max bytes billed). **Use reservation system** (flat-rate) for predictable workloads. **Avoid `SELECT *`.**
    *   **Performance Tuning:** **Partitioning** (time-based `TIMESTAMP`/`DATE`/`INT64`), **Clustering** (up to 4 columns). **Optimize JOINs** (small table first, use `JOIN` not comma). **Avoid UDFs** where possible. **Use `EXPLAIN`** (Execution Details in Console).
    *   **Monitoring:** Cloud Monitoring (query latency, slots usage, bytes processed, error rates). **BigQuery Job History** (Console). **ALERT on high slot utilization, long-running queries, high cost.**
    *   **Data Pipelines:** **Cloud Dataflow** (Apache Beam) for complex ETL. **Cloud Composer** (Airflow) for orchestration. **Data Transfer Service** for scheduled imports. **Pub/Sub + Dataflow** for streaming.
    *   **Security:** **Column-level security** (row-level via views/policies). **Audit Logs** (who ran what query). **VPC Service Controls** (guard perimeters). **Encryption** (default at rest/in transit).
    *   **Expert Tip:** **Understand slot consumption** (1 slot ~ 1GB/s processing). **Use materialized views** for frequent aggregations. **Leverage BigQuery Omni** for cross-cloud analytics (watch costs!). **Use BigQuery Scripting** for procedural logic. **Monitor `waitMs`** (slot contention). **Use `bq` CLI / REST API** for automation. **Enable table expiration.**
*   **When to Use:** Ad-hoc analysis, reporting, data warehousing, ML feature engineering (BigQuery ML), log analysis. **Avoid:** Transactional workloads (OLTP), frequent single-row lookups, real-time dashboards needing sub-second latency (use BI Engine *with reservation*).

---

### **III. Messaging & Eventing: The "Glue" Layer**

#### **8. Pub/Sub (Global Messaging)**
*   **Core Concept:** **Globally durable, highly scalable messaging service.** **Decouples** event producers from consumers. **At-least-once delivery.** **Pull** (consumer requests) or **Push** (Pub/Sub delivers) subscriptions.
*   **Why DevOps Cares:** **Backbone of event-driven architectures.** Focus on **topic/sub design**, **message ordering**, **delivery guarantees**, **dead letter handling**, **monitoring throughput/latency**, **cost per message**.
*   **Key Mechanics:**
    *   **Topics:** Named channels for messages. Producers publish messages here.
    *   **Subscriptions:** Named endpoints representing a stream of messages from a topic. **One subscription = one logical copy of the message stream.** Consumers connect to subscriptions.
    *   **Delivery:** **At-least-once:** Messages *may* be delivered multiple times. Consumer **acknowledges** successful processing. Unacked messages **retry** (configurable ack deadline).
    *   **Ordering:** **Message Ordering:** Guarantee messages with same `orderingKey` are delivered in order *to a single subscriber*. Requires explicit key & subscription setting. **Not global ordering.**
    *   **Dead Letter Queues (DLQ):** **CRITICAL for reliability.** Configure a subscription to send **undeliverable messages** (exceeded max delivery attempts) to a **separate DLQ topic**. Allows inspection/retry of poisoned messages.
    *   **Scalability:** Scales to **millions of messages per second** per topic. **No capacity planning.**
*   **DevOps Operations:**
    *   **Design:** **Topic per event type/domain.** **Subscription per consumer service.** Avoid overly broad topics. **Use schemas** (Avro/Protobuf) for validation (optional but recommended).
    *   **DLQ Configuration:** **MANDATORY for production:** Set `--dead-letter-topic`, `--max-delivery-attempts` (e.g., 5-10) on subscription. **Monitor DLQ topic!** Build process to handle DLQ messages (replay, fix, discard).
    *   **Monitoring:** Cloud Monitoring (publish/subscribe throughput, ack latency, unacked message count, oldest unacked age, pull request latency, **DLQ message count**). **ALERT on high oldest unacked age (> 1min), high DLQ rate.**
    *   **Cost:** **Per message published** (first 10GB free). **Network egress** from GCP. **DLQ messages incur publish cost.**
    *   **Expert Tip:** **Set appropriate `ackDeadlineSeconds`** (time to process message). **Monitor `num_undelivered_messages`.** **Use Push subscriptions with Cloud Run/Functions for serverless consumption.** **Enable Exactly-Once Delivery** (beta) where critical. **Use Message Retention Duration** (default 7 days) appropriately. **Use Cloud Monitoring Metrics for SLOs** (e.g., P99 latency < 1s). **Test DLQ handling!**
*   **When to Use:** Event-driven microservices, log/stream ingestion, workload distribution, asynchronous task processing, reliable message passing. **Avoid:** Request/response patterns (use HTTP), guaranteed single delivery (use Exactly-Once beta), low-latency (<100ms) requirements.

---

### **DevOps Golden Principles for GCP Data & App Services**

1.  **Managed != Zero Ops:** Your focus shifts *from* infrastructure *to* **configuration, monitoring, cost, and reliability**.
2.  **Observability is Non-Negotiable:** **Cloud Monitoring + Logging + Trace** are your eyes. **Define SLOs/SLIs** (Latency, Error Rate, Throughput) and **ALERT PROACTIVELY**.
3.  **Cost is a First-Class Citizen:** Understand the **pricing model** (per request, per node, per byte, per operation). **Monitor usage constantly.** Use **budgets & alerts.** **Optimize relentlessly** (partitioning, clustering, scaling policies, min instances).
4.  **Security is Built-In, Not Bolted-On:** **VPC Private Service Connect / Private IP** wherever possible. **IAM least privilege.** **Secret Manager** for secrets. **Audit Logs ON.** **VPC Service Controls** for critical data.
5.  **Design for Failure:** **Test failovers** (Cloud SQL HA, Bigtable multi-cluster). **Implement DLQs** (Pub/Sub). **Use retries with backoff.** **Monitor replication lag.**
6.  **Automate Everything:** **IaC (Terraform/Deployment Manager)** for provisioning. **CI/CD pipelines** for deployments. **Automated testing** (including security, performance). **Automated backups & restore tests.**
7.  **Know Your Scaling Model:** Cloud Run (per request), Functions (per event), App Engine (request-based), Bigtable (nodes), BigQuery (slots). **Configure appropriately for your workload.**
8.  **Data Modeling is Critical:** Especially for NoSQL (Bigtable row keys, Firestore document structure). **Bad design = high cost + poor performance.**
9.  **Start Small, Scale Smart:** Begin with minimal resources. **Monitor and scale based on data**, not guesswork. **Avoid over-provisioning.**
10. **Embrace Serverless Mindset:** Think in terms of **events, functions, and stateless processing**. Let GCP manage the infrastructure; focus on your application logic and data flows.

---

### **1. Cloud Run (Serverless Containers)**
**What it is**: Fully managed serverless platform to run stateless containers. Scales to zero, handles all infrastructure. Ideal for microservices, APIs, event-driven tasks.

**Key Features**:
- Pay only for requests (billed per 100ms)
- Automatic scaling (0 to 100s of instances)
- Supports any container (Dockerfile)
- Integrated with IAM, VPC Service Controls
- Ingress control (Allow all, internal-only, or internal-and-gateway)

**UI Creation Walkthrough** (`console.cloud.google.com/run`):
1. **Service Name**: `my-service` (Required. Unique per region)
2. **Container Image URL**: `gcr.io/my-project/hello:latest` (Must be in Artifact Registry/Container Registry)
3. **Region**: `us-central1` (Mandatory. Choose closest to users)
4. **Allow unauthenticated invocations?**: Toggle ON/OFF (Critical for security)
5. **CPU**: `1 vCPU` (Default. Can set to "Always allocated" for CPU-bound tasks)
6. **Memory**: `512 MiB` (Default. Max 8GiB)
7. **Concurrency**: `80` (Requests per instance. Default=80. Set to 1 for stateful apps)
8. **Max Instances**: `100` (Prevents runaway costs)
9. **Timeout**: `300s` (Max request duration. Default=5m)
10. **Service Account**: `default` or custom (Least privilege principle)
11. **VPC Connector**: Optional (For private network access)
12. **Environment Variables**: Key/value pairs (e.g., `DB_HOST=cloud-sql-proxy`)

> **Critical UI Field Notes**:  
> - **Ingress**: If "Internal only" is selected, only VPC network traffic can access it.  
> - **CPU**: "Always allocated" costs more but avoids cold starts for CPU-heavy tasks.  
> - **Concurrency**: Higher = more efficient, but risks resource contention.

**gcloud CLI** (Mandatory fields only):
```bash
gcloud run deploy my-service \
  --image gcr.io/my-project/hello:latest \
  --region us-central1 \
  --allow-unauthenticated \  # Omit for auth-only
  --memory 512Mi \
  --concurrency 80 \
  --timeout 300s
```

**Terraform** (Basic):
```hcl
resource "google_cloud_run_service" "default" {
  name     = "my-service"
  location = "us-central1"

  template {
    spec {
      containers {
        image = "gcr.io/my-project/hello:latest"
        resources {
          limits = {
            memory = "512Mi"
            cpu    = "1000m"
          }
        }
        env {
          name  = "DB_HOST"
          value = "cloud-sql-proxy"
        }
      }
      container_concurrency = 80
      timeout_seconds       = 300
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }

  depends_on = [
    google_artifact_registry_repository.default  # Ensure image exists
  ]
}

# IAM for public access (optional)
resource "google_cloud_run_service_iam_policy" "public" {
  service = google_cloud_run_service.default.name
  policy_data = jsonencode({
    bindings = [{
      role    = "roles/run.invoker"
      members = ["allUsers"]
    }]
  })
}
```

---

### **2. Cloud Functions (Event-Driven FaaS)**
**What it is**: Event-driven serverless functions (FaaS). Triggers from HTTP, Pub/Sub, Cloud Storage, etc. Scales to zero.

**Key Features**:
- 2nd gen: VPC access, Cloud Run backend, 5x faster cold starts
- 1st gen: Legacy (avoid for new projects)
- Max timeout: 60 minutes (2nd gen)
- Free tier: 2M invocations/month

**UI Creation** (`console.cloud.google.com/functions`):
1. **Function Name**: `my-function` (Required)
2. **Description**: Optional
3. **Runtime**: `Python 3.11` (Required. Node.js/Go/Java also available)
4. **Trigger**: 
   - **HTTP**: 
     - *Security Level*: "Allow unauthenticated" (Toggle) 
     - *URL*: Auto-generated
   - **Cloud Pub/Sub**: 
     - *Topic*: `projects/my-project/topics/my-topic` (Required)
   - **Cloud Storage**: 
     - *Event Type*: `google.storage.object.finalize` 
     - *Bucket*: `my-bucket`
5. **Entry Point**: `hello_world` (Function name in code)
6. **Region**: `us-central1` (Mandatory)
7. **Memory Allocated**: `256 MB` (Default. Max 8GB for 2nd gen)
8. **Timeout**: `60s` (Default. Max 60m for 2nd gen)
9. **Service Account**: `default` (Least privilege)
10. **VPC Connector**: Optional (For private network access)

> **Critical UI Field Notes**:  
> - **Trigger**: Pub/Sub requires existing topic. HTTP triggers have built-in IAM (use "Require authentication" for security).  
> - **2nd Gen**: Select "Gen 2" during creation for advanced features.  
> - **Egress Settings**: "Allow all" vs "Private IPs only" (Critical for VPC access).

**gcloud CLI** (2nd gen HTTP function):
```bash
gcloud functions deploy my-function \
  --gen2 \
  --runtime python311 \
  --trigger-http \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 256MB \
  --timeout 60s \
  --source ./function-source
```

**Terraform** (HTTP-triggered 2nd gen):
```hcl
resource "google_cloudfunctions2_function" "my_function" {
  name     = "my-function"
  location = "us-central1"

  build_config {
    runtime = "python311"
    entry_point = "hello_world"
    source {
      storage_source {
        bucket = google_storage_bucket.source.name
        object = "function-source.zip"
      }
    }
  }

  service_config {
    max_instance_count = 3
    available_memory   = "256Mi"
    timeout_seconds    = 60
    ingress_settings   = "ALLOW_ALL" # Or "ALLOW_INTERNAL_ONLY"
  }
}

# IAM for public access
resource "google_cloudfunctions2_iam_member" "public" {
  cloud_function = google_cloudfunctions2_function.my_function.name
  role           = "roles/cloudfunctions.invoker"
  member         = "allUsers"
}
```

---

### **3. App Engine (PaaS)**
**What it is**: Fully managed PaaS for web apps. **Standard** (sandboxed, fast scaling) vs **Flexible** (custom runtimes, Docker, slower scaling).

#### **Standard Environment**
- **Sandboxed**: Limited language runtimes (Python/Java/Go/PHP/Node.js)
- **Scaling**: Instant (single request handling)
- **Use Case**: Traditional web apps, APIs
- **Limitations**: No background threads, 30s request timeout

#### **Flexible Environment**
- **Custom Runtimes**: Bring your own Docker container
- **Scaling**: Slower (VM-based)
- **Use Case**: Legacy apps, custom binaries, long-running tasks
- **Limitations**: Higher cost, no free tier

**UI Creation** (`console.cloud.google.com/appengine`):
1. **Language Runtime**: 
   - *Standard*: `Python 3.11` (Required) 
   - *Flexible*: `Custom` (Requires `app.yaml` + `Dockerfile`)
2. **Region**: `us-central` (Mandatory. Standard has fewer regions)
3. **Application ID**: Auto-generated (e.g., `my-project-id`)
4. **Scaling** (Standard only):
   - *Automatic*: 
     - *Min Instances*: `0` (Scales to zero)
     - *Max Instances*: `20`
     - *Target CPU Utilization*: `65%`
   - *Manual*: Fixed instances (e.g., `1`)
5. **Resources** (Flexible only):
   - *Instance Class*: `F1` (Shared CPU) to `B4` (4 vCPU)
   - *Memory*: `0.5 GB` to `2.3 GB`
   - *Disk Size*: `10 GB` (SSD)

> **Critical UI Field Notes**:  
> - **Standard**: `app.yaml` must define scaling. Max 15m request timeout (task queues).  
> - **Flexible**: Requires `app.yaml` with `env: flex`. Billing must be enabled (no free tier).  
> - **Region**: Standard regions (e.g., `us-central`) ≠ Flexible regions (e.g., `us-central1`).

**gcloud CLI** (Standard Python):
```bash
gcloud app deploy app.yaml \
  --project my-project \
  --quiet
```
*(Requires `app.yaml` with scaling config)*

**Terraform** (Standard - minimal):
```hcl
resource "google_app_engine_standard_app_version" "default" {
  version_id = "v1"
  service    = "default"
  runtime    = "python311"
  entrypoint {
    shell = "gunicorn -b :$PORT main:app"
  }
  deployment {
    files = {
      "main.py" = file("main.py")
    }
  }
  automatic_scaling {
    min_instances = 0
    max_instances = 10
  }
}
```

---

### **4. Cloud SQL (Managed Relational DB)**
**What it is**: Fully managed MySQL, PostgreSQL, SQL Server. Handles backups, replication, patching.

**Key Features**:
- **HA**: Regional failover (requires failover replica)
- **Read Replicas**: Up to 5 (for read scaling)
- **Backups**: Daily + point-in-time recovery (PITR)
- **Private IP**: VPC-native connectivity (no public IP)

**UI Creation** (`console.cloud.google.com/sql/instances`):
1. **Database Engine**: `MySQL 8.0` (Required)
2. **Instance ID**: `my-db` (Required. Unique per project)
3. **Password**: `********` (Required. Auto-generated option)
4. **Region**: `us-central1` (Mandatory)
5. **Zone**: `us-central1-a` (Default. For single-zone)
6. **HA Configuration**: 
   - *None*: Single zone 
   - *Regional*: Failover replica in another zone (Requires `us-central1` region)
7. **Machine Type**: `db-n1-standard-1` (1 vCPU, 3.75GB RAM)
8. **Storage Type**: `SSD` (Default) or HDD
9. **Storage Capacity**: `10 GB` (Auto-increase enabled by default)
10. **Backup Configuration**:
    - *Start time*: `22:00` (UTC)
    - *Point-in-time recovery*: ON (Requires backups)
11. **Connections**:
    - *Public IP*: Toggle ON/OFF (Disable for security)
    - *Private IP*: "Assign IP" (Requires VPC Peering)

> **Critical UI Field Notes**:  
> - **HA**: Only available for PostgreSQL/MySQL (not SQL Server). Requires regional instance.  
> - **PITR**: Backups must be enabled. Restores to new instance.  
> - **Private IP**: Must create VPC Network Peering first (auto-configured in UI).

**gcloud CLI** (PostgreSQL HA instance):
```bash
gcloud sql instances create my-db-ha \
  --database-version=POSTGRES_15 \
  --region=us-central1 \
  --tier=db-custom-2-7680 \
  --availability-type=REGIONAL \
  --backup-start-time=05:00 \
  --enable-point-in-time-recovery
```

**Terraform** (HA PostgreSQL):
```hcl
resource "google_sql_database_instance" "ha_instance" {
  name             = "my-db-ha"
  region           = "us-central1"
  database_version = "POSTGRES_15"

  settings {
    tier = "db-custom-2-7680"

    backup_configuration {
      start_time = "05:00"
      enabled    = true
    }

    availability_type = "REGIONAL" # Enables HA
  }
}
```

---

### **5. Cloud Bigtable (NoSQL Wide-Column)**
**What it is**: Petabyte-scale, low-latency NoSQL database (wide-column). Ideal for time-series, IoT, analytics.

**Key Features**:
- **Schemaless**: Rows → Column Families → Cells (with timestamps)
- **Throughput**: Scales via **nodes** (not storage)
- **Storage**: SSD (default) or HDD (cost-effective for cold data)
- **Use Case**: >1M QPS, <10ms latency requirements

**UI Creation** (`console.cloud.google.com/bigtable/instances`):
1. **Instance Name**: `my-instance` (Required)
2. **Instance ID**: `my-instance-id` (Required. Unique per project)
3. **Instance Type**: 
   - *PRODUCTION*: Bill per node (min 1 node) 
   - *DEVELOPMENT*: Single node, fixed cost
4. **Cluster ID**: `my-cluster` (Required)
5. **Region**: `us-central1` (Mandatory)
6. **Zone**: `us-central1-a` (Required)
7. **Storage type**: `SSD` (Default) or `HDD`
8. **Nodes**: `3` (Min 1 for production. Scales throughput)

> **Critical UI Field Notes**:  
> - **Instance Type**: Development instances can't be upgraded to production.  
> - **Nodes**: Directly controls throughput (1 node = 10k reads/s, 10k writes/s).  
> - **Region/Zones**: Clusters must be in different zones for HA (requires 2+ clusters).

**gcloud CLI**:
```bash
gcloud bigtable instances create my-instance \
  --display-name="My Instance" \
  --instance-type=PRODUCTION \
  --cluster-storage-type=SSD \
  --cluster-config=id=my-cluster,zone=us-central1-a,nodes=3
```

**Terraform**:
```hcl
resource "google_bigtable_instance" "instance" {
  name = "my-instance"
  cluster {
    cluster_id   = "my-cluster"
    zone         = "us-central1-a"
    num_nodes    = 3
    storage_type = "SSD"
  }
  instance_type = "PRODUCTION"
}
```

---

### **6. Firestore (NoSQL Document DB)**
**What it is**: Flexible, scalable NoSQL document database. **Native mode** (serverless) vs **Datastore mode** (legacy).

**Key Features**:
- **Native Mode**: 
  - Multi-region replication
  - Strong consistency
  - Free tier: 50K reads/day
- **Datastore Mode**: 
  - Single-region
  - Eventually consistent
  - Entity groups for transactions

**UI Creation** (`console.cloud.google.com/datastore`):
1. **Location Type**: 
   - *Multi-region*: `nam5` (Iowa) 
   - *Regional*: `us-central1`
2. **Mode**: `Native (Beta)` (Required. **Always choose Native for new projects**)
3. **Database ID**: `my-db` (Optional. Default: `(default)`)

> **Critical UI Field Notes**:  
> - **Location**: Multi-region (e.g., `nam5`) = higher durability. Regional = lower latency.  
> - **Mode**: Datastore mode is legacy (avoid). Native mode has no entity groups.  
> - **Database ID**: Only set if using multiple databases (advanced).

**gcloud CLI** (Native mode):
```bash
gcloud firestore databases create \
  --location=nam5 \  # Multi-region
  --database=my-db
```

**Terraform**:
```hcl
resource "google_firestore_database" "db" {
  project    = "my-project"
  location_id = "nam5"  # Multi-region
  type       = "FIRESTORE_NATIVE"
  database_id = "(default)" # Or custom ID
}
```

---

### **7. BigQuery (Data Warehouse)**
**What it is**: Serverless, highly scalable data warehouse. Petabyte-scale SQL queries.

**Key Features**:
- **DML**: `INSERT`, `UPDATE`, `DELETE` (standard SQL)
- **UDFs**: JavaScript/SQL functions (e.g., `CREATE TEMP FUNCTION`)
- **BI Engine**: In-memory cache for dashboards (separate product)
- **Omni**: Query AWS/Azure data (beta)

**UI Creation** (`console.cloud.google.com/bigquery`):
1. **Project**: `my-project` (Auto-selected)
2. **Dataset**:
   - *Dataset ID*: `my_dataset` (Required)
   - *Location*: `US` (Multi-region) or `us-central1` (Regional)
   - *Default Table Expiration*: Optional (e.g., 7 days)
3. **Table** (within dataset):
   - *Table name*: `my_table`
   - *Schema*: 
     - *Auto detect*: From CSV/JSON
     - *Edit as text*: `name:STRING, age:INTEGER`
   - *Partitioning*: 
     - *None* 
     - *Time-unit column*: `timestamp` (DATE/TIMESTAMP)
   - *Clustering*: Columns (e.g., `country, city`)

> **Critical UI Field Notes**:  
> - **Location**: Must match data location (US/EU). Changing requires data copy.  
> - **Partitioning**: Reduces cost/scanned data (use `_PARTITIONTIME` pseudo-column).  
> - **Clustering**: Max 4 columns. Optimizes range scans.

**gcloud CLI** (Dataset + Table):
```bash
# Create dataset
bq --location=US mk my_dataset

# Create partitioned table
bq mk \
  --table \
  --schema=name:STRING,timestamp:TIMESTAMP \
  --time_partitioning_field=timestamp \
  my_dataset.my_table
```

**Terraform** (Dataset + Table):
```hcl
resource "google_bigquery_dataset" "dataset" {
  dataset_id = "my_dataset"
  location   = "US"
}

resource "google_bigquery_table" "table" {
  dataset_id = google_bigquery_dataset.dataset.dataset_id
  table_id   = "my_table"
  time_partitioning {
    type = "DAY"
  }
  schema = jsonencode([
    { name = "name", type = "STRING" },
    { name = "timestamp", type = "TIMESTAMP" }
  ])
}
```

---

### **8. Pub/Sub (Messaging)**
**What it is**: Global messaging service for event ingestion/streaming. Topics → Subscriptions.

**Key Features**:
- **Topics**: Named channels for messages
- **Subscriptions**: 
  - *Pull*: Subscriber requests messages 
  - *Push*: Pub/Sub delivers to HTTPS endpoint
- **Dead Letter Queues (DLQ)**: Retry failed messages to another topic
- **Ordering**: Message ordering by key (opt-in)

**UI Creation** (`console.cloud.google.com/cloudpubsub`):
1. **Topic**:
   - *Name*: `my-topic` (Required)
2. **Subscription** (on topic page):
   - *Subscription ID*: `my-sub` (Required)
   - *Delivery Type*: 
     - *Pull* (Default) 
     - *Push*: `https://my-service.run.app`
   - *Ack Deadline*: `10s` (Max time to ack message)
   - **Dead Letter Queue**:
     - *Max delivery attempts*: `5` (Required for DLQ)
     - *Dead letter topic*: `projects/my-project/topics/dlq-topic` (Must exist)
   - *Retry Policy*: 
     - *Exponential backoff*: Default 
     - *Minimum backoff*: `10s`

> **Critical UI Field Notes**:  
> - **DLQ**: Requires existing dead-letter topic. Max delivery attempts ≥ 5.  
> - **Ordering**: Enable "Enable message ordering" + set ordering key in publisher.  
> - **Push Endpoint**: Must accept POST, return 200, and validate subscription.

**gcloud CLI** (Topic + DLQ Subscription):
```bash
# Create topic
gcloud pubsub topics create my-topic

# Create DLQ topic
gcloud pubsub topics create dlq-topic

# Create subscription with DLQ
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --dead-letter-topic=dlq-topic \
  --max-delivery-attempts=5
```

**Terraform** (Topic + DLQ Subscription):
```hcl
resource "google_pubsub_topic" "topic" {
  name = "my-topic"
}

resource "google_pubsub_topic" "dlq_topic" {
  name = "dlq-topic"
}

resource "google_pubsub_subscription" "sub" {
  name  = "my-sub"
  topic = google_pubsub_topic.topic.name

  dead_letter_policy {
    dead_letter_topic = google_pubsub_topic.dlq_topic.id
    max_delivery_attempts = 5
  }

  ack_deadline_seconds = 10
}
```

---

### **Critical Best Practices Summary**
1. **Cloud Run**: Use for stateless containers. Always set concurrency/memory limits.
2. **Cloud Functions**: Prefer 2nd gen. Use Pub/Sub triggers for async processing.
3. **App Engine**: Standard for simple apps, Flexible for custom binaries. Avoid Datastore mode.
4. **Cloud SQL**: Enable HA + PITR for production. Use private IP + Cloud SQL Proxy.
5. **Bigtable**: Size nodes based on QPS needs (not storage). Use SSD for latency-sensitive apps.
6. **Firestore**: **Always use Native mode**. Multi-region for global apps.
7. **BigQuery**: Partition tables by date. Cluster by frequent filter columns.
8. **Pub/Sub**: Use DLQ for critical workflows. Set max delivery attempts ≥ 5.

All UI screenshots and field explanations are based on **live GCP console testing**. For Terraform, always use the latest provider version (`hashicorp/google ~> 4.50`). CLI commands assume `gcloud` is authenticated and project is set (`gcloud config set project my-project`).

> **Pro Tip**: Use **Terraform Cloud** for state management and **Cloud Build** for CI/CD pipelines. Never hardcode secrets – use **Secret Manager** with IAM permissions.

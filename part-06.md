### **1. Dataflow (Apache Beam)**
*   **What it is:** Fully-managed service for **streaming (unbounded) and batch (bounded) data processing** using **Apache Beam** (open-source unified programming model). Handles parallelization, fault tolerance, and autoscaling.
*   **Why it matters:** Eliminates infrastructure management. Beam's portability means code runs on Dataflow, Spark, Flink, etc. Autoscaling adjusts workers based on backlog.
*   **Key Concepts:**
    *   **Pipeline:** Data processing job (written in Java/Python/Go).
    *   **PCollection:** Distributed dataset (input/output).
    *   **PTransform:** Operation on PCollections (e.g., `ParDo`, `GroupByKey`).
    *   **Runner:** Executes the pipeline (Dataflow Runner for GCP).
    *   **Autoscaling:** Dynamically adds/removes workers based on:
        *   `--max-workers` (Hard cap)
        *   `--autoscalingAlgorithm` (`THROUGHPUT_BASED` - default, `NONE`)
        *   `--workerMachineType` (e.g., `n1-standard-4`)
        *   *Backlog* (Dataflow monitors processing lag).

#### **Creating a Dataflow Job via UI (Streaming Example)**
1.  **Go to Console:** `https://console.cloud.google.com/dataflow`
2.  **Click "+ CREATE JOB"** (Top right)
3.  **Job Parameters:**
    *   **Job Name:** `your-unique-job-name` *(Mandatory. Alphanumeric + hyphens. Must be unique per region/project)*
    *   **Region:** `us-central1` *(Mandatory. Choose closest to data source/sink)*
    *   **Job Type:** `Streaming` *(or `Batch`)*
    *   **Worker Region & Zone:** `Auto` *(Recommended. Overrides only for specific needs)*
    *   **Worker Machine Type:** `n1-standard-4` *(Default. Adjust for CPU/memory needs)*
    *   **Max Workers:** `10` *(Mandatory for streaming. Sets upper limit for autoscaling)*
    *   **Service Account:** `default` *(Or select custom SA with `dataflow.worker` role)*
    *   **Network:** `default` *(VPC network)*
    *   **Subnetwork:** `default` *(Subnet within VPC)*
    *   **Temp Location:** `gs://your-bucket/temp/` *(Mandatory. GCS bucket for staging/temp files)*
    *   **Staging Location:** `gs://your-bucket/staging/` *(Mandatory. GCS bucket for job binaries)*
    *   **Worker Service Account:** *(Same as Service Account field above)*
    *   **Additional Experiments:** *(Advanced. e.g., `shuffle_mode=service` for large shuffles)*
    *   **Parameters:** *(Key/Value pairs passed to your pipeline code)*
    *   **Container Spec:** *(For custom worker containers)*
4.  **Click "CONTINUE"** -> Select template (e.g., "Cloud Pub/Sub to BigQuery") OR **"CUSTOM TEMPLATE"** to point to your Beam JAR/Python file in GCS.
5.  **Click "RUN JOB"**

#### **gcloud CLI (Batch Job Example - WordCount)**
```bash
gcloud dataflow jobs run wordcount-job \
  --region=us-central1 \
  --gcs-location=gs://dataflow-templates/latest/Word_Count \ # Public template
  --parameters \
inputFile=gs://dataflow-samples/shakespeare/kinglear.txt,\
output=gs://your-bucket/output/
```
*   **Mandatory Flags:** `--region`, `--gcs-location` (template path), `--parameters` (job-specific args)

#### **Terraform (Streaming PubSub to BigQuery)**
```hcl
resource "google_dataflow_job" "pubsub_to_bq" {
  name = "pubsub-to-bq-job"
  region = "us-central1"
  template_gcs_path = "gs://dataflow-templates/latest/Cloud_PubSub_to_BigQuery" # Public template
  temp_gcs_location = "gs://your-bucket/temp"
  parameters = {
    inputTopic    = "projects/your-project/topics/input-topic"
    outputTable   = "your-project:dataset.table"
    outputSchema  = "{\"fields\": [{\"name\":\"data\",\"type\":\"STRING\"}]}"
  }
  zone = "us-central1-f" # Optional but recommended
}
```
*   **Mandatory Attributes:** `name`, `region`, `template_gcs_path`, `temp_gcs_location`, `parameters`

---

### **2. Dataproc (Spark/Hadoop)**
*   **What it is:** Fully-managed **Spark and Hadoop** service. Creates ephemeral clusters for ETL, ML, SQL, and stream processing.
*   **Why it matters:** Faster cluster startup (< 2 mins), integrates with GCP IAM, managed OS/Spark updates, **Jupyter Notebooks out-of-the-box**.
*   **Key Concepts:**
    *   **Cluster:** Master + Worker nodes (Preemptible workers for cost savings).
    *   **Initialization Actions:** Scripts run on all nodes during creation (e.g., install packages).
    *   **Metastore:** Hive Metastore (managed or self-managed) for table metadata.
    *   **Component Gateway:** Secure access to cluster UIs (Spark History Server, YARN RM) via HTTPS.
    *   **Jupyter Integration:** Pre-installed JupyterLab on master node (accessible via Component Gateway).

#### **Creating a Dataproc Cluster via UI**
1.  **Go to Console:** `https://console.cloud.google.com/dataproc/clusters`
2.  **Click "+ CREATE CLUSTER"**
3.  **Basic Information:**
    *   **Name:** `your-cluster-name` *(Mandatory. Lowercase, hyphens)*
    *   **Region:** `us-central1` *(Mandatory)*
    *   **Zone:** `No preference` or specific zone *(Recommended for latency control)*
4.  **Cluster Configuration:**
    *   **Software:** *Version:* `2.1 (Debian)` *(Choose Spark/Hadoop version)*
        *   *Components:* Check `Jupyter` *(Mandatory for Notebook access)*, `Spark`, `Hive`, etc.
    *   **Master Node:**
        *   *Machine Type:* `n1-standard-4` *(Default)*
        *   *Boot Disk Type/Size:* `pd-ssd` / `100 GB` *(Adjust for OS/apps)*
    *   **Worker Nodes:**
        *   *Number of Workers:* `2` *(Mandatory)*
        *   *Machine Type:* `n1-standard-4` *(Default)*
        *   *Boot Disk Type/Size:* `pd-ssd` / `100 GB`
        *   *Preemptible Workers:* `2` *(Optional cost saver)*
    *   **Secondary Workers (for Spark):** *(Optional, for dynamic scaling)*
5.  **Node Pools (Advanced):** *(Optional. Define separate pools for master/workers)*
6.  **Network:**
    *   **Network:** `default` *(VPC)*
    *   **Subnetwork:** `default` *(Subnet)*
    *   **Internal IP:** `Primary IP range` *(Recommended for security)*
    *   **Firewall:** `Allow Google Cloud services and systems...` *(Auto-creates rules)*
7.  **Security:**
    *   **Service Account:** `default` *(Or custom SA with `dataproc.worker` role)*
    *   **Enable OS Login:** `Checked` *(Recommended for SSH)*
    *   **Enable Component Gateway:** `Checked` *(Mandatory for secure UI access)*
8.  **Labels:** *(Optional. Key/Value for cost tracking)*
9.  **Initialization Actions:** *(Optional. GCS path to script)*
10. **Click "CREATE"**

#### **gcloud CLI (Cluster with Jupyter)**
```bash
gcloud dataproc clusters create your-cluster \
  --region=us-central1 \
  --zone=us-central1-a \
  --master-machine-type=n1-standard-4 \
  --worker-machine-type=n1-standard-4 \
  --num-workers=2 \
  --image-version=2.1-debian11 \
  --optional-components=JUPYTER \ # Mandatory for Jupyter
  --enable-component-gateway
```
*   **Mandatory Flags:** `--region`, `--zone`, `--master-machine-type`, `--worker-machine-type`, `--num-workers`, `--image-version`

#### **Terraform (Basic Cluster)**
```hcl
resource "google_dataproc_cluster" "cluster" {
  name       = "dataproc-cluster"
  region     = "us-central1"
  zone       = "us-central1-a"

  cluster_config {
    gce_cluster_config {
      network = "default"
    }

    software_config {
      image_version    = "2.1-debian11"
      optional_components = ["JUPYTER"] # Mandatory for Jupyter
    }

    master_config {
      num_instances     = 1
      machine_type      = "n1-standard-4"
      disk_config { boot_disk_size_gb = 100 }
    }

    worker_config {
      num_instances     = 2
      machine_type      = "n1-standard-4"
      disk_config { boot_disk_size_gb = 100 }
    }

    endpoint_config {
      enable_http_port_access = true # For Component Gateway
    }
  }
}
```
*   **Mandatory Blocks:** `cluster_config`, `software_config` (with `image_version`), `master_config`, `worker_config`

---

### **3. Data Fusion (Visual ETL)**
*   **What it is:** **Code-free, visual ETL/ELT** tool built on CDAP (Cask Data Application Platform). Drag-and-drop pipelines for data integration.
*   **Why it matters:** Democratizes ETL for non-engineers. Pre-built connectors (100+), pipeline debugging, metadata management. *Not for complex transformations (use Dataflow/Spark).*
*   **Key Concepts:**
    *   **Pipeline:** Visual workflow (Source -> Transform -> Sink).
    *   **Wrangler:** Interactive data transformation UI (sample-based).
    *   **Connectors:** Plugins for sources/sinks (BigQuery, PubSub, JDBC, SaaS APIs).
    *   **Namespace:** Logical isolation (like a project within Data Fusion).
    *   **Hydrator:** Open-source core (Data Fusion = Hydrator + GCP managed service).

#### **Creating a Pipeline via UI**
1.  **Go to Console:** `https://console.cloud.google.com/data-fusion`
2.  **Click "CREATE INSTANCE"** (First time only)
    *   *Name:* `your-instance` *(Mandatory)*
    *   *Type:* `Basic` (Free tier) / `Enterprise` (Production)
    *   *Version:* `Latest`
    *   *Network:* `default` -> `default` (Subnet)
    *   *Service Account:* `default` (Ensure it has `datafusion.viewer`, `storage.objectAdmin`)
    *   *Click "CREATE"*
3.  **Open Instance -> Click "STUDIO"** (Top nav)
4.  **Click "+ Create Pipeline"** -> **"Batch Pipeline"**
5.  **Add Source:**
    *   Click "+" -> Select "BigQuery" connector
    *   **Configuration Fields:**
        *   *Reference Name:* `bq-source` *(Mandatory internal name)*
        *   *Project:* `your-project-id` *(Mandatory)*
        *   *Dataset:* `your_dataset` *(Mandatory)*
        *   *Table:* `your_table` *(Mandatory)*
        *   *SQL Query:* *(Optional. Overrides table)*
        *   *Service Account:* `default` (Uses instance SA)
6.  **Add Transform (e.g., Filter):**
    *   Drag "Filter" between source and sink
    *   **Configuration Fields:**
        *   *Reference Name:* `filter-1` *(Mandatory)*
        *   *Condition:* `age > 30` *(Mandatory. CDAP expression)*
7.  **Add Sink (e.g., BigQuery):**
    *   Click "+" -> Select "BigQuery" connector
    *   **Configuration Fields:**
        *   *Reference Name:* `bq-sink` *(Mandatory)*
        *   *Project:* `your-project-id` *(Mandatory)*
        *   *Dataset:* `output_dataset` *(Mandatory)*
        *   *Table:* `output_table` *(Mandatory)*
        *   *Create Dataset/Table:* `Checked` (If new)
        *   *Truncate Table:* `Checked` (For overwrite)
8.  **Click "SAVE"** -> **"PUBLISH"** -> **"START"**

#### **gcloud CLI (Create Instance Only - Pipelines require UI/API)**
```bash
gcloud data-fusion instances create your-instance \
  --location=us-central1 \
  --type=basic \
  --network=default \
  --range=10.128.0.0/20
```
*   **Mandatory Flags:** `--location`, `--type`, `--network`, `--range` (Private IP range)

#### **Terraform (Instance Only)**
```hcl
resource "google_data_fusion_instance" "instance" {
  name    = "data-fusion-instance"
  region  = "us-central1"
  version = "6.8.0" # Check latest
  type    = "BASIC" # or ENTERPRISE

  network_config {
    network       = "default"
    ip_allocation = "10.128.0.0/20" # Private range
  }
}
```
*   **Mandatory Attributes:** `name`, `region`, `version`, `type`, `network_config` (with `network` & `ip_allocation`)

---

### **4. Looker (BI & Visualization)**
*   **What it is:** **Modern BI platform** (acquired by Google) for modeling data (**LookML**), creating dashboards, and embedding analytics. Deeply integrated with BigQuery.
*   **Why it matters:** Single source of truth via LookML. Blocks accelerate development. Real-time BigQuery connectivity. *Replaces legacy GCP BI tools.*
*   **Key Concepts:**
    *   **LookML:** YAML-based modeling language (defines dimensions, measures, joins). *Not SQL!*
    *   **Model:** `.model.lkml` file (defines explores/views connection).
    *   **View:** `.view.lkml` file (defines fields from a table).
    *   **Explore:** Logical data set (joins views) for querying.
    *   **Dashboard:** Collection of Looks (visualizations).
    *   **Blocks:** Pre-built LookML/content (e.g., "Salesforce Block", "Google Analytics Block").

#### **Creating a Looker Dashboard via UI**
1.  **Go to Looker:** `https://your-looker-domain.looker.com` (Provisioned via GCP Console)
2.  **Create a View (via LookML):**
    *   *Develop* -> *Project* -> *New View File*
    *   **Mandatory Fields:**
        *   *Filename:* `orders.view.lkml` *(Mandatory. Must match view name)*
        *   *Content:*
          ```yaml
          view: orders {
            sql_table_name: bq_project.dataset.orders ;;
            dimension: order_id {
              type: string
              sql: ${TABLE}.order_id ;;
            }
            measure: total_sales {
              type: sum
              sql: ${TABLE}.amount ;;
            }
          }
          ```
3.  **Create a Model:**
    *   *New Model File* -> `sales.model.lkml`
    *   **Mandatory Fields:**
        ```yaml
        connection: "your_bigquery_connection" # Must exist
        include: "*.view.lkml"
        explore: orders {
          view_name: orders
        }
        ```
4.  **Create a Look (Visualization):**
    *   *Explore* -> Select `orders` explore
    *   Add `order_id` (Dimension) & `total_sales` (Measure)
    *   Click *Visualization* tab -> Choose chart type (e.g., "Table")
    *   *Save As* -> Name: `Top Sales` -> Folder: `Shared`
5.  **Create a Dashboard:**
    *   *Dashboards* -> *New Dashboard* -> *Standard*
    *   **Mandatory Fields:**
        *   *Dashboard Name:* `Sales Overview` *(Mandatory)*
        *   *Space:* `Shared` *(Mandatory)*
    *   Click "+ Add Tile" -> *Existing Look* -> Select `Top Sales`
    *   Configure tile size/position -> *Save*

#### **Using Looker Blocks (e.g., BigQuery Block)**
1.  *Develop* -> *Blocks* -> *Browse Marketplace*
2.  Search "BigQuery" -> Select "BigQuery Block" -> *Install*
3.  **Configuration Fields (Post-Install):**
    *   *Project ID:* `your-project-id` *(Mandatory)*
    *   *Dataset:* `looker_scratch` *(Mandatory. Scratch space)*
    *   *Click "Apply Changes"*

> **Note:** Looker provisioning is done via GCP Console (`https://console.cloud.google.com/looker`), but **content creation (LookML, Dashboards) is ONLY in Looker UI**. No direct `gcloud`/Terraform for LookML.

---

### **5. Vertex AI (ML Platform)**
*   **What it is:** **Unified ML platform** covering the *entire ML lifecycle*: experimentation (Workbench), training (Pipelines), deployment (Endpoints), and management (Feature Store, Model Registry).
*   **Why it matters:** Replaces AI Platform. Integrates AutoML, custom training, MLOps, and responsible AI. Single pane of glass.
*   **Key Concepts:**
    *   **Workbench:** Managed JupyterLab instances (with pre-installed ML libs).
    *   **Pipelines:** Orchestrate ML workflows (Kubeflow Pipelines engine). Define steps (preprocess, train, eval).
    *   **Feature Store:** Central repository for ML features (ingest, serve, share). *Critical for consistency.*
    *   **AutoML:** Train models without coding (Tables, Vision, NLP, Forecasting).
    *   **Explainable AI (XAI):** SHAP/Saliency maps to interpret model predictions.

#### **Creating a Vertex AI Feature Store via UI**
1.  **Go to Console:** `https://console.cloud.google.com/vertex-ai/feature-store`
2.  **Click "+ CREATE FEATURE STORE"**
3.  **Configuration Fields:**
    *   **Feature Store Name:** `your-feature-store` *(Mandatory. Lowercase, hyphens)*
    *   **Region:** `us-central1` *(Mandatory. Must match resource region)*
    *   **Online Serving Capacity:** `1` *(Node count. Scales automatically)*
    *   **Encryption:** `Google-managed key` *(Default)*
    *   **Click "CREATE"**

#### **Creating a Vertex AI Pipeline Job via UI**
1.  **Go to Console:** `https://console.cloud.google.com/vertex-ai/pipelines`
2.  **Click "+ CREATE PIPELINE JOB"**
3.  **Pipeline Configuration:**
    *   **Name:** `training-pipeline` *(Mandatory)*
    *   **Region:** `us-central1` *(Mandatory)*
    *   **Pipeline Package:** Upload `.tar.gz` (KFP DSL compiled) OR GCS path `gs://.../pipeline.yaml`
    *   **Display Name:** `Training Run 1` *(Mandatory)*
4.  **Parameters:** *(If pipeline expects inputs)*
    *   `learning_rate: 0.01`, `batch_size: 32`
5.  **Runtime Configuration:**
    *   **Service Account:** `default` (Ensure it has `aiplatform.user`)
    *   **Network:** `default` (VPC)
    *   **Enable Access to Google Services:** `Checked`
6.  **Click "SUBMIT"**

#### **gcloud CLI (Create Workbench Instance)**
```bash
gcloud ai notebooks instances create workbench-instance \
  --location=us-central1-a \
  --machine-type=n1-standard-4 \
  --install-gpu-driver \
  --container-image-uri="gcr.io/deeplearning-platform-release/pytorch-cpu.1-10:latest"
```
*   **Mandatory Flags:** `--location`, `--machine-type`

#### **Terraform (Feature Store)**
```hcl
resource "google_vertex_ai_featurestore" "featurestore" {
  location = "us-central1"
  project  = "your-project-id"
  name     = "terraform-featurestore" # Mandatory
  online_serving_config {
    fixed_node_count = 1 # Mandatory
  }
}
```
*   **Mandatory Attributes:** `location`, `name`, `online_serving_config` (with `fixed_node_count`)

---

### **6. BigQuery Advanced Analytics**
*   **What it is:** **Advanced analytics capabilities built directly into BigQuery** (serverless, no data movement).
*   **Why it matters:** Run ML, GIS, and custom logic *where your data lives*. Avoid ETL to separate systems.
*   **Key Concepts:**
    *   **BigQuery ML:** Create/train ML models (logistic regression, K-means, TensorFlow) using SQL.
    *   **GIS (Geospatial):** Analyze location data (`ST_GEOGPOINT`, `ST_DWithin`).
    *   **Remote Functions:** Call Cloud Functions from SQL to extend capabilities (e.g., call NLP API).

#### **BigQuery ML Example (Create Model)**
```sql
CREATE OR REPLACE MODEL `dataset.sales_forecast`
OPTIONS(
  model_type='ARIMA_PLUS', -- Time series
  time_series_timestamp_col='order_date',
  time_series_data_col='total_sales'
) AS
SELECT
  order_date,
  SUM(total_sales) AS total_sales
FROM `dataset.orders`
GROUP BY order_date
ORDER BY order_date;
```

#### **GIS Example (Find Stores near Customer)**
```sql
SELECT
  store_id,
  ST_Distance(
    ST_GEOGPOINT(-122.4194, 37.7749), -- Customer (SF)
    store_location
  ) AS distance_meters
FROM `dataset.stores`
WHERE
  ST_DWithin(
    ST_GEOGPOINT(-122.4194, 37.7749),
    store_location,
    5000 -- 5km radius
  )
ORDER BY distance_meters
LIMIT 5;
```

#### **Remote Function Setup (Call Cloud Function from SQL)**
1.  **Create Cloud Function (Python):**
    ```python
    def analyze_text(request):
        import requests
        text = request.get_json()['text']
        # Call some external API (e.g., sentiment analysis)
        response = requests.post("https://api.example.com/sentiment", json={"text": text})
        return {"sentiment": response.json()['score']}
    ```
    *   Deploy to `us-central1` (matches BQ region)
2.  **Create Remote Function in BigQuery UI:**
    *   *Compose New Query* -> *SQL Workspace* -> *Functions* -> *Create function*
    *   **Configuration Fields:**
        *   *Function Name:* `remote_sentiment` *(Mandatory)*
        *   *Function Type:* `Remote function` *(Mandatory)*
        *   *Endpoint:* `https://us-central1-your-project.cloudfunctions.net/analyze_text` *(Mandatory)*
        *   *Connection:* Create new connection `my-connection` (IAM-permissioned)
        *   *Input Data Type:* `JSON` *(Mandatory)*
        *   *Return Data Type:* `JSON` *(Mandatory)*
        *   *Click "Create function"*
3.  **Use in SQL:**
    ```sql
    SELECT
      text,
      remote_sentiment(STRUCT(text AS text)) AS sentiment
    FROM
      `dataset.user_reviews`;
    ```

#### **gcloud CLI (Deploy Cloud Function for Remote Function)**
```bash
gcloud functions deploy analyze_text \
  --runtime=python310 \
  --trigger-http \
  --region=us-central1 \
  --project=your-project-id \
  --allow-unauthenticated
```
*   **Mandatory Flags:** `--runtime`, `--trigger-http`, `--region`

---

### **Critical Summary & Best Practices**
| **Service**       | **When to Use**                                      | **Key Gotcha**                                      |
|-------------------|------------------------------------------------------|-----------------------------------------------------|
| **Dataflow**      | Complex stream/batch ETL, custom logic               | Autoscaling lags; monitor `backlog` metric          |
| **Dataproc**      | Spark/Hadoop jobs, Jupyter for exploration           | Cluster startup time; use preemptibles for workers  |
| **Data Fusion**   | Simple ETL, citizen integrators                      | Limited complex logic; check connector coverage     |
| **Looker**        | Enterprise BI, self-service dashboards               | LookML requires learning; model carefully           |
| **Vertex AI**     | End-to-end ML pipelines, model management            | Feature Store cost; XAI requires model compatibility|
| **BigQuery ML**   | Quick ML on BQ data, no ML expertise                 | Limited algorithms; not for deep learning           |

**Pro Tips:**
1.  **Cost Control:** Set `max-workers` in Dataflow, use preemptibles in Dataproc, delete idle Workbench instances.
2.  **Security:** Always use **Service Accounts** (not user accounts) for services. Apply least privilege.
3.  **Data Locality:** Match regions for Dataflow/Dataproc/Vertex AI with your BigQuery/GCS buckets.
4.  **Looker:** Start with **Blocks** for common data sources (Salesforce, GA4). Avoid duplicating logic in LookML.
5.  **Vertex AI Pipelines:** Store pipeline YAML in **Cloud Source Repositories**. Use **KFP v2** for Vertex AI.
6.  **BigQuery Remote Functions:** Use for **small data batches**; high latency kills large queries.

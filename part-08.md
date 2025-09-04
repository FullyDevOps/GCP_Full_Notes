### **1. Well-Architected Framework (6 Pillars)**  
GCP's framework ensures systems are built for **reliability, security, cost-efficiency, performance, operational excellence, and sustainability**. Each pillar has specific design principles.

#### **a) Reliability**  
*Ensures systems recover from failures and meet availability requirements.*  
- **Key Concepts**:  
  - **Fault Tolerance**: Design for component failures (e.g., multi-AZ deployments).  
  - **Automated Recovery**: Use health checks + auto-healing (e.g., Managed Instance Groups).  
  - **Testing**: Chaos Engineering (e.g., [Chaos Monkey](https://github.com/Netflix/chaosmonkey)).  
- **GCP Tools**:  
  - **Cloud Load Balancing**: Distributes traffic across regions/AZs.  
  - **Cloud Monitoring**: Alerts on downtime (SLOs/SLIs).  
  - **Cloud Run/Functions**: Stateless, auto-scaled services.  

#### **b) Security**  
*Protects data, systems, and infrastructure.*  
- **Key Concepts**:  
  - **Defense-in-Depth**: Layered security controls (network, identity, data).  
  - **Least Privilege**: Grant minimal permissions (via IAM).  
  - **Data Encryption**: At rest (CMEK) and in transit (TLS).  
- **GCP Tools**:  
  - **Cloud IAM**: Centralized access control.  
  - **Security Command Center**: Threat detection.  
  - **VPC Service Controls**: Prevent data exfiltration.  

#### **c) Cost Optimization**  
*Run workloads at lowest possible cost without sacrificing performance.*  
- **Key Concepts**:  
  - **Right-Sizing**: Match resources to actual needs (e.g., CPU/memory).  
  - **Spend Visibility**: Track costs with Billing Reports.  
  - **Automation**: Schedule non-prod resources to turn off.  
- **GCP Tools**:  
  - **Cost Management**: Budgets, alerts, recommender.  
  - **Committed Use Discounts (CUDs)**: Save up to 57% for sustained usage.  

#### **d) Performance Efficiency**  
*Use computing resources efficiently to meet system requirements.*  
- **Key Concepts**:  
  - **Scalability**: Scale horizontally (e.g., GKE, Cloud Spanner).  
  - **Caching**: Use Cloud CDN or Memorystore (Redis).  
  - **Latency Optimization**: Place resources near users (global LB).  
- **GCP Tools**:  
  - **Cloud Profiler**: Identify performance bottlenecks.  
  - **Cloud Trace**: Analyze latency in microservices.  

#### **e) Operational Excellence**  
*Run and monitor systems to deliver business value.*  
- **Key Concepts**:  
  - **Automation**: Infrastructure-as-Code (Terraform), CI/CD.  
  - **Observability**: Logs (Cloud Logging), metrics (Cloud Monitoring), traces.  
  - **Incident Response**: Runbooks, SRE practices.  
- **GCP Tools**:  
  - **Cloud Operations (formerly Stackdriver)**: Unified logging/monitoring.  
  - **Cloud Build**: CI/CD pipelines.  

#### **f) Sustainability**  
*Minimize environmental impact through efficient resource usage.*  
- **Key Concepts**:  
  - **Carbon-Aware Computing**: Run workloads in low-carbon regions (e.g., `europe-west4`).  
  - **Resource Efficiency**: Use serverless (Cloud Run) over VMs where possible.  
- **GCP Tools**:  
  - **Carbon Sense Suite**: Track carbon footprint in Billing.  
  - **Region Selection**: Choose regions with renewable energy (e.g., Finland, Netherlands).  

> **Key Takeaway**: All pillars are interdependent. E.g., over-provisioning (bad for *Cost*) might improve *Reliability* but hurts *Sustainability*.

---

### **2. Resilience & Disaster Recovery**  
*Ensure business continuity during disruptions.*

#### **a) RTO (Recovery Time Objective) & RPO (Recovery Point Objective)**  
- **RTO**: Max tolerable downtime (e.g., 1 hour).  
- **RPO**: Max tolerable data loss (e.g., 5 minutes).  
- **GCP Implementation**:  
  - **RTO Example**: Use Cloud SQL HA (multi-region failover; RTO ~30-60 sec).  
  - **RPO Example**: Spanner (RPO = 0 due to synchronous replication).  

#### **b) Multi-Region Strategies**  
- **Active-Passive**: Primary region handles traffic; secondary takes over during DR.  
  - *Tools*: Global Load Balancer (GLB) + Cloud DNS failover routing.  
- **Active-Active**: Traffic split across regions (e.g., 70/30).  
  - *Tools*: GLB with backend services in multiple regions.  

**How to Set Up Multi-Region DR for Cloud SQL (UI)**:  
1. **Enable HA**:  
   - Go to **Cloud SQL > [Instance] > EDIT**.  
   - Under **Availability**, select **High availability (regional)**.  
   - *Mandatory Fields*:  
     - **Region**: Primary region (e.g., `us-central1`).  
     - **Secondary zone**: Auto-selected (e.g., `us-central1-f`).  
2. **Configure Failover**:  
   - **Failover** button appears during outage (no pre-config needed).  
   - *RPO*: Near-zero (synchronous replication).  
   - *RTO*: ~30-60 seconds (automatic).  

**gcloud CLI**:  
```bash
gcloud sql instances patch [INSTANCE_NAME] \
  --availability-type=REGIONAL \
  --region=us-central1
```

**Terraform**:  
```hcl
resource "google_sql_database_instance" "ha_instance" {
  name             = "prod-db"
  region           = "us-central1" # Mandatory
  database_version = "POSTGRES_15"
  settings {
    availability_type = "REGIONAL" # Mandatory for HA
  }
}
```

---

### **3. Cost Optimization**  
*Reduce costs without compromising performance.*

#### **a) Committed Use Discounts (CUDs)**  
- **What**: Pre-pay for 1-3 years of vCPUs/RAM; save up to 57%.  
- **Types**:  
  - **Regular CUDs**: For specific machine types (e.g., `n2-standard-4`).  
  - **Sustained Use Discounts (SUDs)**: Automatic discount for >25% monthly usage (no commitment).  

**How to Buy CUDs (UI)**:  
1. Go to **Billing > Committed Use Discounts > + PURCHASE**.  
2. *Mandatory Fields*:  
   - **Billing Account**: Select account.  
   - **Resource Type**: `Compute Engine` (vCPU/RAM) or `BigQuery`.  
   - **Commitment Term**: 1 or 3 years.  
   - **Region**: Where resources run (e.g., `us-central1`).  
   - **vCPU/RAM Amount**: Minimum 1 vCPU.  
   - **Machine Type**: e.g., `n2-standard-4` (for Regular CUDs).  

**gcloud CLI**:  
```bash
gcloud beta billing commitments create \
  --billing-account=XXXXXX-XXXXXX-XXXXXX \
  --plan=12-month \
  --resources=VCPU=4,RAM=16 \
  --region=us-central1 \
  --machine-type=n2-standard-4
```

#### **b) Rightsizing**  
- **Steps**:  
  1. **Analyze**: Use **Recommendations** in Cloud Console (under *Compute Engine*).  
  2. **Resize**: Change machine type (e.g., from `n2-highmem-8` to `n2-standard-4`).  
  3. **Autoscale**: Use Managed Instance Groups (MIGs) with autoscaling.  

**UI for Rightsizing (Compute Engine)**:  
- **VM Instance > EDIT**:  
  - *Mandatory Fields*:  
    - **Machine Configuration**: Change "Machine type" (e.g., `e2-medium`).  
    - **Customize**: Adjust vCPU/RAM (e.g., 2 vCPU, 8 GB RAM).  

---

### **4. Security & Compliance Design**  
*Implement security-by-design principles.*

#### **a) Zero Trust**  
- **Principle**: "Never trust, always verify."  
- **GCP Implementation**:  
  - **BeyondCorp Enterprise**:  
    - **Access Context Manager**: Define access levels (e.g., "Require corp network + MFA").  
    - **Identity-Aware Proxy (IAP)**: Secure access to apps without VPN.  
  - **VPC Service Controls**: Create security perimeters to block data exfiltration.  

**How to Create Access Level (UI - Zero Trust)**:  
1. Go to **Security > BeyondCorp Enterprise > Access Levels > + CREATE**.  
2. *Mandatory Fields*:  
   - **Name**: e.g., `corp-network-access`.  
   - **Basic Level**:  
     - **IP ranges**: `10.0.0.0/8` (your corp IP).  
     - **Require MFA**: Toggle ON.  

#### **b) Data Loss Prevention (DLP)**  
- **What**: Scan/transform sensitive data (PII, credit cards).  
- **UI Setup**:  
  1. Go to **DLP > Jobs > + CREATE JOB**.  
  2. *Mandatory Fields*:  
     - **Input Data**: Select Cloud Storage bucket.  
     - **Info Types**: e.g., `CREDIT_CARD_NUMBER`.  
     - **Transformation**: e.g., "Mask" (replace with `XXXX`).  

**gcloud CLI (DLP Job)**:  
```bash
gcloud dlp jobs create inspect \
  --storage-path="gs://my-bucket/*" \
  --info-types=CREDIT_CARD_NUMBER \
  --inspect-config='{"info_types": [{"name": "CREDIT_CARD_NUMBER"}]}'
```

#### **c) Cloud Key Management Service (KMS)**  
- **What**: Manage encryption keys for customer-managed encryption keys (CMEK).  
- **UI Setup**:  
  1. Go to **Security > Cryptographic Keys > + CREATE KEY RING**.  
     - *Mandatory Fields*:  
       - **Location**: e.g., `global` (or regional).  
       - **Key Ring Name**: e.g., `prod-keyring`.  
  2. **+ CREATE KEY** in key ring:  
     - *Mandatory Fields*:  
       - **Key Name**: e.g., `db-encryption-key`.  
       - **Purpose**: `ENCRYPT_DECRYPT`.  

**Terraform (KMS)**:  
```hcl
resource "google_kms_key_ring" "keyring" {
  name     = "prod-keyring" # Mandatory
  location = "global"       # Mandatory
}

resource "google_kms_crypto_key" "key" {
  name     = "db-key"       # Mandatory
  key_ring = google_kms_key_ring.keyring.id
}
```

---

### **5. Migration Strategies**  
*Approaches to move workloads to GCP.*

#### **a) Lift & Shift (Rehost)**  
- **What**: Move VMs as-is (minimal changes).  
- **Tools**:  
  - **Migrate for Compute Engine**: Agent-based migration.  
  - **CloudEndure**: Real-time replication.  
- **UI Setup (Migrate for Compute Engine)**:  
  1. **Compute Engine > Migrate > + ADD SOURCE**.  
     - *Mandatory Fields*:  
       - **Source Platform**: e.g., "VMware".  
       - **Source Manager IP**: VMware vCenter IP.  
       - **Credentials**: vCenter username/password.  

#### **b) Refactor (Replatform)**  
- **What**: Minor optimizations (e.g., move DB to Cloud SQL).  
- **Example**:  
  - On-prem MySQL → Cloud SQL (no code changes).  
  - **UI**: **SQL > + CREATE INSTANCE** → Select "MySQL".  

#### **c) Rebuild (Rearchitect)**  
- **What**: Rewrite app for cloud-native (e.g., monolith → microservices on GKE).  
- **Tools**:  
  - **Anthos**: Modernize apps on-prem/cloud.  
  - **Cloud Run**: Serverless containers.  

---

### **6. Hybrid & Multi-Cloud Patterns**  
*Integrate GCP with on-prem/other clouds.*

#### **a) Anthos Service Mesh (ASM)**  
- **What**: Managed Istio for traffic control, security, and observability across clouds.  
- **UI Setup**:  
  1. **Anthos > Service Mesh > INSTALL**.  
  2. *Mandatory Fields*:  
     - **Project**: Select GCP project.  
     - **Cluster**: Select GKE cluster (must have workload identity enabled).  
     - **Mesh ID**: Auto-generated (e.g., `proj-12345`).  
  3. **Deploy**: ASM control plane installs on cluster.  

**gcloud CLI (ASM)**:  
```bash
gcloud container fleet mesh enable --project=MY_PROJECT
gcloud container fleet mesh update \
  --membership=CLUSTER_NAME \
  --control-plane=managed \
  --location=global
```

#### **b) Identity Federation**  
- **What**: Use existing identity providers (e.g., Active Directory) for GCP access.  
- **Patterns**:  
  - **SAML 2.0**: Federate with Azure AD, Okta.  
  - **Workload Identity Federation**: Grant AWS/Azure roles access to GCP.  
- **UI Setup (SAML with Azure AD)**:  
  1. **IAM & Admin > Identity Federation > + ADD IDENTITY PROVIDER**.  
     - *Mandatory Fields*:  
       - **Provider Name**: e.g., `AzureAD`.  
       - **SAML Metadata**: Paste XML from Azure AD.  
       - **Attribute Mapping**: `googleSubject` → `user.mail`.  
  2. **Create Service Provider (SP) Metadata**: Download XML to configure Azure AD.  

**Terraform (Workload Identity Federation - AWS)**:  
```hcl
resource "google_iam_workload_identity_pool" "pool" {
  workload_identity_pool_id = "aws-pool" # Mandatory
}

resource "google_iam_workload_identity_pool_provider" "provider" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.pool.workload_identity_pool_id
  workload_identity_pool_provider_id = "aws-provider" # Mandatory
  attribute_mapping = {
    "google.subject" = "assertion.arn"
  }
  aws {
    account_id = "123456789012" # Mandatory (AWS account ID)
  }
}
```

---

### **Critical Best Practices Summary**  
| **Area**               | **Do This**                                                                 | **Avoid This**                                     |
|------------------------|-----------------------------------------------------------------------------|----------------------------------------------------|
| **Reliability**        | Use multi-region databases (Spanner/Cloud SQL HA)                           | Single-zone deployments without backups            |
| **Cost**               | Apply SUDs/CUDs + rightsizing quarterly                                     | Over-provisioning VMs without monitoring           |
| **Security**           | Enable VPC Service Controls + CMEK for all data                             | Using default service accounts with broad roles    |
| **Migration**          | Start with lift & shift, then refactor incrementally                        | "Big bang" re-architecting without testing         |
| **Hybrid Cloud**       | Use Anthos for consistent operations across environments                    | Managing separate toolchains for each cloud        |

---

### **Essential Resources**  
1. **[GCP Well-Architected Framework](https://cloud.google.com/architecture/framework)** (Official guide with checklists).  
2. **[Disaster Recovery Playbook](https://cloud.google.com/solutions/dr-scenarios-playbook)** (RTO/RPO templates).  
3. **[Cost Optimization Labs](https://www.qwiklabs.com/quests/91)** (Hands-on CUDs/rightsizing).  
4. **[Zero Trust Architecture](https://cloud.google.com/beyondcorp-enterprise)** (BeyondCorp Enterprise docs).  

> **Pro Tip**: Always enable **Cloud Logging + Monitoring** during setup – 90% of operational issues are caught here first. Use **Terraform** for all production deployments (never manual UI changes).

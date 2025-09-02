### **1. Kubernetes Engine (GKE)**
#### **Autopilot vs. Standard**
*   **Core Concept**:  
    *   **Autopilot**: Fully managed control plane **AND** node infrastructure. GCP handles node provisioning, scaling, security, and optimization. You define *workloads*, GCP manages *infrastructure*. Pay for vCPU/RAM used by pods (not nodes).  
    *   **Standard**: You manage node pools (VMs). Full control over node types, OS, networking, and scaling. Pay for provisioned nodes (even if idle).  
*   **When to Use**:  
    *   **Autopilot**: Simplicity, security, cost-efficiency for stateless apps. Avoid if you need OS/node-level customization, specific hardware (NVMe), or Windows nodes.  
    *   **Standard**: Full infrastructure control, custom node images, Windows nodes, specific hardware (TPUs/GPUs), or complex networking needs.  
*   **Key Differences**:  
    | Feature                | Autopilot                          | Standard                          |
    | :--------------------- | :--------------------------------- | :-------------------------------- |
    | **Node Management**    | Fully Managed by GCP               | User-Managed                      |
    | **Pricing**            | Per vCPU/RAM *used by pods*        | Per VM *provisioned*              |
    | **Node Pools**         | Not visible/managed                | User-defined & managed            |
    | **Customization**      | Limited (OS, kernel params)        | Full (OS, disk, network, labels)  |
    | **Max Pods per Node**  | Fixed (~100)                       | Configurable (up to 256)          |
    | **Node Auto-Provisioning** | Always On (implicit)           | Optional Feature                  |
    | **Windows Nodes**      | ❌ Not Supported                   | ✅ Supported                      |

#### **Workload Identity (WI)**
*   **Core Concept**: Securely connect GKE workloads to Google Cloud services **using Kubernetes service accounts (KSAs)** instead of static keys. Maps a KSA to a Google Cloud IAM service account (GSA).  
*   **Why it Matters**: Eliminates need for service account key files. Granular permissions per pod. Industry best practice.  
*   **How it Works**:  
    1.  Create a GSA (e.g., `gke-app-sa@project-id.iam.gserviceaccount.com`).  
    2.  Grant IAM roles to the GSA (e.g., `roles/storage.objectViewer`).  
    3.  Annotate the KSA with the GSA email: `iam.gke.io/gcp-service-account=gke-app-sa@project-id.iam.gserviceaccount.com`.  
    4.  Pod uses the KSA → GKE metadata server validates mapping → Pod gets temporary tokens for GSA permissions.  
*   **UI Setup (Cluster Level - Mandatory)**:  
    1.  **Create Cluster** (Autopilot or Standard) → **Security** tab.  
    2.  **Workload Identity**: Toggle **ON**.  
        *   *Field: "Workload Identity"*: Toggle ON (Mandatory for WI).  
        *   *Field: "Workload Identity Pools"*: Auto-filled (e.g., `project-id.svc.id.goog`). **DO NOT CHANGE**.  
    3.  *After Cluster Creation*:  
        *   **IAM & Admin → Service Accounts**: Create GSA (e.g., `gke-app-sa`).  
        *   **IAM**: Grant IAM roles to GSA (e.g., Storage Object Viewer).  
        *   **GKE → Clusters → [Your Cluster] → Node Pools**: Edit default node pool → **Security** → **Workload Identity** → Select **Enable Workload Identity**.  
        *   **GKE → Workloads → [Your Namespace] → Service Accounts**: Create KSA (e.g., `app-sa`).  
        *   **Edit KSA YAML**: Add annotation:  
            ```yaml
            apiVersion: v1
            kind: ServiceAccount
            metadata:
              name: app-sa
              namespace: default
              annotations:
                iam.gke.io/gcp-service-account: gke-app-sa@project-id.iam.gserviceaccount.com # MANDATORY
            ```
*   **gcloud (Mandatory Steps)**:  
    ```bash
    # Enable WI on cluster (if not done during creation)
    gcloud container clusters update CLUSTER_NAME \
        --workload-pool=PROJECT_ID.svc.id.goog \
        --zone=ZONE

    # Create GSA & Grant Role
    gcloud iam service-accounts create gke-app-sa
    gcloud projects add-iam-policy-binding PROJECT_ID \
        --member="serviceAccount:gke-app-sa@PROJECT_ID.iam.gserviceaccount.com" \
        --role="roles/storage.objectViewer"

    # Bind KSA to GSA (REPLACES UI Annotation)
    gcloud iam service-accounts add-iam-policy-binding \
        gke-app-sa@PROJECT_ID.iam.gserviceaccount.com \
        --role="roles/iam.workloadIdentityUser" \
        --member="serviceAccount:PROJECT_ID.svc.id.goog[default/app-sa]" # NAMESPACE/KSA_NAME
    ```
*   **Terraform (Minimal)**:  
    ```hcl
    # Cluster with WI Enabled
    resource "google_container_cluster" "primary" {
      name     = "wi-cluster"
      location = "us-central1"
      # ... (other mandatory fields: initial_node_count, etc.)

      workload_identity_config {
        workload_pool = "${var.project_id}.svc.id.goog" # MANDATORY FORMAT
      }
    }

    # GSA & Binding
    resource "google_service_account" "gke_app" {
      account_id = "gke-app-sa"
    }

    resource "google_project_iam_member" "gke_app_storage" {
      role   = "roles/storage.objectViewer"
      member = "serviceAccount:${google_service_account.gke_app.email}"
    }

    # Bind KSA to GSA (Happens at Pod/KSA level, TF often manages KSA via k8s provider)
    # This binding is the critical IAM link
    resource "google_service_account_iam_member" "wi_binding" {
      service_account_id = google_service_account.gke_app.name
      role               = "roles/iam.workloadIdentityUser"
      member             = "serviceAccount:${var.project_id}.svc.id.goog[default/app-sa]" # NAMESPACE/KSA
    }
    ```

#### **Node Pools (Standard Clusters Only)**
*   **Core Concept**: Group of identical VMs (nodes) running your workloads. Autopilot has *implicit*, unmanaged node pools.  
*   **Key Configurations**:  
    *   **Machine Type**: `e2-medium` (default), `n2-standard-4`, `g2-standard-4` (GPU), etc.  
    *   **Disk Size/Type**: Boot disk size (GB) and type (`pd-standard`, `pd-ssd`, `pd-balanced`).  
    *   **Autoscaling**: Min/Max nodes per zone/region.  
    *   **Node Labels/Taints**: Schedule pods on specific pools (e.g., `accelerator=nvidia-tesla-t4:NoSchedule`).  
    *   **Node Metadata**: `enable-oslogin`, `disable-legacy-endpoints`.  
*   **UI Creation (Mandatory Fields)**:  
    1.  **GKE → Clusters → [Your Cluster] → Node Pools → Add Node Pool**.  
    2.  **Basic Settings**:  
        *   *Name*: Unique identifier (e.g., `gpu-pool`).  
        *   *Machine Configuration*: **Machine type** (Mandatory - e.g., `n1-standard-2`).  
        *   *Node count*: **Number of nodes** (Mandatory for non-autoscaling pools).  
    3.  **Advanced Settings → Node Pools**:  
        *   *Boot disk*: **Size (GB)** (Mandatory - min 10GB), **Type** (Mandatory - `pd-ssd` recommended).  
        *   *Networking*: **Node labels** (Optional), **Node taints** (Optional - key/value/effect).  
        *   *Security*: **Enable Workload Identity** (Mandatory if using WI), **Enable Secure Boot** (Shielded VMs).  
        *   *Management*: **Enable autoscaling** → **Minimum nodes**, **Maximum nodes** (Mandatory if enabled).  
*   **gcloud (Mandatory)**:  
    ```bash
    gcloud container node-pools create gpu-pool \
        --cluster=standard-cluster \
        --zone=us-central1-a \
        --machine-type=n1-standard-4 \ # MANDATORY
        --num-nodes=2 \                # MANDATORY (or --enable-autoscaling --min-nodes=1 --max-nodes=5)
        --disk-type=pd-ssd \           # MANDATORY if not default
        --disk-size=100 \              # MANDATORY if not default
        --workload-metadata=GKE_METADATA # For WI (Mandatory if using WI)
    ```
*   **Terraform (Minimal)**:  
    ```hcl
    resource "google_container_node_pool" "gpu_pool" {
      name       = "gpu-pool"
      location   = "us-central1-a"
      cluster    = google_container_cluster.standard.name # MANDATORY (ref to cluster)
      node_count = 2                                      # MANDATORY (or autoscaling block)

      node_config {
        machine_type = "n1-standard-4" # MANDATORY
        disk_size_gb = 100             # MANDATORY if not default
        disk_type    = "pd-ssd"        # MANDATORY if not default

        oauth_scopes = [
          "https://www.googleapis.com/auth/cloud-platform" # MANDATORY for most services
        ]

        workload_metadata_config {
          node_metadata = "GKE_METADATA" # MANDATORY for WI
        }
      }

      # Autoscaling (Alternative to node_count)
      # autoscaling {
      #   min_node_count = 1
      #   max_node_count = 5
      # }
    }
    ```

---

### **2. Anthos (Hybrid/Multi-Cloud)**
*   **Core Concept**: Manage GKE clusters consistently **across Google Cloud, on-premises (Bare Metal, VMware), and other clouds (AWS, Azure)**. Not just Kubernetes - includes service mesh, policy, and configuration management.  
*   **Key Components**:  
    *   **Anthos Config Management (ACM)**: Enforce config/policy across clusters using GitOps (Config Sync pulls configs from Git repo).  
    *   **Anthos Service Mesh (ASM)**: Managed Istio for traffic management, security (mTLS), and observability across clusters.  
    *   **Anthos Fleet**: Central dashboard (Cloud Console) to view/manage *all* clusters (GKE, Anthos attached clusters) in one place.  

#### **Anthos Config Management (ACM)**
*   **How it Works**:  
    1.  Define Kubernetes configs (Namespaces, Policies, Deployments) in a **Git repository** (e.g., Cloud Source Repositories).  
    2.  ACM's **Config Sync** agent on each cluster **pulls** configs from Git.  
    3.  **Policy Controller** (Gatekeeper) enforces constraints (e.g., "all pods must have resource limits").  
*   **UI Setup (Mandatory)**:  
    1.  **Anthos → Config Management → Install**.  
    2.  **Select Clusters**: Check clusters to manage (Mandatory).  
    3.  **Git Repository Settings**:  
        *   *Repository URL*: `https://source.developers.google.com/p/PROJECT_ID/r/REPO_NAME` (Mandatory).  
        *   *Repository directory*: Path within repo (e.g., `acm-config`) (Mandatory).  
        *   *Sync branch*: `main` (Mandatory).  
        *   *Google Cloud Service Account*: Auto-created SA (e.g., `acm-repo-sync@...`) (Mandatory - grants Git read access).  
    4.  **Policy Controller**: Toggle **ON** to enable policy enforcement (Optional but recommended).  
*   **gcloud (Mandatory)**:  
    ```bash
    # Enable ACM on cluster (requires cluster name/location)
    gcloud alpha container hub config-management enable \
        --project=PROJECT_ID \
        --membership=CLUSTER_NAME \ # MANDATORY (cluster identifier)
        --config=/path/to/acm-config.yaml # MANDATORY (defines git repo, etc.)
    ```
    *Sample `acm-config.yaml`*:  
    ```yaml
    applySpec: # MANDATORY ROOT
      git: # MANDATORY SECTION
        syncRepo: https://source.developers.google.com/p/PROJECT_ID/r/REPO_NAME # MANDATORY
        policyDir: acm-config # MANDATORY
        syncBranch: main # MANDATORY
        secretType: gcpserviceaccount # MANDATORY
        gcpServiceAccountEmail: acm-repo-sync@PROJECT_ID.iam.gserviceaccount.com # MANDATORY
    ```

#### **Anthos Service Mesh (ASM)**
*   **Core Concept**: Managed Istio control plane. Handles service-to-service communication (mTLS, traffic routing, telemetry).  
*   **UI Setup (Mandatory)**:  
    1.  **Anthos → Service Mesh → Install**.  
    2.  **Select Clusters**: Choose clusters to install ASM (Mandatory).  
    3.  **Installation Options**:  
        *   *Installation level*: `Regular` (most common) or `Restricted` (stricter security) (Mandatory).  
        *   *Control plane location*: `us-central1` (default) (Mandatory for multi-cluster).  
        *   *Enable mTLS*: Toggle **ON** (Strongly Recommended - Mandatory for secure mesh).  
        *   *Enable automatic sidecar injection*: Toggle **ON** (Recommended - injects Envoy proxy automatically).  
    4.  **Identity Provider**: `Google` (for GKE clusters) (Mandatory).  
*   **gcloud (Mandatory)**:  
    ```bash
    # Install ASM (simplified - requires ASM installation files)
    gcloud mesh install \
        --output_dir=./asm-install \ # MANDATORY (temp dir)
        --project_id=PROJECT_ID \    # MANDATORY
        --cluster_name=CLUSTER_NAME \ # MANDATORY
        --cluster_location=ZONE \    # MANDATORY
        --mode=install \             # MANDATORY
        --channel=regular            # MANDATORY (installation level)
    kubectl apply -f ./asm-install/ # Applies the generated manifests
    ```

#### **Anthos Fleet**
*   **Core Concept**: Single pane of glass in Cloud Console for **all clusters** (GKE, Anthos attached clusters on-prem/AWS/Azure).  
*   **UI Usage**:  
    1.  **Anthos → Fleet**: Shows all registered clusters.  
    2.  **Key Fields per Cluster**:  
        *   *Cluster ID*: Unique identifier.  
        *   *Location*: Cloud region or on-prem location.  
        *   *Connect Agent*: Status (`CONNECTED`, `OFFLINE`).  
        *   *Anthos Features*: Status of ACM, ASM, etc.  
    3.  **Register Cluster (On-prem/AWS)**: Requires downloading/connecting `gkeconnect` agent (out of scope for minimal setup).  

---

### **3. Advanced VPC Networking**
#### **Shared VPC**
*   **Core Concept**: **One project (Host Project)** owns VPC networks/subnets. **Other projects (Service Projects)** attach to those subnets to create resources (VMs, GKE nodes). Centralized networking control, decentralized resource management.  
*   **UI Setup (Mandatory Steps)**:  
    *   **Host Project Setup**:  
        1.  **VPC Network → Shared VPC → Enable Shared VPC**.  
        2.  **VPC Network → VPC networks → [Your VPC] → Enable Shared VPC hosting** (Button).  
        3.  **Shared VPC → Attached projects → + ATTACH PROJECT**.  
            *   *Project*: Select Service Project (Mandatory).  
            *   *Subnets*: Select subnets to share (Mandatory - e.g., `subnet-us-central1`).  
    *   **Service Project Usage**:  
        1.  Create VM/GKE cluster → **Network** dropdown → Select **Shared VPC network** (e.g., `host-project/vpc-name`).  
        2.  **Subnetwork**: Select the shared subnet (e.g., `subnet-us-central1`).  
*   **gcloud (Mandatory)**:  
    ```bash
    # Host Project: Enable hosting
    gcloud compute shared-vpc enable HOST_PROJECT_ID

    # Host Project: Attach Service Project & Subnet
    gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
        --host-project=HOST_PROJECT_ID

    gcloud compute shared-vpc subnets add-association SUBNET_NAME \
        --region=us-central1 \
        --project=HOST_PROJECT_ID \
        --service-project-namespace=SERVICE_PROJECT_ID \
        --subnetwork=SUBNET_NAME # MANDATORY (name in host project)
    ```
*   **Terraform (Minimal - Host Project Side)**:  
    ```hcl
    resource "google_compute_shared_vpc_host_project" "host" {
      project = "host-project-id" # MANDATORY
    }

    resource "google_compute_shared_vpc_service_project" "service" {
      host_project    = google_compute_shared_vpc_host_project.host.project # MANDATORY
      service_project = "service-project-id" # MANDATORY
    }

    # Association is done implicitly by creating resources in service project using host's subnet
    ```

#### **VPC Peering**
*   **Core Concept**: Connect **two VPC networks** (in same/different projects, same/different orgs, even on-prem via HA VPN) **privately**. Traffic stays within Google's network. *Not transitive*.  
*   **UI Setup (Mandatory)**:  
    1.  **VPC Network → VPC network peering → Create Connection**.  
    2.  **Name**: `peer-to-project-b` (Mandatory).  
    3.  **Peer VPC Network**:  
        *   *Project*: `Project ID of Peer` (Mandatory).  
        *   *VPC network*: `peer-vpc-network` (Mandatory).  
    4.  **Export/Import Custom Routes**: Toggle **ON** if you need to share custom routes (e.g., on-prem routes via Interconnect) (Optional but common).  
    *   **CRITICAL**: **BOTH SIDES** must create the peering connection (Initiator + Accepter).  
*   **gcloud (Mandatory - Initiator Side)**:  
    ```bash
    gcloud compute networks peerings create peer-to-project-b \
        --network=vpc-a \          # MANDATORY (your VPC)
        --peer-project=project-b \ # MANDATORY (peer project)
        --peer-network=vpc-b       # MANDATORY (peer VPC name)
    ```
    *Accepter Side (Project B Admin must run)*:  
    ```bash
    gcloud compute networks peerings accept peer-to-project-b \
        --network=vpc-b \          # MANDATORY (peer's VPC)
        --invitation-id=INVITE_ID  # MANDATORY (from initiator's output)
    ```

#### **Cloud Interconnect / Cloud VPN**
*   **Cloud Interconnect (Dedicated/Partner)**:  
    *   **Core Concept**: **High-bandwidth, low-latency physical connection** from on-prem to GCP. Dedicated (private fiber) or Partner (via service provider). Replaces HA VPN for large-scale needs.  
    *   **UI Setup (Simplified)**:  
        1.  **Network Connectivity → Interconnect → VLAN Attachments → + CREATE ATTACHMENT**.  
        2.  *Name*, *Region*, *Cloud Router* (auto-created).  
        3.  *Interconnect type*: `Dedicated` or `Partner` (Mandatory).  
        4.  *VLAN attachment details*: Provided by Google/partner (BGP ASN, IP ranges).  
        *Requires coordination with Google/partner sales.*  
*   **Cloud VPN (HA)**:  
    *   **Core Concept**: **IPsec VPN tunnel** over public internet between on-prem and GCP. HA = 2 tunnels for redundancy.  
    *   **UI Setup (Mandatory)**:  
        1.  **Network Connectivity → VPN → Create HA VPN gateway**.  
        2.  *Name*, *Region*, *Network* (VPC) (Mandatory).  
        3.  **Create Tunnel**:  
            *   *Name*, *VPC network*, *Gateway IP address* (auto).  
            *   *Peer gateway IP address*: Your on-prem firewall IP (Mandatory).  
            *   *Shared secret*: Strong pre-shared key (Mandatory).  
            *   *Routing options*: `Policy-based` (static routes) or `Dynamic` (BGP - Recommended).  
        4.  **Configure BGP (If Dynamic)**:  
            *   *Cloud Router name*, *BGP ASN* (e.g., `16551`), *Peer BGP ASN* (on-prem ASN - Mandatory).  
            *   *Peer IP address*: On-prem BGP IP (Mandatory).  

#### **Cloud Armor**
*   **Core Concept**: **WAF (Web Application Firewall) and DDoS protection** for HTTP(S) Load Balancers. Uses security policies with rules (IP lists, expressions, rate limiting).  
*   **UI Setup (Mandatory)**:  
    1.  **Security → Cloud Armor → Create Security Policy**.  
    2.  *Name*: `prod-waf-policy` (Mandatory).  
    3.  *Description*: (Optional).  
    4.  **Rules**:  
        *   *Rule 1 (Default)*:  
            *   *Action*: `deny(403)` (Mandatory for WAF).  
            *   *Preview*: Toggle OFF (Mandatory for production).  
            *   *Expression*: `evaluatePreconfiguredExpr('xss-stable')` (Mandatory for WAF rule - uses pre-defined XSS rule).  
            *   *Priority*: `1000` (Lower = higher priority).  
        *   *Rule 2 (Allowlist)*:  
            *   *Action*: `allow` (Mandatory).  
            *   *Expression*: `origin.ip in ipset("trusted-ips")` (Mandatory - requires IP list).  
            *   *Priority*: `100` (Higher priority than WAF rule).  
    5.  **IP Address Lists (If used)**:  
        *   **Security → Cloud Armor → IP Address Lists → Create IP Address List**.  
        *   *Name*: `trusted-ips` (Mandatory).  
        *   *IP addresses*: `192.0.2.0/24, 203.0.113.10` (Mandatory - comma separated).  
    6.  **Attach to Backend Service**:  
        *   **Network Services → Backend services → [Your Service] → Edit → Security policy → Select `prod-waf-policy`**.  

---

### **4. Advanced Compute**
#### **Managed Instance Groups (MIGs)**
*   **Core Concept**: Group of **identical VMs** managed as a single entity. Enables autoscaling, auto-healing, rolling updates. Foundation for GKE node pools, regional services.  
*   **UI Setup (Mandatory)**:  
    1.  **Compute Engine → Instance groups → Create Instance Group**.  
    2.  *Name*: `web-mig` (Mandatory).  
    3.  *Location*: `Single zone` or `Region` (Mandatory - Regional for HA).  
    4.  *Region/Zone*: `us-central1` or `us-central1-a` (Mandatory).  
    5.  *Instance template*: Select existing or **Create instance template** (Mandatory).  
        *   *Instance Template UI*:  
            *   *Name*: `web-template` (Mandatory).  
            *   *Machine configuration*: **Machine type** (Mandatory - e.g., `e2-medium`).  
            *   *Boot disk*: **Image** (Mandatory - e.g., `debian-11`).  
            *   *Networking*: **Network tags** (Optional - for firewall rules), **Network service tier** (Optional).  
    6.  *Autoscaling*: Toggle **ON** → Set **Minimum**, **Maximum** instances (Mandatory if enabled).  
    7.  *Autohealing*: Toggle **ON** → **Health check** (Mandatory - e.g., HTTP on port 80).  

#### **Shielded VMs**
*   **Core Concept**: **Hardened VMs** using UEFI firmware, Secure Boot, vTPM (for integrity measurements), and VM Integrity Monitoring. Protects against boot-level malware.  
*   **UI Setup (Mandatory - in Instance Template/MIG)**:  
    1.  **Instance Template or MIG Creation → Security**:  
        *   *Secure Boot*: Toggle **ON** (Mandatory for Shielded VM).  
        *   *vTPM*: Toggle **ON** (Mandatory for integrity measurements).  
        *   *Integrity monitoring*: Toggle **ON** (Mandatory - logs to Cloud Logging).  
    2.  *Requires OS Support*: Debian, CentOS, RHEL, Windows Server 2012+.  

#### **Confidential VMs**
*   **Core Concept**: **Encrypt VM memory** using AMD SEV-ES or Intel TDX. Protects data *while in use* from host OS/kernel attacks. Requires specific machine types (`C2D`, `N2D`, `T2D`).  
*   **UI Setup (Mandatory - in Instance Template/MIG)**:  
    1.  **Instance Template or MIG Creation → Machine configuration**:  
        *   *Machine type*: Select **Confidential VM type** (e.g., `c2d-standard-4`) (Mandatory).  
        *   *Confidential VM settings*: Toggle **ON** (Mandatory - auto-enabled for C2D types).  
    2.  *Requires OS Support*: Specific Linux kernels (Debian 11+, RHEL 8.4+, etc.) or Windows Server 2022+.  

#### **GPUs/TPUs**
*   **GPUs (NVIDIA)**: Attached to VMs for ML/AI/HPC. Requires GPU quota.  
    *   **UI Setup (Mandatory)**:  
        *   *Machine configuration*: **GPU type** (e.g., `NVIDIA Tesla T4`), **Number of GPUs** (Mandatory - e.g., `1`).  
        *   *Container-Optimized OS*: Recommended for GKE/GPU workloads.  
        *   *Install NVIDIA drivers*: Toggle **ON** (Mandatory - auto-installs drivers on boot).  
*   **TPUs (Tensor Processing Units)**: Google's custom ASIC for ML training/inference. Accessed via TPU VMs or as a service.  
    *   **UI Setup (TPU VM - Simplified)**:  
        1.  **Compute Engine → TPUs → TPU VMs → Create TPU VM**.  
        2.  *Name*, *Zone*, *TPU type* (e.g., `v3-8`) (Mandatory).  
        3.  *Software version* (e.g., `tpu-vm-pt-1.12` for PyTorch) (Mandatory).  
        4.  *Network*: VPC network (Mandatory).  

---

### **5. Advanced Storage**
#### **Filestore**
*   **Core Concept**: **Fully managed NFS file server** (v3/v4.1). For stateful apps needing shared file storage (e.g., CMS, media processing).  
*   **UI Setup (Mandatory)**:  
    1.  **Filestore → Create Instance**.  
    2.  *Instance ID*: `prod-nfs` (Mandatory).  
    3.  *Location*: `us-central1` (Mandatory).  
    4.  *Tier*: `Basic HDD`, `Basic SSD`, `High Scale SSD` (Mandatory - choose based on perf needs).  
    5.  *Capacity*: `1 TB` (Mandatory - min 1TB for Basic).  
    6.  *Network*: VPC network (Mandatory).  
    7.  *File share*: `vol1` (Mandatory - name of the NFS share).  
    *   **Mounting on VM**:  
        ```bash
        sudo apt-get install nfs-common
        sudo mkdir /mnt/filestore
        sudo mount 10.0.0.2:/vol1 /mnt/filestore # IP from Filestore instance details
        ```

#### **Persistent Disk Types**
*   **Core Concept**: Block storage for VMs. Types:  
    *   `pd-standard`: HDD-based. Cost-effective.  
    *   `pd-balanced`: Balanced SSD. Good price/performance.  
    *   `pd-ssd`: High-performance SSD.  
    *   `pd-extreme`: Highest throughput/low latency SSD (for extreme workloads).  
    *   `local-ssd`: Ephemeral NVMe/SCSI SSD *physically attached* to VM host (high perf, **data lost on stop**).  
*   **UI Setup (Mandatory - when creating VM/Instance Template)**:  
    *   **Boot Disk**:  
        *   *Type*: `pd-ssd`, `pd-balanced`, etc. (Mandatory - default `pd-balanced`).  
        *   *Size*: `50 GB` (Mandatory - min 10GB).  
    *   **Additional Disks**:  
        *   *Type*, *Size*, *Mode* (`Read/Write` or `Read Only`), *Delete boot disk* (on VM delete).  

#### **Storage Transfer Service (STS)**
*   **Core Concept**: **Import/export data** between GCS, AWS S3, Azure Blob, or on-prem (via agent) **at scale**. Uses network-optimized transfers.  
*   **UI Setup (Mandatory - GCS to GCS)**:  
    1.  **Storage → Transfer → Create Transfer Job**.  
    2.  *Name*: `s3-to-gcs-migration` (Mandatory).  
    3.  *Source*: `Amazon S3` → Enter **Bucket**, **Access Key ID**, **Secret Access Key** (Mandatory).  
    4.  *Destination*: `Google Cloud Storage` → **Bucket** (Mandatory).  
    5.  *Transfer options*:  
        *   *Schedule*: `Now` or `Custom` (Mandatory).  
        *   *Object conditions*: `Include`/`Exclude` prefixes, last modified time (Optional).  
        *   *Transfer options*: `Overwrite existing files`, `Delete files from source after transfer` (Optional).  

---

### **6. Operations Suite (Monitoring & Logging)**
#### **Cloud Monitoring (formerly Stackdriver)**
*   **Core Concept**: **Metrics, dashboards, alerts** for GCP and AWS resources, Kubernetes, custom apps.  
*   **Key UI Elements**:  
    *   **Metrics Explorer**: Build charts (Resource type → Metric → Filter/Aggregation).  
    *   **Alerting**:  
        1.  **Alerting → Create Policy**.  
        2.  *Name*: `high-cpu-alert` (Mandatory).  
        3.  *Conditions*:  
            *   *Target*: `GCE VM Instance` (Mandatory).  
            *   *Filter*: `metric.type = "compute.googleapis.com/instance/cpu/utilization"` (Mandatory).  
            *   *Configuration*: `Above` `0.8` for `5` `minutes` (Mandatory - threshold/duration).  
        4.  *Notification channels*: Select email/SMS channels (Mandatory).  
    *   **Dashboards**: Create custom dashboards with charts.  

#### **Cloud Logging**
*   **Core Concept**: **Centralized log management**. Ingests logs from GCP, AWS, on-prem, custom apps.  
*   **Key UI Elements**:  
    *   **Logs Explorer**:  
        *   *Resource*: Filter by GCE VM, GKE Cluster, etc. (Mandatory for context).  
        *   *Log name*: `syslog`, `container.log`, `cloudaudit.googleapis.com/activity` (Mandatory to select log stream).  
        *   *Query*: Build using `resource.type="gce_instance" AND jsonPayload.level="ERROR"` (Mandatory for search).  
    *   **Logs-based Metrics**: Create counter/metric from log entries (e.g., count HTTP 5xx errors).  
    *   **Sinks**: Export logs to GCS, BigQuery, Pub/Sub.  
        *   *Sink UI*: **Create sink** → *Sink name*, *Filter* (Mandatory), *Destination* (Mandatory).  

#### **Error Reporting**
*   **Core Concept**: **Aggregates and analyzes application errors** (from App Engine, GKE, Compute Engine, Cloud Functions). Groups similar stack traces.  
*   **UI Usage**: **Operations → Error Reporting**. Shows grouped errors, frequency, affected services. Requires client libraries to report errors.  

#### **Cloud Trace**
*   **Core Concept**: **Distributed tracing** for microservices. Visualizes request latency across services.  
*   **UI Usage**: **Operations → Trace → Trace list**. Shows latency breakdown per trace. Requires instrumentation (OpenTelemetry, Cloud Client Libraries).  

---

### **7. Security Command Center (SCC)**
#### **Threat Detection (Premium Tier)**
*   **Core Concept**: **Uses event-based and machine learning detection** to identify threats (e.g., crypto mining, SSH brute force, phishing). Requires **Event Threat Detection** module.  
*   **UI Usage**: **Security Command Center → Findings → Filter by "Event Threat Detection"**. Shows high-fidelity alerts with evidence.  

#### **Security Health Analytics (SHA)**
*   **Core Concept**: **Continuous security scanner** checking for **misconfigurations** against CIS benchmarks and best practices (e.g., "VM has IP forwarding enabled", "Bucket is publicly accessible"). Free tier available.  
*   **UI Usage**: **Security Command Center → Findings → Filter by "Security Health Analytics"**.  
    *   *Key Fields per Finding*:  
        *   *Category*: `Network`, `Storage`, `Compute` (Mandatory context).  
        *   *Severity*: `High`, `Medium`, `Low` (Mandatory priority).  
        *   *Resource*: Link to the misconfigured resource (Mandatory for remediation).  
        *   *Recommendation*: Steps to fix (Mandatory action guide).  
*   **Enabling SHA**: Automatically enabled in Premium tier. Free tier has limited checks.  

---

### **Critical Resources & Best Practices Summary**

1.  **GKE**: **ALWAYS use Workload Identity** instead of service account keys. Prefer **Autopilot** unless you need node-level control.
2.  **Networking**: **Shared VPC** is mandatory for enterprise multi-project setups. **Cloud Armor** is essential for public-facing services.
3.  **Security**: **Shielded VMs** for all production VMs. **Confidential VMs** for highly sensitive workloads. **SCC Premium** is worth the cost for SHA/Threat Detection.
4.  **Operations**: **Enable Monitoring/Logging** from Day 1. Build **alerts on SLOs**, not just infrastructure metrics.
5.  **Storage**: Use **Filestore** for shared file needs. **STS** is the *only* reliable way to migrate large datasets into GCS.
6.  **Anthos**: **ACM (GitOps)** is non-negotiable for cluster configuration at scale. **ASM** is critical for secure multi-cluster communication.

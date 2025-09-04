### **1. Compute Engine (IaaS)**
*The foundation for VM-based workloads. Offers granular control over hardware.*

#### **Key Concepts**
- **VMs (Virtual Machines)**: Isolated compute resources. *Stateless by default* (ephemeral OS disk). Use **Persistent Disks** for state.
- **Machine Types**:
  - **General Purpose**: `e2`, `n2`, `n2d`, `n1` (Balanced CPU/Memory; e.g., `e2-medium`).
  - **Compute Optimized**: `c2`, `c2d` (High CPU; e.g., `c2-standard-4`).
  - **Memory Optimized**: `m2`, `m3` (High RAM; e.g., `m3-ultramem-128`).
  - **Accelerator-Optimized**: `a2` (GPUs; e.g., `a2-highgpu-1g`).
  - *Custom Machines*: Mix vCPUs/RAM (e.g., 2 vCPUs + 13GB RAM).
- **Preemptible VMs**: Up to **91% cheaper**, but **terminated after 24h** or if GCP needs capacity. *Ideal for batch jobs, fault-tolerant workloads*. **No SLA**.
- **Zones**: Single data center (e.g., `us-central1-a`). **Isolated failure domains**.
- **Regions**: Contains multiple zones (e.g., `us-central1`). For **high availability**.

#### **Creating a VM via UI (Step-by-Step)**
1. **Navigation**: `Compute Engine` > `VM instances` > `CREATE INSTANCE`
2. **Basic Fields**:
   - **Name**: `my-vm` (Unique per zone, lowercase, hyphens only).
   - **Region/Zone**: `us-central1` / `us-central1-a` (Critical for latency/failover).
   - **Machine Configuration**:
     - *Series*: `E2` (Cost-effective) or `N2` (Balanced).
     - *Machine Type*: `e2-medium` (2 vCPU, 4GB RAM).
   - **Boot Disk**: Click `Change` > Select `Debian 11` > Size `50 GB` (SSD persistent disk).
   - **Firewall**: ✅ `Allow HTTP traffic` & `Allow HTTPS traffic` (Creates firewall rules).
3. **Advanced Fields**:
   - **Networking, disks, security, management** > **Networking**:
     - *Network tags*: `http-server,https-server` (Triggers firewall rules).
     - *Network interface*: `nic0` > *External IPv4* > `Ephemeral` (or reserve static IP).
   - **Management**:
     - *Preemptibility*: ✅ `On` (For preemptible VMs).
     - *Automatic restart*: ❌ (Disable for stateless batch jobs).
     - *On-host maintenance*: `Migrate VM instance` (Live migration) or `Stop VM instance`.
4. **Click `CREATE`**.

#### **Mandatory `gcloud` CLI**
```bash
# Create standard VM (non-preemptible)
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --tags=http-server,https-server \
  --scopes=cloud-platform \  # Full API access (use minimal scopes in prod!)
  --no-address  # No external IP (for internal-only VMs)

# Create preemptible VM
gcloud compute instances create my-preempt-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --preemptible \
  --maintenance-policy=TERMINATE
```

#### **Essential Terraform**
```hcl
resource "google_compute_instance" "vm" {
  name         = "my-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
      size  = 50
    }
  }

  network_interface {
    network = "default"
    access_config {} # Adds external IP
  }

  tags = ["http-server", "https-server"]

  # Preemptible config (uncomment for preemptible)
  # scheduling {
  #   preemptible = true
  #   on_host_maintenance = "TERMINATE"
  # }
}
```

---

### **2. Cloud Storage**
*Object storage for unstructured data (images, logs, backups).*

#### **Key Concepts**
- **Buckets**: Top-level containers (globally unique name, e.g., `my-project-backups-2023`). **Immutable location** after creation.
- **Storage Classes**:
  - **Standard**: Frequent access (e.g., active website assets).
  - **Nearline**: Infrequent access (min 30-day storage, e.g., backups).
  - **Coldline**: Rare access (min 90-day storage, e.g., disaster recovery).
  - **Archive**: Long-term retention (min 365-day storage, e.g., compliance data).
- **Lifecycle Management**: Automate transitions/deletions (e.g., "Move to Coldline after 30 days").
- **Signed URLs**: Time-limited URL granting temporary access to private objects (e.g., `https://storage.googleapis.com/bucket/file?Expires=...&Signature=...`).

#### **Creating a Bucket via UI (Step-by-Step)**
1. **Navigation**: `Cloud Storage` > `Buckets` > `CREATE`
2. **Basic Fields**:
   - **Name**: `my-unique-bucket-name-123` (Lowercase, hyphens, 3-63 chars).
   - **Location type**: `Region` (e.g., `us-central1`) for lower latency/cost.
   - **Default storage class**: `Standard` (Change per object later).
3. **Advanced Fields**:
   - **Access control**: `Uniform` (IAM only) or `Fine-grained` (ACLs + IAM). **Prefer Uniform**.
   - **Encryption**: `Google-managed key` (Default, free).
   - **Advanced settings** > **Lifecycle rules**:
     - *Add rule* > `Age` > `30` days > `Set storage class` > `Coldline`.
     - *Add rule* > `Age` > `365` days > `Delete`.

#### **Mandatory `gsutil` CLI**
```bash
# Create bucket (Standard class, US region)
gsutil mb -l US -c STANDARD gs://my-bucket-name

# Set lifecycle policy (via JSON file)
cat > lifecycle.json <<EOF
{
  "rule": [{
    "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
    "condition": {"age": 30}
  },{
    "action": {"type": "Delete"},
    "condition": {"age": 365}
  }]
}
EOF
gsutil lifecycle set lifecycle.json gs://my-bucket-name

# Generate signed URL (expires in 1 hour)
gsutil signurl -d 1h p12-key.pem gs://my-bucket/file.txt
```

#### **Essential Terraform**
```hcl
resource "google_storage_bucket" "bucket" {
  name          = "my-bucket-name"
  location      = "US"
  storage_class = "STANDARD"

  uniform_bucket_level_access = true # Enforce IAM-only access

  lifecycle_rule {
    action {
      type = "SetStorageClass"
      storage_class = "COLDLINE"
    }
    condition {
      age = 30
    }
  }

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      age = 365
    }
  }
}
```

---

### **3. Virtual Private Cloud (VPC)**
*Isolated network for your resources (replaces physical data center network).*

#### **Key Concepts**
- **Subnets**: IP ranges (IPv4) within a region (e.g., `10.0.0.0/24` in `us-central1`). **Regional** (not zonal).
- **Firewall Rules**: **Stateful** filters (allow/deny). **Ingress/Egress** rules. *Default VPC has open rules - lock this down!*
- **Routes**: Direct traffic (e.g., `0.0.0.0/0` -> Cloud Router for internet). **System routes** (auto-created) + **custom routes**.
- **Cloud NAT**: Allows VMs **without external IPs** to access the internet (outbound only). Uses **Cloud Router**.

#### **Creating a Custom VPC via UI (Step-by-Step)**
1. **Navigation**: `VPC Network` > `VPC networks` > `CREATE VPC NETWORK`
2. **Basic Fields**:
   - **Name**: `my-vpc` (Unique per project).
   - **Subnet creation mode**: `Custom` (Avoid `Automatic` - too broad).
3. **Subnets Configuration**:
   - *Add subnet*:
     - **Name**: `us-central-subnet`
     - **Region**: `us-central1`
     - **IP range**: `10.0.0.0/24` (Must not overlap with other subnets).
     - **Enable flow logs**: ❌ (Enable for security monitoring).
4. **Firewall Rules** (Post-creation):
   - `VPC Network` > `Firewall` > `CREATE FIREWALL RULE`
     - **Name**: `allow-http`
     - **Network**: `my-vpc`
     - **Targets**: `All instances in the network`
     - **Source IP ranges**: `0.0.0.0/0`
     - **Protocols and ports**: ✅ `Specified protocols and ports` > `tcp:80`
5. **Cloud NAT Setup**:
   - `VPC Network` > `NAT gateways` > `CREATE NAT GATEWAY`
     - **Name**: `my-nat`
     - **VPC network**: `my-vpc`
     - **Cloud Router**: `Create new router` (Name: `my-router`)
     - **Subnetwork**: `us-central-subnet`
     - **Min. ports per VM**: `64` (Higher = more concurrent connections)

#### **Mandatory `gcloud` CLI**
```bash
# Create VPC
gcloud compute networks create my-vpc --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create us-central-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24

# Create firewall rule (allow HTTP)
gcloud compute firewall-rules create allow-http \
  --network=my-vpc \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0

# Create Cloud NAT
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --nat-custom-subnet-ip-ranges=us-central-subnet \
  --auto-allocate-nat-external-ips
```

#### **Essential Terraform**
```hcl
resource "google_compute_network" "vpc" {
  name                    = "my-vpc"
  auto_create_subnetworks = false # Critical for custom subnets
}

resource "google_compute_subnetwork" "subnet" {
  name          = "us-central-subnet"
  ip_cidr_range = "10.0.0.0/24"
  region        = "us-central1"
  network       = google_compute_network.vpc.name
}

resource "google_compute_firewall" "http" {
  name    = "allow-http"
  network = google_compute_network.vpc.name
  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
  source_ranges = ["0.0.0.0/0"]
}

# Cloud NAT (requires Cloud Router)
resource "google_compute_router" "router" {
  name    = "my-router"
  network = google_compute_network.vpc.name
  region  = "us-central1"
}

resource "google_compute_router_nat" "nat" {
  name                               = "my-nat"
  router                             = google_compute_router.router.name
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}
```

---

### **4. Cloud DNS**
*Managed DNS service (public & private zones).*

#### **Key Concepts**
- **Public Zones**: DNS for public domains (e.g., `example.com`). Requires domain registration.
- **Private Zones**: DNS only resolvable within your VPC (e.g., `internal.example.com`). **No public exposure**.
- **Record Sets**: DNS records (A, AAAA, CNAME, MX, etc.). TTL defines cache time.
- **DNSSEC**: Digital signatures to prevent DNS spoofing. **Requires key management**.

#### **Creating a Public Zone via UI (Step-by-Step)**
1. **Navigation**: `Network Services` > `Cloud DNS` > `CREATE ZONE`
2. **Basic Fields**:
   - **Zone type**: `Public`
   - **Name**: `example-com` (Internal reference name)
   - **DNS name**: `example.com.` (Trailing dot required!)
   - **Description**: Optional
3. **DNSSEC** (Optional but recommended):
   - *After zone creation* > `DNSSEC` tab > `Enable DNSSEC`
     - **State**: `on`
     - **Algorithm**: `ECDSAP256SHA256` (Modern standard)
     - **Key signing key (KSK) rotation period**: `365` days
     - **Zone signing key (ZSK) rotation period**: `90` days

#### **Creating a Private Zone via UI**
1. **Navigation**: `Cloud DNS` > `CREATE ZONE`
2. **Basic Fields**:
   - **Zone type**: `Private`
   - **Name**: `internal-zone`
   - **DNS name**: `internal.example.com.`
   - **DNS visibility**: `Only allow queries from the selected networks`
   - **Networks**: Add your VPC (`my-vpc`)

#### **Mandatory `gcloud` CLI**
```bash
# Create public zone
gcloud dns managed-zones create example-com \
  --dns-name="example.com." \
  --zone-type=public

# Create private zone
gcloud dns managed-zones create internal-zone \
  --dns-name="internal.example.com." \
  --zone-type=private \
  --visibility=private \
  --networks=my-vpc

# Add A record (public zone)
gcloud dns record-sets transaction start --zone=example-com
gcloud dns record-sets transaction add "1.2.3.4" --name="www.example.com." --ttl=300 --type=A --zone=example-com
gcloud dns record-sets transaction execute --zone=example-com

# Enable DNSSEC (public zone)
gcloud dns managed-zones update example-com --dnssec-state=on
```

#### **Essential Terraform**
```hcl
# Public Zone
resource "google_dns_managed_zone" "public" {
  name        = "example-com"
  dns_name    = "example.com."
  description = "Public zone for example.com"
}

resource "google_dns_record_set" "www" {
  name         = "www.example.com."
  type         = "A"
  ttl          = 300
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = ["1.2.3.4"]
}

# Private Zone
resource "google_dns_managed_zone" "private" {
  name     = "internal-zone"
  dns_name = "internal.example.com."
  visibility = "private"
  private_visibility_config {
    networks {
      network_url = google_compute_network.vpc.self_link
    }
  }
}
```

---

### **5. Cloud CDN**
*Global content delivery network (caches content at edge locations).*

#### **Key Concepts**
- **Caching**: Stores static content (images, CSS, JS) at **90+ edge points**. Reduces origin load & latency.
- **Cache Keys**: What defines a unique cached object (e.g., `Host`, `Accept-Language`). **Customize to avoid cache misses**.
- **Integration with Load Balancing**: **Only works with HTTP(S) Load Balancer** (not TCP/UDP). Enabled per backend service.

#### **Enabling CDN via UI (Step-by-Step)**
1. **Prerequisite**: Create an **HTTP(S) Load Balancer** (see section 6).
2. **Navigation**: `Network Services` > `Load balancing` > Edit your HTTP(S) LB
3. **Backend Configuration**:
   - Under **Backend services / Backend buckets** > Edit your backend
   - **Enable Cloud CDN**: ✅ `Enable CDN`
4. **Cache Key Settings** (Critical for performance):
   - *Cache key policy* > **Include host**: ✅ (Required if serving multiple domains)
   - **Include protocol**: ✅ (If HTTP/HTTPS both used)
   - **Include query string**: ❌ (Unless query params matter, e.g., `?v=2`)
   - **Query string blacklist**: `utm_source,utm_medium` (Exclude tracking params)
5. **Cache TTLs**:
   - *Default TTL*: `3600` (1 hour - adjust per content type)
   - *Maximum TTL*: `86400` (24 hours - prevents stale content)

#### **Mandatory `gcloud` CLI**
```bash
# Enable CDN on existing backend service
gcloud compute backend-services update my-backend \
  --enable-cdn \
  --cache-key-include-host \
  --cache-key-query-string-blacklist=utm_source,utm_medium \
  --default-ttl=3600 \
  --max-age=86400
```

#### **Essential Terraform**
```hcl
resource "google_compute_backend_service" "backend" {
  name                  = "my-backend"
  enable_cdn            = true
  cdn_policy {
    cache_mode = "CACHE_ALL_STATIC"
    client_ttl = 3600
    max_ttl    = 86400
    negative_caching = true
    cache_key_policy {
      include_host         = true
      include_protocol     = true
      include_query_string = false
      query_string_blacklist = ["utm_source", "utm_medium"]
    }
  }
  # ... (backend instances/buckets, health checks)
}
```

---

### **6. Cloud Load Balancing**
*Global, distributed load balancers (L3-L7). All are **anycast** (single IP worldwide).*

#### **Key Types**
- **HTTP(S) Load Balancer**: **L7** (Global). Routes based on URL, host, etc. **Requires SSL cert** for HTTPS. Integrates with **Cloud CDN** & **Cloud Armor** (WAF).
- **TCP/SSL Load Balancer**: **L4** (Global). For non-HTTP(S) traffic (e.g., MongoDB, custom TCP). SSL offload at proxy.
- **Internal Load Balancer**: **L3/L4** (Regional). For private services within VPC (e.g., microservices).
- **Network Load Balancer**: **L4** (Regional). Passthrough (no proxy). For extreme performance (e.g., gaming).

#### **Creating HTTP(S) LB via UI (Step-by-Step)**
1. **Navigation**: `Network Services` > `Load balancing` > `CREATE LOAD BALANCER`
2. **Select Type**: `HTTP(S) Load Balancing` > `Start configuration`
3. **Frontend Configuration**:
   - **Name**: `web-lb`
   - **IP version**: `IPv4`
   - **IP address**: `Create IP address` (Name: `web-lb-ip`)
   - **Port**: `80` (HTTP) or `443` (HTTPS)
   - **HTTPS**: Requires **SSL certificate** (Create new or upload)
4. **Backend Configuration**:
   - **Backend services & backend buckets** > `Create a backend service`
     - **Name**: `web-backend`
     - **Protocol**: `HTTP`
     - **Named port**: `http` (Port 80)
     - **Backend instances**: Add instance groups (e.g., `my-instance-group`)
     - **Health check**: Create new (Protocol: `HTTP`, Port: `80`, Path: `/health`)
     - ✅ `Enable Cloud CDN` (If static content)
5. **URL Map** (Routing Rules):
   - **Host and path rules**: Default (`/*` -> `web-backend`)
   - *Advanced*: Add host-based rules (e.g., `api.example.com/*` -> `api-backend`)

#### **Mandatory `gcloud` CLI (HTTP(S) LB)**
```bash
# Create health check
gcloud compute health-checks create http web-hc --port=80 --request-path=/health

# Create backend service
gcloud compute backend-services create web-backend \
  --protocol=HTTP \
  --health-checks=web-hc \
  --global

# Add instance group to backend
gcloud compute backend-services add-backend web-backend \
  --instance-group=my-instance-group \
  --instance-group-zone=us-central1-a \
  --global

# Create URL map
gcloud compute url-maps create web-map \
  --default-service=web-backend

# Create target HTTP proxy
gcloud compute target-http-proxies create http-lb-proxy \
  --url-map=web-map

# Reserve static IP
gcloud compute addresses create web-lb-ip --global

# Create forwarding rule
gcloud compute forwarding-rules create http-content-rule \
  --address=web-lb-ip \
  --global \
  --target-http-proxy=http-lb-proxy \
  --ports=80
```

#### **Essential Terraform (HTTP(S) LB)**
```hcl
resource "google_compute_health_check" "web" {
  name = "web-hc"
  http_health_check {
    port = 80
    request_path = "/health"
  }
}

resource "google_compute_backend_service" "web" {
  name          = "web-backend"
  protocol      = "HTTP"
  health_checks = [google_compute_health_check.web.self_link]
  enable_cdn    = true

  backend {
    group = google_compute_instance_group_manager.instance_group.instance_group
  }
}

resource "google_compute_url_map" "web" {
  name            = "web-map"
  default_service = google_compute_backend_service.web.self_link
}

resource "google_compute_target_http_proxy" "web" {
  name    = "http-lb-proxy"
  url_map = google_compute_url_map.web.self_link
}

resource "google_compute_global_address" "web_ip" {
  name = "web-lb-ip"
}

resource "google_compute_global_forwarding_rule" "web" {
  name       = "http-content-rule"
  target     = google_compute_target_http_proxy.web.self_link
  port_range = "80"
  ip_address = google_compute_global_address.web_ip.address
}
```

---

### **Critical Cross-Service Insights**
1. **VPC is the Foundation**: All services (except Cloud Storage buckets) live *inside* a VPC. **Always create a custom VPC first**.
2. **Firewall Rules are Stateful**: Allow ingress? Egress is auto-allowed. **Default VPC rules are dangerous** - delete them!
3. **Zones vs Regions**: 
   - *Zonal*: VMs, disks, internal IPs. **Single point of failure**.
   - *Regional*: Subnets, Cloud SQL, Memorystore. **Survives zone failure**.
   - *Global*: HTTP(S) LB, Cloud Storage, Cloud DNS. **Worldwide**.
4. **Security Order**:
   - **Least privilege IAM** > **VPC Firewall Rules** > **Service Perimeters** > **Shielded VMs**.
5. **Cost Traps**:
   - Egress from us-central1 to internet: **$0.12/GB** (vs $0.01/GB within region).
   - Cloud Storage egress to internet: **$0.12/GB** (use Cloud CDN to reduce!).
   - Preemptible VMs save 90% but **terminate without warning**.

> **Pro Tip**: Always enable **Cloud Logging & Monitoring** for all services. Use **Terraform** for everything (even UI-created resources) via `terraform import`.

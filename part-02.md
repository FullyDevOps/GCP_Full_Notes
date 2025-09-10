### **1. Compute Engine (IaaS)**
*   **Core Purpose:** Run virtual machines (VMs) on Google's global infrastructure. *The foundation for most custom workloads.*
*   **Why DevOps Cares:** Your primary compute layer for stateful apps, legacy systems, custom tooling, batch processing, and where you have full OS control.

#### **Key Concepts Explained (Beginner -> Expert)**

*   **VMs (Virtual Machines):**
    *   **Beginner:** Software emulation of a physical computer (CPU, RAM, Disk, OS). Launched from machine images (pre-configured OS + apps).
    *   **Intermediate:** **Persistent Disks (PD):** Network-attached block storage (SSD or HDD). *Crucial:* Disks are independent of VM lifecycle (stop/delete VM = disk persists). Types: `pd-standard` (HDD), `pd-ssd`, `pd-balanced` (cost/perf sweet spot), `pd-extreme` (high IOPS). **Sizing Matters:** Oversized disks = wasted $; undersized = performance bottlenecks.
    *   **Expert:** **Disk Performance Tuning:** IOPS/Throughput scale *with disk size* (e.g., `pd-ssd` max IOPS = 30k @ 6TB). **Encryption:** All data encrypted at rest by default (Google-managed keys), but use **Customer-Managed Encryption Keys (CMEK)** for compliance (e.g., `gcloud compute disks create --kms-key`). **Local SSDs:** NVMe ephemeral storage *only* for transient, high-speed temp data (lost on stop/terminate). **Shielded VMs:** *Mandatory for security:* Enables Secure Boot, vTPM (for integrity), Integrity Monitoring (detects boot changes). **OS Login:** *Always enable* for secure SSH via IAM, avoids managing OS users.
    *   **DevOps Action:** Automate disk sizing based on app metrics. Enforce CMEK & Shielded VMs via Organization Policies. Use OS Login exclusively. Monitor disk latency (Cloud Monitoring metrics `agent.googleapis.com/disk/read_bytes_count`).

*   **Machine Types:**
    *   **Beginner:** Predefined CPU/RAM combos (e.g., `e2-medium` = 2vCPU, 4GB RAM). Families: `e2` (balanced), `n2`/`n2d` (next-gen), `n1` (legacy), `m1`/`m2` (memory-optimized), `c2`/`c2d` (compute-optimized).
    *   **Intermediate:** **Custom Machine Types:** Mix vCPU & RAM precisely (e.g., `4 vCPU, 15GB RAM`). *Saves significant cost* vs. predefined if your app needs non-standard ratios. **Shared-Core (`f1-micro`, `g1-small`):** Only for dev/test (bursting CPU, low perf).
    *   **Expert:** **Sustained Use Discounts (SUD):** Automatic ~30% discount for VMs running >25% of a *calendar month*. **Committed Use Discounts (CUD):** Up to 57% discount by committing to specific vCPU/RAM for 1-3 years (use for stable prod workloads). **Per-Second Billing:** Billing stops *within 1 minute* of stopping a VM (except SUD/CUD commitments). **Choosing Families:** `n2d` (AMD EPYC) often cheaper than `n2` (Intel) for same perf; `c3` (newest) for highest compute perf. **vCPU Count Impact:** Affects network egress quota (higher vCPU = higher quota).
    *   **DevOps Action:** **Always** use custom machine types for production unless predefined fits *perfectly*. Model SUD/CUD savings (use [CUD Manager](https://cloud.google.com/compute/docs/instances/save-with-cuds)). Monitor vCPU utilization (`agent.googleapis.com/cpu/utilization`) to rightsize. Prefer `n2d`/`c3` over older families.

*   **Preemptible VMs (Now "Spot VMs"):**
    *   **Beginner:** Short-lived VMs (max 24h) that can be *preempted* (terminated) by GCP with 30s notice if capacity needed elsewhere. Up to 91% discount.
    *   **Intermediate:** **Use Cases:** Fault-tolerant, stateless, batch processing (CI/CD runners, rendering, data analysis). **Not For:** Stateful apps, databases, long-running critical services.
    *   **Expert:** **Quotas:** Have *separate quotas* from regular VMs (often low by default - request increase!). **Mitigating Preemption:** Use Managed Instance Groups (MIGs) with auto-healing & auto-scaling to replace preempted VMs instantly. **Statelessness is Key:** Store all state in PD (detached on preemption) or Cloud Storage. **Spot VMs vs Preemptible:** Same concept, newer name/branding. **Pricing:** Spot price fluctuates *per zone* - monitor `compute.googleapis.com/instance/spot_preempted_count`.
    *   **DevOps Action:** **Essential for CI/CD runners & batch jobs.** Always deploy Spot VMs within MIGs. Implement graceful shutdown hooks (e.g., trap SIGTERM) to save state before 30s notice. Monitor preemption rates (`spot_preempted_count`) – high rates indicate zone instability. *Never* use without MIGs/statelessness.

*   **Zones & Regions:**
    *   **Beginner:** **Region:** Geographic area (e.g., `us-central1`, `europe-west4`). **Zone:** Isolated location *within* a region (e.g., `us-central1-a`, `us-central1-b`). Zones have independent power/network.
    *   **Intermediate:** **High Availability (HA):** Deploy critical services across *at least 2 zones* in a region. **Latency:** Traffic *within* a region is low-latency (<1ms); *between* regions is higher (10s-100s ms).
    *   **Expert:** **Region Selection Strategy:** Proximity to users (latency), Compliance (data residency), Service Availability (check [Product Availability](https://cloud.google.com/about/locations)), Cost (varies by region!). **Multi-Region vs Dual-Zone:** `Dual-region` (e.g., `nam6` = `us-central1` + `us-east1`) for *stronger* geo-redundancy (higher cost/latency). **Zonal Resource Dependencies:** Some resources (like PDs, VMs) are zonal. **Global Resources:** Some (like HTTP(S) LB, Cloud DNS) are global.
    *   **DevOps Action:** **Design for Zone Failure:** Assume a zone *will* go down. Use regional MIGs, regional PDs (for databases like Cloud SQL), and global LBs. Avoid "zonal lock-in" – use regional resources where possible. Know your region's zone count (e.g., `us-central1` has 4 zones - good for HA).

---

### **2. Cloud Storage**
*   **Core Purpose:** Scalable, durable, highly available object storage. *The universal data lake for GCP.*
*   **Why DevOps Cares:** Artifact storage (Docker images, binaries), VM boot disks, log archival, data lakes, static website hosting, backup target.

#### **Key Concepts Explained (Beginner -> Expert)**

*   **Buckets:**
    *   **Beginner:** Top-level container for objects (files). Unique name globally (`my-company-backups-2023`). Objects stored as `bucket-name/object-path`.
    *   **Intermediate:** **Location:** Choose region/multi-region at creation (immutable!). **Storage Class:** Set *per bucket* (can override per object). **Uniform Bucket-Level Access (UBLA):** *Strongly Recommended:* Replaces ACLs with IAM only (simpler, more secure). **Bucket Labels:** For cost tracking, automation.
    *   **Expert:** **Bucket Lock:** WORM (Write-Once-Read-Many) compliance (e.g., `Retention Policy`, `Legal Hold`). **Requester Pays:** Downstream users pay egress costs (prevents bill shock from public buckets). **Bucket Encryption:** Default uses Google-managed keys; enforce **CMEK** for sensitive data (`gcloud storage buckets update gs://my-bucket --default-kms-key=my-key`). **Bucket IAM Conditions:** Fine-grained access (e.g., `resource.name.startsWith("projects/_/buckets/my-bucket/finance/")`).
    *   **DevOps Action:** **Always enable UBLA.** Enforce CMEK via Organization Policy. Use labels religiously (`env=prod`, `app=jenkins`). Implement Bucket Lock for audit logs. Avoid public buckets; if needed, use Signed URLs instead.

*   **Storage Classes:**
    *   **Beginner:** `STANDARD` (frequent access), `NEARLINE` (infrequent, 30d min), `COLDLINE` (archival, 90d min), `ARCHIVE` (deep archive, 365d min, cheapest).
    *   **Intermediate:** **Cost Tradeoffs:** Lower storage cost = higher retrieval cost + minimum storage duration + potential retrieval latency. **Choosing:** `STANDARD` for active data, `NEARLINE` for backups/logs (accessed <1/mo), `ARCHIVE` for compliance archives.
    *   **Expert:** **Early Deletion Fees:** Deleting before min duration incurs fee = min duration cost (e.g., delete `NEARLINE` object after 15d = pay for 30d). **Class Changes:** Use Lifecycle Rules to auto-transition objects. **Regional vs Multi-Regional:** `STANDARD`/`NEARLINE` can be regional (cheaper) or multi-regional (higher availability/cost). **Performance:** `STANDARD` has highest throughput/lowest latency. `ARCHIVE` retrieval can take hours.
    *   **DevOps Action:** **Model costs rigorously:** Factor in storage + retrieval + early deletion. Use Lifecycle Rules aggressively (e.g., move logs to `NEARLINE` after 7d, `ARCHIVE` after 90d). Prefer *regional* `STANDARD`/`NEARLINE` unless global access needed.

*   **Lifecycle Management:**
    *   **Beginner:** Rules to automate actions (delete, change storage class) based on object age.
    *   **Intermediate:** **Conditions:** Age (days since creation), matchesPrefix/suffix, isLive (non-versioned), daysSinceNoncurrentTime (for versioned buckets).
    *   **Expert:** **Versioned Buckets:** Critical for ransomware protection! Lifecycle rules manage *noncurrent* versions (e.g., `Delete noncurrent versions after 30 days`). **Chaining Rules:** Object transitions `STANDARD` -> `NEARLINE` -> `ARCHIVE` over time. **Cost Optimization:** Target biggest cost drivers first (e.g., deleting old CI artifacts saves more than transitioning logs).
    *   **DevOps Action:** **Enable Object Versioning on critical buckets (backups, configs).** Create lifecycle rules *immediately* upon bucket creation. Monitor rule effectiveness (`storage.googleapis.com/lifecycle/action_count`).

*   **Signed URLs:**
    *   **Beginner:** Time-limited URL granting temporary access to a *private* object without IAM.
    *   **Intermediate:** Generated using a service account key. Expiry time (max 7 days). Permissions (read/write/delete) defined when generated.
    *   **Expert:** **Security:** *Never* use user keys; use dedicated service account with *minimal* permissions (`roles/storage.objectViewer`). **Key Rotation:** Rotate service account keys frequently. **Alternatives:** Use **V4 Signed URLs** (more secure HMAC-SHA256) over legacy V2. **Use Cases:** Secure file downloads/uploads from web apps, bypassing CDN cache (see Cloud CDN), temporary access for partners.
    *   **DevOps Action:** **Automate generation** (e.g., Cloud Function on upload event). **Set shortest possible expiry** (minutes/hours, not days). Audit usage logs (`storage.googleapis.com/activity`). Prefer Signed URLs over public buckets *always*.

---

### **3. Virtual Private Cloud (VPC)**
*   **Core Purpose:** Isolated, private network environment for your GCP resources. *The security and connectivity backbone.*
*   **Why DevOps Cares:** Network segmentation, security boundaries, hybrid connectivity, service communication. **Fundamental to zero-trust security.**

#### **Key Concepts Explained (Beginner -> Expert)**

*   **Subnets:**
    *   **Beginner:** IP address ranges (`10.0.0.0/24`) within a VPC network, *mapped to a specific region*. Resources (VMs) attach to subnets.
    *   **Intermediate:** **Regional Scope:** Subnets span *all zones* in a region. **IP Allocation:** Use `/28` to `/20` ranges. **Private Google Access:** Allows VMs *without* public IPs to reach Google APIs/services (via special DNS `*.googleapis.com`).
    *   **Expert:** **Custom Mode vs Auto Mode:** **ALWAYS use Custom Mode VPCs.** Auto Mode creates default subnets (bad for security/control). **Shared VPC:** Host project (network) + Service projects (compute). *Essential for enterprise*. **VPC Peering:** Connect 2 VPCs (private IP traffic), *non-transitive*. **VPC Network Controls (Firewall Rules):** Rules are *stateful* (reply traffic allowed automatically).
    *   **DevOps Action:** **Design VPCs in Custom Mode.** Plan IP ranges meticulously (use `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` avoiding overlaps). Enable Private Google Access on *all* subnets. Implement Shared VPC for multi-project orgs.

*   **Firewall Rules:**
    *   **Beginner:** Control ingress/egress traffic to/from VMs (based on IP, port, protocol, tags/service accounts).
    *   **Intermediate:** **Stateful:** Allow ingress = auto-allow egress reply. **Direction:** `Ingress` (to VM), `Egress` (from VM). **Targets:** Apply to VMs with specific *Network Tags* or *Service Accounts*. **Priority:** Lower number = higher priority (evaluated top-down).
    *   **Expert:** **Implicit Rules:** Default `deny all` ingress, `allow all` egress. **Principle of Least Privilege (PoLP):** Default-deny ingress! **Service Account Targeting:** *Superior to Tags:* Immutable, tied to identity, not config. **Logging:** Enable **Firewall Rule Logging** (ingress/egress denies) for security monitoring. **Adaptive Protection:** ML-based DDoS mitigation (part of Cloud Armor).
    *   **DevOps Action:** **Implement strict ingress deny-all policy.** Allow *only* necessary ports (e.g., `tcp:22` for bastion, `tcp:80,443` for web). **Use Service Accounts as targets, NOT tags.** Enable deny logging *immediately*. Audit rules monthly (`gcloud compute firewall-rules list`).

*   **Routes:**
    *   **Beginner:** Direct traffic within VPC or to destinations outside (on-prem, internet).
    *   **Intermediate:** **Types:** `default` (VPC internal), `custom` (user-defined), `dynamic` (from Cloud Routers). **Next Hop:** VM, VPN tunnel, Interconnect, Internet Gateway, VPC Peer.
    *   **Expert:** **Route Priority:** Most specific route wins. **Custom Route Advertisements:** Control what on-prem learns via Cloud Router (BGP). **Transitive Routes:** VPC Peering is *non-transitive*; use **Hub & Spoke** with Cloud Routers for transitivity. **Private Google Access Routes:** Managed automatically (`199.36.153.4/30`, `199.36.153.8/30`).
    *   **DevOps Action:** **Minimize custom routes.** Prefer VPC Peering over complex route tables. Understand Cloud Router for hybrid connectivity. Monitor route conflicts (`gcloud compute routes list`).

*   **Cloud NAT:**
    *   **Beginner:** Allows VMs *without* public IPs to access the internet (outbound only).
    *   **Intermediate:** **Managed Service:** No VMs to maintain. **Subnetworks:** Specify which subnets use NAT. **Min/Max Ports:** Controls port allocation per VM (affects scale).
    *   **Expert:** **High Availability:** Configure *at least 2* NAT gateways (one per zone) for HA. **SNAT IPs:** Use dedicated IPs (not ephemeral) for whitelisting. **Port Preservation:** `DISABLED` (default, scales better) vs `ENABLED` (needed for some legacy apps). **Logging:** Enable NAT logging for troubleshooting/security.
    *   **DevOps Action:** **Mandatory for all private VMs needing internet (updates, APIs).** Deploy with 2+ gateways. Use dedicated IPs for external service whitelisting. Monitor port allocation (`compute.googleapis.com/nat/ports_allocated`).

---

### **4. Cloud DNS**
*   **Core Purpose:** Highly available, scalable, managed DNS service. *The global address book for your services.*
*   **Why DevOps Cares:** Service discovery, global load balancing, domain management, internal name resolution (critical for microservices).

#### **Key Concepts Explained (Beginner -> Expert)**

*   **Public vs Private Zones:**
    *   **Beginner:** **Public Zone:** Resolves names publicly on the internet (e.g., `myapp.com`). **Private Zone:** Resolves names *only* within specific VPC networks (e.g., `internal.myapp.com` for internal services).
    *   **Intermediate:** **Private Zone Scope:** Can be visible to *all* VPCs in a project, *all* VPCs in a Shared VPC host project, or specific VPCs. **Peering:** Private zones can be shared via VPC Network Peering.
    *   **Expert:** **Split-Horizon DNS:** Use *same domain name* with different records in Public & Private zones (e.g., `api.myapp.com` points to LB externally, to internal ILB internally). **DNS Forwarding:** Route queries for specific domains to on-prem DNS (via Cloud DNS Forwarding Config). **Private Zone Security:** Restrict access via VPC firewall rules (allow DNS port 53).
    *   **DevOps Action:** **Use Private Zones for *all* internal service communication** (replaces `/etc/hosts`, complex DNS servers). Implement Split-Horizon for seamless internal/external access. Secure Private Zone access with firewall rules.

*   **Record Sets:**
    *   **Beginner:** DNS records (A, AAAA, CNAME, MX, TXT, etc.) within a zone. Point names to IPs/services.
    *   **Intermediate:** **Managed Zones:** Created per domain (`myapp.com.`). **TTL:** Time-to-Live (cache duration). **Health Checking:** Integrate with Cloud Load Balancing for failover (see LB section).
    *   **Expert:** **Geo-Targeting (Latency-Based Routing):** Use **Routing Policies** (Primary-Backup, Geo, Load-Based) with Cloud DNS + Global LB for optimal user experience. **CNAME Flattening:** Resolve CNAME chains at the edge (faster). **SOA Record:** Tune `refresh`, `retry`, `expire` times carefully (affects propagation).
    *   **DevOps Action:** **Always use Managed Instance Group (MIG) health checks with DNS routing policies for failover.** Set appropriate TTLs (low for critical changes, high for stability). Avoid deep CNAME chains; use A records pointing to LB VIPs.

*   **DNSSEC:**
    *   **Beginner:** Security extension to prevent DNS spoofing/cache poisoning (signs DNS responses).
    *   **Intermediate:** **State:** `on`, `transfer`, `off`. Requires registrar support (must publish DS record).
    *   **Expert:** **Key Management:** Zone Signing Key (ZSK), Key Signing Key (KSK). **Rolling Keys:** Requires careful process (double-signing period). **Cloud DNS Managed Keys:** GCP handles key rotation securely. **Validator:** Enable DNSSEC validation on resolvers (e.g., Google Public DNS `8.8.8.8` does).
    *   **DevOps Action:** **Enable DNSSEC (`state=on`) for all Public Zones.** Coordinate DS record update with your domain registrar *during initial setup*. Monitor DNSSEC status (`gcloud dns managed-zones describe my-zone --format="value(dnssecConfig.state)"`).

---

### **5. Cloud CDN**
*   **Core Purpose:** Global edge caching layer *integrated with HTTP(S) Load Balancing*. *Speed up content delivery, reduce origin load/cost.*
*   **Why DevOps Cares:** Web performance optimization, cost reduction (less egress from origin), DDoS mitigation (at edge).

#### **Key Concepts Explained (Beginner -> Expert)**

*   **Caching:**
    *   **Beginner:** Stores static content (images, CSS, JS) at Google's edge locations closer to users.
    *   **Intermediate:** **Cache Keys:** What defines a unique cached object (default: full URL). **Cache Modes:** `CACHE_ALL_STATIC` (auto-caches static mime types), `USE_ORIGIN_HEADERS` (respects `Cache-Control`), `FORCE_CACHE_ALL` (caches everything - dangerous!).
    *   **Expert:** **Cache Invalidation:** Purge specific paths (`gcloud cdn url-maps invalidate-cdn-cache`) or *entire cache* (use sparingly - quota limited). **TTL Control:** Primarily via origin `Cache-Control` headers (`max-age`, `s-maxage`). **Negative Caching:** Cache 404s (configure via LB backend service). **Cache Hit Ratio:** Critical metric (`cdn.googleapis.com/http/request_count` - `cache_fill_bytes_count`).
    *   **DevOps Action:** **Set precise `Cache-Control` headers at origin** (e.g., `public, max-age=31536000, immutable` for hashed assets). Avoid `FORCE_CACHE_ALL`. Monitor hit ratio relentlessly. Use invalidation *only* for critical updates (prefer versioned URLs).

*   **Cache Keys:**
    *   **Beginner:** Components used to generate the cache key (e.g., URL path, query string).
    *   **Intermediate:** **Customization:** Include/exclude query string parameters (`?v=123`), HTTP headers (`Accept-Language`), cookies (rarely needed).
    *   **Expert:** **Optimizing Keys:** Exclude non-relevant query params (e.g., `?utm_source=...`) to increase cache hits. **Include Critical Params:** For multi-tenant apps, include tenant ID in key. **Header Sensitivity:** Only include headers that *actually* change content (e.g., `Accept-Encoding` for gzip/br). **Cookie Handling:** Generally avoid caching if cookies are present (set `includeHost=true` only if needed).
    *   **DevOps Action:** **Analyze traffic patterns** to determine optimal cache key configuration. *Exclude* analytics/query params. *Include* only essential params affecting content. Test cache behavior with `Cache-Control` overrides.

*   **Integration with Load Balancing:**
    *   **Beginner:** CDN is *not* standalone; it's a feature **enabled on an HTTP(S) Load Balancer backend service**.
    *   **Intermediate:** **Enabling:** Set `enable-cdn=true` when creating/updating backend service. **Origin:** The backend service (Instance Group, Serverless NEG, Bucket) is the CDN origin.
    *   **Expert:** **Signed URLs/ Cookies:** Bypass cache for personalized content (see Cloud Storage Signed URLs). **Cache Tiering:** CDN edge -> CDN POPs -> Origin. **Bypassing CDN:** Use specific headers (`Pragma: no-cache`, `Cache-Control: no-store`) or paths (configure in LB). **WAF Integration:** Cloud Armor security policies apply *before* CDN cache (protects origin).
    *   **DevOps Action:** **CDN is always paired with Global HTTP(S) LB.** Configure CDN settings *within* the LB backend service. Use Signed URLs for cache-busting. Ensure Cloud Armor WAF is applied at the LB layer.

---

### **6. Cloud Load Balancing**
*   **Core Purpose:** Distribute traffic across healthy backend instances. *The traffic director for availability, scalability, and performance.*
*   **Why DevOps Cares:** High availability, scalability, global access, SSL termination, security (WAF), traffic management.

#### **Key Concepts Explained (Beginner -> Expert)**

*   **HTTP(S) Load Balancing (Global):**
    *   **Beginner:** L7 (application layer) LB for web traffic (HTTP/HTTPS). Global anycast IP. Integrates with CDN, Cloud Armor (WAF).
    *   **Intermediate:** **Components:** Frontend (IP/Port, SSL Cert), URL Map (routes `/api/*` -> backend), Backend Service (MIGs, NEGs), Health Checks. **Global Static IP:** Reserved IP used as single entry point worldwide.
    *   **Expert:** **Advanced Routing:** Host-based (`api.myapp.com` vs `www.myapp.com`), path-based (`/api/*`), header-based routing. **Session Affinity:** `CLIENT_IP`, `GENERATED_COOKIE` (for stateful apps - use sparingly). **Backend Services:** Can point to MIGs (VMs), Serverless NEGs (Cloud Run/Fn), or Bucket backends. **Global vs Regional:** HTTP(S) LB is *global*; other LBs are regional.
    *   **DevOps Action:** **Mandatory for all public web apps.** Implement strict health checks (deep path, e.g., `/healthz`). Use Cloud Armor WAF rules *always*. Configure precise URL maps. Prefer Serverless NEGs for modern apps.

*   **TCP/UDP Load Balancing (Regional):**
    *   **Beginner:** L4 (transport layer) LB for non-HTTP(S) traffic (e.g., MQTT, custom protocols, databases). Regional (single region).
    *   **Intermediate:** **Types:** **Network LB:** Passthrough (no proxy, preserves client IP), for TCP/UDP. **Internal TCP/UDP LB:** For traffic *within* VPC (regional).
    *   **Expert:** **Network LB:** Target Pools (legacy) or NEGs. Health checks are *critical* (no L7 checks). **Internal TCP/UDP LB:** Requires `proxy protocol` enabled if backend needs client IP. **Session Affinity:** `CLIENT_IP`, `CLIENT_IP_PROTO`, `NONE`. **Backend Limits:** Network LB max 1 backend per zone; Internal LB max 5 backends per zone.
    *   **DevOps Action:** **Use Network LB for high-performance, low-latency TCP/UDP (e.g., gaming, IoT).** Use Internal TCP/UDP LB for internal services (databases, message queues). Configure aggressive health checks. Avoid if HTTP(S) LB fits.

*   **Internal Load Balancing (HTTP(S) & TCP/UDP):**
    *   **Beginner:** LB accessible *only* from within your VPC network (or connected networks via peering/VPC SC). Uses private IP.
    *   **Intermediate:** **Types:** Internal HTTP(S) LB (L7), Internal TCP/UDP LB (L4). **Use Cases:** Expose services internally (e.g., microservices, databases), isolate tiers (web -> app -> db).
    *   **Expert:** **Internal HTTP(S) LB:** Requires specific subnet (`purpose=INTERNAL_HTTPS_LOAD_BALANCER`). Uses URL maps like global LB. **Internal TCP/UDP LB:** Uses target pools or NEGs. **Firewall Rules:** Must allow traffic from LB's health check ranges (`35.191.0.0/16`, `130.211.0.0/22`) and proxy ranges (`10.129.0.0/26` for HTTP(S) LB).
    *   **DevOps Action:** **Use Internal LBs for *all* multi-tier internal architectures.** Enforce zero-trust: Web tier talks *only* to App tier ILB, not directly to DBs. Configure strict firewall rules between tiers.

*   **Network Load Balancing (Regional Passthrough):**
    *   **Beginner:** L4 (TCP/UDP) passthrough LB. *No Google proxy*, preserves client source IP. Regional.
    *   **Intermediate:** **Best For:** Extreme performance needs, custom protocols, IP-based routing, when client IP preservation is mandatory.
    *   **Expert:** **Health Checks:** Less granular than proxy LBs (only TCP/HTTP checks). **Scaling:** Scales to 10s of Gbps per single anycast IP. **Backend Limits:** Max 1 healthy backend per zone (can cause imbalance). **No Global Anycast:** Regional IP only.
    *   **DevOps Action:** **Use only when client IP preservation or extreme perf is critical** (e.g., financial trading, specific legacy apps). Prefer Internal TCP/UDP LB or Global HTTP(S) LB otherwise. Monitor backend health aggressively.

---

### **Critical DevOps Integration & Best Practices (The Expert Edge)**

1.  **Infrastructure as Code (IaC):** **MANDATORY.** Use **Terraform** (industry standard) or **Deployment Manager** to define *all* this infrastructure. Version control, peer review, automated testing (e.g., `tflint`, `checkov`). *No manual console changes in production.*
2.  **Immutable Infrastructure:** **Always** use Managed Instance Groups (MIGs) with auto-healing/scaling for VMs. Deploy new versions via canary/blue-green using MIG `versions`. Never SSH to fix prod VMs.
3.  **Observability:** **Integrate from Day 1.**
    *   **Cloud Monitoring:** Custom dashboards for LB errors, VM CPU, disk latency, CDN hit ratio, DNS query latency. Set *meaningful* alerts (e.g., LB 5xx rate > 1%, VM CPU > 80% sustained).
    *   **Cloud Logging:** Centralize logs. Use MQL for complex alerts (e.g., "5xx errors increasing faster than traffic"). Enable VPC Flow Logs, Firewall Deny Logs, Audit Logs.
    *   **Error Budgets:** Define SLOs (e.g., 99.9% LB success rate) and track error budget consumption.
4.  **Security Hardening:**
    *   **Principle of Least Privilege (PoLP):** IAM roles per service account (e.g., VM SA has *only* `storage.objectViewer`).
    *   **VPC-SC (Service Controls):** *Essential for sensitive data:* Define perimeters restricting data egress (e.g., "Prod data can't leave VPC").
    *   **Shielded VMs + UBLA + CMEK + DNSSEC:** Baseline security config.
    *   **Cloud Armor:** WAF rules (OWASP Core Rule Set), IP allow/deny lists, rate limiting on *all* public LBs.
    *   **Regular Audits:** Use Security Command Center, run `gcloud asset search-all-resources`, review IAM.
5.  **Cost Optimization Engine:**
    *   **Tag Everything:** `env=prod`, `app=api`, `owner=team-xyz`. Use in Billing Reports.
    *   **Automated Sizing:** Use MIG auto-scaling based on CPU/requests. Use custom machine types.
    *   **Spot VMs for Stateful Workloads:** With MIGs + PDs + graceful shutdown.
    *   **Storage Lifecycle:** Aggressive transitions to `NEARLINE`/`ARCHIVE`.
    *   **Right-Size LBs:** Use Network LB only when needed; prefer regional services where possible.
    *   **Monitor Idle Resources:** `gcloud compute instances list --filter="status=RUNNING"` + low CPU.
6.  **Disaster Recovery (DR):**
    *   **Zonal Failure:** Covered by multi-zone MIGs + regional resources.
    *   **Regional Failure:** Requires multi-region design: Global LBs, data replicated (Cloud Storage dual-region, Spanner), stateless apps. **Test failover regularly!** (e.g., flip DNS, reroute traffic).

---

**This is the foundation you *must* master.** GCP evolves fast, but these core services and patterns are enduring. **As a DevOps Engineer:**

*   **Think Infrastructure as Code FIRST.**
*   **Automate Everything Possible** (deployment, scaling, healing, cleanup).
*   **Obsess Over Observability & SLOs.**
*   **Security is Non-Negotiable** (build it in, don't bolt it on).
*   **Cost is a Feature** – optimize relentlessly.

Go build, break, and fix things in a lab project. Nothing beats hands-on experience with these services. Good luck! 🚀

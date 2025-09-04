### **1. Resource Manager & Organization Policies**
**What it is**: The foundation for hierarchical resource organization (Organization > Folders > Projects) and centralized policy enforcement.

#### **Key Concepts**
- **Organization Resource**: Root node (requires domain-verified GCP Organization).
- **Folders**: Logical containers for Projects (unlimited nesting).
- **Policy Inheritance**: Policies set at higher levels (Org/Folder) apply to all children *unless* explicitly overridden at a lower level.
- **Constraints**: Predefined rules (e.g., `constraints/compute.vmExternalIpAccess` blocks public IPs).

#### **UI Walkthrough: Create a Folder**
1. **Go to**: [Cloud Console > Resource Manager](https://console.cloud.google.com/cloud-resource-manager)
2. **Click**: "CREATE FOLDER" (top-right)
3. **Fields**:
   - **Folder name**: *Mandatory*. Human-readable name (e.g., `prod-east`).
   - **Parent resource**: *Mandatory*. Select Organization or existing Folder.
   - **Folder ID**: *Optional*. System-generated; you can't set it here.
4. **Click**: "CREATE"
   > **Pro Tip**: Folders are critical for policy scoping. Always name them by environment/region (e.g., `dev`, `prod-eu`).

#### **gcloud CLI: Create Folder**
```bash
gcloud resource-manager folders create \
  --display-name="prod-east" \
  --parent=organizations/123456789012  # Replace with your Org ID
```

#### **Terraform: Create Folder**
```hcl
resource "google_folder" "prod_east" {
  display_name = "prod-east"
  parent       = "organizations/123456789012" # Org ID
}
```

---

### **2. Organization Policies & Constraints**
**What it is**: Enforce rules across your resource hierarchy (e.g., "Only allow specific regions").

#### **UI Walkthrough: Set a Policy (e.g., Restrict Compute Regions)**
1. **Go to**: [Cloud Console > Organization Policies](https://console.cloud.google.com/iam-admin/orgpolicies)
2. **Select Resource**: Choose Organization/Folder/Project at top.
3. **Find Policy**: Search for `constraints/compute.regions`.
4. **Edit Policy**:
   - **Policy type**: *Mandatory*. Choose:
     - *Allow all*: No restrictions.
     - *Allow custom*: Whitelist specific regions (e.g., `us-central1`, `europe-west1`).
     - *Deny custom*: Blacklist regions (rarely used).
   - **Custom value**: *Mandatory if "Allow custom"*. Enter regions (comma-separated, **no spaces**): `us-central1,europe-west1`
   - **Enforcement**: Toggle ON/OFF (default: ON).
5. **Click**: "SAVE"

> **Critical Notes**:
> - **Inheritance**: Policies set at Org level apply to all Projects unless overridden at Folder/Project level.
> - **Conflict Resolution**: Child policies **override** parent policies (e.g., Project policy > Folder policy).
> - **Common Constraints**:
>   - `constraints/iam.allowedPolicyMemberDomains`: Restrict IAM principals to your domain.
>   - `constraints/sql.restrictAuthorizedNetworks`: Force Cloud SQL to private IPs.
>   - `constraints/compute.vmCanIpForward`: Block IP forwarding (security risk).

#### **gcloud CLI: Restrict Compute Regions**
```bash
gcloud resource-manager org-policies set-policy - <<EOF
{
  "constraint": "constraints/compute.regions",
  "listPolicy": {
    "allowedValues": ["us-central1", "europe-west1"],
    "inheritFromParent": true
  }
}
EOF
--project=my-project  # Or --folder=123 --organization=123456789012
```

#### **Terraform: Restrict Compute Regions**
```hcl
resource "google_organization_policy" "restrict_regions" {
  constraint = "constraints/compute.regions"
  boolean_policy {
    enforced = true
  }
  list_policy {
    allowed_values = ["us-central1", "europe-west1"]
    inherit_from_parent = true # Critical for inheritance
  }
  org_id = "123456789012" # Or project/folder ID
}
```

---

### **3. Infrastructure as Code (IaC)**
**What it is**: Define cloud resources in config files (declarative), enabling version control and reproducibility.

#### **Key Tools**
- **Deployment Manager (DM)**: GCP-native (YAML/Jinja). Simple but limited.
- **Terraform**: Multi-cloud (HCL). Industry standard for complex setups.

#### **When to Use Which**
| **Tool**          | **Best For**                          | **Limitations**               |
|--------------------|---------------------------------------|-------------------------------|
| **Deployment Manager** | Simple GCP-only deployments          | No state locking, weak modularity |
| **Terraform**      | Production, multi-cloud, complex apps | steeper learning curve       |

---

### **4. Deployment Manager (DM)**
**What it is**: GCP's native IaC tool using YAML templates.

#### **UI Walkthrough: Deploy a VM**
1. **Go to**: [Cloud Console > Deployment Manager](https://console.cloud.google.com/dm/deployments)
2. **Click**: "CREATE DEPLOYMENT"
3. **Fields**:
   - **Deployment name**: *Mandatory*. Unique ID (e.g., `web-vm`).
   - **Config file**: *Mandatory*. Upload YAML (or use "Template" tab for samples).
     ```yaml
     # Example: simple-vm.yaml
     resources:
     - name: my-vm
       type: compute.v1.instance
       properties:
         zone: us-central1-a
         machineType: zones/us-central1-a/machineTypes/n1-standard-1
         disks:
         - deviceName: boot
           boot: true
           autoDelete: true
           initializeParams:
             sourceImage: projects/debian-cloud/global/images/family/debian-11
         networkInterfaces:
         - network: global/networks/default
     ```
   - **Preview**: *Optional*. Validates config.
4. **Click**: "DEPLOY"

> **Critical Notes**:
> - DM **does not manage state externally** (unlike Terraform). All state is in GCP.
> - Avoid for production: No drift detection, limited community support.

#### **gcloud CLI: Deploy with DM**
```bash
gcloud deployment-manager deployments create web-vm \
  --config=simple-vm.yaml
```

---

### **5. Terraform**
**What it is**: Industry-standard IaC tool (HashiCorp) with GCP provider.

#### **Minimal Setup (Critical for Production)**
1. **Enable APIs**:
   ```bash
   gcloud services enable cloudresourcemanager.googleapis.com \
     iam.googleapis.com \
     compute.googleapis.com
   ```
2. **Create Service Account** (for Terraform state):
   ```bash
   gcloud iam service-accounts create terraform \
     --display-name="Terraform Admin"
   gcloud projects add-iam-policy-binding my-project \
     --member="serviceAccount:terraform@my-project.iam.gserviceaccount.com" \
     --role="roles/editor"
   ```

#### **Terraform: Deploy a VM (Minimal Config)**
```hcl
# main.tf
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
      version = ">= 4.0"
    }
  }
}

provider "google" {
  project = "my-project" # Mandatory
  region  = "us-central1" # Mandatory
  zone    = "us-central1-a" # Optional but recommended
}

resource "google_compute_instance" "default" {
  name         = "web-vm" # Mandatory
  machine_type = "n1-standard-1" # Mandatory
  zone         = "us-central1-a" # Overrides provider zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11" # Mandatory
    }
  }

  network_interface {
    network = "default" # Mandatory
  }
}
```

#### **Run Terraform**
```bash
terraform init    # Downloads provider
terraform plan    # Shows changes
terraform apply   # Deploys resources
```

> **Pro Tips**:
> - **Always use `terraform state` commands** (e.g., `terraform state list`).
> - **Store state remotely** (use `google_storage_bucket` backend).
> - **Lock state** with Cloud Storage bucket (enable "Object Versioning").

---

### **6. Cloud Build (CI/CD)**
**What it is**: Managed CI/CD service for building, testing, deploying code.

#### **Core Components**
- **Build Config**: `cloudbuild.yaml` defining steps (Docker build, tests, deploy).
- **Triggers**: Automate builds on Git events (push, PR).
- **Secrets**: Securely pass credentials (via Secret Manager).

#### **UI Walkthrough: Create a Build Trigger**
1. **Go to**: [Cloud Console > Cloud Build > Triggers](https://console.cloud.google.com/cloud-build/triggers)
2. **Click**: "CREATE TRIGGER"
3. **Fields**:
   - **Name**: *Mandatory*. (e.g., `deploy-to-prod`).
   - **Event**: *Mandatory*. Choose:
     - *Push to a branch*: Build on git push.
     - *Pull request*: Build on PR open/update.
   - **Source**: *Mandatory*.
     - *Repository*: Select Cloud Source Repo (or GitHub/Bitbucket).
     - *Branch*: Regex for branches (e.g., `^main$` for main branch).
   - **Build configuration**: *Mandatory*. Choose:
     - *cloudbuild.yaml*: Best practice (store config in repo).
     - *Inline config*: Paste YAML directly (not recommended).
   - **Substitution variables**: *Optional*. e.g., `_IMAGE_NAME: my-app`
4. **Click**: "CREATE"

#### **Minimal `cloudbuild.yaml` Example**
```yaml
steps:
  # Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/my-app', '.']
  
  # Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/my-app']
  
  # Deploy to Cloud Run (example)
  - name: 'gcr.io/google-cloud-sdk/gcloud'
    entrypoint: gcloud
    args: ['run', 'deploy', 'my-service', '--image', 'gcr.io/$PROJECT_ID/my-app', '--region', 'us-central1']
```

#### **gcloud CLI: Create Trigger**
```bash
gcloud builds triggers create github \
  --name="deploy-main" \
  --repo-name="my-repo" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml" \
  --repo-owner="my-github-username"
```

#### **Terraform: Create Build Trigger**
```hcl
resource "google_cloudbuild_trigger" "deploy_main" {
  name        = "deploy-main"
  description = "Deploy on main push"
  trigger_template {
    branch_name = "^main$"
    repo_name   = "my-repo"
  }
  filename = "cloudbuild.yaml" # Path in repo
}
```

---

### **7. Artifact Registry**
**What it is**: Managed artifact repository (Docker, Maven, npm, Python, etc.).

#### **UI Walkthrough: Create a Docker Repository**
1. **Go to**: [Cloud Console > Artifact Registry > Repositories](https://console.cloud.google.com/artifact-registry)
2. **Click**: "CREATE REPOSITORY"
3. **Fields**:
   - **Repository name**: *Mandatory*. (e.g., `my-docker-repo`).
   - **Format**: *Mandatory*. Select `Docker`.
   - **Location**: *Mandatory*. Choose region (e.g., `us-central1`). **Critical for latency/cost**.
   - **Description**: *Optional*.
   - **Customer-managed encryption key (CMEK)**: *Optional*. For advanced security.
4. **Click**: "CREATE"

> **Repository Formats**:
> - **Docker**: `gcr.io` (legacy) or `us-central1-docker.pkg.dev`
> - **Maven**: `us-central1-maven.pkg.dev`
> - **npm**: `us-central1-npm.pkg.dev`
> - **Python**: `us-central1-python.pkg.dev`

#### **Push Docker Image (gcloud)**
```bash
# Authenticate
gcloud auth configure-docker us-central1-docker.pkg.dev

# Build & push
docker build -t us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app:v1 .
docker push us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app:v1
```

#### **Terraform: Create Docker Repo**
```hcl
resource "google_artifact_registry_repository" "docker_repo" {
  location      = "us-central1" # Mandatory
  repository_id = "my-docker-repo" # Mandatory
  format        = "DOCKER" # Mandatory
  project       = "my-project"
}
```

---

### **8. Secret Manager**
**What it is**: Securely store/access API keys, passwords, certificates (NOT for encryption keys - use Cloud KMS).

#### **UI Walkthrough: Create & Access a Secret**
1. **Go to**: [Cloud Console > Secret Manager](https://console.cloud.google.com/security/secret-manager)
2. **Click**: "CREATE SECRET"
3. **Fields**:
   - **Secret ID**: *Mandatory*. (e.g., `db-password`). **Never store in code!**
   - **Secret value**: *Mandatory*. Paste your secret (e.g., `p@ssW0rd!`).
   - **Replication**: *Mandatory*. Choose:
     - *Automatic*: Multi-region (recommended).
     - *User-managed*: Single region (for latency-sensitive apps).
4. **Click**: "CREATE SECRET"
5. **Grant Access**: Click secret > "ADD MEMBER" > Assign IAM role `roles/secretmanager.secretAccessor` to service accounts/apps.

#### **Access Secret in Cloud Build (Example)**
```yaml
steps:
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        DB_PASS=$(gcloud secrets versions access latest \
          --secret="db-password" \
          --format="value(payload.data)" | base64 --decode)
        echo "Connecting to DB..."
```

#### **gcloud CLI: Create Secret**
```bash
echo -n "p@ssW0rd!" | gcloud secrets create db-password \
  --data-file=- \
  --replication-policy="automatic"
```

#### **Terraform: Create Secret**
```hcl
resource "google_secret_manager_secret" "db_password" {
  secret_id = "db-password" # Mandatory
  replication {
    automatic = true # Mandatory
  }
}

resource "google_secret_manager_secret_version" "secret_val" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = "p@ssW0rd!" # Mandatory
}
```

---

### **9. Cloud Source Repositories (Git)**
**What it is**: Fully managed, private Git repos integrated with GCP (like GitHub but simpler).

#### **UI Walkthrough: Create a Repo**
1. **Go to**: [Cloud Console > Source Repositories](https://console.cloud.google.com/code/repos)
2. **Click**: "ADD REPOSITORY"
3. **Fields**:
   - **Repository name**: *Mandatory*. (e.g., `my-app`).
   - **Location**: *Optional*. Default is global.
   - **Initialize with README**: *Optional*. Creates initial commit.
4. **Click**: "CREATE"

#### **Connect Local Git (gcloud)**
```bash
# Enable API first!
gcloud services enable sourcerepo.googleapis.com

# Authenticate
gcloud init
gcloud source repos clone my-app --project=my-project

cd my-app
echo "Hello GCP" > README.md
git add .
git commit -m "Initial commit"
git push -u origin master
```

#### **Terraform: Create Repo**
```hcl
resource "google_sourcerepo_repository" "default" {
  name     = "my-app" # Mandatory
  project  = "my-project"
}
```

> **Critical Notes**:
> - **No forks/PRs**: Basic Git only (use GitHub/GitLab for advanced workflows).
> - **Tight IAM integration**: Control access via `roles/source.reader`, `roles/source.writer`.

---

### **10. Policy Intelligence (Recommender API)**
**What it is**: AI-driven insights for optimizing policies (e.g., "Remove unused IAM roles"), **powered by Recommender API**.

#### **Key Features**
- **Policy Insights**: Analyzes Org Policies for misconfigurations (e.g., overly permissive constraints).
- **Security Health Analytics**: Finds security gaps (e.g., "VMs with public IPs").
- **Recommender API**: Programmatic access to insights (REST/gRPC).

#### **UI Walkthrough: View Policy Insights**
1. **Go to**: [Cloud Console > Security Command Center > Recommendations](https://console.cloud.google.com/security/command-center/recommendations)
2. **Filter by**: 
   - *Recommender ID*: `google.policy.insight.Recommender`
   - *Priority*: `HIGH` or `MEDIUM`
3. **View Insights**:
   - **Title**: e.g., "Remove unused IAM roles"
   - **Description**: Details of the issue.
   - **Impact**: Estimated cost/security risk.
   - **Action**: "FIX" button (applies changes via API).

> **How it Works**:
> 1. Recommender scans your resources against best practices.
> 2. Generates actionable recommendations.
> 3. Integrates with **Security Command Center** (free tier available).

#### **gcloud CLI: List Policy Insights**
```bash
gcloud recommender recommendations list \
  --project=my-project \
  --recommender=google.policy.insight.Recommender \
  --location=global
```

#### **Terraform: Not Applicable**
> Policy Intelligence is an **observability tool**, not an infrastructure component. Use it to inform your Terraform/DM configs.

---

### **Critical Best Practices Summary**
1. **Organization Policies**: Start with 3 constraints: 
   - `constraints/compute.regions` (restrict regions)
   - `constraints/iam.allowedPolicyMemberDomains` (restrict domains)
   - `constraints/compute.vmExternalIpAccess` (block public IPs)
2. **IaC**: **Always use Terraform** (not DM) for production. Store state in Cloud Storage with locking.
3. **Cloud Build**: 
   - Store `cloudbuild.yaml` in your repo (not inline).
   - Use **Secret Manager** for secrets (never hardcode).
4. **Artifact Registry**: Use **region-specific endpoints** (e.g., `us-central1-docker.pkg.dev`) for lowest latency.
5. **Secret Manager**: 
   - **Never use for encryption keys** (use Cloud KMS).
   - Rotate secrets automatically with **Terraform + Cloud Scheduler**.
6. **Policy Intelligence**: Check **Security Command Center weekly** - it finds critical gaps you'll miss.

---

### **Troubleshooting Cheat Sheet**
| **Issue**                          | **Solution**                                                                 |
|------------------------------------|------------------------------------------------------------------------------|
| "Policy constraint not enforced"   | Check inheritance: Parent policy might override child. Use `gcloud resource-manager org-policies describe` |
| Cloud Build fails with secret error | 1. Grant Cloud Build SA `secretmanager.secretAccessor`<br>2. Use `gcloud secrets add-iam-policy-binding` |
| Terraform "permission denied"      | Service account needs `roles/editor` + `roles/iam.serviceAccountTokenCreator` |
| Artifact Registry "PERMISSION_DENIED" | Enable `artifactregistry.repositories.uploadArtifacts` role for SA         |

### **1. Account & Billing Setup**
#### **Why it Matters**
*   **Foundation:** Nothing runs on GCP without a billing account linked to a project.
*   **Cost Control:** Prevents runaway costs and surprise bills.
*   **Compliance:** Essential for enterprise governance.

---

#### **1.1 Free Tier**
*   **What it is:** $300 credit + limited free usage of many services for 90 days after sign-up. *Not* a permanent free plan.
*   **Key Limits (Examples):**
    *   Compute Engine: 1 f1-micro VM per month (US regions)
    *   Cloud Storage: 5 GB regional storage
    *   BigQuery: 1 TB query processing / month
    *   Many APIs have daily free quotas.
*   **Critical Notes:**
    *   **Credit Expires:** After 90 days or when credit is used up.
    *   **Usage Continues:** Services *will* bill normally after free tier ends/expires if not stopped.
    *   **No Auto-Stop:** Free tier resources *do not* automatically stop. You *must* manage them.
*   **How to Check:**
    *   **UI:** Billing > Billing Overview > "Free Trial" section (if active). *Always* check "Spending to date".
    *   **CLI:** `gcloud alpha billing accounts list` (Shows active/free trial status)
*   **Action Required:** **Set up Budget Alerts IMMEDIATELY** (see below).

---

#### **1.2 Billing Accounts**
*   **What it is:** The *top-level container* for payment methods and linked projects. **One billing account can pay for many projects.** One project can only be linked to *one* billing account.
*   **Ownership:** Created by the individual who signs up (Personal Account) or by a Google Workspace admin (Organization Account). Can be managed by Billing Account Administrators.
*   **Key Concepts:**
    *   **Payment Profile:** Bank account / credit card details attached to the billing account.
    *   **Billing Account Number:** Unique ID (e.g., `XXXXXX-XXXXXX-XXXXXX`).
    *   **Project Linking:** Projects *must* be explicitly linked to a billing account to use paid services.

##### **Creating a Billing Account (UI)**
1.  **Prerequisite:** Must be a **Billing Account Creator** (usually requires Google Account owner or Workspace admin).
2.  **Steps:**
    *   Go to **Cloud Console > Billing** ([https://console.cloud.google.com/billing](https://console.cloud.google.com/billing))
    *   Click **"Manage billing accounts"**.
    *   Click **"CREATE ACCOUNT"**.
    *   **Fill Form:**
        *   **Account Name:** *Mandatory.* Descriptive name (e.g., "Prod-Billing-2024"). *No technical impact.*
        *   **Account ID:** *Mandatory.* Unique alphanumeric ID (e.g., `prod-billing-2024`). *Immutable after creation.* Choose wisely.
        *   **Account Type:** *Mandatory.*
            *   `Personal account`: For individual use (uses personal payment method).
            *   `Organization account`: For company use (requires Google Workspace or Cloud Identity domain). *Strongly preferred for enterprises.*
        *   **Organization (if Account Type = Organization):** *Mandatory if applicable.* Select your Organization from the dropdown. *This links billing to your Org hierarchy.*
    *   Click **"CONTINUE"**.
    *   **Enter Payment Information:** *Mandatory.* Fill in Credit Card/Bank details. *Billing account is inactive until payment is verified.*
    *   Click **"SUBMIT AND ENABLE ACCOUNT"**.
3.  **Result:** New billing account created. **You MUST link projects to it** (see below).

##### **Linking a Project to a Billing Account (UI)**
1.  **Prerequisite:** Must be **Billing Account User** on the Billing Account *and* **Project Owner/Editor** on the Project.
2.  **Steps:**
    *   Go to **Cloud Console > Billing**.
    *   Select the desired **Billing Account** from the dropdown (top-left).
    *   Click **"LINK A PROJECT"**.
    *   **Select Project:** *Mandatory.* Choose from the list of projects you have access to. *Project must exist first.*
    *   Click **"LINK"**.
3.  **Result:** Project starts accruing charges to this billing account. **This is the single most critical step after project creation.**

##### **CLI (`gcloud`) - Link Project to Billing**
```bash
# List available billing accounts (Requires Billing Account User role)
gcloud alpha billing accounts list

# Link project (my-project-id) to billing account (XXXXXX-XXXXXX-XXXXXX)
gcloud beta billing projects link my-project-id \
    --billing-account XXXXXX-XXXXXX-XXXXXX
```
*   **Mandatory Fields:** `project-id`, `billing-account-id`.
*   **Note:** `alpha`/`beta` commands require enabled Cloud Billing API.

##### **Terraform - Link Project to Billing**
```hcl
# Provider Configuration (Requires Billing Account User role)
provider "google" {
  billing_project = "my-project-id" # Project managing billing links
  user_project_override = true
}

# Resource: Link Project to Billing Account
resource "google_project_billing_info" "main" {
  project_id         = "my-project-id" # Target project ID
  billing_account_id = "XXXXXX-XXXXXX-XXXXXX" # Billing Account ID
}
```
*   **Mandatory Fields:** `project_id`, `billing_account_id`.
*   **Critical:** The Terraform execution project (`billing_project`) needs `roles/billing.user` on the billing account.

---

#### **1.3 Budget Alerts**
*   **Why:** Prevent bill shock. Get notified *before* you exceed spending limits.
*   **How it Works:** Define a budget (monthly spend limit). Set alert thresholds (e.g., 50%, 90%, 100% of budget).

##### **Creating a Budget Alert (UI)**
1.  **Prerequisite:** Must be **Billing Account Viewer** or higher on the Billing Account.
2.  **Steps:**
    *   Go to **Cloud Console > Billing**.
    *   Select the **Billing Account**.
    *   Click **"Budgets & alerts"** > **"CREATE BUDGET"**.
    *   **Configure Budget:**
        *   **Name:** *Mandatory.* Descriptive (e.g., "Prod-Monthly-$500").
        *   **Scope:** *Mandatory.*
            *   `All projects linked to this billing account`: (Default) Tracks total spend.
            *   `Specific projects`: Select individual projects. *Crucial for cost allocation.*
            *   `Services`, `SKUs`, `Labels`: Advanced filtering.
        *   **Budget Amount:** *Mandatory.* Numeric value (e.g., `500`). Currency is set by billing account.
        *   **Time Frame:** *Mandatory.* `Monthly` (most common), `Quarterly`, `Annually`.
        *   **Forecast Alerts:** *Optional but Recommended.* Check "Notify me when forecasted to exceed budget".
    *   **Configure Alerts:**
        *   **Alert Rules:** *Mandatory Thresholds.* Click "ADD ALERT RULE".
            *   **Threshold:** *Mandatory.* `% of budget` (e.g., `50`, `90`, `100`). Or `Actual spend` (less common).
            *   **Frequency:** *Mandatory.* `As soon as threshold is met` (Recommended) or `Daily`.
            *   **Emails:** *Mandatory.* Comma-separated list of **Google Group emails** (BEST PRACTICE) or individual emails. *Never use personal emails only.*
        *   *(Repeat for multiple thresholds)*
    *   **(Optional) Budget Actions:** Set actions (e.g., disable APIs) at 100% (Advanced).
    *   Click **"SAVE"**.

##### **CLI (`gcloud`) - Create Budget (Simplified)**
*   **Note:** Budget creation via `gcloud` is complex (uses Cloud Billing Budget API). **UI is strongly recommended for initial setup.** Terraform is better for IaC.
*   Basic command structure (requires JSON config):
    ```bash
    gcloud billing budgets create --billing-account=XXXXXX-XXXXXX-XXXXXX \
        --config-file=budget-config.json
    ```
    *   `budget-config.json` must define all fields (name, amount, thresholds, emails). See [API docs](https://cloud.google.com/billing/docs/how-to/budgets#create-a-budget).

##### **Terraform - Create Budget**
```hcl
resource "google_billing_budget" "monthly_budget" {
  billing_account = "XXXXXX-XXXXXX-XXXXXX" # Billing Account ID
  display_name    = "Prod-Monthly-$500"

  amount {
    specified_amount {
      currency_code = "USD" # Usually USD
      units         = "500" # Budget amount
    }
  }

  threshold_rules {
    threshold_percent = 0.5 # 50%
    spend_basis       = "CURRENT_SPEND" # Forecasted spend
  }
  threshold_rules {
    threshold_percent = 0.9 # 90%
    spend_basis       = "CURRENT_SPEND"
  }
  threshold_rules {
    threshold_percent = 1.0 # 100%
    spend_basis       = "CURRENT_SPEND"
  }

  all_updates_rule {
    pubsub_topic = "" # Optional PubSub topic
    schema_version = "1.0"
    monitoring_notification_channels = [] # Optional Monitoring channels
    disable_default_iam_recipients = false
    # Email recipients defined here OR via Monitoring channels
    email_subject = "Budget Alert: ${google_billing_budget.monthly_budget.display_name}"
    # CRITICAL: Use Google Groups!
    monitoring_notification_channels = [
      "projects/my-project-id/notificationChannels/12345" # Get ID from Monitoring API
    ]
  }
}
```
*   **Mandatory Fields:** `billing_account`, `display_name`, `amount` (with `specified_amount`), at least one `threshold_rules`.
*   **Critical Email Setup:** `all_updates_rule` requires either `monitoring_notification_channels` (preferred, uses Monitoring API channels) or `pubsub_topic`. **Direct email lists are NOT supported in Terraform resource.** Use Google Groups configured in Monitoring.

---

### **2. Navigating the GCP Console**
*   **Why:** Efficient navigation is crucial for managing resources.
*   **Key Areas:**
    *   **Top Bar:** Project selector (MOST IMPORTANT), Search, Notifications, Account.
    *   **Navigation Menu (☰):** All GCP services (Compute, Storage, Networking, Databases, IAM & Admin, etc.).
    *   **Main Content Area:** Service-specific UI.
    *   **Cloud Shell:** Integrated terminal (bottom-right).
*   **Pro Tip:** Use **/search** in the top search bar for instant service/resource lookup.

---

### **3. Projects, Resources & Organization Hierarchy**
#### **3.1 Core Concepts**
*   **Resource:** Any GCP component (VM, Bucket, Database, Function).
*   **Project:** *Fundamental organizational unit.* Container for resources. Has:
    *   Unique ID (`my-project-id-123`)
    *   Unique Name (Human-readable, editable)
    *   Number (Immutable, e.g., `123456789012`)
    *   **Mandatory Link:** To a Billing Account.
    *   **Scope:** IAM policies, APIs, quotas, data location.
*   **Organization (`Organization`):** *Top-level container for enterprises.* Requires:
    *   A Google Workspace or Cloud Identity domain.
    *   Created by domain super admin.
    *   **Holds:** Folders, Projects, Policies (IAM, Org Policy, Audit Logs), Billing Accounts.
*   **Folder:** *Intermediate container* within an Organization. For:
    *   Grouping projects (e.g., `prod`, `dev`, `department-finance`).
    *   Inheriting IAM policies & Organization Policies downward.
    *   Can contain other folders (hierarchy).

#### **3.2 Hierarchy Flow**
```
Organization (mycompany.com)
│
├── Folder: Production
│   ├── Folder: Web Apps
│   │   ├── Project: Prod-Web-App-A
│   │   └── Project: Prod-Web-App-B
│   └── Folder: Databases
│       └── Project: Prod-DB-Cluster
│
├── Folder: Development
│   ├── Project: Dev-Web-App
│   └── Project: Dev-DB
│
└── Project: Shared-Services (e.g., Central Logging)
```
*   **Inheritance:** Policies set at a higher level (Org/Folder) apply to all children *unless* explicitly overridden at a lower level. **Least Restrictive Policy Wins? NO! Most Restrictive Policy Wins.** (Org Policy exception: Deny rules override allow).

---

#### **Creating an Organization (UI)**
*   **Prerequisite:** **Google Workspace or Cloud Identity domain admin** (e.g., `admin@mycompany.com`). *Individual Gmail accounts CANNOT create Orgs.*
*   **Steps:**
    1.  Go to **Cloud Console > IAM & Admin > Settings** ([https://console.cloud.google.com/iam-admin/settings](https://console.cloud.google.com/iam-admin/settings)).
    2.  Under **Organization**, click **"CREATE"**.
    3.  **Domain Selection:** *Mandatory.* Select your verified domain (e.g., `mycompany.com`) from the dropdown. *Must be managed by your admin account.*
    4.  Click **"CREATE"**.
*   **Result:** Organization created. Root node of your hierarchy. **Only needs to be done ONCE per domain.**

#### **Creating a Folder (UI)**
*   **Prerequisite:** Must be **Folder Admin** (`roles/resourcemanager.folderAdmin`) on the parent (Org or Folder).
*   **Steps:**
    1.  Go to **Cloud Console > IAM & Admin > Organization** ([https://console.cloud.google.com/iam-admin/organization](https://console.cloud.google.com/iam-admin/organization)).
    2.  Select the **Organization** or **Parent Folder** where you want the new folder.
    3.  Click **"CREATE"** (top button).
    4.  **Fill Form:**
        *   **Folder Name:** *Mandatory.* Descriptive (e.g., `Production`). *No technical impact.*
        *   **Folder ID:** *Mandatory.* Unique ID within parent (e.g., `prod`). *Immutable after creation.* Choose wisely.
    5.  Click **"CREATE"**.

#### **Creating a Project (UI)**
*   **Prerequisite:** Must be **Project Creator** (`roles/resourcemanager.projectCreator`) on the parent (Org or Folder).
*   **Steps:**
    1.  Go to **Cloud Console > IAM & Admin > Manage Resources** ([https://console.cloud.google.com/cloud-resource-manager](https://console.cloud.google.com/cloud-resource-manager)).
    2.  Select the **Organization** or **Parent Folder** where you want the project.
    3.  Click **"CREATE PROJECT"**.
    4.  **Fill Form:**
        *   **Project Name:** *Mandatory.* Human-readable (e.g., `Prod-Web-App-A`). *Editable later.*
        *   **Project ID:** *Mandatory.* **UNIQUE GLOBALLY** (e.g., `prod-web-app-a-123`). *Immutable after creation.* **CRITICAL:** Choose carefully (lowercase, hyphens, no underscores). *This is used in URLs, APIs, CLI.*
        *   **Parent Resource:** *Mandatory.* Organization or Folder to contain this project. *Determines policy inheritance.*
        *   **Location (Optional):** For App Engine/Datastore. Usually leave blank for most projects.
    5.  Click **"CREATE"**.
    6.  **LINK BILLING ACCOUNT:** *MANDATORY NEXT STEP.* Immediately go to Billing and link it (as shown earlier).

#### **CLI (`gcloud`) - Create Project**
```bash
# Create project under an Organization (Org ID = 123456789012)
gcloud projects create prod-web-app-a-123 \
    --name="Prod-Web-App-A" \
    --organization=123456789012

# Create project under a Folder (Folder ID = folders/456789)
gcloud projects create dev-db-001 \
    --name="Dev-DB" \
    --folder=456789
```
*   **Mandatory Fields:** `project-id` (must be globally unique), `--organization` *or* `--folder`.
*   **Note:** Billing link must be done *separately* (as shown in Billing section).

#### **Terraform - Create Project (Under Folder)**
```hcl
# Provider needs permissions on parent resource
provider "google" {
  # Credentials with permissions on folder
}

resource "google_project" "prod_web" {
  name            = "Prod-Web-App-A"   # Human-readable
  project_id      = "prod-web-app-a-123" # MUST be globally unique, immutable
  folder_id       = "folders/456789"     # Parent Folder ID (or organization = "123456789012")
  billing_account = "XXXXXX-XXXXXX-XXXXXX" # Link billing HERE (Terraform-only convenience)
}
```
*   **Mandatory Fields:** `name`, `project_id`, `folder_id` *or* `organization`, `billing_account`.
*   **Key Advantage:** Terraform links billing *in one step* via `billing_account` (uses `google_project_billing_info` internally).

---

### **4. Core Billing Concepts**
#### **4.1 SKUs (Stock Keeping Units)**
*   **What:** Unique identifiers for *specific billable products/features*.
*   **Example:** `Compute Engine Instance Core running in us-central1` has SKU `B833-6D1C-9D7C`.
*   **Why:** Used in detailed cost reports and billing exports to pinpoint *exactly* what you're paying for.
*   **Find SKUs:**
    *   **UI:** Billing > Cost Table > Click a line item > "SKU" column.
    *   **API:** [Cloud Billing Catalog API](https://cloud.google.com/billing/docs/reference/rest/v1/services.skus/list)
    *   **Export:** `sku.id` field in BigQuery billing export.

#### **4.2 Cost Reports**
*   **What:** Visualizations and breakdowns of your spending.
*   **Key Reports (UI - Billing > Reports):**
    *   **Summary:** Total spend over time (daily/weekly/monthly).
    *   **Cost Table:** Detailed line-item costs (grouped by Project, Service, SKU, etc.).
    *   **Filtering:** CRITICAL - Use `Project`, `Service`, `SKU`, `Labels`, `Credits` to analyze.
    *   **Grouping:** Group by `Project`, `Service`, `Location`, `SKU`, `Label` for cost allocation.
    *   **Forecasting:** Estimates future spend based on current usage.
*   **Pro Tip:** **Apply Cost Labels** (e.g., `env=prod`, `team=marketing`) to resources *during creation* for accurate cost allocation.

#### **4.3 Billing Exports**
*   **What:** Automatically export detailed billing data to BigQuery or Cloud Storage for deep analysis.
*   **Why:** Essential for custom reporting, anomaly detection, chargeback/showback.
*   **How (UI - Billing > Billing Export):**
    1.  Select **Billing Account**.
    2.  Click **"BigQuery export"** or **"Cloud Storage export"**.
    3.  **BigQuery Export Setup:**
        *   **Dataset Project:** *Mandatory.* Project to hold the dataset (e.g., `billing-reports`).
        *   **Dataset Name:** *Mandatory.* Name of BigQuery dataset (e.g., `gcp_billing_export`).
        *   **Export Frequency:** `Daily` (Recommended) or `Monthly`.
        *   **(Optional) Schema Version:** `v1` (Current) or `v2` (Newer fields).
    4.  Click **"SAVE"**.
    5.  **Permissions:** Ensure Billing Account has `BigQuery Data Editor` on the dataset project.
*   **Result:** Daily tables (`gcp_billing_export_YYYYMMDD`) appear in BigQuery. Contains *every single line item* (SKU, usage, cost, labels, project, etc.).
*   **CLI/Terraform:** Complex setup (requires BigQuery dataset + permissions + export config). **UI is simplest for initial setup.** Terraform example available but lengthy (uses `google_billing_account_budget` + BigQuery resources).

---

### **5. Cloud SDK (gcloud, gsutil, bq)**
*   **What:** Command-line tools for GCP management.
    *   `gcloud`: Core CLI (Compute, IAM, Projects, etc.)
    *   `gsutil`: Cloud Storage CLI
    *   `bq`: BigQuery CLI
*   **Why:** Automation, scripting, IaC (Terraform), advanced operations not in UI.

#### **5.1 Installation**
*   **Best Method:** Install **Google Cloud SDK** package.
    *   **Windows:** [Installer](https://cloud.google.com/sdk/docs/install#windows)
    *   **Mac (Homebrew):** `brew install google-cloud-sdk`
    *   **Linux:** [Script](https://cloud.google.com/sdk/docs/install#linux)
*   **Includes:** `gcloud`, `gsutil`, `bq`, `docker` (for Cloud Build), `kubectl` (for GKE).

#### **5.2 Authentication**
*   **User Login (Interactive):**
    ```bash
    gcloud auth login
    # Opens browser for OAuth. Grants access to *your* user account.
    ```
*   **Service Account Key (Non-Interactive - For Scripts/CI):**
    1.  Create SA key (see Service Accounts section).
    2.  ```bash
        gcloud auth activate-service-account SA_NAME@PROJECT_ID.iam.gserviceaccount.com \
            --key-file=/path/to/key.json
        ```
*   **Application Default Credentials (ADC):** Used by client libraries. Set via:
    ```bash
    gcloud auth application-default login  # For user auth
    gcloud auth application-default login --impersonate-service-account=SA_EMAIL  # Impersonation
    ```

#### **5.3 CLI Basics**
*   **Core Command Structure:**
    ```bash
    gcloud [GROUP] [COMMAND] [ARGS] [FLAGS]
    # Example:
    gcloud compute instances create my-vm --zone=us-central1-a --machine-type=e2-micro
    ```
*   **Essential Commands:**
    *   `gcloud config list`: Show active config (project, account, zone).
    *   `gcloud config set project MY_PROJECT`: Set default project.
    *   `gcloud projects list`: List accessible projects.
    *   `gcloud compute instances list`: List VMs.
    *   `gsutil ls gs://my-bucket`: List bucket contents.
    *   `bq query "SELECT * FROM mydataset.mytable LIMIT 10"`: Run BigQuery query.
*   **Help:** `gcloud help [COMMAND]` or `gcloud [GROUP] [COMMAND] --help`

---

### **6. Identity & Access Management (IAM)**
*   **What:** Controls **WHO** (Principal) can do **WHAT** (Role) on **WHICH** (Resource) in GCP.
*   **Core Principle:** **Least Privilege** - Grant only the minimum permissions needed.

#### **6.1 Principals**
*   **Who is accessing?**
    *   **Google Account:** `user:alex@example.com` (Individual human)
    *   **Service Account:** `serviceAccount:my-sa@project-id.iam.gserviceaccount.com` (Non-human identity)
    *   **Google Group:** `group:admins@example.com` (Collection of users/SAs)
    *   **G Suite/Cloud Identity Domain:** `domain:example.com` (All users in domain)
    *   **Special Groups:** `allUsers` (Anyone on internet), `allAuthenticatedUsers` (Any authenticated Google user)

#### **6.2 Roles**
*   **What permissions are granted?** A Role = Collection of **Permissions** (e.g., `storage.objects.create`).
*   **Role Types:**
    *   **Primitive Roles (LEGACY - Avoid):** `Owner`, `Editor`, `Viewer`. **Too broad!** Grant excessive permissions. Only use if absolutely necessary (rare).
    *   **Predefined Roles:** **RECOMMENDED.** Finely-grained, service-specific (e.g., `roles/storage.admin`, `roles/compute.viewer`). Hundreds exist. [Full List](https://cloud.google.com/iam/docs/understanding-roles)
    *   **Custom Roles:** Create your *own* role by combining specific permissions. Use when Predefined roles are too broad/too narrow. *Requires `roles/iam.roleAdmin`.*
        *   **Type:** `Basic` (Org-level) or `Project` (Project-level).
        *   **Limitations:** Max 100 permissions, cannot include `*.admin` permissions (usually).

#### **6.3 Policies**
*   **What:** The actual binding of **Principals** to **Roles** on a **Resource** (Project, Folder, Org, Bucket, etc.).
*   **Structure:**
    ```json
    {
      "bindings": [
        {
          "role": "roles/editor",
          "members": [
            "user:alex@example.com",
            "group:devs@example.com"
          ]
        },
        {
          "role": "roles/storage.objectViewer",
          "members": ["serviceAccount:web-sa@my-project.iam.gserviceaccount.com"]
        }
      ]
    }
    ```

##### **Setting IAM Policy (UI - Project Level)**
1.  Go to **Cloud Console > IAM & Admin > IAM**.
2.  Select your **Project**.
3.  **Add Principal:**
    *   Click **"+ ADD"**.
    *   **New members:** *Mandatory.* Enter email (user, group, SA). *Use Groups!*
    *   **Select a role:** *Mandatory.* **SEARCH** for the *most specific* Predefined role possible (e.g., `Storage Admin` NOT `Editor`). Avoid Primitive roles.
    *   **(Optional) Condition:** Advanced (e.g., time-based access). *Use sparingly.*
    *   Click **"SAVE"**.
4.  **Editing/Removing:** Click pencil icon next to member/role. Click trash icon to remove.

##### **CLI (`gcloud`) - Add IAM Binding**
```bash
# Grant 'roles/storage.objectViewer' to a SA on a PROJECT
gcloud projects add-iam-policy-binding my-project-id \
    --member="serviceAccount:web-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# Grant 'roles/compute.viewer' to a Group on a SPECIFIC VM (Resource-level)
gcloud compute instances add-iam-policy-binding my-vm \
    --zone=us-central1-a \
    --member="group:devs@example.com" \
    --role="roles/compute.viewer"
```
*   **Mandatory Fields:** `resource-id` (project-id, vm-name, etc.), `--member`, `--role`.
*   **Resource Scope:** Command changes policy on the *resource* specified by the command (`projects`, `compute instances`, `storage buckets`).

##### **Terraform - IAM Binding (Project Level)**
```hcl
resource "google_project_iam_binding" "storage_viewer" {
  project = "my-project-id" # Project ID
  role    = "roles/storage.objectViewer"

  members = [
    "serviceAccount:web-sa@my-project.iam.gserviceaccount.com",
    "group:devs@example.com",
  ]
}
```
*   **Mandatory Fields:** `project`, `role`, `members`.
*   **Alternatives:** `google_project_iam_member` (for single member), `google_project_iam_binding` (for multiple members per role - more efficient).
*   **Resource-Level:** Use resources like `google_storage_bucket_iam_binding`, `google_compute_instance_iam_binding`.

---

### **7. Service Accounts (SAs)**
*   **What:** **Non-human identities** for applications/services to authenticate and authorize *themselves* to GCP APIs. *Not for humans.*
*   **Why:** Securely run code without using user credentials. Essential for VMs, Cloud Functions, Cloud Run, etc.
*   **Email Format:** `SA_NAME@PROJECT_ID.iam.gserviceaccount.com` (e.g., `web-sa@my-project.iam.gserviceaccount.com`)

#### **7.1 Creation**
##### **Creating a Service Account (UI)**
1.  Go to **Cloud Console > IAM & Admin > Service Accounts**.
2.  Select your **Project**.
3.  Click **"CREATE SERVICE ACCOUNT"**.
4.  **Fill Form:**
    *   **Service account name:** *Mandatory.* Descriptive (e.g., `Web App SA`). *No technical impact.*
    *   **Service account ID:** *Mandatory.* Unique ID within project (e.g., `web-sa`). *Immutable.* Forms part of email (`web-sa@...`). **Choose short, lowercase, hyphenated.**
    *   **Description (Optional):** e.g., "Accesses Storage for web app".
5.  Click **"CREATE AND CONTINUE"**.
6.  **Grant Access (Optional but Common):** *Skip for now.* Best practice: **Create SA first, THEN grant minimal IAM roles separately.**
7.  Click **"DONE"**.

##### **CLI (`gcloud`) - Create SA**
```bash
gcloud iam service-accounts create web-sa \
    --display-name="Web App Service Account" \
    --project=my-project-id
```
*   **Mandatory Fields:** `SA-ID` (e.g., `web-sa`), `--project`.

##### **Terraform - Create SA**
```hcl
resource "google_service_account" "web_sa" {
  account_id   = "web-sa" # ID within project (forms email)
  display_name = "Web App Service Account"
  project      = "my-project-id" # Optional if default project set
}
```
*   **Mandatory Fields:** `account_id`.

#### **7.2 Key Management**
*   **Critical:** **Keys are SECRETS.** Treat like passwords. **NEVER commit to source control.**
*   **Best Practice:** **AVOID KEYS IF POSSIBLE.** Use:
    *   **Compute Engine / GKE / Cloud Run / Cloud Functions:** Attach SA directly to the resource (no key needed).
    *   **Application Default Credentials (ADC):** `gcloud auth application-default login` (for local dev).
*   **When Keys *Are* Necessary:** (e.g., on-prem servers, non-GCP clouds)
    *   **Create Key (UI):**
        1.  Go to SA details page (IAM > Service Accounts > [SA Name]).
        2.  Click **"ADD KEY"** > **"Create new key"**.
        3.  **Key Type:** `JSON` (Recommended). *P12 is legacy.*
        4.  Click **"CREATE"**. **Browser downloads JSON key file IMMEDIATELY.**
    *   **Security:**
        *   **Rotate Keys:** Regularly (e.g., 90 days). Delete old keys.
        *   **Limit Keys:** Max 10 keys per SA.
        *   **Store Securely:** Use secret manager (Secret Manager, Vault), *never* plaintext.
*   **CLI Key Creation (Use Sparingly):**
    ```bash
    gcloud iam service-accounts keys create ~/key.json \
        --iam-account=web-sa@my-project.iam.gserviceaccount.com
    ```
    *   **Mandatory Fields:** `--iam-account` (full SA email).

#### **7.3 Impersonation**
*   **What:** Allow one principal (User or SA) to **temporarily act as** another SA. **Eliminates need for SA keys.**
*   **Why:** Secure delegation. User/SA gets SA's permissions without handling its key.
*   **Requirements:**
    *   **Granter:** Must have `roles/iam.serviceAccountTokenCreator` on the *target SA*.
    *   **Grantee:** Must have `roles/iam.serviceAccountActor` (implicit for users with Token Creator role).
*   **How (CLI - User Impersonates SA):**
    ```bash
    # Set default auth to impersonate
    gcloud config set auth/impersonate_service_account web-sa@my-project.iam.gserviceaccount.com

    # Or use --impersonate-service-account flag per command
    gcloud storage ls gs://my-bucket --impersonate-service-account=web-sa@my-project.iam.gserviceaccount.com
    ```
*   **How (Code - Using ADC):**
    ```python
    from google.auth import impersonated_credentials
    from google.cloud import storage

    target_sa = "web-sa@my-project.iam.gserviceaccount.com"
    source_credentials, _ = google.auth.default() # Your user/SA credentials
    credentials = impersonated_credentials.Credentials(
        source_credentials=source_credentials,
        target_principal=target_sa,
        target_scopes=['https://www.googleapis.com/auth/cloud-platform'],
        lifetime=3600  # Seconds
    )
    client = storage.Client(credentials=credentials)
    ```
*   **Terraform:** Uses `impersonate_service_account` in provider config:
    ```hcl
    provider "google" {
      impersonate_service_account = "web-sa@my-project.iam.gserviceaccount.com"
      # Credentials must have Token Creator role on web-sa
    }
    ```

---

### **Critical Summary & Best Practices**

1.  **Billing is King:** Always set up Budget Alerts *immediately*. Link projects *explicitly*.
2.  **Hierarchy Matters:** Use Folders for logical grouping (Prod/Dev) and policy inheritance. Avoid flat project structures.
3.  **IAM Principle:** **LEAST PRIVILEGE.** Use Predefined Roles. **NEVER use Primitive Roles (Owner/Editor) unless absolutely forced.** Use Groups, not individuals.
4.  **Service Accounts > Keys:** **Avoid SA keys whenever possible.** Use resource attachment or Impersonation. If keys *must* be used, rotate and store securely.
5.  **Labels are Crucial:** Apply cost and environment labels (`env=prod`, `app=web`) during resource creation for cost tracking and management.
6.  **Terraform for IaC:** Use Terraform for project/folder/billing setup and IAM. More reliable than UI/CLI for complex setups.
7.  **Organization Account:** **ALWAYS** use an Organization Account (with domain) for enterprises. Avoid personal billing accounts.
8.  **Free Tier ≠ Free Forever:** Monitor usage closely. Set budgets *below* free tier limits initially.

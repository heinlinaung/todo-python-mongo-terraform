# Terraform Azure Deployment Guide

A step-by-step guide for deploying this ToDo app to Azure using Terraform and the Azure Developer CLI (`azd`). Written for Pluralsight cloud sandbox environments (single resource group, limited permissions).

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Azure Resource Group (rg-<env>)                 │
│                                                                     │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  Log         │───▶│  Application     │    │  Azure Portal    │  │
│  │  Analytics   │    │  Insights        │    │  Dashboard       │  │
│  │  Workspace   │    │  (monitoring)    │    │  (charts/metrics)│  │
│  └──────────────┘    └────────┬─────────┘    └──────────────────┘  │
│                               │ telemetry                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  App Service Plan (B3 Linux, shared)                         │  │
│  │                                                              │  │
│  │  ┌────────────────────────┐  ┌────────────────────────────┐ │  │
│  │  │  Web App (Node 20)     │  │  API App (Python 3.10)     │ │  │
│  │  │  React SPA             │  │  FastAPI + gunicorn        │ │  │
│  │  │  served via pm2        │  │  SystemAssigned Identity ──┼─┼──┐│
│  │  │  port 3000             │  │  port 8000                 │ │  ││
│  │  └────────────┬───────────┘  └────────────────────────────┘ │  ││
│  │               │ VITE_API_BASE_URL                            │  ││
│  └───────────────┼─────────────────────────────────────────────┘  ││
│                  │ HTTP requests                                    ││
│                  ▼                                                  ││
│  ┌──────────────────────────────────────────────────────────────┐  ││
│  │  Key Vault                                                   │◀─┘│
│  │  secret: AZURE-COSMOS-CONNECTION-STRING                      │   │
│  └───────────────────────────┬──────────────────────────────────┘   │
│                              │ connection string                     │
│                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Cosmos DB (MongoDB API, Serverless)                         │   │
│  │  database: "Todo"                                            │   │
│  │  collections: TodoList, TodoItem                             │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### How it fits together

| Component | Azure Resource | Purpose |
|---|---|---|
| Frontend | App Service (Node 20-lts) | Serves the React SPA with pm2 |
| Backend API | App Service (Python 3.10) | Runs FastAPI via gunicorn + uvicorn |
| Database | Cosmos DB (MongoDB API, Serverless) | Stores todo lists and items |
| Secrets | Key Vault | Holds the Cosmos connection string |
| Monitoring | Application Insights + Log Analytics | Traces, metrics, dashboards |
| Shared compute | App Service Plan (B3 Linux) | Shared by both App Services |

### Secret flow at runtime

```
API App starts
    → reads AZURE_KEY_VAULT_ENDPOINT from app settings
    → authenticates to Key Vault using Managed Identity (no passwords!)
    → fetches secret "AZURE-COSMOS-CONNECTION-STRING"
    → connects to Cosmos DB
```

### Terraform module structure

```
infra/
├── provider.tf          # azurerm + azurecaf providers, TF version constraint
├── variables.tf         # location, environment_name, principal_id, useAPIM
├── main.tf              # resource group + all module calls
├── output.tf            # exports URLs, connection strings, etc. for azd
├── main.tfvars.json     # maps azd env vars → TF variables
└── modules/
    ├── loganalytics/        # Log Analytics Workspace
    ├── applicationinsights/ # App Insights + Portal Dashboard
    ├── cosmos/              # Cosmos DB account, database, 2 collections
    ├── keyvault/            # Key Vault + access policies + secrets
    ├── appserviceplan/      # Shared B3 Linux App Service Plan
    ├── appservicepython/    # API App Service (Python)
    ├── appservicenode/      # Web App Service (Node)
    ├── apim/                # API Management (optional, off by default)
    └── apim-api/            # APIM API definition (optional, off by default)
```

---

## Prerequisites

Install these tools on your local machine before starting:

| Tool | Version | Install |
|---|---|---|
| [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) | latest | `brew install azure-cli` (Mac) |
| [Azure Developer CLI (azd)](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd) | latest | `brew tap azure/azd && brew install azd` (Mac) |
| [Terraform](https://developer.hashicorp.com/terraform/install) | >= 1.1.7, < 2.0 | `brew tap hashicorp/tap && brew install hashicorp/tap/terraform` (Mac) |
| [Node.js](https://nodejs.org/) | 18+ | `brew install node` (Mac) |
| [Python](https://www.python.org/downloads/) | 3.10+ | `brew install python@3.10` (Mac) |
| [Git](https://git-scm.com/) | latest | pre-installed on Mac |

Verify everything is installed:

```bash
az --version
azd version
terraform version
node --version
python3 --version
```

---

## Part 1: Understanding the Terraform Code

Before deploying, take 10 minutes to read through the key files:

### 1.1 `infra/provider.tf` — provider configuration

```hcl
terraform {
  required_version = ">= 1.1.7, < 2.0.0"
  required_providers {
    azurerm = { version = "~>3.97.1" }   # Azure resource management
    azurecaf = { version = "~>1.2.24" }  # Azure naming conventions
  }
}

provider "azurerm" {
  skip_provider_registration = "true"    # Required for sandboxes!
  ...
}
```

> **Sandbox note**: `skip_provider_registration = "true"` is critical. In a Pluralsight sandbox the service principal cannot register new resource providers at the subscription level. This flag skips that step.

### 1.2 `infra/variables.tf` — inputs

| Variable | Required | Description |
|---|---|---|
| `location` | Yes | Azure region (e.g. `eastus`) |
| `environment_name` | Yes | Name for this deployment (e.g. `myapp`) |
| `principal_id` | No | Your user object ID, for Key Vault access |
| `useAPIM` | No | Deploy API Management? (default: false) |

### 1.3 `infra/main.tf` — the orchestration

This file creates the resource group, then calls each module in dependency order:

1. Resource Group → 2. Log Analytics → 3. App Insights → 4. Cosmos DB → 5. Key Vault → 6. App Service Plan → 7. Web App → 8. API App

### 1.4 `infra/output.tf` — what gets exported

After `terraform apply`, azd reads these outputs and injects them as environment variables into the app deployments. For example, `API_BASE_URL` becomes `VITE_API_BASE_URL` in the frontend build.

---

## Part 2: Pluralsight Sandbox Setup

### 2.1 Log in to your sandbox

1. Start a Pluralsight cloud sandbox lab (Azure)
2. Copy the **Username**, **Password**, and **Resource Group** name from the lab panel
3. Open your terminal and log in:

```bash
az login --username "<sandbox-username>" --password "<sandbox-password>"
```

Or use the interactive browser login:

```bash
az login
```

### 2.2 Find your subscription and resource group

```bash
# List your subscriptions (sandbox usually has one)
az account list --output table

# Set the active subscription
az account set --subscription "<subscription-id>"

# Verify you can see the pre-created resource group
az group list --output table
```

> **Important**: Pluralsight sandboxes give you a **pre-created resource group** (e.g. `learn-abc123`). This Terraform config creates its **own** resource group by default. See [Part 5](#part-5-adapting-for-single-resource-group) for how to adapt to the sandbox constraint.

### 2.3 Find your object ID (for Key Vault access)

```bash
az ad signed-in-user show --query id --output tsv
```

Save this value — you'll need it as `principal_id`.

---

## Part 3: Initialize azd

### 3.1 Clone and enter the repo

```bash
git clone <repo-url>
cd todo-python-mongo-terraform
```

### 3.2 Initialize azd environment

```bash
azd init --no-prompt
```

This creates a `.azure/` directory locally (gitignored). It stores your environment config.

### 3.3 Create a new environment

```bash
azd env new myapp
```

Replace `myapp` with any name (lowercase letters, numbers, hyphens only, max 20 chars). This becomes the base of your resource names.

### 3.4 Set required environment variables

```bash
# Azure region — pick one available in your sandbox
azd env set AZURE_LOCATION eastus

# Your user object ID from step 2.3
azd env set AZURE_PRINCIPAL_ID "<your-object-id>"

# Subscription ID from step 2.2
azd env set AZURE_SUBSCRIPTION_ID "<subscription-id>"
```

Verify your environment:

```bash
azd env get-values
```

---

## Part 4: Deploy (Happy Path — No Restrictions)

If your sandbox allows creating new resource groups, this is the quickest path:

### 4.1 Provision infrastructure

```bash
azd provision
```

This runs `terraform init` + `terraform plan` + `terraform apply`. Expect it to take **8–12 minutes**. You will see:

- Resource group created: `rg-myapp`
- All Azure resources provisioned in order
- Terraform outputs captured by azd

### 4.2 Deploy application code

```bash
azd deploy
```

This:
1. Builds the React frontend (`npm run build` with env vars from Terraform outputs)
2. Packages the Python API
3. Deploys both to their App Services via SCM (Kudu)

### 4.3 Open the app

```bash
azd show
```

This prints the web app URL. Open it in your browser — you should see the ToDo application.

### 4.4 One-command shortcut (provision + deploy together)

```bash
azd up
```

---

## Part 5: Adapting for Single Resource Group (Sandbox Constraint)

Pluralsight sandboxes often give you a pre-created resource group and don't allow creating new ones. Here's how to handle this.

### Option A: Use the existing resource group name as your environment name

The resource group name is generated as `rg-<environment_name>`. So if your sandbox resource group is named `rg-learnapp`, set:

```bash
azd env new learnapp   # or rename your existing env
```

This makes Terraform try to create `rg-learnapp` — which already exists. Terraform will adopt it (since the name matches).

> However, if the naming doesn't match (sandbox creates `learn-abc123` rather than `rg-<name>`), use Option B.

### Option B: Override the resource group in Terraform

Edit `infra/main.tf` to import the existing resource group instead of creating a new one.

**Step 1**: Comment out the resource group creation and replace with a data source:

```hcl
# Comment out these lines in infra/main.tf:
# resource "azurecaf_name" "rg_name" { ... }
# resource "azurerm_resource_group" "rg" { ... }

# Add this instead:
data "azurerm_resource_group" "rg" {
  name = "learn-abc123"   # your sandbox resource group name
}
```

**Step 2**: Update all references from `azurerm_resource_group.rg` to `data.azurerm_resource_group.rg`:

```bash
# Search for all references that need updating
grep -n "azurerm_resource_group.rg" infra/main.tf
```

Replace each occurrence:
- `azurerm_resource_group.rg.name` → `data.azurerm_resource_group.rg.name`
- `azurerm_resource_group.rg.location` → `data.azurerm_resource_group.rg.location`
- `azurerm_resource_group.rg.tags` → `data.azurerm_resource_group.rg.tags`

**Step 3**: Update `infra/variables.tf` to add a variable for the resource group name:

```hcl
variable "resource_group_name" {
  type        = string
  description = "Name of the existing resource group"
}
```

**Step 4**: Set it in your azd environment:

```bash
azd env set RESOURCE_GROUP_NAME "learn-abc123"
```

**Step 5**: Add it to `infra/main.tfvars.json`:

```json
{
  "location": "${AZURE_LOCATION}",
  "environment_name": "${AZURE_ENV_NAME}",
  "principal_id": "${AZURE_PRINCIPAL_ID}",
  "resource_group_name": "${RESOURCE_GROUP_NAME}",
  "useAPIM": "${USE_APIM=false}"
}
```

### Option C: Run Terraform directly (skip azd)

If azd is causing friction, you can run Terraform directly with a `.tfvars` file:

```bash
cd infra

terraform init

# Create a tfvars file with your values
cat > sandbox.tfvars <<EOF
location         = "eastus"
environment_name = "myapp"
principal_id     = "<your-object-id>"
useAPIM          = false
EOF

terraform plan -var-file="sandbox.tfvars"
terraform apply -var-file="sandbox.tfvars"
```

---

## Part 6: Step-by-Step Terraform Walkthrough (Learning Mode)

Run each step manually to understand what Terraform is doing.

### 6.1 Initialize Terraform

```bash
cd infra
terraform init
```

This downloads the provider plugins (`azurerm` and `azurecaf`) into `.terraform/`. Check what was downloaded:

```bash
ls .terraform/providers/
```

### 6.2 Review the plan (no changes made yet)

```bash
terraform plan -var-file="../.azure/myapp/.env" \
  -var="location=eastus" \
  -var="environment_name=myapp" \
  -var="principal_id=<your-object-id>"
```

Or using a tfvars file:

```bash
terraform plan -var-file="sandbox.tfvars"
```

Read the plan output carefully:
- Lines starting with `+` = resources to create
- Lines starting with `-` = resources to destroy
- Lines starting with `~` = resources to modify
- `Plan: X to add, Y to change, Z to destroy` summary at the bottom

### 6.3 Apply in stages (to observe each resource)

Instead of applying everything at once, target individual resources:

```bash
# Stage 1: Create only the resource group
terraform apply -target="azurerm_resource_group.rg" -var-file="sandbox.tfvars"

# Verify in Azure portal or CLI
az group show --name "rg-myapp"

# Stage 2: Create Log Analytics
terraform apply -target="module.loganalytics" -var-file="sandbox.tfvars"

# Stage 3: Create App Insights
terraform apply -target="module.applicationinsights" -var-file="sandbox.tfvars"

# Stage 4: Create Cosmos DB (this takes ~3 minutes)
terraform apply -target="module.cosmos" -var-file="sandbox.tfvars"

# Stage 5: Create App Service Plan
terraform apply -target="module.appserviceplan" -var-file="sandbox.tfvars"

# Stage 6: Create Web and API App Services
terraform apply -target="module.web" -target="module.api" -var-file="sandbox.tfvars"

# Stage 7: Create Key Vault (needs API identity from stage 6)
terraform apply -target="module.keyvault" -var-file="sandbox.tfvars"

# Stage 8: Set CORS origins (needs both web URI and API name)
terraform apply -target="null_resource.api_set_allow_origins" -var-file="sandbox.tfvars"

# Stage 9: Apply remaining resources / confirm nothing left
terraform apply -var-file="sandbox.tfvars"
```

### 6.4 Inspect Terraform state

```bash
# List all managed resources
terraform state list

# Show details of a specific resource
terraform state show module.cosmos.azurerm_cosmosdb_account.cosmos_account

# Show all outputs
terraform output
```

### 6.5 View the Terraform state file

```bash
# State is stored locally at:
ls terraform.tfstate
cat terraform.tfstate | python3 -m json.tool | head -100
```

The state file maps Terraform resource names → Azure resource IDs. **Never edit this manually.**

---

## Part 7: Deploy Application Code

After infrastructure is provisioned, deploy the app code.

### 7.1 Deploy with azd

```bash
cd ..   # back to repo root
azd deploy
```

### 7.2 Manual deploy (if not using azd)

**Frontend:**

```bash
cd src/web
npm install

# Set the API URL (from terraform output)
API_URL=$(terraform -chdir=infra output -raw API_BASE_URL)
echo "VITE_API_BASE_URL=$API_URL" > .env.local

npm run build   # produces dist/

# Deploy dist/ to the web App Service
az webapp deploy \
  --resource-group "rg-myapp" \
  --name "<web-app-name>" \
  --src-path dist/ \
  --type zip
```

**Backend:**

```bash
cd src/api
pip install -r requirements.txt

# Package and deploy
zip -r api.zip . -x "*.pyc" -x "__pycache__/*"
az webapp deploy \
  --resource-group "rg-myapp" \
  --name "<api-app-name>" \
  --src-path api.zip \
  --type zip
```

---

## Part 8: Verify the Deployment

### 8.1 Check App Service health

```bash
# Get the web app URL
terraform -chdir=infra output REACT_APP_WEB_BASE_URL

# Get the API URL
terraform -chdir=infra output API_BASE_URL

# Check the API health endpoint
curl "$(terraform -chdir=infra output -raw API_BASE_URL)/health"
```

### 8.2 View logs

```bash
# Stream API logs
az webapp log tail \
  --resource-group "rg-myapp" \
  --name "<api-app-name>"

# Or in the Azure Portal: App Service → Log stream
```

### 8.3 Check App Insights

1. Open the Azure Portal
2. Navigate to your resource group
3. Open the Application Insights resource
4. Check **Live Metrics**, **Failures**, and **Performance** blades

---

## Part 9: Clean Up

When you're done learning, destroy all resources to avoid charges.

### 9.1 Destroy via azd

```bash
azd down
```

This runs `terraform destroy` and removes all resources.

### 9.2 Destroy via Terraform directly

```bash
cd infra
terraform destroy -var-file="sandbox.tfvars"
```

Type `yes` when prompted. This takes 5–10 minutes.

> **Note**: Key Vault uses soft-delete (even with `purge_protection_enabled = false`). If you redeploy with the same environment name within 90 days, you may need to purge it first:
> ```bash
> az keyvault purge --name "<kv-name>" --location eastus
> ```

---

## Part 10: Common Issues & Fixes

### "Insufficient permissions to register resource providers"

Already handled: `skip_provider_registration = "true"` is set in `provider.tf`. If you still see this error, the providers haven't been pre-registered in your sandbox. Contact your sandbox provider or try a different region.

### "Cannot create resource group"

Your sandbox may not allow creating new resource groups. Follow [Part 5, Option B](#option-b-override-the-resource-group-in-terraform) to use the pre-existing one.

### "App Service name already taken"

App Service names must be globally unique. The `resource_token` (a 13-char hash) handles this automatically. If you see a collision, try a different `environment_name`.

### Key Vault name conflict (soft-delete)

```bash
# List soft-deleted vaults
az keyvault list-deleted --output table

# Purge a specific one
az keyvault purge --name "kv-<resource_token>" --location eastus
```

### Cosmos DB taking too long

Cosmos DB account creation can take 5–8 minutes. This is normal. Don't cancel the apply.

### API can't connect to Cosmos DB

Check that the API App's managed identity has Key Vault access:

```bash
# Show managed identity object ID
terraform -chdir=infra output   # look for IDENTITY_PRINCIPAL_ID in state

# Verify Key Vault access policy
az keyvault show --name "kv-<token>" --query properties.accessPolicies
```

### CORS errors in browser

The `null_resource.api_set_allow_origins` sets the `API_ALLOW_ORIGINS` app setting after both apps are created. If you see CORS errors, verify:

```bash
az webapp config appsettings list \
  --resource-group "rg-myapp" \
  --name "<api-app-name>" \
  --query "[?name=='API_ALLOW_ORIGINS']"
```

---

## Quick Reference

```bash
# Login
az login
azd auth login

# Initialize
azd env new <name>
azd env set AZURE_LOCATION eastus
azd env set AZURE_SUBSCRIPTION_ID <id>
azd env set AZURE_PRINCIPAL_ID <object-id>

# Deploy everything
azd up

# Or step by step
azd provision   # Terraform only
azd deploy      # App code only

# View deployed URLs
azd show

# Tear down
azd down

# Terraform directly (from infra/)
terraform init
terraform plan  -var-file="sandbox.tfvars"
terraform apply -var-file="sandbox.tfvars"
terraform state list
terraform output
terraform destroy -var-file="sandbox.tfvars"
```

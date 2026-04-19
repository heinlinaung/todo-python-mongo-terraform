# Terraform-Only Deployment Guide

Deploy this ToDo app to Azure using **pure Terraform commands** — no Azure Developer CLI (`azd`) involved.

---

## Prerequisites

| Tool | Version | Install (Mac) |
|---|---|---|
| [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) | latest | `brew install azure-cli` |
| [Terraform](https://developer.hashicorp.com/terraform/install) | >= 1.1.7, < 2.0 | `brew tap hashicorp/tap && brew install hashicorp/tap/terraform` |
| [Node.js](https://nodejs.org/) | 18+ | `brew install node` |
| [Python](https://www.python.org/downloads/) | 3.10+ | `brew install python@3.10` |

Verify:

```bash
az --version
terraform version
node --version
python3 --version
```

---

## Overview of Steps

```
1. Log in to Azure
2. Create a terraform.tfvars file
3. terraform init
4. terraform plan
5. terraform apply
6. Read outputs (URLs, keys)
7. Build & deploy frontend (npm)
8. Deploy backend (zip + az webapp deploy)
9. Verify the app works
10. terraform destroy (cleanup)
```

---

## Step 1 — Log in to Azure

```bash
az login
```

For a Pluralsight sandbox with username/password credentials:

```bash
az login --username "<sandbox-username>" --password "<sandbox-password>"
```

Set your active subscription:

```bash
az account list --output table
az account set --subscription "<subscription-id>"
```

Find your user object ID (needed for Key Vault access):

```bash
az ad signed-in-user show --query id --output tsv
```

Save the output — you will use it in Step 2.

---

## Step 2 — Create a `terraform.tfvars` file

All Terraform commands below are run from the `infra/` directory. Go there first:

```bash
cd infra
```

Create a `terraform.tfvars` file with your values:

```hcl
location         = "eastus"
environment_name = "myapp"
principal_id     = "<your-object-id-from-step-1>"
useAPIM          = false
```

> **`location`** — Azure region. Common sandbox regions: `eastus`, `westus2`, `westeurope`.
>
> **`environment_name`** — Becomes the base of all resource names (e.g. `rg-myapp`, `kv-<hash>`). Use lowercase letters, numbers, hyphens only. Keep it short (under 15 chars).
>
> **`principal_id`** — Your user object ID. This grants you permission to read secrets from Key Vault. Leave it as `""` if you don't need local Key Vault access.
>
> **`useAPIM`** — Keep `false` for a sandbox. API Management takes 30+ minutes to provision and costs more.

> **Sandbox — single resource group restriction?**
> If your sandbox has a pre-existing resource group and blocks creating new ones, see
> [Appendix A](#appendix-a-using-a-pre-existing-resource-group) before running `terraform apply`.

---

## Step 3 — `terraform init`

Download the required provider plugins:

```bash
terraform init
```

Expected output:
```
Initializing the backend...
Initializing provider plugins...
- Installing hashicorp/azurerm v3.97.x ...
- Installing aztfmod/azurecaf v1.2.x ...
Terraform has been successfully initialized!
```

What this creates:
```
infra/
└── .terraform/
    └── providers/       ← downloaded provider binaries
.terraform.lock.hcl      ← locks exact provider versions
```

---

## Step 4 — `terraform plan`

Preview every resource that will be created — **no changes made yet**:

```bash
terraform plan -var-file="terraform.tfvars"
```

How to read the output:

| Symbol | Meaning |
|---|---|
| `+` green | Resource will be **created** |
| `-` red | Resource will be **destroyed** |
| `~` yellow | Resource will be **modified** |

Look for the summary line at the bottom:
```
Plan: 17 to add, 0 to change, 0 to destroy.
```

You should see these resources planned:
- `azurerm_resource_group.rg`
- `module.loganalytics` — 1 resource
- `module.applicationinsights` — 2 resources (App Insights + Portal Dashboard)
- `module.cosmos` — 3 resources (account + database + 2 collections)
- `module.keyvault` — 3 resources (vault + access policies + secret)
- `module.appserviceplan` — 1 resource
- `module.web` — 1 App Service
- `module.api` — 1 App Service
- `null_resource.api_set_allow_origins` — 1 local-exec

> **Tip:** Save the plan to a file and apply exactly that plan (no surprises):
> ```bash
> terraform plan -var-file="terraform.tfvars" -out="myplan.tfplan"
> terraform apply "myplan.tfplan"
> ```

---

## Step 5 — `terraform apply`

Create all the Azure resources:

```bash
terraform apply -var-file="terraform.tfvars"
```

Terraform will show the plan again and prompt:
```
Do you want to perform these actions?
  Enter a value: yes
```

Type `yes` and press Enter.

**Expected duration:** 8–15 minutes. The slow steps are:
- Cosmos DB account: ~5–8 minutes
- App Service provisioning: ~2–3 minutes

You will see each resource being created in real time:
```
azurerm_resource_group.rg: Creating...
azurerm_resource_group.rg: Creation complete after 2s
module.loganalytics...: Creating...
...
Apply complete! Resources: 17 added, 0 changed, 0 destroyed.
```

### Skip the confirmation prompt (CI/automation)

```bash
terraform apply -var-file="terraform.tfvars" -auto-approve
```

---

## Step 6 — Read the Outputs

After apply completes, read the values you need for deploying app code:

```bash
terraform output
```

Example output:
```
API_BASE_URL                    = "https://app-api-abc1234567890.azurewebsites.net"
AZURE_COSMOS_DATABASE_NAME      = "Todo"
AZURE_COSMOS_CONNECTION_STRING_KEY = "AZURE-COSMOS-CONNECTION-STRING"
REACT_APP_WEB_BASE_URL          = "https://app-web-abc1234567890.azurewebsites.net"
AZURE_LOCATION                  = "eastus"
```

> Sensitive outputs (`AZURE_KEY_VAULT_ENDPOINT`, `APPLICATIONINSIGHTS_CONNECTION_STRING`) are hidden by default.
> Reveal them individually:
> ```bash
> terraform output -raw AZURE_KEY_VAULT_ENDPOINT
> terraform output -raw APPLICATIONINSIGHTS_CONNECTION_STRING
> ```

Save these values — you need them in Steps 7 and 8.

---

## Step 7 — Build & Deploy the Frontend

```bash
cd ../src/web
npm install
```

Create a `.env.local` file using the Terraform outputs from Step 6:

```bash
# Replace the values with your actual outputs
echo 'VITE_API_BASE_URL="https://app-api-abc1234567890.azurewebsites.net"' > .env.local
echo 'VITE_APPLICATIONINSIGHTS_CONNECTION_STRING="<paste-value-here>"' >> .env.local
```

Or use command substitution to pull directly from Terraform:

```bash
API_URL=$(terraform -chdir=../infra output -raw API_BASE_URL)
APP_INSIGHTS=$(terraform -chdir=../infra output -raw APPLICATIONINSIGHTS_CONNECTION_STRING)

echo "VITE_API_BASE_URL=\"$API_URL\"" > .env.local
echo "VITE_APPLICATIONINSIGHTS_CONNECTION_STRING=\"$APP_INSIGHTS\"" >> .env.local
```

> **Why is this needed?** Vite bakes environment variables into the JavaScript bundle at build time.
> If you skip this step, the deployed app won't know where the API lives.

Build the frontend:

```bash
npm run build
```

This produces a `dist/` folder. Deploy it to the Web App Service:

```bash
# Get the web app name from Terraform state
WEB_APP_NAME=$(terraform -chdir=../infra state show module.web.azurerm_linux_web_app.appservice \
  | grep '^ *name ' | awk '{print $3}' | tr -d '"')

# Or just hardcode it — check the Azure Portal or `terraform output`
WEB_APP_NAME="app-web-abc1234567890"
RESOURCE_GROUP="rg-myapp"

# Zip and deploy
cd dist
zip -r ../web-dist.zip .
cd ..

az webapp deploy \
  --resource-group "$RESOURCE_GROUP" \
  --name "$WEB_APP_NAME" \
  --src-path web-dist.zip \
  --type zip
```

Clean up the local env file:

```bash
rm .env.local
```

---

## Step 8 — Deploy the Backend API

```bash
cd ../src/api
```

Package the Python source (exclude compiled files and caches):

```bash
zip -r ../../api.zip . \
  --exclude "*.pyc" \
  --exclude "__pycache__/*" \
  --exclude ".pytest_cache/*" \
  --exclude "tests/*" \
  --exclude ".env"
```

Deploy to the API App Service:

```bash
API_APP_NAME="app-api-abc1234567890"   # replace with your actual name
RESOURCE_GROUP="rg-myapp"

az webapp deploy \
  --resource-group "$RESOURCE_GROUP" \
  --name "$API_APP_NAME" \
  --src-path ../../api.zip \
  --type zip
```

> **No `pip install` needed locally.** The App Service setting `SCM_DO_BUILD_DURING_DEPLOYMENT=true`
> (set by Terraform in `main.tf`) tells Azure to run `pip install -r requirements.txt`
> on the server after the zip is uploaded.

---

## Step 9 — Verify the Deployment

### Check the web app

Open the web URL in your browser:

```bash
terraform -chdir=../infra output -raw REACT_APP_WEB_BASE_URL
```

You should see the ToDo application.

### Check the API health endpoint

```bash
API_URL=$(terraform -chdir=../infra output -raw API_BASE_URL)
curl "$API_URL/health"
# Expected: {"status":"ok"} or similar
```

### Stream live logs

```bash
# API logs
az webapp log tail \
  --resource-group "rg-myapp" \
  --name "app-api-abc1234567890"

# Web app logs
az webapp log tail \
  --resource-group "rg-myapp" \
  --name "app-web-abc1234567890"
```

### Inspect Terraform state

```bash
cd infra

# List every managed resource
terraform state list

# Inspect a specific resource in detail
terraform state show module.cosmos.azurerm_cosmosdb_account.cosmos_account
terraform state show module.api.azurerm_linux_web_app.appservice
terraform state show module.keyvault.azurerm_key_vault.keyvault
```

---

## Step 10 — Destroy (Clean Up)

When you are done, remove all Azure resources:

```bash
cd infra
terraform destroy -var-file="terraform.tfvars"
```

Type `yes` when prompted. Takes 5–10 minutes.

```bash
# Or skip the prompt:
terraform destroy -var-file="terraform.tfvars" -auto-approve
```

### Key Vault soft-delete warning

Even after destroy, Key Vault names are reserved for 90 days (soft-delete). If you redeploy
with the same `environment_name`, you may see an error like:
```
A resource with the ID "/subscriptions/.../vaults/kv-abc1234567890" already exists
```

Fix it by purging the deleted vault:

```bash
# List soft-deleted vaults
az keyvault list-deleted --output table

# Purge it (permanent, cannot be undone)
az keyvault purge --name "kv-abc1234567890" --location eastus
```

---

## Useful Terraform Commands (Reference)

```bash
# Initialise — run once, or after adding new providers/modules
terraform init

# Preview changes without applying
terraform plan -var-file="terraform.tfvars"

# Save plan to file, then apply exactly that plan
terraform plan -var-file="terraform.tfvars" -out="myplan.tfplan"
terraform apply "myplan.tfplan"

# Apply (with confirmation prompt)
terraform apply -var-file="terraform.tfvars"

# Apply (skip confirmation — for automation)
terraform apply -var-file="terraform.tfvars" -auto-approve

# Apply a single resource only (useful for learning/debugging)
terraform apply -target="module.cosmos" -var-file="terraform.tfvars"
terraform apply -target="azurerm_resource_group.rg" -var-file="terraform.tfvars"

# Show all outputs
terraform output

# Show a single output value (raw, no quotes — good for scripting)
terraform output -raw API_BASE_URL

# List all resources tracked in state
terraform state list

# Show full details of one resource in state
terraform state show module.cosmos.azurerm_cosmosdb_account.cosmos_account

# Refresh state (re-sync with actual Azure resources)
terraform refresh -var-file="terraform.tfvars"

# Format all .tf files (fix indentation/style)
terraform fmt

# Validate syntax without connecting to Azure
terraform validate

# Destroy all resources
terraform destroy -var-file="terraform.tfvars"

# Destroy a single resource only
terraform destroy -target="module.applicationinsights" -var-file="terraform.tfvars"
```

---

## Appendix A: Using a Pre-Existing Resource Group

Pluralsight sandboxes often provide a resource group (e.g. `learn-abc123`) and block creating new ones.
Here is how to modify the Terraform code to use it instead of creating one.

### 1. Edit `infra/main.tf`

Replace the resource group resource block with a data source:

```hcl
# REMOVE or comment out these two blocks:
# resource "azurecaf_name" "rg_name" { ... }
# resource "azurerm_resource_group" "rg" { ... }

# ADD this instead:
data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}
```

### 2. Update all references in `infra/main.tf`

Every module that references the resource group must be updated.
Find them all:

```bash
grep -n "azurerm_resource_group.rg\." infra/main.tf
```

Change every occurrence:
- `azurerm_resource_group.rg.name`     → `data.azurerm_resource_group.rg.name`
- `azurerm_resource_group.rg.location` → `data.azurerm_resource_group.rg.location`
- `azurerm_resource_group.rg.tags`     → `data.azurerm_resource_group.rg.tags`

### 3. Add the variable to `infra/variables.tf`

```hcl
variable "resource_group_name" {
  description = "Name of the pre-existing resource group (for sandbox environments)"
  type        = string
  default     = ""
}
```

### 4. Add the value to your `terraform.tfvars`

```hcl
location             = "eastus"
environment_name     = "myapp"
principal_id         = "<your-object-id>"
useAPIM              = false
resource_group_name  = "learn-abc123"
```

### 5. Continue from Step 3 normally

```bash
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"
```

---

## Appendix B: Finding Resource Names After Apply

If you need to find the actual Azure resource names created by Terraform:

```bash
cd infra

# All resource names at once
terraform state list | while read r; do
  echo "--- $r ---"
  terraform state show "$r" | grep '^ *name ' | head -1
done

# Or check the Azure Portal:
# Resource Groups → rg-myapp → see all resources listed

# Or use the Azure CLI:
az resource list --resource-group "rg-myapp" --output table
```

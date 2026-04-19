# How Azure Developer CLI (azd) Works

A plain-English explanation of `azd provision`, `azd deploy`, and how `azure.yaml` fits in.

---

## `azd provision` and `azd deploy` are built-in CLI commands

They are **not** defined in `azure.yaml`. They are hardcoded commands that ship with the `azd` tool itself — just like `git commit` or `npm install` are built-in to their CLIs.

```
azd  (the CLI tool)
 ├── built-in commands: provision, deploy, package, up, down, init, env, ...
 │        These always exist regardless of your project
 │
 └── azure.yaml
          This tells each command HOW to behave in YOUR project
```

---

## What does `azure.yaml` actually do?

It is a **configuration file** that customises the built-in commands for your specific project. Here is each section explained:

```yaml
name: todo-python-mongo-terraform      # just a label

infra:
  provider: terraform    # tells `azd provision`: "run Terraform" (not Bicep)
                         # tells it to look in ./infra/ for main.tf

services:
  web:                   # tells `azd deploy`: "there's a service called web"
    project: ./src/web   #   source code is here
    language: js         #   run npm install + npm run build
    host: appservice     #   deploy result to an Azure App Service
    dist: dist           #   the build output folder is dist/
    hooks:
      prepackage: ...    #   run this script BEFORE building
      postdeploy: ...    #   run this script AFTER deploying

  api:
    project: ./src/api
    language: py         #   run pip install + package the code
    host: appservice     #   deploy to an Azure App Service

workflows:
  up:                    # OVERRIDE the default behaviour of `azd up`
    steps:
      - azd: provision    #   step 1: run the built-in provision command
      - azd: deploy --all #   step 2: run the built-in deploy command
```

---

## What each command does, step by step

### `azd provision`

1. Reads `infra.provider: terraform` from `azure.yaml`
2. Goes to `./infra/` folder, runs `terraform init`
3. Reads `infra/main.tfvars.json`, substitutes your env vars (`AZURE_LOCATION`, `AZURE_ENV_NAME`, etc.)
4. Runs `terraform plan` then `terraform apply`
5. **Captures all `output` values from `infra/output.tf`** and saves them into `.azure/<env-name>/.env`

That last step is critical. After provision completes, your local `.env` file gains values like:

```
API_BASE_URL=https://app-api-xyz.azurewebsites.net
APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=...
```

These are then available to `azd deploy` in the next step.

### `azd deploy`

1. Reads the `services:` section from `azure.yaml`
2. Loads the env vars saved by `azd provision` from `.azure/<env-name>/.env`
3. For each service, runs in order:

**web service:**
```
prepackage hook runs
    → writes .env.local with API_BASE_URL from Terraform output
npm run build
    → Vite reads .env.local and bakes the URL into the JS bundle
zip dist/ folder
find the App Service tagged azd-service-name: web in Azure
upload zip to that App Service
postdeploy hook runs
    → deletes .env.local
```

**api service:**
```
zip src/api/ folder
find the App Service tagged azd-service-name: api in Azure
upload zip to that App Service
Azure runs: pip install -r requirements.txt  (server-side build)
```

#### How does `azd deploy` find the right App Service?

It searches Azure for resources tagged with `azd-service-name: web` (or `api`). Those tags were set in `infra/main.tf` when Terraform provisioned the resources:

```hcl
# infra/main.tf
module "web" {
  tags = merge(local.tags, { azd-service-name : "web" })
}

module "api" {
  tags = merge(local.tags, { "azd-service-name" : "api" })
}
```

The service name in `azure.yaml` must match the tag value in Terraform. No hardcoded resource names needed.

---

## What is `workflows.up`?

By default, `azd up` runs three steps in this order: `package → provision → deploy`.

The `workflows.up` block in `azure.yaml` **overrides** that default:

```yaml
workflows:
  up:
    steps:
      - azd: provision
      - azd: deploy --all
```

This version skips the separate `package` step (bundling it into `deploy`) and ensures provisioning always happens before deploying.

Running `azd up` follows the workflow steps. Running `azd provision` or `azd deploy` independently still works — the workflow only affects `azd up`.

---

## The full data flow

```
azure.yaml                         .azure/<env>/.env
    │                                      │
    │  infra.provider: terraform           │  AZURE_LOCATION=eastus
    │  infra.path: infra/ (default)        │  AZURE_ENV_NAME=myapp
    ▼                                      ▼
azd provision ──────────────────── terraform apply
                                           │
                                           │  outputs written back
                                           ▼
                                   .azure/<env>/.env  ← updated with:
                                           │          API_BASE_URL=...
                                           │          APPLICATIONINSIGHTS_CONNECTION_STRING=...
    azure.yaml                             │
    services.web.hooks.prepackage ─────────┤
         │                                 ▼
         │  echo VITE_API_BASE_URL=... > .env.local
         │                             azd deploy
         │                                 │
         │                                 ├── npm run build (reads .env.local)
         │                                 ├── zip dist/
         │                                 ├── find App Service by tag "web"
         │                                 └── upload zip
         │
         └── Vite bakes the URL permanently into the JS bundle
```

> **Why the prepackage hook matters:** The React frontend is a static SPA — once built,
> the API URL is compiled into the JavaScript files. There is no server to inject config
> at runtime. So the URL must be written to `.env.local` *before* `npm run build` runs,
> or the deployed app will have no idea where the API is.

---

## Summary table

| Thing | Built-in to azd? | Configured in azure.yaml? |
|---|---|---|
| `azd provision` command exists | ✅ Yes | ❌ No |
| `azd deploy` command exists | ✅ Yes | ❌ No |
| `azd up` command exists | ✅ Yes | ❌ No |
| "Use Terraform (not Bicep)" | ❌ No | ✅ `infra.provider` |
| "Look in `./infra/` folder" | ❌ No | ✅ `infra.path` (default: `infra/`) |
| How to run Terraform / Bicep | ✅ azd knows | ❌ No |
| Which services to deploy | ❌ No | ✅ `services:` section |
| How to build JS / Python code | ✅ azd knows | ❌ No |
| Run a script before the build | ❌ No | ✅ `hooks.prepackage` |
| Run a script after deploy | ❌ No | ✅ `hooks.postdeploy` |
| Order of steps in `azd up` | ✅ default order | ✅ override with `workflows:` |

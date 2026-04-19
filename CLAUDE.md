# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A full-stack ToDo application template for Azure deployment:
- **Frontend**: React 18 + TypeScript + Vite + Fluent UI (port 3000/5173)
- **Backend**: Python FastAPI + Beanie ODM (port 3100)
- **Database**: Azure Cosmos DB (MongoDB-compatible API)
- **Infrastructure**: Terraform for Azure resources
- **Deployment**: Azure Developer CLI (`azd`)

## Commands

### Backend (src/api)

```bash
cd src/api
pip install -r requirements.txt
# Run with a Cosmos connection string:
AZURE_COSMOS_CONNECTION_STRING="<conn>" API_ENVIRONMENT=develop uvicorn todo.app:app --port 3100 --reload
```

```bash
# Tests
pip install -r requirements-test.txt
AZURE_COSMOS_DATABASE_NAME=test_db python -m pytest tests/ -v
```

### Frontend (src/web)

```bash
cd src/web
npm install
npm run dev        # dev server (Vite proxies API calls to localhost:3100)
npm run build      # TypeScript compile + Vite production build
npm run lint       # ESLint (max-warnings 0 — zero warnings allowed)
npm run preview    # preview production build
```

### End-to-End Tests (tests/)

```bash
cd tests
npm install
npm test           # Playwright (Chromium), requires running app
```

### Infrastructure (infra/)

```bash
# Full provision + deploy via Azure Developer CLI
azd up

# Or separately
azd provision      # Terraform apply
azd deploy         # Deploy app code
```

## Architecture

### Data Flow

```
React (src/web) → Axios → FastAPI (src/api) → Beanie ODM → Azure Cosmos DB (MongoDB)
                                ↑
                        Azure Key Vault (connection string via Managed Identity)
                                ↓
                    Application Insights (OpenTelemetry traces)
```

### Backend Structure (src/api/todo/)

- `app.py` — FastAPI app creation, CORS config, OpenTelemetry instrumentation, Beanie init
- `routes.py` — REST endpoints for `/lists` and `/lists/{id}/items` (full CRUD)
- `models.py` — Beanie documents (`TodoList`, `TodoItem`) and Pydantic `Settings`

The `Settings` class loads from `.env` and Azure Key Vault. Set `API_ENVIRONMENT=develop` locally to disable CORS restrictions.

### Frontend Structure (src/web/src/)

- `App.tsx` — Root component with routing and `useReducer`-based state management
- `components/` — `TodoList`, `TodoItem`, `Telemetry`
- `services/` — Axios-based API client wrappers
- `reducers/` + `actions/` — Redux-like state update pattern
- `config/` — App configuration (reads `VITE_API_BASE_URL`)

Vite proxies API requests to `localhost:3100` in development — no CORS setup needed locally.

### Infrastructure (infra/)

Terraform modules under `infra/modules/`:
- `appserviceplan/` — B3 SKU plan shared by API and Web apps
- `appservicepython/` — API App Service (gunicorn + 4 uvicorn workers)
- `appservicenode/` — Web App Service (pm2 static file serving)
- `cosmos/` — Azure Cosmos DB serverless (MongoDB API)
- `keyvault/` — Key Vault storing Cosmos connection string
- `applicationinsights/` + `loganalytics/` — Monitoring stack
- `apim/` + `apim-api/` — Optional API Management (controlled by `useAPIM` variable)

Key Terraform variables: `location`, `environment_name`, `principal_id`, `useAPIM`.

### Azure Cosmos DB Note

The database uses Azure Cosmos DB for MongoDB (not native MongoDB). The `mongo_server_version` in Terraform must be compatible with Cosmos DB's supported versions.

## Local Development Notes

- Set `API_ENVIRONMENT=develop` to disable CORS — the API otherwise restricts origins to the deployed web app URL
- The `AZURE_COSMOS_CONNECTION_STRING` env var is required to start the API; locally use a `.env` file in `src/api/`
- VSCode tasks (`.vscode/tasks.json`) can start both API and Web together; debug configurations are in `.vscode/launch.json`
- Dev container (`.devcontainer/`) has Python 3.10, Node 18, Terraform, and Azure CLI pre-installed with ports 3000 and 3100 forwarded

## CI/CD

GitHub Actions workflow (`.github/workflows/azure-dev.yml`) triggers on push to main and runs `azd provision` then `azd deploy`. Required secrets: `AZURE_CREDENTIALS`, `ARM_CLIENT_SECRET`, `AZD_INITIAL_ENVIRONMENT_CONFIG`, `AZURE_ENV_NAME`, `AZURE_LOCATION`, `AZURE_SUBSCRIPTION_ID`, `ARM_TENANT_ID`, `ARM_CLIENT_ID`, plus Terraform remote state vars (`RS_RESOURCE_GROUP`, `RS_STORAGE_ACCOUNT`, `RS_CONTAINER_NAME`).

Azure DevOps pipeline config is in `.azdo/`.

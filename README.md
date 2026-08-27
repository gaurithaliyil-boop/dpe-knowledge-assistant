# DPE Knowledge Assistant

An AI-powered knowledge assistant for the Data Platform Engineering team, built on **Databricks Apps** with **FastAPI + React** and deployed via **Databricks Asset Bundles (DABs)** and Jenkins CI/CD.

Ask questions in natural language and get answers sourced from GitHub, Confluence, Slack (#ask-databricks), and the Genie SQL interface — all in one place.

## Features

- **Multi-source knowledge retrieval** — GitHub MCP, Slack MCP, Confluence vector search, and Genie run concurrently for each query
- **Slack bot** — automatically replies in threads to messages in `#ask-databricks`, responds to DMs, and answers `@DPE Knowledge Assistant` mentions in any channel
- **Per-user auth (OBO)** — downstream Databricks calls run as the logged-in user via `x-forwarded-access-token`
- **Bookmarks** — save and revisit assistant answers; stored per-user in Delta
- **FAQ panel** — top questions from `#ask-databricks`, refreshed daily by a notebook job

## Deployments

| Environment | Workspace |
|---|---|
| Development | https://ias-dataplatform-dev.cloud.databricks.com/ |
| Production | https://ias-dataplatform-prod.cloud.databricks.com/ |

## Repository Structure

```
├── databricks.yml              # Bundle config
├── app.dev.yaml                # Runtime config for dev deploys
├── app.prod.yaml               # Runtime config for prod deploys
├── app.local.yaml              # Runtime config for local development
├── resources/
│   ├── app.yml                 # Databricks App resource definition
│   └── jobs.yml                # FAQ generation daily job
├── src/
│   ├── app/                    # FastAPI backend + React SPA
│   │   ├── app.py              # FastAPI routes + Slack Socket Mode listener
│   │   ├── service.py          # Knowledge retrieval, MCP clients, chat logic
│   │   ├── app.yaml            # Active runtime config (overwritten by CI — do not edit directly)
│   │   ├── requirements.txt    # Python dependencies
│   │   └── frontend/           # React SPA (Vite + shadcn/ui)
│   └── notebooks/
│       └── slack/
│           └── 03_generate_faq.py  # Daily FAQ generation via Slack MCP
└── .ci/
    ├── dabs-setup.sh           # Configure Databricks CLI credentials and derive BRANCH_SLUG
    ├── dabs-build.sh           # Build frontend and install deps
    ├── dabs-validate.sh        # Select env app.yaml and validate the bundle
    ├── dabs-cleanup.sh         # Delete existing app and workspace bundle state
    ├── dabs-deploy.sh          # Deploy bundle and start the app
    ├── dabs-pr-cleanup.sh      # Delete the PR-specific dev app after merge
    ├── Jenkinsfile.groovy      # Jenkins pipeline definition
    └── version.properties      # Release version
```

## Quick Start

### Prerequisites

- Python 3.10+
- Node.js 18+ (for frontend builds)
- Databricks CLI (`databricks-cli: 0.297.2` — set in `.jervis.yml`)
- Access to IAS Dataplatform workspaces

### Local Setup

1. **Clone and configure:**

   ```bash
   git clone <repository-url>
   cd databricks-app-dpe-assistant
   cp conf/local-env.example .env
   # Edit .env with your token and endpoint details
   export $(grep -v '^#' .env | xargs)
   ```

2. **Install dependencies:**

   ```bash
   make install
   ```

3. **Validate bundle:**

   ```bash
   make validate
   ```

4. **Run locally:**

   ```bash
   make run-local
   ```

5. **Deploy to development:**

   ```bash
   make deploy
   ```

## App Configuration (env-level app.yaml)

Runtime config is managed per environment via files at the repo root:

| File | Used when |
|---|---|
| `app.dev.yaml` | CI dev deploys and `make validate` / `make deploy` |
| `app.prod.yaml` | CI prod deploys and `make deploy-prod` |
| `app.local.yaml` | `make run-local` |

Before each deploy the CI (and Makefile) copies the appropriate file to `src/app/app.yaml`. **Do not edit `src/app/app.yaml` directly** — it will be overwritten.

### Environment variables

| Variable | Description |
|---|---|
| `DATABRICKS_SERVING_BASE_URL` | Model serving endpoint base URL |
| `DATABRICKS_MODEL` | Serving endpoint name (injected via `valueFrom`) |
| `DATABRICKS_VECTOR_SEARCH_ENDPOINT` | Vector search endpoint name |
| `DATABRICKS_GITHUB_MCP_HOST` | Workspace host for GitHub MCP |
| `GITHUB_MCP_CONNECTION` | GitHub MCP connection name |
| `DATABRICKS_CONFLUENCE_INDEX_PATH` | Unity Catalog path for Confluence vector index |
| `DATABRICKS_SQL_WAREHOUSE_ID` | SQL warehouse ID |
| `DATABRICKS_FEEDBACK_TABLE` | Unity Catalog table for feedback |
| `DATABRICKS_BOOKMARKS_TABLE` | Unity Catalog table for bookmarks |
| `DATABRICKS_FAQ_TABLE` | Unity Catalog table for FAQ |
| `GENIE_SPACE_ID` | Genie space ID |
| `SLACK_SECRET_SCOPE` | Databricks secrets scope (default: `dataplatform`) |
| `SLACK_SIGNING_SECRET_KEY` | Secret key name for Slack signing secret |
| `SLACK_BOT_TOKEN_KEY` | Secret key name for Slack bot token (`xoxb-`) |
| `SLACK_APP_TOKEN_KEY` | Secret key name for Slack app-level token (`xapp-`) |
| `SLACK_ASK_DATABRICKS_CHANNEL_ID` | Channel ID for `#ask-databricks` |

## Authentication

The app uses two complementary auth models:

- **Service principal (M2M)** — Databricks Apps injects `DATABRICKS_CLIENT_ID` and `DATABRICKS_CLIENT_SECRET` automatically. Used for background operations (FAQ, bookmarks, Slack bot replies).
- **Per-user OBO** — The user's `x-forwarded-access-token` is forwarded to downstream Databricks calls (vector search, MCP, Genie) so they run with the caller's identity and permissions.

For local development, set `DATABRICKS_TOKEN` in `.env` as a fallback.

## Slack Bot

The bot uses **Socket Mode** (outbound WebSocket to Slack — no public URL required) and responds to:

- Every new top-level message in `#ask-databricks`
- Direct messages to the bot
- `@DPE Knowledge Assistant` mentions in any channel

### Required Slack app scopes

`channels:history`, `chat:write`, `im:history`, `im:write`, `app_mentions:read`

### Required secrets

```bash
databricks secrets put-secret dataplatform slack_bot_token   --string-value "xoxb-..."
databricks secrets put-secret dataplatform slack_app_token   --string-value "xapp-..."
```

The app's service principal must have READ permission on the secrets scope:
```bash
databricks secrets put-acl dataplatform <app-sp-client-id> READ
```

## CI/CD Pipeline

The pipeline runs as five discrete steps:

| Step | Script | Purpose |
|---|---|---|
| 1 | `dabs-setup.sh` | Write `.databrickscfg`, export `BRANCH_SLUG` from `RELEASE_VERSION` |
| 2 | `dabs-build.sh` | Build React frontend, install Python deps |
| 3 | `dabs-validate.sh` | Copy env app.yaml, run `databricks bundle validate` |
| 4 | `dabs-cleanup.sh` | Delete existing app and workspace bundle state |
| 5 | `dabs-deploy.sh` | `bundle deploy`, `bundle run` |

### Deployment strategy

**PR / feature branch** → deploys to **dev** as `dpe-knowledge-assistant-pr-<N>`

After the PR merges, `dabs-pr-cleanup.sh` deletes the PR app.

**Master branch** → deploys to **dev** as `dpe-knowledge-assistant`

**Tag** → validates on dev first, then deploys to prod (6-stage flow).

### Required Jenkins credentials

| Variable | Description |
|---|---|
| `DATABRICKS_TOKEN` | Databricks service principal token |
| `DATABRICKS_HOST` | Databricks workspace URL |
| `DATABRICKS_APP_ID` | App service principal user ID |
| `DATABRICKS_APP_NAME` | App display name |

### Manual deployment

```bash
export $(grep -v '^#' .env | xargs)
source .ci/dabs-setup.sh --ops_env=dev --ias_env=dev
.ci/dabs-build.sh
.ci/dabs-validate.sh
.ci/dabs-cleanup.sh
.ci/dabs-deploy.sh
```

## Makefile Commands

```bash
make install       # Install Python dependencies
make test          # Run tests
make lint          # Run linting
make format        # Format code
make validate      # Copy app.dev.yaml and validate bundle (requires credentials)
make deploy        # Copy app.dev.yaml, deploy + start in dev
make deploy-prod   # Copy app.prod.yaml, deploy + start in prod
make run-local     # Copy app.local.yaml and run app locally
make status        # Show bundle status
make clean         # Remove .pyc, __pycache__, test_results.xml
make setup         # Create .venv
```

## Troubleshooting

**`DATABRICKS_TOKEN` not set**: Load credentials before running make targets:
```bash
export $(grep -v '^#' .env | xargs)
```

**App stuck in ERROR state**: `dabs-deploy.sh` detects this and polls until deletion completes before redeploying. Run cleanup manually if needed:
```bash
ops_env=dev .ci/dabs-cleanup.sh
```

**Slack bot not starting**: Check logs for `Could not start Slack Socket Mode listener`. Most common causes:
- App service principal missing READ on secrets scope → run `databricks secrets put-acl`
- `slack_app_token` or `slack_bot_token` not stored → run `databricks secrets put-secret`

**PR app not cleaned up**: Run `dabs-pr-cleanup.sh` manually — it extracts the PR number from the last merge commit and deletes `dpe-knowledge-assistant-pr-<N>`.

**Bundle validation fails locally**:
```bash
databricks bundle validate --target dev --var app_suffix=""
```

## Documentation

- [CI/CD Guide](.ci/README.md)
- [Databricks Apps Documentation](https://docs.databricks.com/applications/index.html)
- [Databricks Asset Bundles](https://docs.databricks.com/dev-tools/bundles/index.html)
- [Databricks Apps Cookbook](https://apps-cookbook.dev/)

# CLAUDE.md — C-PAT (Crane POA&M Automation Tool)

## What This Is

C-PAT is a forked Navy open-source tool (NSWC Crane) for managing Plan of Action & Milestones (POA&Ms) — the RMF "Assess" step output. We're deploying it as an internal compliance tracking tool for Ask Sage STIG/SRG findings.

**Upstream:** `NSWC-Crane/C-PAT` | **Fork:** `Ask-Sage/C-PAT`

## Architecture

```
Angular 21 (PrimeNG) ──→ Express.js API (Node.js) ──→ MySQL (Sequelize ORM)
         ↑                        ↑
    OIDC Auth              STIG Manager / Tenable.sc integrations
  (Entra ID)
```

- **Client:** Angular 21 + PrimeNG, lives in `client/`
- **API:** Express.js, lives in `api/`
- **Database:** MySQL via Sequelize ORM (dialect configurable via `CPAT_DB_DIALECT`)
- **Auth:** OIDC (Keycloak default, swapped to Entra ID for our deployment)
- **Docs:** Sphinx, lives in `docs/`
- **Port:** 8086

## Key Directories

```
api/
  Controllers/       # Route handlers
  Models/            # Sequelize models (poam, poamAsset, poamMilestones, etc.)
  Services/          # Business logic (poamService, poamApproverService, etc.)
  Services/migrations/  # Database migrations
  bootstrap/         # App startup (middlewares, signals, server)
  utils/             # Config, logger, state management
  specification/     # OpenAPI spec (C-PAT.yaml)

client/
  src/app/pages/
    poam-processing/    # Core POA&M CRUD, approval, extension, milestones
    import-processing/  # STIG Manager + Tenable import
    admin-processing/   # User mgmt, collections, app config
    asset-processing/   # Asset management
    label-processing/   # Label/tag management
    metrics-processing/ # Dashboard metrics
    marketplace/        # Extension marketplace
```

## Configuration

All config is environment variable driven — see `api/utils/config.js` for the full list.

### Critical Env Vars for Ask Sage Deployment

```bash
# Auth — Entra ID OIDC (replaces Keycloak)
CPAT_OIDC_PROVIDER=https://login.microsoftonline.us/<tenant-id>/v2.0
CPAT_OIDC_CLIENT_ID=<entra-app-client-id>
CPAT_JWT_AUD_VALUE=<entra-app-client-id>
CPAT_JWT_USERNAME_CLAIM=preferred_username
CPAT_JWT_NAME_CLAIM=name

# Database — Azure MySQL Flexible Server
CPAT_DB_DIALECT=mysql
CPAT_DB_HOST=<server>.mysql.database.azure.com
CPAT_DB_PORT=3306
CPAT_DB_USER=<username>
CPAT_DB_PASSWORD=<password>
CPAT_DB_SCHEMA=cpat
CPAT_DB_TLS_CA_FILE=/certs/DigiCertGlobalRootCA.crt.pem

# App Settings
CPAT_CLASSIFICATION=U
CPAT_DOD_DEPLOYMENT=true
CPAT_API_PORT=8086
```

### Optional Integrations

```bash
# STIG Manager (if deployed alongside)
STIGMAN_API_URL=http://<stigman-host>:54000/api
STIGMAN_OIDC_CLIENT_ID=stig-manager

# Tenable.sc / ACAS
TENABLE_ENABLED=true
TENABLE_URL=https://<tenable-host>
TENABLE_ACCESS_KEY=<key>
TENABLE_SECRET_KEY=<key>
```

## Deployment

### CI/CD Pipeline
- **Workflow:** `.github/workflows/deploy.yml`
- **Pattern:** GitHub Actions → ACR → Azure Web App (same as AskSage-LLM-Manager)
- **Secrets needed:** `AZURE_ACR_LOGIN_SERVER`, `AZURE_ACR_USERNAME`, `AZURE_ACR_PASSWORD`

### Docker
```bash
# Build locally
docker build -t cpat .

# Run locally (needs MySQL)
docker run -p 8086:8086 --env-file .env cpat
```

### Database
Sequelize auto-migrates on startup. First run creates all tables. Migrations in `api/Services/migrations/`.

## Development

```bash
# API
cd api && npm install && npm start

# Client (separate terminal)
cd client && npm install && npm start
# Angular dev server runs on port 4200, proxies API calls to 8086
```

## Key Features

- **POA&M lifecycle:** Create → Assign → Milestones → Approve → Close
- **STIG Manager integration:** Import STIG findings directly into POA&Ms
- **Tenable.sc integration:** Import vulnerability scan results
- **AI mitigation generator:** Uses AI SDK (Anthropic, OpenAI, Google, etc.) to generate remediation text
- **Approval workflows:** Multi-level POA&M approval with audit trail
- **Dashboards:** Charts by status, severity, scheduled completion, labels
- **VRAM import:** Vulnerability remediation asset management
- **Export:** POA&M export with status filtering

## Fork Management

- **Ask Sage-specific changes** (Entra ID, Azure deployment, CI/CD) stay in our fork
- **Bug fixes and improvements** should be PR'd upstream to `NSWC-Crane/C-PAT`
- **Sync upstream** periodically: `git fetch upstream && git merge upstream/main`
- **Don't diverge unnecessarily** — minimize fork-specific code changes

## What NOT to Do

- Don't hardcode Azure credentials or secrets — use env vars
- Don't modify the Sequelize models without understanding migration impact
- Don't remove the Keycloak config defaults — they're needed for upstream compatibility
- Don't skip the docs build step in Docker (Dockerfile expects `docs/_build/html/`)

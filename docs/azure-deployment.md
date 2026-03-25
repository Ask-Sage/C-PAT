# C-PAT Azure Deployment Guide

## Architecture

C-PAT and STIG Manager are deployed as containerized applications on Azure Government using Azure Container Apps:

```
GitHub Actions → Azure Container Registry (ACR) → Azure Container Apps
                                                       ↓
                                              Azure Database for MySQL
                                              Flexible Server (shared)
```

Both apps share the same MySQL server (separate schemas) and Entra ID tenant for authentication.

## Azure Resources

### Resource Group: `asksage-cpat-rg`

| Resource | Name | SKU/Details |
|----------|------|-------------|
| ACR | `asksagecpat.azurecr.us` | Basic, admin enabled |
| MySQL | `asksage-cpat-mysql` | Burstable B1ms, v8.4 |
| Container Apps Env | `cpat-env` | usgovvirginia |
| C-PAT | `cpat` container app | 0.5 CPU, 1Gi, port 8086 |
| STIG Manager | `stigman` container app | 0.5 CPU, 1Gi, port 54000 |

### Entra ID App Registrations (Commercial tenant: asksage.ai)

| App | Client ID | Purpose |
|-----|-----------|---------|
| C-PAT | `d16e6e88-8eac-4e7d-bb2a-d2218065680f` | Primary OIDC for C-PAT |
| C-PAT-StigManager | `e4f84457-4816-464f-9290-e7089ace2961` | OIDC for STIG Manager |

## Environment Variables

### C-PAT Container App

#### Required
| Variable | Description | Example |
|---|---|---|
| `CPAT_OIDC_PROVIDER` | Entra ID OIDC authority URL | `https://login.microsoftonline.com/<tenant-id>/v2.0` |
| `CPAT_OIDC_CLIENT_ID` | Entra ID app client ID | `d16e6e88-...` |
| `CPAT_JWT_AUD_VALUE` | Expected JWT audience (use `api://<client-id>` for Entra) | `api://d16e6e88-...` |
| `CPAT_JWT_SCOPE_CLAIM` | JWT scope claim name (`scp` for Entra, `scope` for Keycloak) | `scp` |
| `CPAT_JWT_PRIVILEGES_CLAIM` | JWT privileges claim (`roles` for Entra, `realm_access.roles` for Keycloak) | `roles` |
| `CPAT_JWT_ASSERTION_CLAIM` | JWT assertion claim (`uti` for Entra, `jti` for Keycloak) | `uti` |
| `CPAT_JWT_USERNAME_CLAIM` | JWT username claim | `preferred_username` |
| `CPAT_JWT_NAME_CLAIM` | JWT display name claim | `name` |
| `CPAT_DB_HOST` | MySQL hostname | `<server>.mysql.database.usgovcloudapi.net` |
| `CPAT_DB_PORT` | MySQL port | `3306` |
| `CPAT_DB_USER` | Database username | `cpatadmin` |
| `CPAT_DB_PASSWORD` | Database password | `<from-key-vault>` |
| `CPAT_DB_SCHEMA` | Database name | `cpat` |
| `CPAT_DB_DIALECT` | Database dialect | `mysql` |

#### STIG Manager Integration
| Variable | Description | Example |
|---|---|---|
| `STIGMAN_API_URL` | STIG Manager API URL | `https://stigman.<env>.azurecontainerapps.us/api` |
| `STIGMAN_OIDC_CLIENT_ID` | STIG Manager OIDC client ID (set same as C-PAT to use single auth) | `d16e6e88-...` |
| `STIGMAN_SCOPE_PREFIX` | Scope prefix for STIG Manager (must match C-PAT when using same client) | `api://d16e6e88-.../` |
| `CPAT_SCOPE_PREFIX` | Scope prefix for C-PAT | `api://d16e6e88-.../` |

#### Optional
| Variable | Description | Default |
|---|---|---|
| `CPAT_CLASSIFICATION` | Classification banner | `U` |
| `CPAT_DOD_DEPLOYMENT` | Enable DoD features | `true` |
| `CPAT_API_PORT` | API port | `8086` |
| `CPAT_LOG_LEVEL` | Log level (1-4) | `3` |

### STIG Manager Container App

| Variable | Description | Example |
|---|---|---|
| `STIGMAN_OIDC_PROVIDER` | Entra ID OIDC authority URL | `https://login.microsoftonline.com/<tenant-id>/v2.0` |
| `STIGMAN_CLIENT_ID` | Entra ID app client ID | `e4f84457-...` |
| `STIGMAN_JWT_AUD_VALUE` | Expected JWT audience (set to C-PAT's `api://` URI for cross-app tokens) | `api://d16e6e88-...` |
| `STIGMAN_JWT_SCOPE_CLAIM` | JWT scope claim (`scp` for Entra) | `scp` |
| `STIGMAN_JWT_PRIVILEGES_CLAIM` | JWT privileges claim (`roles` for Entra) | `roles` |
| `STIGMAN_JWT_ASSERTION_CLAIM` | JWT assertion claim (`uti` for Entra) | `uti` |
| `STIGMAN_JWT_USERNAME_CLAIM` | JWT username claim | `preferred_username` |
| `STIGMAN_CLIENT_SCOPE_PREFIX` | Scope prefix | `api://e4f84457-.../` |
| `STIGMAN_CLIENT_STRICT_PKCE` | Disable strict PKCE check (Entra doesn't advertise it) | `false` |
| `STIGMAN_DB_HOST` | MySQL hostname | `<server>.mysql.database.usgovcloudapi.net` |
| `STIGMAN_DB_PORT` | MySQL port | `3306` |
| `STIGMAN_DB_USER` | Database username | `cpatadmin` |
| `STIGMAN_DB_PASSWORD` | Database password | `<from-key-vault>` |
| `STIGMAN_DB_SCHEMA` | Database name | `stigman` |
| `STIGMAN_CLASSIFICATION` | Classification banner | `U` |
| `STIGMAN_LOG_LEVEL` | Log level (1-4) | `3` |

## Entra ID Configuration

### C-PAT App Registration
1. Create app registration (Single tenant, SPA platform)
2. Add SPA redirect URIs: app URL and `/silent-renew.html`
3. Set identifier URI: `api://<client-id>`
4. Add API scopes: `c-pat:read`, `c-pat:write`, `c-pat:op`, plus STIG Manager scopes
5. Add optional access token claims: `preferred_username`, `email`
6. Create app roles: `admin`, `user`
7. Pre-authorize the app for its own scopes
8. Grant admin consent

### STIG Manager App Registration
1. Create app registration (Single tenant, SPA platform)
2. Add SPA redirect URIs: STIG Manager URL, C-PAT URL, and `/reauth.html`
3. Set identifier URI: `api://<client-id>`
4. Add API scopes: `stig-manager:stig`, `stig-manager:stig:read`, `stig-manager:collection`, etc.
5. Add optional access token claims: `preferred_username`, `email`
6. Create app roles: `admin`, `user`
7. Pre-authorize the app for its own scopes
8. Grant admin consent

### Key Entra ID Differences from Keycloak
| Setting | Keycloak | Entra ID |
|---------|----------|----------|
| Scope claim | `scope` | `scp` |
| Assertion claim | `jti` | `uti` |
| Privileges claim | `realm_access.roles` | `roles` |
| Audience format | Client ID string | `api://<client-id>` |
| Scope prefix | (none) | `api://<client-id>/` |
| PKCE advertisement | Advertised in discovery | Not advertised (set `STIGMAN_CLIENT_STRICT_PKCE=false`) |

### Single OIDC Config Mode (Recommended for Entra ID)
C-PAT's dual OIDC configuration (separate `cpat` and `stigman` configs) causes issues with Entra ID because PKCE state is lost between sequential redirects. The workaround:

1. Set `STIGMAN_OIDC_CLIENT_ID` to the **same value** as `CPAT_OIDC_CLIENT_ID`
2. Set `STIGMAN_SCOPE_PREFIX` to the **same value** as `CPAT_SCOPE_PREFIX`
3. Set STIG Manager's `STIGMAN_JWT_AUD_VALUE` to C-PAT's `api://<client-id>` URI

This makes both OIDC configs share the same token, and C-PAT's client code detects the matching client IDs and handles the auth flow with a single redirect.

## Database Setup

Both apps use automatic schema migration on startup.

### MySQL Flexible Server
1. Create server (Burstable B1ms, MySQL 8.4)
2. Create databases: `cpat` and `stigman`
3. Enable `event_scheduler` via Azure CLI (C-PAT migration 0008 requires it):
   ```bash
   az mysql flexible-server parameter set --name event_scheduler --value ON
   ```
4. Widen `user.userName` column if using long email addresses:
   ```sql
   ALTER TABLE cpat.user MODIFY userName varchar(100) NOT NULL;
   ```

### Known Migration Issues
- **Migration 0006**: Initial schema includes `collectionId` on `assetdeltalist` but migration tries to add it again. Fix: drop the column before migration runs, or mark 0006 as complete.
- **Migration 0008**: Uses `SET GLOBAL event_scheduler = ON` which requires SUPER privilege. Azure MySQL doesn't allow this. Fix: set via Azure CLI and mark 0008 as complete.

## CI/CD

### GitHub Secrets
- `AZURE_ACR_LOGIN_SERVER` — ACR login server URL
- `AZURE_ACR_USERNAME` — ACR admin username
- `AZURE_ACR_PASSWORD` — ACR admin password

### Deployment
Push to `main` triggers `.github/workflows/deploy.yml` which builds and pushes to ACR.

# C-PAT Azure Deployment Guide

## Architecture

C-PAT is deployed as a containerized application on Azure using the same pattern as AskSage-LLM-Manager:

```
GitHub Actions → Azure Container Registry (ACR) → Azure Web App (Container)
                                                       ↓
                                              Azure Database for MySQL
                                              Flexible Server
```

## Azure Resources Required

### 1. Azure Container Registry (ACR)
- **Existing:** Use the same ACR as LLM Manager
- Image: `<acr-login-server>/cpat:latest`

### 2. Azure Web App (Container)
- **SKU:** B1 or S1 (minimal resource requirements)
- **Runtime:** Linux container from ACR
- **Port:** 8086

### 3. Azure Database for MySQL Flexible Server
- **SKU:** Burstable B1ms (1 vCore, 2 GiB RAM) — sufficient for POA&M tracking
- **Version:** 8.0
- **Storage:** 20 GiB (auto-grow enabled)
- **SSL:** Required (TLS 1.2+)

### 4. Entra ID App Registration (OIDC)
- **Name:** `cpat-web`
- **Type:** Single tenant
- **Redirect URI:** `https://<web-app-name>.azurewebsites.net`
- **Scopes:** `openid`, `profile`, `email`

## Environment Variables

### Required
| Variable | Description | Example |
|---|---|---|
| `CPAT_OIDC_PROVIDER` | Entra ID OIDC authority URL | `https://login.microsoftonline.us/<tenant-id>/v2.0` |
| `CPAT_OIDC_CLIENT_ID` | Entra ID app registration client ID | `<app-client-id>` |
| `CPAT_DB_HOST` | MySQL Flexible Server hostname | `cpat-db.mysql.database.azure.com` |
| `CPAT_DB_PORT` | MySQL port | `3306` |
| `CPAT_DB_USER` | Database username | `cpatadmin` |
| `CPAT_DB_PASSWORD` | Database password | `<from-key-vault>` |
| `CPAT_DB_SCHEMA` | Database name | `cpat` |
| `CPAT_DB_DIALECT` | Database dialect | `mysql` |

### Optional
| Variable | Description | Default |
|---|---|---|
| `CPAT_CLASSIFICATION` | Classification banner | `U` |
| `CPAT_DOD_DEPLOYMENT` | Enable DoD-specific features | `true` |
| `CPAT_API_PORT` | API port | `8086` |
| `CPAT_JWT_USERNAME_CLAIM` | JWT claim for username | `preferred_username` |
| `CPAT_JWT_NAME_CLAIM` | JWT claim for display name | `name` |
| `CPAT_JWT_FIRST_NAME_CLAIM` | JWT claim for first name | `given_name` |
| `CPAT_JWT_AUD_VALUE` | Expected JWT audience | `<app-client-id>` |
| `CPAT_DB_TLS_CA_FILE` | Path to MySQL CA cert | `/certs/DigiCertGlobalRootCA.crt.pem` |
| `CPAT_INACTIVITY_TIMEOUT` | Session timeout (minutes) | `15` |
| `STIGMAN_API_URL` | STIG Manager API URL (if integrated) | `http://localhost:54000/api` |
| `TENABLE_ENABLED` | Enable Tenable.sc integration | `false` |

### GitHub Secrets (CI/CD)
Same ACR secrets as LLM Manager:
- `AZURE_ACR_LOGIN_SERVER`
- `AZURE_ACR_USERNAME`
- `AZURE_ACR_PASSWORD`

## Entra ID OIDC Configuration

C-PAT uses standard OIDC with JWKS validation. To configure for Entra ID:

1. Create App Registration in Azure portal
2. Set redirect URI to the Web App URL
3. Enable ID tokens under Authentication
4. Add optional claims: `preferred_username`, `name`, `given_name`, `email`
5. Set `CPAT_OIDC_PROVIDER` to `https://login.microsoftonline.us/<tenant-id>/v2.0` (Gov) or `https://login.microsoftonline.com/<tenant-id>/v2.0` (Commercial)
6. Set `CPAT_OIDC_CLIENT_ID` to the app's client ID
7. Set `CPAT_JWT_AUD_VALUE` to the app's client ID (for audience validation)

## Database Setup

The application uses Sequelize ORM with automatic migrations. On first startup, it will create the schema and tables automatically.

For Azure MySQL Flexible Server:
1. Create the server with SSL enforcement enabled
2. Create a database named `cpat`
3. Download the DigiCert Global Root CA certificate for SSL connections
4. Set `CPAT_DB_TLS_CA_FILE` to the cert path

## Deployment Steps

1. **Fork done** ✅ — `Ask-Sage/C-PAT`
2. **CI/CD workflow** ✅ — `.github/workflows/deploy.yml`
3. **Add ACR secrets** to the repo (same values as LLM Manager)
4. **Create Azure resources:**
   - MySQL Flexible Server (Burstable B1ms)
   - Web App for Containers (B1/S1, Linux)
   - Entra ID App Registration
5. **Configure Web App** environment variables
6. **Push to main** → GitHub Actions builds and pushes to ACR → Web App pulls the image
7. **First startup** → Sequelize auto-migrates the database schema

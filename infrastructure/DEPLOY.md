# KayCare HMS — Azure Deployment Guide

## Prerequisites

Install the Azure CLI:
https://learn.microsoft.com/en-us/cli/azure/install-azure-cli

## One-Time Setup

### 1. Login to Azure
```bash
az login
```

### 2. Create a Resource Group
```bash
az group create \
  --name kaycare-rg \
  --location southafricanorth
```

### 3. Generate a JWT Signing Key (run once, save the output securely)
```bash
# PowerShell
[System.Convert]::ToBase64String((1..64 | ForEach-Object { Get-Random -Maximum 256 }) -as [byte[]])
```

### 4. Deploy All Azure Resources
```bash
az deployment group create \
  --resource-group kaycare-rg \
  --template-file infrastructure/bicep/main.bicep \
  --parameters infrastructure/bicep/parameters.bicepparam \
               sqlAdminPassword="<YourStrongSQLPassword>" \
               jwtKey="<YourJwtSigningKeyFromStep3>"
```

This provisions:
- Key Vault (stores all secrets)
- SQL Server + Basic Database (~$5/month)
- Blob Storage — LRS (~$1/month)
- App Service Plan B1 + API Web App (~$13/month)
- Static Web App — Free ($0/month)

### 5. Update the Blob Storage Connection String in Key Vault

After deployment, run this to store the real blob connection string:

```bash
# Get storage account name
STORAGE_NAME="kaycareprodstorstor"   # as printed in deployment output

# Get the connection string
BLOB_CONN=$(az storage account show-connection-string \
  --name $STORAGE_NAME \
  --resource-group kaycare-rg \
  --query connectionString -o tsv)

# Store it in Key Vault
az keyvault secret set \
  --vault-name kaycare-prod-kv \
  --name BlobStorageConnection \
  --value "$BLOB_CONN"
```

### 6. Run Database Migrations

After deployment, point the EF CLI at Azure SQL:

```bash
# Set the Azure connection string temporarily
export ConnectionStrings__DefaultConnection="Server=tcp:<sql-server-fqdn>,1433;Initial Catalog=KayCareDb;User ID=kaycare_admin;Password=<YourPassword>;Encrypt=True;"

dotnet ef database update \
  --project src/KayCare.Infrastructure \
  --startup-project src/KayCare.API
```

### 7. Get Static Web App Deploy Token (for GitHub Actions)

```bash
az staticwebapp secrets list \
  --name kaycare-prod-web \
  --resource-group kaycare-rg \
  --query "properties.apiKey" -o tsv
```

Save this token as a GitHub secret named `AZURE_STATIC_WEB_APPS_API_TOKEN`.

---

## Deployment Outputs

After `az deployment group create` completes, the CLI prints:

| Output | Value |
|---|---|
| `apiUrl` | `https://kaycare-prod-api.azurewebsites.net` |
| `staticWebUrl` | `https://kaycare-prod-web.azurestaticapps.net` |
| `keyVaultName` | `kaycare-prod-kv` |
| `sqlServerFqdn` | `kaycare-prod-sql.database.windows.net` |

---

## Monthly Cost Estimate

| Resource | Tier | Cost |
|---|---|---|
| App Service Plan B1 | Basic | ~$13 |
| Azure SQL Basic | 5 DTU, 2GB | ~$5 |
| Blob Storage LRS | Pay-as-you-go | ~$1 |
| Key Vault | Standard | ~$0.50 |
| Static Web Apps | Free | $0 |
| **Total** | | **~$20/month** |

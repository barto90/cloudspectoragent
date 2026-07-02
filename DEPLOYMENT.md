# CloudSpector Agent — Azure Container Instance Deployment

Deploy the CloudSpector Agent to your own Azure subscription using Azure Container Instances with persistent Azure Files storage and a dedicated Azure SQL Database.

> **Prerequisite:** You must receive an ACR token (username + password) from 7NodeIT before you can deploy. Contact support@7nodeit.com to request one.

---

## What Gets Deployed

One ARM template deploys these resources into your chosen resource group:

| Resource | Type | Purpose |
|---|---|---|
| Storage Account | Standard_LRS | Hosts the Azure Files share |
| File Share (`cloudspector-data`) | Azure Files | Persistent storage for provider packages |
| Container Group | Azure Container Instance | Runs the CloudSpector Agent on port 8080, with a system-assigned managed identity |
| SQL Server + Database | Microsoft.Sql (Basic tier) | Agent's local database (baselines, design decisions, users, feed items, check metadata) |

Estimated monthly cost (West Europe): **€36–51/month**
- ACI (1 vCPU / 1.5 GB, always running): ~€30–44/month
- Storage (Standard_LRS, 32 GB share): ~€1–2/month
- Azure SQL Database (Basic tier): ~€5/month

**No SQL password anywhere.** The SQL Server uses Azure AD-only authentication — the Container Group's system-assigned managed identity is set as the server's sole administrator during deployment. There is no SQL login, no password to store in Key Vault, and nothing to rotate. See ADR-043 in the CloudSpector repo for the reasoning.

---

## Prerequisites

Complete these steps before deploying.

### 1. Azure Subscription

You need an active Azure subscription. If you don't have one, create one at [portal.azure.com](https://portal.azure.com).

### 2. Resource Group

Create or choose a resource group for the deployment. All resources will be deployed into this group.

### 3. Azure Key Vault

Create a Key Vault in your subscription (or use an existing one) and enable it for template deployment:

**In the Azure Portal:**
1. Open your Key Vault → **Settings** → **Properties**
2. If your vault uses the **legacy Access Policy** model: enable **Azure Resource Manager for template deployment** and add an access policy for your user with `Get` permission on Secrets
3. If your vault uses the **RBAC** model: assign yourself the **Key Vault Secrets User** role on the vault — no extra property needed

### 4. Create Key Vault Secrets

Create exactly these two secrets in your Key Vault:

| Secret name | Value |
|---|---|
| `cloudspector-acr-token-password` | The ACR token password provided by 7NodeIT |
| `cloudspector-auth-pepper` | A random string of at least 32 characters (generate one at [random.org](https://www.random.org/strings/)) |

> **Important:** Store the `authPepper` value somewhere safe. If it is lost, existing user passwords cannot be verified and all agent accounts must be recreated.

**No SQL Key Vault secret is needed.** Database access is granted automatically via the Container Group's managed identity — there is nothing to create for it.

---

## Deployment — Azure Portal

### Step 1: Open the Custom Template Deployment

1. Go to [portal.azure.com](https://portal.azure.com)
2. In the search bar, type **Deploy a custom template** and select it
3. Click **Build your own template in the editor**

### Step 2: Upload the Template

1. Click **Load file**
2. Select `azuredeploy.json` from this folder
3. Click **Save**

### Step 3: Fill In the Parameters

You will see a form with the following fields:

| Field | What to enter |
|---|---|
| **Subscription** | Your Azure subscription |
| **Resource group** | Select existing or create new |
| **Region** | Your preferred Azure region (e.g. West Europe) |
| **Container Group Name** | A name for the container, e.g. `cloudspector-agent` |
| **Storage Account Name** | A globally unique name, 3–24 lowercase letters/numbers, e.g. `csagentcontoso001` |
| **File Share Name** | Leave as `cloudspector-data` unless you have a reason to change it |
| **Agent Image Name** | Leave as default (`7nodeit.azurecr.io/cloudspector.agent:latest`) |
| **Acr Token Username** | The ACR token name provided by 7NodeIT (not the password) |
| **Acr Token Password** | Click the Key Vault icon → select your vault → select `cloudspector-acr-token-password` |
| **Auth Pepper** | Click the Key Vault icon → select your vault → select `cloudspector-auth-pepper` |
| **Cloud Spector Api Base Url** | Leave as default |
| **Sql Server Name** | A globally unique name, e.g. `csagent-sql-contoso001` |
| **Sql Database Name** | Leave as `cloudspector-agent-db` unless you have a reason to change it |
| **Sql Database Sku Name** | Leave as `Basic` unless you need more performance (S0/S1) |

### Step 4: Deploy

1. Click **Review + Create**
2. Wait for validation to pass (green tick)
3. Click **Create**
4. Deployment takes approximately 2–3 minutes

### Step 5: Get the Agent URL

1. When deployment completes, click **Go to resource group** → then click the **Outputs** tab on the deployment
2. Copy the value of `agentUrl` (format: `http://<ip>:8080`)

---

## Post-Deployment: First-Run Setup Wizard

1. Open the `agentUrl` in your browser
2. The setup wizard will appear (this only runs once on a fresh installation)
3. **Step 1 — Authentication:** Create your admin account (username + password)
4. **Step 2 — License:** Upload the `.lic` file from 7NodeIT, or skip and upload later via Settings
5. **Step 3 — Connectivity Mode:** Choose **Online** (agent syncs with CloudSpector API) or **Offline** (fully air-gapped)
6. Click **Complete Setup**

The agent is now operational.

---

## Deployment — Azure CLI (Alternative)

If you prefer the command line, edit `azuredeploy.parameters.json` first:
- Replace all `<placeholder>` values with your actual values
- Replace the Key Vault `id` field with your Key Vault's resource ID (find it in the portal under Key Vault → Properties → Resource ID)

Then run:

```bash
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file azuredeploy.json \
  --parameters azuredeploy.parameters.json
```

---

## Parameters Reference

| Parameter | Type | Default | Description |
|---|---|---|---|
| `containerGroupName` | string | — | Name for the ACI resource |
| `location` | string | Resource group location | Azure region |
| `storageAccountName` | string | — | Globally unique storage account name (3–24 chars, lowercase alphanumeric) |
| `fileShareName` | string | `cloudspector-data` | Azure Files share name |
| `agentImageName` | string | `7nodeit.azurecr.io/cloudspector.agent:latest` | Container image (do not change unless told to) |
| `acrTokenUsername` | string | — | ACR token name from 7NodeIT |
| `acrTokenPassword` | securestring | — | ACR token password (from Key Vault) |
| `authPepper` | securestring | — | Password hashing secret (from Key Vault, min 32 chars) |
| `cloudSpectorApiBaseUrl` | string | Production URL | Backend API URL (do not change) |
| `sqlServerName` | string | — | Globally unique Azure SQL logical server name (lowercase, numbers, hyphens) |
| `sqlDatabaseName` | string | `cloudspector-agent-db` | Azure SQL Database name |
| `sqlDatabaseSkuName` | string | `Basic` | Azure SQL Database pricing tier (`Basic`, `S0`, or `S1`) |

---

## Troubleshooting

### Container repeatedly restarts right after deployment

This is expected for the first 1–2 minutes: the Container Group starts before the SQL Server/Database finish provisioning (this is intentional — see the `comments` field on the `Microsoft.Sql/servers` resource in `azuredeploy.json`). The container's `restartPolicy` is `Always`, so it retries automatically until the database is reachable. Give it a couple of minutes before troubleshooting further.

### Container is in a persistent restart loop (beyond a few minutes)

Check the container logs in the Azure Portal: Container Instance → **Containers** → **Logs**.

Common causes:
- **Database connection failing:** Verify the `sqlServerName` / `sqlDatabaseName` parameters match what was actually deployed. The agent connects via `ConnectionStrings__CloudSpector`, which is built automatically from those parameters — you should never need to set it manually.
- **Managed identity not recognized as SQL admin:** Confirm the Container Group has a system-assigned identity (Container Instance → **Identity** tab → should show "On"). If it's missing, the deployment likely failed partway through — redeploy.
- **Wrong `Auth__Pepper`:** The env var name is case-sensitive — must be exactly `Auth__Pepper`.

### Cannot connect to the SQL Database with SSMS / Azure Data Studio

The SQL Server uses **Azure AD-only authentication** — SQL logins/passwords do not work here, by design (see ADR-043). To connect manually for troubleshooting:
1. Azure Portal → your SQL Server → **Microsoft Entra ID** (formerly Azure AD)
2. Add yourself (or a group you're in) as an additional Microsoft Entra admin
3. Connect using "Microsoft Entra Account" / "Active Directory - Universal with MFA" authentication in SSMS/Azure Data Studio — not "SQL Server Authentication"

### Azure Files share fails to mount

The container will fail to start if the volume cannot be mounted.

- Verify the storage account name in the parameters exactly matches what was deployed
- Check that the file share exists: Storage Account → **File shares**
- Verify the storage account is in the same region as the container group

### ACR pull error (image not found / unauthorized)

- Confirm the ACR token username and password are correct
- Verify the token has `pull` permission on the `cloudspector.agent` repository
- Contact 7NodeIT to get a new token if the current one has expired

### Finding the exact agent version for support

```bash
az container show \
  --name <containerGroupName> \
  --resource-group <resource-group> \
  --query "containers[0].image" \
  --output tsv
```

Include this image digest when contacting 7NodeIT support.

---

## Updating the Agent

To pull the latest image without changing any settings:

1. Azure Portal → Container Instances → your container group
2. Click **Restart**

The container will pull `latest` from ACR on restart. Your data in `/data` (provider packages, license) and in the Azure SQL Database (baselines, users, check metadata) is unaffected — neither is stored inside the container itself.

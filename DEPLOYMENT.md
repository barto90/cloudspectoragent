# CloudSpector Agent — Azure Container Instance Deployment

Deploy the CloudSpector Agent to your own Azure subscription using Azure Container Instances with persistent Azure Files storage.

> **Prerequisite:** You must receive an ACR token (username + password) from 7NodeIT before you can deploy. Contact support@7nodeit.com to request one.

---

## What Gets Deployed

One ARM template deploys three resources into your chosen resource group:

| Resource | Type | Purpose |
|---|---|---|
| Storage Account | Standard_LRS | Hosts the Azure Files share |
| File Share (`cloudspector-data`) | Azure Files | Persistent storage for database and provider packages |
| Container Group | Azure Container Instance | Runs the CloudSpector Agent on port 8080 |

Estimated monthly cost (West Europe): **€31–46/month**
- ACI (1 vCPU / 1.5 GB, always running): ~€30–44/month
- Storage (Standard_LRS, 32 GB share): ~€1–2/month

---

## Prerequisites

Complete these steps before deploying.

### 1. Azure Subscription

You need an active Azure subscription. If you don't have one, create one at [portal.azure.com](https://portal.azure.com).

### 2. Resource Group

Create or choose a resource group for the deployment. All three resources will be deployed into this group.

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

### Step 4: Deploy

1. Click **Review + Create**
2. Wait for validation to pass (green tick)
3. Click **Create**
4. Deployment takes approximately 2 minutes

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

---

## Troubleshooting

### Container is in a restart loop

Check the container logs in the Azure Portal: Container Instance → **Containers** → **Logs**.

Common causes:
- **Missing `CLOUDSPECTOR_SQL_CONNECTION`:** The SQLite migration must be complete in the deployed image. Contact 7NodeIT if you see `InvalidOperationException: No database connection string found`.
- **Wrong `Auth__Pepper`:** The env var name is case-sensitive — must be exactly `Auth__Pepper`.

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

The container will pull `latest` from ACR on restart. Your data in `/data` (database, providers, license) is unaffected — it is stored in Azure Files, not in the container.

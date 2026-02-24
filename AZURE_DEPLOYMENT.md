# Azure Container Apps Deployment Guide

Complete step-by-step guide to deploy the Sentiment Analysis backend and frontend to Azure Container Apps using GitHub Actions.

---

## 📋 Prerequisites

- Azure Account with active subscription
- GitHub Repository with this code
- Azure CLI (install via `winget install Microsoft.AzureCLI` on Windows)

---

## 🛠️ Step 1: Azure Setup (Run in Azure Cloud Shell or Terminal)

### 1.1 Login to Azure
```bash
az login
```
This opens a browser window - sign in with your Azure account.

### 1.2 Get Your Subscription ID
```bash
az account show --query id -o tsv
```
**Save this output** - you'll need it for step 1.4.

### 1.3 Create Resource Group
```bash
az group create --name sentiment-rg --location centralindia
```
> Note: Use a location allowed by your subscription (e.g., `centralindia`, `eastus`, `southeastasia`)

### 1.4 Create Service Principal (for AZURE_CREDENTIALS)
Replace `YOUR-SUBSCRIPTION-ID` with the ID from step 1.2:
```bash
az ad sp create-for-rbac --name "github-actions" --role contributor --scopes /subscriptions/YOUR-SUBSCRIPTION-ID/resourceGroups/sentiment-rg --sdk-auth
```

**Save the entire JSON output** - this is your `AZURE_CREDENTIALS` secret:
```json
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  ...
}
```

### 1.5 Create Azure Container Registry
```bash
az acr create --resource-group sentiment-rg --name sentimentacr2026 --sku Basic --admin-enabled true
```
> Note: ACR name must be globally unique. Change `sentimentacr2026` if it's taken.

### 1.6 Get ACR Credentials
```bash
az acr credential show --name sentimentacr2026
```

**Save the output:**
- `username` → This is your `ACR_USERNAME` secret
- `passwords[0].value` → This is your `ACR_PASSWORD` secret

### 1.7 Create Container Apps Environment
```bash
# Register provider (only needed once)
az provider register -n Microsoft.OperationalInsights --wait

# Create environment
az containerapp env create --name sentiment-env --resource-group sentiment-rg --location centralindia
```
This takes 2-3 minutes.

---

## 🔐 Step 2: Add Secrets to GitHub

### 2.1 Navigate to Secrets Page
1. Go to your GitHub repository.
2. Click **Settings** (top menu)
3. Click **Secrets and variables** (left sidebar)
4. Click **Actions**
5. Click **New repository secret** (green button)

### 2.2 Add Each Secret

Add these **6 secrets** (one at a time):

| # | Secret Name | Value |
|---|-------------|-------|
| 1 | `AZURE_CREDENTIALS` | Paste the **entire JSON** from step 1.4 |
| 2 | `ACR_NAME` | `sentimentacr2026` |
| 3 | `ACR_USERNAME` | `sentimentacr2026` |
| 4 | `ACR_PASSWORD` | Password from step 1.6 |
| 5 | `AZURE_RESOURCE_GROUP` | `sentiment-rg` |
| 6 | `CONTAINER_APP_ENVIRONMENT` | `sentiment-env` |

**For each secret:**
1. Click **New repository secret**
2. Enter the **Name** exactly as shown
3. Paste the **Value**
4. Click **Add secret**

---

## 🚀 Step 3: Push Code and Deploy Backend

### 3.1 Commit and Push
```bash
git add .
git commit -m "Add Azure deployment workflows"
git push origin main
```

### 3.2 Manually Trigger Backend Deployment
1. Go to your GitHub repo
2. Click **Actions** tab
3. Click **Deploy Backend to Azure Container Apps** (left sidebar)
4. Click **Run workflow** dropdown (right side)
5. Click **Run workflow** button

### 3.3 Wait for Deployment
- Click on the running workflow to watch progress
- Takes ~5-10 minutes for first deployment
- Green checkmark = success

### 3.4 Get Backend URL
After successful deployment, run:
```bash
az containerapp show --name sentiment-backend --resource-group sentiment-rg --query properties.configuration.ingress.fqdn -o tsv
```

**Save this URL** (add `https://` prefix) - e.g., `https://sentiment-backend.redwave-90d5a156.centralindia.azurecontainerapps.io`

---

## 🎨 Step 4: Deploy Frontend

### 4.1 Add API_BASE_URL Secret
1. Go to GitHub → Settings → Secrets → Actions → **New repository secret**
2. Name: `API_BASE_URL`
3. Value: The backend URL from step 3.4 (with `https://`)
4. Click **Add secret**

### 4.2 Trigger Frontend Deployment
1. Go to **Actions** tab
2. Click **Deploy Frontend to Azure Container Apps**
3. Click **Run workflow** → **Run workflow**

### 4.3 Get Frontend URL
```bash
az containerapp show --name sentiment-frontend --resource-group sentiment-rg --query properties.configuration.ingress.fqdn -o tsv
```

**Your app is live at this URL!** 🎉

---

## 📊 Step 5: Monitor & Debug

### View Logs
```bash
# Backend logs
az containerapp logs show --name sentiment-backend --resource-group sentiment-rg --follow

# Frontend logs
az containerapp logs show --name sentiment-frontend --resource-group sentiment-rg --follow
```

### Check Container App Status
```bash
az containerapp show --name sentiment-backend --resource-group sentiment-rg --query properties.runningStatus
az containerapp show --name sentiment-frontend --resource-group sentiment-rg --query properties.runningStatus
```

### View in Azure Portal
1. Go to https://portal.azure.com
2. Search for "Container Apps"
3. Click on `sentiment-backend` or `sentiment-frontend`

---

## 🔄 Automatic Deployments

After initial setup, deployments happen automatically:

| Trigger | Action |
|---------|--------|
| Push to `backend/` folder | Backend redeploys |
| Push to `frontend/` folder | Frontend redeploys |
| Manual trigger in Actions | Deploy selected service |

---

## 🗑️ Cleanup (Delete All Resources)

To delete everything and stop billing:
```bash
az group delete --name sentiment-rg --yes --no-wait
```

---

## 📁 Files Reference

| File | Purpose |
|------|---------|
| `.github/workflows/deploy-backend.yml` | Backend CI/CD workflow |
| `.github/workflows/deploy-frontend.yml` | Frontend CI/CD workflow |
| `backend/Dockerfile` | Backend container config |
| `frontend/Dockerfile` | Frontend container config |

---

## ❓ Troubleshooting

### Error: "MissingSubscription"
- Using Git Bash? Add `MSYS_NO_PATHCONV=1` before the command
- Or use Azure Cloud Shell: https://shell.azure.com

### Error: "Resource provider not registered"
```bash
az provider register -n Microsoft.OperationalInsights --wait
az provider register -n Microsoft.App --wait
```

### Error: "ACR name already taken"
- Change `sentimentacr2026` to a unique name in step 1.5 and 1.6

### Deployment failing?
1. Check Actions tab for error logs
2. Verify all secrets are added correctly
3. Check spelling of secret names (case-sensitive)

# 🚀 SOCRadar-Azure Sentinel Bidirectional Integration v2.0

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Forcunsami%2Fazure-bidirectional-incident-app-fork-orcun%2Fmain%2Fsocradar-sentinel-integration-arm.json)

## ✨ What's New in v2.0

**MAJOR UPDATE:** Switched to **polling architecture** for fully automated deployment!

**Problems Solved:**
- ✅ No more "Missing required permissions" errors
- ✅ No manual portal configuration needed
- ✅ Fully automated one-click deployment
- ✅ Production-ready immediately

**What Changed:**
- 🔄 Trigger: `ApiConnectionWebhook` → `Recurrence` (polling every 5 minutes)
- 🔄 Connector: `azuresentinel` → `azuremonitorlogs`
- ❌ Removed: Automation Rule creation (not needed)
- ❌ Removed: Deployment script (not needed)

**Trade-off:** 5-10 minute delay after closing incident (previously instant) - acceptable for this workflow.

---

## 🎯 Quick Start

### Prerequisites

- Azure subscription with **Microsoft Sentinel enabled**
- Existing **Log Analytics Workspace** with Sentinel
- **SOCRadar API Key** and **Company ID**

### One-Click Deployment

**Option 1: Portal (Recommended)**

Click the **"Deploy to Azure"** button above, fill parameters, done!

**Option 2: Azure CLI**

```bash
az deployment group create \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --template-file socradar-sentinel-integration-arm.json \
  --parameters \
    WorkspaceName=<YOUR_WORKSPACE_NAME> \
    SocradarApiKey=<YOUR_API_KEY> \
    CompanyId=<YOUR_COMPANY_ID> \
    PollingIntervalMinutes=5
```

**Option 3: PowerShell**

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName <YOUR_RESOURCE_GROUP> `
  -TemplateFile socradar-sentinel-integration-arm.json `
  -WorkspaceName <YOUR_WORKSPACE_NAME> `
  -SocradarApiKey <YOUR_API_KEY> `
  -CompanyId <YOUR_COMPANY_ID> `
  -PollingIntervalMinutes 5
```

---

## 📋 Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| **WorkspaceName** | ✅ Yes | - | Log Analytics workspace **NAME** (not ID!) - e.g., `my-workspace-001` |
| **SocradarApiKey** | ✅ Yes | - | Your SOCRadar API authentication token |
| **CompanyId** | ✅ Yes | - | Your SOCRadar company ID |
| PlaybookName | No | SOCRadar-CloseAlarm-Polling | Name for the Logic App |
| PollingIntervalMinutes | No | 5 | Check frequency (1-60 minutes) |
| Location | No | Resource group location | Azure region |

---

## ⚠️ CRITICAL: Workspace Name vs Workspace ID

**COMMON MISTAKE:** Using Workspace **ID** instead of Workspace **NAME**

### ❌ WRONG (Workspace ID - GUID format):
```
cab033c7-927c-46d1-bb24-7af8090ae3fc
```

### ✅ CORRECT (Workspace Name - string format):
```
company-log-analytics-workspace-0001
```

### How to Find Workspace Name:
1. Azure Portal → **Log Analytics workspaces**
2. Click on your workspace
3. Copy the **Name** field (NOT the Workspace ID!)

**Example:**
- Name: `company-log-analytics-workspace-0001` ✅ Use this!
- Workspace ID: `cab033c7-927c-46d1-bb24-7af8090ae3fc` ❌ Don't use this!

---

## 🔄 How It Works

### Architecture

```
┌──────────────────────────────────────┐
│       SOCRadar Platform              │
│                                      │
│  Alarm: OPEN ──────────► RESOLVED   │
│         ▲                    ▲       │
└─────────┼────────────────────┼───────┘
          │ Webhook            │ API PATCH
          │ (Repo 1)           │ (This Repo)
┌─────────▼────────────────────┼───────┐
│    Azure Sentinel            │       │
│                              │       │
│  Incident: NEW ──────► CLOSED───┐    │
│                                │    │
│  ┌──────────────────────────┐  │    │
│  │  Logic App               │  │    │
│  │  (Runs every 5 min)      │  │    │
│  │                          │  │    │
│  │  1. Query: Status=Closed │──┘    │
│  │  2. Filter: SOCRadar-*   │       │
│  │  3. Extract Alarm ID     │       │
│  │  4. PATCH SOCRadar API   │───────┘
│  └──────────────────────────┘       │
└──────────────────────────────────────┘
```

### Workflow

1. **Every 5 minutes** (configurable), Logic App runs
2. **Queries Sentinel** for incidents:
   - Status = "Closed"
   - Title contains "SOCRadar-AlarmID-"
   - Modified in last 10 minutes
3. **For each matching incident:**
   - Extract alarm ID from title (e.g., "SOCRadar-AlarmID-12345" → 12345)
   - Call SOCRadar API: `POST /alarms/status/change` with alarm_ids and status: "2" (RESOLVED)
4. **SOCRadar alarm** automatically updated!

### Timing Example

- **14:00** - User closes incident in Sentinel
- **14:05** - Logic App polls, finds closed incident, updates SOCRadar
- **Total delay:** 5 minutes ✅ (acceptable for incident closure)

---

## 🧪 Testing

### Test 1: Verify Deployment

```bash
# Check Logic App status
az logic workflow show \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --query "state"

# Expected: "Enabled"
```

### Test 2: Manual Trigger

```bash
# Manually run Logic App
az logic workflow run trigger \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --trigger-name Recurrence
```

### Test 3: End-to-End

1. Create a test alarm in SOCRadar → Creates incident in Sentinel
2. Close the incident in Sentinel (Status: Closed)
3. Wait 5-10 minutes
4. Verify alarm status in SOCRadar:

```bash
curl -X GET \
  "https://platform.socradar.com/api/companies/<YOUR_COMPANY_ID>/alarms/<ALARM_ID>" \
  -H "Authorization: Token <YOUR_API_KEY>"
```

Expected: `"status": "RESOLVED"`

---

## 📊 Monitoring

### View Run History

**Portal:**
1. Go to Logic App → Overview → Runs history
2. Click on any run to see details

**CLI:**
```bash
az logic workflow run list \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --top 10 \
  --query "[].{name:name, status:status, startTime:startTime}" \
  --output table
```

### Check for Errors

```bash
az logic workflow run show \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --name <RUN_NAME>
```

---

## 🔍 Troubleshooting

### Problem: Logic App Not Running

**Symptom:** No runs in history

**Check:**
```bash
az logic workflow show \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --query "{state:state, lastTriggerTime:definition.triggers.Recurrence}"
```

**Solution:** Ensure state is "Enabled"

### Problem: Query Returns No Results

**Symptom:** Logic App runs but doesn't update SOCRadar

**Debug:**
1. Go to Log Analytics workspace
2. Run query manually:
```kql
SecurityIncident
| where TimeGenerated > ago(10m)
| where Status == "Closed"
| where Title contains "SOCRadar-AlarmID-"
| project Title, Status, LastModifiedTime
```

**Solutions:**
- Check incident title format: must contain "SOCRadar-AlarmID-{id}"
- Verify incidents exist in timeframe
- Check workspace permissions

### Problem: SOCRadar API Error

**Symptom:** Logic App runs but gets 401/403 from SOCRadar

**Check API Key:**
```bash
curl -X GET \
  "https://platform.socradar.com/api/companies/<COMPANY_ID>/alarms" \
  -H "Authorization: Token <API_KEY>"
```

**Solutions:**
- Verify API key is correct
- Check company ID matches
- Ensure API key has write permissions

---

## 💰 Cost Estimate

**Logic App Consumption Plan:**
- Runs: 288 per day (every 5 minutes)
- Actions per run: ~3-5 (varies by closed incidents)
- Estimated cost: **$5-15/month** (depends on incident volume)

**Cost Optimization:**
- Increase polling interval to 10-15 minutes
- Use Standard plan for high volume (predictable cost)

---

## 🔒 Security

- ✅ API Key stored as `securestring` (encrypted in ARM deployment)
- ✅ Managed Identity for Azure authentication (no credentials)
- ✅ RBAC: Logic App has **Microsoft Sentinel Responder** role only
- ✅ HTTPS-only communication
- ✅ Secrets never appear in logs

---

## 🆚 v1 vs v2 Comparison

| Feature | v1.0 (Webhook) | v2.0 (Polling) |
|---------|----------------|----------------|
| **Deployment** | ❌ Requires manual Portal steps | ✅ Fully automated |
| **Trigger Type** | ApiConnectionWebhook | Recurrence |
| **Automation Rule** | Required (caused errors) | Not needed |
| **Latency** | Instant | 5-10 minutes |
| **Production Ready** | After manual setup | Immediately |
| **Customer Friendly** | ❌ Complex setup | ✅ One-click |
| **Error-Prone** | ✅ "Missing permissions" | ❌ None |

**Decision:** v2.0 polling is better for automated customer deployments.

---

## 🤝 Related Repositories

- **Webhook Integration (SOCRadar → Sentinel):** [socradar-sentinel-alarm-connector-api-fork-orcun](https://github.com/orcunsami/socradar-sentinel-alarm-connector-api-fork-orcun)

Both repos work together for full bidirectional sync:
1. **Webhook repo:** SOCRadar alarms create Sentinel incidents (instant)
2. **This repo:** Sentinel incident closures update SOCRadar alarms (5-min delay)

---

## 📚 FAQ

**Q: Why polling instead of webhook?**
A: Webhook triggers require automated webhook subscription registration that's impossible via ARM/CLI. Polling eliminates this complexity while maintaining functionality.

**Q: Is 5-minute delay acceptable?**
A: For incident closure workflow, yes. Incidents aren't closed in real-time anyway.

**Q: Can I change polling interval?**
A: Yes! Set `PollingIntervalMinutes` parameter (1-60 minutes). Default is 5.

**Q: Will it process same incident twice?**
A: No. Query uses `summarize arg_max()` to get latest state only. SOCRadar API is also idempotent.

**Q: What if I have multiple SOCRadar accounts?**
A: Deploy multiple Logic Apps with different `CompanyId` and `SocradarApiKey`.

---

## 📝 Changelog

### v2.0.0 (2025-11-17)
- ✨ **BREAKING:** Switched to polling architecture
- ✨ Removed webhook trigger (incompatible with automation)
- ✨ Removed Automation Rule (not needed)
- ✨ Removed deployment script (not needed)
- ✨ Zero manual steps required
- 🐛 Fixed "missing permissions" error permanently
- ✅ Production-ready on first deployment

### v1.0.0 (2025-11-15)
- Initial release with webhook trigger
- Required manual Portal configuration

---

## 📞 Support

**Issues?**
1. Check [Troubleshooting](#-troubleshooting) section
2. Review Logic App run history
3. Verify query in Log Analytics
4. Open GitHub issue with details

---

**Happy Automating! 🎉**

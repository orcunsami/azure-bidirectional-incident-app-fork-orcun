# SOCRadar-Azure Sentinel Bidirectional Integration

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Forcunsami%2Fazure-bidirectional-incident-app-fork-orcun%2Fmain%2Fsocradar-sentinel-integration-arm.json)

Automatically synchronize incident status between SOCRadar and Azure Sentinel. When you close a Sentinel incident, the corresponding SOCRadar alarm is automatically resolved.

## Architecture

This Logic App uses a polling mechanism to detect closed incidents in Sentinel and update SOCRadar alarms accordingly:

1. **Every 5 minutes** (configurable), the Logic App queries Sentinel for closed incidents
2. **Filters** incidents with title pattern `SOCRadar-AlarmID-*`
3. **Extracts** alarm ID from incident title
4. **Updates** SOCRadar alarm status via API to RESOLVED

**Trade-off:** 5-10 minute delay after closing incident (acceptable for incident closure workflow).

## Prerequisites

- Azure subscription with Microsoft Sentinel enabled
- Existing Log Analytics Workspace with Sentinel
- SOCRadar API Key and Company ID

## Deployment

### Option 1: Portal (Recommended)

Click the **Deploy to Azure** button above and fill in the parameters.

### Option 2: Azure CLI

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

### Option 3: PowerShell

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName <YOUR_RESOURCE_GROUP> `
  -TemplateFile socradar-sentinel-integration-arm.json `
  -WorkspaceName <YOUR_WORKSPACE_NAME> `
  -SocradarApiKey <YOUR_API_KEY> `
  -CompanyId <YOUR_COMPANY_ID> `
  -PollingIntervalMinutes 5
```

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| **WorkspaceName** | Yes | - | Log Analytics workspace **NAME** (not ID!) |
| **SocradarApiKey** | Yes | - | SOCRadar API authentication token |
| **CompanyId** | Yes | - | SOCRadar company identifier |
| PlaybookName | No | SOCRadar-CloseAlarm-Polling | Logic App resource name |
| PollingIntervalMinutes | No | 5 | Polling frequency (1-60 minutes) |
| Location | No | Resource group location | Azure region |

### ⚠️ Important: Workspace Name vs Workspace ID

**Common mistake:** Using Workspace **ID** instead of Workspace **NAME**

- ✅ **CORRECT:** `company-log-analytics-workspace-0001` (Name)
- ❌ **WRONG:** `cab033c7-927c-46d1-bb24-7af8090ae3fc` (Workspace ID)

Find your workspace name:
1. Azure Portal → Log Analytics workspaces
2. Click on your workspace
3. Copy the **Name** field (NOT the Workspace ID)

## How It Works

```
┌─────────────────────────────────┐
│     SOCRadar Platform           │
│                                 │
│  Alarm: OPEN ──────► RESOLVED  │
│         ▲                ▲      │
└─────────┼────────────────┼──────┘
          │ Webhook        │ API PATCH
          │ (Repo 1)       │ (This Repo)
┌─────────▼────────────────┼──────┐
│   Azure Sentinel         │      │
│                          │      │
│  Incident: NEW ──► CLOSED───┐   │
│                            │   │
│  ┌──────────────────────┐  │   │
│  │  Logic App           │  │   │
│  │  (Every 5 minutes)   │  │   │
│  │                      │  │   │
│  │  1. Query: Closed    │──┘   │
│  │  2. Filter: SOCRadar-* │    │
│  │  3. Extract Alarm ID │     │
│  │  4. PATCH API        │─────┘
│  └──────────────────────┘     │
└────────────────────────────────┘
```

### Query Logic

The Logic App queries Sentinel using KQL:

```kql
SecurityIncident
| where TimeGenerated > ago(10m)
| where Status == "Closed"
| where Title contains "SOCRadar-AlarmID-"
| summarize arg_max(TimeGenerated, *) by IncidentNumber
| project Title, IncidentNumber, Status
```

For each matching incident:
- Extract alarm ID from title (e.g., `SOCRadar-AlarmID-12345` → `12345`)
- Call SOCRadar API: `POST /alarms/status/change` with status "2" (RESOLVED)

## Testing

### Verify Deployment

```bash
# Check Logic App status
az logic workflow show \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --query "state"

# Expected: "Enabled"
```

### Manual Trigger

```bash
# Run Logic App manually
az logic workflow run trigger \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --trigger-name Recurrence
```

### End-to-End Test

1. Create a test alarm in SOCRadar (creates incident in Sentinel)
2. Close the incident in Sentinel
3. Wait 5-10 minutes
4. Verify alarm status in SOCRadar:

```bash
curl -X GET \
  "https://platform.socradar.com/api/companies/<COMPANY_ID>/alarms/<ALARM_ID>" \
  -H "Authorization: Token <API_KEY>"

# Expected response: "status": "RESOLVED"
```

## Monitoring

### View Run History

**Portal:**
1. Logic App → Overview → Runs history
2. Click on any run for details

**CLI:**
```bash
az logic workflow run list \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --top 10 \
  --query "[].{name:name, status:status, startTime:startTime}" \
  --output table
```

## Troubleshooting

### Logic App Not Running

**Check state:**
```bash
az logic workflow show \
  --resource-group <YOUR_RG> \
  --name SOCRadar-CloseAlarm-Polling \
  --query "state"
```

**Solution:** Ensure state is "Enabled"

### Query Returns No Results

**Debug in Log Analytics:**
```kql
SecurityIncident
| where TimeGenerated > ago(10m)
| where Status == "Closed"
| where Title contains "SOCRadar-AlarmID-"
| project Title, Status, LastModifiedTime
```

**Verify:**
- Incident title contains `SOCRadar-AlarmID-{id}`
- Incidents exist in timeframe
- Workspace permissions are correct

### SOCRadar API Error

**Test API key:**
```bash
curl -X GET \
  "https://platform.socradar.com/api/companies/<COMPANY_ID>/alarms" \
  -H "Authorization: Token <API_KEY>"
```

**Check:**
- API key is valid
- Company ID is correct
- API key has write permissions

## Cost Estimate

**Logic App Consumption Plan:**
- Runs: 288 per day (every 5 minutes)
- Actions per run: ~3-5 (varies by closed incidents)
- Estimated cost: **$5-15/month**

**Optimization:**
- Increase polling interval to 10-15 minutes
- Use Standard plan for high volume

## Security

- API Key stored as `securestring` (encrypted)
- Managed Identity for Azure authentication
- RBAC: Logic App has **Microsoft Sentinel Responder** role only
- HTTPS-only communication
- Secrets never appear in logs

## Related Repositories

**Complete bidirectional sync requires both:**
1. [Webhook Integration](https://github.com/orcunsami/socradar-sentinel-alarm-connector-api-fork-orcun) - SOCRadar → Sentinel (instant)
2. This repository - Sentinel → SOCRadar (5-min delay)

## FAQ

**Q: Why polling instead of webhook?**
A: Webhook triggers require manual webhook subscription registration that cannot be automated via ARM/CLI. Polling eliminates this complexity.

**Q: Is 5-minute delay acceptable?**
A: Yes, for incident closure workflow. Incidents are not closed in real-time anyway.

**Q: Can I change polling interval?**
A: Yes, set `PollingIntervalMinutes` parameter (1-60 minutes).

**Q: Will it process the same incident twice?**
A: No. Query uses `summarize arg_max()` to get latest state only. SOCRadar API is also idempotent.

**Q: Multiple SOCRadar accounts?**
A: Deploy multiple Logic Apps with different `CompanyId` and `SocradarApiKey`.

## Support

1. Check [Troubleshooting](#troubleshooting) section
2. Review Logic App run history
3. Verify query in Log Analytics
4. Open GitHub issue with details

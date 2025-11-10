# SOCRadar - Azure Sentinel Integration Guide

## 📋 Overview

This Logic App automatically closes the corresponding alarm in SOCRadar platform when an incident is closed in Azure Sentinel.

### ✨ Features
- ✅ **One-click deployment**: Automatic deployment with ARM template
- ✅ **Automatic synchronization**: SOCRadar alarm closes when Sentinel incident closes
- ✅ **Smart closure**: Distinguishes between False Positive and True Positive
- ✅ **Comment addition**: Automatically adds comments to both platforms
- ✅ **Error management**: Alerts when Alarm ID is not found

---

## 🚀 Installation Steps

### Step 1: Prerequisites

**Required Information:**
1. ✅ SOCRadar API Key
   - Get it from SOCRadar Platform → Settings → API Keys
   
2. ✅ Azure Sentinel Workspace Information
   - Workspace Name
   - Resource Group Name

**Required Permissions:**
- "Contributor" or "Logic App Contributor" role in Azure
- "Microsoft Sentinel Responder" role on Sentinel Workspace

---

### Step 2: Logic App Deployment

#### **Option A: Deploy from Azure Portal (Recommended)**

1. **Go to Azure Portal**: https://portal.azure.com

2. **Navigate to "Deploy a custom template"**:
   - Type "deploy custom template" in the search bar
   - Select "Deploy a custom template"

3. **Load the template**:
   - Click "Build your own template in the editor"
   - Paste the contents of `socradar-sentinel-integration.json`
   - Click "Save"

4. **Fill in the parameters**:
   ```
   Subscription: [Select your Azure subscription]
   Resource Group: [Select the RG where Sentinel is or create new]
   Region: [Select your region]
   
   Playbook Name: SOCRadar-CloseAlarm-OnIncidentClose
   SOCRadar API Key: [Enter your API Key - will be hidden as •••••]
   Workspace Name: [Your Sentinel workspace name]
   Workspace Resource Group: [RG where workspace is located]
   ```

5. **Deploy**:
   - Click "Review + create"
   - Click "Create"
   - Wait for deployment to complete (1-2 minutes)

#### **Option B: Deploy with Azure CLI**

```bash
# Create resource group (skip if exists)
az group create --name SentinelAutomation-RG --location westeurope

# Deploy template
az deployment group create \
  --resource-group SentinelAutomation-RG \
  --template-file socradar-sentinel-integration.json \
  --parameters \
    PlaybookName="SOCRadar-CloseAlarm-OnIncidentClose" \
    SOCRadarAPIKey="YOUR_API_KEY_HERE" \
    WorkspaceName="YourSentinelWorkspace" \
    WorkspaceResourceGroup="YourWorkspaceRG"
```

#### **Option C: Deploy with PowerShell**

```powershell
# Login to Azure
Connect-AzAccount

# Deploy template
New-AzResourceGroupDeployment `
  -ResourceGroupName "SentinelAutomation-RG" `
  -TemplateFile ".\socradar-sentinel-integration.json" `
  -PlaybookName "SOCRadar-CloseAlarm-OnIncidentClose" `
  -SOCRadarAPIKey "YOUR_API_KEY_HERE" `
  -WorkspaceName "YourSentinelWorkspace" `
  -WorkspaceResourceGroup "YourWorkspaceRG"
```

---

### Step 3: Authorization (Authentication)

After deployment completes:

1. **Open the Logic App**:
   - Azure Portal → Logic Apps
   - Select "SOCRadar-CloseAlarm-OnIncidentClose"

2. **Authorize Sentinel connection**:
   - Click "API connections"
   - Select "azuresentinel-..." connection
   - Click "Edit API connection"
   - Click "Authorize" button
   - Sign in with your Azure account
   - Click "Save"

3. **Grant Managed Identity Permission**:
   ```bash
   # Grant Sentinel permission to Logic App's Managed Identity
   az role assignment create \
     --assignee [Logic App Object ID] \
     --role "Microsoft Sentinel Responder" \
     --scope /subscriptions/[subscription-id]/resourceGroups/[workspace-rg]/providers/Microsoft.OperationalInsights/workspaces/[workspace-name]
   ```

   **Via Portal:**
   - Sentinel Workspace → Access Control (IAM)
   - Click "Add role assignment"
   - Role: "Microsoft Sentinel Responder"
   - Assign access to: "Managed Identity"
   - Members: Select your Logic App
   - "Review + assign"

---

### Step 4: Configure SOCRadar Alerts

For SOCRadar alerts to match correctly in Sentinel:

#### **Analytics Rule Configuration**

When sending SOCRadar alerts to Sentinel, add the SOCRadar Alarm ID to the ProductName field:

**Example Data Connector Mapping:**
```kusto
// When sending SOCRadar alert to Sentinel
let socradarAlarmId = 12345;
let productName = strcat("SOCRadar-AlarmID-", socradarAlarmId);

SecurityAlert
| extend ProductName = productName
| extend AlertType = "SOCRadar Threat Intelligence"
```

#### **Custom Log or API Integration**

If you're sending alerts from SOCRadar via API:

```python
# Python example
def send_to_sentinel(alarm_id, alarm_data):
    alert_payload = {
        "AlertName": alarm_data["title"],
        "ProductName": f"SOCRadar-AlarmID-{alarm_id}",  # ← IMPORTANT
        "Severity": alarm_data["severity"],
        "Description": alarm_data["description"],
        # ... other fields
    }
    # Send to Sentinel
```

```json
// JSON example
{
  "AlertName": "Phishing Domain Detected",
  "ProductName": "SOCRadar-AlarmID-12345",
  "Severity": "High",
  "Description": "Suspicious domain detected..."
}
```

---

## 🎯 How It Works

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  Incident Closed in Sentinel                                │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  Logic App Triggered                                        │
│  Trigger: "When incident is updated"                        │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  Check if Status = "Closed"                                 │
└────────────────┬────────────────────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
       YES               NO → No action, end
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Extract SOCRadar Alarm ID from Alert ProductName           │
│  Format: "SOCRadar-AlarmID-12345"                           │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  Alarm ID found?                                            │
└────────────────┬────────────────────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
      YES               NO → Add warning comment
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Decide based on Incident Classification:                   │
│  • False Positive → Close as False Positive in SOCRadar     │
│  • True Positive → Resolve in SOCRadar                      │
│  • Other → Resolve in SOCRadar                              │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  Call SOCRadar API                                          │
│  Endpoint: /incidents/{alarmId}/resolve or                  │
│           /incidents/{alarmId}/false-positive               │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  Add success comment to Sentinel incident                   │
│  "✅ SOCRadar alarm automatically closed"                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧪 Testing

### Test Scenario

1. **Create Test Incident**:
   - Create a new incident in Sentinel
   - Add `SOCRadar-AlarmID-12345` to alert's ProductName

2. **Close Incident**:
   - Status: Closed
   - Classification: False Positive or True Positive

3. **Verify**:
   - Logic App Runs → View last run
   - SOCRadar Platform → Verify alarm is closed
   - Sentinel Incident → Check comment was added

### Logic App Logs

```bash
# View logs with Azure CLI
az monitor activity-log list \
  --resource-id "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Logic/workflows/SOCRadar-CloseAlarm-OnIncidentClose" \
  --start-time 2025-01-01 \
  --query "[].{Time:eventTimestamp, Status:status, Operation:operationName}"
```

---

## 🔧 Customization

### Add More Closure Reasons

You can edit the Logic App to add more classifications:

```json
"cases": {
  "BenignPositive": {
    "case": "BenignPositive",
    "actions": {
      "Close_as_Benign": {
        // API call
      }
    }
  }
}
```

### Different SOCRadar Endpoints

You can customize the template for different SOCRadar modules:

```json
// For Dark Web Monitoring
"uri": "@{parameters('SOCRadarAPIEndpoint')}/dark-web/@{outputs('Parse_Alarm_ID')}/resolve"

// For Brand Protection
"uri": "@{parameters('SOCRadarAPIEndpoint')}/brand-protection/@{outputs('Parse_Alarm_ID')}/resolve"
```

---

## ❓ Troubleshooting

### Problem: "Alarm ID not found" warning

**Solution:**
- Check alert's ProductName field
- Format should be: `SOCRadar-AlarmID-[number]`
- Example: `SOCRadar-AlarmID-12345`

### Problem: "Authorization failed"

**Solution:**
1. Verify SOCRadar API Key is valid
2. Check API Key is in correct parameter
3. Re-enter Logic App parameters

### Problem: "Managed Identity permission denied"

**Solution:**
```bash
# Add permission to Sentinel Workspace
az role assignment create \
  --assignee [Logic App Principal ID] \
  --role "Microsoft Sentinel Responder" \
  --scope [Sentinel Workspace Resource ID]
```

### Problem: Logic App not running

**Checklist:**
1. ✅ Is Logic App in "Enabled" state?
2. ✅ Is Sentinel connection authorized?
3. ✅ Is trigger configured correctly?
4. ✅ Are there errors in Run History?

---

## 📊 Monitoring and Reporting

### Logic App Metrics

```kusto
// Application Insights Query
requests
| where cloud_RoleName == "SOCRadar-CloseAlarm-OnIncidentClose"
| summarize 
    TotalRuns = count(),
    SuccessRate = countif(success == true) * 100.0 / count(),
    AvgDuration = avg(duration)
    by bin(timestamp, 1d)
```

### Create Dashboard

Azure Portal → Dashboards → New Dashboard

**Tiles to Add:**
- Logic App Runs (last 24 hours)
- Success Rate
- Average Duration
- Failed Runs

---

## 🔐 Security Best Practices

1. **API Key Security**:
   - Never hardcode API Key in code
   - Use Azure Key Vault (recommended)
   - Rotate regularly

2. **Least Privilege**:
   - Grant only necessary permissions to Logic App
   - Use custom role definitions

3. **Logging**:
   - Enable diagnostic settings
   - Send to Log Analytics

---

## 📞 Support

### Report Issues

If you encounter a problem:

1. Collect error details from Logic App Run History
2. Check SOCRadar API response
3. Note Sentinel incident ID and details

### Useful Links

- SOCRadar Documentation: https://docs.socradar.com
- Azure Sentinel Docs: https://docs.microsoft.com/azure/sentinel
- Logic Apps Docs: https://docs.microsoft.com/azure/logic-apps

---

## 📝 Version History

- **v1.0** (2025-11-10): Initial release
  - Automatic alarm closure
  - False Positive/True Positive support
  - Comment addition feature

---

## 📄 License

This template is provided free of charge for SOCRadar customers.

---

**✅ Installation complete! Your Sentinel incidents will now automatically sync with SOCRadar alarms when closed.**

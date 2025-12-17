# Azure Cost Monitor - Quick Start Guide

## ⚡ 5-Minute Setup

This guide gets you from zero to monitoring Azure costs in under 10 minutes.

### Prerequisites Checklist
- [ ] Azure subscription with active resources
- [ ] Power Automate Premium license
- [ ] Teams channel for alerts
- [ ] 5 minutes of setup time

---

## Step 1: Create Service Principal (2 min)

```bash
# Run in Azure Cloud Shell
az ad sp create-for-rbac \
  --name "PowerAutomate-CostMonitor" \
  --role "Cost Management Reader" \
  --scopes /subscriptions/YOUR-SUBSCRIPTION-ID

# Save the output - you'll need these values:
# - appId (CLIENT_ID)
# - password (CLIENT_SECRET)
# - tenant (TENANT_ID)
```

**Alternative: Using Azure Portal**
1. Navigate to Azure Active Directory → App registrations
2. Click "New registration"
3. Name: "PowerAutomate-CostMonitor"
4. Click "Register"
5. Go to Certificates & secrets → New client secret
6. Save the secret value
7. Go to your Subscription → Access control (IAM)
8. Add role assignment → Cost Management Reader
9. Select the app you just created

---

## Step 2: Get Log Analytics Details (1 min)

1. Navigate to your Log Analytics workspace (or create new)
2. Go to **Agents** → **Log Analytics agent instructions**
3. Copy these values:
   - **Workspace ID**: `12345678-1234-1234-1234-123456789abc`
   - **Primary Key**: `long-base64-encoded-string==`

---

## Step 3: Import Flow (1 min)

### Option A: Via Power Automate Portal
1. Go to https://make.powerautomate.com
2. Click **My flows** → **Import** → **Import Package (Legacy)**
3. Upload `azure-cost-monitor-solution.zip`
4. Follow the import wizard

### Option B: Via Flow Definition
1. Go to **My flows** → **New flow** → **Automated cloud flow**
2. Name it "Azure Cost Anomaly Detection"
3. Skip trigger selection (we'll paste the definition)
4. Click **...** (three dots) → **Open in new tab**
5. Switch to **Code View** (left panel)
6. Paste contents from `flow-definition.json`
7. Save

---

## Step 4: Configure Connections (2 min)

The flow needs these connections:

### Teams Connection
1. Click **Test** → Choose a connection → **Create**
2. Sign in with your Microsoft account
3. Grant permissions

### Office 365 Connection
1. Same process as Teams
2. Use same account for consistency

### Log Analytics Connection
1. Click the connection → Configure
2. Enter **Workspace ID** from Step 2
3. Enter **Primary Key** from Step 2
4. Test connection

### HTTP - Azure Resource Manager
This is already configured in the flow using Service Principal from Step 1.

---

## Step 5: Update Flow Variables (1 min)

Click on the flow → **Edit** → Update these variables:

| Variable | Your Value | Example |
|----------|------------|---------|
| varSubscriptionId | Your Azure subscription ID | `12345678-abcd-...` |
| varThresholdMultiplier | 1.5 (or customize) | `1.5` |
| varCriticalThreshold | Your budget threshold | `5000` |
| varFinanceEmail | Finance team email | `finance@company.com` |

**Where to find Subscription ID:**
```bash
# Azure CLI
az account show --query id -o tsv

# Or in Azure Portal
Portal → Subscriptions → Copy subscription ID
```

---

## Step 6: Test Run (1 min)

1. Click **Test** in the flow editor
2. Select **Manually**
3. Click **Test**
4. Watch the flow run (30-60 seconds)

### ✅ Success Indicators:
- [ ] Flow completes without errors
- [ ] Teams message appears in your channel
- [ ] Email arrives (if anomaly detected)
- [ ] Log Analytics query returns data:
```kql
AzureCostAnomalies_CL
| top 10 by TimeGenerated desc
```

### ❌ Common Issues:

**"Unauthorized" on HTTP step**
- Check Service Principal has Cost Management Reader role
- Verify subscription ID is correct
- Ensure SP credentials in flow are accurate

**Teams message not appearing**
- Add bot to Teams channel by @mentioning it
- Verify channel ID is correct
- Check Teams connection permissions

**No email sent**
- Normal if no anomaly detected on first run
- Check threshold is lower than current cost for testing
- Verify Office 365 connection

---

## Step 7: Schedule It (30 seconds)

1. Go to flow trigger (first step)
2. Set frequency: **Daily**
3. Set time: **06:00 AM**
4. Set timezone: Your local timezone
5. **Save** the flow
6. Toggle flow to **On**

---

## 🎉 You're Done!

The flow will now:
- ✅ Run every morning at 6 AM
- ✅ Analyze last 30 days of cost data
- ✅ Alert if today's cost > 150% of baseline
- ✅ Send Teams + Email notifications
- ✅ Log all anomalies to Log Analytics

---

## Next Steps

### Customize Alerts
Edit these in the flow:
```javascript
// Change threshold (currently 150%)
varThresholdMultiplier = 2.0  // Now 200%

// Change critical alert level
varCriticalThreshold = 10000  // $10K instead of $5K

// Add more recipients
emailMessage/To = "finance@company.com;cto@company.com"
```

### Add Slack Integration
If your team uses Slack:
1. Add Slack connector
2. Duplicate the "Post to Teams" action
3. Change to "Post message" (Slack)
4. Use same adaptive card format

### Connect to ServiceNow
For critical alerts:
1. Add ServiceNow connector
2. Add condition: `if varTodayCost > varCriticalThreshold`
3. Create incident with cost details

### Create Dashboard
Query Log Analytics:
```kql
// Monthly anomaly trend
AzureCostAnomalies_CL
| where TimeGenerated > ago(90d)
| summarize AnomalyCount=count(), TotalCost=sum(currentCost_d) by bin(TimeGenerated, 1d)
| render timechart

// Top resource groups by anomalies
AzureCostAnomalies_CL
| extend ResourceGroups = parse_json(topResourceGroups_s)
| mv-expand ResourceGroups
| summarize AnomalyCount=count() by tostring(ResourceGroups)
| top 10 by AnomalyCount
```

---

## Monitoring & Maintenance

### Weekly Tasks
- [ ] Review false positive rate
- [ ] Adjust threshold if needed
- [ ] Check flow run history for errors

### Monthly Tasks
- [ ] Review total alerts sent
- [ ] Validate cost savings from early detection
- [ ] Update recipient list if team changes

### Quarterly Tasks
- [ ] Rotate Service Principal secret
- [ ] Review and optimize baseline calculation
- [ ] Gather feedback from finance team

---

## Troubleshooting Quick Reference

| Symptom | Fix |
|---------|-----|
| No alerts for 7+ days | Lower threshold temporarily to test |
| Too many false positives | Increase threshold multiplier to 1.8 or 2.0 |
| Flow fails every run | Check Service Principal permissions |
| Missing cost data in reports | Verify subscription ID is correct |
| Log Analytics query returns nothing | Check Log Type name matches flow configuration |

---

## Cost Breakdown

**What does this cost to run?**

- Power Automate: Included in Premium ($15/user/month or in M365 E3/E5)
- Azure API calls: ~$0.01/day (Cost Management API is free)
- Log Analytics: ~$0.50/month (100 MB ingestion)
- **Total: ~$15-16/month** (mostly the Power Automate license)

**ROI**: If this flow catches ONE cost spike of $500+, it pays for itself for the year.

---

## Support & Resources

- **Flow Issues**: Check run history in Power Automate portal
- **Azure Permissions**: Review IAM in Azure Portal
- **Cost Management API Docs**: https://learn.microsoft.com/en-us/rest/api/cost-management/
- **Report Issues**: Create issue in GitHub repo

---

## Security Reminder

🔒 **Never commit these values to Git:**
- Service Principal secrets
- Log Analytics keys  
- Subscription IDs (mask last digits)
- Team/channel IDs

Use the `connections.template.json` file with placeholders instead.

---

**Quick Start Complete! 🚀**

Your Azure costs are now being monitored automatically. Set it and forget it!

*Questions? Check the main README.md or create a GitHub issue.*

# 🔧 Troubleshooting Guide - Root Cause Analysis

## Problem: "Missing required permissions" Error

### What Happened

When creating Automation Rules via ARM template or CLI, error occurred:
```
Missing required permissions for Microsoft Sentinel on the playbook resource
```

### What We Tried

1. ✅ Granted Microsoft Sentinel Responder role to Logic App
2. ✅ Granted Logic App Contributor to Azure Security Insights SP
3. ✅ Granted Microsoft Sentinel Automation Contributor to resource group
4. ❌ Still got same error

### Root Cause

The error message is **misleading**. Actual problem:

**ApiConnectionWebhook triggers need webhook subscription registration** that happens automatically in Portal but **cannot be done via ARM/CLI**.

When Automation Rule is created:
1. Validates permissions ✅
2. Tries to register webhook subscription ❌ **Fails here**
3. Reports as "missing permissions" ❌ **Wrong error message**

### Why Portal Works

Portal does **more** than just role assignments:
- Grants permissions ✅
- **Initializes webhook subscription metadata** ✅
- Creates automation rule with webhook binding ✅

This is why everyone says "just use portal" - but that's not automatable!

### Solution

**Use polling instead of webhooks:**
- No webhook subscription needed
- No Automation Rule needed
- Fully automatable
- Trade-off: 5-minute delay (acceptable)

## Architecture Comparison

### Old (v1 - Webhook)
```
Incident closed → Automation Rule → Webhook subscription → Logic App → SOCRadar
                   ^^^^^^^^^^^^^^    ^^^^^^^^^^^^^^^^^^
                   Can't automate    Can't register via ARM
```

### New (v2 - Polling)
```
Logic App (every 5 min) → Query Sentinel → Find closed → Update SOCRadar
          ^^^^^^^^^^^^    ^^^^^^^^^^^^^^    ^^^^^^^^^^^^
          Simple timer    Standard query    Works perfectly
```

## Lessons Learned

1. **"Missing permissions" doesn't always mean permissions**
2. **Portal UI != ARM template capabilities**
3. **Webhook subscriptions have hidden dependencies**
4. **Polling is more reliable for automation**

---

*This document explains why v2.0 exists and why we can't use webhooks.*

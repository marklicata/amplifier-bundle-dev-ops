---
skill: azure-cname-setup
version: 1.0.0
context: current
description: |
  Quick reference guide for adding custom domains (CNAMEs) to Azure Container Apps.
  Covers Microsoft internal DNS request process, Azure CLI configuration, SSL certificate
  binding, and common troubleshooting. Use when setting up custom domains like
  myapp.microsoft.com for Container Apps.
tags: [azure, cname, custom-domain, dns, container-apps, ssl, certificate]
---

# Azure Custom Domain Setup (Quick Reference)

Quick guide for adding custom domains to Azure Container Apps using Microsoft internal DNS.

## When to Use This Skill

- Setting up a custom domain (e.g., `myapp.microsoft.com`) for a Container App
- Adding SSL certificates to custom domains
- Troubleshooting custom domain issues (HTTP 431, verification failures)
- Understanding the DNS approval timeline

For complete guided workflow, use the **azure-cname-setup recipe** instead:
```
amplifier recipes execute dev-ops:recipes/azure-cname-setup.yaml
```

## Prerequisites

Before starting:

```bash
# 1. Azure CLI authenticated
az account show

# 2. Container App exists
az containerapp show --name YOUR_APP --resource-group YOUR_RG

# 3. Access to Microsoft DNS portal
# https://prod.msftdomains.com
```

---

## Quick Start (5 Steps)

### Step 1: Get Container App FQDN

```bash
az containerapp show \
  --name YOUR_APP_NAME \
  --resource-group YOUR_RG \
  --query properties.configuration.ingress.fqdn -o tsv
```

Save this FQDN - you'll need it for the CNAME request.

### Step 2: Request CNAME (Manual - via Portal)

1. **Check availability**: https://prod.msftdomains.com/Management/OwnerLookup
2. **Request CNAME**: https://prod.msftdomains.com/Dns/Form/
   - CNAME name: `myapp` (your subdomain)
   - Target: `<your-fqdn-from-step-1>`
   - Submit and wait for approval (1 hour - 3 days)

### Step 3: Add Domain to Container App

```bash
# After CNAME is approved and active
az containerapp hostname add \
  --hostname myapp.microsoft.com \
  --name YOUR_APP_NAME \
  --resource-group YOUR_RG
```

Azure may provide a TXT verification value. If so:

### Step 4: Request TXT Record (If Required)

1. **Request TXT**: https://prod.msftdomains.com/Dns/Form/
   - Request type: TXT
   - Hostname: `myapp.microsoft.com`
   - TXT Value: `<value-from-azure>`
   - Submit and wait for approval

### Step 5: Bind Certificate

```bash
# Get environment name first
ENV_NAME=$(az containerapp show \
  --name YOUR_APP_NAME \
  --resource-group YOUR_RG \
  --query properties.environmentId -o tsv | \
  grep -oP '(?<=environments/)[^/]+')

# Bind managed certificate
az containerapp hostname bind \
  --hostname myapp.microsoft.com \
  --name YOUR_APP_NAME \
  --resource-group YOUR_RG \
  --environment $ENV_NAME \
  --validation-method CNAME
```

Wait 5-15 minutes for certificate provisioning.

---

## Verification

```bash
# Test HTTPS access
curl -I https://myapp.microsoft.com

# Should return HTTP 200 with valid SSL
```

---

## Common Issues & Fixes

### Issue: HTTP 431 (Request Header Fields Too Large)

**Fix:**
```bash
az containerapp update \
  --name YOUR_APP_NAME \
  --resource-group YOUR_RG \
  --set-env-vars "WEBSITE_AUTH_DISABLE_IDENTITY_FLOW=true"
```

### Issue: Domain verification pending

**Check DNS propagation:**
```bash
nslookup myapp.microsoft.com           # Check CNAME
nslookup -type=TXT myapp.microsoft.com # Check TXT
```

Wait 15 minutes - 24 hours for DNS propagation.

### Issue: Certificate binding failed

**Verify domain is verified first:**
```bash
az containerapp hostname list \
  --name YOUR_APP_NAME \
  --resource-group YOUR_RG
```

Look for `verificationState: "Verified"` before binding certificate.

---

## Timeline Expectations

| Step | Duration | Can Control? |
|------|----------|--------------|
| CNAME request approval | 1 hour - 3 days | No |
| TXT request approval | 1 hour - 3 days | No |
| DNS propagation | 15 min - 24 hours | No |
| Azure configuration | 5-15 minutes | Yes |

**Total realistic timeline: 1-3 days** due to manual approval processes.

---

## Useful Commands

```bash
# List all custom domains
az containerapp hostname list --name APP --resource-group RG

# Check certificate status
az containerapp env certificate list --name ENV --resource-group RG

# View Container App logs
az containerapp logs show --name APP --resource-group RG --tail 50

# Delete custom domain (if need to re-add)
az containerapp hostname delete --hostname DOMAIN --name APP --resource-group RG
```

---

## Next Steps After Setup

1. **Update documentation** - Reference new custom domain
2. **Update CI/CD** - Test custom domain in workflows
3. **Monitor certificate** - Managed certs auto-renew, but verify
4. **Test thoroughly** - Verify all application endpoints work

---

## Need More Help?

**For detailed context and troubleshooting:**
- Read: `@dev-ops:context/azure-cname-setup.md`
- Ask: Delegate to `dev-ops:agents/cname-orchestrator`

**For guided workflow with approval gates:**
```bash
# Run the complete recipe
amplifier recipes execute dev-ops:recipes/azure-cname-setup.yaml \
  context='{"container_app_name": "myapp", "resource_group": "my-rg", "custom_domain": "myapp.microsoft.com"}'
```

The recipe handles the entire workflow with human approval gates for manual DNS steps.

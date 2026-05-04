# Azure Custom Domain (CNAME) Setup for Container Apps

Complete guide for adding custom domains to Azure Container Apps using Microsoft internal DNS and SSL certificates.

## Overview

Setting up a custom domain involves:
1. **Manual DNS setup** (Microsoft internal portal)
2. **Azure configuration** (Azure CLI or Portal)
3. **Certificate management** (SSL/TLS)
4. **Troubleshooting** (common issues)

This is a hybrid manual/automated process due to Microsoft's internal DNS management.

---

## The Complete Workflow

### Prerequisites

- Azure CLI authenticated: `az account show`
- Access to Microsoft internal DNS portal: https://prod.msftdomains.com
- Container App deployed and running
- Domain name decided (e.g., `myapp.microsoft.com`)

---

## Phase 1: DNS Setup (Manual - Microsoft Portal)

These steps must be done via the Microsoft internal DNS portal.

### Step 1: Check CNAME Availability

**Portal:** https://prod.msftdomains.com/Management/OwnerLookup

1. Navigate to the Owner Lookup page
2. Enter your desired CNAME (e.g., `myapp`)
3. Verify it's available for registration
4. **Document the exact CNAME** you'll use

### Step 2: Request CNAME Creation

**Portal:** https://prod.msftdomains.com/Dns/Form/

1. Navigate to the DNS Form
2. Select request type: **CNAME**
3. Fill in the form:
   - **CNAME name**: Your chosen subdomain (e.g., `myapp`)
   - **Target**: Your Container App's default FQDN
     ```bash
     # Get your Container App FQDN
     az containerapp show \
       --name YOUR_APP_NAME \
       --resource-group YOUR_RG \
       --query properties.configuration.ingress.fqdn -o tsv
     ```
   - **TTL**: Use default (typically 3600)
4. Submit the request
5. **Wait for approval** (can take hours to days depending on team processes)

### Step 3: Request TXT Record (for domain verification)

**Portal:** https://prod.msftdomains.com/Dns/Form/

1. Navigate to the DNS Form again
2. Select request type: **TXT**
3. Fill in the form:
   - **Hostname**: Your CNAME (e.g., `myapp`)
   - **TXT Value**: Will be provided by Azure in Phase 2
4. **WAIT** - Don't submit this yet. You'll get the TXT value from Azure in Phase 2.

---

## Phase 2: Azure Container App Configuration

Once your CNAME request is **approved and active**, configure Azure.

### Step 1: Add Custom Domain to Container App

#### Option A: Azure CLI

```bash
# Set variables
CONTAINER_APP_NAME="your-app-name"
RESOURCE_GROUP="your-resource-group"
CUSTOM_DOMAIN="myapp.microsoft.com"  # Your approved CNAME

# Add the custom domain
az containerapp hostname add \
  --hostname $CUSTOM_DOMAIN \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP
```

#### Option B: Azure Portal

1. Navigate to your Container App
2. Go to **Networking** → **Custom domains**
3. Click **Add custom domain**
4. Enter your CNAME: `myapp.microsoft.com`
5. Azure will provide a **TXT verification record**
6. **Copy this TXT value** - you'll need it for Phase 1, Step 3

**IMPORTANT:** If Azure asks for domain verification:
- Go back to https://prod.msftdomains.com/Dns/Form/
- Submit the TXT record request with the verification value Azure provided
- Wait for TXT record approval
- Return to Azure and complete the custom domain addition

### Step 2: Verify Domain Ownership

```bash
# Check domain verification status
az containerapp hostname list \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP

# Look for "verificationState": "Verified"
```

---

## Phase 3: SSL Certificate Setup

### Step 1: Add Certificate to Container App Environment

#### Option A: Managed Certificate (Recommended)

```bash
# Azure will automatically provision a managed certificate
# This happens when you bind the domain in the next step
```

#### Option B: Upload Custom Certificate

```bash
# If you have your own certificate
az containerapp env certificate upload \
  --name YOUR_ENV_NAME \
  --resource-group YOUR_RG \
  --certificate-file /path/to/cert.pfx \
  --certificate-password "your-password"
```

### Step 2: Bind Certificate to Custom Domain

#### Via Azure Portal

1. Go to your Container App
2. Navigate to **Networking** → **Custom domains**
3. Find your custom domain in the list
4. Click **Add binding**
5. Select certificate source:
   - **Managed certificate** (Azure will create one automatically)
   - **Existing certificate** (if you uploaded one)
6. Click **Add**

#### Via Azure CLI

```bash
# For managed certificate
az containerapp hostname bind \
  --hostname $CUSTOM_DOMAIN \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment YOUR_ENV_NAME \
  --validation-method CNAME

# Certificate will be auto-provisioned and bound
```

### Step 3: Verify HTTPS Access

```bash
# Test HTTPS endpoint
curl -I https://myapp.microsoft.com

# Should return HTTP 200 with valid SSL certificate
```

---

## Phase 4: Troubleshooting

### Issue 1: HTTP 431 Request Header Fields Too Large

**Symptom:** Accessing your custom domain returns HTTP 431 errors.

**Cause:** Azure Container Apps authentication flow adds headers that exceed size limits.

**Solution:** Disable identity flow for the web entry point.

```bash
# Set environment variable to disable identity flow
az containerapp update \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --set-env-vars "WEBSITE_AUTH_DISABLE_IDENTITY_FLOW=true"
```

**When to apply:**
- Only if you encounter HTTP 431 errors after custom domain setup
- Specifically for container apps with authentication enabled
- Safe to apply to web entry points that don't need identity header passthrough

### Issue 2: Domain Verification Fails

**Symptoms:**
- Azure says "Verification pending" or "Not verified"
- Custom domain addition fails

**Solutions:**

1. **Check TXT record propagation:**
   ```bash
   # Check if TXT record is live
   nslookup -type=TXT myapp.microsoft.com
   
   # Or use dig
   dig TXT myapp.microsoft.com
   ```

2. **Verify CNAME is pointing correctly:**
   ```bash
   nslookup myapp.microsoft.com
   
   # Should resolve to your Container App FQDN
   ```

3. **Wait for DNS propagation:**
   - Microsoft internal DNS changes can take 15 minutes to 24 hours
   - Retry domain verification after waiting

4. **Re-request DNS records if needed:**
   - If records are missing or incorrect, submit new requests at https://prod.msftdomains.com/Dns/Form/

### Issue 3: Certificate Binding Fails

**Symptoms:**
- "Certificate provisioning failed"
- Custom domain shows "Not secured"

**Solutions:**

1. **Verify domain ownership first:**
   ```bash
   az containerapp hostname list \
     --name $CONTAINER_APP_NAME \
     --resource-group $RESOURCE_GROUP
   
   # Ensure verificationState is "Verified"
   ```

2. **Check Container App Environment certificates:**
   ```bash
   az containerapp env certificate list \
     --name YOUR_ENV_NAME \
     --resource-group $RESOURCE_GROUP
   ```

3. **Try manual certificate management:**
   - Generate a certificate manually (Let's Encrypt, Azure Key Vault)
   - Upload to Container App Environment
   - Bind to custom domain

### Issue 4: Custom Domain Not Accessible

**Symptoms:**
- Domain resolves but shows errors
- 404 or 502 errors

**Diagnostic Steps:**

1. **Check Container App ingress:**
   ```bash
   az containerapp show \
     --name $CONTAINER_APP_NAME \
     --resource-group $RESOURCE_GROUP \
     --query properties.configuration.ingress
   ```

2. **Verify custom domain is listed:**
   ```bash
   az containerapp hostname list \
     --name $CONTAINER_APP_NAME \
     --resource-group $RESOURCE_GROUP
   ```

3. **Check Container App logs:**
   ```bash
   az containerapp logs show \
     --name $CONTAINER_APP_NAME \
     --resource-group $RESOURCE_GROUP \
     --tail 50
   ```

4. **Test default FQDN:**
   ```bash
   # Get default FQDN
   DEFAULT_FQDN=$(az containerapp show \
     --name $CONTAINER_APP_NAME \
     --resource-group $RESOURCE_GROUP \
     --query properties.configuration.ingress.fqdn -o tsv)
   
   # Test default domain
   curl -I https://$DEFAULT_FQDN
   
   # If default works but custom doesn't, issue is with DNS/certificate binding
   ```

---

## Quick Reference Commands

### Get Container App Details

```bash
# Get default FQDN
az containerapp show \
  --name YOUR_APP \
  --resource-group YOUR_RG \
  --query properties.configuration.ingress.fqdn -o tsv

# List all custom domains
az containerapp hostname list \
  --name YOUR_APP \
  --resource-group YOUR_RG

# Check environment variables
az containerapp show \
  --name YOUR_APP \
  --resource-group YOUR_RG \
  --query properties.template.containers[0].env
```

### Certificate Management

```bash
# List certificates in environment
az containerapp env certificate list \
  --name YOUR_ENV \
  --resource-group YOUR_RG

# Upload new certificate
az containerapp env certificate upload \
  --name YOUR_ENV \
  --resource-group YOUR_RG \
  --certificate-file cert.pfx \
  --certificate-password "password"

# Delete certificate (if rebinding needed)
az containerapp env certificate delete \
  --name YOUR_ENV \
  --resource-group YOUR_RG \
  --certificate CERT_ID
```

### DNS Verification

```bash
# Check CNAME record
nslookup myapp.microsoft.com

# Check TXT record (for verification)
nslookup -type=TXT myapp.microsoft.com

# Detailed DNS query with dig
dig myapp.microsoft.com
dig TXT myapp.microsoft.com
```

---

## Timeline Expectations

| Phase | Typical Duration | Blocking Factors |
|-------|------------------|------------------|
| CNAME availability check | 5 minutes | None |
| CNAME request approval | 1 hour - 3 days | Microsoft DNS team review |
| TXT record request approval | 1 hour - 3 days | Microsoft DNS team review |
| DNS propagation | 15 min - 24 hours | DNS caching |
| Azure domain verification | 5 minutes | DNS records being live |
| Certificate provisioning | 5-15 minutes | Domain verification |
| Total (optimistic) | 2-3 hours | All approvals fast |
| Total (realistic) | 1-3 days | Normal approval processes |

---

## Checklist for Custom Domain Setup

Use this checklist to track progress:

- [ ] **Phase 1: DNS Setup**
  - [ ] Checked CNAME availability (Owner Lookup)
  - [ ] Requested CNAME record (DNS Form)
  - [ ] CNAME request approved and active
  - [ ] Got Azure TXT verification value
  - [ ] Requested TXT record (DNS Form)
  - [ ] TXT request approved and active

- [ ] **Phase 2: Azure Configuration**
  - [ ] Added custom domain to Container App
  - [ ] Domain verification status: Verified
  - [ ] Custom domain appears in hostname list

- [ ] **Phase 3: Certificate**
  - [ ] Certificate added to Container App Environment
  - [ ] Certificate bound to custom domain
  - [ ] HTTPS access works: `curl -I https://myapp.microsoft.com`

- [ ] **Phase 4: Testing**
  - [ ] HTTP redirects to HTTPS
  - [ ] No certificate warnings in browser
  - [ ] Application loads correctly
  - [ ] No HTTP 431 errors (or fixed with WEBSITE_AUTH_DISABLE_IDENTITY_FLOW)

---

## Integration with GitHub Workflows

Once custom domain is set up, update your deployment workflows:

```yaml
# In .github/workflows/deploy.yml
env:
  CUSTOM_DOMAIN: myapp.microsoft.com

jobs:
  deploy:
    steps:
      # After deployment, verify custom domain works
      - name: Test Custom Domain
        run: |
          echo "Testing custom domain..."
          curl -sf https://${{ env.CUSTOM_DOMAIN }}/health
          
      # Optional: Update custom domain binding on redeploy
      - name: Ensure Custom Domain Binding
        run: |
          az containerapp hostname list \
            --name ${{ env.CONTAINER_APP_NAME }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --query "[?name=='${{ env.CUSTOM_DOMAIN }}']"
```

---

## For Your Team

**When to use custom domains:**
- Production deployments that need branded URLs
- Services accessed by users outside your team
- Public-facing APIs or applications

**When default FQDN is fine:**
- Development and staging environments
- Internal tools and services
- Quick prototypes and experiments

**Security considerations:**
- Always use HTTPS (enforced by Azure for custom domains)
- Keep certificates up to date (managed certificates auto-renew)
- Monitor certificate expiration if using custom certificates
- Review access logs for unauthorized domain access attempts

---

**This pattern is proven in production** across real Microsoft internal deployments.
Use it as the default approach for adding custom domains to Azure Container Apps.

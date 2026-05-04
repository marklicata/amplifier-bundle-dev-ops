# Azure Custom Domain (CNAME) Setup - Bundle Updates

This document describes the updates made to the `azure-bundle-dev-ops` bundle to support Azure custom domain (CNAME) setup for Container Apps.

## Summary

The bundle now provides comprehensive support for setting up custom domains (CNAMEs) on Azure Container Apps through Microsoft's internal DNS portal, including:

- ✅ Complete documentation of the CNAME setup process
- ✅ Specialized agent for orchestrating the workflow
- ✅ Staged recipe with approval gates for manual DNS steps
- ✅ Quick reference skill for common scenarios
- ✅ Integration with existing Azure deployment patterns

## What Was Created

### 1. Context Documentation

**File:** `context/azure-cname-setup.md`

Comprehensive guide covering:
- Complete workflow (4 phases: DNS setup, Azure config, certificate binding, verification)
- Microsoft internal DNS portal processes
- Azure CLI commands for custom domain configuration
- SSL certificate provisioning and binding
- Troubleshooting guide (HTTP 431 errors, verification failures, DNS propagation)
- Timeline expectations (realistic: 1-3 days due to approvals)
- Quick reference commands
- Integration with GitHub workflows

### 2. CNAME Orchestrator Agent

**File:** `agents/cname-orchestrator.md`

Specialized agent that:
- Guides users step-by-step through CNAME setup
- Waits for user confirmation at manual approval gates
- Provides exact commands with user's specific values
- Handles troubleshooting for common issues
- Manages timeline expectations
- Verifies each step before proceeding

**When to use:**
```
"I need to add a custom domain to my container app"
"How do I set up myapp.microsoft.com for my Azure app?"
```

### 3. Guided Recipe

**File:** `recipes/azure-cname-setup.yaml`

Staged recipe with approval gates:
- **Stage 1: Preparation** - Verify prerequisites, get default FQDN
- **Stage 2: DNS Request** - Manual CNAME request (approval gate)
- **Stage 3: Azure Configuration** - Add custom domain, get TXT verification value
- **Stage 4: TXT Verification** - Manual TXT request (approval gate)
- **Stage 5: Certificate Binding** - Provision and bind SSL certificate
- **Stage 6: Verification** - Test HTTPS access, fix HTTP 431 if needed
- **Stage 7: Completion** - Final checklist and success summary

**How to run:**
```bash
amplifier recipes execute dev-ops:recipes/azure-cname-setup.yaml \
  context='{"container_app_name": "myapp", "resource_group": "my-rg", "custom_domain": "myapp.microsoft.com"}'
```

### 4. Quick Reference Skill

**File:** `skills/azure-cname-setup/SKILL.md`

Quick reference guide for:
- 5-step quick start process
- Common commands
- Timeline expectations
- Troubleshooting common issues
- When to use the full recipe vs quick reference

**How to load:**
```
load_skill(skill_name="azure-cname-setup")
```

### 5. Bundle Configuration Updates

**Modified Files:**
- `behaviors/dev-ops.yaml` - Registered cname-orchestrator agent
- `context/devops-awareness.md` - Added CNAME capabilities documentation

## How to Use

### For New Users (Guided Workflow)

Use the recipe for a complete guided experience with approval gates:

```bash
amplifier recipes execute dev-ops:recipes/azure-cname-setup.yaml \
  context='{"container_app_name": "myapp", "resource_group": "my-rg", "custom_domain": "myapp.microsoft.com"}'
```

The recipe will:
1. Guide you through prerequisites
2. Wait for your CNAME DNS request approval
3. Configure Azure custom domain
4. Wait for your TXT DNS request approval
5. Bind SSL certificate
6. Verify everything works
7. Provide final checklist

### For Experienced Users (Quick Reference)

Load the skill for quick command reference:

```
load_skill(skill_name="azure-cname-setup")
```

Or consult the context documentation:

```
@dev-ops:context/azure-cname-setup.md
```

### For Interactive Guidance

Delegate to the cname-orchestrator agent:

```
delegate(agent="dev-ops:agents/cname-orchestrator", 
         instruction="Help me set up myapp.microsoft.com for my container app")
```

## Key Features

### ✅ Approval Gates for Manual Steps

The recipe pauses at manual DNS request steps and waits for user confirmation before proceeding. This prevents the workflow from getting ahead of the DNS approval process.

### ✅ Realistic Timeline Management

The documentation and agents set realistic expectations:
- CNAME approval: 1 hour - 3 days
- TXT approval: 1 hour - 3 days
- DNS propagation: 15 minutes - 24 hours
- Total realistic timeline: 1-3 days

### ✅ Comprehensive Troubleshooting

Covers common issues:
- **HTTP 431**: `WEBSITE_AUTH_DISABLE_IDENTITY_FLOW=true` fix
- **Verification failures**: DNS propagation checks
- **Certificate binding failures**: Domain verification requirements
- **Accessibility issues**: Diagnostic commands and log analysis

### ✅ Integration with Existing Patterns

The CNAME setup integrates with existing Azure deployment patterns documented in:
- `context/azure-deployment-patterns.md` - Container Apps best practices
- Includes GitHub workflow integration examples
- References managed identity patterns
- Follows secrets management conventions

## Timeline Expectations

| Phase | Duration | Can Control? |
|-------|----------|--------------|
| CNAME request approval | 1 hour - 3 days | No |
| TXT request approval | 1 hour - 3 days | No |
| DNS propagation | 15 min - 24 hours | No |
| Azure configuration | 5-15 minutes | Yes |

**Total realistic timeline: 1-3 days** due to manual DNS approval processes.

## Microsoft Internal DNS Portals

The workflow uses these Microsoft internal portals:

1. **CNAME Availability Check**: https://prod.msftdomains.com/Management/OwnerLookup
2. **DNS Request Form**: https://prod.msftdomains.com/Dns/Form/

Both CNAME and TXT records are requested via the DNS Form.

## Success Criteria

Setup is complete when all four criteria are met:

1. ✅ `curl -I https://myapp.microsoft.com` returns HTTP 200
2. ✅ Certificate is valid (no browser warnings)
3. ✅ Application loads correctly
4. ✅ No HTTP 431 errors

## Examples

### Basic Setup

```bash
# 1. Run the recipe
amplifier recipes execute dev-ops:recipes/azure-cname-setup.yaml \
  context='{"container_app_name": "semantic-workbench", "resource_group": "rg-semantic-workbench", "custom_domain": "workbench.microsoft.com"}'

# 2. Follow the prompts:
#    - Verify prerequisites
#    - Request CNAME (manual)
#    - Approve stage after CNAME is active
#    - Azure adds custom domain
#    - Request TXT (manual)
#    - Approve stage after TXT is active
#    - Certificate binding
#    - Verification
#    - Success!
```

### Quick Commands Only

```bash
# Get default FQDN
az containerapp show \
  --name myapp \
  --resource-group my-rg \
  --query properties.configuration.ingress.fqdn -o tsv

# Add custom domain
az containerapp hostname add \
  --hostname myapp.microsoft.com \
  --name myapp \
  --resource-group my-rg

# Bind certificate
az containerapp hostname bind \
  --hostname myapp.microsoft.com \
  --name myapp \
  --resource-group my-rg \
  --environment my-env \
  --validation-method CNAME

# Test
curl -I https://myapp.microsoft.com
```

### Troubleshooting HTTP 431

```bash
# Fix HTTP 431 (Request Header Fields Too Large)
az containerapp update \
  --name myapp \
  --resource-group my-rg \
  --set-env-vars "WEBSITE_AUTH_DISABLE_IDENTITY_FLOW=true"

# Re-test
curl -I https://myapp.microsoft.com
```

## Documentation Cross-References

All documentation is cross-referenced and consistent:

- **Agent** (`agents/cname-orchestrator.md`) → References context files
- **Context** (`context/azure-cname-setup.md`) → Complete standalone guide
- **Skill** (`skills/azure-cname-setup/SKILL.md`) → Quick reference, links to recipe
- **Recipe** (`recipes/azure-cname-setup.yaml`) → Uses cname-orchestrator agent
- **Awareness** (`context/devops-awareness.md`) → Lists all CNAME capabilities

## Testing & Validation

### YAML Validation

All YAML files have been validated:
- ✅ `behaviors/dev-ops.yaml` - Valid YAML
- ✅ `recipes/azure-cname-setup.yaml` - Valid YAML

### File Structure

```
agents/
  cname-orchestrator.md          # New agent
  methodology-expert.md
  pr-orchestrator.md
  workflow-coach.md

context/
  azure-cname-setup.md           # New comprehensive guide
  azure-deployment-patterns.md
  devops-awareness.md            # Updated with CNAME info
  ...

recipes/
  azure-cname-setup.yaml         # New staged recipe
  dev-to-pr.yaml

skills/
  azure-cname-setup/             # New skill
    SKILL.md
  azure-deployment-guide/
    SKILL.md
  ...

behaviors/
  dev-ops.yaml                   # Updated to include cname-orchestrator
```

## Next Steps

1. **Test the recipe** with a real Container App deployment
2. **Gather user feedback** on the approval gate UX
3. **Add examples** to documentation based on real usage
4. **Consider automating** TXT verification polling instead of manual approval

## Contributing

To improve the CNAME setup experience:

1. **Add new troubleshooting scenarios** to `context/azure-cname-setup.md`
2. **Enhance the orchestrator** with better diagnostic commands
3. **Improve timeline accuracy** based on real approval data
4. **Add verification steps** to the recipe based on edge cases

## Questions or Issues?

For help with CNAME setup:

1. **Load the skill**: `load_skill(skill_name="azure-cname-setup")`
2. **Delegate to agent**: `delegate(agent="dev-ops:agents/cname-orchestrator", instruction="...")`
3. **Run the recipe**: Use `azure-cname-setup.yaml` for guided workflow
4. **Read the context**: `@dev-ops:context/azure-cname-setup.md` for complete details

---

**Created:** 2026-05-04  
**Bundle:** azure-bundle-dev-ops v1.0.0  
**Author:** Amplifier AI Assistant

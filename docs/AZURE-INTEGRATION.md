# Azure Integration Guide

This bundle integrates proven Azure deployment patterns from real production deployments.

## What Was Analyzed

### Resource Groups Examined
- **my-app-rg** — Production application deployment (API + Web)
- **my-onboarding-rg** — Production onboarding deployment

### Components Analyzed
- Container Apps (my-api, my-web, my-onboarding-web)
- Container Registries (myappcr, myonboardingcr)
- Application Insights (my-app-insights, my-onboarding-insights)
- Key Vault (my-project-kv with RBAC)
- Managed Identities (system-assigned for ACR pull)
- GitHub Workflows (my-api-build-deploy.yml)

## The 7 Proven Patterns

### 1. Managed Identity for ACR Pull
**What it is:** System-assigned managed identity with AcrPull role assignment.

**Why it matters:**
- Zero credential storage (no passwords anywhere)
- Automatic rotation (managed by Azure)
- Least privilege (AcrPull only, not AcrPush)

**Evidence:** Both container apps use `identity: "system-environment"` with AcrPull role assignments on their respective ACRs.

### 2. Secrets vs Environment Variables
**What it is:** Sensitive data stored as Container App secrets, referenced via `secretref:` in environment variables.

**Pattern:**
```yaml
secrets:
  - db-password: "actual-password-value"
  - api-key: "actual-key-value"
  - telemetry-connection-string: "InstrumentationKey=..."

environment_variables:
  - DATABASE_PASSWORD: secretref:db-password       # SENSITIVE
  - API_KEY: secretref:api-key                     # SENSITIVE
  - DATABASE_HOST: db.postgres.database.azure.com  # NOT SENSITIVE
  - LOG_LEVEL: info                                # NOT SENSITIVE
```

**Evidence:** The production API has 8 secrets (passwords, keys, connection strings) and 14 plain environment variables (hosts, ports, flags).

### 3. OIDC Authentication
**What it is:** GitHub Actions authenticate to Azure using workload identity federation (no service principal passwords).

**Why it matters:**
- No passwords to rotate
- Scoped to specific branches/environments
- Automatic token expiration
- Auditable in Azure AD logs

**Evidence:** The production build-deploy workflow uses `azure/login@v2` with `client-id`, `tenant-id`, and `subscription-id` (OIDC tokens), no password secrets.

### 4. Application Insights Integration
**What it is:** Connection string stored as secret, telemetry configuration via environment variables.

**Pattern:**
```yaml
secrets:
  - telemetry-connection-string: "InstrumentationKey=...;IngestionEndpoint=..."

environment_variables:
  - TELEMETRY_APP_INSIGHTS_CONNECTION_STRING: secretref:telemetry-connection-string
  - TELEMETRY_ENVIRONMENT: production
  - TELEMETRY_SAMPLE_RATE: 0.1
  - TELEMETRY_ENABLE_DEV_LOGGER: false
```

**Evidence:** Both Application Insights instances follow this pattern.

### 5. Blue-Green Deployment
**What it is:** Deploy new revision with 0% traffic → smoke test → promote to 100%.

**Flow:**
1. Build and push Docker image (`github.sha` tag)
2. Deploy new revision with 0% traffic
3. Run comprehensive smoke tests against revision-specific FQDN
4. PASS → Promote to 100% traffic
5. FAIL → Deactivate new revision (production unchanged)

**Evidence:** The production build-deploy workflow implements complete blue-green flow with revision management and traffic shifting.

### 6. Comprehensive Smoke Tests
**What it is:** 5-stage smoke test suite verifying infrastructure, APIs, WebSocket, and LLM integration.

**Stages:**
1. **Health + Infrastructure** — /health, /version, /smoke-tests/quick
2. **Config + Session** — Create config, create session, verify round-trip
3. **Memory Endpoints** — Test profile and context APIs
4. **WebSocket** — Verify WebSocket upgrade handshake
5. **LLM Integration** — Send message, verify LLM response, check history

**Evidence:** The production build-deploy workflow has all 5 smoke test stages, takes ~3 minutes total.

### 7. GitHub Secrets Management
**What it is:** Structured approach to what goes in GitHub Secrets vs workflow environment variables.

**GitHub Secrets (sensitive):**
- AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID (OIDC)
- ACR_REGISTRY (registry URL)
- DATABASE_PASSWORD, ANTHROPIC_API_KEY, CONFIG_ENCRYPTION_KEY, JWT_SECRET_KEY
- APP_INSIGHTS_CONNECTION_STRING

**Workflow Env Vars (non-sensitive):**
- ACR_NAME, IMAGE_NAME, CONTAINER_APP_NAME, RESOURCE_GROUP
- DATABASE_HOST, DATABASE_USER, DATABASE_NAME

**Evidence:** The production build-deploy workflow uses this exact pattern.

## Integration with Azure ZAP

The bundle includes [amplifier-bundle-azure-zap](https://github.com/your-org/amplifier-bundle-azure-zap) for intelligent deployment orchestration.

### Azure ZAP Capabilities
- **project-analyzer** — Examines codebase to determine Azure deployment needs
- **azure-mcp-expert** — Authoritative consultant for Azure MCP Server
- **azure-task-planner** — Creates multi-phase deployment plans
- **azure-task-watchdog** — Validates budget, quotas, and production safeguards
- **azure-task-executor** — Executes plans using Azure MCP tools
- **recipe-generator** — Converts successful deployments to reusable recipes

### When to Use What

| Task | Use |
|------|-----|
| Learn Azure patterns | `load_skill(skill_name="azure-deployment")` |
| Reference pattern docs | `@dev-ops:context/azure-deployment-patterns.md` |
| Deploy from scratch | Natural language to Azure ZAP agents |
| Manual setup guidance | Azure deployment skill (step-by-step) |

## Files Created

### Context Files
- **context/azure-deployment-patterns.md** — 7 proven patterns with code examples

### Skills
- **skills/azure-deployment-guide/SKILL.md** — Step-by-step deployment walkthrough

### Documentation
- **docs/AZURE-INTEGRATION.md** — This file
- **README.md** — Updated with Azure integration section

### Bundle Configuration
- **bundle.md** — Updated to include azure-zap bundle dependency

## Using the Azure Integration

### Option 1: Guided Setup (Skill)

```bash
amplifier chat
> load_skill(skill_name="azure-deployment")
# Follow step-by-step instructions
```

### Option 2: Reference Patterns (Context)

```bash
# Reference in your own work
@dev-ops:context/azure-deployment-patterns.md

# Specific pattern examples
# Pattern 1: Managed Identity setup
# Pattern 2: Secrets vs env vars
# Pattern 5: Blue-green deployment
```

### Option 3: Intelligent Deployment (Azure ZAP)

```bash
amplifier chat
> Deploy my Node.js API to Azure with PostgreSQL

# Azure ZAP orchestrates:
# 1. Project analysis
# 2. Service recommendations
# 3. Cost estimation
# 4. Safety validation
# 5. Phased execution
```

## Team Adoption

### For Developers
1. **First deployment:** Use azure-deployment skill for guided setup
2. **Reference:** Keep azure-deployment-patterns.md open during development
3. **Automation:** Use Azure ZAP for subsequent deployments

### For DevOps Engineers
1. **Review patterns:** Understand the 7 proven patterns
2. **Customize workflows:** Adapt the build-deploy workflow template
3. **Team training:** Walk through azure-deployment skill in group session

### For Architects
1. **Pattern compliance:** Ensure new deployments follow the 7 patterns
2. **Security review:** Verify managed identity + OIDC + secrets management
3. **Cost monitoring:** Use Azure Advisor + Application Insights for optimization

## Success Metrics

Track these to measure Azure deployment quality:

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Credential storage | 0 | No passwords in env vars or code |
| Managed identity usage | 100% | All Container Apps use MI for ACR |
| OIDC adoption | 100% | All workflows use workload identity |
| Blue-green deployments | 100% | All workflows use 0% → 100% pattern |
| Smoke test coverage | 5 stages | All 5 stages passing before promotion |
| App Insights integration | 100% | All apps report telemetry |
| Deployment failures | <5% | Monitor workflow success rate |

---

**This integration is based on real production deployments.** The patterns are proven, not theoretical.

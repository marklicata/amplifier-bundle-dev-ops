# Azure Deployment Patterns for Container Apps

Common patterns for ANY application deploying to Azure Container Apps,
extracted from real production deployments.

## The Azure Stack

Every production deployment includes these components:

```
Container Apps Environment
  ├─ Container App(s) — Your application
  ├─ Container Registry (ACR) — Docker images
  ├─ Application Insights — Telemetry and monitoring
  ├─ Key Vault (optional) — Secrets management
  ├─ Log Analytics Workspace — Centralized logging
  └─ Managed Identity — Secure ACR access
```

## Pattern 1: Managed Identity for ACR Pull

**Problem:** Container Apps need to pull Docker images from ACR without storing credentials.

**Solution:** System-assigned managed identity with AcrPull role.

### Setup (Azure CLI)

```bash
# 1. Enable system-assigned managed identity on Container App
az containerapp identity assign \
  --name your-app \
  --resource-group your-rg \
  --system-assigned

# 2. Get the principal ID
PRINCIPAL_ID=$(az containerapp show \
  --name your-app \
  --resource-group your-rg \
  --query identity.principalId -o tsv)

# 3. Assign AcrPull role to the ACR
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role AcrPull \
  --scope /subscriptions/YOUR_SUB_ID/resourceGroups/your-rg/providers/Microsoft.ContainerRegistry/registries/youracr

# 4. Configure Container App to use managed identity for registry auth
az containerapp update \
  --name your-app \
  --resource-group your-rg \
  --registry-server youracr.azurecr.io \
  --registry-identity system
```

### In GitHub Workflows

```yaml
- name: Deploy Container App
  run: |
    az containerapp update \
      --name ${{ env.CONTAINER_APP_NAME }} \
      --resource-group ${{ env.RESOURCE_GROUP }} \
      --image ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
    # No registry username/password needed — uses managed identity
```

**Why this works:**
- No credentials stored anywhere
- Automatic rotation (managed by Azure)
- Least privilege (AcrPull only, not AcrPush)
- Works seamlessly in GitHub Actions

---

## Pattern 2: Environment Variables vs Secrets

**Rule:** Sensitive data = secret. Everything else = environment variable.

### What Goes Where

| Data Type | Storage | Example |
|-----------|---------|---------|
| **Database passwords** | Secret | `secretref:db-password` |
| **API keys** | Secret | `secretref:anthropic-api-key` |
| **JWT secrets** | Secret | `secretref:jwt-secret-key` |
| **Encryption keys** | Secret | `secretref:config-encryption-key` |
| **App Insights connection string** | Secret | `secretref:telemetry-connection-string` |
| **Database host** | Env var | `DATABASE_HOST=db.postgres.database.azure.com` |
| **Database port** | Env var | `DATABASE_PORT=5432` |
| **Feature flags** | Env var | `AUTH_REQUIRED=false` |
| **Log level** | Env var | `LOG_LEVEL=info` |
| **Environment name** | Env var | `TELEMETRY_ENVIRONMENT=production` |

### Setting Secrets in GitHub Workflows

```yaml
# Step 1: Update Container App secrets
- name: Set Container App secrets
  run: |
    az containerapp secret set \
      --name ${{ env.CONTAINER_APP_NAME }} \
      --resource-group ${{ env.RESOURCE_GROUP }} \
      --secrets \
        db-password="${{ secrets.DATABASE_PASSWORD }}" \
        anthropic-api-key="${{ secrets.ANTHROPIC_API_KEY }}" \
        config-encryption-key="${{ secrets.CONFIG_ENCRYPTION_KEY }}" \
        telemetry-connection-string="${{ secrets.APP_INSIGHTS_CONNECTION_STRING }}"

# Step 2: Reference secrets in environment variables
- name: Deploy with environment variables
  run: |
    az containerapp update \
      --name ${{ env.CONTAINER_APP_NAME }} \
      --resource-group ${{ env.RESOURCE_GROUP }} \
      --set-env-vars \
        "DATABASE_PASSWORD=secretref:db-password" \
        "ANTHROPIC_API_KEY=secretref:anthropic-api-key" \
        "DATABASE_HOST=${{ env.DATABASE_HOST }}" \
        "DATABASE_PORT=5432" \
        "LOG_LEVEL=info"
```

**Critical:** Use `secretref:secret-name` format to reference secrets in environment variables.

---

## Pattern 3: OIDC Authentication for GitHub Actions

**Problem:** GitHub workflows need to authenticate to Azure without storing service principal passwords.

**Solution:** OpenID Connect (OIDC) workload identity federation.

### Setup (One-time per repository)

```bash
# 1. Create Azure AD app registration
az ad app create --display-name "GitHub-OIDC-YourRepo"

# 2. Get the app ID
APP_ID=$(az ad app list --display-name "GitHub-OIDC-YourRepo" --query "[0].appId" -o tsv)

# 3. Create service principal
az ad sp create --id $APP_ID

# 4. Get service principal object ID
SP_OBJECT_ID=$(az ad sp show --id $APP_ID --query id -o tsv)

# 5. Create federated credential for GitHub Actions
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-main-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# IMPORTANT — if your GitHub org enforces immutable claims (e.g. the `microsoft`
# org does), the subject above will not match. GitHub will present a claim of
# the form:
#   repository_owner_id:<NNN>:repository_id:<NNN>:ref:refs/heads/main
# and you'll get AADSTS700213 "No matching federated identity record found".
#
# Fix: read the exact subject from the failing workflow run's logs (the
# azure/login step prints "Federated token details: subject claim - ..."),
# then create the federated credential with that literal string.
# In the Azure portal, you must pick the "Other issuer" scenario — the
# "GitHub Actions" preset hard-codes the name-based format and won't accept
# the ID-based one. CLI works for both formats. Don't try to disable
# immutable claims; the setting exists so renamed repos can't impersonate.

# 6. Assign permissions (Contributor on resource group)
az role assignment create \
  --assignee $SP_OBJECT_ID \
  --role Contributor \
  --scope /subscriptions/YOUR_SUB_ID/resourceGroups/your-rg

# 7. Save these as GitHub secrets:
# - AZURE_CLIENT_ID (the APP_ID)
# - AZURE_TENANT_ID (your tenant ID)
# - AZURE_SUBSCRIPTION_ID (your subscription ID)
```

### In GitHub Workflows

```yaml
permissions:
  id-token: write  # CRITICAL for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      # Now authenticated — use az CLI commands
```

**Why OIDC over service principal secrets:**
- No passwords to rotate
- Scoped to specific branches/environments
- Automatic expiration (tokens are short-lived)
- Auditable (Azure AD logs all authentications)

---

## Pattern 4: Application Insights Integration

**Problem:** Need comprehensive telemetry without exposing connection strings in environment variables.

**Solution:** Store connection string as secret, configure via environment variable reference.

### Setup

```bash
# 1. Create Application Insights
az monitor app-insights component create \
  --app your-app-insights \
  --location eastus \
  --resource-group your-rg \
  --application-type web

# 2. Get connection string
CONN_STR=$(az monitor app-insights component show \
  --app your-app-insights \
  --resource-group your-rg \
  --query connectionString -o tsv)

# 3. Store as Container App secret
az containerapp secret set \
  --name your-app \
  --resource-group your-rg \
  --secrets telemetry-connection-string="$CONN_STR"

# 4. Reference in environment variables
az containerapp update \
  --name your-app \
  --resource-group your-rg \
  --set-env-vars \
    "TELEMETRY_APP_INSIGHTS_CONNECTION_STRING=secretref:telemetry-connection-string" \
    "TELEMETRY_ENVIRONMENT=production" \
    "TELEMETRY_SAMPLE_RATE=0.1" \
    "TELEMETRY_ENABLE_DEV_LOGGER=false"
```

### Configuration Pattern

| Environment Variable | Purpose | Example Value |
|---------------------|---------|---------------|
| `TELEMETRY_APP_INSIGHTS_CONNECTION_STRING` | Connection (secret) | `secretref:telemetry-connection-string` |
| `TELEMETRY_ENVIRONMENT` | Environment tag | `production`, `staging`, `dev` |
| `TELEMETRY_SAMPLE_RATE` | Sampling percentage | `0.1` (10%), `1.0` (100%) |
| `TELEMETRY_ENABLE_DEV_LOGGER` | Console logging | `false` (prod), `true` (dev) |

**Python SDK Usage:**

```python
from azure.monitor.opentelemetry import configure_azure_monitor
import os

# Reads TELEMETRY_APP_INSIGHTS_CONNECTION_STRING automatically
configure_azure_monitor(
    connection_string=os.environ.get("TELEMETRY_APP_INSIGHTS_CONNECTION_STRING"),
    sampling_rate=float(os.environ.get("TELEMETRY_SAMPLE_RATE", "0.1"))
)
```

---

## Pattern 5: Blue-Green Deployment (Staging → Production)

**Problem:** Zero-downtime deployments with verification before production traffic.

**Solution:** Deploy to 0% traffic revision, smoke test, then promote to 100%.

### The Flow

```
Build & Push Docker Image
  ↓
Deploy New Revision (0% traffic)
  ↓
Run Comprehensive Smoke Tests
  ↓
PASS → Promote to 100% traffic
  ↓
Deactivate Old Revisions

FAIL → Deactivate New Revision (production unchanged)
```

### GitHub Workflow Implementation

```yaml
# Job 1: Build and push
build-and-push:
  steps:
    - name: Build and push Docker image
      run: |
        docker build -t $ACR.azurecr.io/myapp:${{ github.sha }} .
        docker push $ACR.azurecr.io/myapp:${{ github.sha }}

# Job 2: Deploy to staging revision (0% traffic)
deploy-staging:
  needs: build-and-push
  steps:
    - name: Deploy new revision with 0% traffic
      run: |
        # WARNING: deriving the suffix from github.sha means re-running the
        # workflow on the SAME commit is a silent no-op. Container Apps
        # requires unique revision suffixes; the second `az containerapp update`
        # call sees the suffix already exists and does not roll a new revision.
        # The build & push steps run, ACR happily overwrites the :sha tag,
        # but the running replica keeps serving the OLD image. This is the
        # most common "I redeployed and nothing changed" trap.
        #
        # To force a fresh revision without a code change, push an empty commit:
        #   git commit --allow-empty -m "chore: redeploy" && git push
        # Or append a build counter / timestamp to the suffix so re-runs differ:
        #   REVISION_SUFFIX="$(echo ${{ github.sha }} | cut -c1-8)-${{ github.run_number }}"
        REVISION_SUFFIX=$(echo ${{ github.sha }} | cut -c1-8)
        REVISION_NAME="myapp--${REVISION_SUFFIX}"
        
        az containerapp update \
          --name myapp \
          --resource-group my-rg \
          --image $ACR.azurecr.io/myapp:${{ github.sha }} \
          --revision-suffix "$REVISION_SUFFIX"
        
        # Ensure 0% traffic on new revision
        CURRENT_REV=$(az containerapp revision list \
          --name myapp \
          --resource-group my-rg \
          --query "[?properties.trafficWeight > \`0\`].name | [0]" -o tsv)
        
        az containerapp ingress traffic set \
          --name myapp \
          --resource-group my-rg \
          --revision-weight "$CURRENT_REV=100" "$REVISION_NAME=0"

# Job 3: Smoke test staging revision
smoke-test:
  needs: deploy-staging
  steps:
    - name: Test health endpoint
      run: curl -sf "$REVISION_URL/health"
    
    - name: Test API endpoints
      run: |
        # Your smoke tests here
    
    - name: Rollback on failure
      if: failure()
      run: |
        az containerapp revision deactivate \
          --name myapp \
          --resource-group my-rg \
          --revision "$REVISION_NAME"

# Job 4: Promote to production
promote-to-production:
  needs: [deploy-staging, smoke-test]
  steps:
    - name: Shift 100% traffic to new revision
      run: |
        az containerapp ingress traffic set \
          --name myapp \
          --resource-group my-rg \
          --revision-weight "$REVISION_NAME=100"
    
    - name: Deactivate old revisions
      run: |
        az containerapp revision list \
          --name myapp \
          --resource-group my-rg \
          --query "[?name!='$REVISION_NAME' && properties.active].name" \
          --output tsv | while read OLD_REV; do
            az containerapp revision deactivate \
              --name myapp \
              --resource-group my-rg \
              --revision "$OLD_REV"
          done
```

**Why this works:**
- Production traffic never touches new revision until tests pass
- Smoke tests run against revision-specific FQDN
- Rollback is instant (just deactivate new revision)
- Old revision stays running until new one is verified

---

## Pattern 6: Comprehensive Smoke Tests

**What to test:** Infrastructure, config, API endpoints, WebSocket, LLM integration.

### The 5 Smoke Test Stages

```yaml
smoke-test:
  steps:
    # Stage 1: Health and infrastructure
    - name: "Smoke 1: Health check"
      run: |
        curl -sf "$TEST_URL/health" | jq '.status == "healthy"'
        curl -sf "$TEST_URL/version"
        curl -sf "$TEST_URL/smoke-tests/quick"

    # Stage 2: Config and session round-trip
    - name: "Smoke 2: Config persistence"
      run: |
        CONFIG_ID=$(curl -sf -X POST "$TEST_URL/configs" -d "$CONFIG_JSON" | jq -r '.config_id')
        curl -sf "$TEST_URL/configs/$CONFIG_ID"
        SESSION_ID=$(curl -sf -X POST "$TEST_URL/sessions" -d "{\"config_id\": \"$CONFIG_ID\"}" | jq -r '.session_id')
        curl -sf -X DELETE "$TEST_URL/sessions/$SESSION_ID"
        curl -sf -X DELETE "$TEST_URL/configs/$CONFIG_ID"

    # Stage 3: Memory endpoints
    - name: "Smoke 3: Memory API"
      run: |
        curl -sf "$TEST_URL/memory/users/ci-test/profile"
        curl -sf "$TEST_URL/memory/users/ci-test/context"

    # Stage 4: WebSocket connectivity
    - name: "Smoke 4: WebSocket"
      run: |
        # Test WebSocket upgrade handshake
        curl -sf -H "Connection: Upgrade" -H "Upgrade: websocket" \
          "$TEST_URL/sessions/$SESSION_ID/ws"

    # Stage 5: LLM message round-trip
    - name: "Smoke 5: LLM integration"
      run: |
        RESPONSE=$(curl -sf -X POST "$TEST_URL/sessions/$SESSION_ID/messages" \
          -d '{"message": "Reply with: SMOKE_TEST_OK"}' --max-time 60)
        echo "$RESPONSE" | grep -q "SMOKE_TEST_OK"
```

**Test design principles:**
1. **Fast** — All 5 stages complete in <3 minutes
2. **Independent** — Each stage cleans up its own resources
3. **Non-blocking** — Warnings for expected failures (WebSocket 404 on staging)
4. **Evidence-based** — Verify actual response content, not just HTTP codes

---

## Pattern 7: GitHub Secrets Management

**What to store as GitHub secrets:**

| Secret Name | Value | Used For |
|-------------|-------|----------|
| `AZURE_CLIENT_ID` | App registration client ID | OIDC auth |
| `AZURE_TENANT_ID` | Your Azure AD tenant ID | OIDC auth |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID | OIDC auth |
| `ACR_REGISTRY` | `youracr.azurecr.io` | Docker registry |
| `DATABASE_PASSWORD` | Postgres password | Container App secret |
| `ANTHROPIC_API_KEY` | LLM API key | Container App secret |
| `CONFIG_ENCRYPTION_KEY` | Encryption key | Container App secret |
| `JWT_SECRET_KEY` | JWT signing key | Container App secret |
| `APP_INSIGHTS_CONNECTION_STRING` | Telemetry connection string | Container App secret |

**NOT secrets (use env vars in workflow):**
- ACR name (just the name, not credentials)
- Resource group name
- Container App name
- Database host/port/user (non-sensitive)

### One identity per secret name

`AZURE_CLIENT_ID` is the *deployment* service principal — the workload identity
that GitHub federates to for `azure/login`. It is **not** a user-facing app
registration. If you have a SPA / web app where users sign in, that app has its
own client ID and must be a separate secret:

| Secret Name | Identity | Used For |
|-------------|----------|----------|
| `AZURE_CLIENT_ID` | Deployment SP (workload identity) | `azure/login@v2` OIDC |
| `PUBLIC_APP_CLIENT_ID` | User-facing app registration | Baked into SPA bundle, MSAL config, Easy Auth |

**Why this matters:** the two identities need different things — the deployment
SP needs RBAC on Azure resources (AcrPush, Container Apps Contributor); the
user-facing app needs redirect URIs, delegated Graph scopes, and an SPA
platform registration. Reusing one secret name for both means a Vite build
that pipes `secrets.AZURE_CLIENT_ID` into `VITE_APP_ID` will bake the
deployment SP's GUID into the browser bundle. Users then send that GUID to
`/authorize`, and AAD correctly rejects them with AADSTS50011 ("no redirect
URI for `https://your-app.com`") — because the deployment SP has no redirect
URIs and never will.

Combining them into one app is technically possible but discouraged: a leaked
user token then also unlocks ACR push rights.

---

## Pattern 8: Build-Time Config Inlining (SPA Frameworks)

**Problem:** Vite, Next.js, Create React App, Astro, and similar frameworks
inline environment variables into the compiled JS bundle at *build time*. After
`npm run build` / `docker build`, those values are baked into the static assets.
Changing the source secret has zero effect on production until the image is
rebuilt **and** a new revision actually rolls.

This breaks the intuition that "config = runtime" and produces a category of
bug where you change a secret, redeploy, see green checkmarks, and the app
still misbehaves — because users are being served the JS bundle that was
compiled with the old value.

### Where the values get baked in

```dockerfile
# apps/web/Dockerfile
FROM node:20
ARG VITE_APP_ID                      # ← received from --build-arg
ARG VITE_TENANT_ID
ENV VITE_APP_ID=${VITE_APP_ID}       # ← available to `npm run build`
ENV VITE_TENANT_ID=${VITE_TENANT_ID}
COPY . .
RUN npm ci && npm run build          # ← Vite reads VITE_* env, inlines into JS
```

```yaml
# docker-compose.prod.yml — forwards host env to the build
services:
  web:
    build:
      context: .
      args:
        VITE_APP_ID: ${VITE_APP_ID}
        VITE_TENANT_ID: ${VITE_TENANT_ID}
```

```yaml
# .github/workflows/deploy.yml — workflow → compose → Dockerfile → bundle
- name: Build image
  env:
    VITE_APP_ID: ${{ secrets.PUBLIC_APP_CLIENT_ID }}
    VITE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  run: docker compose -f docker-compose.prod.yml build
```

### The diagnostic that actually tells you what's deployed

When something looks wrong in production, do not trust the source tree — it
only tells you what *could* be built. Inspect the running container:

```bash
# Portal → Container App → Console → pick a replica → /bin/sh
grep -roh '<expected-value>\|<wrong-value>' /app | sort -u

# Or, if you don't know where the bundle lives:
find / -name "*.js" -path "*assets*" 2>/dev/null | head
```

If the wrong value appears, the image is stale or was built with the wrong
secret. The Azure-side config (Authentication blade, Container App env vars)
is irrelevant — the browser sends whatever was compiled in.

### Why this combines lethally with Pattern 5

Pattern 5's revision-suffix scheme means that *even after* you fix the source
secret and re-run the workflow, the running container may keep serving the
old image. You need *both*: a fresh build (correct secret) *and* a new revision
suffix (new SHA or unique suffix). Check both before declaring victory.

### Order of operations when fixing a stale-bundle bug

1. Confirm the wrong value is actually in the deployed bundle (grep the
   container — see above). Don't fix what isn't broken.
2. Update the GitHub secret to the correct value.
3. Trigger a fresh build *with a new commit* (empty commit is fine).
4. After the deploy, exec back into a *new* replica and grep again to
   confirm the correct value is now baked in.
5. Have one user test in an incognito window — browser disk cache and service
   workers can still serve old hashed chunks even after the image rolls.

---

## Quick Reference: Common Commands

```bash
# Check Container App status
az containerapp show --name myapp --resource-group my-rg

# List revisions and traffic split
az containerapp revision list \
  --name myapp \
  --resource-group my-rg \
  --query "[].{name:name, active:properties.active, traffic:properties.trafficWeight}"

# View Container App logs (last 100 lines)
az containerapp logs show \
  --name myapp \
  --resource-group my-rg \
  --tail 100

# Get Application Insights connection string
az monitor app-insights component show \
  --app my-app-insights \
  --resource-group my-rg \
  --query connectionString -o tsv

# List Container App secrets (names only, not values)
az containerapp secret list \
  --name myapp \
  --resource-group my-rg

# Check ACR role assignments
az role assignment list \
  --scope /subscriptions/SUB_ID/resourceGroups/my-rg/providers/Microsoft.ContainerRegistry/registries/myacr

# Get managed identity principal ID
az containerapp show \
  --name myapp \
  --resource-group my-rg \
  --query identity.principalId -o tsv
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Correct Pattern |
|--------------|-------------|-----------------|
| Storing passwords in environment variables | Visible in logs, config | Use secrets with secretref |
| Using admin ACR credentials | Over-permissioned, rotatable | Managed identity with AcrPull |
| Service principal passwords in GitHub | Rotation burden, security risk | OIDC federation |
| Deploying straight to 100% traffic | No rollback window | Blue-green with 0% → 100% |
| Skipping smoke tests | Production incidents | Always smoke test before promotion |
| App Insights key in code | Exposed in version control | Secret reference in env vars |
| Manual secret rotation | Human error, forgotten | Managed identities where possible |
| Re-running the deploy workflow expecting a new revision | Same SHA = same revision suffix = `az containerapp update` no-op | Empty commit, or include `github.run_number` in the suffix (Pattern 5) |
| One secret name (`AZURE_CLIENT_ID`) for two identities | Wrong GUID gets baked into the SPA bundle, AADSTS50011 in prod | Separate `AZURE_CLIENT_ID` (deploy SP) and `PUBLIC_APP_CLIENT_ID` (user-facing app) (Pattern 7) |
| Diagnosing a deploy bug by grepping the source tree | Source ≠ what's running; misses build-arg drift entirely | Exec into the replica and grep the served bundle (Pattern 8) |
| Setting env vars on the Container App to fix SPA config | Vite/Next/CRA inlined values at build; runtime env doesn't reach the browser | Rebuild the image with correct build args (Pattern 8) |

---

## For Your Team

**Checklist for every new Azure app:**

- [ ] Container Apps Environment created
- [ ] Container Registry (ACR) created
- [ ] System-assigned managed identity enabled
- [ ] AcrPull role assigned to ACR
- [ ] Application Insights created
- [ ] App Insights connection string stored as secret
- [ ] GitHub OIDC federation configured
- [ ] GitHub secrets configured (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID)
- [ ] Blue-green deployment workflow implemented
- [ ] 5-stage smoke tests written
- [ ] Environment variable pattern documented (secrets vs env vars)

---

**These patterns are proven in production** across real deployments.
Use them as defaults for any new Azure Container Apps deployment.

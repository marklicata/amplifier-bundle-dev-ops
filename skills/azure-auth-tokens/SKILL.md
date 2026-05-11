---
skill: azure-auth-tokens
version: 1.0.0
context: current
description: |
  Definitive reference for Azure authentication and token infrastructure on Container Apps.
  Covers Easy Auth vs MSAL division of responsibility, Container Apps token store gotchas,
  MSAL.js pitfalls, cross-boundary encryption, JWT validation, exclusion paths, secret
  topology, SvelteKit BFF patterns, and deploy hardening. Use when building or debugging
  any Azure web application authentication layer.
  Extracted from 353+ production sessions across Amplifier, Copilot, and Claude (Jan-May 2026).
tags: [azure, authentication, msal, tokens, security, container-apps, sveltekit]
---

# Azure Authentication & Token Infrastructure

Definitive reference for how the Azure platform provides, manages, and secures tokens. Extracted from 353+ production sessions across 3 AI platforms (278 Amplifier, 33 Copilot, 42 Claude). Every pattern here broke prod at least once.

For how Amplifier modules consume these tokens, see `authenticated-tool-patterns`.

---

## 1. Easy Auth vs MSAL: Division of Responsibility

These are layers, not alternatives. Conflating them silently breaks production.

| | Easy Auth | MSAL |
|--|-----------|------|
| **What** | Platform middleware (`authConfigs/current`) | In-app authentication library |
| **Role** | The bouncer. Confirms tenant identity, injects headers | Everything in-app: tokens, scopes, Graph, OBO, refresh |
| **Owns** | `/.auth/login/aad/callback`, `x-ms-token-aad-id-token` header | Token acquisition, silent refresh, popup/redirect login |
| **Level** | Infrastructure (Azure platform) | Application code |

**Production Nexus uses both.** Easy Auth gates the door; MSAL handles everything inside.

### Redirect URI Platform Split (Same App Registration)

A single app registration with TWO platform entries:

| Platform | Used By | Why |
|----------|---------|-----|
| **Web** | Easy Auth callback | `response_mode=form_post` requires Web |
| **SPA** | MSAL.js redirects | PKCE requires SPA |

Mixing them produces `AADSTS9002326`. Both FQDN and CNAME must be under SPA for `PublicClientApplication`.

### Easy Auth Removal

We completed a full Easy Auth removal from the application layer (10 tasks, 6 commits) to consolidate on MSAL-only for in-app auth. Easy Auth remains as the platform gate.

**Production rule:** Never use `az account` in production mode. Gate the dev fallback behind `NODE_ENV !== 'production' && AUTH_MODE !== 'production'`.

---

## 2. Container Apps Token Store (The Silent Prod-Breaker)

Easy Auth only injects `x-ms-token-aad-id-token` when the **token store is enabled**. App Service does this by default; Container Apps does not. Without it you only get `x-ms-client-principal`.

### Requirements

- BYO blob storage + managed identity
- SAS-URL approach is dead if storage has `allowSharedKeyAccess: false` (common in MS-internal subs)
- System-assigned MI often doesn't work -- Easy Auth often requires **user-assigned MI**
- The `--blob-container-identity system` CLI flag generates a bogus user-assigned-identity ID; the canonical value is the literal string `"system"` set via `az rest`/PUT

### Silent Killers

| Setting | Effect |
|---------|--------|
| `WEBSITE_AUTH_DISABLE_IDENTITY_FLOW=true` | Silently strips identity headers -- check first |
| `unauthenticatedClientAction = AllowAnonymous` | Kills `x-ms-token-aad-id-token` for signed-in users too, breaks the entire MSAL/OBO chain |

### Diagnostics via `/.auth/me`

| Response | Meaning |
|----------|---------|
| Empty `[]` | Not authenticated session |
| `404` | Token store is off |
| `500` on `/.auth/login/aad/callback` | Token store can't write to blob (MI permissions or `allowSharedKeyAccess`) |

---

## 3. MSAL Patterns & Pitfalls

### Platform Type

App registration platform MUST be **SPA**, not Web, for MSAL.js + PKCE. Wrong platform produces `AADSTS50011` and an infinite login loop. Both FQDN and CNAME redirect URIs must be under the SPA section.

### Popup Handling

- Dedicated static `/auth/redirect` route (`csr = false`). Never set the popup redirect URI to the SPA root -- MSAL re-instantiates inside the popup, spawns a blocked second popup, and never closes.
- `temporaryCacheLocation: 'localStorage'` required for popup flow -- default `sessionStorage` is per-window and not shared with the popup. Removed in MSAL.js v5 -- pin the version.
- Decision rule: `localhost` -> popup; deployed host -> full-page redirect.

### Build-Time vs Runtime

`VITE_*` is build-time, not runtime. Setting on the running container is a no-op -- must pass as `--build-arg` at `npm run build`. Symptom: empty clientId baked in -> `AADSTS900144`.

`api://...` is a resource URI, NOT a clientId. MSAL needs the bare GUID. Setting `VITE_NEXUS_APP_ID=api://...` causes silent iframe timeouts.

### Stale Scope Cache

MSAL caches tokens by scopes in `localStorage`. After permission changes, `acquireTokenSilent` keeps returning the old token with outdated `scp` claim. Inspect the `scp` claim; force re-auth on mismatch.

### Async SSE Callback Trap

`acquireTokenPopup` from inside async SSE callbacks is blocked (no user gesture). 60s timeout in prod traced to this. **Mitigation:** pre-fetch Graph scope at sign-in click and cache.

### Tauri / WebView2

Isolated cookie store means hidden-iframe SSO times out. Set `iframeBridgeTimeout: 2000` (2s), fall back to popup.

### MSAL Python (Windows)

With Conditional Access requiring a compliant device, device-code flow is blocked:

```python
# Fix: WAM broker
app = msal.PublicClientApplication(
    client_id,
    enable_broker_on_windows=True,       # Required
    parent_window_handle=app.CONSOLE_WINDOW_HANDLE
)
# pip install msal[broker]
# Register custom broker scheme: msal<client-id>://auth
# Persist cache via msal.SerializableTokenCache
```

### az account Scope Limits

`az account get-access-token --resource https://graph.microsoft.com` carries the Azure CLI's own app permissions (`04b07795-...` client) with only `User.Read`. Auth through your own app via MSAL for broader scopes.

### Auth Init Order

MSAL must complete BEFORE `workspace.init()`. Never race auth with API calls. Use `authPhase` state to show "Signing in..." during auth.

---

## 4. SSE-Driven Token Exchange Architecture

The core token exchange pattern. No OBO server flow, no `client_secret` needed. One mechanism for both initial acquisition and refresh.

```
1. Backend tool needs a Graph token for a specific scope
   |
   +--> Emits `token_needed` SSE event with required scope
   |
2. Frontend receives SSE event
   |
   +--> MSAL acquireTokenSilent (or popup fallback)
   |
3. Frontend: POST /sessions/{id}/provide-token
   |
   +--> Backend resolves awaited asyncio.Future
   |
4. Backend tool continues with the scoped token
```

### Critical Infrastructure Rules

| Rule | Why |
|------|-----|
| `SessionManager` MUST be a process-wide singleton | `/stream` and `/provide-token` must share the same in-memory `_sessions`/futures dict. Per-request `Depends` instantiation broke OBO. |
| `bearer_token` must be threaded through `send_message`/`stream_message` | Auto-resumed sessions need to re-attach `UserTokenStore` |
| Use per-scope async locks | Prevents concurrent token requests for the same scope |
| Never use ID tokens for OBO | OBO flow needs an access token |
| Never cache without expiry tracking | Stale tokens cause silent 401s |
| 60s timeout on provide-token | Popup blocked from async SSE callback (no user gesture). Pre-fetch at sign-in click. |

---

## 5. Cross-Boundary Encryption

SvelteKit -> backend headers are AES-256-GCM encrypted with `CONFIG_ENCRYPTION_KEY`.

| Property | Value |
|----------|-------|
| Algorithm | AES-256-GCM |
| Key derivation | PBKDF2, 100k iterations |
| Prefix | `enc:` (allows plain-text passthrough for tests) |
| Encrypted headers | `X-App-ID`, `X-API-Key`, `Authorization` |

### Rules

- **Cache the `ConfigEncryption` instance per key.** Without caching, every request runs 100k PBKDF2 iterations.
- `CONFIG_ENCRYPTION_KEY` must match across both Container Apps. Mismatch -> `"Failed to decrypt authentication headers"` 401.
- Verified bidirectionally (TS <-> Python). Always test both directions.
- Consumers in the codebase: `config_manager.py`, `secrets_encryption.py`, `middleware/auth.py`. Easy to forget when rotating.

---

## 6. JWT Validation

JWT audience must accept BOTH forms:

```python
valid_audiences = [client_id, f"api://{client_id}"]
```

ID tokens vs client-credential tokens use different audience forms. Single-audience validation rejects half the legitimate traffic.

**"Configured != Granted":** Decode the JWT in-browser and verify `aud`, `scp`, `upn`, `exp` before assuming anything is broken. Admin consent is the gating step, not code deployment.

---

## 7. Exclusion Paths (Two Layers)

### Layer 1: Platform (Easy Auth `globalValidation.excludedPaths`)

**Live workspace (web app):**
```
/.well-known
/.well-known/
/.well-known/microsoft-identity-association.json
/api/healthz
/api/healthz/
```

**Live API app:** `/api/healthz` only.

#### Path Matching Rules

| Rule | Detail |
|------|--------|
| Case-sensitive prefix | No case folding |
| No globs | Each path literal |
| No trailing-slash equivalence | List both `/api/healthz` AND `/api/healthz/` |
| Each `.well-known` file enumerated | No wildcard for subdirectory |

#### How to Edit (CLI/Portal Don't Expose This)

```bash
# Use az rest, NOT inline JSON (PowerShell quoting mangles arrays)
az rest --method patch \
  --url "https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.App/containerApps/{app}/authConfigs/current?api-version=2024-03-01" \
  --body @auth-config.json

# Or use resources.azure.com PUT
```

**Send the FULL `globalValidation` block.** Partial PATCH wipes siblings like `unauthenticatedClientAction`.

#### Propagation

Changes apply on next request, but the running revision's auth-proxy may **cache config for ~60 minutes**. Restart the revision to force reload.

#### Outer Layer Override

Front Door fronting your domain can still 401 a path Easy Auth would let through. Apply exclusions at the **outermost layer** too.

### Layer 2: Application (FastAPI `PUBLIC_PATHS`)

```python
PUBLIC_PATHS = ["/health", "/login", "/set-session", "/logout"]
```

| Path | Status |
|------|--------|
| `/health`, `/login`, `/set-session`, `/logout` | Public |
| `/version` | Keep OUT (broke `test_protected_paths_require_auth`) |
| `/docs`, `/redoc`, `/openapi.json` | Gated by admin session cookie, NOT in `PUBLIC_PATHS` |

**Health endpoints behind auth = dead containers.** Azure probes can't authenticate.

---

## 8. microsoft-identity-association

```
Location: apps/workspace/static/.well-known/microsoft-identity-association.json
Body:     { "associatedApplications": [{ "applicationId": "<entra-app-id>" }] }
```

SvelteKit's `sirv` serves `static/` before `hooks.server.ts`, so app-level auth is bypassed automatically. But Easy Auth at the platform layer still gates it -- must be added to `globalValidation.excludedPaths`.

**Validation trick:** Hit `/.well-known/microsoft-identity-association.json`:
- **404** ("no such route") = Easy Auth let it through (good)
- **401** = still gated (bad, update excludedPaths)

---

## 9. Secret Topology

### Backend Container App (6 Secrets, NOT Touched by CI)

| Secret | Purpose |
|--------|---------|
| `DATABASE_PASSWORD` | Database access |
| `CONFIG_ENCRYPTION_KEY` | Header encryption (must match frontend) |
| `ANTHROPIC_API_KEY` | LLM provider |
| `SECRET_KEY` | The actual env var the backend reads |
| `OPENAI_API_KEY` | Optional LLM provider |
| `APP_INSIGHTS_CONNECTION_STRING` | Telemetry |

CI's `--set-env-vars` writes only non-secret config.

### Frontend Container App (7 Vars)

| Var | Purpose |
|-----|---------|
| `NEXUS_API_URL` | Backend URL |
| `NEXUS_APP_ID` | Entra app client ID (bare GUID) |
| `NEXUS_API_KEY` | API key for backend |
| `NEXUS_APP_NAME` | Display name |
| `CONFIG_ENCRYPTION_KEY` | Must match backend |
| `NEXUS_AZURE_TENANT_ID` | Entra tenant |
| `NEXUS_AZURE_RESOURCE` | `api://<app-id>` for token acquisition |

### Aliasing Gotcha

`SECRET_KEY` is what the backend reads. `NEXUS_JWT_SECRET` is a docker-compose alias for the HS256 dev fallback path only. **Drop `NEXUS_JWT_SECRET` in production.**

---

## 10. SvelteKit BFF & Frontend Patterns

### Token Resolution Priority (hooks.server.ts)

```
1. MSAL cookie (msal_id_token) set by /api/auth/token
2. az account get-access-token -- DEV ONLY, double-guarded:
   NODE_ENV !== 'production' && AUTH_MODE !== 'production'
3. Real x-ms-token-aad-id-token from Easy Auth (prod, requires token store)
```

### Rules

| Rule | Why |
|------|-----|
| Don't mutate `event.request` | Read-only. Use `event.locals` declared on `App.Locals` in `app.d.ts` |
| Same-origin proxy at `/nexus-api/[...path]/+server.ts` | No CORS preflight, centralizes token attachment server-side |
| Use `execFile` (argv array), not `exec` | Avoids shell parsing when calling `az` |
| Never run az fallback in production | `az` isn't installed in containers |

### Other Frontend Rules

- **Timezone:** Send `Intl.DateTimeFormat().resolvedOptions().timeZone` so the AI knows the user's local time.
- **Cookie `secure` flag:** Depends on `AUTH_MODE`. Workflow vs deploy script can set different values -- keep consistent.
- **`authPhase` state:** Show "Signing in..." during MSAL init. Never show app UI before auth completes.

---

## 11. Container Apps Deploy Patterns

### Revision Management

```bash
# Set single-revision mode (once, separate prerequisite)
az containerapp revision set-mode --mode single
# az containerapp update has NO --mode flag

# Multi-revision traffic: ALL revisions in one call (omitted = unchanged)
az containerapp ingress traffic set \
  --revision-weight rev1=80 rev2=20

# Never reuse a revision suffix ("latest" collides). Use commit SHA.
```

### KEDA Scaler

KEDA HTTP scaler is incompatible with 0%-traffic canary deployments (`ScaledObjectCheckFailed`). Use a **CPU scaler**.

### az CLI Hardening

```bash
# Windows/WSL appends \r to --output tsv
az containerapp show ... --output tsv | tr -d '\r'

# POSIX = not == in [ ] tests
[ "$var" = "value" ]    # correct
[ "$var" == "value" ]   # bashism, breaks in sh
```

Container Apps reject non-alphanumeric characters in revision suffixes. The `\r` from `az` output triggers this.

### Docker

- Cached layers mask missing Dockerfile steps. Use `--no-cache` to catch.
- `*.sh` files need LF endings: `*.sh text eol=lf` in `.gitattributes`.

### Source vs Wheel

Source-code edits don't apply if the container runs the installed wheel (`uv pip install --no-deps .`). When a fix "isn't being picked up," check the traceback path for `/venv/lib/...` -- that's the wheel. Run `docker compose build`.

---

## 12. Middleware & Logging

Auth middleware: API key prefix lookup + JWT/JWKS. Bearer token from header into `request.state`, not from body.

```python
# LOG_LEVEL needs .lower() for uvicorn
uvicorn.run(app, log_level=os.getenv("LOG_LEVEL", "info").lower())
```

| Level | Use For |
|-------|---------|
| `DEBUG` | Per-event (individual tool calls, API responses) |
| `INFO` | Lifecycle (session created, auth completed, deploy finished) |

Per-session debug toggle via `?debug=true` query parameter.

---

## 13. Anti-Patterns

| Anti-Pattern | What Happens | Fix |
|--------------|-------------|-----|
| Multiple auth mechanisms simultaneously | Unclear which is active, maintenance nightmare | Pick ONE (MSAL for app-level) |
| `unauthenticatedClientAction = AllowAnonymous` | Strips identity headers for signed-in users | Never set globally |
| Wrong platform type (Web instead of SPA) | `AADSTS50011`, infinite login loop | SPA for MSAL.js, Web for Easy Auth callback only |
| `api://` as clientId | Silent iframe timeout | Use bare GUID |
| MSAL popup redirect = SPA root | Re-instantiates MSAL, blocked popup, never closes | Dedicated static `/auth/redirect` route |
| `acquireTokenPopup` from async SSE | Blocked (no user gesture), 60s timeout | Pre-fetch at sign-in click |
| `az account` in production | `az` not installed in containers, wrong scopes | Easy Auth or MSAL only |
| `VITE_*` set on running container | No-op (build-time only) | Pass as `--build-arg` at build |
| Partial PATCH on `globalValidation` | Wipes siblings like `unauthenticatedClientAction` | Send full block |
| Racing auth with API calls | Intermittent failures | MSAL must complete BEFORE `workspace.init()` |
| Reusing revision suffix "latest" | Collision | Use commit SHA |
| ID token for OBO exchange | Exchange fails silently | Use access token |
| Caching tokens without expiry | Stale tokens, silent 401s | Track `exp`, force re-auth on mismatch |

---

## 14. Troubleshooting Reference

### AADSTS Error Codes

| Code | Cause | Fix |
|------|-------|-----|
| `AADSTS50011` | Redirect URI under Web instead of SPA | Move to SPA platform |
| `AADSTS9002326` | Mixed Web/SPA for same redirect URI | Easy Auth callback under Web, MSAL under SPA |
| `AADSTS900144` | Empty clientId (VITE_* not baked at build) | Pass as `--build-arg` during `npm run build` |
| `AADSTS53000` | Conditional Access requires compliant device | Not an app issue; use WAM broker on Windows |

### HTTP Error Patterns

| Error | Cause | Fix |
|-------|-------|-----|
| 401 "Failed to decrypt authentication headers" | `CONFIG_ENCRYPTION_KEY` mismatch | Sync key across both Container Apps |
| 401 on all requests | `WEBSITE_AUTH_DISABLE_IDENTITY_FLOW=true` | Remove env var |
| 401 on `/.well-known/*` | Still gated by Easy Auth | Add to `globalValidation.excludedPaths` |
| 404 on `/.well-known/*` | Easy Auth let it through | This is GOOD -- exclusion paths work |

### Token Store Diagnostics

```bash
curl -s https://your-app/.auth/me | jq .
# Empty [] = not authenticated
# 404     = token store off
# 500     = MI can't write to blob
```

### Container Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Fix "isn't being picked up" | Runs installed wheel, not source | `docker compose build` |
| Health checks failing | `/healthz` not excluded | Add to Easy Auth + app-level exclusions |
| Cryptic revision suffix error | `\r` in az output | `az ... \| tr -d '\r'` |
| `ScaledObjectCheckFailed` | KEDA HTTP scaler + 0% canary | Switch to CPU scaler |
| Config change not taking effect | Auth-proxy caches ~60min | Restart the revision |

---

## 15. Deployment Checklist

### Azure AD App Registration
- [ ] App registration created
- [ ] SPA platform: redirect URIs for localhost + production (both FQDN and CNAME)
- [ ] Web platform: Easy Auth callback URI only
- [ ] Admin consent granted (requires app-reg owner, not deploy SP)

### Container Apps Token Store
- [ ] Blob storage provisioned
- [ ] User-assigned managed identity created and assigned
- [ ] Token store enabled via `az rest`/PUT (not CLI flags)
- [ ] `allowSharedKeyAccess` checked on storage account
- [ ] `WEBSITE_AUTH_DISABLE_IDENTITY_FLOW` NOT set
- [ ] `/.auth/me` returns user data (not empty `[]`, 404, or 500)

### Exclusion Paths
- [ ] `globalValidation.excludedPaths` configured via `az rest --body @file.json`
- [ ] Full `globalValidation` block sent (not partial PATCH)
- [ ] Both `/api/healthz` and `/api/healthz/` listed
- [ ] All `.well-known` paths enumerated explicitly
- [ ] Front Door / outer layer exclusions match
- [ ] Revision restarted after config change

### Encryption
- [ ] `ConfigEncryption` instance cached (not per-request)
- [ ] Same `CONFIG_ENCRYPTION_KEY` on both Container Apps
- [ ] Bidirectional test (TS -> Python and Python -> TS)
- [ ] `enc:` prefix passthrough works for plain-text in tests

### Frontend
- [ ] MSAL configured with bare GUID (not `api://`)
- [ ] `VITE_*` vars passed as `--build-arg` at build time
- [ ] Dedicated `/auth/redirect` route (static, `csr = false`)
- [ ] `temporaryCacheLocation: 'localStorage'` for popup
- [ ] Auth init completes before any API calls

### Deploy
- [ ] Single-revision mode set
- [ ] Revision suffix uses commit SHA
- [ ] `*.sh` files have LF endings
- [ ] `az` output piped through `tr -d '\r'`
- [ ] Docker built with `--no-cache` for release builds
- [ ] Health checks pass after deploy

---

## 16. Process Patterns

| Pattern | Rule |
|---------|------|
| **Gene transfer** | Fix one place -> find all parallels -> apply everywhere |
| **Docs hygiene** | Audit docs after every major change (stale paths, wrong commands, wrong test counts) |
| **Orphaned code** | Periodic audits. Deleted `nexus-core` (86 MB, never imported) |
| **Iterative** | Not big-bang. Don't fix auth + deploy + logging in one PR |
| **Regression** | "This was working yesterday" = diagnose, don't paper over |

---

**Version:** 1.0.0
**Last Updated:** May 8, 2026
**Based on:** 353+ sessions across Amplifier (278), Copilot (33), Claude (42)

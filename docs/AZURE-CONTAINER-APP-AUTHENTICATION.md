# Azure Container App Authentication Setup Guide

## Overview

This guide documents how to configure Azure Active Directory (Entra ID) authentication for Azure Container Apps using Easy Auth. It's based on analysis of two production applications: Nexus and Amplifier-Onboarding.

## Table of Contents

- [What is Easy Auth?](#what-is-easy-auth)
- [Authentication Architecture](#authentication-architecture)
- [Complete Setup Process](#complete-setup-process)
- [Production Configuration Examples](#production-configuration-examples)
- [Common Patterns & Best Practices](#common-patterns--best-practices)
- [Troubleshooting](#troubleshooting)

---

## What is Easy Auth?

**Easy Auth** (Azure App Service Authentication/Authorization) is a built-in authentication feature in Azure that handles authentication flows without requiring code changes in your application. It:

- Intercepts HTTP requests before they reach your app
- Validates tokens and manages authentication state
- Provides automatic token refresh
- Works with multiple identity providers (Azure AD, Google, Facebook, etc.)

### Key Benefits

1. **No code changes required** - Authentication is handled at the platform level
2. **Automatic token management** - Tokens are validated and refreshed automatically
3. **Built-in session management** - User sessions are maintained by the platform
4. **Multiple providers** - Support for various identity providers out of the box

---

## Authentication Architecture

```
User Request
    ↓
Azure Container App (Front Door)
    ↓
Easy Auth Middleware
    ├─ Unauthenticated? → Redirect to Identity Provider (Entra ID)
    │                      ↓
    │                   User authenticates
    │                      ↓
    │                   Redirect back to /.auth/login/aad/callback
    │                      ↓
    └─ Authenticated? → Add auth headers (X-MS-CLIENT-PRINCIPAL, etc.)
                          ↓
                      Your Application
```

### Authentication Flow

1. **User accesses protected route**: User tries to access your application
2. **Easy Auth intercepts**: Middleware checks for authentication
3. **Redirect to IdP**: If not authenticated, redirects to Entra ID login
4. **User authenticates**: User logs in with Microsoft credentials
5. **Callback**: Entra ID redirects back to `/.auth/login/aad/callback`
6. **Token validation**: Easy Auth validates the token and creates session
7. **Request forwarding**: Authenticated request forwarded to your app with user info headers

---

## Complete Setup Process

### Prerequisites

- Azure subscription
- Azure Container App deployed
- Access to create/manage Entra ID app registrations
- Azure CLI installed

### Step 1: Create App Registration in Entra ID

```bash
# Create new app registration
az ad app create \
  --display-name "my-app-name" \
  --sign-in-audience AzureADMyOrg \
  --web-redirect-uris "https://my-app.azurecontainerapps.io/.auth/login/aad/callback"

# Note the appId from the output - this is your CLIENT_ID
```

**Key parameters:**
- `--display-name`: Human-readable name for the app
- `--sign-in-audience`: 
  - `AzureADMyOrg` = Single tenant (recommended for internal apps)
  - `AzureADMultipleOrgs` = Multi-tenant
- `--web-redirect-uris`: Where Entra ID redirects after authentication

### Step 2: Configure Redirect URIs

The redirect URI **must** be in the format:
```
https://<your-app-fqdn>/.auth/login/aad/callback
```

#### For Production (Web Platform)

```bash
# Get your app's object ID
APP_ID="<your-client-id>"
OBJECT_ID=$(az ad app show --id $APP_ID --query id -o tsv)

# Add production redirect URI
az ad app update --id $OBJECT_ID \
  --web-redirect-uris \
    "https://my-app.azurecontainerapps.io/.auth/login/aad/callback" \
    "https://my-custom-domain.com/.auth/login/aad/callback"
```

#### For Local Development (SPA Platform)

```bash
# Add localhost redirect URIs for local development
az ad app update --id $OBJECT_ID \
  --spa-redirect-uris \
    "http://localhost:4173/auth/redirect" \
    "http://localhost:4173"
```

**Important:** Web vs SPA platform selection affects the OAuth flow type:
- **Web platform**: Uses authorization code flow (server-side)
- **SPA platform**: Uses authorization code flow with PKCE (client-side)

### Step 3: Enable Implicit Grant (if needed)

For applications that need tokens immediately:

```bash
az ad app update --id $OBJECT_ID \
  --enable-access-token-issuance true \
  --enable-id-token-issuance true
```

**When to enable:**
- If your frontend needs immediate access to tokens
- For SPAs that make API calls directly
- When using older OAuth flows

**Best practice:** Modern apps should use authorization code flow with PKCE instead.

### Step 4: Configure Container App Easy Auth

```bash
# Enable authentication on your container app
az containerapp auth update \
  --name my-app-web \
  --resource-group my-app-rg \
  --enabled true \
  --action RedirectToLoginPage \
  --redirect-provider azureactivedirectory \
  --aad-client-id "<your-client-id>" \
  --aad-tenant-id "<your-tenant-id>"
```

**Key parameters:**
- `--enabled true`: Turns on Easy Auth
- `--action RedirectToLoginPage`: What to do with unauthenticated requests
  - `RedirectToLoginPage`: Redirect to login (most common)
  - `AllowAnonymous`: Let requests through (authentication optional)
  - `Return401`: Return 401 for unauthenticated (APIs)
- `--redirect-provider`: Which provider to use for redirects
- `--aad-client-id`: Your app registration's client ID
- `--aad-tenant-id`: Your Azure AD tenant ID

### Step 5: Configure Excluded Paths (Optional)

Some paths should be accessible without authentication:

```bash
az containerapp auth update \
  --name my-app-web \
  --resource-group my-app-rg \
  --excluded-paths "/.well-known" "/.well-known/" "/.well-known/microsoft-identity-association.json"
```

**Common excluded paths:**
- `/.well-known/*`: For domain verification and OpenID discovery
- `/health`: For health checks and monitoring
- `/api/public/*`: For public API endpoints

---

## Production Configuration Examples

### Example 1: Nexus (Web Application with Custom Domain)

**Container App Easy Auth:**
```json
{
  "platform": {
    "enabled": true
  },
  "globalValidation": {
    "unauthenticatedClientAction": "RedirectToLoginPage",
    "redirectToProvider": "azureactivedirectory",
    "excludedPaths": [
      "/.well-known",
      "/.well-known/",
      "/.well-known/microsoft-identity-association.json"
    ]
  },
  "identityProviders": {
    "azureActiveDirectory": {
      "registration": {
        "clientId": "f87f1f37-53a2-4cde-9e82-d2450a3f04c8",
        "openIdIssuer": "https://login.microsoftonline.com/72f988bf-86f1-41af-91ab-2d7cd011db47/v2.0"
      }
    }
  }
}
```

**Entra ID App Registration:**
```json
{
  "displayName": "Nexus AI - Amplifier",
  "signInAudience": "AzureADMyOrg",
  "web": {
    "redirectUris": [
      "https://nexus.ai.microsoft.com/.auth/login/aad/callback",
      "https://nexus-web.calmground-554d7ad7.eastus.azurecontainerapps.io/.auth/login/aad/callback"
    ],
    "implicitGrantSettings": {
      "enableAccessTokenIssuance": true,
      "enableIdTokenIssuance": true
    }
  },
  "spa": {
    "redirectUris": [
      "http://localhost:4173/auth/redirect",
      "http://localhost:4173"
    ]
  },
  "api": {
    "oauth2PermissionScopes": [
      {
        "value": "access_as_user",
        "adminConsentDisplayName": "Access MS tenant data",
        "adminConsentDescription": "Allow the application to access MS tenant data.",
        "type": "User"
      }
    ]
  },
  "identifierUris": [
    "api://f87f1f37-53a2-4cde-9e82-d2450a3f04c8"
  ]
}
```

**Key Features:**
- ✅ Custom domain with redirect URI
- ✅ Default azurecontainerapps.io domain also configured
- ✅ Localhost redirect URIs for local development
- ✅ Custom OAuth scope for API access
- ✅ Excluded paths for domain verification
- ✅ Implicit grant enabled

### Example 2: Amplifier-Onboarding (SPA-First Configuration)

**Container App Easy Auth:**
```json
{
  "platform": {
    "enabled": true
  },
  "globalValidation": {
    "unauthenticatedClientAction": "RedirectToLoginPage",
    "redirectToProvider": "azureactivedirectory",
    "excludedPaths": []
  },
  "identityProviders": {
    "azureActiveDirectory": {
      "registration": {
        "clientId": "3ac75ca6-8a11-4ea5-b711-556ad1433bd6",
        "openIdIssuer": "https://login.microsoftonline.com/72f988bf-86f1-41af-91ab-2d7cd011db47/v2.0"
      }
    }
  }
}
```

**Entra ID App Registration:**
```json
{
  "displayName": "github-deployer-amplifier",
  "signInAudience": "AzureADMyOrg",
  "spa": {
    "redirectUris": [
      "https://amp-onboarding-web.thankfulplant-e16c9b72.eastus2.azurecontainerapps.io/.auth/login/aad/callback"
    ]
  },
  "web": {
    "redirectUris": [],
    "implicitGrantSettings": {
      "enableAccessTokenIssuance": true,
      "enableIdTokenIssuance": true
    }
  }
}
```

**Key Features:**
- ✅ SPA platform redirect URI (PKCE flow)
- ✅ Simpler configuration (no custom scopes)
- ✅ No excluded paths
- ✅ Single redirect URI (production only)
- ⚠️ No local development redirect URIs

---

## Common Patterns & Best Practices

### Pattern 1: Multiple Redirect URIs

Always configure redirect URIs for all environments:

```bash
# Production custom domain
https://my-app.com/.auth/login/aad/callback

# Production default domain (backup)
https://my-app-web.azurecontainerapps.io/.auth/login/aad/callback

# Local development
http://localhost:4173/auth/redirect
http://localhost:4173
```

**Why multiple URIs?**
- Custom domain might fail, default domain provides fallback
- Local development URIs enable testing without deploying
- Staging/testing environments need their own URIs

### Pattern 2: Excluded Paths for Domain Verification

When using custom domains, exclude `.well-known` paths:

```bash
az containerapp auth update \
  --name my-app-web \
  --resource-group my-app-rg \
  --excluded-paths \
    "/.well-known" \
    "/.well-known/" \
    "/.well-known/microsoft-identity-association.json"
```

**Why?**
- Domain verification requires unauthenticated access to these paths
- Microsoft identity requires accessing `microsoft-identity-association.json`
- Without exclusion, domain verification fails

### Pattern 3: Custom OAuth Scopes for API Access

If your app exposes an API, define custom scopes:

```bash
# This is typically done in the Azure Portal or via Microsoft Graph API
# The scope appears as: api://<client-id>/access_as_user

# Applications requesting access then include:
# scope=api://<client-id>/access_as_user
```

**When to use custom scopes:**
- Your app exposes an API consumed by other apps
- You need granular permission control
- Different users should have different access levels

### Pattern 4: Web vs SPA Platform Selection

**Use Web platform when:**
- You have a server-side application
- Backend handles token exchange
- You want traditional authorization code flow

**Use SPA platform when:**
- You have a single-page application (React, Vue, Angular)
- Frontend handles authentication directly
- You want PKCE-enhanced security

**You can use both:**
- Configure both platforms with appropriate redirect URIs
- Allows flexibility for different client types

---

## Accessing User Information in Your App

Once authenticated, Easy Auth adds headers to every request:

### Available Headers

```
X-MS-CLIENT-PRINCIPAL-ID: <user-object-id>
X-MS-CLIENT-PRINCIPAL-NAME: <user-email>
X-MS-CLIENT-PRINCIPAL: <base64-encoded-json>
X-MS-TOKEN-AAD-ID-TOKEN: <jwt-token>
X-MS-TOKEN-AAD-ACCESS-TOKEN: <jwt-token>
```

### Example: Decoding User Principal

```python
import base64
import json
from fastapi import Header, Request

def get_user_info(x_ms_client_principal: str = Header(None)):
    if not x_ms_client_principal:
        return None
    
    # Decode the base64-encoded JSON
    decoded = base64.b64decode(x_ms_client_principal).decode('utf-8')
    user_info = json.loads(decoded)
    
    return {
        "user_id": user_info.get("userId"),
        "name": user_info.get("userDetails"),
        "claims": user_info.get("claims", [])
    }
```

### Example: Getting Tokens

```python
from fastapi import Header

def get_access_token(x_ms_token_aad_access_token: str = Header(None)):
    # This is already a valid JWT token
    return x_ms_token_aad_access_token
```

---

## Retrieving Current Configuration

### Get Container App Auth Config

```bash
# Get complete auth configuration
az containerapp auth show \
  --name <container-app-name> \
  --resource-group <resource-group> \
  --output json

# Pretty print with jq
az containerapp auth show \
  --name <container-app-name> \
  --resource-group <resource-group> \
  --output json | jq .
```

### Get App Registration Details

```bash
# Get app registration by client ID
az ad app show --id <client-id> --output json

# Get redirect URIs specifically
az ad app show --id <client-id> --query "web.redirectUris" -o table
az ad app show --id <client-id> --query "spa.redirectUris" -o table
```

---

## Layered Auth: Easy Auth + Client-Side MSAL.js

Easy Auth and client-side MSAL.js can coexist on the same Container App, and
in practice that's what apps like Nexus actually do. The two layers serve
different purposes and need **different redirect URIs** registered on the
**same app registration**:

| Layer | What it does | Redirect URI it uses |
|-------|--------------|----------------------|
| Easy Auth | Platform-level interception; sets `X-MS-CLIENT-PRINCIPAL-*` headers; gates the whole app behind sign-in | `https://<fqdn>/.auth/login/aad/callback` — registered under **Web** platform |
| Client-side MSAL.js | Acquires fresh tokens for specific Graph / API scopes the SPA needs to call directly | Whatever the SPA passes as `redirectUri` (commonly `window.location.origin` → `https://<fqdn>`) — registered under **SPA** platform (PKCE) |

If only the Easy Auth callback is registered, users sign in fine through Easy
Auth but then hit AADSTS50011 the moment MSAL.js tries to acquire its own
token — because the bare origin URL isn't in the SPA redirect list.

### What to register when you layer them

```bash
APP_ID="<client-id>"
OBJECT_ID=$(az ad app show --id "$APP_ID" --query id -o tsv)

# Web platform — for Easy Auth callbacks
az ad app update --id "$OBJECT_ID" \
  --web-redirect-uris \
    "https://your-app.com/.auth/login/aad/callback" \
    "https://your-app-web.azurecontainerapps.io/.auth/login/aad/callback"

# SPA platform — for MSAL.js loginRedirect / acquireTokenRedirect
az ad app update --id "$OBJECT_ID" \
  --spa-redirect-uris \
    "https://your-app.com" \
    "https://your-app.com/" \
    "http://localhost:4173" \
    "http://localhost:4173/auth/redirect"
```

Register both trailing-slash and no-slash variants of the SPA URI — some
flows include the slash and you'll hit this same error a second time without
it.

**Why SPA platform specifically:** MSAL.js with `loginRedirect` and
`acquireTokenSilent` uses PKCE and the browser's `fetch` to the token
endpoint. Registering the redirect URI under Web instead of SPA yields
AADSTS9002326 ("Cross-origin token redemption is permitted only for the
'Single-Page Application' client-type").

### Which client_id does the SPA actually send?

Whatever was compiled into the bundle. If the framework is Vite, Next, CRA,
or similar, the client ID was inlined at `npm run build` from a
`VITE_*`/`NEXT_PUBLIC_*`/`REACT_APP_*` env var. Changing Container App env
vars at runtime has no effect on the browser. See
`@dev-ops:context/azure-deployment-patterns.md` Pattern 8 for the diagnostic
(exec into the replica, grep the served bundle).

---

## Troubleshooting

### Issue: "Redirect URI mismatch" error (AADSTS50011)

**Symptoms:** Users redirected to Entra ID but get an error after login

**Causes:**
- Redirect URI in Entra ID doesn't match the callback URL
- Wrong format (missing `/.auth/login/aad/callback` for Easy Auth)
- Wrong platform — registered under Web when the flow needs SPA, or vice versa
- Protocol mismatch (http vs https)
- **The browser is sending the wrong `client_id` entirely** — i.e. the app is rejecting `client_id=X`'s sign-in attempt because `X` has no redirect URIs, when your *intended* app is `Y`

**First, identify which case you're in.** Open the failing flow in DevTools →
Network, watch the request to `login.microsoftonline.com/.../oauth2/v2.0/authorize`,
and read the `client_id` query parameter:

- **`client_id` matches your intended app** → it's a redirect URI / platform / format issue. Use the solution below.
- **`client_id` is something else** (e.g. your deployment service principal) → the SPA bundle was built with the wrong secret. The Entra portal can't fix this; you need to rebuild. See the build-time inlining diagnostic in `@dev-ops:context/azure-deployment-patterns.md` Pattern 8.

**Solution (when the `client_id` is correct):**
```bash
# Check current redirect URIs on both platforms
az ad app show --id <client-id> \
  --query "{web:web.redirectUris, spa:spa.redirectUris}"

# Easy Auth callbacks (format is fixed by the platform):
# https://<fqdn>/.auth/login/aad/callback     ← register under Web
az ad app update --id <object-id> \
  --web-redirect-uris "https://correct-url.com/.auth/login/aad/callback"

# MSAL.js redirects (format is whatever your SPA passes):
# typically https://<fqdn> and https://<fqdn>/  ← register under SPA
az ad app update --id <object-id> \
  --spa-redirect-uris "https://correct-url.com" "https://correct-url.com/"
```

**Solution (when the `client_id` is wrong):**
The build-time secret pointed at the wrong identity. Most common cause is
reusing `AZURE_CLIENT_ID` (the deployment service principal) as the
`VITE_*_APP_ID` build arg. Confirm by exec'ing into a running replica:
```bash
# Portal → Container App → Console → /bin/sh
grep -roh '<expected-guid>\|<wrong-guid>' /app | sort -u
```
If the wrong GUID is there, use a separate GitHub secret for the user-facing
app and trigger a new build with a fresh commit. Re-running the workflow on
the same SHA is a no-op — see `@dev-ops:context/azure-deployment-patterns.md`
Patterns 5 and 8.

### Issue: Authentication loops (redirects indefinitely)

**Symptoms:** User gets stuck in redirect loop between app and Entra ID

**Causes:**
- Excluded paths configuration incorrect
- Token validation failing
- Session cookies not being set

**Solution:**
```bash
# Check excluded paths
az containerapp auth show --name <app> --resource-group <rg> \
  --query "globalValidation.excludedPaths"

# Check if cookies are being set (browser dev tools)
# Look for .AspNetCore.* cookies

# Verify tenant ID matches
az containerapp auth show --name <app> --resource-group <rg> \
  --query "identityProviders.azureActiveDirectory.registration.openIdIssuer"
```

### Issue: Local development fails

**Symptoms:** Can't authenticate when running locally

**Causes:**
- No localhost redirect URI configured
- HTTPS required but using HTTP locally
- CORS issues

**Solution:**
```bash
# Add localhost redirect URIs
az ad app update --id <object-id> \
  --spa-redirect-uris \
    "http://localhost:4173" \
    "http://localhost:4173/auth/redirect" \
    "http://localhost:3000" \
    "http://localhost:5173"

# Note: localhost can use HTTP, production must use HTTPS
```

### Issue: "unauthorized_client" error

**Symptoms:** Error message saying client is not authorized

**Causes:**
- App registration not found
- Client ID mismatch
- App not granted necessary permissions

**Solution:**
```bash
# Verify client ID matches
az containerapp auth show --name <app> --resource-group <rg> \
  --query "identityProviders.azureActiveDirectory.registration.clientId"

# Verify app registration exists
az ad app show --id <client-id>

# Check tenant ID is correct
az account show --query tenantId -o tsv
```

### Issue: Custom domain not working

**Symptoms:** Authentication works with default domain but not custom domain

**Causes:**
- Custom domain redirect URI not configured
- `.well-known` paths not excluded
- DNS not properly configured

**Solution:**
```bash
# Add custom domain redirect URI
az ad app update --id <object-id> \
  --web-redirect-uris \
    "https://custom-domain.com/.auth/login/aad/callback" \
    "https://default-domain.azurecontainerapps.io/.auth/login/aad/callback"

# Exclude .well-known paths
az containerapp auth update \
  --name <app> \
  --resource-group <rg> \
  --excluded-paths "/.well-known" "/.well-known/" "/.well-known/microsoft-identity-association.json"

# Verify DNS points to container app
nslookup custom-domain.com
```

---

## Quick Reference: Azure CLI Commands

### Authentication Configuration

```bash
# View current auth config
az containerapp auth show --name <app> --resource-group <rg>

# Enable authentication
az containerapp auth update \
  --name <app> \
  --resource-group <rg> \
  --enabled true \
  --action RedirectToLoginPage \
  --redirect-provider azureactivedirectory \
  --aad-client-id <client-id> \
  --aad-tenant-id <tenant-id>

# Disable authentication
az containerapp auth update \
  --name <app> \
  --resource-group <rg> \
  --enabled false

# Add excluded paths
az containerapp auth update \
  --name <app> \
  --resource-group <rg> \
  --excluded-paths "/.well-known" "/health" "/api/public"
```

### App Registration Management

```bash
# Create app registration
az ad app create \
  --display-name "My App" \
  --sign-in-audience AzureADMyOrg

# Get app details
az ad app show --id <client-id>

# Update redirect URIs
az ad app update --id <object-id> \
  --web-redirect-uris "https://app.com/.auth/login/aad/callback"

# Add SPA redirect URIs
az ad app update --id <object-id> \
  --spa-redirect-uris "http://localhost:3000" "http://localhost:4173"

# Enable implicit grant
az ad app update --id <object-id> \
  --enable-access-token-issuance true \
  --enable-id-token-issuance true
```

---

## Next Steps

1. **Test authentication flow**: Try accessing your app and verify redirect to Entra ID
2. **Configure application code**: Update your app to read user info from Easy Auth headers
3. **Set up additional environments**: Create app registrations for dev/staging/prod
4. **Configure authorization**: Set up role-based access control if needed
5. **Monitor authentication**: Set up logging and monitoring for auth failures

## Additional Resources

- [Azure Container Apps Authentication Documentation](https://learn.microsoft.com/en-us/azure/container-apps/authentication)
- [Azure AD App Registration Documentation](https://learn.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)
- [OAuth 2.0 and OpenID Connect Protocols](https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-protocols)
- [Easy Auth Configuration Reference](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad)

---

**Document Version:** 1.0  
**Last Updated:** 2026-05-06  
**Based on configurations from:** Nexus (nexus-web) and Amplifier-Onboarding (amp-onboarding-web)

---
skill: authenticated-tool-patterns
version: 1.0.0
context: current
description: |
  How Amplifier modules (like tool-msgraph) leverage exchanged user tokens for authenticated
  API flows. Covers the UserTokenStore capability, SSE-driven lazy token acquisition from
  the tool's perspective, Graph scope management, the three-way match rule, graceful
  degradation, session lifecycle with tokens, and prompt discipline for authenticated tools.
  Use when building or debugging any Amplifier tool that calls authenticated external APIs.
  Extracted from 353+ production sessions across Amplifier, Copilot, and Claude (Jan-May 2026).
tags: [amplifier, tools, modules, graph-api, tokens, obo, msgraph, authenticated-apis]
---

# Authenticated Tool Patterns

How Amplifier modules consume exchanged user tokens for authenticated external API calls. This covers the module developer's perspective -- how your tool requests, receives, and uses scoped tokens within the Amplifier session lifecycle.

For how the Azure platform provides and manages these tokens, see `azure-auth-tokens`.

---

## 1. The Token Capability Model

Tokens are passed to tools through the coordinator capability system. A tool never manages its own authentication -- it receives a token from the session infrastructure and uses it.

### UserTokenStore

A per-session, async-safe token container registered as a coordinator capability:

```python
class UserTokenStore:
    """Per-session token storage. Async lock for thread safety."""

    def __init__(self, bearer_token: Optional[str] = None):
        self._token = bearer_token
        self._lock = asyncio.Lock()

    async def get(self) -> Optional[str]:
        async with self._lock:
            return self._token

    async def set(self, new_token: str) -> None:
        async with self._lock:
            self._token = new_token
```

### Registration

The session manager registers the token store as a capability when a session is created with a token:

```python
async def create_session(self, config_id, bearer_token=None):
    # ... session creation ...
    if bearer_token:
        token_store = UserTokenStore(bearer_token)
        amplifier_session.coordinator.register_capability(
            "user.token_store", token_store
        )
```

### Tool Access Pattern

Every authenticated tool follows this pattern:

```python
async def execute(self, input: dict, coordinator) -> dict:
    # 1. Look up the token capability
    token_store = coordinator.get_capability("user.token_store")

    if not token_store:
        # No token available -- degrade gracefully
        return {"status": "skipped", "reason": "no_token"}

    token = await token_store.get()
    if not token:
        return {"status": "error", "reason": "token_expired"}

    # 2. Use the token for the API call
    headers = {"Authorization": f"Bearer {token}"}
    # ... make authenticated API call ...
```

---

## 2. SSE-Driven Lazy Token Acquisition (Tool's Perspective)

When a tool needs a token for a specific Graph scope, it doesn't acquire the token itself. It signals the frontend via SSE, and the frontend handles MSAL acquisition.

### Flow from the Tool's Point of View

```
Tool.execute() called
  |
  +--> Check coordinator for existing token with required scope
  |
  +--> If missing/expired: emit `token_needed` SSE event
  |    (includes required scope, e.g., "Mail.Read")
  |
  +--> await asyncio.Future (with timeout)
  |    ... frontend acquires via MSAL ...
  |    ... frontend POSTs to /sessions/{id}/provide-token ...
  |    ... future resolves with scoped token ...
  |
  +--> Use token for Graph API call
  |
  +--> Return result
```

### Implementation Rules for Tools

| Rule | Detail |
|------|--------|
| Never acquire tokens directly | Tools request via SSE; the frontend handles MSAL |
| Use per-scope async locks | Prevents multiple concurrent requests for the same scope |
| Respect the 60s timeout | If the frontend can't provide a token (popup blocked), fail gracefully |
| Never use ID tokens | OBO flow needs an access token. ID tokens fail silently. |
| Never cache without expiry | Check the `exp` claim. Stale tokens cause silent 401s. |
| Thread `bearer_token` on resume | Auto-resumed sessions must re-attach the `UserTokenStore` |

### Graceful Degradation

Tools MUST work when no token is available. The `bearer_token` field on session create is optional -- not all sessions need Graph access.

```python
# Good: graceful degradation
if not token_store:
    return {
        "status": "skipped",
        "reason": "no_token",
        "message": "Graph API not available. Sign in to enable."
    }

# Bad: hard failure
if not token_store:
    raise RuntimeError("Token required!")  # Breaks non-Graph workflows
```

---

## 3. Graph Scopes & the Three-Way Match Rule

### Currently Consented Scopes (Delegated, READ-Only)

```
User.Read           # Basic profile
Mail.Read           # Read emails
Calendars.Read      # Read calendar
People.Read         # People suggestions
Presence.Read       # Online status
Sites.Read.All      # SharePoint sites
Files.ReadWrite     # OneDrive (the lone partial-write)
```

Each maps 1:1 to a mounted tool in `tool-msgraph`.

### Coded But NOT Consented (Awaiting Admin)

```
Contacts.Read       # Tool code exists, admin consent pending
Tasks.Read          # Tool code exists, admin consent pending
Notes.Read.All      # Tool code exists, admin consent pending
```

### The Three-Way Match Rule

These three MUST all agree. If any one is out of sync, something breaks:

```
    Consented Scopes          Mounted Tools           Prompt Documentation
    (Entra app reg)      (tool-msgraph/__init__.py)   (MSGRAPH_USAGE_GUIDE.md)
         |                        |                          |
         +--------  MUST MATCH  --+--------  MUST MATCH  ----+
```

| Mismatch | Result |
|----------|--------|
| Coded but not consented | `403 Forbidden` at runtime |
| Consented but not mounted | Wasted permission, user can't access |
| Documented but not mounted | Model apologizes ("I don't have that capability") instead of calling the tool |
| Mounted but not documented | Model doesn't know the tool exists, never calls it |

### Add-a-Tool Sequence (3 Steps, In Order)

1. **Admin consent in Entra** -- only app-reg owners can do this, not the deploy service principal
2. **Re-enable matching import** in `tool-msgraph/__init__.py`
3. **Deploy**

Skipping step 1 means step 2 gives you a tool that always returns 403.

### Strategy: READ-First, Max Out READ Before WRITE

```
Phase 1: User.Read only (login works)
Phase 2: Add all READ scopes (Mail.Read, Calendars.Read, etc.)
Phase 3: Only after READ works end-to-end, add WRITE scopes
```

Why:
- READ permissions are lower risk, easier to get approved
- Validates the entire token flow before mutations are possible
- Iterative debugging is simpler with read-only operations
- Security review is simpler

---

## 4. Making Graph API Calls from Tools

### Standard Pattern

```python
import httpx

GRAPH_BASE = "https://graph.microsoft.com/v1.0"

async def call_graph(
    endpoint: str,
    token: str,
    method: str = "GET",
    data: dict | None = None,
) -> dict:
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    }
    async with httpx.AsyncClient() as client:
        resp = await client.request(
            method=method,
            url=f"{GRAPH_BASE}{endpoint}",
            headers=headers,
            json=data,
        )

        if resp.status_code == 401:
            raise TokenExpiredError("Token expired or invalid")
        if resp.status_code == 403:
            raise InsufficientPermissionsError(
                f"Missing scope for {endpoint}. "
                "Check admin consent in Entra."
            )
        resp.raise_for_status()
        return resp.json()
```

### Error Handling Requirements

Every tool that calls Graph MUST handle these specifically:

| Status | Meaning | Tool Response |
|--------|---------|---------------|
| `401` | Token expired/invalid | Signal token refresh via SSE, retry once |
| `403` | Scope not consented | Return clear message: "Admin consent needed for X scope" |
| `404` on `Files.ReadWrite.All` | User has no OneDrive provisioned | Surface specifically: "OneDrive not provisioned for this user" |
| `429` | Throttled | Respect `Retry-After` header, back off |

**Do not** return generic "Graph API error" messages. The user needs to know exactly what failed and what to do about it.

### JWT Verification in Tools

When debugging, decode the token and verify:

| Claim | Check |
|-------|-------|
| `aud` | Must be your app's client ID (accepts both bare GUID and `api://<guid>`) |
| `scp` | Must contain the scope your tool needs |
| `exp` | Must be in the future |
| `upn` | Confirms the user identity |

**"Configured != granted":** A scope listed in the app registration is not the same as a scope present in the token. Always check the actual `scp` claim.

---

## 5. Session Lifecycle with Tokens

### Session Creation

```
Frontend: User signs in via MSAL
  |
  +--> POST /api/sessions/create
  |    { config_id: "default", bearer_token: accessToken }
  |
Backend: SessionManager.create_session()
  |
  +--> UserTokenStore(bearer_token)
  +--> coordinator.register_capability("user.token_store", store)
  +--> Return session ID
  |
Tools: Can now call coordinator.get_capability("user.token_store")
```

### Session Resume

```
Frontend: User returns (has session ID)
  |
  +--> acquireTokenSilent() to refresh if expired
  |
  +--> POST /api/sessions/{id}/resume
  |    { bearer_token: freshAccessToken }
  |
Backend: Re-register token store with fresh token
  |
  +--> Session ready with updated token
```

**Critical:** `bearer_token` must be threaded through `send_message`/`stream_message` so auto-resumed sessions re-attach the `UserTokenStore`. If this threading is missing, resumed sessions silently lose Graph access.

### SessionManager Singleton Requirement

`SessionManager` MUST be a process-wide singleton. The `/stream` endpoint and `/provide-token` endpoint must share the same in-memory `_sessions` and futures dict. Per-request `Depends` instantiation broke OBO because `/provide-token` couldn't find the future that `/stream` created.

---

## 6. Prompt Discipline for Authenticated Tools

### What to Tell the Model

```
Call the tool first. The UI handles MSAL authentication automatically.
Do NOT ask the user to authenticate before calling a tool.
```

The model should attempt the tool call. If the token is missing, the tool returns a graceful degradation response. If the token is available, it works. The model should never gate a tool call behind "are you logged in?"

### Tool Description Patterns

Tool descriptions visible to the model must match reality:

```python
# Good: matches what's actually mounted and consented
"Read the user's recent emails (requires Mail.Read scope)"

# Bad: describes capability that isn't consented
"Manage the user's calendar events"  # if only Calendars.Read is consented
```

### Scope Documentation in Prompt

The prompt's tool guidance (`base-bundle.md`, `MSGRAPH_USAGE_GUIDE.md`) must list:
- Which tools are available
- What each tool does
- What scope each requires
- Whether the scope is READ or WRITE

This documentation is the model's only way to know what capabilities exist. If a tool is mounted but not documented, the model won't call it.

---

## 7. Building a New Authenticated Tool

Step-by-step for adding a new Graph-backed tool to `tool-msgraph`:

### Step 1: Verify the Scope

```bash
# Check what's actually consented (decode a real token)
# Look at the 'scp' claim, not the app registration
```

If the scope isn't consented, stop here. File the admin consent request first.

### Step 2: Implement the Tool

```python
# tool-msgraph/tools/read_contacts.py

class ReadContactsTool:
    """Read user's contacts. Requires Contacts.Read scope."""

    name = "read_contacts"
    description = "Read the signed-in user's contacts"

    async def execute(self, input: dict, coordinator) -> dict:
        token_store = coordinator.get_capability("user.token_store")

        if not token_store:
            return {
                "status": "skipped",
                "reason": "no_token",
                "message": "Sign in to access contacts."
            }

        token = await token_store.get()
        if not token:
            return {"status": "error", "reason": "token_expired"}

        try:
            result = await call_graph("/me/contacts", token)
            return {"contacts": result.get("value", [])}
        except InsufficientPermissionsError:
            return {
                "status": "error",
                "reason": "scope_not_consented",
                "message": "Contacts.Read scope needs admin consent."
            }
```

### Step 3: Register in `__init__.py`

```python
# tool-msgraph/__init__.py
from .tools.read_contacts import ReadContactsTool  # Enable when consented
```

### Step 4: Update Prompt Documentation

Add to `MSGRAPH_USAGE_GUIDE.md`:
```
- **read_contacts** -- Read the user's contacts (Contacts.Read)
```

### Step 5: Deploy

Follow the three-step sequence: consent -> code -> deploy.

---

## 8. Anti-Patterns

| Anti-Pattern | What Happens | Fix |
|--------------|-------------|-----|
| Tool acquires tokens directly | Bypasses SSE flow, breaks frontend/backend separation | Use `coordinator.get_capability("user.token_store")` |
| Hard failure when no token | Breaks all non-Graph sessions | Return `{"status": "skipped", "reason": "no_token"}` |
| Making `bearer_token` required in API | Forces dummy tokens for non-Graph use cases | Keep it `Optional` |
| Storing tokens in tool state | Security risk, stale across requests | Always read from `UserTokenStore` |
| Frontend deciding scopes | Scope changes require frontend updates | Backend tools declare scope needs; frontend only handles MSAL |
| Requesting all scopes upfront | Over-permissioned, approval delays | Iterative READ-first expansion |
| Generic "Graph error" messages | User can't diagnose the problem | Return specific status + scope info |
| Mounting a tool before consent | Tool always returns 403 | Three-step sequence: consent -> code -> deploy |
| Documenting unmounted tools | Model apologizes instead of saying "not available" | Prompt docs must match mounted tools exactly |
| Tool asks model to check auth | Wastes a turn, confusing UX | Tool calls Graph, handles auth internally |
| Not threading `bearer_token` on resume | Resumed sessions silently lose Graph access | Thread through `send_message`/`stream_message` |
| Per-request `Depends` for SessionManager | `/provide-token` can't find `/stream`'s future | Singleton SessionManager |

---

## 9. Troubleshooting Tools

| Symptom | Cause | Fix |
|---------|-------|-----|
| Tool returns `{"status": "skipped"}` | No `user.token_store` capability registered | Verify `bearer_token` was sent on session create |
| `403` from Graph | Scope not admin-consented | Check `scp` claim in JWT. Request consent from app-reg owner. |
| `404` on Files endpoint | User has no OneDrive | Surface this to user specifically |
| `401` from Graph | Token expired | Trigger SSE re-acquisition, retry once |
| Tool works on first call, fails on resume | `bearer_token` not threaded through resume path | Fix `send_message`/`stream_message` to pass token |
| Model never calls the tool | Tool not documented in prompt | Add to `MSGRAPH_USAGE_GUIDE.md` |
| Model asks user to log in first | Prompt tells model to check auth | Fix prompt: "call the tool first, UI handles auth" |
| `token_needed` SSE never resolves | Popup blocked in async context | Pre-fetch at sign-in click |
| Two tools request same scope simultaneously | No per-scope lock, duplicate popups | Add `asyncio.Lock` per scope |

---

## 10. Checklist: Adding an Authenticated Tool

### Before Writing Code
- [ ] Scope identified (e.g., `Contacts.Read`)
- [ ] Admin consent requested from app-reg owner
- [ ] Consent confirmed (decode a real JWT, check `scp`)

### Implementation
- [ ] Tool uses `coordinator.get_capability("user.token_store")`
- [ ] Graceful degradation when no token (returns `skipped`, not exception)
- [ ] Specific error messages for 401, 403, 404
- [ ] No direct token acquisition (uses SSE flow)
- [ ] No token stored in tool instance state

### Integration
- [ ] Import enabled in `tool-msgraph/__init__.py`
- [ ] Tool description in `MSGRAPH_USAGE_GUIDE.md` matches reality
- [ ] Prompt lists the tool with its scope requirement
- [ ] Three-way match verified: consent = mounted = documented

### Testing
- [ ] Works with token (happy path)
- [ ] Works without token (graceful skip)
- [ ] Works on session resume (token re-attached)
- [ ] 403 produces actionable error message
- [ ] Model calls the tool without prompting for auth

---

**Version:** 1.0.0
**Last Updated:** May 8, 2026
**Based on:** 353+ sessions across Amplifier (278), Copilot (33), Claude (42)

# Azure Authentication Fundamentals

A no-bullshit reference for how auth works in Azure, from the token itself up through the credential types you'll actually use.

---

## JWT (JSON Web Token)

A JWT is three base64-encoded strings separated by dots:

```
<header>.<payload>.<signature>
```

### Header

Metadata about the token — what algorithm signed it and the token type.

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

### Payload

The claims — who the token belongs to, what it can access, and when it expires.

```json
{
  "sub": "javier@covest.com",
  "oid": "a]b1c2d3-...",
  "aud": "https://analysis.windows.net/powerbi/api",
  "iss": "https://login.microsoftonline.com/<tenant-id>/v2.0",
  "exp": 1714500000,
  "scp": "Dataset.ReadWrite.All"
}
```

Key claims to know:

| Claim | What it means |
|-------|---------------|
| `sub` | Subject — who authenticated (user principal or app ID) |
| `oid` | Object ID — the unique ID for the user/app in Entra ID |
| `aud` | Audience — which API this token is valid for |
| `iss` | Issuer — who minted the token (always Microsoft's login endpoint + your tenant) |
| `exp` | Expiration — Unix timestamp, typically ~1 hour from issue |
| `scp` | Scopes — what permissions the token carries (delegated/user context) |
| `roles` | Roles — app-level permissions (service principal context) |
| `tid` | Tenant ID — which Azure AD tenant issued the token |

`scp` shows up for delegated (user) flows. `roles` shows up for application (service principal) flows. They don't appear together.

### Signature

Microsoft signs the header + payload with their private key. The receiving API (Power BI, Graph, etc.) verifies the signature using Microsoft's public key. This proves the token wasn't forged or tampered with.

You can decode any JWT at [jwt.ms](https://jwt.ms) to inspect claims and debug permission issues.

---

## OAuth2 — The Flow Behind the Token

OAuth2 is the protocol that governs *how* you get the token. Azure AD (Entra ID) implements several OAuth2 flows, but you'll realistically encounter three.

### Authorization Code Flow (interactive / user context)

This is what happens when `InteractiveBrowserCredential` fires.

1. SDK spins up a temporary HTTP server on `localhost`.
2. Opens your browser to `https://login.microsoftonline.com/.../authorize` with a `redirect_uri` pointing back to localhost.
3. You log in (or your existing session is reused).
4. Microsoft redirects back to localhost with an **authorization code**.
5. SDK exchanges that code for an **access token** (and a refresh token) via a POST to the `/token` endpoint.
6. `.get_token()` returns the access token string.

The authorization code itself is short-lived and single-use. It's never sent to the resource API — only the resulting access token is.

### Client Credentials Flow (service principal / no user)

This is what `ClientSecretCredential` uses. No browser, no user interaction.

1. SDK sends a POST to the `/token` endpoint with `client_id`, `client_secret`, and `scope`.
2. Microsoft validates the credentials and returns an access token.
3. Done.

The token has `roles` instead of `scp` because there's no user — permissions are assigned directly to the app registration.

### Managed Identity (Azure-hosted, no secrets)

This is what happens in Azure Functions, App Service, VMs, etc.

1. Your code calls `ManagedIdentityCredential().get_token(scope)`.
2. SDK hits Azure's internal metadata endpoint (`169.254.169.254`) — no secrets leave the host.
3. Azure mints a token scoped to the identity assigned to your resource.
4. Done.

No secrets to rotate. No `.env` files. This is the target state for anything running in Azure.

---

## Scopes

A scope tells Azure AD *which API* the token is for and *what permissions* to include.

```python
# Power BI
"https://analysis.windows.net/powerbi/api/.default"

# Microsoft Graph (email, calendar, files, etc.)
"https://graph.microsoft.com/.default"

# Azure SQL
"https://database.windows.net/.default"

# Azure Key Vault
"https://vault.azure.net/.default"

# Azure Storage (Blob, Queue, Table)
"https://storage.azure.com/.default"
```

The `/.default` suffix means "give me all permissions that have been configured for this app." You can also request granular scopes like `User.Read` or `Dataset.ReadWrite.All`, but `/.default` is the standard approach for service-to-service calls.

A token minted for Power BI **will not work** against the Graph API. One token, one audience. If your script calls both Power BI and Graph, you need two tokens.

---

## Azure Identity Credential Types

All credential classes in the `azure-identity` SDK implement the same interface: `.get_token(scope) -> AccessToken`. The difference is *how* they authenticate.

### `InteractiveBrowserCredential`

- Opens a browser for user login.
- Use case: local development, scripts you run at your desk.
- Requires a human present.

### `ClientSecretCredential`

- Authenticates with `tenant_id`, `client_id`, and `client_secret`.
- Use case: automated scripts, CI/CD pipelines, anything headless outside Azure.
- Requires an app registration in Entra ID with a secret.
- Secrets expire and need rotation.

### `ManagedIdentityCredential`

- No secrets at all. Azure handles it internally via the instance metadata service.
- Use case: Azure Functions, App Service, VMs, Container Apps — anything hosted in Azure.
- System-assigned (tied to the resource lifecycle) or user-assigned (shared across resources).
- This is the correct choice for production Azure workloads.

### `DefaultAzureCredential`

- Tries ~8 credential types in sequence: environment variables → managed identity → Azure CLI → Visual Studio → interactive browser → etc.
- Use case: libraries or apps that need to work across environments without code changes.
- Downside: when it fails, you get a wall of errors from every credential type it tried.

### `ChainedTokenCredential`

- Same idea as `DefaultAzureCredential` but you pick exactly which types and in what order.
- Use case: when you want the fallback behavior but need a tighter error surface.

```python
from azure.identity import (
    ChainedTokenCredential,
    ManagedIdentityCredential,
    InteractiveBrowserCredential,
)

cred = ChainedTokenCredential(
    ManagedIdentityCredential(),       # try this first (Azure-hosted)
    InteractiveBrowserCredential(),    # fall back to browser (local dev)
)
token = cred.get_token("https://analysis.windows.net/powerbi/api/.default").token
```

---

## Putting It Together

The entire auth flow in a Python script:

```python
from azure.identity import InteractiveBrowserCredential

# 1. Pick a credential type
cred = InteractiveBrowserCredential()

# 2. Request a token for a specific API
token = cred.get_token("https://analysis.windows.net/powerbi/api/.default").token

# 3. Attach it to your HTTP request
headers = {"Authorization": f"Bearer {token}"}
```

That `Bearer` prefix is part of the OAuth2 spec — it tells the API "this is a bearer token, whoever holds it gets access." Which is why you never log tokens, commit them, or send them over unencrypted connections.

---

## Common Gotchas

**Token is valid but the API returns 403.** The token was minted successfully but doesn't have the right permissions. Decode it at jwt.ms and check `scp` or `roles`. Usually means the app registration is missing an API permission, or admin consent hasn't been granted.

**Token works for Graph but not Power BI (or vice versa).** Tokens are audience-locked. A token with `aud: https://graph.microsoft.com` will be rejected by the Power BI API. You need a separate `.get_token()` call with the correct scope.

**`DefaultAzureCredential` works locally but fails in Azure.** Usually means managed identity isn't enabled on the Azure resource, or the identity doesn't have the required role assignment on the target resource.

**Token expired mid-script.** Access tokens live ~60-75 minutes. For long-running processes, call `.get_token()` again — the SDK handles refresh tokens and caching internally.

**`InteractiveBrowserCredential` hangs in a container or SSH session.** There's no browser to open. Use `ClientSecretCredential` or `ManagedIdentityCredential` for headless environments.

# Azure Blob Storage

---

## What Is a Storage Account?

A top-level Azure resource that holds all your storage services. Think of it as a container for containers. One storage account can hold:

- **Blob Storage** — unstructured data like files, PDFs, images
- **Queue Storage** — message queues for async communication
- **Table Storage** — NoSQL key-value storage (Durable Functions uses this for replay state)
- **File Storage** — managed file shares (SMB protocol)

Durable Functions uses table and queue storage behind the scenes for orchestration state. You don't configure that part — the framework manages it automatically.

---

## Blob Storage Structure

Three levels of hierarchy:

```
Storage Account (mystorageaccount)
  └── Container (e.g., "reports")
       └── Blob (e.g., "Acme Corp – Staples – March 2026.pdf")
```

A **container** is like a folder at the top level. You can't nest containers. A **blob** is any file — PDF, CSV, image, whatever. The blob name can include `/` to simulate folder structure (`2026/03/report.pdf`) but those aren't real folders, just naming conventions.

---

## Authentication: Two Approaches

### Storage Account Keys (avoid when possible)

Two static keys (key1, key2) generated when the account is created. Full read/write/delete access to everything. Never expire.

Where they live: Azure Portal → Storage Account → Access Keys.

Problems:
- If leaked, an attacker owns your entire storage account
- No audit trail of who used them
- Must be manually rotated
- Stored as secrets in config, which means they can end up in code, .env files, or deployment logs

### Managed Identity (recommended)

The Function App has a system-assigned managed identity. Azure AD handles authentication automatically. No secrets stored anywhere.

```python
# DefaultAzureCredential picks up managed identity in Azure
# and your dev credentials locally
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
blob_service = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=credential,
)
```

Benefits:
- No secrets to rotate or leak
- Audit logs in Azure AD
- Revoke access by disabling the identity
- Works locally with your developer credentials via `DefaultAzureCredential`

---

## Uploading Blobs

```python
blob_client = blob_service.get_blob_client(
    container=container_name,
    blob=blob_name,
)
blob_client.upload_blob(data, overwrite=True)
```

`overwrite=True` is important. Without it, uploading to an existing blob name throws an error. With it, the old blob is silently replaced. For idempotent pipelines where reruns are expected, this prevents failures on duplicate uploads.

---

## SAS Tokens: Temporary Access Links

A **Shared Access Signature (SAS) token** is a URL query string that grants time-limited access to a specific blob without requiring authentication. The member clicks a link, downloads the PDF, no Azure AD login needed.

A SAS URL looks like:

```
https://mystorageaccount.blob.core.windows.net/reports/report.pdf?sv=2022-11-02&se=2026-04-28&sr=b&sp=r&sig=abc123...
```

Key parameters in the token:
- `se` — expiry datetime (when the link dies)
- `sr` — resource type (`b` = blob)
- `sp` — permissions (`r` = read only)
- `sig` — cryptographic signature proving the token is legitimate

The signature is what makes the token trustworthy. Azure Storage verifies the signature before granting access. A tampered token fails signature verification and gets rejected.

---

## Signing SAS Tokens: Two Options

### Option 1: Storage Account Key (not recommended)

```python
sas_token = generate_blob_sas(
    account_name=account_name,
    container_name=container_name,
    blob_name=blob_name,
    account_key=account_key,        # static secret
    permission=BlobSasPermissions(read=True),
    expiry=now + timedelta(days=14),
)
```

The token is signed with the storage account key. If the key leaks, anyone can forge SAS tokens for any blob with any permissions and any expiry. No time limit on the damage.

### Option 2: User Delegation Key (recommended)

```python
# Step 1: Get a temporary signing key from Azure AD
delegation_key = blob_service.get_user_delegation_key(
    key_start_time=now,
    key_expiry_time=now + timedelta(days=7),  # 7 days max, Azure enforced
)

# Step 2: Sign the SAS token with the delegation key
sas_token = generate_blob_sas(
    account_name=account_name,
    container_name=container_name,
    blob_name=blob_name,
    user_delegation_key=delegation_key,   # temporary key
    permission=BlobSasPermissions(read=True),
    expiry=now + timedelta(days=7),       # cannot exceed delegation key expiry
)
```

The delegation key is a temporary credential generated on demand through Azure AD. Tied to the managed identity. Maximum lifespan of 7 days (Azure hard limit — request 8 and the API rejects it).

---

## Delegation Key vs Storage Account Key

Think of the SAS token as a hall pass. The signature is the stamp that makes it valid.

| | Storage Account Key | User Delegation Key |
|---|---|---|
| Lifespan | Permanent until manually rotated | 7 days max |
| Scope | Full access to entire storage account | Tied to a specific identity |
| If leaked | Attacker has unlimited access | Damage is time-limited |
| Audit trail | None | Azure AD logs who requested it |
| Revocation | Rotate the key (breaks all existing SAS tokens) | Disable the identity |
| Secrets to manage | Yes — must store key securely | None — managed identity handles it |

---

## The Expiry Trap

The SAS token has its own expiry. The delegation key that signed it has its own expiry. **The shorter one wins.**

If the delegation key expires in 7 days but the SAS token says 14 days, the link breaks on day 8. Azure checks both. The delegation key is dead, so the signature is no longer valid, and the SAS token returns a 403 Forbidden — even though the token's own expiry hasn't passed.

**Rule: SAS token expiry must be ≤ delegation key expiry.**

Both the SAS token expiry and delegation key expiry should be set to the same value (typically 7 days). Setting the SAS token expiry longer than the delegation key expiry results in download links that silently break — they return 403 Forbidden even though the token's own expiry hasn't passed.

---

## BlobServiceClient: The Entry Point

Everything starts here:

```python
from azure.storage.blob import BlobServiceClient

blob_service = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=credential,
)
```

From the service client, you navigate down:

```python
# Get a reference to a container
container_client = blob_service.get_container_client("reports")

# Get a reference to a specific blob
blob_client = blob_service.get_blob_client(
    container="reports",
    blob="report.pdf",
)

# Upload
blob_client.upload_blob(pdf_bytes, overwrite=True)

# Download
data = blob_client.download_blob().readall()

# Delete
blob_client.delete_blob()

# List blobs in a container
for blob in container_client.list_blobs():
    print(blob.name)
```

---

## Example: Report Delivery Pipeline Flow

A practical example of blob storage in a report delivery pipeline:

1. **Export** — Generate a PDF report from a reporting service, download the bytes.

2. **Upload** — Create a `BlobServiceClient` with managed identity credentials. Upload the PDF with `overwrite=True`.

3. **Delegation key** — Request a user delegation key valid for 7 days.

4. **SAS token** — Generate a read-only SAS token signed by the delegation key, expiring in 7 days (matching the delegation key).

5. **SAS URL** — Combine the blob URL with the SAS token. This becomes a shareable download link.

6. **Return** — `{**job, "sas_url": sas_url}` — new dict with the SAS URL added, original job dict untouched (immutable dict pattern).

---

## Common Operations Reference

| Task | Method |
|---|---|
| Upload a file | `blob_client.upload_blob(data, overwrite=True)` |
| Download a file | `blob_client.download_blob().readall()` |
| Delete a blob | `blob_client.delete_blob()` |
| List blobs | `container_client.list_blobs()` |
| Check if blob exists | `blob_client.exists()` |
| Get blob properties | `blob_client.get_blob_properties()` |
| Create a container | `blob_service.create_container("name")` |
| Get delegation key | `blob_service.get_user_delegation_key(start, expiry)` |
| Generate SAS token | `generate_blob_sas(account, container, blob, ...)` |

---

## Key Imports

```python
from azure.storage.blob import (
    BlobServiceClient,
    BlobSasPermissions,
    generate_blob_sas,
)
from azure.identity import DefaultAzureCredential
from datetime import datetime, timedelta, timezone
```

# Lab 3: Data & Application Security

Check box if done: [ ]

## Overview
This lab replaces the two most common secrets-management mistakes — hardcoded connection strings and storage account keys floating around config files — with the pattern actually used in production: Key Vault with RBAC authorization, a managed identity that authenticates without any stored credential, and customer-managed encryption keys for data at rest.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 (Key Vault and managed identities are free; you pay only for the small storage account)

---

## Scenario
An application needs three things: a database connection string it shouldn't hardcode, an encryption key for its storage account that your org controls (not just Microsoft's default key), and a way to authenticate to Key Vault without a credential sitting in a config file. You're building all three, the way a security engineer would review and approve.

---

## Objectives
- Deploy Key Vault with **RBAC authorization** (not the legacy access-policy model) and understand why that distinction matters
- Store and retrieve secrets, keys, and certificates
- Configure a storage account to encrypt data at rest with a **customer-managed key (CMK)** stored in Key Vault
- Use a **managed identity** so an application authenticates to Key Vault with zero stored credentials
- Prove the negative: an identity without a role assignment is denied

---

## Part 1: Deploy Key Vault with RBAC Authorization

### Step 1: Why RBAC Over Access Policies
Key Vault has two authorization models. The legacy **access policy** model is all-or-nothing per vault — an assigned identity gets broad permissions (all secrets, all keys, all certs) with no scoping to individual objects, and it doesn't integrate with Azure's standard RBAC audit trail. The **RBAC** model uses the same role assignments as everything else in Azure (`Key Vault Secrets User`, `Key Vault Crypto User`, etc.), supports scoping down to a single secret, and shows up in the same access reviews and `Get-AzRoleAssignment` queries as every other resource. RBAC is the current recommended model — always choose it for new vaults.

### Step 2: Create the Vault

```bash
az group create --name az500-lab3-rg --location eastus2

az keyvault create \
  --resource-group az500-lab3-rg \
  --name az500kv$RANDOM \
  --location eastus2 \
  --enable-rbac-authorization true \
  --enable-purge-protection true \
  --retention-days 7
```

> Save the vault name: `az keyvault list -g az500-lab3-rg --query "[].name" -o tsv`

**Why purge protection?** Without it, someone with delete permission can permanently purge a vault (and everything in it — every secret, every key) with no recovery window. Purge protection enforces the retention period even against a malicious or accidental purge attempt.

### Step 3: Grant Yourself Access (You Have None by Default Under RBAC)
This is the first thing that surprises people coming from the access-policy model: creating the vault does **not** grant you access to it under RBAC.

```bash
KV_NAME=$(az keyvault list -g az500-lab3-rg --query "[0].name" -o tsv)
KV_ID=$(az keyvault show -g az500-lab3-rg -n $KV_NAME --query id -o tsv)
MY_ID=$(az ad signed-in-user show --query id -o tsv)

az role assignment create \
  --role "Key Vault Administrator" \
  --assignee $MY_ID \
  --scope $KV_ID
```

**Validation checkpoint**: Try `az keyvault secret list --vault-name $KV_NAME` *before* this role assignment propagates (it can take a minute) — expect a `Forbidden` error. Retry after a minute; it should succeed (returning an empty list).

---

## Part 2: Secrets, Keys, and Certificates

### Step 4: Store a Secret

```bash
az keyvault secret set \
  --vault-name $KV_NAME \
  --name db-connection-string \
  --value "Server=tcp:example.database.windows.net;Database=appdb;Authentication=Active Directory Managed Identity"
```

Note the value itself demonstrates the pattern — even the *connection string* uses managed identity auth rather than embedding a username/password.

### Step 5: Create a Key (for Customer-Managed Encryption in Part 3)

```bash
az keyvault key create \
  --vault-name $KV_NAME \
  --name storage-cmk \
  --kty RSA \
  --size 2048
```

### Step 6: Create a Self-Signed Certificate

```bash
az keyvault certificate create \
  --vault-name $KV_NAME \
  --name app-tls-cert \
  --policy "$(az keyvault certificate get-default-policy)"
```

**Validation checkpoint**: `az keyvault certificate show --vault-name $KV_NAME --name app-tls-cert --query "policy.keyProperties" -o table` — confirm it exists and note the key type/size used.

---

## Part 3: Customer-Managed Keys for Storage Encryption

### Step 7: Why Customer-Managed Keys
By default, Azure Storage encrypts data at rest with a Microsoft-managed key — solid encryption, but you have no control over key rotation, no ability to revoke access by disabling the key, and no way to prove key custody for compliance requirements that demand it. A customer-managed key (CMK) puts the key in your Key Vault, under your access policies and audit trail.

### Step 8: Create a Storage Account with a System-Assigned Managed Identity

```bash
az storage account create \
  --name az500lab3$RANDOM \
  --resource-group az500-lab3-rg \
  --location eastus2 \
  --sku Standard_LRS \
  --assign-identity
```

The storage account needs its own identity because *it* is what authenticates to Key Vault to use the encryption key — not you.

### Step 9: Grant the Storage Account's Identity Access to the Key

```bash
STORAGE_ACCOUNT=$(az storage account list -g az500-lab3-rg --query "[0].name" -o tsv)
STORAGE_PRINCIPAL_ID=$(az storage account show -g az500-lab3-rg -n $STORAGE_ACCOUNT --query identity.principalId -o tsv)

az role assignment create \
  --role "Key Vault Crypto Service Encryption User" \
  --assignee $STORAGE_PRINCIPAL_ID \
  --scope $KV_ID
```

### Step 10: Configure the Storage Account to Use the Key

```bash
az storage account update \
  --name $STORAGE_ACCOUNT \
  --resource-group az500-lab3-rg \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault "https://$KV_NAME.vault.azure.net" \
  --encryption-key-name storage-cmk
```

**Validation checkpoint**: `az storage account show -g az500-lab3-rg -n $STORAGE_ACCOUNT --query encryption.keyVaultProperties` should show your key vault URI and key name. If you rotate or revoke the key in Key Vault going forward, this storage account's data becomes unreadable until access is restored — that's the tradeoff of customer control: real control also means real responsibility.

---

## Part 4: Managed Identity for Application-to-Vault Authentication

### Step 11: Create a User-Assigned Managed Identity for "the App"

```bash
az identity create \
  --resource-group az500-lab3-rg \
  --name app-identity

APP_IDENTITY_ID=$(az identity show -g az500-lab3-rg -n app-identity --query principalId -o tsv)
```

### Step 12: Prove the Negative First — Access Should Be Denied
Before granting any role, confirm the identity genuinely has no access. (In a real environment you'd test this from a VM/App Service running as this identity; here we reason about it directly via role assignment list.)

```bash
az role assignment list --assignee $APP_IDENTITY_ID --scope $KV_ID
```

**Expected result**: empty. This identity cannot read anything from the vault yet.

### Step 13: Grant Least-Privilege Access — Secrets Only, Not Keys or Certs
The app only needs the connection string secret. Don't grant it `Key Vault Administrator` or even blanket `Key Vault Secrets Officer` (which can also *delete* secrets) — grant the narrowest role that does the job.

```bash
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $APP_IDENTITY_ID \
  --scope $KV_ID
```

`Key Vault Secrets User` is read-only on secrets and grants nothing on keys or certificates — exactly the app's actual need.

### Step 14: What This Looks Like in Application Code
If this identity were attached to a VM or App Service, the application code would use `DefaultAzureCredential` from the Azure SDK — no connection string, no client secret, nothing stored anywhere:

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(vault_url=f"https://{KV_NAME}.vault.azure.net", credential=credential)
secret = client.get_secret("db-connection-string")
```

`DefaultAzureCredential` automatically picks up the managed identity when running inside Azure. Locally during development, it falls back to your `az login` session — same code, no environment-specific branching.

---

## Part 5: Cleanup

```bash
az group delete --name az500-lab3-rg --yes --no-wait
```

> **Note**: Because purge protection is enabled, the Key Vault itself enters a soft-deleted state for the retention period rather than disappearing immediately. If you need the exact vault name again for a future lab, run `az keyvault purge --name $KV_NAME --location eastus2` to force-remove it (only do this in a disposable lab subscription).

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **RBAC-mode Key Vault** | Consistent, auditable, scopable access model shared with the rest of Azure |
| **Purge protection** | Prevents permanent, unrecoverable loss of every secret in the vault |
| **Customer-managed keys for storage** | Org controls encryption key lifecycle instead of trusting a Microsoft-managed default |
| **Managed identity for app authentication** | Zero credentials in code or config; nothing to leak, rotate, or forget |
| **Least-privilege role scoping (`Secrets User`, not `Administrator`)** | An app compromise doesn't cascade into full vault control |
| **Verifying denial before granting access** | Confirms the "before" state so the role assignment's effect is provable, not assumed |

---

## Common Mistakes to Avoid
- **Using access policies instead of RBAC on new vaults**: loses fine-grained scoping and consistent audit trail
- **Granting `Key Vault Administrator` to an application identity**: administrators can delete and purge — a compromised app identity shouldn't be able to destroy the vault
- **Forgetting purge protection**: without it, a compromised high-privilege account can permanently destroy every secret with no recovery
- **Storing the storage account key or connection string in application config instead of using managed identity**: the exact anti-pattern this lab replaces
- **Not testing that access is actually denied before granting it**: without that baseline, you can't be sure the role assignment did anything

---

## Next Steps
- Set up automatic key rotation policies in Key Vault for the CMK used in Part 3
- Add Key Vault firewall rules or a Private Endpoint (see [Lab 2](lab-2-platform-network-security.md)) so the vault itself isn't reachable from the public internet
- Configure Key Vault diagnostic logging to a Log Analytics workspace and alert on any `SecretGet` or `KeyGet` operation from an unexpected identity (see [Lab 4](lab-4-security-ops-automation.md))
- Extend the managed identity pattern to an actual App Service or Azure Function and test `DefaultAzureCredential` end-to-end

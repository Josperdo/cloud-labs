# Lab 7: SQL & Storage Data Security

Check box if done: [ ]

## Overview
Lab 3 covered encryption at rest with customer-managed keys — solid, but transparent: the database engine handles it, and anyone with a valid query permission sees plaintext. This lab goes one level deeper into data-layer controls that matter specifically for structured, queryable data: Microsoft Defender for SQL catching misconfigurations and suspicious queries, Always Encrypted keeping specific columns unreadable even to the database engine itself, Dynamic Data Masking controlling what a query *result* shows without touching the underlying bytes, and storage encryption scopes giving you per-container key isolation instead of one key for an entire storage account.

**Estimated time**: 75–90 minutes
**Cost**: ~$0–$2 (Azure SQL Database on a serverless/Basic tier for the lab window; Defender for SQL bills per-server while enabled — disable it in Cleanup)

---

## Scenario
A team is standing up a customer database that will hold names, emails, and partial payment card references for support lookups. Compliance has asked for three specific things before this goes anywhere near production: proof that the database itself is being scanned for vulnerabilities and suspicious activity, a column-level encryption story for the most sensitive fields that protects them even from a DBA with full query access, and a masking story so a support agent doing a routine lookup doesn't see a customer's full card number on screen. You're building all three and explaining, clearly, what each one actually protects against — because they protect against different things, and conflating them is a common mistake reviewers will catch.

---

## Objectives
- Deploy Azure SQL Database and enable Microsoft Defender for SQL, reviewing vulnerability assessment findings
- Configure Always Encrypted on a sensitive column and understand deterministic vs. randomized encryption
- Apply Dynamic Data Masking and explain why it is not a substitute for real encryption
- Create storage account encryption scopes for container-level key isolation
- Compare all three controls against what they do and do not protect against

---

## Part 1: Azure SQL Database and Microsoft Defender for SQL

### Step 1: Deploy a Sample Database

```bash
az group create --name az500-lab7-rg --location eastus2

az sql server create \
  --resource-group az500-lab7-rg \
  --name az500sqlsrv$RANDOM \
  --location eastus2 \
  --admin-user sqladmin \
  --admin-password "<placeholder-strong-password>"

SQL_SERVER=$(az sql server list -g az500-lab7-rg --query "[0].name" -o tsv)

az sql db create \
  --resource-group az500-lab7-rg \
  --server $SQL_SERVER \
  --name appdb \
  --sample-name AdventureWorksLT \
  --edition GeneralPurpose \
  --compute-model Serverless \
  --family Gen5 \
  --capacity 1
```

> Serverless compute auto-pauses when idle, which keeps this lab's SQL cost close to $0 outside of active use.

### Step 2: Allow Your Client IP for Testing

```bash
MY_IP=$(curl -s https://ifconfig.me)

az sql server firewall-rule create \
  --resource-group az500-lab7-rg \
  --server $SQL_SERVER \
  --name AllowMyIP \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP
```

**Validation checkpoint**: connect with `sqlcmd -S $SQL_SERVER.database.windows.net -d appdb -U sqladmin -P "<placeholder-strong-password>" -Q "SELECT TOP 5 FirstName, LastName FROM SalesLT.Customer"` — confirm rows return. This is the un-hardened baseline the rest of the lab improves on.

### Step 3: Enable Microsoft Defender for SQL

```bash
az security pricing create --name SqlServers --tier Standard
```

1. Portal → **az500sqlsrv...** → **Security** → **Microsoft Defender for Cloud**
2. Confirm **Microsoft Defender for SQL** shows **Enabled**

### Step 4: Review Vulnerability Assessment Findings
1. Same blade → **Findings** (vulnerability assessment runs automatically once Defender for SQL is enabled, though initial results can take some time to populate)
2. Expect findings similar to: excessive permissions granted to non-admin logins, auditing not enabled, or TDE-related configuration notes on a sample database like AdventureWorksLT
3. Note this is the same posture-first workflow as Lab 4's Defender for Cloud recommendations — CSPM visibility before you touch anything

**Validation checkpoint**: **Findings** tab lists at least a handful of results for the AdventureWorksLT sample schema — confirm you can identify which ones are relevant to a real production concern (excessive `db_owner` grants) versus sample-data noise.

---

## Part 2: Always Encrypted

### Step 5: What Always Encrypted Actually Protects Against
Always Encrypted encrypts specific columns **client-side**, before the data ever leaves the application, using keys the database engine never has access to. This means a compromised or malicious DBA, a stolen backup, or a SQL injection vulnerability that dumps a table all see ciphertext — the plaintext only ever exists in application memory holding the column encryption key. This is a fundamentally different threat model than TDE (Lab 3's storage CMK is the analogous concept, applied at rest transparently by the engine) — TDE protects against someone stealing the physical disk/backup; Always Encrypted protects against someone with legitimate query access to the live database.

### Step 6: Deterministic vs. Randomized Encryption
Always Encrypted supports two encryption types per column, and the choice matters:

| | Deterministic | Randomized |
|---|---|---|
| **Same plaintext produces** | Same ciphertext every time | Different ciphertext every time |
| **Supports equality search / joins / grouping** | Yes | No |
| **Vulnerable to pattern inference** (e.g., an attacker who can see ciphertext frequency can infer which rows share a value) | Yes, some risk for low-cardinality data | No |
| **Best fit** | Columns you need to query/join on (e.g., a lookup key) | Highly sensitive columns with no query requirement (e.g., SSN, card number) |

### Step 7: Configure Column Master Key and Column Encryption Key (Conceptual + T-SQL)
Always Encrypted setup happens through SSMS's wizard or PowerShell in a real environment; the underlying T-SQL shows what's actually happening:

```sql
-- Column master key: references a key stored in Key Vault (from Lab 3's pattern), the database never sees it
CREATE COLUMN MASTER KEY CMK_CustomerData
WITH (
  KEY_STORE_PROVIDER_NAME = 'AZURE_KEY_VAULT',
  KEY_PATH = 'https://<your-keyvault-name>.vault.azure.net/keys/always-encrypted-cmk/<placeholder-key-version>'
);

-- Column encryption key: generated by the client tool, encrypted by the CMK above
CREATE COLUMN ENCRYPTION KEY CEK_CustomerData
WITH VALUES (
  COLUMN_MASTER_KEY = CMK_CustomerData,
  ALGORITHM = 'RSA_OAEP',
  ENCRYPTED_VALUE = <placeholder-encrypted-value-generated-by-client-tool>
);

-- Applying it to a column: card reference is highly sensitive, no query need -> Randomized
ALTER TABLE SalesLT.CustomerPayment
ALTER COLUMN CardReference varchar(25)
COLLATE Latin1_General_BIN2
ENCRYPTED WITH (
  COLUMN_ENCRYPTION_KEY = CEK_CustomerData,
  ENCRYPTION_TYPE = RANDOMIZED,
  ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
);
```

### Step 8: Prove the Negative — a Connection Without the Key Sees Ciphertext
```bash
# A connection string WITHOUT "Column Encryption Setting=Enabled" and without access to the CMK
sqlcmd -S $SQL_SERVER.database.windows.net -d appdb -U sqladmin -P "<placeholder-strong-password>" \
  -Q "SELECT CardReference FROM SalesLT.CustomerPayment"
```

**Expected result**: the `CardReference` column returns opaque binary/ciphertext, not a readable value — because this connection never presented the column encryption key. Only a client configured with `Column Encryption Setting=Enabled` *and* access to the Key Vault-backed column master key can decrypt it, exactly like Lab 3's managed-identity-gated secret access.

---

## Part 3: Dynamic Data Masking

### Step 9: What DDM Does — and Explicitly Does Not Do
Dynamic Data Masking rewrites query *results* on the fly for users without an unmask permission — the underlying bytes in the table are completely unchanged. This is a presentation-layer control, not an encryption control, and the distinction is exactly what an AZ-500 exam question (and a real compliance reviewer) will test: DDM does **not** protect data at rest, does **not** protect data in transit, and is trivially bypassed by anyone with sufficient query permission (e.g., exporting the table via a tool that isn't subject to masking, or simply being granted the `UNMASK` permission). Always Encrypted actually encrypts; DDM just changes what a masked user *sees*.

### Step 10: Apply Masking Rules

```sql
-- Default masking function: full redaction pattern
ALTER TABLE SalesLT.Customer
ALTER COLUMN Phone
ADD MASKED WITH (FUNCTION = 'default()');

-- Email masking function: preserves the first letter and domain suffix pattern
ALTER TABLE SalesLT.Customer
ALTER COLUMN EmailAddress
ADD MASKED WITH (FUNCTION = 'email()');

-- Custom partial masking: shows only the last 4 characters, a common "last 4 of card" pattern
ALTER TABLE SalesLT.CustomerPayment
ALTER COLUMN CardReference
ADD MASKED WITH (FUNCTION = 'partial(0,"XXXX-XXXX-XXXX-",4)');
```

### Step 11: Test as a Masked (Non-Privileged) User

```sql
CREATE USER support_agent WITHOUT LOGIN;
GRANT SELECT ON SalesLT.Customer TO support_agent;

EXECUTE AS USER = 'support_agent';
SELECT FirstName, EmailAddress, Phone FROM SalesLT.Customer;
REVERT;
```

**Expected result**: `EmailAddress` and `Phone` return masked patterns (e.g., `aXXX@XXXX.com`) for `support_agent`, while the same query run as `sqladmin` (or any principal with `UNMASK`) returns the real values.

**Validation checkpoint**: Confirm masking is presentation-only by checking the raw bytes are untouched — `sqladmin`'s query above returns full plaintext even though the column has masking rules applied, because masking evaluates per-querying-principal, not per-row-storage.

---

## Part 4: Storage Account Encryption Scopes

### Step 12: Why Container-Level Scopes Beyond Lab 3's Account-Wide CMK
Lab 3 configured one customer-managed key for an entire storage account — every container, every blob, same key. That's fine for a single-tenant use case, but a multi-tenant application (e.g., SaaS storing each customer's files in a separate container) often needs **per-customer key isolation**: the ability to revoke or rotate one customer's key without touching anyone else's data, and to prove to an auditor that customer A's data was encrypted with a key customer A's contract specifically authorized.

### Step 13: Create a Storage Account and Multiple Encryption Scopes

```bash
az storage account create \
  --name az500lab7$RANDOM \
  --resource-group az500-lab7-rg \
  --location eastus2 \
  --sku Standard_LRS \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

STORAGE_ACCOUNT=$(az storage account list -g az500-lab7-rg --query "[0].name" -o tsv)

az storage account encryption-scope create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group az500-lab7-rg \
  --name tenant-a-scope \
  --key-source Microsoft.Storage

az storage account encryption-scope create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group az500-lab7-rg \
  --name tenant-b-scope \
  --key-source Microsoft.Storage
```

> For a scope backed by a customer-managed key instead of a Microsoft-managed key per scope, set `--key-source Microsoft.Keyvault` and reference a Key Vault key the same way Lab 3 did for the whole account — encryption scopes support the same CMK model, just applied at a finer grain.

### Step 14: Assign Scopes to Containers

```bash
az storage container create \
  --account-name $STORAGE_ACCOUNT \
  --name tenant-a-files \
  --default-encryption-scope tenant-a-scope \
  --deny-encryption-scope-override true

az storage container create \
  --account-name $STORAGE_ACCOUNT \
  --name tenant-b-files \
  --default-encryption-scope tenant-b-scope \
  --deny-encryption-scope-override true
```

`--deny-encryption-scope-override true` locks the container to its assigned scope — nobody uploading a blob can accidentally (or deliberately) write it under a different tenant's key, which is the actual isolation guarantee this feature provides.

**Validation checkpoint**: `az storage container show --account-name $STORAGE_ACCOUNT --name tenant-a-files --query "{scope:properties.defaultEncryptionScope, deny:properties.denyEncryptionScopeOverride}"` — confirm the scope name and `deny: true`. If a compliance requirement ever demands revoking tenant A's data access entirely, disabling `tenant-a-scope`'s key (if CMK-backed) makes every blob under `tenant-a-files` unreadable without touching `tenant-b-files` at all.

---

## Part 5: Cleanup

```bash
# Disable Defender for SQL (subscription-scoped, same pattern as Lab 4's Defender for Storage)
az security pricing create --name SqlServers --tier Free

az group delete --name az500-lab7-rg --yes --no-wait
```

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Microsoft Defender for SQL vulnerability assessment** | Continuous scanning for misconfigurations (excessive grants, missing auditing) instead of a one-time review |
| **Always Encrypted with deterministic vs. randomized columns** | Protects sensitive data even from principals with legitimate query access — a different threat model than at-rest encryption |
| **Dynamic Data Masking** | Reduces incidental exposure in day-to-day query results without claiming to be a real encryption control |
| **Correctly distinguishing DDM from Always Encrypted** | Prevents a false sense of security — a common compliance-review failure point |
| **Storage account encryption scopes** | Per-tenant/per-customer key isolation and revocation at a finer grain than one key for a whole account |

---

## Common Mistakes to Avoid
- **Treating Dynamic Data Masking as encryption**: it only changes what a masked query *result* shows — the raw data is unprotected at rest and in transit, and anyone with `UNMASK` or export tooling bypasses it completely
- **Using deterministic encryption on high-cardinality-unnecessary, highly sensitive columns**: if you don't need to query/join on it, randomized encryption removes the (small but real) pattern-inference risk deterministic encryption carries
- **Assuming Always Encrypted protects against a compromised application**: the app itself holds the column encryption key in memory to do its job — Always Encrypted protects against a compromised *database* (DBA, backup theft, SQL injection dump), not a compromised application tier
- **Using one storage account key for a genuinely multi-tenant workload**: revoking one customer's access means revoking everyone's; encryption scopes fix this at low operational cost
- **Forgetting Defender for SQL is subscription/server-scoped like Defender for Storage in Lab 4**: deleting the resource group doesn't disable the pricing tier — set it back to Free explicitly

---

## Next Steps
- Extend the Key Vault RBAC pattern from [Lab 3](lab-3-data-app-security.md) to scope exactly which identities can read the Always Encrypted column master key — that key is as sensitive as any secret in the vault
- Add SQL auditing (`az sql db audit-policy update`) writing to the same Log Analytics workspace from [Lab 4](lab-4-security-ops-automation.md), and alert on any query against a masked or Always Encrypted column from an unexpected identity
- Apply the encryption-scope pattern to a real multi-tenant container layout and document the key-revocation runbook for an offboarded customer
- Continue to [Lab 8: Sentinel SOAR Playbooks & IR Automation](lab-8-sentinel-soar-automation.md) to automate the response side of everything built across this track

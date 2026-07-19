# Lab 2: Securing Storage & Databases

Check box if done: [ ]

## Overview
Encryption at rest is table stakes — AZ-500 already covered customer-managed keys for storage. What this lab adds is the layer most teams skip: active threat detection on the data plane (Defender for Storage's malware scanning, Defender for SQL's vulnerability assessment) and locking database encryption to a key you control, not just the storage account. By the end, the workload from Lab 1 has a storage account and a SQL database that are both monitored for threats and encrypted with keys you own.

**Estimated time**: 60–90 minutes
**Cost**: ~$1–$4 (Defender for Storage and Defender for SQL bill per-resource, hourly; Azure SQL Database on a serverless tier auto-pauses when idle)

---

## Scenario
The workload's storage account and a new SQL database are about to hold real (simulated) customer data. Encryption at rest alone doesn't tell you if someone's actively exfiltrating data or if the database has an exploitable misconfiguration — that requires active monitoring. You're turning on Defender for Storage and Defender for SQL, locking the storage account's network access down to a range for now, and rotating the SQL database to a customer-managed encryption key.

---

## Objectives
- Enable **Defender for Storage** and understand what its malware scanning and sensitive-data threat detection add over baseline encryption
- Deploy an **Azure SQL Database** and enable **Defender for SQL** (vulnerability assessment + advanced threat protection)
- Configure **Transparent Data Encryption (TDE) with a customer-managed key** for the SQL database
- Restrict a storage account's network access with **firewall rules** (IP and VNet-based, without a Private Endpoint)
- Review and act on findings from both Defender plans

---

## Part 1: Defender for Storage

### Step 1: Create the Resource Group and Storage Account
```bash
az group create --name sc500-lab2-rg --location eastus2

az storage account create \
  --name sc500lab2$RANDOM \
  --resource-group sc500-lab2-rg \
  --location eastus2 \
  --sku Standard_LRS \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --tags workload=sc500-demo-app

STORAGE_ACCOUNT=$(az storage account list -g sc500-lab2-rg --query "[0].name" -o tsv)
STORAGE_ID=$(az storage account show -g sc500-lab2-rg -n $STORAGE_ACCOUNT --query id -o tsv)
```

### Step 2: Why Defender for Storage Over Baseline Encryption Alone
Encryption at rest protects data if a disk is physically stolen — it does nothing if an attacker has valid credentials and is actively uploading malware into a blob container or exfiltrating sensitive files through the normal data plane. Defender for Storage adds **malware scanning on upload**, **sensitive data threat detection** (flags access patterns touching data classified as sensitive), and **activity monitoring** for anomalous access patterns — all things baseline encryption can't see.

### Step 3: Enable Defender for Storage on This Account
```bash
az security pricing create \
  --name StorageAccounts \
  --tier Standard \
  --resource-group sc500-lab2-rg 2>/dev/null || \
az security pricing create --name StorageAccounts --tier Standard
```

> Defender for Storage can be enabled per-subscription (as above) or per-storage-account via `az security pricing create --name StorageAccounts --tier Standard` scoped through the portal's per-resource override — the subscription-level setting is simpler and is what most environments use.

### Step 4: Trigger and Review a Finding
Upload a benign test file that simulates a suspicious pattern, then check recommendations.

```bash
az storage container create \
  --account-name $STORAGE_ACCOUNT \
  --name test-container \
  --auth-mode login

echo "test content" > test-file.txt

az storage blob upload \
  --account-name $STORAGE_ACCOUNT \
  --container-name test-container \
  --name test-file.txt \
  --file test-file.txt \
  --auth-mode login
```

1. Portal → **Microsoft Defender for Cloud** → **Recommendations** → filter by resource type **Storage account** — confirm Defender-for-Storage-specific recommendations now appear (malware scan status, sensitive data discovery configuration) that weren't there under free CSPM alone
2. **Validation checkpoint**: `az security pricing show --name StorageAccounts --query "{tier: pricingTier}"` should return `Standard`

---

## Part 2: Azure SQL Database and Defender for SQL

### Step 5: Deploy a SQL Server and Serverless Database
```bash
az sql server create \
  --name sc500-lab2-sql-$RANDOM \
  --resource-group sc500-lab2-rg \
  --location eastus2 \
  --admin-user sqladmin \
  --admin-password '<StrongP@ssw0rd-Placeholder>'

SQL_SERVER=$(az sql server list -g sc500-lab2-rg --query "[0].name" -o tsv)

az sql db create \
  --resource-group sc500-lab2-rg \
  --server $SQL_SERVER \
  --name workload-db \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 1 \
  --compute-model Serverless \
  --auto-pause-delay 60
```

**Why serverless with a 60-minute auto-pause**: this is a lab database with no real traffic — serverless with auto-pause means you're not billed for idle compute the moment you stop actively working with it.

### Step 6: Enable Defender for SQL
```bash
az security pricing create --name SqlServers --tier Standard
```

### Step 7: Configure Vulnerability Assessment
```bash
SQL_SERVER_ID=$(az sql server show -g sc500-lab2-rg -n $SQL_SERVER --query id -o tsv)

az sql server ad-admin create \
  --resource-group sc500-lab2-rg \
  --server-name $SQL_SERVER \
  --display-name sc500-sql-admin \
  --object-id $(az ad signed-in-user show --query id -o tsv) 2>/dev/null || \
  echo "Optional: skip if you don't need Entra-based SQL admin for this lab"
```

1. Portal → **SQL server** (`$SQL_SERVER`) → **Security** → **Microsoft Defender for Cloud** → confirm **Vulnerability Assessment** is enabled and set to scan periodically
2. Configure a storage account for scan result storage if prompted (can reuse the account from Part 1)

### Step 8: Review Vulnerability Assessment Findings
1. **SQL server** → **Microsoft Defender for Cloud** → **Findings**
2. Note the severity breakdown — a fresh database typically flags baseline items like "Auditing should be enabled" or "Advanced Threat Protection types should be configured for the maximum set"

**Validation checkpoint**: `az security pricing show --name SqlServers --query "{tier: pricingTier}"` should return `Standard`.

---

## Part 3: Transparent Data Encryption with a Customer-Managed Key

### Step 9: Why Rotate TDE to a Customer-Managed Key
Azure SQL encrypts every database with TDE using a Microsoft-managed key by default — solid encryption, but exactly the same limitation as storage's default key from AZ-500 Lab 3: no rotation control, no revocation, no audit trail tied to your own Key Vault. The fix is the same pattern, applied to a different resource type.

### Step 10: Create a Key Vault and Key for TDE
```bash
az keyvault create \
  --resource-group sc500-lab2-rg \
  --name sc500kv$RANDOM \
  --location eastus2 \
  --enable-rbac-authorization true \
  --enable-purge-protection true \
  --retention-days 7

KV_NAME=$(az keyvault list -g sc500-lab2-rg --query "[0].name" -o tsv)
KV_ID=$(az keyvault show -g sc500-lab2-rg -n $KV_NAME --query id -o tsv)

az role assignment create \
  --role "Key Vault Administrator" \
  --assignee $(az ad signed-in-user show --query id -o tsv) \
  --scope $KV_ID

az keyvault key create \
  --vault-name $KV_NAME \
  --name sql-tde-cmk \
  --kty RSA \
  --size 2048
```

### Step 11: Give the SQL Server a Managed Identity and Grant It Key Access
```bash
az sql server update \
  --resource-group sc500-lab2-rg \
  --name $SQL_SERVER \
  --set identity.type=SystemAssigned

SQL_PRINCIPAL_ID=$(az sql server show -g sc500-lab2-rg -n $SQL_SERVER --query identity.principalId -o tsv)

az role assignment create \
  --role "Key Vault Crypto Service Encryption User" \
  --assignee $SQL_PRINCIPAL_ID \
  --scope $KV_ID
```

### Step 12: Point TDE at the Customer-Managed Key
```bash
KEY_ID=$(az keyvault key show --vault-name $KV_NAME --name sql-tde-cmk --query key.kid -o tsv)

az sql server key create \
  --resource-group sc500-lab2-rg \
  --server $SQL_SERVER \
  --kid $KEY_ID

az sql server tde-key set \
  --resource-group sc500-lab2-rg \
  --server $SQL_SERVER \
  --server-key-type AzureKeyVault \
  --kid $KEY_ID
```

**Validation checkpoint**: `az sql server tde-key show --resource-group sc500-lab2-rg --server $SQL_SERVER --query "{type: serverKeyType, uri: uri}"` should show `AzureKeyVault` and your key's URI. As with the storage CMK pattern in AZ-500, revoking this key's access going forward makes the database's data unreadable until access is restored.

---

## Part 4: Network-Restricted Storage Account (Firewall Rules, Not a Private Endpoint)

### Step 13: Why Firewall Rules as an Interim Control
Private Endpoints (covered in [Lab 3](lab-3-network-security.md)) are the strongest network control — but not every workload is ready to adopt them immediately, and IP/VNet firewall rules are a legitimate interim or complementary control that's faster to reason about for a lab like this one.

```bash
az storage account update \
  --name $STORAGE_ACCOUNT \
  --resource-group sc500-lab2-rg \
  --default-action Deny

# Allow only a specific placeholder range — replace with your actual admin range
az storage account network-rule add \
  --account-name $STORAGE_ACCOUNT \
  --resource-group sc500-lab2-rg \
  --ip-address <placeholder-admin-cidr>
```

**Validation checkpoint**: `az storage account show -g sc500-lab2-rg -n $STORAGE_ACCOUNT --query networkRuleSet.defaultAction` should return `Deny`. Attempt `az storage blob list --account-name $STORAGE_ACCOUNT --container-name test-container --auth-mode login` from an IP outside the allowed range (or reason through it if you can't easily test from a second network) — it should be rejected.

---

## Part 5: Cleanup

```bash
az group delete --name sc500-lab2-rg --yes --no-wait

# Subscription-level Defender plans are not removed by deleting the resource group
az security pricing create --name StorageAccounts --tier Free
az security pricing create --name SqlServers --tier Free
```

> **Important**: because purge protection is enabled on the Key Vault, it enters a soft-deleted state rather than disappearing immediately. Purge it explicitly in a disposable lab subscription if you need the name back: `az keyvault purge --name $KV_NAME --location eastus2`.

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Defender for Storage malware scanning / sensitive data detection** | Adds active threat detection on top of encryption at rest, which only protects against physical theft |
| **Defender for SQL vulnerability assessment** | Surfaces exploitable misconfigurations (weak auditing, missing threat protection types) before an attacker finds them |
| **TDE with a customer-managed key** | Same key-custody and revocation control from AZ-500's storage CMK pattern, applied to database encryption |
| **Storage account firewall rules (`default-action Deny`)** | A meaningful network control that doesn't require the additional complexity of a Private Endpoint |
| **Subscription-scoped Defender pricing tiers** | Understanding that billing and the recommendation engine are tied to the plan, not the individual resource, changes how you audit for coverage gaps |

---

## Common Mistakes to Avoid
- **Assuming encryption at rest is sufficient data protection**: it protects against physical media theft only — it does nothing against a credentialed attacker actively misusing the data plane, which is what Defender for Storage/SQL are for
- **Leaving TDE on the Microsoft-managed key when compliance requires key custody**: many regulatory frameworks specifically require customer-managed keys, not just "encryption enabled"
- **Forgetting the SQL server needs its own managed identity before it can use a Key Vault key**: TDE-to-CMK fails silently as a permission error if this step is skipped
- **Treating storage firewall rules as equivalent to a Private Endpoint**: firewall rules restrict by network path but traffic can still traverse the public endpoint; Private Endpoint removes the public path entirely (see Lab 3)
- **Not disabling subscription-level Defender plans after the lab**: they keep billing hourly per protected resource regardless of whether the resource group still exists

---

## Next Steps
- Continue to [Lab 3: Securing Networking for Cloud Workloads](lab-3-network-security.md) to replace this lab's firewall-rule approach with a Private Endpoint for the SQL database
- Revisit [AZ-500 Lab 3](../AZ-500/lab-3-data-app-security.md) if the Key Vault RBAC and managed-identity patterns here felt unfamiliar — this lab assumes that foundation
- Add Defender for SQL's **Advanced Threat Protection** alert types (SQL injection, anomalous login) and route them to the workspace you'll build in [Lab 7](lab-7-defender-cspm-compliance.md)
- Extend the storage firewall rule to a VNet rule instead of an IP rule once the VNet from Lab 3 exists

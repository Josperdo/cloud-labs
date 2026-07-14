# Lab 2: Data Storage Solution Design

Check box if done: [ ]

## Overview
AZ-305 doesn't test whether you can click "Create storage account" — it tests whether you pick the *right* storage service for a workload's access pattern, and the right redundancy tier for a stated recovery point objective (RPO), instead of defaulting to whatever's most familiar. This lab works through a multi-workload scenario with five distinct data needs, builds the decision matrix an architect defends in a design review, then deploys the storage account, lifecycle policy, and customer-managed encryption that decision leads to.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 (LRS/ZRS storage account, minimal Key Vault operations — GZRS and archive tier calls are described but sized to stay near-free)

---

## Scenario
You're the architect for a new e-commerce workload. Product and engineering hand you five requirements at once: unstructured product-image and video uploads from sellers, a set of shared configuration files that multiple front-end VMs need mounted simultaneously, OS and data disks for those same VMs, a globally-distributed session store for the web app so logged-in users get low-latency reads from whichever region they hit, and relational transactional data for an order-processing system that needs ACID guarantees. Leadership has also set a hard requirement: an RPO of 15 minutes across regions for the product-media storage, since a regional outage that loses seller uploads is unacceptable. You need to map each workload to the correct Azure service and pick a redundancy tier that actually satisfies that RPO — not just the cheapest or most familiar option.

---

## Objectives
- Select the correct Azure storage service for five distinct workload shapes based on access pattern and protocol needs
- Choose a redundancy tier against a stated RPO requirement, understanding the cost tradeoff at each tier
- Deploy a storage account with the chosen SKU and validate it
- Configure a blob lifecycle management policy to move data through Hot, Cool, and Archive tiers automatically
- Enable customer-managed key (CMK) encryption via Key Vault and verify it's active

---

## Part 1: Design Decision — Storage Service Selection & Redundancy Tier

### Step 1: Map Each Workload to the Right Azure Service

| Workload | Azure Service | Deciding Factor |
|----------|---------------|------------------|
| Unstructured product media (images, video) | **Blob Storage** | Object storage over REST/HTTPS; no shared filesystem semantics needed; supports lifecycle tiering for cost control at scale |
| Shared config files mounted by several front-end VMs | **Azure Files** | Needs a real filesystem protocol (SMB or NFS) so multiple VMs can mount the same share concurrently — Blob Storage has no native mount/lock semantics for this |
| VM OS and data disks | **Managed Disks** | Block storage attached to a single VM at a time (Premium SSD v2 / Ultra Disk shared is the narrow exception); OS boot and low-latency data I/O require block semantics, not object or file protocols |
| Globally-distributed session store | **Azure Cosmos DB** | Key-value/document access pattern, sub-10ms reads needed from multiple regions, tunable consistency (session or eventual) — a relational engine can't give multi-region active-active writes with this latency profile |
| Relational transactional order data | **Azure SQL Database** | Requires ACID transactions, foreign keys, and joins across normalized order/customer/inventory tables — this is exactly what a relational engine is for; Cosmos DB's consistency models don't map cleanly onto multi-table transactional integrity |

**Why this matters on the exam**: every AZ-305 storage question is an access-pattern question in disguise — "mounted concurrently" points to Files, "joins and transactions" points to SQL, "flat objects" points to Blob. Get the access pattern right first; redundancy is a second, independent decision.

### Step 2: Choose the Redundancy Tier Against the Stated RPO

| Tier | Replication | Protects Against | Typical RPO | Relative Cost |
|------|-------------|-------------------|--------------|----------------|
| **LRS** | 3 copies, single datacenter | Disk/node failure only | N/A (no cross-site replication) | Cheapest baseline |
| **ZRS** | 3 copies across availability zones in one region | Datacenter/AZ-level failure | ~0 for zone failure; no protection from a regional outage | ZRS ≈ 1.25–1.5x LRS |
| **GRS** | LRS + async replication to paired region | Regional outage | ~15 min (Microsoft's typical figure, not a hard SLA) | GRS ≈ 2x LRS |
| **RA-GRS** | GRS + readable secondary endpoint | Regional outage, with continued reads during it | ~15 min | RA-GRS ≈ GRS + read endpoint premium |
| **GZRS** | ZRS in primary region + async replication to secondary region | Zone failure **and** regional outage | ~15 min | Most expensive standard tier |
| **RA-GZRS** | GZRS + readable secondary endpoint | Zone and regional outage, with reads during either | ~15 min | Highest — GZRS + read endpoint premium |

**Reading this against the scenario's RPO of 15 minutes across regions**: GRS, RA-GRS, GZRS, and RA-GZRS all satisfy a 15-minute cross-region RPO on paper — same async geo-replication mechanism underneath. The decision isn't the RPO number, it's what *else* needs protecting. ZRS alone fails the requirement outright (no cross-region replication). Between the geo-tiers, GZRS beats plain GRS because the media storage also needs to survive an availability-zone failure inside the primary region without a failover event — GRS's primary is only LRS-protected, so a zone going down there is still a customer-facing outage even though the data is safe in the secondary region.

### Step 3: Recommendation for This Scenario

**GZRS for the product-media storage account** — the only tier that satisfies both "survive a zone failure with zero customer impact" and the stated 15-minute cross-region RPO in one SKU. Add **RA-GZRS** if the app needs to keep serving reads from the secondary region during a regional outage (worth it for a customer-facing catalog; not worth it for the config-file share, which can tolerate write-side downtime while VMs re-provision).

Part 2's hands-on deployment uses **ZRS**, not GZRS — same mechanics, near-zero cost. Upgrading to GZRS/RA-GZRS is a one-flag change (`--sku Standard_GZRS` or `Standard_RAGZRS`), called out there so the design decision and the lab cost stay separate.

---

## Part 2: Deploy Storage Account with the Chosen Redundancy Tier

### Step 4: Create the Resource Group and Storage Account

```bash
az group create --name az305-lab2-rg --location eastus2

az storage account create \
  --name az305lab2$RANDOM \
  --resource-group az305-lab2-rg \
  --location eastus2 \
  --sku Standard_ZRS \
  --kind StorageV2 \
  --https-only true \
  --min-tls-version TLS1_2
```

`Standard_ZRS` keeps this lab near-free while exercising the same account structure as production. In a real deployment matching Part 1's requirement, swap `--sku Standard_ZRS` for `--sku Standard_GZRS` (or `Standard_RAGZRS` if secondary-region reads are needed) — no other command in this lab changes.

### Step 5: Save the Account Name for Later Steps

```bash
STORAGE_ACCOUNT=$(az storage account list -g az305-lab2-rg --query "[0].name" -o tsv)
echo $STORAGE_ACCOUNT
```

**Validation checkpoint**:

```bash
az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group az305-lab2-rg \
  --query "{sku:sku.name, kind:kind, tls:minimumTlsVersion}" -o table
```

Confirm `sku.name` shows `Standard_ZRS`.

---

## Part 3: Blob Lifecycle Management Policy

### Step 6: Why Automate the Tiering

Product images get hit hard right after a seller uploads them and rarely again after 30–90 days, but they still need to exist for years for order history and returns. Paying Hot-tier rates for cold data is the single most common storage cost mistake — a lifecycle policy moves blobs down the tier ladder automatically based on age, with no application code involved.

### Step 7: Write the Policy JSON

Create `lifecycle-policy.json`:

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "media-tiering",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 730 }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["media/"]
        }
      }
    }
  ]
}
```

`prefixMatch` scopes the rule to the `media/` container path so it never accidentally tiers config or log blobs stored elsewhere in the same account. The 730-day delete is a data-retention decision, not a technical one — confirm the real figure against your org's retention policy before using this in production.

### Step 8: Apply the Policy

```bash
az storage account management-policy create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group az305-lab2-rg \
  --policy @lifecycle-policy.json
```

**Validation checkpoint**:

```bash
az storage account management-policy show \
  --account-name $STORAGE_ACCOUNT \
  --resource-group az305-lab2-rg -o json
```

Confirm the `media-tiering` rule appears with `"enabled": true` and the three actions from Step 7.

---

## Part 4: Customer-Managed Key Encryption via Key Vault

### Step 9: Create or Reuse a Key Vault

If you still have the vault from [AZ-500 Lab 3](../AZ-500/lab-3-data-app-security.md), reuse it and skip to Step 10. Otherwise:

```bash
az keyvault create \
  --resource-group az305-lab2-rg \
  --name az305kv$RANDOM \
  --location eastus2 \
  --enable-rbac-authorization true \
  --enable-purge-protection true \
  --retention-days 7

KV_NAME=$(az keyvault list -g az305-lab2-rg --query "[0].name" -o tsv)
KV_ID=$(az keyvault show -g az305-lab2-rg -n $KV_NAME --query id -o tsv)
```

`--enable-purge-protection true` isn't optional here — Azure Storage refuses to enable CMK encryption against a vault that doesn't have both soft-delete (default-on) and purge protection enabled, because a purged key would make the storage account's data permanently unreadable with no recovery path.

### Step 10: Create the Encryption Key

```bash
az keyvault key create \
  --vault-name $KV_NAME \
  --name storage-cmk-lab2 \
  --kty RSA \
  --size 2048
```

### Step 11: Give the Storage Account an Identity, Then Grant It Access to the Key

```bash
az storage account update \
  --name $STORAGE_ACCOUNT \
  --resource-group az305-lab2-rg \
  --assign-identity

STORAGE_PRINCIPAL_ID=$(az storage account show \
  -g az305-lab2-rg -n $STORAGE_ACCOUNT \
  --query identity.principalId -o tsv)

az role assignment create \
  --role "Key Vault Crypto Service Encryption User" \
  --assignee $STORAGE_PRINCIPAL_ID \
  --scope $KV_ID
```

Grant this role before enabling CMK in the next step, not after — the storage account will fail to activate CMK encryption if it can't yet authenticate to the vault.

### Step 12: Enable CMK Encryption on the Storage Account

```bash
az storage account update \
  --name $STORAGE_ACCOUNT \
  --resource-group az305-lab2-rg \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault "https://$KV_NAME.vault.azure.net" \
  --encryption-key-name storage-cmk-lab2
```

**Validation checkpoint**:

```bash
az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group az305-lab2-rg \
  --query "{keySource:encryption.keySource, vaultUri:encryption.keyVaultProperties.keyVaultUri, keyName:encryption.keyVaultProperties.keyName}" -o table
```

Confirm `keySource` shows `Microsoft.Keyvault` and the vault URI/key name match Steps 9–10. In the portal, **Storage account → Encryption** shows **Encryption type: Customer-managed keys** once this is active.

---

## Cleanup

```bash
# Delete the storage account
az storage account delete \
  --name $STORAGE_ACCOUNT \
  --resource-group az305-lab2-rg \
  --yes

# Delete the Key Vault key (soft-deleted, not gone yet)
az keyvault key delete --vault-name $KV_NAME --name storage-cmk-lab2

# Delete the vault itself if it was dedicated to this lab
az keyvault delete --name $KV_NAME --resource-group az305-lab2-rg

# Purge protection means the vault is soft-deleted, not gone — force-remove it
# only if this is a fully disposable lab subscription
az keyvault purge --name $KV_NAME --location eastus2

# Remove everything else in one shot
az group delete --name az305-lab2-rg --yes --no-wait
```

If you reused the AZ-500 Lab 3 vault, skip the `keyvault delete`/`purge` steps and just remove the key and role assignment so the vault stays intact for other labs.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Redundancy tier (LRS/ZRS/GRS/GZRS)** | How many copies of data Azure keeps and where — single datacenter, across zones, or across regions. Higher tiers cost more and protect against larger blast radii |
| **RPO (Recovery Point Objective)** | Maximum acceptable data loss, measured in time — "RPO of 15 minutes" means you can lose at most the last 15 minutes of writes |
| **RTO (Recovery Time Objective)** | Maximum acceptable time to restore service after a failure — a separate axis from RPO; a tier can meet your RPO and still fail your RTO if failover isn't automatic |
| **Blob access tiers (Hot/Cool/Archive)** | Storage cost/retrieval-latency tradeoff within Blob Storage — Hot is most expensive to store but instant to read; Archive is cheapest to store but takes hours to rehydrate |
| **Lifecycle management policy** | Rule-based automation that moves or deletes blobs based on age or last-access time, with no application code involved |
| **Customer-managed key (CMK)** | Encryption-at-rest key you create and control in Key Vault, versus Microsoft-managed keys where Microsoft controls rotation and you have no revocation ability |
| **Purge protection** | Key Vault setting that blocks permanent deletion during the soft-delete retention window — required before a vault can back CMK storage encryption |
| **RA- prefix (e.g., RA-GRS, RA-GZRS)** | "Read-access" — adds a readable secondary endpoint so the app can serve reads during a regional outage, before or without triggering failover |

---

## Common Mistakes
- **Defaulting to GRS/GZRS for everything "just to be safe"**: if the real requirement is surviving a datacenter/AZ failure and there's no cross-region RPO stated, ZRS meets it at a fraction of the cost — geo-redundancy solves a different problem than zone redundancy
- **Enabling CMK without purge protection on the vault**: the storage account update will fail, or worse, succeed against a vault where the key could later be purged with no recovery path
- **Assuming Archive tier retrieval is instant**: standard rehydration takes hours; only use Archive for data you're confident won't be needed on short notice, and set retrieval priority to "High" (still not instant) if urgency is possible
- **Choosing Managed Disks for a "shared across multiple VMs" requirement**: standard managed disks attach to one VM at a time — that requirement is Azure Files (or Ultra Disk shared, a narrow, expensive exception), not disks
- **Granting the storage account's role assignment after trying to enable CMK instead of before**: the account can't authenticate to the vault yet and the encryption update fails

---

## Next Steps
Continue to [Lab 3: Business Continuity Design](lab-3-business-continuity-design.md) to extend this redundancy-tier thinking into full RTO/RPO-driven backup and Site Recovery design, or go back to [Lab 1: Identity, Governance & Monitoring Design](lab-1-identity-governance-monitoring-design.md) if you haven't done it yet. For the storage fundamentals this lab assumes, see [AZ-104 Lab 3: Storage](../AZ-104/lab-3-storage.md); for Key Vault RBAC and managed identity depth beyond what Part 4 covers, see [AZ-500 Lab 3: Data & Application Security](../AZ-500/lab-3-data-app-security.md).

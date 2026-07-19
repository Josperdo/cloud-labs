# Lab 8: Data Security Architecture Design (Capstone)

Check box if done: [ ]

## Overview
Every lab in this track has orbited the same asset: the finance data an attacker reached in Lab 1's incident. Identity (Lab 2), security operations (Lab 3), compliance (Lab 4), hybrid/multicloud posture (Lab 5), network access (Lab 6), and API security (Lab 7) all exist to protect *something* — this capstone lab designs protection for that something directly: encryption/key management strategy and automated data discovery/classification, then synthesizes all seven prior labs into one Zero Trust architecture with data at the center.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$3 (Key Vault Premium SKU for HSM-backed keys and a small storage account; no compute or hourly-billed network appliance is deployed)

---

## Scenario
The finance data from Lab 1's incident has never had a formal encryption key management decision — it's protected by whatever Azure's platform-managed encryption does by default, which satisfies "encrypted at rest" on a checkbox but gives the org no control over key rotation, no ability to revoke access by destroying a key, and no independent audit trail of who accessed the key material. Separately, nobody has ever formally discovered and classified *where* sensitive data actually lives across the estate — the assumption that "the finance data is on the finance file server" is exactly the kind of assumption that's often wrong once someone actually looks. Leadership's final ask for this engagement: a data protection design, and a one-page synthesis showing how everything built this track actually adds up to Zero Trust, not eight disconnected labs.

---

## Objectives
- Decide between Microsoft-managed keys, customer-managed keys (CMK) in Key Vault, and hold-your-own-key (HYOK/BYOK with HSM) for encryption
- Decide between manual data tagging and automated discovery/classification via Microsoft Purview Data Security Posture Management (DSPM)
- Deploy a Key Vault with customer-managed key encryption for the finance storage account, with purge protection enabled
- Provision automated sensitivity classification for the finance data
- Synthesize identity, network, application, and data designs from Labs 1–7 into a single Zero Trust architecture narrative

---

## Part 1: Design Decision — Key Management Model and Data Discovery Approach

### Decision 1: Microsoft-Managed Keys vs. Customer-Managed Keys (CMK) vs. Hold-Your-Own-Key (HYOK/BYOK with HSM)

| Factor | Microsoft-Managed Keys (default, no configuration) | Customer-Managed Keys (CMK) in Azure Key Vault | Hold-Your-Own-Key (HYOK) / BYOK with dedicated HSM |
|---|---|---|---|
| **Compliance requirement fit** | Satisfies baseline "encryption at rest" but fails any requirement for customer key control, independent audit of key access, or key revocation capability | Satisfies most regulatory requirements needing demonstrable key ownership and rotation control — the tier ISO 27001/NIST 800-53 (Lab 4) expect for sensitive data | Required only for the strictest regimes (e.g., specific government/defense contracts, some financial services mandates) that require keys never leave a customer-controlled HSM boundary |
| **Operational overhead** | None — fully managed, zero rotation responsibility | Moderate — key rotation policy, access policy/RBAC on the vault, and a recovery/availability plan for the vault itself (if Key Vault is unavailable, encrypted data becomes unreadable) | High — dedicated HSM (Azure Dedicated HSM or on-prem HSM with BYOK import) adds infrastructure, licensing, and specialized operational skill requirements |
| **Revocation capability** | None — Microsoft controls the keys; there's no customer action that revokes access to the data by destroying a key | Strong — destroying or disabling the CMK in Key Vault immediately renders the encrypted data unreadable, a genuine "kill switch" | Strongest, but at the cost of the operational overhead above |
| **Cost** | Free | Key Vault cost (low, per-operation) plus the operational cost of managing the vault | Highest — dedicated HSM instances bill substantially more than a standard Key Vault |
| **Fit for this scenario** | Insufficient — the org just committed (Lab 4) to ISO 27001 and NIST 800-53 evidence, both of which expect demonstrable key control for sensitive financial data | Correct fit — meets the compliance bar without the HSM-tier operational burden this org's scale doesn't yet justify | Overkill for this scenario — no stated requirement demands HSM-isolated key custody |

### Decision 2: Manual Data Tagging vs. Purview DSPM Automated Discovery vs. Third-Party DLP

| Factor | Manual Tagging (someone labels files/databases by hand as they're created or discovered) | Microsoft Purview DSPM — Automated Discovery + Sensitivity Labels | Third-Party DLP Tooling (separate vendor, separate console) |
|---|---|---|---|
| **Coverage completeness** | Weak — depends entirely on someone remembering to tag correctly, at creation time, forever; the exact assumption ("finance data is on the finance server") that's often wrong | Strong — scans structured and unstructured data across Microsoft 365, Azure, and (via connectors) multicloud storage for sensitive information types and applies labels automatically based on content, not location assumptions | Variable — coverage depends entirely on the specific vendor's connector maturity for the org's estate |
| **Integration with the rest of this track's controls** | None inherent — a manual tag doesn't automatically inform Conditional Access (Lab 2), Sentinel (Lab 3), or the compliance mapping (Lab 4) | Direct — sensitivity labels feed Conditional Access (label-aware session policies), Sentinel (label-aware alerting on sensitive data movement), and Compliance Manager (Lab 4) improvement actions natively, since they're the same Microsoft Purview/Entra ecosystem | Requires custom integration work to connect findings back into the Conditional Access, Sentinel, and compliance surfaces this track already built |
| **Operational overhead** | High and continuous — every new dataset needs manual review | Low ongoing overhead after initial policy configuration — discovery runs continuously | Moderate — vendor-managed scanning, but integration maintenance falls on the org |
| **Fit for this scenario** | Poor — this is exactly the "we assumed we knew where the data was" gap in the scenario | Strong — directly closes the discovery gap and reinforces every other lab's design instead of operating in isolation | Weaker fit — introduces a disconnected tool into an otherwise unified Microsoft security stack this track has been building |

### Recommendation for This Scenario
**Customer-managed keys in Azure Key Vault** for the finance data and any other classified-sensitive workload, with Key Vault RBAC (not legacy access policies), purge protection, and soft-delete enabled — giving the org genuine key ownership and revocation capability without HSM-tier overhead the scenario doesn't justify. **Microsoft Purview DSPM** for automated discovery and classification — replaces the flawed assumption about where sensitive data lives with continuous, content-based discovery that feeds directly into the Conditional Access, Sentinel, and Compliance Manager surfaces already built in this track. Part 2–3 implement both halves.

---

## Part 2: Deploy Key Vault with Customer-Managed Key Encryption

### Step 1: Create the Key Vault with Purge Protection
```bash
az group create --name sc100-lab8-rg --location eastus

az keyvault create \
  --resource-group sc100-lab8-rg --name sc100-lab8-kv \
  --location eastus --sku premium \
  --enable-purge-protection true \
  --enable-rbac-authorization true \
  --retention-days 90
```
`--enable-purge-protection true` is what makes a CMK-based revocation strategy credible — without it, a compromised or malicious admin could delete both the key *and* its soft-deleted recovery copy, which is destruction, not revocation with an audit trail. `--sku premium` enables HSM-backed keys, satisfying the "genuine key ownership" bar from Part 1 without requiring a fully dedicated HSM instance.

### Step 2: Create the Encryption Key
```bash
az role assignment create \
  --assignee "$(az ad signed-in-user show --query id -o tsv)" \
  --role "Key Vault Crypto Officer" \
  --scope "$(az keyvault show --name sc100-lab8-kv --query id -o tsv)"

az keyvault key create \
  --vault-name sc100-lab8-kv --name finance-data-cmk \
  --kty RSA-HSM --size 3072 \
  --rotation-policy "$(cat <<'EOF'
{
  "lifetimeActions": [{ "trigger": { "timeAfterCreate": "P9M" }, "action": { "type": "Rotate" } }],
  "attributes": { "expiryTime": "P1Y" }
}
EOF
)"
```
The rotation policy enforces the "moderate operational overhead" tradeoff accepted in Part 1 — the key rotates automatically before its 1-year expiry, rather than depending on someone remembering to rotate it manually.

### Step 3: Create the Storage Account and Apply the Customer-Managed Key
```bash
az storage account create \
  --name sc100lab8fin$RANDOM --resource-group sc100-lab8-rg \
  --location eastus --sku Standard_GRS \
  --min-tls-version TLS1_2 --allow-blob-public-access false \
  --assign-identity

STORAGE_ACCOUNT=$(az storage account list -g sc100-lab8-rg --query "[0].name" -o tsv)
IDENTITY_PRINCIPAL_ID=$(az storage account show -n $STORAGE_ACCOUNT -g sc100-lab8-rg --query identity.principalId -o tsv)

az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Key Vault Crypto Service Encryption User" \
  --scope "$(az keyvault show --name sc100-lab8-kv --query id -o tsv)"

az storage account update \
  --name $STORAGE_ACCOUNT --resource-group sc100-lab8-rg \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault "$(az keyvault show --name sc100-lab8-kv --query properties.vaultUri -o tsv)" \
  --encryption-key-name finance-data-cmk
```
The storage account's system-assigned managed identity — not a shared secret — is what authenticates to Key Vault for decryption operations, consistent with the identity-centric design running through every prior lab.

**Validation checkpoint**:
```bash
az storage account show --name $STORAGE_ACCOUNT --resource-group sc100-lab8-rg \
  --query "encryption.keyVaultProperties.keyName" -o tsv
```
Expect `finance-data-cmk`. To prove the revocation capability from Part 1's decision matters, in a non-production test: disabling the key (`az keyvault key set-attributes --vault-name sc100-lab8-kv --name finance-data-cmk --enabled false`) makes the storage account's data immediately unreadable until the key is re-enabled — the "kill switch" that Microsoft-managed keys never offered.

---

## Part 3: Provision Automated Data Classification with Purview DSPM

### Step 1: Create the Purview Account
```bash
az extension add --name purview --upgrade

az purview account create \
  --resource-group sc100-lab8-rg --account-name sc100-lab8-purview \
  --location eastus \
  --managed-resource-group-name sc100-lab8-purview-managed
```

### Step 2: Register the Finance Storage Account as a DSPM Data Source
```bash
PURVIEW_ENDPOINT=$(az purview account show \
  --resource-group sc100-lab8-rg --account-name sc100-lab8-purview \
  --query "properties.endpoints.catalog" -o tsv)

STORAGE_ID=$(az storage account show --name $STORAGE_ACCOUNT --resource-group sc100-lab8-rg --query id -o tsv)

az rest --method PUT \
  --url "$PURVIEW_ENDPOINT/scan/datasources/sc100-lab8-finance-storage?api-version=2022-02-01-preview" \
  --body "{\"kind\":\"AzureStorage\",\"properties\":{\"resourceId\":\"$STORAGE_ID\",\"collection\":{\"referenceName\":\"sc100-lab8-purview\"}}}" \
  --headers "Content-Type=application/json"
```
Grant Purview's managed identity the **Storage Blob Data Reader** role on the storage account (via `az role assignment create`) before running a scan — without read access, DSPM has nothing to classify.

### Step 3: Trigger a Classification Scan
```bash
az rest --method PUT \
  --url "$PURVIEW_ENDPOINT/scan/datasources/sc100-lab8-finance-storage/scans/sc100-lab8-scan?api-version=2022-02-01-preview" \
  --body '{"properties":{"scanRulesetName":"AzureStorage","scanRulesetType":"System"}}' \
  --headers "Content-Type=application/json"

az rest --method POST \
  --url "$PURVIEW_ENDPOINT/scan/datasources/sc100-lab8-finance-storage/scans/sc100-lab8-scan/run?api-version=2022-02-01-preview"
```
**Expected result**: DSPM scans the storage account's content against built-in sensitive information type classifiers (credit card numbers, bank account/routing numbers, SSNs) — this is the automated, content-based discovery from Part 1's Decision 2, replacing the manual assumption about where finance data lives.

**Validation checkpoint**: In **Microsoft Purview → Data Security Posture Management → Data risk overview**, confirm the finance storage account appears with a classification summary. Sensitive data found here is automatically eligible for sensitivity labeling — labels that, once applied, can drive Lab 2's Conditional Access session controls (e.g., block download of "Highly Confidential" labeled content on unmanaged contractor devices) and Lab 3's Sentinel alerting (label-aware exfiltration detection).

---

## Part 4: Zero Trust Architecture Synthesis

This is the deliverable that ties every lab in this track together — the "one-page synthesis" the scenario asked for.

| Zero Trust Pillar | Design Decision | Implemented In | Key Control |
|---|---|---|---|
| **Identity** | Layered Conditional Access + PIM-eligible privileged roles, no standing access | [Lab 2](lab-2-identity-access-architecture-design.md) | Persona-scoped CA policies, JIT role activation |
| **Security Operations** | Multi-workspace Sentinel with residency-compliant topology and tiered retention | [Lab 3](lab-3-security-operations-architecture-design.md) | Scheduled analytics rule detecting credential-based attack patterns |
| **Governance/Compliance** | Unified control framework, automated Policy evidence + procedural Compliance Manager tracking | [Lab 4](lab-4-regulatory-compliance-governance-architecture.md) | NIST 800-53 + ISO 27001 initiatives assigned to the same resource scope |
| **Hybrid/Multicloud** | Password Hash Sync + Arc-extended governance + multicloud CSPM connector | [Lab 5](lab-5-hybrid-multicloud-infrastructure-security-design.md) | On-prem server and AWS subsidiary both visible in one Defender for Cloud posture surface |
| **Network** | SSE (Private Access/Internet Access) for user-to-app traffic, hub-spoke retained for workload traffic | [Lab 6](lab-6-network-security-architecture-design.md) | Per-application, per-port access replacing flat VPN trust |
| **Applications/APIs** | Layered WAF + APIM policies, shift-left SAST/DAST in CI/CD | [Lab 7](lab-7-application-api-security-architecture-design.md) | JWT validation and per-subscription rate limiting at the gateway |
| **Data** | Customer-managed key encryption + Purview DSPM automated classification | This lab | CMK revocation capability, content-based sensitive data discovery |

**The thread connecting all seven**: Lab 1's incident happened because a single compromised credential granted implicit, network-wide, indefinite trust to reach unclassified, Microsoft-key-encrypted finance data with no privileged-access time limit, no layered detection, and no compliance evidence trail. Every subsequent lab replaced exactly one piece of that implicit trust with an explicit, verified, time-bound, or content-aware control — which is the operational definition of "verify explicitly, use least privilege access, assume breach" this track opened with in Lab 1, now demonstrated end to end rather than just defined.

---

## Cleanup
```bash
az group delete --name sc100-lab8-rg --yes --no-wait
```
Purge protection means the Key Vault and its keys enter a recoverable soft-deleted state rather than being immediately destroyed — this is expected behavior, not a stuck deletion. If the resource group name needs to be reused immediately, purge the vault explicitly: `az keyvault purge --name sc100-lab8-kv --location eastus` (only do this if you're certain the key material doesn't need to be recoverable).

---

## What You Practiced

| Concept | Why It Matters |
|---|---|
| CMK vs. Microsoft-managed vs. HYOK trade-offs | SC-100 tests matching the key management tier to the actual compliance requirement, not defaulting to the most extreme option |
| Purge protection as a prerequisite for genuine revocation | Without it, "destroy the key to revoke access" is also "permanently destroy the data with no recovery" |
| Purview DSPM as continuous, content-based discovery | Replaces the common but false assumption that sensitive data location is already known |
| Sensitivity labels as the connective tissue across pillars | Labels drive Conditional Access session controls and Sentinel alerting — data classification isn't an isolated control, it feeds every other pillar |
| Synthesizing multiple designs into one Zero Trust narrative | This is the actual SC-100 exam skill — not any single control, but showing how independent designs reinforce each other |

---

## Common Mistakes to Avoid
- **Choosing CMK or HYOK by default without a stated compliance driver**: HYOK's operational cost is only justified by a specific regulatory mandate — recommending it without one is over-engineering
- **Enabling customer-managed keys without purge protection**: this turns "revoke access" into "irrecoverably destroy the data" if the key is ever deleted by mistake, not just intentionally disabled
- **Treating data classification as a one-time project instead of continuous discovery**: new sensitive data appears constantly; DSPM's value is in the recurring scan, not a single inventory snapshot
- **Applying sensitivity labels without connecting them to Conditional Access or Sentinel**: an unused label is a compliance checkbox, not a control — the value is in the label driving an actual access or detection decision
- **Presenting each lab's design as a standalone control instead of a system**: SC-100 scenario questions frequently require identifying how a gap in one pillar (e.g., network) undermines a control in another (e.g., data) — the synthesis in Part 4 is the skill being tested, not optional color commentary

---

## Next Steps
- This is the capstone lab of the SC-100 track — from here, the operational depth for each pillar lives in [SC-200](../SC-200/README.md) (security operations), [SC-300](../SC-300/README.md) (identity governance), and [SC-500](../SC-500/README.md) (information protection and data governance)
- For the general architecture-design method this entire track applies to security specifically, see [AZ-305](../AZ-305/README.md)
- To revisit the maturity assessment and roadmap this capstone's synthesis was measured against, see [Lab 1: Zero Trust Strategy & Design Decision Frameworks](lab-1-zero-trust-strategy-design.md)

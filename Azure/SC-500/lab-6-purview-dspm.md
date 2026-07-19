# Lab 6: Microsoft Purview Data Security Posture Management

Check box if done: [ ]

## Overview
Lab 5 secured the AI workload's network path and its behavior against malicious input. This lab answers a different question entirely: what data is actually flowing into and out of that AI workload, is any of it sensitive, and would you even know if it were? Microsoft Purview's Data Security Posture Management (DSPM) for AI is built specifically to answer that — it doesn't secure the model, it observes and governs the data the model touches.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$2 (core DSPM and classification features are included with Microsoft 365/Purview licensing already present in most tenants; this lab avoids paid Purview add-ons)

---

## Scenario
The chat feature from Lab 5 is about to go live, and legal has one question nobody can answer yet: could a user paste sensitive data (a customer's SSN, a contract with confidential terms) into the chat, and would anyone know? You're standing up Purview's sensitive-data discovery and classification, turning on DSPM for AI to specifically monitor the AI workload's interactions with that data, and writing a data security policy that acts on what gets found.

---

## Objectives
- Understand what **Microsoft Purview DSPM** covers versus Defender for Cloud CSPM
- Configure **sensitive information type (SIT)** detection and **classification** on a data source
- Enable **DSPM for AI** and understand what it observes in AI application interactions
- Build a **data security policy** that acts on sensitive data reaching an AI workload
- Review DSPM's reporting and recommendations

---

## Part 1: DSPM vs. CSPM — Two Different Postures

### Step 1: Why This Is a Distinct Discipline, Not a Defender for Cloud Feature
Defender for Cloud's CSPM (used throughout this track, and in depth in [Lab 7](lab-7-defender-cspm-compliance.md)) answers "is this *resource* configured securely" — encryption enabled, network access restricted, patches applied. Purview DSPM answers a different question: "where does *sensitive data* live, where does it flow, and who/what can reach it" — regardless of whether the resource holding it is otherwise perfectly configured. A storage account can pass every CSPM check and still be leaking customer PII into an AI model's context window with zero visibility, because CSPM was never designed to look at data content or flow.

### Step 2: Access Purview
Purview is accessed through the [Microsoft Purview portal](https://purview.microsoft.com), separate from the Azure portal — it's a Microsoft 365-integrated governance surface, not an Azure Resource Manager resource with its own resource group.

1. Navigate to `https://purview.microsoft.com` and sign in with the same tenant credentials used for the rest of this track
2. If this is the first time Purview has been accessed in the tenant, allow a few minutes for initial provisioning

---

## Part 2: Sensitive Information Types and Classification

### Step 3: Understand Sensitive Information Types (SITs)
A SIT is a pattern-based (and increasingly AI-based) definition of what "sensitive" means for a given category — built-in SITs cover things like credit card numbers, SSNs, and passport numbers using regex plus checksum validation plus proximity keywords (to reduce false positives from a random 9-digit number that isn't actually an SSN).

1. Purview portal → **Data classification** → **Sensitive info types**
2. Review a built-in SIT, e.g., **U.S. Social Security Number (SSN)** — note the pattern, confidence levels, and supporting evidence it looks for

### Step 4: Create a Custom Sensitive Information Type
The workload has an internal customer ID format the built-in SITs won't recognize. Define one.

1. **Sensitive info types** → **+ Create sensitive info type**
2. **Name**: `Contoso Customer ID`
3. **Pattern**: regex matching the org's format, e.g., `CID-\d{8}` — set via the custom pattern editor
4. **Confidence level**: High, requiring an exact pattern match
5. **Publish**

### Step 5: Register the Workload's Storage Account as a Data Source
```bash
# Confirm the storage account from Lab 2 (or a new one) exists to register as a scan target
az storage account list --query "[].{name:name, resourceGroup:resourceGroup}" -o table
```

1. Purview portal → **Data map** → **Data sources** → **+ Register**
2. Select **Azure Blob Storage**, point it at the storage account from [Lab 2](lab-2-storage-database-security.md)
3. Grant Purview's managed identity the **Storage Blob Data Reader** role on that account so it can scan content (same least-privilege pattern from Lab 1):

```bash
STORAGE_ID=$(az storage account show -g sc500-lab2-rg -n <your-lab2-storage-account> --query id -o tsv)
PURVIEW_PRINCIPAL_ID=$(az purview account show --resource-group <purview-rg> --name <purview-account-name> --query identity.principalId -o tsv 2>/dev/null || echo "<purview-managed-identity-object-id>")

az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee $PURVIEW_PRINCIPAL_ID \
  --scope $STORAGE_ID
```

### Step 6: Run a Classification Scan
1. **Data map** → your registered storage source → **+ New scan**
2. **Scan rule set**: include the built-in **System** classification rule set plus the custom `Contoso Customer ID` SIT from Step 4
3. **Trigger**: Once
4. **Save and run**

**Validation checkpoint**: after the scan completes, **Data map** → **Insights** → review the classification summary — it should report which SITs were found and in how many assets, even if the count is low or zero in this lab's minimal test data.

---

## Part 3: DSPM for AI

### Step 7: What DSPM for AI Specifically Adds
Standard DSPM discovers and classifies data at rest. **DSPM for AI** extends that visibility to AI application *interactions* — it observes prompts and responses flowing through Microsoft 365 Copilot and, via the Purview SDK/API, custom AI applications like the Azure OpenAI-backed chat feature from Lab 5. It reports which users are pasting sensitive data into AI prompts, which AI apps have accessed sensitive files, and where oversharing risk exists (e.g., a user's Copilot session surfacing a document they technically have access to but shouldn't practically see).

### Step 8: Enable DSPM for AI
1. Purview portal → **Data Security Posture Management** → **AI hub** (or **DSPM for AI**)
2. Follow the enablement wizard — it activates AI interaction auditing across connected AI apps in the tenant
3. Review the **Recommendations** tab — a fresh tenant typically surfaces baseline findings like "Sensitivity labels aren't applied to files referenced in AI interactions" or "Oversharing risk detected in [SharePoint/OneDrive] locations"

### Step 9: Review AI Interaction Reports
1. **DSPM for AI** → **Reports** → **Sensitive data in AI interactions** (naming varies by portal version/rollout stage)
2. This surfaces which sensitivity labels/SITs have appeared in prompts or AI app responses across the tenant's connected AI surfaces, and which users/apps were involved

**Note for a lab tenant**: without real production AI traffic and licensed Copilot seats, this report may show little to no data — the goal here is to understand the capability and where you'd look, not necessarily to generate populated findings in a brand-new lab tenant.

### Step 10: Register the Custom Azure OpenAI App (Conceptual)
Custom AI applications (like the Lab 5 chat feature, as opposed to Microsoft 365 Copilot) get DSPM for AI visibility by integrating the **Microsoft Purview SDK** into the application, which logs interactions (prompt/response metadata, not necessarily full content) to Purview's audit and compliance backend.

```python
# Conceptual integration point — the app calls the Purview SDK alongside its
# Azure OpenAI call so the interaction is visible to DSPM for AI reporting.
from azure.purview.datamap import PurviewDataMapClient  # illustrative import

def log_ai_interaction(user_id: str, prompt_classification: str, response_classification: str):
    """
    Called after each chat turn. Sends interaction metadata to Purview so
    DSPM for AI can include this custom app in its sensitive-data-in-AI reporting,
    the same way it natively covers Microsoft 365 Copilot.
    """
    ...
```

This is the architectural link between Lab 5's application and Lab 6's governance layer — DSPM for AI can't see into a custom AI app's data flow unless the app is instrumented to report into it.

### Step 11: Review the Oversharing Assessment
DSPM for AI's most commonly cited real-world finding isn't a malicious prompt injection — it's an AI assistant surfacing a document a user technically has *permission* to see (an over-permissioned SharePoint site, a stale group membership) but has no practical business reason to encounter. This is a permissions hygiene problem that predates AI, but AI assistants make it far more discoverable, since the assistant will actively search and summarize anything in scope.

1. **DSPM for AI** → **Data risk assessment for AI** (or **Oversharing assessment**, depending on rollout)
2. Review the top sites/locations flagged for oversharing risk — typically ranked by how many users have broader access than their activity pattern suggests they need
3. Note that remediating this is an *access* fix (tightening SharePoint/OneDrive permissions), not a *DSPM policy* fix — it's a good illustration of how DSPM for AI surfaces problems that get fixed back in the identity and access layer from [Lab 1](lab-1-identity-access-governance-foundations.md), not inside Purview itself

---

## Part 4: Data Security Policies

### Step 11: Build a Policy That Acts on Sensitive Data Reaching the AI Workload
Discovery and classification are only useful if something acts on the findings.

1. Purview portal → **Data Security Posture Management** → **Policies** (or **Data loss prevention** depending on portal version) → **+ Create policy**
2. **Policy type**: **Data security policy for AI apps** (or equivalent AI-scoped DLP policy)
3. **Name**: `Block SSN and Customer ID in AI Prompts`
4. **Locations**: the AI apps connected in Step 8, plus the custom app registered in Step 10 if using the SDK integration
5. **Conditions**: content matching **U.S. Social Security Number (SSN)** or the custom `Contoso Customer ID` SIT
6. **Action**: **Block** the interaction and notify the user with policy tip text explaining why, or start with **Audit only** to observe impact before enforcing (same audit-before-deny discipline from Lab 1's Azure Policy work)
7. **Save and turn on**

### Step 12: Validate the Policy Concept
Since a fresh lab tenant likely has no real AI interaction volume to test against immediately, validate structurally instead:
1. **Policies** → confirm the policy shows **Status: On** and lists the correct SITs and locations
2. **Activity explorer** (Purview's audit view) → confirm it's configured to log matches once real interactions occur

**Validation checkpoint**: the policy should list both `Contoso Customer ID` and the built-in SSN type under its conditions, and its status should be `On` (or `Audit`, if you intentionally started conservative).

---

## Part 5: Cleanup

```bash
# Remove the Purview role assignment on the storage account
az role assignment delete \
  --assignee $PURVIEW_PRINCIPAL_ID \
  --scope $STORAGE_ID \
  --role "Storage Blob Data Reader"
```

1. Purview portal → **Data map** → delete the registered data source scan if you don't want it recurring
2. **Policies** → turn off or delete the `Block SSN and Customer ID in AI Prompts` policy if this was a disposable lab tenant
3. **Sensitive info types** → the custom `Contoso Customer ID` SIT can remain published at no cost — delete only if you want a clean tenant state
4. Purview itself has no resource group to delete via `az group delete` — it's a tenant-level service, not an ARM resource, so cleanup is policy/scan-level as above

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **DSPM vs. CSPM as distinct disciplines** | CSPM secures the resource; DSPM tracks the data itself, regardless of how well-configured the resource holding it is |
| **Custom sensitive information types** | Built-in SITs cover common patterns (SSN, credit card); most orgs also have proprietary formats worth classifying |
| **DSPM for AI interaction visibility** | Extends classification from data-at-rest to data actually flowing into AI prompts and responses — the specific new risk AI workloads introduce |
| **Instrumenting a custom AI app for DSPM visibility** | Native coverage exists for Microsoft 365 Copilot; a custom app like Lab 5's chat feature needs explicit integration to get the same visibility |
| **Data security policies with audit-then-enforce** | Same governance discipline as Azure Policy in Lab 1, applied to data flowing into AI interactions instead of resource configuration |

---

## Common Mistakes to Avoid
- **Assuming Defender for Cloud's CSPM covers data sensitivity**: it doesn't look at data content at all — DSPM is a separate, necessary layer
- **Enabling DSPM for AI and expecting immediate populated reports in a low-traffic lab tenant**: the capability activates immediately; meaningful findings require real interaction volume
- **Skipping custom SIT creation because built-in types "probably cover it"**: org-specific identifiers (employee IDs, internal customer IDs, project codenames) need custom definitions — built-in SITs only cover common public formats
- **Building a custom AI app and assuming Purview automatically sees its traffic**: unlike Microsoft 365 Copilot, custom apps need explicit SDK/API integration to appear in DSPM for AI reporting
- **Jumping straight to a blocking policy without an audit period**: same risk as any deny-effect guardrail — verify what would have been blocked before you actually block it

---

## Next Steps
- Continue to [Lab 7: Defender for Cloud — CSPM & Regulatory Compliance](lab-7-defender-cspm-compliance.md), which ties this lab's data-layer findings together with resource-layer posture from Labs 1–5
- Revisit [Lab 5: AI Workload Security Fundamentals](lab-5-ai-workload-security.md) if the custom-app SDK integration point in Step 10 was unclear — it's the bridge between the two labs
- Explore Purview's **Insider Risk Management** for AI, which extends beyond DLP-style blocking into behavioral risk scoring for users interacting with sensitive data and AI apps
- Compare this data-centric governance approach against the identity-centric governance in [SC-300](../SC-300/README.md) — both matter, and DSPM increasingly depends on the identity signals SC-300 covers (who is the user generating this AI interaction, and what's their risk level)

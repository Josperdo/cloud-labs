# Lab 4: Secrets & Config Management at Scale

Check box if done: [ ]

## Overview
Applications need two different things that get conflated constantly: secrets (connection strings, API keys — material that grants access) and non-secret configuration (feature flags, environment settings, timeout values — material that changes behavior but isn't sensitive). Treating both the same way either over-secures trivial settings or under-secures real credentials. This lab builds the Key Vault + App Configuration pattern used at real scale, going beyond [AZ-500 Lab 3](../AZ-500/lab-3-data-app-security.md)'s Key Vault fundamentals into how config and secrets are unified behind one read path for the app.

**Estimated time**: 60-75 minutes
**Cost**: ~$0 (Key Vault and App Configuration free tiers cover this lab's usage entirely)

---

## Scenario
An application needs a database connection string (a secret) and several feature-flag-style settings — a maintenance-mode toggle, a request timeout, a beta-feature switch (non-secret config). Right now both are hardcoded in source. You're building the pattern where secrets live in Key Vault, non-secret config lives in App Configuration, App Configuration *references* the Key Vault secret rather than duplicating it, and the app authenticates to both via managed identity — no credentials anywhere in code or config files.

---

## Objectives
- Create a Key Vault with RBAC authorization mode (not the legacy access-policy model) and store a secret
- Create an App Configuration store and add plain, non-secret key-values
- Add a Key-Vault-referenced value in App Configuration so the app has one config read path for both secret and non-secret settings
- Grant a compute resource's managed identity least-privilege access to both stores
- Rotate a secret and confirm the App Configuration reference resolves the new version with zero redeployment

---

## Part 1: Create a Key Vault with RBAC Authorization (Not Access Policies)

### Step 1: Why RBAC Mode Is the Default Now
Key Vault has two authorization models. The legacy **access policy** model is all-or-nothing per vault — an assigned identity gets broad permissions with no per-object scoping, and it's tracked separately from the rest of Azure's audit trail. The **RBAC** model uses the same role assignments as every other Azure resource (`Key Vault Secrets User`, `Key Vault Secrets Officer`, etc.), shows up in the same Activity Log and access reviews as everything else, and supports Conditional Access / PIM being layered onto the role itself rather than a vault-specific permission blob. Microsoft's own guidance is to use RBAC for all new vaults — access policies exist only for backward compatibility.

### Step 2: Create the Vault

```bash
az group create --name lab4-secrets-rg --location eastus2

az keyvault create \
  --resource-group lab4-secrets-rg \
  --name lab4kv$RANDOM \
  --location eastus2 \
  --enable-rbac-authorization true
```

```bash
KV_NAME=$(az keyvault list -g lab4-secrets-rg --query "[0].name" -o tsv)
KV_ID=$(az keyvault show -g lab4-secrets-rg -n $KV_NAME --query id -o tsv)
```

### Step 3: Grant Yourself Access
Same surprise as any RBAC-mode vault: creating it doesn't grant you access. Assign yourself a role before trying to write a secret.

```bash
MY_ID=$(az ad signed-in-user show --query id -o tsv)

az role assignment create \
  --role "Key Vault Secrets Officer" \
  --assignee $MY_ID \
  --scope $KV_ID
```

### Step 4: Store the Secret

```bash
az keyvault secret set \
  --vault-name $KV_NAME \
  --name db-connection-string \
  --value "Server=tcp:example.database.windows.net;Database=appdb;Authentication=Active Directory Managed Identity"
```

**Validation checkpoint**: `az keyvault secret show --vault-name $KV_NAME --name db-connection-string --query id -o tsv` — note the returned URI, including the version segment (e.g., `.../secrets/db-connection-string/<version-id>`). That version ID matters in Part 5.

---

## Part 2: Create an App Configuration Store

### Step 5: Why a Separate Service for Non-Secret Config
Key Vault is built and priced around secret material — it's not designed to hold hundreds of plain settings, doesn't have first-class support for labels/environments, and every read is an audited, throttled operation meant for sensitive data. App Configuration is built for exactly the opposite: cheap, high-volume reads of feature flags and settings, with labeling (dev/staging/prod), point-in-time snapshots, and feature-flag-specific tooling. Splitting the two isn't redundant — it's using each service for what it's optimized for.

```bash
az appconfig create \
  --resource-group lab4-secrets-rg \
  --name lab4-appconfig$RANDOM \
  --location eastus2 \
  --sku free
```

```bash
APPCONFIG_NAME=$(az appconfig list -g lab4-secrets-rg --query "[0].name" -o tsv)
APPCONFIG_ENDPOINT=$(az appconfig show -g lab4-secrets-rg -n $APPCONFIG_NAME --query endpoint -o tsv)
```

### Step 6: Add a Plain (Non-Secret) Key-Value

```bash
az appconfig kv set \
  --name $APPCONFIG_NAME \
  --key "AppSettings:RequestTimeoutSeconds" \
  --value "30" \
  --yes

az appconfig kv set \
  --name $APPCONFIG_NAME \
  --key "AppSettings:BetaFeatureEnabled" \
  --value "false" \
  --yes
```

**Validation checkpoint**: `az appconfig kv list --name $APPCONFIG_NAME -o table` should list both keys with their plain values visible — this is expected; nothing here is sensitive.

---

## Part 3: Add a Key-Vault-Referenced Value in App Configuration

### Step 7: Create the Reference, Not a Copy

```bash
az appconfig kv set-keyvault \
  --name $APPCONFIG_NAME \
  --key "AppSettings:DbConnectionString" \
  --secret-identifier "https://$KV_NAME.vault.azure.net/secrets/db-connection-string" \
  --yes
```

Note the `--secret-identifier` omits the version segment — this intentionally points at "whatever is current" rather than pinning a specific version. Pinning is covered in Part 5.

### Step 8: Why This Matters
App Configuration now holds a **pointer**, not the secret. `az appconfig kv show` for this key returns a JSON content-type payload describing the vault and secret name — the connection string itself never leaves Key Vault and is never stored, logged, or cached in App Configuration. The app gets one unified read path (query App Configuration for every setting, secret or not) while the actual secret material stays governed by Key Vault's access model, audit trail, and rotation lifecycle. This is the core value of the pattern: config sprawl doesn't force a choice between "secrets everywhere" and "app has to know which service to call for which setting."

**Validation checkpoint**:

```bash
az appconfig kv show \
  --name $APPCONFIG_NAME \
  --key "AppSettings:DbConnectionString"
```

Confirm `content_type` is `application/vnd.microsoft.appconfig.keyvaultref+json;charset=utf-8` and `value` is a JSON blob containing the `uri` to the Key Vault secret — not the connection string text itself.

---

## Part 4: Managed Identity Access (No Stored Credentials)

### Step 9: Assign a System-Assigned Managed Identity
Use an App Service (reuse the one from [AZ-305 Lab 4](../AZ-305/lab-4-compute-app-infrastructure-design.md) if you're working through that track too) or a VM — either works for this pattern. Example against an App Service:

```bash
az webapp identity assign \
  --resource-group lab4-secrets-rg \
  --name <your-app-service-name>

APP_PRINCIPAL_ID=$(az webapp identity show \
  --resource-group lab4-secrets-rg \
  --name <your-app-service-name> \
  --query principalId -o tsv)
```

### Step 10: Grant Least-Privilege Roles on Both Stores

```bash
APPCONFIG_ID=$(az appconfig show -g lab4-secrets-rg -n $APPCONFIG_NAME --query id -o tsv)

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $APP_PRINCIPAL_ID \
  --scope $KV_ID

az role assignment create \
  --role "App Configuration Data Reader" \
  --assignee $APP_PRINCIPAL_ID \
  --scope $APPCONFIG_ID
```

`Key Vault Secrets User` is read-only on secrets. `App Configuration Data Reader` is read-only on key-values. Neither can write, delete, or manage either store — a compromised app identity can't tamper with config or destroy secrets, only read what it's scoped to.

### Step 11: What the App's Own Settings Look Like
The application itself stores exactly one non-secret value: the App Configuration endpoint URL. Everything else — including the database connection string — is resolved at runtime:

```python
from azure.identity import DefaultAzureCredential
from azure.appconfiguration.provider import load

credential = DefaultAzureCredential()
config = load(
    endpoint="https://<appconfig-name>.azconfig.io",
    credential=credential,
    key_vault_options={"credential": credential},
)

timeout = config["AppSettings:RequestTimeoutSeconds"]
conn_string = config["AppSettings:DbConnectionString"]  # resolved from Key Vault transparently
```

`DefaultAzureCredential` picks up the App Service's managed identity automatically in Azure and falls back to your `az login` session locally — same code path either way. The App Configuration provider follows the Key Vault reference and resolves the secret using the same credential, so the app never sees a Key Vault URL, client ID, or secret hardcoded anywhere.

**Validation checkpoint**: `az role assignment list --assignee $APP_PRINCIPAL_ID --all -o table` should show exactly the two role assignments above — nothing broader. If you have the app deployed, confirm it starts and reads both a plain setting and the Key-Vault-referenced setting with no connection string in its App Service configuration blade.

---

## Part 5: Secret Rotation

### Step 12: Create a New Secret Version

```bash
az keyvault secret set \
  --vault-name $KV_NAME \
  --name db-connection-string \
  --value "Server=tcp:example.database.windows.net;Database=appdb;Authentication=Active Directory Managed Identity;ConnectRetryCount=3"
```

### Step 13: Understand Key Vault's Versioning Model
Setting a secret with the same name doesn't overwrite anything — it creates a new **version**, and the old version still exists and is still retrievable by its explicit version ID. One version is marked **current**; any read that doesn't pin a specific version ID automatically gets whatever is current at that moment.

### Step 14: How the App Configuration Reference Resolves This
The reference created in Part 3 used a secret identifier **without** a version segment, meaning it always resolves to current. The result: rotating the secret in Key Vault is enough — no change to App Configuration, no redeployment, no app restart required. The next time the app's config provider refreshes (either on its normal polling interval or on next cold start, depending on SDK configuration), it picks up the new connection string automatically.

If you instead need to pin a specific version (e.g., to stage a rotation and cut over deliberately rather than automatically), include the version ID in `--secret-identifier`:

```bash
az appconfig kv set-keyvault \
  --name $APPCONFIG_NAME \
  --key "AppSettings:DbConnectionString" \
  --secret-identifier "https://$KV_NAME.vault.azure.net/secrets/db-connection-string/<version-id>" \
  --yes
```

**Validation checkpoint**: `az keyvault secret list-versions --vault-name $KV_NAME --name db-connection-string -o table` should show two versions. Confirm the second one is flagged as the current/enabled version, and that the App Configuration reference (Step 7's unversioned form) resolves to it without any App Configuration change.

---

## Cleanup

```bash
# Remove role assignments
az role assignment delete --assignee $APP_PRINCIPAL_ID --scope $KV_ID
az role assignment delete --assignee $APP_PRINCIPAL_ID --scope $APPCONFIG_ID
az role assignment delete --assignee $MY_ID --scope $KV_ID

# Delete the App Configuration store
az appconfig delete --name $APPCONFIG_NAME --resource-group lab4-secrets-rg --yes

# Delete the Key Vault
az keyvault delete --name $KV_NAME --resource-group lab4-secrets-rg

# Delete the resource group (removes everything else, including any App Service/VM)
az group delete --name lab4-secrets-rg --yes --no-wait
```

> **Note**: Key Vault has soft-delete enabled by default and cannot be turned off. `az keyvault delete` leaves the vault in a soft-deleted state for the retention period (90 days by default) rather than removing it immediately — during that window the vault **name stays reserved** and can't be reused. If you need the name back or want full removal for a disposable lab subscription, run `az keyvault purge --name $KV_NAME --location eastus2`.

---

## Key Concepts

| Term | Definition |
|------|-----------|
| **RBAC authorization mode** | Key Vault access model using standard Azure role assignments (`Key Vault Secrets User`, etc.) instead of the legacy per-vault access-policy list; scopable, auditable, and consistent with the rest of Azure |
| **Key Vault reference** | An App Configuration key-value whose stored content is a pointer (vault + secret name + optional version) to a Key Vault secret, not the secret's value itself |
| **Managed identity (system-assigned)** | An identity tied to the lifecycle of one specific resource (e.g., an App Service) — created and deleted with it, usable by nothing else |
| **Managed identity (user-assigned)** | A standalone identity resource that can be attached to multiple compute resources and outlives any single one of them |
| **Secret versioning** | Every `az keyvault secret set` on an existing name creates a new version rather than overwriting; old versions remain retrievable by explicit version ID |
| **Current version** | The version returned by any read that doesn't pin a specific version ID; rotation is the act of making a new version current |
| **DefaultAzureCredential** | Azure SDK credential chain that tries managed identity, environment variables, and the local `az login` session in order — same code works in Azure and on a dev machine |
| **App Configuration Data Reader** | Least-privilege RBAC role for reading key-values from an App Configuration store, with no write or delete permission |

---

## Common Mistakes
- **Using access policies instead of RBAC mode on a new vault**: loses fine-grained scoping and the shared Azure audit trail — there's no reason to choose the legacy model for anything new
- **Copying a secret's value into App Configuration instead of referencing it**: recreates the exact sprawl problem this pattern solves — now the secret exists in two places with two rotation paths to keep in sync
- **Granting `Key Vault Administrator` or `Key Vault Secrets Officer` to an application identity**: those roles can delete and manage secrets; an app only needs to read one, so `Key Vault Secrets User` is the correct scope
- **Forgetting soft-delete reserves the vault name after deletion**: a redeployed lab or pipeline that tries to reuse the same vault name will fail until the soft-deleted vault is purged or its retention period expires
- **Pinning every Key Vault reference to a specific version**: defeats the point of rotation-without-redeployment — only pin a version when you deliberately need to stage a cutover instead of rotating automatically

---

## Next Steps
Continue to [AZ-500 Lab 3](../AZ-500/lab-3-data-app-security.md) if you haven't already covered Key Vault fundamentals (RBAC basics, purge protection, customer-managed keys), or [AZ-305 Lab 2](../AZ-305/lab-2-data-storage-design.md) for CMK encryption as another Key Vault use case beyond app secrets. From here, [Lab 5](lab-5-observability-apm.md) adds Application Insights so you can trace how the app actually behaves once it's reading config and secrets this way in production.

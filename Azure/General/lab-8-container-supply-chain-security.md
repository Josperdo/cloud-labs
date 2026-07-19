# Lab 8: Container Supply-Chain Security

Check box if done: [ ]

## Overview
"We scan for vulnerabilities" is not the same claim as "we know an image wasn't tampered with between build and deploy, and we can prove what's inside it." Recent supply-chain incidents (the SolarWinds build-system compromise, the xz-utils backdoor) succeeded precisely because nothing downstream verified that what got deployed was what was actually built — no signature check, no software inventory, no gate stopping an unverified artifact from running. This lab closes that gap end-to-end: push an image to Azure Container Registry, let Microsoft Defender for Containers scan it for known CVEs, generate and inspect a real Software Bill of Materials (SBOM), cryptographically sign the image with Notation and an Azure Key Vault-backed key, and enforce in AKS itself that unsigned images are refused admission — not just flagged in a dashboard nobody reads.

**Estimated time**: 90-110 minutes
**Cost**: ~$1-$4 — reuses [Lab 1](lab-1-aks-fundamentals.md)'s AKS cluster if it's still running (node-hour cost already accounted for there); this lab adds a Basic-tier ACR (a few cents/day), a Key Vault (free-tier operations), and Microsoft Defender for Containers, which bills per vCore-hour on the cluster. Delete everything in Cleanup the same session.

---

## Scenario
Your team ships containers to the AKS cluster from Lab 1, but right now nothing stops an unscanned, unsigned image from reaching that cluster — `kubectl apply` with any image reference anyone has push access to just works. You're building the missing controls: Defender for Containers scanning every image pushed to the registry, an SBOM generated and attached to the image so "what's actually in this container" is answerable without re-pulling and inspecting it by hand, a Notation signature backed by a Key Vault key proving the image came from your build process, and an AKS admission control policy that denies any pod spec referencing an unsigned image — tested in both directions, the same way Lab 1 tested its RBAC deny/allow behavior.

---

## Objectives
- Create an Azure Container Registry and attach it to the Lab 1 AKS cluster
- Enable Microsoft Defender for Containers and review real vulnerability scan findings on a pushed image
- Generate an SBOM for the image with Syft and attach it to the registry as an OCI referrer artifact
- Sign the image with Notation using a Key Vault-backed signing key — no exportable private key anywhere
- Enable AKS Image Integrity and enforce an admission policy that blocks unsigned images, verified with a deny/allow test

---

## Part 1: Reuse the AKS Cluster and Create a Registry

### Step 1: Confirm (or Recreate) the Lab 1 Cluster
```bash
az aks show --resource-group rg-aks-lab --name aks-lab-cluster --query provisioningState -o tsv
```
If this returns `Succeeded`, the cluster from Lab 1 is still up — reuse it. If it returns a "not found" error because you followed Lab 1's Cleanup already, recreate it with Lab 1 Part 1, Steps 1-4 before continuing; everything in this lab assumes `aks-lab-cluster` in `rg-aks-lab` exists.

### Step 2: Create the Container Registry
Basic tier is enough — Notation-based signing stores signatures as standard OCI referrer artifacts, which every ACR tier supports; Premium is only needed for geo-replication or the older Docker Content Trust feature, neither of which this lab uses.

```bash
ACR_NAME=acrsupplylab$RANDOM

az acr create \
  --resource-group rg-aks-lab \
  --name $ACR_NAME \
  --sku Basic
```

### Step 3: Attach the Registry to the Cluster
```bash
az aks update \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster \
  --attach-acr $ACR_NAME
```
This grants the cluster's kubelet identity `AcrPull` on the registry via role assignment — no image pull secret to create or rotate by hand.

### Step 4: Build and Push a Demo Image
Build in the cloud with `az acr build` rather than requiring a local Docker install. The base image is deliberately an old, EOL Node.js release so the vulnerability scan in Part 2 has real findings to show.

`Dockerfile`:
```dockerfile
FROM node:14-alpine
CMD ["node", "-e", "require('http').createServer((_,res)=>res.end('ok')).listen(3000)"]
```

```bash
az acr build \
  --registry $ACR_NAME \
  --image supply-chain-demo:v1 \
  .
```

**Validation checkpoint**:
```bash
az acr repository show-tags --name $ACR_NAME --repository supply-chain-demo -o table
```
Expect `v1` listed.

---

## Part 2: Enable Defender for Containers and Review the Vulnerability Scan

### Step 5: Enable the Defender for Containers Plan
This is a subscription-level setting, not per-registry.

```bash
az security pricing create \
  --name Containers \
  --tier Standard
```

### Step 6: Trigger a Scan
Defender for Containers scans images on push automatically once the plan is enabled, and re-scans periodically for newly-disclosed CVEs. If the plan was enabled *after* Step 4's push, force a re-scan by pushing again:

```bash
az acr build \
  --registry $ACR_NAME \
  --image supply-chain-demo:v1 \
  .
```

Scan results typically take a few minutes to appear after the enablement propagates.

### Step 7: Review the Findings
In the Azure Portal: **Microsoft Defender for Cloud → Recommendations**, filter for "container" — expect **Container registry images should have vulnerability findings resolved** listing `supply-chain-demo:v1` with multiple CVEs (an EOL Node 14 base image reliably has known, unpatched findings). Open the recommendation to see severity, the affected package, and the fixed version where one exists.

**Validation checkpoint**: The recommendation shows `supply-chain-demo:v1` as an unhealthy resource with at least one CVE listed. Note one CVE ID and its affected package — you'll cross-reference it against the SBOM in Part 3.

---

## Part 3: Generate and Review an SBOM

An SBOM is a manifest of every package and library actually inside the image — the concrete answer to "what's in this container," independent of and complementary to the vulnerability scan.

### Step 8: Install Syft
```bash
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
```
(Windows: `winget install anchore.Syft`)

### Step 9: Generate the SBOM Against the Pushed Image
```bash
az acr login --name $ACR_NAME

syft $ACR_NAME.azurecr.io/supply-chain-demo:v1 -o spdx-json > sbom.spdx.json
```
Syft reads the image's layers directly from the registry (using the credentials `az acr login` just cached) and enumerates every OS package and language-level dependency it can identify — no local `docker pull` required.

### Step 10: Review It
```bash
grep -i "<cve-affected-package-name-from-step-7>" sbom.spdx.json
```
Expect the package Defender for Cloud flagged in Step 7 to show up here too, with its exact installed version — the SBOM and the vulnerability scan are describing the same underlying package inventory from two different angles: one lists what's present, the other lists what's present *and* known-exploitable.

### Step 11: Attach the SBOM to the Image
Attach it as an OCI referrer artifact so the SBOM travels with the image in the registry, instead of living only as a local file that can drift out of sync.

```bash
oras attach \
  --artifact-type application/spdx+json \
  $ACR_NAME.azurecr.io/supply-chain-demo:v1 \
  sbom.spdx.json:application/spdx+json
```
(Install ORAS first if needed: `winget install oras-project.oras` or `curl -LO` the release binary from the ORAS GitHub releases page.)

**Validation checkpoint**:
```bash
oras discover -o tree $ACR_NAME.azurecr.io/supply-chain-demo:v1
```
Expect a tree view showing `supply-chain-demo:v1` with the SBOM listed as an attached artifact underneath it.

---

## Part 4: Sign the Image with Notation and Azure Key Vault

### Step 12: Create a Key Vault and a Signing Certificate
The private key never leaves the vault — Notation calls out to Key Vault to perform the signing operation remotely via the `notation-azure-kv` plugin, rather than exporting key material to the local machine.

```bash
az keyvault create \
  --resource-group rg-aks-lab \
  --name kvsignlab$RANDOM \
  --location eastus

KV_NAME=$(az keyvault list -g rg-aks-lab --query "[0].name" -o tsv)
```

Create a self-signed certificate with the key usage a code-signing scenario needs (`digitalSignature`, EKU `1.3.6.1.5.5.7.3.3` — code signing):

```bash
cat > cert_policy.json << 'EOF'
{
  "issuerParameters": { "name": "Self" },
  "keyProperties": { "exportable": false, "keyType": "RSA", "keySize": 2048, "reuseKey": false },
  "x509CertificateProperties": {
    "subject": "CN=supply-chain-lab",
    "keyUsage": ["digitalSignature"],
    "ekus": ["1.3.6.1.5.5.7.3.3"],
    "validityInMonths": 12
  }
}
EOF

az keyvault certificate create \
  --vault-name $KV_NAME \
  --name notation-signing-cert \
  --policy @cert_policy.json
```

### Step 13: Install Notation and the Azure Key Vault Plugin
```bash
curl -Lo notation.tar.gz https://github.com/notaryproject/notation/releases/latest/download/notation_linux_amd64.tar.gz
tar -xzf notation.tar.gz notation && sudo mv notation /usr/local/bin/
```
(Windows: pull the current release from the [notation GitHub releases page](https://github.com/notaryproject/notation/releases).)

Install the `azure-kv` plugin — pull the current release URL and checksum from the [notation-azure-kv releases page](https://github.com/Azure/notation-azure-kv/releases) rather than hardcoding a version here, since plugin releases move independently of `notation` itself:

```bash
notation plugin install --url <current-notation-azure-kv-release-url> --sha256sum <checksum-from-release-page> azure-kv
```

### Step 14: Trust the Certificate Locally
Download the public portion of the cert and register it as a trusted CA in Notation's local trust store:

```bash
az keyvault certificate download \
  --vault-name $KV_NAME \
  --name notation-signing-cert \
  --file notation-cert.crt \
  --encoding PEM

notation cert add --type ca --store supply-chain-lab notation-cert.crt

notation policy import --force << 'EOF'
{
  "version": "1.0",
  "trustPolicies": [{
    "name": "supply-chain-lab",
    "registryScopes": ["*"],
    "signatureVerification": { "level": "strict" },
    "trustStores": ["ca:supply-chain-lab"],
    "trustedIdentities": ["*"]
  }]
}
EOF
```

### Step 15: Sign the Image
```bash
KEY_ID=$(az keyvault certificate show \
  --vault-name $KV_NAME \
  --name notation-signing-cert \
  --query kid -o tsv)

notation sign $ACR_NAME.azurecr.io/supply-chain-demo:v1 \
  --id $KEY_ID \
  --plugin azure-kv
```

### Step 16: Verify the Signature
Signatures bind to a specific content digest, not a mutable tag — resolve and verify by digest, the same reason a `git commit` hash is trusted over a branch name that can move:

```bash
DIGEST=$(az acr repository show \
  --name $ACR_NAME \
  --image supply-chain-demo:v1 \
  --query digest -o tsv)

notation verify $ACR_NAME.azurecr.io/supply-chain-demo@$DIGEST
```

**Validation checkpoint**: `notation verify` reports a successful verification. `oras discover -o tree $ACR_NAME.azurecr.io/supply-chain-demo:v1` now shows both the SBOM and a signature artifact attached to the image.

---

## Part 5: Enforce Admission Control in AKS

Everything so far produces evidence — a scan result, an SBOM, a signature. This step makes that evidence load-bearing: AKS refuses to run a pod referencing an unsigned image, instead of trusting the deploying human to have checked.

### Step 17: Register the Image Integrity Preview Feature
```bash
az feature register \
  --namespace "Microsoft.ContainerService" \
  --name "AKSImageIntegrityPreview"

az feature show \
  --namespace "Microsoft.ContainerService" \
  --name "AKSImageIntegrityPreview" \
  --query properties.state -o tsv
```
Wait until this returns `Registered` (can take several minutes), then refresh the resource provider:
```bash
az provider register --namespace Microsoft.ContainerService
```

### Step 18: Enable the Azure Policy Add-on and Image Integrity
```bash
az aks enable-addons \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster \
  --addons azure-policy

az aks update \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster \
  --enable-image-integrity
```

### Step 19: Assign the Signature-Enforcement Policy
Find the current built-in policy definition for image signature enforcement — its exact name has moved during preview, so look it up rather than hardcoding a definition ID:
```bash
az policy definition list \
  --query "[?contains(displayName, 'Image Integrity') || contains(displayName, 'signed')]" \
  -o table
```
Assign it in **Deny** mode, scoped to the cluster's resource group:
```bash
CLUSTER_ID=$(az aks show -g rg-aks-lab -n aks-lab-cluster --query id -o tsv)

az policy assignment create \
  --name enforce-signed-images \
  --scope $CLUSTER_ID \
  --policy <definition-id-from-previous-command> \
  --params '{"effect":{"value":"Deny"}}'
```
Policy assignments on AKS sync to the in-cluster Gatekeeper constraint on a delay — allow up to 15 minutes before testing.

### Step 20: Confirm Denial of an Unsigned Image
```bash
kubectl create deployment unsigned-test --image=nginx:stable -n demo
```
Expect the pod to stay in a non-running state and `kubectl describe pod -n demo <pod-name>` to show an admission/policy denial event referencing image verification — `nginx:stable` was never signed with this lab's key, so it's correctly refused.

### Step 21: Confirm the Signed Image Is Allowed
```bash
kubectl create deployment signed-test \
  --image=$ACR_NAME.azurecr.io/supply-chain-demo:v1 \
  -n demo
```

**Validation checkpoint**: `kubectl get pods -n demo` shows `signed-test` reach `Running` while `unsigned-test` remains blocked — proof the policy is actually discriminating on signature status, not just failing open or closed for everything.

---

## Cleanup

```bash
# Remove the test deployments
kubectl delete deployment unsigned-test signed-test -n demo

# Remove the admission policy
az policy assignment delete --name enforce-signed-images --scope $CLUSTER_ID

# Disable image integrity and the policy add-on
az aks update --resource-group rg-aks-lab --name aks-lab-cluster --disable-image-integrity
az aks disable-addons --resource-group rg-aks-lab --name aks-lab-cluster --addons azure-policy

# Turn Defender for Containers back off if this was enabled only for this lab
# (subscription-wide setting — confirm nothing else in your subscription relies on it first)
az security pricing create --name Containers --tier Free

# Delete the registry
az acr delete --name $ACR_NAME --resource-group rg-aks-lab --yes

# Delete the Key Vault (soft-delete retains the name for the retention period)
az keyvault delete --name $KV_NAME --resource-group rg-aks-lab
az keyvault purge --name $KV_NAME --location eastus

# Local trust store cleanup
notation cert delete --type ca --store supply-chain-lab notation-signing-cert --all
```

If you're not continuing straight into another lab that needs it, follow Lab 1's Cleanup to remove `aks-lab-cluster` and `rg-aks-lab` itself — the ACR and Key Vault deletions above only remove what this lab added on top of it.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **SBOM (Software Bill of Materials)** | A manifest of every package/library inside an image at a point in time — answers "what's in here," independent of whether any of it is currently known-vulnerable |
| **OCI referrer artifact** | A signature, SBOM, or other attestation attached to an image manifest via the OCI distribution spec — travels with the image in the registry instead of living as a separate untracked file |
| **Notation** | The CLI implementation of the Notary Project (Notary v2) spec for signing and verifying OCI artifacts — signs by content digest, not by tag |
| **`notation-azure-kv` plugin** | Lets Notation delegate the actual signing operation to Azure Key Vault, so the private key stays in the vault's key store and is never exported to disk |
| **Trust policy** | Local configuration telling `notation verify` which trust stores and identities to accept — verification without one configured trusts nothing by default |
| **Digest vs. tag** | A digest (`sha256:...`) is immutable and content-addressed; a tag (`v1`) can be repointed at any time — signatures bind to the digest specifically so a tag repoint can't silently invalidate or misattribute a signature |
| **AKS Image Integrity** | Preview AKS feature that verifies image signatures at admission time via Azure Policy/Gatekeeper, denying pods whose image reference fails verification |
| **Defender for Containers** | Microsoft Defender for Cloud plan that scans images pushed to ACR for known CVEs and surfaces results as Defender for Cloud recommendations |

---

## Common Mistakes
- **Treating vulnerability scanning as the whole supply-chain story**: a clean scan says nothing about whether the image was tampered with after the scan ran — signing and admission enforcement are the pieces that actually close that gap
- **Verifying by tag instead of digest**: `notation verify <image>:v1` checks whatever `v1` currently points to, which can be repointed after signing; always resolve to the digest first, as Step 16 does
- **Exporting the signing key instead of using the Key Vault plugin**: an exportable key defeats the entire point — anyone with the exported key file can forge signatures outside Key Vault's audit trail
- **Assuming Azure Policy on AKS applies instantly**: the in-cluster Gatekeeper constraint syncs from the policy assignment on a delay (minutes, not seconds) — testing immediately after `az policy assignment create` looks like a broken policy when it's actually still propagating
- **Enabling Defender for Containers as a per-lab toggle without checking subscription-wide impact**: it's a subscription-level pricing setting, not scoped to one registry or cluster — confirm nothing else depends on it before disabling in Cleanup

---

## Next Steps
This lab enforces signature checks at the AKS admission boundary; for the pipeline-side half of this story — signing images as part of a CI/CD run instead of by hand, and gating merges on a passing scan — see [AZ-400 Lab 4: Pipeline Security & Compliance](../AZ-400/lab-4-pipeline-security-compliance.md). For the Defender for Containers findings this lab surfaced in Part 2, [SC-500 Lab 4: Securing Compute Workloads](../SC-500/lab-4-compute-workload-security.md) covers the Defender for Servers/JIT side of the same compute-security posture. Back in this folder, this lab assumed [Lab 1 — AKS Fundamentals](lab-1-aks-fundamentals.md) was already built; if you haven't done Lab 3 yet, its OIDC federated-auth pattern is the natural next step for wiring this lab's `az acr build`/`notation sign` steps into a keyless CI pipeline instead of running them by hand.

# Lab 8: Container Supply-Chain Security

Check box if done: [ ]

## Overview
"We scan for vulnerabilities" is not the same claim as "we know an image wasn't tampered with between build and deploy, and we can prove what's inside it." Recent supply-chain incidents (the SolarWinds build-system compromise, the xz-utils backdoor) succeeded precisely because nothing downstream verified that what got deployed was what was actually built — no signature check, no software inventory, no gate stopping an unverified artifact from running. This lab closes that gap end-to-end: push an image to ECR, let Amazon Inspector's enhanced scanning find real CVEs, generate and inspect a real Software Bill of Materials (SBOM), cryptographically sign the image with Sigstore's cosign backed by an AWS KMS key, and enforce in EKS itself — via Kyverno, since EKS has no first-party admission-control equivalent to AKS's Image Integrity feature — that unsigned images are refused admission.

**Estimated time**: 90–110 minutes
**Cost**: ~$1–$4 — reuses [Lab 1](lab-1-eks-fundamentals.md)'s EKS cluster if it's still running (control-plane and node cost already accounted for there); this lab adds ECR storage (cents), Amazon Inspector enhanced scanning (~$0.09 per image scanned plus continuous rescans), and an asymmetric KMS key (~$1/month prorated, plus $0.03/10,000 requests). Delete everything in Cleanup the same session, **then** tear down Lab 1's cluster.

---

## Scenario
Your team ships containers to the EKS cluster from Lab 1, but right now nothing stops an unscanned, unsigned image from reaching that cluster — `kubectl apply` with any image reference anyone has push access to just works. You're building the missing controls: Amazon Inspector scanning every image pushed to ECR, an SBOM generated and attached to the image so "what's actually in this container" is answerable without re-pulling and inspecting it by hand, a cosign signature backed by a KMS key proving the image came from your build process, and an EKS admission policy that denies any pod spec referencing an unsigned image — tested in both directions, the same way Lab 1 tested its IRSA deny/allow behavior.

---

## Objectives
- Create an ECR repository attached to the Lab 1 EKS cluster's pull access, with enhanced (Inspector-powered) scanning enabled
- Review real vulnerability findings from Amazon Inspector on a pushed image
- Generate an SBOM with Syft and attach it to the image in ECR as an OCI referrer artifact
- Sign the image with cosign using an AWS KMS-backed signing key — no exportable private key anywhere
- Install Kyverno and enforce an admission policy that blocks unsigned images, verified with a deny/allow test

---

## Part 1: Reuse the EKS Cluster and Create a Registry

### Step 1: Confirm the Lab 1 Cluster
```bash
aws eks describe-cluster --name eks-lab-cluster --query cluster.status --output text
```
If this returns `ACTIVE`, the cluster from Lab 1 is still up — reuse it. If it errors because you followed Lab 1's Cleanup already, recreate it with Lab 1 Part 1 before continuing; everything here assumes `eks-lab-cluster` exists.

### Step 2: Create the ECR Repository With Enhanced Scanning
```bash
aws ecr create-repository \
  --repository-name supply-chain-demo \
  --image-scanning-configuration scanOnPush=true

aws inspector2 enable --resource-types ECR

aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{"scanFrequency":"CONTINUOUS_SCAN","repositoryFilters":[{"filter":"supply-chain-demo","filterType":"WILDCARD"}]}]'
```
**Basic vs. enhanced scanning matters here**: Basic scanning (ECR's default) runs a one-time, Clair-based CVE check at push time and stops there. **Enhanced scanning hands the job to Amazon Inspector**, which continuously rescans as new CVEs are disclosed — a package that was clean last week and has a CVE published today gets flagged without you pushing a new image. `aws inspector2 enable` is a separate, required step; enhanced scanning silently does nothing without it.

### Step 3: Attach the Registry to the Cluster
```bash
eksctl create iamserviceaccount \
  --cluster eks-lab-cluster \
  --region us-east-1 \
  --namespace demo \
  --name ecr-pull-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly \
  --approve
```
Unlike AKS's `--attach-acr`, EKS nodes' own instance role typically already carries ECR pull permissions via `AmazonEKSWorkerNodePolicy`, so this step is usually redundant for basic pulls — it's included here as the IRSA-scoped equivalent, useful when a specific workload (not every pod on the node) should be the one with explicit pull rights, consistent with Lab 1's "don't put application-level permissions on the shared node role" principle.

### Step 4: Build and Push a Demo Image
Deliberately use an old, EOL Node.js base image so Part 2's scan has real findings to show.

`Dockerfile`:
```dockerfile
FROM node:14-alpine
CMD ["node", "-e", "require('http').createServer((_,res)=>res.end('ok')).listen(3000)"]
```

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

docker build -t <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v1 .
docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v1
```

**Validation checkpoint**:
```bash
aws ecr describe-images --repository-name supply-chain-demo --query "imageDetails[*].imageTags"
```
Expect `v1` listed.

---

## Part 2: Review the Amazon Inspector Findings

### Step 5: Wait for the Scan and Pull Findings
Enhanced scanning typically surfaces initial results within a few minutes of the push.

```bash
aws inspector2 list-findings \
  --filter-criteria '{"ecrImageRepositoryName": [{"comparison": "EQUALS", "value": "supply-chain-demo"}]}' \
  --query "findings[*].{Severity:severity,Title:title,Package:packageVulnerabilityDetails.vulnerablePackages[0].name,FixedVersion:packageVulnerabilityDetails.vulnerablePackages[0].fixedInVersion}"
```

### Step 6: Read the Findings
Expect multiple CVEs at varying severities — an EOL Node 14 base image reliably has known, unpatched findings. Note one CVE and its affected package name; you'll cross-reference it against the SBOM in Part 3.

**Validation checkpoint**: at least one finding with `severity` of `HIGH` or `CRITICAL` is present, tied to a package inside the `node:14-alpine` base layer, not just the application code you added.

---

## Part 3: Generate and Attach an SBOM

An SBOM is a manifest of every package and library actually inside the image — the concrete answer to "what's in this container," independent of and complementary to the vulnerability scan.

### Step 7: Install Syft
```bash
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
```
(Windows: `winget install anchore.Syft`)

### Step 8: Generate the SBOM
```bash
syft <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v1 -o spdx-json > sbom.spdx.json
```
Syft reads the image's layers directly from ECR using the credentials `docker login` cached, and enumerates every OS package and language-level dependency it can identify — no local re-pull required beyond what Docker already has cached from Step 4.

An AWS-native alternative worth knowing: Amazon Inspector can export SBOMs it already generated internally during scanning —
```bash
aws inspector2 create-sbom-export --report-format SPDX_2_3 \
  --resource-filter-criteria '{"ecrImageTags": [{"comparison": "EQUALS", "value": "v1"}]}' \
  --s3-destination bucketName=<your-sbom-export-bucket>,keyPrefix=sboms/
```
Syft is used here instead because it's portable across registries, not ECR-specific.

### Step 9: Cross-Reference the Scan Finding
```bash
grep -i "<cve-affected-package-name-from-step-6>" sbom.spdx.json
```
Expect the package Inspector flagged in Step 6 to show up here too, with its exact installed version — the SBOM and the vulnerability scan describe the same underlying package inventory from two different angles: one lists what's present, the other lists what's present *and* known-exploitable.

### Step 10: Attach the SBOM as an OCI Referrer Artifact
ECR supports OCI referrer artifacts the same way ACR does — attaching the SBOM here means it travels with the image in the registry instead of living only as a local file that can drift out of sync.

```bash
oras attach \
  --artifact-type application/spdx+json \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v1 \
  sbom.spdx.json:application/spdx+json
```
(Install ORAS first if needed: pull the current release binary from the ORAS GitHub releases page.)

**Validation checkpoint**:
```bash
oras discover -o tree <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v1
```
Expect a tree view showing `supply-chain-demo:v1` with the SBOM listed as an attached artifact underneath it.

---

## Part 4: Sign the Image With cosign and AWS KMS

**AWS has no native Notation-equivalent signing service the way Azure pairs Key Vault with Notation.** Sigstore's cosign is the de facto standard here instead, and it happens to support AWS KMS directly as a remote signing backend through an `awskms://` key reference — giving the same "the private key never leaves the vault" property Notation + Key Vault provides on Azure, just via a different tool.

### Step 11: Create an Asymmetric KMS Signing Key
```bash
aws kms create-key \
  --key-usage SIGN_VERIFY \
  --key-spec ECC_NIST_P256 \
  --description "cosign image signing key for supply-chain-demo"

KMS_KEY_ID=$(aws kms list-keys --query "Keys[?KeyId!=null] | [0].KeyId" --output text)
KMS_KEY_ARN=$(aws kms describe-key --key-id $KMS_KEY_ID --query KeyMetadata.Arn --output text)

aws kms create-alias --alias-name alias/cosign-supply-chain-demo --target-key-id $KMS_KEY_ID
```
`SIGN_VERIFY` with `ECC_NIST_P256` is the key type cosign's KMS integration expects — the private key material is generated inside KMS and is **not exportable**, the same hard guarantee an HSM-backed Key Vault key provides on Azure.

### Step 12: Grant cosign's IAM Principal Least-Privilege KMS Access
```bash
aws iam create-policy \
  --policy-name cosign-kms-signing \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["kms:Sign", "kms:GetPublicKey", "kms:DescribeKey"],
      "Resource": "'$KMS_KEY_ARN'"
    }]
  }'
```
Scoped to exactly the three actions cosign needs on exactly this one key — the same least-privilege discipline as every prior lab's IAM policy in this track, not `kms:*`.

### Step 13: Install cosign and Sign
```bash
curl -O -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign-linux-amd64 && sudo mv cosign-linux-amd64 /usr/local/bin/cosign
```
(Windows: `winget install sigstore.cosign`)

Resolve the digest first — signatures bind to a specific content digest, not a mutable tag, the same reason a `git commit` hash is trusted over a branch name that can move:
```bash
DIGEST=$(aws ecr describe-images --repository-name supply-chain-demo --image-ids imageTag=v1 --query "imageDetails[0].imageDigest" --output text)

cosign sign --key awskms:///$KMS_KEY_ARN \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo@$DIGEST
```

### Step 14: Verify the Signature
```bash
cosign verify --key awskms:///$KMS_KEY_ARN \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo@$DIGEST
```

**Validation checkpoint**: `cosign verify` reports a successful verification and prints the signature payload. `oras discover -o tree` now shows both the SBOM and a signature artifact attached to the image.

---

## Part 5: Enforce Admission Control in EKS With Kyverno

Everything so far produces evidence — a scan result, an SBOM, a signature. This step makes that evidence load-bearing: EKS refuses to run a pod referencing an unsigned image, instead of trusting the deploying human to have checked. **Unlike AKS, which layers Azure Policy's Gatekeeper-based admission control natively via an add-on, EKS has no first-party policy engine** — Kyverno (used here) or OPA Gatekeeper must be installed and managed directly, same as any self-managed Kubernetes cluster.

### Step 15: Install Kyverno
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace
```

### Step 16: Push a Second, Unsigned Image to the Same Repository
This is what the policy in Step 17 will deny — a second tag in the trusted repo that was never signed.
```bash
docker tag <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v1 \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v2-unsigned
docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v2-unsigned
```

### Step 17: Create the ClusterPolicy
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: verify-cosign-kms-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:*"
          attestors:
            - entries:
                - keys:
                    kms: "awskms:///<kms-key-arn>"
```
```bash
kubectl apply -f verify-image-signatures.yaml
```
`imageReferences` scopes the check to this specific repository — this is deliberate, not a shortcut. An image reference that doesn't match the pattern isn't evaluated by this rule at all (it's neither allowed nor denied by *this* policy), which is why the test in the next steps uses two tags in the *same* repo rather than comparing against an unrelated public image.

### Step 18: Confirm Denial of the Unsigned Image
```bash
kubectl create deployment unsigned-test \
  --image=<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v2-unsigned \
  -n demo
```
Expect the pod to fail admission — `kubectl describe replicaset -n demo -l app=unsigned-test` shows an admission webhook denial event referencing image verification failure, since `v2-unsigned` was never signed with the KMS key this policy trusts.

### Step 19: Confirm the Signed Image Is Allowed
```bash
kubectl create deployment signed-test \
  --image=<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/supply-chain-demo:v1 \
  -n demo
```

**Validation checkpoint**: `kubectl get pods -n demo` shows `signed-test` reach `Running` while `unsigned-test` never creates a running pod — proof the policy is actually discriminating on signature status, not just failing open or closed for everything in the repository.

---

## Cleanup

```bash
# Remove the test deployments
kubectl delete deployment unsigned-test signed-test -n demo

# Remove the Kyverno policy and Kyverno itself
kubectl delete -f verify-image-signatures.yaml
helm uninstall kyverno --namespace kyverno
kubectl delete namespace kyverno

# Remove the IRSA service account and policy from Part 1
eksctl delete iamserviceaccount --cluster eks-lab-cluster --region us-east-1 --namespace demo --name ecr-pull-sa

# Remove the cosign KMS IAM policy
aws iam delete-policy --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/cosign-kms-signing

# Schedule deletion of the KMS key — KMS keys cannot be deleted immediately,
# only scheduled, and continue billing ~$1/month during the waiting period,
# so use the minimum 7-day window rather than the 30-day default
aws kms schedule-key-deletion --key-id $KMS_KEY_ID --pending-window-in-days 7

# Delete the ECR repository (force removes both images)
aws ecr delete-repository --repository-name supply-chain-demo --force
```

If you're not continuing straight into more Lab 1 work, follow **[Lab 1's Cleanup](lab-1-eks-fundamentals.md#cleanup)** now to remove `eks-lab-cluster` itself — everything above only removes what this lab added on top of it, and the EKS control plane keeps billing $0.10/hr until the cluster is actually deleted.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **SBOM (Software Bill of Materials)** | A manifest of every package/library inside an image at a point in time — answers "what's in here," independent of whether any of it is currently known-vulnerable |
| **OCI referrer artifact** | A signature, SBOM, or other attestation attached to an image manifest via the OCI distribution spec — travels with the image in the registry instead of living as a separate untracked file |
| **Basic vs. enhanced ECR scanning** | Basic runs a one-time Clair-based CVE check at push; enhanced hands scanning to Amazon Inspector, which continuously rescans as new CVEs are disclosed — a real depth difference, not just a naming one |
| **cosign / Sigstore** | The de facto container-signing tool in the AWS ecosystem, filling the role Notation fills on Azure — AWS has no native equivalent service, so this is a third-party tool AWS's KMS integration happens to support well |
| **`awskms://` key reference** | cosign's mechanism for delegating the actual sign/verify operation to AWS KMS, so the private key stays in KMS's key store and is never exported to disk |
| **Digest vs. tag** | A digest (`sha256:...`) is immutable and content-addressed; a tag (`v1`) can be repointed at any time — signatures bind to the digest specifically so a tag repoint can't silently invalidate or misattribute a signature |
| **Kyverno as EKS's admission-control layer** | EKS has no first-party equivalent to AKS Image Integrity — Kyverno (or OPA Gatekeeper) must be installed and managed directly to get signature-verifying admission control |
| **`imageReferences` scoping** | A Kyverno `verifyImages` rule only evaluates images matching its `imageReferences` pattern — an unmatched image isn't blocked *by that rule*, it's simply not checked, which is why the policy must be scoped to the registries you actually trust |

---

## Common Mistakes
- **Treating vulnerability scanning as the whole supply-chain story**: a clean scan says nothing about whether the image was tampered with after the scan ran — signing and admission enforcement are the pieces that actually close that gap
- **Verifying by tag instead of digest**: `cosign verify <image>:v1` checks whatever `v1` currently points to, which can be repointed after signing; always resolve to the digest first, as Step 13 does
- **Enabling enhanced scanning without also running `aws inspector2 enable`**: the registry scanning configuration alone doesn't activate Inspector — both steps are required, and skipping the second leaves scanning silently inert
- **Scoping Kyverno's `imageReferences` to `*`**: requires every single image in the cluster — including `kube-system` control-plane components — to carry a valid signature, which will break the cluster if applied without excluding system namespaces; scope it to your own trusted registries instead, as Step 17 does
- **Letting a KMS key sit in the default 30-day pending-deletion window during cleanup**: it keeps billing ~$1/month throughout — use the minimum 7-day `--pending-window-in-days` for lab/test keys

---

## Next Steps
This lab enforces signature checks at the EKS admission boundary; for the pipeline-side half of this story — signing images as part of a CI run instead of by hand, gated on a passing scan — see [Lab 3: CI/CD with OIDC Federated Auth](lab-3-cicd-oidc.md), and consider wiring this lab's `docker push`/`cosign sign` steps into that pipeline instead of running them manually. For the Amazon Inspector findings this lab surfaced in Part 2, [SCS-C02 Lab 6: Compute & Container Security](../SCS-C02/aws-sec-lab-6-compute-container-security.md) covers Inspector and least-privilege instance roles in more security-specific depth. This lab assumed [Lab 1 — EKS Fundamentals](lab-1-eks-fundamentals.md) was already built and reused its cluster directly — **tear that cluster down now** if you're not continuing into further work, since it's been billing since Lab 1.

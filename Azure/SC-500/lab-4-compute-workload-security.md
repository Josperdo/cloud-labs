# Lab 4: Securing Compute Workloads

Check box if done: [ ]

## Overview
Compute is where most exploitation actually lands — a vulnerable package, an unpatched CVE, a container image nobody scanned before it shipped. This lab covers the compute security baseline for both VM and container workloads: Defender for Servers with vulnerability assessment, Just-in-Time VM access as the last piece of the "no standing open ports" story started in AZ-500 with Bastion, and Defender for Containers on an AKS cluster reviewing image scan results before anything reaches production.

**Estimated time**: 75–90 minutes
**Cost**: ~$2–$6 (Defender for Servers and Defender for Containers bill per-node/hour; a minimal AKS cluster and a small VM add modest compute cost — delete both the same session)

---

## Scenario
The workload now includes a VM running a legacy component and a small AKS cluster running the containerized parts of the app. Neither has been assessed for vulnerabilities, the VM still has a management port technically reachable (even if narrowly scoped), and nobody has looked at whether the container images being deployed have known CVEs baked in. You're closing all three gaps.

---

## Objectives
- Enable **Defender for Servers Plan 2** and review vulnerability assessment findings on a VM
- Configure **Just-in-Time (JIT) VM access** to eliminate standing management-port exposure
- Deploy a minimal **AKS cluster** with **Defender for Containers** enabled
- Review **container image vulnerability scan** results before deployment
- Understand what each Defender compute plan adds over the free CSPM baseline

---

## Part 1: Defender for Servers and Vulnerability Assessment

### Step 1: Deploy a Target VM
```bash
az group create --name sc500-lab4-rg --location eastus2

az network vnet create \
  --resource-group sc500-lab4-rg \
  --name sc500-vnet \
  --address-prefix 10.40.0.0/16 \
  --subnet-name workload-subnet \
  --subnet-prefix 10.40.1.0/24

az vm create \
  --resource-group sc500-lab4-rg \
  --name sc500-legacy-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name sc500-vnet \
  --subnet workload-subnet \
  --public-ip-address "" \
  --admin-username labadmin \
  --generate-ssh-keys \
  --tags workload=sc500-demo-app

az network nsg create --resource-group sc500-lab4-rg --name legacy-vm-nsg

az network vnet subnet update \
  --resource-group sc500-lab4-rg --vnet-name sc500-vnet \
  --name workload-subnet --network-security-group legacy-vm-nsg
```

No public IP, matching the AZ-500 Lab 2 pattern — but note there's still no NSG rule allowing SSH inbound at all yet. That gap gets closed properly with JIT in Part 2, not with a standing allow rule.

### Step 2: Enable Defender for Servers Plan 2
```bash
az security pricing create --name VirtualMachines --tier Standard --sub-plan P2
```

**What Plan 2 adds over Plan 1**: both plans include agentless disk scanning and threat detection; Plan 2 adds **JIT VM access**, **file integrity monitoring**, **adaptive application controls**, and **agentless container posture** — the specific features this lab and Lab 4's container section rely on.

### Step 3: Review Vulnerability Assessment Findings
Defender for Servers Plan 2 includes Microsoft Defender Vulnerability Management at no extra cost.

1. Portal → **Microsoft Defender for Cloud** → **Recommendations** → filter by resource type **Virtual machine**
2. Find **"Machines should have a vulnerability assessment solution"** — it should show as provisioning or already enabled if the built-in MDVM integration activated
3. Once results populate (can take up to a few hours in a live environment), open **Vulnerability Management** blade → review the CVE list for `sc500-legacy-vm`, sorted by severity

**Validation checkpoint**: `az security pricing show --name VirtualMachines --query "{tier: pricingTier, subPlan: subPlan}"` should show `Standard` / `P2`.

---

## Part 2: Just-in-Time VM Access

### Step 4: Why JIT Instead of a Standing NSG Allow Rule
A standing "allow SSH from my IP" NSG rule is exposed to the entire duration it exists — anyone who compromises a device on that allowed range, or any scenario where your IP changes and someone else picks it up later, has a live path in. JIT VM access closes the port by default and opens it only for a specific requester, specific source IP, and specific time window — typically minutes, not indefinitely.

### Step 5: Enable JIT on the VM
```bash
cat > jit-policy.json <<EOF
{
  "virtualMachines": [
    {
      "id": "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/sc500-lab4-rg/providers/Microsoft.Compute/virtualMachines/sc500-legacy-vm",
      "ports": [
        {
          "number": 22,
          "protocol": "*",
          "allowedSourceAddressPrefix": "*",
          "maxRequestAccessDuration": "PT3H"
        }
      ]
    }
  ]
}
EOF

az security jit-policy create \
  --resource-group sc500-lab4-rg \
  --location eastus2 \
  --name default \
  --virtual-machines @jit-policy.json 2>/dev/null || \
  echo "If the CLI extension doesn't support this directly, configure via Portal: Defender for Cloud > Workload protections > Just-in-time VM access > Enable"
```

1. Portal fallback → **Microsoft Defender for Cloud** → **Workload protections** → **Just-in-time VM access** → **Not Configured** tab → select `sc500-legacy-vm` → **Enable JIT on VM** → accept the default port 22 rule → **Add**

### Step 6: Request Time-Boxed Access
1. **Just-in-time VM access** → **Configured** tab → select `sc500-legacy-vm` → **Request access**
2. Set the source IP to your current placeholder IP range and duration to the minimum needed
3. **Open ports**

**Validation checkpoint**: `az network nsg rule list --resource-group sc500-lab4-rg --nsg-name legacy-vm-nsg -o table` — during the approved window, you should see a temporary allow rule scoped to your specific source IP and a near-term expiration; before the request and after it expires, no such rule exists.

---

## Part 3: AKS and Defender for Containers

### Step 7: Deploy a Minimal AKS Cluster
```bash
az aks create \
  --resource-group sc500-lab4-rg \
  --name sc500-aks \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \
  --enable-defender \
  --vnet-subnet-id "$(az network vnet subnet show -g sc500-lab4-rg --vnet-name sc500-vnet -n workload-subnet --query id -o tsv)" \
  --network-plugin azure \
  --tags workload=sc500-demo-app
```

`--enable-defender` wires up the Defender for Containers agent at cluster creation instead of layering it on after the fact — the preferred approach for any cluster you're standing up with security in mind from the start.

### Step 8: Confirm Defender for Containers Is Active
```bash
az security pricing create --name Containers --tier Standard

az aks show --resource-group sc500-lab4-rg --name sc500-aks --query "securityProfile.defender"
```

**Validation checkpoint**: the query above should show `securityMonitoring.enabled: true`.

### Step 9: Get Cluster Credentials and Deploy a Test Workload
```bash
az aks get-credentials --resource-group sc500-lab4-rg --name sc500-aks

kubectl create deployment nginx-test --image=nginx:1.25
```

Using a specific image tag (`nginx:1.25`) rather than `latest` is itself a security-relevant choice — `latest` makes vulnerability scanning and rollback both harder to reason about since the underlying image can change without a version bump.

### Step 10: Review Container Image Vulnerability Findings
1. Portal → **Microsoft Defender for Cloud** → **Recommendations** → filter by resource type **Container image / Kubernetes**
2. Find **"Container registry images should have vulnerability findings resolved"** or **"Running container images should have vulnerability findings resolved"**
3. Open the finding for the `nginx` image → review CVEs by severity

**This is the point of scanning before, not after, deployment**: in a real pipeline, this same scan runs against the image in your registry *before* it's allowed to deploy, gated in CI — reviewing it post-deployment here is for visibility into what the scanner reports, not the ideal place to catch it operationally.

---

## Part 4: Cleanup

```bash
az group delete --name sc500-lab4-rg --yes --no-wait

# Subscription-level Defender plans are not removed by deleting the resource group
az security pricing create --name VirtualMachines --tier Free
az security pricing create --name Containers --tier Free
```

> **Important**: confirm the AKS cluster is actually gone before considering this lab closed — `az aks list --resource-group sc500-lab4-rg` — since it's the highest per-hour cost in this lab alongside Defender for Servers Plan 2.

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Defender for Servers Plan 2 vulnerability assessment** | Surfaces exploitable CVEs on running VMs instead of relying on manual patch tracking |
| **Just-in-Time VM access** | Eliminates standing management-port exposure entirely — a port only opens for an approved requester, IP, and time window |
| **Defender for Containers at cluster creation (`--enable-defender`)** | Security monitoring is active from the cluster's first day instead of a gap between creation and remediation |
| **Container image vulnerability scanning** | Catches known-CVE base images and dependencies before (ideally) or after deployment, closing a gap CSPM alone doesn't cover |
| **Pinned image tags instead of `latest`** | Makes vulnerability scan results and rollback decisions meaningful and reproducible |

---

## Common Mistakes to Avoid
- **Opening a standing NSG rule for SSH "just for the lab" instead of using JIT**: defeats the entire point of this lab's Part 2 and reintroduces the exact exposure AZ-500 Lab 2 eliminated with Bastion
- **Assuming Defender for Servers Plan 1 covers vulnerability assessment**: JIT, file integrity monitoring, and the bundled MDVM integration are Plan 2 features specifically
- **Enabling Defender for Containers after the cluster already has workloads running**: retrofitting security monitoring means everything deployed before that point had zero coverage — enable it at creation when possible
- **Deploying with `latest` tags and being surprised vulnerability scans are inconsistent run to run**: pin versions so a scan result is reproducible and a rollback target is unambiguous
- **Forgetting AKS bills continuously per node regardless of Defender status**: delete the cluster promptly, don't just disable the Defender plan

---

## Next Steps
- Continue to [Lab 5: AI Workload Security Fundamentals](lab-5-ai-workload-security.md) — the compute pattern here (network isolation, active threat detection, scanning before trust) repeats for AI workloads with AI-specific controls layered on top
- Revisit [AZ-500 Lab 2](../AZ-500/lab-2-platform-network-security.md) for the Bastion-based admin access pattern this lab's JIT approach complements
- Add Adaptive Application Controls (Defender for Servers Plan 2) to allowlist expected processes on `sc500-legacy-vm` and alert on anything outside that baseline
- Wire container image scan failures into a CI/CD gate so vulnerable images never reach the cluster in the first place, rather than being caught post-deployment

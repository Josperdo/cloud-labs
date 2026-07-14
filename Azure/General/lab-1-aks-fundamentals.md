# Lab 1: AKS Fundamentals

Check box if done: [ ]

## Overview
Kubernetes shows up in more cloud job postings than almost any other single keyword, and no Azure certification path (AZ-104, AZ-305, AZ-500) covers it beyond a surface-level mention. This lab exists to close that gap directly: stand up a real AKS cluster, deploy and expose a workload the way it's actually done in production, and govern access through Entra ID instead of a static admin credential.

**Estimated time**: 75-90 minutes
**Cost**: ~$1-$3 — **the AKS control plane itself is free on the Free tier, but node pool VMs and the Standard Load Balancer/public IP created in Part 3 bill hourly the moment they exist.** Use a single small node, don't leave the cluster running between sessions, and follow the Cleanup section the same day you build this.

---

## Scenario
You need to stand up your first real AKS cluster and prove you can run something on it end-to-end — not just click through the Azure portal wizard. That means deploying a containerized app, exposing it properly through an ingress controller instead of a `LoadBalancer` Service per app, and making sure cluster access is governed by who someone is in Entra ID rather than whoever happens to have a copy of the admin kubeconfig file. This lab builds all three, then proves the RBAC piece actually works by testing it with a second identity.

---

## Objectives
- Create an AKS cluster sized for cost, not production load
- Practice core `kubectl` operations: deployments, pods, services
- Install Helm and use it to deploy the ingress-nginx controller
- Expose an application through a Kubernetes Ingress resource
- Configure Entra ID-integrated cluster RBAC and verify it actually blocks/allows access as expected
- Tear the whole thing down before it accumulates node-hour cost

---

## Part 1: Create a Cost-Conscious AKS Cluster

### Step 1: Create the Resource Group
```bash
az group create \
  --name rg-aks-lab \
  --location eastus
```

### Step 2: Create the Cluster
One system node pool, one small VM, Entra ID-integrated Azure RBAC instead of the default static admin credential.

```bash
az aks create \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster \
  --location eastus \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --enable-aad \
  --enable-azure-rbac \
  --generate-ssh-keys
```

- `--enable-aad` turns on Entra ID authentication for the cluster's Kubernetes API — users sign in with their Entra identity instead of a shared `kubeconfig` bearer token.
- `--enable-azure-rbac` moves authorization into Azure RBAC, so `kubectl` permissions come from `az role assignment create` against built-in roles like **Azure Kubernetes Service RBAC Reader**. Together these flags eliminate the biggest AKS anti-pattern: a `kubeconfig --admin` file that grants cluster-admin to anyone who has the file, with no per-user audit trail and no way to revoke just one person's access.

**Cheaper alternative — spot node pool**: if you want to shave the node cost further, add a spot-priced node pool instead of (or alongside) the default one:
```bash
az aks nodepool add \
  --resource-group rg-aks-lab \
  --cluster-name aks-lab-cluster \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --no-wait
```
Spot nodes run at a discount but can be **evicted with as little as 30 seconds' notice** whenever Azure needs the capacity back. Fine here since everything is stateless and disposable; never use spot for a workload you can't afford to lose mid-request.

### Step 3: Grant Yourself an Azure RBAC Role on the Cluster
With `--enable-azure-rbac` on, being the cluster *creator* — or even Owner/Contributor on the resource group — does **not** grant `kubectl` access. Data-plane access is a separate Azure RBAC role assignment against the cluster resource.

```bash
CLUSTER_ID=$(az aks show \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster \
  --query id -o tsv)

MY_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

az role assignment create \
  --assignee-object-id $MY_OBJECT_ID \
  --assignee-principal-type User \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope $CLUSTER_ID
```

### Step 4: Get Credentials and Sign In
```bash
az aks get-credentials \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster \
  --overwrite-existing
```

The first `kubectl` command against this cluster triggers an interactive Entra ID device-code sign-in (a URL and a short code print to the terminal) — this is the per-user authentication `--enable-aad` buys you, replacing the shared static credential entirely.

**Validation checkpoint**:
```bash
kubectl get nodes
```
Expect one node in `Ready` status. If you get a `Forbidden` error, the role assignment from Step 3 hasn't propagated yet — wait a minute and retry.

---

## Part 2: kubectl Fundamentals — Deploy a Sample App

### Step 5: Create a Namespace
Namespaces are how Part 4's RBAC scoping works, so create one now instead of using `default`.

```bash
kubectl create namespace demo
```

### Step 6: Deploy the App
```bash
kubectl create deployment demo-app \
  --image=nginx:stable \
  --namespace=demo
```

### Step 7: Inspect What Got Created
```bash
kubectl get deployments -n demo
kubectl get pods -n demo
```
A `Deployment` manages a `ReplicaSet`, which manages the actual `Pod`(s) — this is why `kubectl get pods` shows a pod with a generated suffix rather than the deployment name itself.

### Step 8: Expose It Internally
Write the Service as a manifest rather than `kubectl expose` so it's reusable and reviewable.

`demo-svc.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app-svc
  namespace: demo
spec:
  selector:
    app: demo-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f demo-svc.yaml
```

`ClusterIP` is intentionally **not** internet-reachable. Part 3 handles external exposure through an ingress controller instead of switching this to `type: LoadBalancer`, which would provision a separate public IP and LB per app.

**Validation checkpoint**:
```bash
kubectl get pods -n demo        # STATUS: Running
kubectl get svc -n demo         # demo-app-svc, TYPE: ClusterIP
```

---

## Part 3: Install Helm and an Ingress Controller

### Step 9: Install the Helm CLI
```powershell
winget install Helm.Helm
helm version
```
(On Linux/macOS: `curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`)

### Step 10: Add the ingress-nginx Repo and Install It
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

This provisions the NGINX ingress controller **and** an Azure-backed `LoadBalancer` Service in front of it — a Standard Load Balancer and public IP that bill hourly on top of the node VM. Small (a few cents/hour combined), but it's a second cost source in this lab, not just the node.

### Step 11: Wait for the Public IP
```bash
kubectl get service ingress-nginx-controller \
  --namespace ingress-nginx \
  --watch
```
Wait until `EXTERNAL-IP` moves from `<pending>` to an actual address, then `Ctrl+C`.

### Step 12: Create the Ingress Resource
An `Ingress` resource does nothing without a controller already watching for it — Step 10 has to happen first, or this manifest just sits there inert.

`demo-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-app-svc
                port:
                  number: 80
```

```bash
kubectl apply -f demo-ingress.yaml
```

**Validation checkpoint**:
```bash
kubectl get ingress -n demo
# note the ADDRESS column — this is the same external IP from Step 11

curl http://<ingress-external-ip>/
```
Expect the default NGINX welcome page HTML back. This confirms the full path works: external request → Azure Load Balancer → ingress controller → Ingress rule → Service → Pod.

---

## Part 4: Verify Entra ID RBAC Is Actually Governing Access

The point of `--enable-aad`/`--enable-azure-rbac` isn't just that it sounds more secure — it's that access decisions are enforced by Azure RBAC role assignments, auditable in the Activity Log, and revocable per user without touching a shared credential. Prove that by testing with a second identity instead of taking it on faith.

### Step 13: Confirm the Second User Is Currently Blocked
Using an existing test account (or a throwaway one created with `az ad user create`), have that user run `az aks get-credentials` against this same cluster and attempt:
```bash
kubectl get pods -n demo
```
Expect a `Forbidden` error — the request reaches the API server and authenticates successfully via Entra ID, but Azure RBAC has no role assignment for this user against the cluster, so it's denied. This is the behavior that a static admin `kubeconfig` file can never give you: authentication and authorization are fully decoupled, and "the app deployed" is worthless as identity proof.

### Step 14: Grant Read-Only Access
```bash
TEST_USER_ID=$(az ad user show --id <test-user-upn> --query id -o tsv)

az role assignment create \
  --assignee-object-id $TEST_USER_ID \
  --assignee-principal-type User \
  --role "Azure Kubernetes Service RBAC Reader" \
  --scope $CLUSTER_ID
```

**Advanced/optional**: scope this to just the `demo` namespace instead of the whole cluster using an ABAC condition on the role assignment (`Microsoft.ContainerService/managedClusters:namespace`) — pull the current syntax from Microsoft Learn's "Use Azure RBAC for Kubernetes Authorization" page rather than hand-typing it, then verify with Step 15 that it actually restricts to `demo`.

### Step 15: Confirm Access Now Works — and Confirm It's Read-Only
Wait a minute for the role assignment to propagate, then re-run as the test user:
```bash
kubectl get pods -n demo
```
Expect success this time. Then confirm the **Reader** role is genuinely read-only, not a rubber stamp:
```bash
kubectl delete pod <pod-name> -n demo
```
Expect another `Forbidden` — Reader grants `get`/`list`/`watch`, not `delete`. This is the difference between "RBAC is configured" and "RBAC actually constrains what this identity can do," and it's worth testing both directions any time you stand up a permission model.

**Validation checkpoint**: `az role assignment list --scope $CLUSTER_ID -o table` shows the test user's Reader assignment, and both `kubectl` outcomes above match what the role should and shouldn't allow.

---

## Cleanup
Node VMs and the Load Balancer/public IP from Part 3 bill by the hour from the moment they're created — do this the same session, not "later today."

```bash
# Remove the ingress controller (also deprovisions the LB + public IP)
helm uninstall ingress-nginx --namespace ingress-nginx

# Optional — the app/Service/Ingress are namespaced and get deleted with the
# cluster anyway, but tearing them down explicitly is good practice
kubectl delete namespace demo

# Delete the cluster itself (removes all node VMs)
az aks delete \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster \
  --yes \
  --no-wait

# Delete the resource group to catch anything left behind (disks, public IPs, NSGs)
az group delete \
  --name rg-aks-lab \
  --yes \
  --no-wait
```
Confirm both deletions actually completed — `az group delete` with `--no-wait` returns immediately, so check `az group show --name rg-aks-lab` a few minutes later and expect a "not found" error before considering this done.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Control plane** | The managed Kubernetes API server, scheduler, and etcd store that Azure runs and patches for you — free on the AKS Free tier, and not something you can SSH into or size yourself |
| **Node pool** | The group of VMs that actually run your workloads; this is the part you pay for and the part you size for cost in Part 1 |
| **Pod** | The smallest deployable unit in Kubernetes — one or more tightly-coupled containers sharing network and storage |
| **Deployment** | A controller that manages ReplicaSets/Pods for you — handles rollouts, self-healing, and scaling declaratively |
| **Service** | A stable network identity (ClusterIP, NodePort, or LoadBalancer) in front of a changing set of Pods, selected by label |
| **Ingress / Ingress controller** | `Ingress` is just a routing rule (host/path → Service); it does nothing until an `Ingress controller` (ingress-nginx here) is actually running to read and act on those rules |
| **Helm chart** | A packaged, templated bundle of Kubernetes manifests — `helm install` is the equivalent of `apt install` for cluster software like ingress-nginx |
| **Entra ID-integrated RBAC vs. static admin kubeconfig** | With `--enable-aad`/`--enable-azure-rbac`, access is per-user, audited, and revocable via `az role assignment`; the default admin `kubeconfig` is a single shared bearer credential with no per-user accountability |
| **Spot node pool** | Discounted, interruptible VM capacity that Azure can reclaim with ~30 seconds' notice — cheaper, but wrong for anything that needs to stay up |

---

## Common Mistakes
- **Leaving the cluster running overnight or between sessions**: node VMs and the Load Balancer bill hourly whether you're using them or not — delete the cluster the same day
- **Using the static admin kubeconfig (`az aks get-credentials --admin`) instead of Entra RBAC**: fine for a throwaway lab, but it's a single shared credential in a real environment with no per-user audit trail — don't build the habit
- **Forgetting the ingress controller has to be installed and its LB IP ready before an `Ingress` resource does anything**: an `Ingress` with no controller watching for it is just inert YAML
- **Putting stateful or long-lived workloads on a spot node pool**: evictions happen with almost no warning and Azure gives no guarantee of when capacity comes back
- **Assuming resource-group Owner/Contributor grants `kubectl` access under `--enable-azure-rbac`**: it doesn't — data-plane access is a separate role assignment scoped to the cluster resource, as Part 1 Step 3 and Part 4 both demonstrate

---

## Next Steps
This lab's RBAC pattern (Entra ID identity → Azure role assignment → scoped resource access) is the same model covered more broadly in [AZ-104 Lab 4: Identity](../AZ-104/lab-4-identity.md); for where AKS fits against other Azure hosting models (App Service, Container Apps, VMs) in an architecture decision, see [AZ-305 Lab 4: Compute & App Infrastructure Design](../AZ-305/lab-4-compute-app-infrastructure-design.md). Next in this folder: [Lab 2 — provisioning this same cluster as code with Bicep](lab-2-iac-bicep.md).

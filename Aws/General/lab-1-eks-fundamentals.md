# Lab 1: EKS Fundamentals

Check box if done: [ ]

## Overview
Kubernetes shows up in more cloud job postings than almost any other single keyword, and neither AWS certification path ([SAA-C03](../SAA-C03/README.md), [SCS-C02](../SCS-C02/README.md)) covers it beyond a surface-level mention. This lab exists to close that gap directly: stand up a real EKS cluster, deploy and expose a workload the way it's actually done in production, and prove that pod-level access to AWS APIs is governed by scoped IAM roles instead of the node's broad instance-profile permissions.

**Estimated time**: 90–110 minutes (EKS cluster creation and deletion each take 15–20 minutes via CloudFormation — budget for that, it's the biggest time sink in this lab, not the kubectl work itself)
**Cost**: ~$2–$5 — **the EKS control plane bills $0.10/hour from the moment it exists, with no free tier at all (unlike AKS's Free tier control plane).** Add a single spot-priced node (~$0.006/hr for a `t3.small`) and one NAT Gateway (~$0.045/hr plus data processing) if you let `eksctl` provision a new VPC, and a Network Load Balancer (~$0.0225/hr) once Part 3 installs the ingress controller. None of this is expensive per hour, but it adds up if the cluster sits idle — delete it the same day.

---

## Scenario
You need to stand up your first real EKS cluster and prove you can run something on it end-to-end — not just describe the architecture in an interview. That means deploying a containerized app, exposing it properly through an ingress controller instead of a `LoadBalancer` Service per app, and making sure a pod's access to AWS APIs comes from a tightly-scoped IAM role bound to its own service account — not from whatever broad permissions happen to be sitting on the EC2 node's instance profile. This lab builds all three, then proves the last piece actually works by testing it against a real AWS resource.

---

## Objectives
- Create an EKS cluster sized for cost, not production load, with IAM OIDC federation enabled
- Practice core `kubectl` operations: deployments, pods, services
- Install Helm and use it to deploy the ingress-nginx controller
- Expose an application through a Kubernetes Ingress resource
- Configure IAM Roles for Service Accounts (IRSA) and verify it actually governs — and limits — a pod's access to an AWS resource
- Tear the whole thing down before it accumulates control-plane and node-hour cost

---

## Part 1: Create a Cost-Conscious EKS Cluster

### Step 1: Confirm Tooling
```bash
aws sts get-caller-identity
eksctl version
kubectl version --client
```
`eksctl` is the de facto standard for standing up EKS — it wraps the CloudFormation stacks AWS itself recommends (VPC, IAM roles, node groups, OIDC provider) so you're not hand-assembling a dozen resources the way the raw `aws eks create-cluster` API would require.

### Step 2: Create the Cluster
One managed node group, one small spot instance, and `--with-oidc` to associate an IAM OIDC identity provider with the cluster — this is the prerequisite Part 4's IRSA setup needs, so it's on from the start rather than bolted on later.

```bash
eksctl create cluster \
  --name eks-lab-cluster \
  --region us-east-1 \
  --version 1.30 \
  --nodegroup-name spot-ng \
  --node-type t3.small \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 1 \
  --spot \
  --managed \
  --vpc-nat-mode Single \
  --with-oidc
```

- `--spot` requests discounted, interruptible capacity — fine for a disposable lab, wrong for anything that can't tolerate a ~2-minute eviction warning.
- `--vpc-nat-mode Single` caps the default VPC eksctl creates at **one** NAT Gateway instead of one per Availability Zone — the single biggest lever for keeping this lab's networking cost down, since a NAT Gateway bills hourly whether or not it's forwarding traffic.
- `--with-oidc` provisions an IAM OIDC provider trusting this cluster's token issuer — without it, no IAM role can ever trust a Kubernetes service account, and Part 4 has nothing to attach to.

This takes roughly 15–20 minutes; `eksctl` is provisioning a VPC, subnets, an IAM role for the node group, the EKS control plane itself, and the managed node group's Auto Scaling Group via CloudFormation under the hood.

### Step 3: Confirm Cluster Access
`eksctl` automatically writes your kubeconfig and — via an EKS access entry for the IAM principal that created the cluster — grants you cluster-admin. There's no separate "IAM Owner doesn't equal kubectl access" gap to close the way there is on AKS, because EKS access entries map the creating principal to `system:masters` by default.

```bash
kubectl config current-context
kubectl get nodes
```

**Validation checkpoint**: Expect one node in `Ready` status, and the context pointing at `eks-lab-cluster`. If `kubectl` hangs or errors, re-run `aws eks update-kubeconfig --name eks-lab-cluster --region us-east-1` — this is the AWS equivalent of `az aks get-credentials`.

---

## Part 2: kubectl Fundamentals — Deploy a Sample App

### Step 4: Create a Namespace
Namespaces are how Part 4's IRSA scoping works, so create one now instead of using `default`.

```bash
kubectl create namespace demo
```

### Step 5: Deploy the App
```bash
kubectl create deployment demo-app \
  --image=nginx:stable \
  --namespace=demo
```

### Step 6: Inspect What Got Created
```bash
kubectl get deployments -n demo
kubectl get pods -n demo
```
A `Deployment` manages a `ReplicaSet`, which manages the actual `Pod`(s) — this is why `kubectl get pods` shows a pod with a generated suffix rather than the deployment name itself.

### Step 7: Expose It Internally
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

`ClusterIP` is intentionally **not** internet-reachable. Part 3 handles external exposure through an ingress controller instead of switching this to `type: LoadBalancer`, which would provision a separate Elastic Load Balancer per app.

**Validation checkpoint**:
```bash
kubectl get pods -n demo        # STATUS: Running
kubectl get svc -n demo         # demo-app-svc, TYPE: ClusterIP
```

---

## Part 3: Install Helm and an Ingress Controller

### Step 8: Install the Helm CLI
```powershell
winget install Helm.Helm
helm version
```
(On Linux/macOS: `curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`)

### Step 9: Add the ingress-nginx Repo and Install It
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb
```
The annotation forces AWS's in-tree cloud-controller-manager to provision a **Network Load Balancer** instead of the older Classic Load Balancer default — an NLB is cheaper, operates at Layer 4, and is the current recommended default for any new ingress-nginx install on EKS. This is a second hourly-billed resource on top of the node — small (a couple cents/hour), but real.

### Step 10: Wait for the Load Balancer Hostname
```bash
kubectl get service ingress-nginx-controller \
  --namespace ingress-nginx \
  --watch
```
Wait until `EXTERNAL-IP` moves from `<pending>` to an actual hostname (AWS NLBs expose a DNS name, not a bare IP, unlike Azure's Standard Load Balancer), then `Ctrl+C`.

### Step 11: Create the Ingress Resource
An `Ingress` resource does nothing without a controller already watching for it — Step 9 has to happen first, or this manifest just sits there inert.

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
# note the ADDRESS column — this is the same NLB hostname from Step 10

curl http://<nlb-hostname>/
```
Expect the default NGINX welcome page HTML back. This confirms the full path works: external request → Network Load Balancer → ingress controller → Ingress rule → Service → Pod.

---

## Part 4: Verify IRSA Is Actually Governing Access

The point of IAM Roles for Service Accounts isn't just that it sounds more secure — it's that a pod's access to AWS APIs comes from a role scoped to exactly what that pod needs, auditable via CloudTrail, and completely independent of whatever broad permissions the EC2 node's own instance profile happens to carry. Prove that by testing against a real S3 bucket instead of taking it on faith.

### Step 12: Create a Test Bucket and Object
```bash
BUCKET_NAME=irsa-lab-$RANDOM
aws s3 mb s3://$BUCKET_NAME --region us-east-1
echo "irsa test object" > test-object.txt
aws s3 cp test-object.txt s3://$BUCKET_NAME/test-object.txt
```

### Step 13: Confirm a Pod Without IRSA Has No Access
Run a throwaway pod using the default service account — nothing in this lab has granted the node's own instance role any S3 permissions, so this should fail cleanly:
```bash
kubectl run aws-cli-test --rm -it --restart=Never \
  --image=amazon/aws-cli \
  -n demo \
  -- s3 cp s3://$BUCKET_NAME/test-object.txt -
```
**Expected result**: an `AccessDenied` or credential error. This is the baseline that makes Step 16 meaningful — without it, you can't tell whether IRSA is what granted access or whether the node's own permissions did.

### Step 14: Create a Read-Only IAM Policy and Role
Scope the policy to exactly this bucket, exactly `GetObject`/`ListBucket` — no wildcard resource, no write or delete actions.

```bash
cat > irsa-readonly-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::$BUCKET_NAME",
      "arn:aws:s3:::$BUCKET_NAME/*"
    ]
  }]
}
EOF

aws iam create-policy \
  --policy-name irsa-lab-readonly \
  --policy-document file://irsa-readonly-policy.json
```

Create the IAM role and Kubernetes service account together — `eksctl` wires up the trust policy (scoped to this exact OIDC provider, namespace, and service account name) automatically:

```bash
eksctl create iamserviceaccount \
  --cluster eks-lab-cluster \
  --region us-east-1 \
  --namespace demo \
  --name s3-readonly-sa \
  --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/irsa-lab-readonly \
  --approve
```
`--approve` applies the ServiceAccount and its `eks.amazonaws.com/role-arn` annotation to the cluster directly. Under the hood, the IAM role's trust policy contains a `StringEquals` condition on `<oidc-provider>:sub` equal to `system:serviceaccount:demo:s3-readonly-sa` — this is the mechanism that scopes trust to one specific service account, not the whole cluster.

### Step 15: Confirm Read Access Works
```bash
kubectl run aws-cli-readonly --rm -it --restart=Never \
  --image=amazon/aws-cli \
  -n demo \
  --overrides='{"spec":{"serviceAccountName":"s3-readonly-sa"}}' \
  -- s3 cp s3://$BUCKET_NAME/test-object.txt -
```
**Expected result**: the file contents print (`irsa test object`). The EKS Pod Identity webhook injected temporary credentials scoped to the `irsa-lab-readonly` policy into this pod automatically, based on the service account it's running as — nothing was hardcoded, and no long-lived access key exists anywhere.

### Step 16: Confirm the Same Role Cannot Delete
```bash
kubectl run aws-cli-delete-test --rm -it --restart=Never \
  --image=amazon/aws-cli \
  -n demo \
  --overrides='{"spec":{"serviceAccountName":"s3-readonly-sa"}}' \
  -- s3 rm s3://$BUCKET_NAME/test-object.txt
```
**Expected result**: `AccessDenied` — the IAM policy grants `GetObject`/`ListBucket` only, no `DeleteObject`. This is the difference between "IRSA is configured" and "IRSA actually constrains what this pod can do," and it's worth testing both directions any time a permission model gets stood up.

**Validation checkpoint**: Step 13 fails with no credentials, Step 15 succeeds, Step 16 fails with `AccessDenied` — all three outcomes match what the policy should and shouldn't allow.

---

## Cleanup
Control plane, node, NAT Gateway, and NLB all bill by the hour from the moment they're created — do this the same session, not "later today." **Skip this if you're continuing straight into [Lab 8](lab-8-container-supply-chain-security.md), which reuses this exact cluster** — tear it down after finishing that lab instead.

```bash
# Remove the ingress controller (also deprovisions the NLB)
helm uninstall ingress-nginx --namespace ingress-nginx

# Remove the IRSA service account and its IAM role
eksctl delete iamserviceaccount \
  --cluster eks-lab-cluster \
  --region us-east-1 \
  --namespace demo \
  --name s3-readonly-sa

aws iam delete-policy --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/irsa-lab-readonly

# Empty and delete the test bucket
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME

# Optional — namespaced resources get deleted with the cluster anyway,
# but tearing them down explicitly is good practice
kubectl delete namespace demo

# Delete the cluster itself (removes the node group, VPC, NAT Gateway, and control plane)
eksctl delete cluster --name eks-lab-cluster --region us-east-1
```
`eksctl delete cluster` takes 10–15 minutes and tears down the CloudFormation stacks it created in Step 2. Confirm it actually finished — `aws eks describe-cluster --name eks-lab-cluster` should return a "not found" error before considering this done; a stuck NAT Gateway or ENI left behind by a partial CloudFormation rollback is the most common reason a "deleted" cluster keeps billing.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Control plane** | The managed Kubernetes API server, scheduler, and etcd store that AWS runs and patches for you — **bills $0.10/hour with no free tier**, unlike AKS's Free-tier control plane |
| **Managed node group** | The Auto Scaling Group of EC2 instances that actually run your workloads, wrapped by EKS to handle AMI updates and lifecycle — the part you size for cost in Part 1 |
| **Pod** | The smallest deployable unit in Kubernetes — one or more tightly-coupled containers sharing network and storage |
| **Deployment** | A controller that manages ReplicaSets/Pods for you — handles rollouts, self-healing, and scaling declaratively |
| **Service** | A stable network identity (ClusterIP, NodePort, or LoadBalancer) in front of a changing set of Pods, selected by label |
| **Ingress / Ingress controller** | `Ingress` is just a routing rule (host/path → Service); it does nothing until an `Ingress controller` (ingress-nginx here) is actually running to read and act on those rules |
| **Helm chart** | A packaged, templated bundle of Kubernetes manifests — `helm install` is the equivalent of `apt install` for cluster software like ingress-nginx |
| **IAM OIDC provider** | The trust anchor that lets AWS IAM verify tokens issued by the cluster's own Kubernetes API server — the prerequisite for any IRSA role, created by `--with-oidc` in Step 2 |
| **IRSA (IAM Roles for Service Accounts)** | Binds a specific IAM role to a specific Kubernetes ServiceAccount via a scoped trust policy — a pod using that service account gets exactly that role's permissions, nothing inherited from the node |
| **Node instance profile vs. IRSA** | The node's own IAM role should carry only what `kubelet` itself needs (pulling images, writing logs); application-level AWS access belongs on IRSA-bound service accounts, never bundled onto the node role, to avoid every pod on that node implicitly getting every permission the node has |
| **Spot node group** | Discounted, interruptible EC2 capacity that AWS can reclaim with roughly 2 minutes' warning — cheaper, but wrong for anything that needs to stay up |

---

## Common Mistakes
- **Leaving the cluster running overnight or between sessions**: the control plane alone is $0.10/hr with no free tier — delete the cluster the same day, more aggressively than you might with AKS
- **Skipping `--with-oidc` at cluster creation**: IRSA has no OIDC provider to attach to without it, and adding one after the fact (`eksctl utils associate-iam-oidc-provider`) is an extra step easy to forget
- **Putting application-level AWS permissions on the node's instance role instead of IRSA**: every pod scheduled on that node inherits the node role's permissions by default — a broad node role is a much bigger blast radius than a single over-permissioned pod
- **Forgetting the ingress controller has to be installed and its load balancer hostname ready before an `Ingress` resource does anything**: an `Ingress` with no controller watching for it is just inert YAML
- **Letting `eksctl` provision one NAT Gateway per Availability Zone (the default without `--vpc-nat-mode Single`)**: fine in production for resilience, unnecessary cost for a lab that tolerates a single point of failure

---

## Next Steps
This lab's IRSA pattern (OIDC-federated identity → IAM role → scoped resource access) is the same trust-federation model covered more broadly for CI/CD in [Lab 3: CI/CD with OIDC Federated Auth](lab-3-cicd-oidc.md) — same mechanism, different token issuer. For IAM policy fundamentals underpinning the least-privilege policy in Part 4, see [SAA-C03 Lab 1](../SAA-C03/aws-lab-1-vpc-iam.md); for a deeper security-specific treatment of least-privilege instance roles and container security scanning, see [SCS-C02 Lab 6: Compute & Container Security](../SCS-C02/aws-sec-lab-6-compute-container-security.md). **Keep this cluster running if you're continuing to [Lab 8 — Container Supply-Chain Security](lab-8-container-supply-chain-security.md)**, which reuses it directly instead of standing up a second one.

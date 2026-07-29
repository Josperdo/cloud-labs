# Lab 4: Compute & Application Architecture Design

Check box if done: [ ]

## Overview
SAA-C03 teaches you to stand up an EC2 Auto Scaling Group behind an ALB. SAP-C02 asks a harder question first: *should* this workload run on EC2 at all, or does it belong on ECS, EKS, or Lambda instead — and once that's decided, how do new versions of it actually reach production without a deploy that risks the whole fleet at once. This lab runs the hosting-model decision first, then deploys the model it leads to with autoscaling and a real blue/green deployment, not a single `docker run`.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$3 (Fargate task compute for the lab's duration, an Application Load Balancer's hourly rate for under two hours — both deleted at the end)

---

## Scenario
An engineering team has containerized their order-processing API in Docker — it already runs as a container in their local dev environment and in CI. Traffic is moderate with predictable daily peaks, not spiky enough to justify Lambda's per-invocation model, and the team has no in-house Kubernetes operational experience and no near-term need for multi-cloud portability or the CNCF ecosystem EKS unlocks. Leadership wants deployments that can roll back in seconds if a bad release ships, not the multi-minute instance-replacement cycle of a traditional EC2 rolling deploy. You need to pick the hosting model and prove out the deployment strategy before this becomes the team's standard pattern for every future service.

---

## Objectives
- Build a hosting-model comparison across EC2, ECS, EKS, and Lambda and justify the recommended one for this workload
- Deploy the chosen compute model running a containerized service behind an Application Load Balancer
- Configure target-tracking autoscaling driven by real utilization, not a fixed instance count
- Execute an actual blue/green deployment and observe traffic shift from the old task set to the new one
- Explain why the "just add more instances" rolling-deploy model wasn't chosen for this requirement

---

## Part 1: Design Decision — Hosting Model Selection

### Decision: EC2 vs. ECS vs. EKS vs. Lambda

| Factor | EC2 (self-managed or ASG) | ECS (Fargate) | EKS | Lambda |
|---|---|---|---|---|
| **Operational overhead** | Highest — you patch the OS, manage the container runtime yourself, and own instance lifecycle entirely | Low — no servers to patch or size; AWS manages the underlying compute for each task | Moderate-to-high — no worker node patching with Fargate-backed EKS, but the control plane's API surface, RBAC, and add-on ecosystem (CNI, ingress controllers, cluster autoscaler) is real ongoing overhead regardless | Lowest — no containers, no cluster, no instances; you ship a function |
| **Fit for an already-containerized app** | Requires you to still manage the container runtime and orchestration yourself, or bolt on ECS/EKS anyway | Native fit — the team's existing Docker image runs here largely unchanged | Native fit, but requires the team to learn Kubernetes manifests, kubectl, and cluster operations they don't currently have | Requires repackaging as a function handler — most containerized apps don't map cleanly onto Lambda's invocation model without rework |
| **Deployment strategy options** | Rolling replacement via ASG instance refresh — minutes-scale, and rollback means another instance-replacement cycle | Native blue/green via CodeDeploy integration — traffic shifts between task sets in seconds, rollback is a traffic-shift, not a redeploy | Blue/green achievable via ingress/service mesh tooling, but it's you assembling it, not a built-in integration | Built-in traffic shifting via weighted aliases, but the unit being shifted is a function version, not a long-running service |
| **Team's existing skill fit** | Matches infrastructure skills the team already has, but doesn't match "we already containerized this" | Matches both the container skill the team has and requires no new orchestration skill | Requires net-new Kubernetes operational skill this team doesn't have, for a workload that doesn't need multi-cloud portability | Would require re-architecting a running container service into functions — a rewrite, not a redeploy |
| **Cost shape for "moderate, predictably peaking" traffic** | Pay for provisioned instances whether idle or busy, unless ASG is tuned tightly | Pay per task's vCPU/memory while running — scales with actual demand, no idle server floor | Similar Fargate-backed cost to ECS, plus the (often free-tier-eligible, but still present) EKS control plane hourly charge | Cheapest at low, spiky volume; at moderate *sustained* volume with predictable peaks, per-invocation pricing loses its advantage over provisioned-per-task pricing |

### Recommendation for This Scenario
**ECS on Fargate.** The app is already a container — ECS is the model that requires the least rework to adopt. EKS is ruled out on team-skill fit: nothing about this workload needs Kubernetes' portability or ecosystem, and taking on cluster operations the team has never run is unjustified operational risk for a single service. Lambda is ruled out on architecture fit: repackaging a running containerized API as function handlers is a rewrite, and the traffic pattern (moderate, predictable, not spiky) doesn't play to Lambda's per-invocation cost advantage anyway. EC2 is ruled out because it reintroduces exactly the operational overhead (OS patching, runtime management) the team already escaped by containerizing in the first place.

**ECS's native CodeDeploy blue/green integration is what directly satisfies the "rollback in seconds" requirement** — traffic shifts between an already-running old task set and an already-running new task set, rather than a rolling replacement that takes minutes to converge and the same amount of time again to roll back. Parts 2–4 build exactly this.

---

## Part 2: Deploy the ECS Fargate Service Behind an ALB

### Step 1: Create the Cluster and Push a Test Image
```bash
aws ecs create-cluster --cluster-name sap-c02-lab4-cluster

aws ecr create-repository --repository-name sap-c02-lab4-app

ECR_URI=$(aws ecr describe-repositories --repository-names sap-c02-lab4-app --query "repositories[0].repositoryUri" --output text)
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI

# Stand-in for the team's real API image — nginx serving a version marker, so the blue/green
# traffic shift in Part 4 is visually obvious
docker pull public.ecr.aws/nginx/nginx:latest
docker tag public.ecr.aws/nginx/nginx:latest $ECR_URI:v1
docker push $ECR_URI:v1
```

### Step 2: Register the Task Definition
```bash
cat > task-def.json <<EOF
{
  "family": "sap-c02-lab4-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<your-account-id>:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "app",
    "image": "$ECR_URI:v1",
    "portMappings": [{"containerPort": 80, "protocol": "tcp"}]
  }]
}
EOF
aws ecs register-task-definition --cli-input-json file://task-def.json
```

### Step 3: Create the ALB With Two Target Groups (Blue and Green)
CodeDeploy's ECS blue/green deployment type requires two target groups on the same ALB — one active, one idle, swapped at deployment time.

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[].SubnetId" --output text | tr '\t' ' ')

ALB_SG=$(aws ec2 create-security-group --group-name sap-c02-lab4-alb-sg --description "ALB ingress" --vpc-id $VPC_ID --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0

ALB_ARN=$(aws elbv2 create-load-balancer --name sap-c02-lab4-alb \
  --subnets $SUBNET_IDS --security-groups $ALB_SG --query "LoadBalancers[0].LoadBalancerArn" --output text)

TG_BLUE=$(aws elbv2 create-target-group --name sap-c02-lab4-tg-blue --protocol HTTP --port 80 \
  --vpc-id $VPC_ID --target-type ip --health-check-path / --query "TargetGroups[0].TargetGroupArn" --output text)
TG_GREEN=$(aws elbv2 create-target-group --name sap-c02-lab4-tg-green --protocol HTTP --port 80 \
  --vpc-id $VPC_ID --target-type ip --health-check-path / --query "TargetGroups[0].TargetGroupArn" --output text)

LISTENER_ARN=$(aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_BLUE --query "Listeners[0].ListenerArn" --output text)
```

### Step 4: Create the Service With the CODE_DEPLOY Deployment Controller
```bash
SVC_SG=$(aws ec2 create-security-group --group-name sap-c02-lab4-svc-sg --description "ECS tasks, ALB only" --vpc-id $VPC_ID --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --group-id $SVC_SG --protocol tcp --port 80 --source-group $ALB_SG

aws ecs create-service \
  --cluster sap-c02-lab4-cluster \
  --service-name sap-c02-lab4-service \
  --task-definition sap-c02-lab4-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --deployment-controller type=CODE_DEPLOY \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$SVC_SG],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=$TG_BLUE,containerName=app,containerPort=80"
```
`deployment-controller type=CODE_DEPLOY` is the setting that makes blue/green possible at all — the default `ECS` rolling controller can only replace tasks in place, one batch at a time, which is exactly the slower rollback model Part 1's decision ruled out.

**Validation checkpoint**:
```bash
aws ecs describe-services --cluster sap-c02-lab4-cluster --services sap-c02-lab4-service \
  --query "services[0].{Status:status,Running:runningCount,Desired:desiredCount}"
```
Confirm `Running` equals `Desired` (2), then hit the ALB's DNS name (`aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN --query "LoadBalancers[0].DNSName"`) and confirm you get an HTTP response.

---

## Part 3: Configure Target-Tracking Autoscaling

### Step 5: Register the Service as a Scalable Target
```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/sap-c02-lab4-cluster/sap-c02-lab4-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 6
```

### Step 6: Apply a Target-Tracking Policy on CPU Utilization
```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/sap-c02-lab4-cluster/sap-c02-lab4-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name sap-c02-lab4-cpu-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"},
    "ScaleInCooldown": 120,
    "ScaleOutCooldown": 60
  }'
```
Target tracking (not step scaling) is the right fit here — it continuously adjusts desired count to hold CPU near 60%, without hand-tuning individual scale-in/out thresholds the way step scaling requires.

**Validation checkpoint**:
```bash
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs --resource-id service/sap-c02-lab4-cluster/sap-c02-lab4-service \
  --query "ScalingPolicies[0].{Name:PolicyName,Target:TargetTrackingScalingPolicyConfiguration.TargetValue}"
```

---

## Part 4: Execute a Blue/Green Deployment

### Step 7: Set Up CodeDeploy for the ECS Service
```bash
aws deploy create-application --application-name sap-c02-lab4-app --compute-platform ECS

cat > deployment-group.json <<EOF
{
  "applicationName": "sap-c02-lab4-app",
  "deploymentGroupName": "sap-c02-lab4-dg",
  "serviceRoleArn": "arn:aws:iam::<your-account-id>:role/ecsCodeDeployRole",
  "deploymentStyle": {"deploymentType": "BLUE_GREEN", "deploymentOption": "WITH_TRAFFIC_CONTROL"},
  "blueGreenDeploymentConfiguration": {
    "terminateBlueInstancesOnDeploymentSuccess": {"action": "TERMINATE", "terminationWaitTimeInMinutes": 5},
    "deploymentReadyOption": {"actionOnTimeout": "CONTINUE_DEPLOYMENT"}
  },
  "ecsServices": [{"clusterName": "sap-c02-lab4-cluster", "serviceName": "sap-c02-lab4-service"}],
  "loadBalancerInfo": {"targetGroupPairInfoList": [{
    "targetGroups": [{"name": "sap-c02-lab4-tg-blue"}, {"name": "sap-c02-lab4-tg-green"}],
    "prodTrafficRoute": {"listenerArns": ["$LISTENER_ARN"]}
  }]}
}
EOF
aws deploy create-deployment-group --cli-input-json file://deployment-group.json
```

### Step 8: Push a New Version and Trigger the Deployment
```bash
docker tag public.ecr.aws/nginx/nginx:latest $ECR_URI:v2
docker push $ECR_URI:v2
sed 's/:v1/:v2/' task-def.json > task-def-v2.json
NEW_TASK_DEF_ARN=$(aws ecs register-task-definition --cli-input-json file://task-def-v2.json --query "taskDefinition.taskDefinitionArn" --output text)

cat > appspec.json <<EOF
{
  "version": 1,
  "Resources": [{"TargetService": {"Type": "AWS::ECS::Service", "Properties": {
    "TaskDefinition": "$NEW_TASK_DEF_ARN",
    "LoadBalancerInfo": {"ContainerName": "app", "ContainerPort": 80}
  }}}]
}
EOF

aws deploy create-deployment \
  --application-name sap-c02-lab4-app \
  --deployment-group-name sap-c02-lab4-dg \
  --revision "revisionType=AppSpecContent,appSpecContent={content='$(cat appspec.json)'}"
```

### Step 9: Watch the Traffic Shift
```bash
DEPLOYMENT_ID=$(aws deploy list-deployments --application-name sap-c02-lab4-app \
  --deployment-group-name sap-c02-lab4-dg --query "deployments[0]" --output text)

aws deploy get-deployment --deployment-id $DEPLOYMENT_ID --query "deploymentInfo.status"
```
CodeDeploy stands up the v2 task set behind the idle (green) target group, health-checks it, then shifts the ALB listener's traffic to it — the v1 (blue) task set keeps running, untouched, for the `terminationWaitTimeInMinutes` window, which is what makes rollback a traffic-shift back rather than a redeploy.

**Validation checkpoint**: poll the ALB's DNS name with `curl` repeatedly during the deployment. Requests continue succeeding throughout — there is no window where the service is unreachable, unlike a rolling deploy's brief per-batch capacity dip. Confirm final state:
```bash
aws deploy get-deployment --deployment-id $DEPLOYMENT_ID --query "deploymentInfo.status"
```
Expect `Succeeded`.

### Step 10: Prove Rollback Is Fast
If a deployment needs to roll back, it's the same mechanism in reverse — no redeploy required:
```bash
aws deploy stop-deployment --deployment-id $DEPLOYMENT_ID --auto-rollback-enabled
```
Within the wait window, this shifts the ALB listener straight back to the still-running blue target group — seconds, not the minutes an EC2 rolling-deploy rollback would take to re-provision the previous version's instances.

---

## Cleanup

```bash
# 1. CodeDeploy application (also removes the deployment group)
aws deploy delete-application --application-name sap-c02-lab4-app

# 2. Autoscaling policy and scalable target
aws application-autoscaling delete-scaling-policy --service-namespace ecs \
  --resource-id service/sap-c02-lab4-cluster/sap-c02-lab4-service \
  --scalable-dimension ecs:service:DesiredCount --policy-name sap-c02-lab4-cpu-tracking
aws application-autoscaling deregister-scalable-target --service-namespace ecs \
  --resource-id service/sap-c02-lab4-cluster/sap-c02-lab4-service \
  --scalable-dimension ecs:service:DesiredCount

# 3. ECS service (scale to 0 first, then delete)
aws ecs update-service --cluster sap-c02-lab4-cluster --service sap-c02-lab4-service --desired-count 0
aws ecs delete-service --cluster sap-c02-lab4-cluster --service sap-c02-lab4-service

# 4. ALB, listener, target groups
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 delete-target-group --target-group-arn $TG_BLUE
aws elbv2 delete-target-group --target-group-arn $TG_GREEN

# 5. Security groups, ECS cluster, ECR repo
aws ec2 delete-security-group --group-id $SVC_SG
aws ec2 delete-security-group --group-id $ALB_SG
aws ecs delete-cluster --cluster sap-c02-lab4-cluster
aws ecr delete-repository --repository-name sap-c02-lab4-app --force
```
The ALB is the only hourly-billed resource in this lab — confirm it's gone first:
```bash
aws elbv2 describe-load-balancers --query "LoadBalancers[?LoadBalancerName=='sap-c02-lab4-alb']"
```
Should return empty.

---

## Key Concepts

| Term | Definition |
|---|---|
| **Fargate** | Serverless compute for ECS (or EKS) tasks — no EC2 instances to provision, patch, or size; you specify task-level vCPU/memory instead |
| **Deployment controller (ECS)** | Determines how a service's tasks are replaced on update — `ECS` (rolling, in-place) or `CODE_DEPLOY` (blue/green, traffic-shift based) |
| **Blue/green deployment** | Two full environments (task sets, in this case) exist simultaneously; traffic shifts from old to new at the load balancer, and rollback is a traffic-shift back, not a redeploy |
| **Target-tracking scaling policy** | An autoscaling policy that continuously adjusts capacity to hold a chosen metric near a target value, rather than requiring hand-tuned step thresholds |
| **CodeDeploy AppSpec** | The deployment specification telling CodeDeploy which new task definition and container/port to route the shifted traffic to |
| **EKS control plane** | The managed Kubernetes API server AWS runs on your behalf — still bills hourly and still requires you to manage RBAC, add-ons, and manifests, even when worker nodes are Fargate-backed |

---

## Common Mistakes
- **Defaulting to EKS because "Kubernetes is the industry standard"**: standard doesn't mean correct for every team — taking on cluster operations a team has never run, for a workload with no portability requirement, is unjustified risk
- **Using the default `ECS` rolling deployment controller and expecting blue/green behavior**: blue/green requires `CODE_DEPLOY` explicitly and two target groups configured up front — it isn't a flag you flip on an existing rolling-deploy service after the fact
- **Setting `terminateBlueInstancesOnDeploymentSuccess` to terminate immediately**: a short wait window (this lab used 5 minutes) is what actually makes rollback fast — terminating the old task set the instant the new one goes live removes the rollback safety net entirely
- **Choosing Lambda for "moderate, predictably peaking" traffic on cost assumptions alone**: per-invocation pricing doesn't automatically beat provisioned-per-task pricing at sustained, predictable volume — model the actual cost, don't assume serverless is always cheaper
- **Forgetting the ALB is billed hourly regardless of task count**: scaling ECS tasks to zero in cleanup doesn't stop the ALB's charge — it has to be deleted explicitly

---

## Next Steps
This lab assumes comfort with the ALB and Auto Scaling Group basics from [SAA-C03 Lab 2: EC2, Auto Scaling & Load Balancing](../SAA-C03/aws-lab-2-ec2-autoscaling.md) and the serverless tradeoffs from [SAA-C03 Lab 6: Serverless Architecture](../SAA-C03/aws-lab-6-serverless.md) — this lab's Decision table builds directly on the Lambda-vs-EC2 reasoning that lab introduces. Continue to [Lab 5: Network Infrastructure Design](lab-5-network-infrastructure-design.md) for the hub-spoke network foundation this compute tier would sit inside at real scale.

# SAA-C03: AWS Solutions Architect Associate — Hands-On Labs

A comprehensive AWS learning path designed to make you **interview-ready**. Eight hands-on labs that take you from AWS fundamentals through a production-grade multi-cloud Terraform project, resilient architecture, serverless, cost optimization, and a fully modular Terraform-on-AWS capstone.

> Labs 1–4 were the original AWS fundamentals track. Labs 5–8 extend it with SAA-C03-specific depth: resilience, serverless, cost/performance review, and infrastructure as code.

---

## 📋 Labs Overview

| Lab | Focus | Duration | Cost | Interview Value |
|-----|-------|----------|------|-----------------|
| **Lab 1** | VPC, Security Groups, IAM | 3–4 hours | ~$0.50 | "Explain networking in AWS" |
| **Lab 2** | EC2, Auto Scaling, Load Balancing | 3–4 hours | ~$1–$2 | "Build a scalable web app" |
| **Lab 3** | S3, Lifecycle Policies, Security | 3–4 hours | ~$0.25 | "How do you optimize cloud storage?" |
| **Lab 4** | Multi-Cloud Terraform (Capstone) | 4–6 hours | ~$2–$5 | "Show me a portfolio project" |
| **Lab 5** | Resilient Multi-AZ Architecture (ALB/ASG + RDS Multi-AZ) | 3–4 hours | ~$1–$3 | "Walk me through what happens if an AZ fails" |
| **Lab 6** | Serverless Architecture (Lambda + API Gateway + DynamoDB) | 3–4 hours | ~$0 | "When would you choose serverless over EC2?" |
| **Lab 7** | Well-Architected Cost & Performance Review | 2–3 hours | ~$0 | "Tell me about a cost optimization you identified" |
| **Lab 8** | Terraform on AWS (modules + remote state) | 4–5 hours | ~$0.50–$2 | "How do you structure IaC for reuse?" |

**Total time**: ~26–34 hours
**Total cost**: ~$5–$13 (if done sequentially and cleaned up)

---

## 🎯 Why These Labs?

These four labs are strategically chosen to:

1. **Cover 80% of Cloud Engineer interview questions**
   - Networking (VPCs, security)
   - Compute (instances, scaling, load balancing)
   - Storage (S3, lifecycle, access control)
   - Infrastructure as Code (the capstone)

2. **Build confidence through hands-on work**
   - You're not just reading; you're deploying real infrastructure
   - Every lab includes validation steps so you know it works

3. **Create portfolio evidence**
   - Labs 1–3 prove you understand AWS fundamentals
   - Lab 4 (Terraform project) becomes your primary portfolio piece

4. **Teach "Why," not just "How"**
   - Each lab includes interview prep sections
   - Compare AWS to Azure so you understand cloud-agnostic patterns

---

## 📚 Lab Progression

### Week 1: Fundamentals (Labs 1–3)

**Mon–Tue**: Lab 1 (VPC + Security)
- Learn how AWS networking works
- Understand Security Groups (like Azure NSGs but instance-level)
- Create IAM users with different permissions
- **Deliverable**: Secure network architecture understanding

**Wed–Thu**: Lab 2 (EC2 + Auto Scaling)
- Deploy actual servers that do work (Nginx)
- Set up a load balancer to distribute traffic
- Understand auto-scaling (scale up/down based on demand)
- **Deliverable**: Understanding of stateless, scalable architecture

**Fri**: Lab 3 (S3 + Storage Security)
- Learn object storage concepts
- Configure lifecycle policies for cost optimization
- Practice access control (presigned URLs, bucket policies)
- **Deliverable**: Know how to architect storage systems

### Week 2: Integration (Lab 4)

**Mon–Wed**: Lab 4 (Multi-Cloud Terraform)
- Refactor your Azure Terraform project to support AWS
- Write code that targets both clouds
- Deploy infrastructure to both platforms
- **Deliverable**: GitHub portfolio project that proves you're a cloud engineer

**Thu–Fri**: Polish & apply
- Update resume with cloud infrastructure experience
- Start applying to Cloud Engineer / Cloud Security Engineer roles

---

## 🚀 How to Use These Labs

### Option A: Sequential (Recommended for First-Time AWS Users)
1. Do Lab 1 → Understand AWS networking fundamentals
2. Do Lab 2 → Build something real (load-balanced web app)
3. Do Lab 3 → Master storage and security
4. Do Lab 4 → Bring it all together in Terraform

### Option B: Focused (If You're Short on Time)
1. Skim Lab 1 (you understand networking; just learn AWS-specific syntax)
2. Do Lab 2 (this is what impresses in interviews—working servers)
3. Skim Lab 3 (you understand storage; just learn S3 API)
4. Do Lab 4 (this is your portfolio proof)

### Option C: Integration-First (If You Have Strong Cloud Background)
1. Jump straight to Lab 4
2. Use Labs 1–3 as reference material when you get stuck
3. Do focused deep-dives on anything unclear

---

## 🔐 Cost Management

Each lab is designed to cost **minimal money**:

- **Lab 1**: Free tier (VPC, security groups, IAM all free)
- **Lab 2**: t3.micro instances (free tier) and ALB (~$16/month but you delete after 4 hours)
- **Lab 3**: S3 (free tier, minimal uploads)
- **Lab 4**: t3.small instances (~30¢ × 2 × 2 hours)

**Total: ~$4–8 if you destroy resources after each lab**

### Cost-Saving Tips:
- ✅ Use `t3.micro` (free tier eligible)
- ✅ Delete resources immediately after each lab
- ✅ Use `terraform destroy` to eliminate all resources at once
- ✅ Monitor the **Billing Dashboard** if you're concerned

---

## 🛠️ Prerequisites

Before starting:

### Required
- [x] AWS account (free tier is fine)
- [x] AWS CLI installed and configured (`aws sts get-caller-identity` works)
- [x] Terraform installed (`terraform version` shows v1.0+)
- [x] SSH client (built-in on Mac/Linux; PuTTY on Windows)

### Nice to Have
- [x] Azure environment still available (for Lab 4 comparison)
- [x] Your existing Azure Terraform project (for Lab 4 refactoring)
- [x] Text editor (VS Code recommended)

### Setup Commands

```bash
# Verify AWS CLI
aws sts get-caller-identity

# Verify Terraform
terraform version

# Verify SSH (Mac/Linux)
ssh -V

# Verify you can reach AWS
aws ec2 describe-regions --query 'Regions[0].RegionName'
```

---

## 📖 How Each Lab is Structured

Every lab follows this format:

1. **Overview** — What you're building and why it matters
2. **Objectives** — Skills you'll gain
3. **Step-by-step walkthrough** — Exact AWS Console clicks (no "it's obvious")
4. **Validation checkpoints** — How you know it worked
5. **Key concepts table** — Definitions for the exam
6. **Interview prep section** — Common questions and how you'd answer them
7. **Cleanup instructions** — Delete resources and save money
8. **Next steps** — Extensions for deeper learning

---

## 🎬 Getting Started

### Step 1: Start Lab 1
```bash
# Read the overview and objectives
# Open AWS Console
# Follow the step-by-step instructions

# You're learning AWS networking fundamentals
```

### Step 2: Validate Each Step
- Don't skip validation checkpoints
- If something doesn't work, re-read that step
- Google the error message if you're stuck

### Step 3: Take Notes
- Write down what confused you
- Document "aha" moments
- These become your interview talking points

### Step 4: Clean Up
- Delete resources immediately (they cost money)
- Run `terraform destroy` for Lab 4
- Verify in AWS Console that resources are gone

### Step 5: Move to Next Lab
- Don't start Lab 2 until Lab 1 is fully cleaned up
- Each lab is independent

---

## 🔍 What to Expect in Each Lab

### Lab 1: VPC, Security Groups & IAM

**You'll do**:
- Create a VPC (isolated network)
- Create subnets (public and private)
- Create Security Groups (firewalls)
- Create IAM users with different permissions
- Understand routing and Internet Gateway

**Interview questions this prepares you for**:
- "Walk me through AWS networking"
- "What's a Security Group?"
- "How would you restrict database access?"
- "Explain IAM policies"

**Time breakdown**:
- 1 hour: Create VPC and subnets
- 1 hour: Create Security Groups with rules
- 1 hour: Create IAM users and understand policies
- Validation & cleanup: 30 min

---

### Lab 2: EC2, Auto Scaling & Load Balancing

**You'll do**:
- Create a Launch Template (blueprint for instances)
- Create an Auto Scaling Group (fleet manager)
- Attach a Load Balancer (traffic distributor)
- Deploy Nginx on instances (via user data)
- Test that traffic is distributed

**Interview questions this prepares you for**:
- "How would you deploy a scalable web app?"
- "What's an Auto Scaling Group?"
- "Explain load balancing"
- "How does your app survive instance failures?"

**Time breakdown**:
- 1 hour: Create Launch Template and ASG
- 1 hour: Create Load Balancer and test
- 1 hour: Understand scaling and health checks
- Validation & cleanup: 30 min

---

### Lab 3: S3, Lifecycle Policies & Security

**You'll do**:
- Create S3 buckets (object storage)
- Upload and version objects
- Create lifecycle policies (auto-tiering)
- Configure access control (bucket policies, IAM)
- Generate presigned URLs (temporary access)
- Enable access logging (audit trail)

**Interview questions this prepares you for**:
- "How would you architect a data lake?"
- "How do you optimize storage costs?"
- "Explain S3 lifecycle policies"
- "How would you secure sensitive data in S3?"

**Time breakdown**:
- 1 hour: Create buckets and upload objects
- 1 hour: Configure versioning and lifecycle
- 1 hour: Practice access control and presigned URLs
- Validation & cleanup: 30 min

---

### Lab 4: Multi-Cloud Terraform (Capstone)

**You'll do**:
- Refactor your Azure Terraform project
- Add AWS provider support
- Use `count` to conditionally create resources
- Deploy the same architecture to both clouds
- Document your decisions
- Create a portfolio-quality README

**Interview questions this prepares you for**:
- "Show me a portfolio project"
- "How do you think about cloud-agnostic architecture?"
- "What did you learn building multi-cloud infrastructure?"
- "Walk me through your infrastructure code"

**Time breakdown**:
- 1 hour: Understand your existing Azure code
- 2 hours: Write multi-cloud Terraform
- 1 hour: Test Azure and AWS deployments
- 1 hour: Document and Polish
- Validation & cleanup: 30 min

---

### Lab 5: Resilient Multi-AZ Architecture

**You'll do**:
- Deploy an ALB + ASG spanning two Availability Zones
- Deploy an RDS Multi-AZ database
- Force an AZ failure and an RDS failover, and measure the actual downtime

**Interview questions this prepares you for**:
- "Walk me through what happens if an entire AZ goes down"
- "What's the difference between an RDS read replica and Multi-AZ?"
- "How would you design for high availability?"

See: [aws-lab-5-resilient-architecture.md](aws-lab-5-resilient-architecture.md)

---

### Lab 6: Serverless Architecture

**You'll do**:
- Build a REST API with Lambda, API Gateway, and DynamoDB
- Scope a Lambda execution role to least privilege
- Observe cold starts and reason about when serverless is (and isn't) the right call

**Interview questions this prepares you for**:
- "When would you choose Lambda over EC2?"
- "How do you scope IAM permissions for a Lambda function?"
- "What's a cold start and when does it matter?"

See: [aws-lab-6-serverless.md](aws-lab-6-serverless.md)

---

### Lab 7: Well-Architected Cost & Performance Review

**You'll do**:
- Audit real spend from Labs 1–6 with Cost Explorer
- Revisit S3 storage classes and lifecycle rules with real pricing numbers
- Right-size compute using CloudWatch metrics
- Run a lightweight Well-Architected Framework review

**Interview questions this prepares you for**:
- "Tell me about a cost optimization you identified"
- "How do you decide which S3 storage class to use?"
- "Walk me through the Well-Architected Framework"

See: [aws-lab-7-well-architected-cost.md](aws-lab-7-well-architected-cost.md)

---

### Lab 8: Terraform on AWS

**You'll do**:
- Set up S3 + DynamoDB remote state with locking
- Structure networking and compute into reusable Terraform modules
- Force and recover from a state lock conflict
- Compare the AWS and Azure Terraform provider patterns directly

**Interview questions this prepares you for**:
- "How do you structure Terraform for reuse across projects?"
- "Why use remote state instead of local state?"
- "What AWS/Azure Terraform differences have you actually run into?"

See: [aws-lab-8-terraform-aws.md](aws-lab-8-terraform-aws.md)

---

## 🎯 Interview Prep Strategy

### After Lab 1:
- Be able to explain: "A VPC is an isolated network. You define the IP space. Subnets are smaller segments. Security Groups are firewalls."

### After Lab 2:
- Be able to explain: "I deployed a load-balanced web app that auto-scales based on CPU. If one instance fails, the load balancer stops sending traffic to it and the ASG launches a replacement."

### After Lab 3:
- Be able to explain: "I configured S3 with versioning and lifecycle policies to automatically move old data to cheaper tiers. I used bucket policies and IAM to restrict access to authorized users only."

### After Lab 4:
- Be able to explain: "I built a Terraform project that deploys infrastructure to both Azure and AWS. I used conditional logic to target different clouds with the same code, demonstrating multi-cloud thinking."

### After Lab 5:
- Be able to explain: "I built a Multi-AZ architecture and actually forced a failover to measure recovery time, instead of just reading about it — RDS failover took about a minute, and the ALB kept serving traffic from the healthy AZ the whole time."

### After Lab 6:
- Be able to explain: "I built a serverless API with Lambda, API Gateway, and DynamoDB, scoped to a least-privilege execution role, and can explain concretely when I'd choose that over an always-on EC2 architecture."

### After Lab 7:
- Be able to explain: "I ran a real cost review against my own project using Cost Explorer, found what was actually driving spend, and fixed it — not a hypothetical, a project I can point to."

### After Lab 8:
- Be able to explain: "I structured my Terraform into reusable networking and compute modules with remote state in S3 and locking via DynamoDB, and I've actually diagnosed and recovered from a state lock conflict."

---

## ❓ Common Questions & Troubleshooting

### "Why are these labs taking longer than expected?"
- Don't rush. Understanding *why* is more important than speed.
- If you're stuck, re-read the step and the validation checkpoint.
- Google the error message.

### "The AWS Console looks different than the screenshots"
- AWS updates the UI. The core functionality stays the same.
- Find the same feature by searching (click the search bar at the top).

### "I don't have a credit card / free tier available"
- You can still read the labs and understand the concepts
- When you're ready to apply for jobs, they'll provide an environment

### "I'm getting permission errors"
- Make sure your AWS credentials are configured: `aws sts get-caller-identity`
- Make sure your user has EC2, VPC, S3, IAM permissions

### "I forgot to clean up resources and got a bill"
- Go to **EC2 Dashboard** and **Terminate instances**
- Go to **S3** and **Delete buckets**
- Check **CloudTrail** to see what's running
- Contact AWS Support if the bill is large

---

## 📋 Checklist: Am I Ready to Apply?

After all 8 labs:

- [ ] I can explain VPC networking without reading notes
- [ ] I can draw a diagram of public/private subnet architecture
- [ ] I understand Security Groups and IAM policies
- [ ] I deployed a working web application (Nginx)
- [ ] I configured auto-scaling
- [ ] I set up a load balancer
- [ ] I understand S3 tiers and lifecycle policies
- [ ] I have a multi-cloud Terraform project on GitHub
- [ ] I've forced a Multi-AZ failover and can explain the recovery timeline
- [ ] I built a serverless API and can explain when Lambda beats EC2
- [ ] I've run a real cost review and found (and fixed) an actual issue
- [ ] I structured a Terraform project into reusable modules with remote state
- [ ] My project has a solid README explaining my architecture
- [ ] I can talk about my projects in technical interviews

**If you check all boxes, you're ready to apply to Cloud Engineer roles.**

---

## 🚀 Next Steps (After These Labs)

These labs are the foundation. To go deeper:

### For Cloud Security Engineers:
- Add GuardDuty (threat detection) to Lab 2
- Enable CloudTrail logging to S3
- Implement encryption with AWS KMS
- Create Security Hub rules

### For DevOps Engineers:
- Add Docker to your instances (containerize Nginx)
- Set up AWS CodePipeline (CI/CD)
- Use AWS CodeDeploy for automated deployments
- Implement auto-scaling based on custom metrics

### For Platform Engineers:
- Deploy Kubernetes (EKS) instead of EC2
- Set up service mesh (Istio)
- Implement multi-cluster architecture
- Build a platform for your team

### For Everyone:
- Deploy a real application (Node.js, Python, Go)
- Set up monitoring with CloudWatch
- Implement disaster recovery (backups, RDS)
- Practice for AWS certifications (if interested)

---

## 📚 Additional Resources

### AWS Official
- [AWS Skill Builder](https://skillbuilder.aws.com/) (free tier has labs)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Security Best Practices](https://aws.amazon.com/security/security-best-practices/)

### Terraform
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)

### Learning Communities
- [AWS Tech Community](https://techcommunity.aws/)
- [r/aws on Reddit](https://reddit.com/r/aws)
- [Stack Overflow - AWS tag](https://stackoverflow.com/questions/tagged/amazon-web-services)

---

## 📝 How to Customize These Labs

These labs are templates. Feel free to:
- ✅ Add extra resources (e.g., RDS database, ElastiCache)
- ✅ Use different instance types or regions
- ✅ Add monitoring/logging
- ✅ Deploy a real application
- ✅ Extend the Terraform project with modules

The goal is to *understand*, not memorize.

---

## 🎓 Final Thought

Cloud infrastructure seems intimidating at first. But after doing these labs, you'll realize:
- Networking is just "defining who can talk to whom"
- Compute is just "running code on someone else's server"
- Storage is just "paying for data space"
- Auto-scaling is just "launch more servers when busy, fewer when quiet"

You've got this. Start with Lab 1 and move forward. **Two weeks from now, you'll be a Cloud Engineer.**

---

**Ready? → Start with `aws-lab-1-vpc-iam.md`**

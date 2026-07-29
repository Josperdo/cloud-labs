# AWS General — Certification-Agnostic Applied Labs

Eight labs covering high-demand AWS skills that show up constantly in job postings and interviews but aren't fully owned by any single certification: Kubernetes (EKS), infrastructure-as-code beyond Terraform, CI/CD with keyless cloud authentication, secrets/config management at scale, application observability, cost governance (FinOps), enterprise landing-zone patterns at the AWS Organizations level, and container supply-chain security.

This track exists for a specific reason: certifications prove you know the exam's slice of AWS, but hiring managers and ATS keyword scans look for EKS, CDK, CI/CD, observability, and FinOps by name. These labs are the answer to "what have you actually built with this?" for the keywords that [SAA-C03](../SAA-C03/README.md) and [SCS-C02](../SCS-C02/README.md) don't cover in depth — built to be well-rounded, resume-searchable, and defensible in an interview, not to prep for a specific test.

---

## Labs Overview

| Lab | Focus Area | Duration | Cost | Key Skills |
|-----|-----------|----------|------|-------------|
| **Lab 1** | EKS Fundamentals | 90–110 min | ~$2–$5 | EKS cluster (spot managed node group), kubectl, Helm, ingress-nginx, IAM Roles for Service Accounts (IRSA) |
| **Lab 2** | Infrastructure as Code with AWS CDK | 60–75 min | ~$0–$1 | CDK (TypeScript), stacks, cross-stack references, synth/diff/deploy/destroy, CDK vs. Terraform trade-offs |
| **Lab 3** | CI/CD with OIDC Federated Auth | 60–75 min | ~$0 | GitHub Actions, IAM OIDC identity provider, repo/branch-scoped trust policy, plan-on-PR/apply-on-merge |
| **Lab 4** | Secrets & Config Management at Scale | 60–75 min | ~$0–$1 | Secrets Manager, automated Lambda rotation, SSM Parameter Store, IAM-scoped least privilege |
| **Lab 5** | Observability & Application Performance Monitoring | 60–75 min | ~$0–$1 | CloudWatch metrics, Logs Insights, X-Ray distributed tracing, dashboards, composite alarms |
| **Lab 6** | Cost Management & FinOps | 45–60 min | ~$0 | Cost Explorer, AWS Budgets, cost-allocation tagging, Compute Optimizer rightsizing |
| **Lab 7** | Landing Zone & Governance at Scale | 60–75 min | ~$0 | AWS Organizations OU hierarchy, Service Control Policies, Tag Policies, policy-as-code |
| **Lab 8** | Container Supply-Chain Security | 90–110 min | ~$1–$4 | ECR enhanced scanning (Inspector), SBOM generation, cosign/Sigstore image signing with KMS, EKS admission control via Kyverno |

**Total time**: ~9–11 hours
**Total cost**: ~$3–$14 if done sequentially and cleaned up after each lab (Lab 1's EKS cluster is the only resource spanning multiple labs — the control plane bills hourly with no free tier, so delete it the same session you finish Lab 8, since Lab 8 reuses it)

---

## Prerequisites

- AWS account with billing alerts configured (or free tier, though EKS itself is never free — see Cost Management below)
- Completion of [SAA-C03 Labs 1–3](../SAA-C03/README.md) or equivalent comfort with VPCs, IAM, and S3
- **AWS CLI** authenticated (`aws sts get-caller-identity` works) and a default region set
- **kubectl**, **Helm**, and **eksctl** installed (Lab 1)
- **Node.js 18+** and the **AWS CDK CLI** (`npm install -g aws-cdk`) for Lab 2
- **Docker** installed for Lab 8 (building and pushing images to ECR)
- A **GitHub account** and the **GitHub CLI** (`gh`) for Lab 3's OIDC pipeline
- Familiarity with the Terraform basics from [SAA-C03 Lab 8](../SAA-C03/aws-lab-8-terraform-aws.md) helps for Lab 2's comparison and Lab 7's policy-as-code section, but isn't required

---

## How to Use These Labs

Labs are independent of each other in concept but loosely build infrastructure you can reuse — the IAM OIDC provider and role from Lab 3 are convenient (not required) to reuse for any future pipeline in this repo.

- **Targeting DevOps/platform or security engineer roles?** Labs 1, 3, and 4 are the strongest resume/interview trio — containers, keyless CI/CD, and secrets management are three of the most-screened-for keywords in current job postings.
- **Want a FinOps or cost-optimization talking point?** Lab 6 is short and self-contained — good for rounding out an interview answer about "how do you keep cloud spend under control," and it covers ground [SAA-C03 Lab 7](../SAA-C03/aws-lab-7-well-architected-cost.md) doesn't (Budgets API automation, tag-activation propagation, Compute Optimizer).
- **Want a supply-chain security talking point?** Lab 8 closes the loop on Lab 1's EKS cluster — SBOM generation and Sigstore/cosign image signing are increasingly asked about directly, post-SolarWinds/xz-utils, in security engineer interviews.
- **Prepping for a platform/architecture-adjacent interview?** Lab 7's landing-zone lab pairs well with [SCS-C02 Lab 7](../SCS-C02/aws-sec-lab-7-multi-account-governance.md) — one is governance breadth (OU design, Tag Policies, policy-as-code), the other is security-guardrail depth (SCPs as an incident-response control, delegated administration, cross-account Athena forensics).

---

## Lab Format

Same format as the other tracks: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Cost Management
- **EKS (Lab 1) is the one lab where the control plane itself bills hourly — $0.10/hr with no free tier, unlike AKS's Free tier control plane.** Combined with a spot node and a single NAT Gateway, expect roughly $0.15–$0.20/hr while the cluster exists. Delete it (`eksctl delete cluster`) the same session, and don't leave it running between Lab 1 and Lab 8.
- Cost Explorer's API (used in Lab 6) bills $0.01 per request — trivial at lab scale, but a real difference from Azure's free Cost Management API worth knowing for an interview.
- AWS Secrets Manager (Lab 4) has no free tier — each secret bills $0.40/month, prorated, and continues billing during the standard 30-day pending-deletion window unless you force-delete without recovery.
- Everything else in this track is either AWS free-tier services (SSM Parameter Store standard tier, CloudWatch's first 10 alarms, Budgets, Organizations SCPs/Tag Policies) or scoped to stay under $1.
- Delete every resource in a lab's Cleanup section the same session — several of these services (Secrets Manager, KMS keys, CloudWatch dashboards beyond the first 3) keep billing in the background if left behind.

### Why "Certification-Agnostic"
Neither SAA-C03 nor SCS-C02 tests Kubernetes operations, AWS CDK, GitHub Actions OIDC, or FinOps tooling in real depth — they're each a small footnote at best. This track exists to close that gap deliberately, so the portfolio demonstrates the parts of the job that show up in day-to-day work and job descriptions, not just what's on an exam blueprint.

---

## Additional Resources

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [AWS CDK Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
- [GitHub Actions OIDC with AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [AWS Organizations User Guide](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html)

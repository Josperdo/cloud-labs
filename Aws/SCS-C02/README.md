# SCS-C02: AWS Certified Security — Specialty — Hands-On Labs

Eight labs covering the full breadth of core SCS-C02 domains: IAM policy mechanics, detection and monitoring, data protection, automated incident response, network and perimeter security, compute/container security, multi-account governance, and a capstone that ties all seven prior labs into one end-to-end incident response exercise. Where [SAA-C03](../SAA-C03/README.md) builds and secures infrastructure as a byproduct of good architecture, this track puts security itself in the driver's seat — the questions here are "how do you detect this," "how do you prove this is encrypted," "what can reach this at the network layer," and "how do you respond automatically before a human even looks at it."

These labs assume comfort with the AWS fundamentals from [SAA-C03 Labs 1–3](../SAA-C03/README.md) (VPC, IAM basics, S3). They go significantly deeper on IAM policy evaluation logic, detection tooling, network/perimeter controls, and automated remediation than the SAA track does.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-------------|
| **Lab 1** | IAM Deep Dive | 60–90 min | ~$0 | Identity vs. resource policies, permission boundaries, policy simulator, cross-account roles |
| **Lab 2** | Detection & Monitoring | 60–90 min | ~$0–$1 | CloudTrail, GuardDuty, Security Hub, AWS Config rules |
| **Lab 3** | Data Protection | 60–75 min | ~$0–$1 | KMS customer-managed keys, S3 Block Public Access, Secrets Manager |
| **Lab 4** | Incident Response Automation | 75–90 min | ~$0–$1 | EventBridge + Lambda auto-remediation, deployed via Terraform |
| **Lab 5** | Network & Perimeter Security | 90–110 min | ~$2–$6 | Security groups vs. NACLs, VPC endpoints, Network Firewall, WAF/Shield, Flow Logs + Athena |
| **Lab 6** | Compute & Container Security | 75–90 min | ~$0–$2 | Session Manager, Amazon Inspector, least-privilege instance roles, ECR scan-on-push, GuardDuty EKS Protection |
| **Lab 7** | Multi-Account Security & Governance | 75–90 min | ~$0–$1 | AWS Organizations SCPs, delegated administration, cross-account Athena queries |
| **Lab 8** | Incident Response Capstone | 90–120 min | ~$1–$3 | EC2 quarantine, EBS snapshot forensics, automated credential revocation, full detect-to-remediate walkthrough |

**Total time**: ~9.75–12.5 hours
**Total cost**: ~$3–$15 if done sequentially and cleaned up after each lab (Lab 5's Network Firewall is the single largest cost driver in the track — see Cost Management below)

---

## Prerequisites

- AWS account with IAM, CloudTrail, GuardDuty, Security Hub, Config, KMS, Lambda, VPC, Network Firewall, WAF, Inspector, and ECR permissions
- **AWS CLI** authenticated (`aws sts get-caller-identity` works)
- **Terraform CLI** (Lab 4)
- **Docker** (Lab 6, for pushing a test image to ECR)
- **AWS Organizations access** (Lab 7, optional) — a real multi-account Organization is needed to fully exercise cross-account SCP enforcement and delegated administration; Lab 7 provides an explicit single-account conceptual fallback if you don't have one
- Completion of, or comfort with, [SAA-C03 Labs 1–3](../SAA-C03/README.md)

---

## How to Use These Labs

- **Weak on IAM policy evaluation?** Start with Lab 1 — it's the single highest-yield SCS-C02 domain and the one most candidates underestimate.
- **Coming from a SOC/detection background?** Lab 2 will feel familiar; the AWS-specific value is in how GuardDuty, Security Hub, and Config layer together rather than duplicate each other.
- **Want the strongest interview story?** Lab 4's EventBridge + Lambda auto-remediation, deployed via Terraform, is the one to lead with — "I built infrastructure that detects and fixes a misconfiguration automatically, without waiting for a human." Lab 8 extends the exact same pattern to a live credential-compromise scenario.
- **Coming from a network engineering background?** Lab 5 is the fastest on-ramp — security groups, NACLs, and Network Firewall map closely to concepts you already know, layered with AWS-specific detail (stateful vs. stateless enforcement, VPC endpoint routing).
- **Want the capstone story for an interview?** Lab 8 is built to be narrated end to end — "walk me through how you'd handle a compromised instance" — and it explicitly draws on every prior lab in the track, not just its own material.

---

## Lab Format

Same format as the other tracks: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Cost Management
- GuardDuty has a 30-day free trial per account — if you've used it before on this account, it may bill immediately; check **GuardDuty** → **Usage** before enabling
- Security Hub and Config both bill per check/evaluation — small for a single lab, but disable them in cleanup, don't just delete the resources they were watching
- **AWS Network Firewall (Lab 5) bills hourly (~$0.395/hr) per endpoint plus per-GB data processing, regardless of traffic volume** — this is the most expensive single resource in the entire track; deploy it, validate it, and tear it down in the same session
- Delete CloudTrail trails and disable Config/GuardDuty/Security Hub/Inspector explicitly at the end of each lab — these are account-level services that keep running (and billing) independently of any specific resource group
- Lab 7's delegated administration and organization trail features are free to configure but assume an AWS Organization already exists — don't create a throwaway Organization you don't intend to keep, since leaving it takes deliberate cleanup of its own

### Exam Alignment
These eight labs map to all five SCS-C02 exam domains:

| Exam Domain | Labs |
|---|---|
| Threat Detection & Incident Response | Lab 2, Lab 4, Lab 8 |
| Security Logging & Monitoring | Lab 2, Lab 5 (Flow Logs), Lab 7 (Athena) |
| Infrastructure Security | Lab 5, Lab 6, Lab 7 |
| Identity & Access Management | Lab 1, Lab 6 (instance roles), Lab 7 (SCPs) |
| Data Protection | Lab 3 |

The original four-lab version of this track had no dedicated coverage of Infrastructure Security — the domain most exam-takers report as the hardest to prepare for outside a real job. Labs 5, 6, and 7 close that gap directly. As with the other tracks, these labs are built for defensible hands-on depth on realistic scenarios, not exhaustive coverage of every exam objective.

---

## Additional Resources

- [AWS Certified Security – Specialty](https://aws.amazon.com/certification/certified-security-specialty/)
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html)
- [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)
- [AWS Network Firewall Developer Guide](https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html)

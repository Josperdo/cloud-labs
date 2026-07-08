# SCS-C02: AWS Certified Security — Specialty — Hands-On Labs

Four labs covering the core SCS-C02 domains: IAM policy mechanics, detection and monitoring, data protection, and automated incident response. Where [SAA-C03](../SAA-C03/README.md) builds and secures infrastructure as a byproduct of good architecture, this track puts security itself in the driver's seat — the questions here are "how do you detect this," "how do you prove this is encrypted," and "how do you respond automatically before a human even looks at it."

These labs assume comfort with the AWS fundamentals from [SAA-C03 Labs 1–3](../SAA-C03/README.md) (VPC, IAM basics, S3). They go significantly deeper on IAM policy evaluation logic, detection tooling, and automated remediation than the SAA track does.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-------------|
| **Lab 1** | IAM Deep Dive | 60–90 min | ~$0 | Identity vs. resource policies, permission boundaries, policy simulator, cross-account roles |
| **Lab 2** | Detection & Monitoring | 60–90 min | ~$0–$1 | CloudTrail, GuardDuty, Security Hub, AWS Config rules |
| **Lab 3** | Data Protection | 60–75 min | ~$0–$1 | KMS customer-managed keys, S3 Block Public Access, Secrets Manager |
| **Lab 4** | Incident Response Automation | 75–90 min | ~$0–$1 | EventBridge + Lambda auto-remediation, deployed via Terraform |

**Total time**: ~4.5–5.5 hours
**Total cost**: ~$1–$3 if done sequentially and cleaned up after each lab

---

## Prerequisites

- AWS account with IAM, CloudTrail, GuardDuty, Security Hub, Config, KMS, and Lambda permissions
- **AWS CLI** authenticated (`aws sts get-caller-identity` works)
- **Terraform CLI** (Lab 4)
- Completion of, or comfort with, [SAA-C03 Labs 1–3](../SAA-C03/README.md)

---

## How to Use These Labs

- **Weak on IAM policy evaluation?** Start with Lab 1 — it's the single highest-yield SCS-C02 domain and the one most candidates underestimate.
- **Coming from a SOC/detection background?** Lab 2 will feel familiar; the AWS-specific value is in how GuardDuty, Security Hub, and Config layer together rather than duplicate each other.
- **Want the strongest interview story?** Lab 4's EventBridge + Lambda auto-remediation, deployed via Terraform, is the one to lead with — "I built infrastructure that detects and fixes a misconfiguration automatically, without waiting for a human."

---

## Lab Format

Same format as the other tracks: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Cost Management
- GuardDuty has a 30-day free trial per account — if you've used it before on this account, it may bill immediately; check **GuardDuty** → **Usage** before enabling
- Security Hub and Config both bill per check/evaluation — small for a single lab, but disable them in cleanup, don't just delete the resources they were watching
- Delete CloudTrail trails and disable Config/GuardDuty/Security Hub explicitly at the end of each lab — these are account-level services that keep running (and billing) independently of any specific resource group

### Exam Alignment
These four labs map to the highest-yield SCS-C02 domains: identity and access management, threat detection and incident response, infrastructure security, and data protection. As with the other tracks, they're built for defensible hands-on depth on realistic scenarios, not exhaustive coverage of every exam objective.

---

## Additional Resources

- [AWS Certified Security – Specialty](https://aws.amazon.com/certification/certified-security-specialty/)
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html)
- [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)

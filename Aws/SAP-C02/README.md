# SAP-C02: AWS Certified Solutions Architect – Professional — Hands-On Labs

Eight labs covering the four SAP-C02 domains: organizational complexity, new solution design, continuous improvement of existing solutions, and migration/modernization. SAP-C02 is a *design* exam — the questions aren't "how do you create a Transit Gateway attachment," they're "given these requirements, which of four legitimate options is correct, and why do the other three fail." Every lab here pairs a requirements-driven decision (a comparison table built before any CLI command runs) with a hands-on deployment of the option that decision leads to, closing with a capstone (Lab 8) that pulls every prior domain into one governed landing zone and a Well-Architected Framework review pass.

These labs assume the [SAA-C03](../SAA-C03/README.md) foundation — VPCs, EC2, RDS, IAM basics, S3 — is already solid; SAP-C02 doesn't reteach any of that, it's about choosing *between* the services SAA-C03 already taught you how to deploy. Where a design decision overlaps security depth that [SCS-C02](../SCS-C02/README.md) already covers in more detail (multi-account SCPs and delegated administration, KMS key policy design, network perimeter controls), this track cross-links to that lab instead of re-deriving it.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-------------|
| **Lab 1** | Multi-Account Organizational Design | 75–90 min | ~$0 | AWS Organizations OU hierarchy, Service Control Policies, AWS RAM cross-account sharing, organization-wide CloudTrail |
| **Lab 2** | Data & Storage Database Design | 75–90 min | ~$1–$3 | S3/EBS/EFS/FSx decision matrix, RDS/Aurora/DynamoDB decision matrix, customer-managed KMS key encryption |
| **Lab 3** | Business Continuity Design | 75–100 min | ~$2–$5 | RTO/RPO-driven DR strategy selection, AWS Backup cross-region copy, forced failover with measured recovery time |
| **Lab 4** | Compute & Application Architecture Design | 75–90 min | ~$1–$3 | EC2/ECS/EKS/Lambda decision matrix, ECS Fargate autoscaling, CodeDeploy blue/green deployment |
| **Lab 5** | Network Infrastructure Design | 90–120 min | ~$3–$8 | VPC peering vs. Transit Gateway, Direct Connect vs. VPN, Transit Gateway hub-spoke, PrivateLink, WAF/Shield/CloudFront placement |
| **Lab 6** | Multi-Region & Global Distribution Design | 90–110 min | ~$1–$4 | Route 53 routing policy comparison, DynamoDB Global Tables vs. Aurora Global Database, CloudFront origin failover, timed failover test |
| **Lab 7** | Migration & Modernization Design | 75–90 min | ~$1–$4 | The 6 R's decision matrix, DMS full-load-and-CDC migration, MGN workflow, strangler-fig pattern |
| **Lab 8** | Capstone — Well-Architected Landing Zone | 100–120 min | ~$2–$5 | Synthesis of Labs 1–7, scaled-down governed landing zone, Well-Architected Framework review pass |

**Total time**: ~11–13.8 hours
**Total cost**: ~$11–$32 if done sequentially and cleaned up after each lab (Lab 5's Transit Gateway is the single biggest line item by far — deploy, do the lab, tear down the same session; nothing else in the track approaches its hourly-plus-per-GB cost)

---

## Prerequisites

- AWS account with Organizations, RAM, Backup, DMS, MGN, WAF, Transit Gateway, Route 53, and CloudFront permissions
- Completion of [SAA-C03](../SAA-C03/README.md), or equivalent comfort with VPCs, EC2, RDS, IAM, and S3 — this track builds design judgment on top of that hands-on knowledge rather than re-teaching it
- **AWS CLI** authenticated (`aws sts get-caller-identity` works)
- A second AWS region available (Labs 3, 5, and 6 all require a secondary region for DR, hub-spoke design, and multi-region routing)
- A registered Route 53 hosted zone for a domain you control, or comfort adapting Lab 6's DNS steps to a test zone
- Organizations/RAM setup note: enabling organization-wide RAM sharing and creating an AWS Organization are one-time, largely console-first settings in a real deployment — every operational step this track repeats (OU creation, SCP attachment, resource shares) is fully CLI-drivable and is what's actually scripted below

---

## How to Use These Labs

Work them in domain order — each lab's decision matrix references concepts (account structure, storage service selection, hosting models) that later labs assume you've already reasoned through once.

- **Coming straight from SAA-C03?** Start with Lab 1 — organizational design is the connective tissue between "I can configure this service" (SAA-C03) and "I know which account it should live in and who governs it" (SAP-C02).
- **Weak on the exam's design-judgment style?** Every lab's Design Decision step is the one to slow down on — the goal isn't the deployment, it's being able to defend the choice out loud against the options that were ruled out.
- **Want the strongest interview story?** Lab 5's Transit Gateway hub-spoke build is the one senior interviewers probe hardest for network depth — but Lab 8's capstone is the one to walk through if asked "design a landing zone" as an open-ended prompt, since it's the only lab that shows all seven prior decisions reasoned through for a single company at once.
- **Short on time?** Labs 1, 2, and 7 run at or near the low end of the cost range and cover Domain 1 and Domain 4 outright; Labs 4–6 are the ones to prioritize if the goal is rounding out Design for New Solutions specifically, since that's the single highest-weighted domain on the exam.

---

## Lab Format

Same format as the other tracks — Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps — with one addition specific to this track: each lab opens its hands-on section with a **Design Decision** step, a comparison table of the viable options and why one wins for the stated requirements, before any CLI work begins.

---

## Important Notes

### Cost Management
- **Transit Gateway (Lab 5) bills hourly per attachment plus per-GB data processed, regardless of traffic volume** — this is the single most expensive resource in the entire track; deploy it, complete the lab, and delete every attachment the same session
- AWS Backup (Lab 3) and its cross-region copy incur ongoing storage costs independent of the source resource — deleting the database doesn't stop backup storage billing; recovery points must be deleted explicitly
- The Application Load Balancer (Lab 4) has a real hourly rate — delete it at the end of the lab rather than leaving it "for later," the way every other track in this repo treats its own hourly-billed meters
- CloudFront (Lab 6) and DMS replication instances (Lab 7) bill in small but real increments for as long as they exist — tear both down at the end of their respective labs
- Delete or disable every account-level service (organization trails, budgets) explicitly at the end of each lab — these keep running independently of any single resource group being deleted

### Exam Alignment
These eight labs map to all four SAP-C02 exam domains:

| Exam Domain | Weight | Labs |
|---|---|---|
| Domain 1: Design Solutions for Organizational Complexity | ~26% | Lab 1, Lab 8 |
| Domain 2: Design for New Solutions | ~29% | Lab 2, Lab 3, Lab 4, Lab 5, Lab 6 |
| Domain 3: Continuous Improvement for Existing Solutions | ~25% | Lab 3, Lab 6, Lab 8 |
| Domain 4: Accelerate Workload Migration and Modernization | ~20% | Lab 7 |

Domain 2 — 29% of the exam by itself — is covered across five labs instead of one, reflecting its weight: storage/database selection (Lab 2), business continuity as new-solution design (Lab 3), compute hosting models (Lab 4), network topology (Lab 5), and multi-region distribution (Lab 6). Labs 3 and 6 also reinforce Domain 3's continuous-improvement objectives — a DR strategy and a multi-region routing design are both things an *existing* solution gets retrofitted with, not just greenfield decisions — which is why they appear against both domains. Lab 8's capstone doesn't add new exam-objective breadth on its own; it's where Labs 1–7's separate domain coverage gets exercised together, which is the format SAP-C02's longer scenario-based questions actually use. These labs are built for design-judgment depth on the highest-yield scenarios, not exhaustive coverage of every exam objective — SAP-C02 rewards being able to explain trade-offs, and that's what each lab's Design Decision step is for.

### Relationship to Other Tracks
SAP-C02 is deliberately the professional-level counterpart to [SAA-C03](../SAA-C03/README.md) in this repo, the same way [AZ-305](../../Azure/AZ-305/README.md) sits above AZ-104 on the Azure side: SAA-C03 builds and secures infrastructure, SAP-C02 decides *between* the options SAA-C03 already taught you how to build. Lab 1's SCPs and Lab 5's WAF placement overlap directly with [SCS-C02](../SCS-C02/README.md)'s multi-account governance (Lab 7) and network/perimeter security (Lab 5) content — this track doesn't re-teach that depth, it cross-links to it inline wherever the overlap happens. Lab 8 is this track's own internal capstone — it assumes Labs 1 through 7 are already done and doesn't introduce a new domain, only synthesizes the seven that came before it into one landing zone design, closing with a Well-Architected Framework review pass against what was actually built.

---

## Additional Resources

- [AWS Certified Solutions Architect – Professional](https://aws.amazon.com/certification/certified-solutions-architect-professional/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Prescriptive Guidance: Migration](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/welcome.html)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)

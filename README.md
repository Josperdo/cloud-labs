# Cloud Labs

Hands-on, certification-aligned labs across Azure and AWS — portal work, CLI, and Infrastructure as Code — built for reps and interview-readiness, not just theory. Every lab deploys real (cheap, disposable) infrastructure, includes validation checkpoints, and ends with cleanup so nothing lingers on a bill.

**88 labs across 11 certification tracks.** Pick the cert you're studying for, work the labs in order, and walk away with something you actually built — not just a score report.

---

## Who this is for

Anyone prepping for an Azure or AWS certification who wants to *do* the exam objectives, not just read about them — self-taught learners, career-changers, and working engineers filling gaps before a test. No prior lab-writing or IaC experience assumed; each track's README lists exactly what you need before you start.

This is one person's study log turned into a portfolio, shared in case it's useful to someone else on the same path. It's not official Microsoft or AWS training material, and isn't affiliated with either company — see [Disclaimer](#disclaimer) below.

---

## Tracks

### Azure

| Track | Certification | Focus | Labs |
|-------|---------------|-------|------|
| [Azure/AZ-104](Azure/AZ-104/README.md) | AZ-104 — Azure Administrator | Networking, compute, storage, identity, monitoring, Terraform fundamentals, chaos engineering | 8 |
| [Azure/AZ-305](Azure/AZ-305/README.md) | AZ-305 — Azure Solutions Architect Expert | Identity/governance/monitoring design, data storage design, business continuity design, compute + network infrastructure design, migration/modernization, multi-region design, landing-zone capstone | 8 |
| [Azure/AZ-400](Azure/AZ-400/README.md) | AZ-400 — DevOps Engineer Expert | Source control strategy, CI/CD pipelines, release management, pipeline security, IaC automation, instrumentation, SRE practices, package management | 8 |
| [Azure/AZ-500](Azure/AZ-500/README.md) | AZ-500 — Azure Security Engineer | Identity & access protection, network security, data/app security, security operations, hybrid identity, advanced network security, SQL/storage security, SOAR automation + Terraform | 8 |
| [Azure/General](Azure/General/README.md) | N/A — certification-agnostic | AKS, Bicep, CI/CD with OIDC, secrets/config management, observability/APM, FinOps, landing zone governance, container supply-chain security | 8 |
| [Azure/SC-100](Azure/SC-100/README.md) | SC-100 — Cybersecurity Architect Expert | Zero Trust strategy, identity/access architecture, security operations architecture, compliance/governance, hybrid/multicloud security, network security architecture, app/API security, data security architecture | 8 |
| [Azure/SC-200](Azure/SC-200/README.md) | SC-200 — Security Operations Analyst | Sentinel setup & KQL, detection engineering, Defender XDR, incident response, SOAR automation, threat hunting, cross-workload correlation | 8 |
| [Azure/SC-300](Azure/SC-300/README.md) | SC-300 — Identity and Access Administrator | Identity governance, app integration/SSO, access governance, identity protection + Graph automation, hybrid identity, external identities, workload identity, Lifecycle Workflows | 8 |
| [Azure/SC-500](Azure/SC-500/README.md) | SC-500 — Cloud and AI Security Engineer Associate (beta) | Identity/access/governance for cloud workloads, storage/database security, network security, compute security, AI workload security, Purview DSPM, Defender for Cloud CSPM, Security Copilot | 8 |

### AWS

| Track | Certification | Focus | Labs |
|-------|---------------|-------|------|
| [Aws/SAA-C03](Aws/SAA-C03/README.md) | SAA-C03 — AWS Solutions Architect Associate | VPC/IAM, compute/scaling, storage, resilient architecture, serverless, cost, Terraform | 8 |
| [Aws/SCS-C02](Aws/SCS-C02/README.md) | SCS-C02 — AWS Security Specialty | IAM deep dive, detection & monitoring, data protection, automated incident response, network/perimeter security, compute/container security, multi-account governance, IR forensics capstone | 8 |

---

## Suggested learning path

Tracks are independent — jump straight to whichever cert you're studying for. If you're building this out roughly in order, this is the dependency chain the labs assume:

```
AZ-104 (Azure fundamentals)
   │
   ├──► AZ-500 (security engineering)  ──►  SC-500 (cloud & AI security)
   ├──► SC-300 (identity administration)
   ├──► SC-200 (security operations)
   │
   └──► AZ-305 (solutions architecture)  ──►  SC-100 (security architecture)

AZ-400 (DevOps) and Azure/General (certification-agnostic skills) layer on top of any of the above.

SAA-C03 (AWS fundamentals) ──► SCS-C02 (AWS security specialty) — parallel AWS track, same structure.
```

Later labs within a track assume familiarity with concepts from earlier ones — work each track in order.

---

## Why this structure

Each certification gets its own subfolder so the repo doubles as a study log and a portfolio: jump straight to the cert you care about, see exactly what was built, and read the reasoning behind each decision. Labs build on each other within a track — later labs reuse resources, naming conventions, and concepts from earlier ones — and Infrastructure as Code (Terraform) is woven into the tracks where it's actually relevant on the job, rather than bolted on as a separate afterthought.

[Azure/General](Azure/General/README.md) is the one exception to "each track maps to a certification": it's deliberately certification-agnostic, built to cover high-demand Azure skills (Kubernetes, IaC breadth, CI/CD security, observability, FinOps, landing-zone governance) that show up in job postings and interviews but aren't fully tested by any single exam.

## How to use these labs

1. **Pick a track** based on the certification or skill you're targeting.
2. **Read that track's README** for prerequisites, the lab overview table, and suggested pacing.
3. **Work labs in order** — later labs assume familiarity with concepts from earlier ones in the same track.
4. **Always clean up** at the end of a lab. Every lab includes a cleanup section; skipping it costs real money.

## Prerequisites (all tracks)

- Azure subscription and/or AWS account with free-tier or trial credit
- Azure CLI (`az`) and/or AWS CLI (`aws`) installed and authenticated
- Terraform CLI — install via [terraform.io](https://developer.hashicorp.com/terraform/install)
- A terminal and a text editor
- Basic networking and cloud fundamentals (start with AZ-104 or Aws/SAA-C03 labs 1–3 if you're new to cloud)

## Cost & safety notes

- Every lab targets **$0–$10** and lists an estimated cost up front (a few labs with premium services like Azure Firewall or Security Copilot run higher — flagged explicitly in that track's README)
- **Delete resource groups / `terraform destroy` at the end of every lab** — don't leave infrastructure running between sessions
- Labs that intentionally misconfigure something (chaos/incident-response labs) call that out explicitly and always include the fix
- No real IP addresses, tenant names, account IDs, or credentials are ever committed to this repo — replace placeholders like `<your-subscription-id>` with your own values locally

## Contributing

This is primarily a personal study log, but if you spot an error, a broken link, or an outdated CLI flag, feel free to open an issue or a PR — corrections are welcome even if new labs generally aren't being accepted.

## Disclaimer

This repository is an independent, unofficial study resource. It is not affiliated with, endorsed by, or sponsored by Microsoft or Amazon Web Services. Certification names (AZ-104, AZ-305, AZ-400, AZ-500, SC-100, SC-200, SC-300, SC-500, SAA-C03, SCS-C02, etc.) are trademarks of their respective owners and are used here solely to describe the exam objectives each track targets. Always verify current exam content and study guidance against the official certification pages linked in each track's README.

## License

See [LICENSE](LICENSE).

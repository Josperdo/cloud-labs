# Cloud Labs

A collection of hands-on Azure and AWS labs I put together while studying for various certifications [or just in general). Each lab has you deploy real infrastructure (cheap, disposable stuff), run through validation checks, and clean up at the end so nothing sits there racking up a bill. The goal was to actually build the things these exams test you on, not just read about them.

11 certification tracks, 8 labs apiece, 88 labs total. Grab whichever track matches what you're studying for.

---

## Who this is for

Anyone working toward an Azure or AWS cert who'd rather get hands-on than just memorize exam objectives — self-taught learners, people switching careers into tech, engineers filling in gaps before a test. You don't need prior experience writing IaC or building labs; each track's README spells out what to install first.

This started as my own study log and turned into something I figured might help someone else on the same path. It's not official Microsoft or AWS material, and I'm not affiliated with either company (more on that in the disclaimer below).

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

## Suggested order

The tracks don't depend on each other, so feel free to jump straight to whatever you're studying for. If you're working through this whole repo, though, here's roughly how it's meant to build:

Start with **AZ-104** for the Azure fundamentals. From there it branches: **AZ-500** (security engineering) leads into **SC-500** (cloud & AI security), **SC-300** covers identity administration, **SC-200** covers security operations, and **AZ-305** (solutions architecture) leads into **SC-100** (security architecture). **AZ-400** and **Azure/General** are more standalone — they layer on top of whichever of the above you've already done. On the AWS side, **SAA-C03** comes before **SCS-C02**, same idea as the Azure fundamentals-then-security progression.

Within any single track, work the labs in order — later ones assume you've already got the concepts from earlier ones.

---

## Why it's organized this way

Each cert gets its own folder so this repo works as both a study log and a portfolio — you can jump straight to the track you care about and see exactly what was built and why. Labs build on each other within a track (later ones reuse resources, naming conventions, and ideas from earlier ones), and Terraform shows up where it's actually relevant to the job rather than being bolted on for its own sake.

The one exception is [Azure/General](Azure/General/README.md), which isn't tied to a specific cert. It covers stuff like Kubernetes, IaC beyond Terraform, CI/CD security, and FinOps — skills that show up constantly in job postings but aren't fully tested by any single exam.

## How to use these labs

Pick a track, read its README for prerequisites and pacing, then work the labs in order. Every lab ends with a cleanup step — don't skip it, or you'll end up paying for infrastructure you're not using anymore.

## Before you start

- An Azure subscription and/or AWS account (free tier or trial credit is enough)
- Azure CLI (`az`) and/or AWS CLI (`aws`), authenticated
- Terraform — [terraform.io](https://developer.hashicorp.com/terraform/install)
- A terminal and a text editor
- Basic cloud/networking fundamentals — if you're new to this, start with AZ-104 or the first few SAA-C03 labs

## Cost and cleanup

Most labs land somewhere between $0 and $10, and each one lists its estimated cost up front (a handful of labs touching pricier services like Azure Firewall or Security Copilot run higher — those are called out explicitly). Delete the resource group or run `terraform destroy` when you're done with a lab; don't leave things running between sessions. A few labs deliberately misconfigure something on purpose (the chaos/incident-response ones) — those always walk you through the fix too.

No real IPs, tenant names, account IDs, or credentials are committed anywhere in here. Placeholders like `<your-subscription-id>` are yours to fill in locally.

## Contributing

This is mostly a personal study log, but if you find a broken link, a typo, or a CLI flag that's gone stale, open an issue or a PR. I'm not really looking to add new labs from outside contributors, but corrections are always welcome.

## Disclaimer

This is an independent study resource, not affiliated with or endorsed by Microsoft or AWS. The certification names used throughout (AZ-104, AZ-305, SC-100, SAA-C03, and so on) are trademarks of their respective owners, used here only to describe what each track is studying for. Exam content changes — check the official certification pages (linked in each track's README) for the current word on what's actually tested.

## License

See [LICENSE](LICENSE).

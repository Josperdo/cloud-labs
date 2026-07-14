# Cloud & Linux Labs

Hands-on, certification-aligned labs across Azure and AWS — portal work, CLI, and Infrastructure as Code — built for reps and interview-readiness, not just theory. Every lab deploys real (cheap, disposable) infrastructure, includes validation checkpoints, and ends with cleanup so nothing lingers on a bill.

---

## Tracks

| Track | Certification | Focus | Labs | Status |
|-------|---------------|-------|------|--------|
| [Azure/AZ-104](Azure/AZ-104/README.md) | AZ-104 — Azure Administrator | Networking, compute, storage, identity, monitoring, Terraform fundamentals, chaos engineering | 8 | Completed |
| [Azure/AZ-500](Azure/AZ-500/README.md) | AZ-500 — Azure Security Engineer | Identity & access protection, network security, data/app security, security operations + Terraform | 4 | In progress |
| [Azure/SC-300](Azure/SC-300/README.md) | SC-300 — Identity and Access Administrator | Identity governance, app integration/SSO, access governance, identity protection + Graph automation | 4 | In progress |
| [Azure/AZ-305](Azure/AZ-305/README.md) | AZ-305 — Azure Solutions Architect Expert | Identity/governance/monitoring design, data storage design, business continuity design, compute + network infrastructure design | 5 | In progress |
| [Azure/General](Azure/General/README.md) | N/A — certification-agnostic | AKS, Bicep, CI/CD with OIDC, secrets/config management, observability/APM, FinOps, landing zone governance | 7 | In progress |
| [Aws/SAA-C03](Aws/SAA-C03/README.md) | SAA-C03 — AWS Solutions Architect Associate | VPC/IAM, compute/scaling, storage, resilient architecture, serverless, cost, Terraform | 8 | In progress |
| [Aws/SCS-C02](Aws/SCS-C02/README.md) | SCS-C02 — AWS Security Specialty | IAM deep dive, detection & monitoring, data protection, automated incident response | 4 | In progress |

---

## Why this structure

Each certification gets its own subfolder so the repo doubles as a study log and a portfolio: a reviewer (or future me) can jump straight to the cert they care about, see exactly what was built, and read the reasoning behind each decision. Labs build on each other within a track — later labs reuse resources, naming conventions, and concepts from earlier ones — and Infrastructure as Code (Terraform) is woven into the tracks where it's actually relevant on the job, rather than bolted on as a separate afterthought.

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

- Every lab targets **$0–$5** and lists an estimated cost up front
- **Delete resource groups / `terraform destroy` at the end of every lab** — don't leave infrastructure running between sessions
- Labs that intentionally misconfigure something (chaos/incident-response labs) call that out explicitly and always include the fix
- No real IP addresses, tenant names, account IDs, or credentials are ever committed to this repo — replace placeholders like `<your-subscription-id>` with your own values locally

## License

See [LICENSE](LICENSE).

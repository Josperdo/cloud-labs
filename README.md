# Azure Hands-On Labs

A comprehensive set of hands-on Azure labs covering portal-based administration and Terraform-based infrastructure as code. Labs 1–5 focus on AZ-104 domains via the Azure portal. Labs 6–7 cover Terraform, with skills that transfer directly to AWS. Lab 8 puts everything together by intentionally breaking a live environment to understand real failure modes.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-----------|
| **Lab 1** | Virtual Networks & Connectivity | 45–60 min | ~$0.50 | VNets, subnets, NSGs, peering |
| **Lab 2** | Compute (VMs & Availability) | 50–70 min | ~$2–$5 | VM creation, disk management, availability options |
| **Lab 3** | Storage & Data | 45–60 min | ~$0.50 | Blob tiers, lifecycle policies, replication |
| **Lab 4** | Identity & Access Control | 60–75 min | ~$0 | RBAC, Entra ID, managed identities, custom roles |
| **Lab 5** | Monitoring & Logging | 60–75 min | ~$2–$5 | Log Analytics, KQL, alerts, diagnostics |
| **Lab 6** | Terraform Fundamentals | 60–75 min | ~$0.50 | HCL, plan/apply/destroy, variables, state |
| **Lab 7** | Terraform Modules & Remote State | 65–80 min | ~$1–$2 | Remote state, modules, multi-environment deployments |
| **Lab 8** | Break Things (Chaos Engineering) | 60–90 min | ~$1–$3 | Resource locks, NSG debugging, RBAC scoping, SAS tokens, Terraform lock recovery |

**Total time**: ~9–12 hours
**Total cost**: ~$8–$17 (if done sequentially and cleaned up after each lab)

---

## Prerequisites

- **Azure subscription** with $200+ credit (or free tier)
- **Azure Portal** access
- **SSH/RDP client** (PuTTY, Windows Terminal, or native OpenSSH)
- **Text editor** for viewing queries and configs
- **Basic understanding** of networking, cloud concepts, and security
- **Terraform CLI** (Labs 6–7 only) — install via `brew install hashicorp/tap/terraform` or from [terraform.io](https://developer.hashicorp.com/terraform/install)
- **Azure CLI** (Labs 6–7 only) — install via `brew install azure-cli` or from [Microsoft docs](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)

---

## How to Use These Labs

### Option 1: Sequential (Recommended)
Work through labs 1–8 in order. Each builds foundational concepts:
1. Networks (Lab 1) → Compute (Lab 2) → Storage (Lab 3) → Identity (Lab 4) → Monitoring (Lab 5) → Terraform Fundamentals (Lab 6) → Terraform Modules (Lab 7) → Break Things (Lab 8)

### Option 2: Target Weak Areas
Identify which AZ-104 domain is your weakness and jump to that lab:
- Struggling with VNet configs? → Start with Lab 1
- Unsure about RBAC? → Start with Lab 4
- Monitoring is fuzzy? → Start with Lab 5

### Option 3: Exam Prep Sprint
Do one portal lab per day for 5 days (Labs 1–5). Then tackle the Terraform labs (6–7) as a two-day IaC sprint.

### Option 4: IaC Fast Track
If you already know Azure basics, jump straight to Labs 6 and 7 to focus on Terraform. These skills transfer directly to AWS (same tool, different provider).

---

## Lab Format

Each lab follows this structure:

1. **Overview**: What you'll build and why it matters
2. **Objectives**: Skills you'll gain
3. **Step-by-step walkthrough**: Exact portal clicks and configurations
4. **Validation checkpoints**: How to confirm it worked
5. **Cleanup instructions**: How to delete resources and save credits
6. **Key concepts table**: Exam-relevant definitions
7. **Exam tips**: Patterns you'll see on the test
8. **Next steps**: Extensions to deepen learning

---

## Important Notes

### Cost Management
- Each lab is designed to cost **$0–$5**
- **Always clean up** at the end of each lab (delete the resource group)
- Use **B-series VMs** and **Standard storage** for learning
- Check **Cost Analysis** in Portal if you're concerned

### Timing
- Estimates are conservative; you may finish faster
- Diagrams and explanations are skipped for brevity; Google concepts if unclear
- Portal UI changes; screenshots aren't included—follow the "search for X" instructions

### Exam Alignment
- These labs cover **~80% of AZ-104 exam domains**
- Labs focus on *how things work*, not just definitions
- Each lab includes exam tips relevant to that domain

---

## Common Issues & Troubleshooting

### Issue: "Deployment failed" on resource creation
- **Solution**: Wait a minute and retry. Portal sometimes times out.

### Issue: SSH/RDP connection refused
- **Solution**: 
  - Check NSG has the right inbound rule
  - Check your public IP is correct (use "what's my IP")
  - Wait 5 minutes for rules to apply

### Issue: No data appearing in Log Analytics
- **Solution**: Diagnostic logs take 5–10 minutes to flow. Wait and retry queries.

### Issue: Quota exceeded or "operation not permitted"
- **Solution**: You may have hit resource limits. Delete unused resources or contact Azure support.

### Issue: Confusion about RBAC inheritance
- **Solution**: Draw the scope hierarchy (Subscription → RG → Resource) and trace role assignments top-down.

---

## Additional Resources

### Microsoft Official
- [AZ-104 Study Guide](https://learn.microsoft.com/en-us/certifications/azure-administrator/)
- [Microsoft Learn AZ-104 Path](https://learn.microsoft.com/en-us/training/paths/az-104-administrator-prerequisites/)
- [Azure Portal Documentation](https://docs.microsoft.com/en-us/azure/)

### Query References
- [KQL Cheat Sheet](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference) (for Lab 5)
- [RBAC Built-in Roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles) (for Lab 4)

### Community
- [Azure Tech Community](https://techcommunity.microsoft.com/t5/azure/ct-p/Azure)
- [Reddit: r/Azure](https://reddit.com/r/Azure)
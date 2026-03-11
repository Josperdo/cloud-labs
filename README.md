# AZ-104 Hands-On Labs

A comprehensive set of 5 hands-on Azure labs designed to bridge the gap between exam knowledge and practical portal experience. Each lab focuses on a specific AZ-104 domain with real-world scenarios.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-----------|
| **Lab 1** | Virtual Networks & Connectivity | 45–60 min | ~$0.50 | VNets, subnets, NSGs, peering |
| **Lab 2** | Compute (VMs & Availability) | 50–70 min | ~$2–$5 | VM creation, disk management, availability options |
| **Lab 3** | Storage & Data | 45–60 min | ~$0.50 | Blob tiers, lifecycle policies, replication |
| **Lab 4** | Identity & Access Control | 60–75 min | ~$0 | RBAC, Entra ID, managed identities, custom roles |
| **Lab 5** | Monitoring & Logging | 60–75 min | ~$2–$5 | Log Analytics, KQL, alerts, diagnostics |

**Total time**: ~4.5–5.5 hours  
**Total cost**: ~$5–$11 (if done sequentially and cleaned up after each lab)

---

## Prerequisites

- **Azure subscription** with $200+ credit (or free tier)
- **Azure Portal** access
- **SSH/RDP client** (PuTTY, Windows Terminal, or native OpenSSH)
- **Text editor** for viewing queries and configs
- **Basic understanding** of networking, cloud concepts, and security

---

## How to Use These Labs

### Option 1: Sequential (Recommended)
Work through labs 1–5 in order. Each builds foundational concepts:
1. Networks (Lab 1) → Compute (Lab 2) → Storage (Lab 3) → Identity (Lab 4) → Monitoring (Lab 5)

### Option 2: Target Weak Areas
Identify which AZ-104 domain is your weakness and jump to that lab:
- Struggling with VNet configs? → Start with Lab 1
- Unsure about RBAC? → Start with Lab 4
- Monitoring is fuzzy? → Start with Lab 5

### Option 3: Exam Prep Sprint
Do one lab per day for 5 days. Each lab takes 45–75 minutes plus cleanup.

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

## Suggested Reading Order

Start with the lab overview, then work through each step methodically:

1. **Read the objective** to understand what you're learning
2. **Do each step** in the Portal (don't skip steps, even obvious ones)
3. **Validate** at each checkpoint
4. **Read the key concepts** to reinforce learning
5. **Review exam tips** before moving to the next lab

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

## Study Tips for the Exam

### After Each Lab
- Write a 1-sentence summary of the domain
- Note 1–2 things you weren't sure about
- Bookmark Azure Docs pages for deep dives

### Before Taking the Exam
- Review the "Key Concepts" tables from all 5 labs
- Re-read all "Exam Tips" sections
- Do a quick mental walkthrough of how to create each resource type

### Exam Day
- If you see a question about NSGs, think back to Lab 1's "allow/deny" logic
- If you see RBAC questions, think about scope hierarchy from Lab 4
- If you see KQL, remember Lab 5's `where`, `summarize`, `project` operators

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

---

## Lab Cleanup Checklist

After each lab, verify you've deleted:
- [ ] Resource groups (this deletes everything inside)
- [ ] Any orphaned disks (Storage → Disks)
- [ ] Any orphaned public IPs (Networking → Public IP addresses)
- [ ] Downloaded SSH/RDP keys (local machine)

Check **Cost Analysis** → Last 7 days to confirm costs dropped to near $0.

---

## Roadmap: What's NOT Covered

These labs focus on core AZ-104 infrastructure. Not extensively covered:

- **App Services & PaaS** (Azure App Service, Functions)
- **Databases** (SQL Server, PostgreSQL, Cosmos DB) — foundational but not deep
- **Hybrid scenarios** (Azure Stack, Arc)
- **Advanced networking** (ExpressRoute, VPN gateways in depth)
- **Backup & disaster recovery** (Azure Backup, Site Recovery)
- **Cost optimization** (Reservations, Spot VMs)

If you finish these 5 labs with time before the exam, dive into those topics via Microsoft Learn.

---

## Feedback

If you find errors, outdated portal paths, or unclear instructions:
- Note the lab and step number
- Screenshot the error or confusing UI
- Document what you expected vs. what happened

This helps improve future versions.

---

## Good Luck! 🎯

You're investing time in hands-on experience, which is the **hardest part** of exam prep. Most people skip this and wonder why they fail. You're not doing that. Stick with the labs, take your time, and trust the process.

The exam will feel much more approachable after doing these.

---

**Start with Lab 1 →** `lab-1-virtual-networks.md`

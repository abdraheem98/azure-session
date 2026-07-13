# Hands-on Deliverables Checklist and Review Notes Template

## Deliverables Checklist

Learners submit the following for trainer review:

- [ ] **Architecture diagram** — matches deployed resources, includes subnet CIDR ranges, NSG associations, and data flow direction
- [ ] **Implementation evidence** — CLI output / screenshots for: VNet+subnets, NSGs, Key Vault (RBAC mode), VMSS + autoscale rule, Application Gateway backend health, SQL private endpoint + DNS resolution proof, diagnostic settings, alert rule
- [ ] **Review notes** (template below) covering commands run, screenshots referenced, and findings
- [ ] **Trade-off justification write-up** — VMSS vs AKS decision, and deny-by-default justification (2–3 sentences each)
- [ ] **Risks/assumptions/next-actions section** (template below)

---

## Review Notes Template (learner fills in during lab)

```markdown
# WAF Review Notes — <Learner Name> — Module 1 Lab

## Resources Deployed
| Resource | Name | Notes |
|---|---|---|
| VNet | | |
| Subnets (4) | | |
| NSGs (4) | | |
| Key Vault | | RBAC mode: yes/no |
| VMSS | | instance count, SKU |
| App Gateway | | SKU, backend health status |
| SQL Database | | tier, private endpoint: yes/no |
| Log Analytics WS | | |
| Alert Rules | | |

## WAF Pillar Self-Assessment
- **Reliability:** (e.g., zone redundancy used? autoscale tested? single points of failure remaining?)
- **Security:** (NSG posture, private endpoint status, Key Vault RBAC, any public exposure remaining)
- **Cost Optimization:** (SKU choices vs $800/month ceiling — estimate actual monthly cost)
- **Operational Excellence:** (what's monitored, what alerts exist, what's NOT yet covered)
- **Performance Efficiency:** (autoscale thresholds, load distribution validation)

## Assumptions Made
-

## Known Risks / Gaps
-

## Next Actions (if this were a real engagement)
-
```

---

## Tools & Environment Reference
Azure CLI, Azure IAM/RBAC, VNets, NSGs, Azure Load Balancer/Application Gateway, Azure VM Scale Sets, Azure SQL/Storage, Azure Monitor, Key Vault, Azure DNS overview, ARM/Bicep/Terraform overview, Git.

**Pre-lab check:** confirm all learners have `az --version` ≥ 2.60, an active subscription with Contributor access, and Git configured for committing diagrams/notes to the shared repo.

---

## Bloom Levels & Competency Mapping
- **Apply:** deploying the reference architecture components (Tasks 1, 3, 4, 5)
- **Analyze:** decision-table trade-off discussion, DNS/connectivity troubleshooting
- **Evaluate:** VMSS vs AKS justification, deny-by-default justification, WAF pillar self-assessment
- **Competency built:** Azure architecture decision-making under real constraints (cost ceiling, team size, compliance posture)

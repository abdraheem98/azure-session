# Hands-on Deliverables Checklist and Review Notes Template

## Deliverables Checklist

**Per-lab checkpoints (Labs 1–3)** — see each lab file for its own inline checkpoints. Summary:
- [ ] **Lab 1** — four Entra ID groups created, object ID screenshot
- [ ] **Lab 2** — built-in role notes, custom role JSON, role assignment list, Check access screenshot
- [ ] **Lab 3** — managed identity principal ID, Key Vault role assignment, system- vs user-assigned comparison notes

**Capstone (Lab 4) deliverables:**
- [ ] Built-in role fit-check table with justification
- [ ] DBA assignment scope-confirmation screenshot (GUI)
- [ ] Full role assignment list — all human roles correct
- [ ] Managed identity role assignment confirmed, scoped correctly
- [ ] Three Check access screenshots (Ops/Support, Security Auditor, managed identity)
- [ ] Full access control diagram (human + machine identities) + completed review notes

---

## Review Notes Template

```markdown
# RBAC Review Notes — <Learner Name> — Module 2 Capstone (Lab 4)

## Access Model Summary
| Identity | Type | Role | Scope | Built-in or Custom |
|---|---|---|---|---|
| retailco-app-developers | Group | | | |
| retailco-dba | Group | | | |
| retailco-security-audit | Group | | | |
| retailco-ops-support | Group | | | |
| lab-vmss managed identity | Managed Identity | | | |

## Least-Privilege Justification
- Why does no group have Owner or Contributor at the resource-group level?
- Why was a custom role necessary for Ops/Support?
- Why does the managed identity use the same RBAC mechanism as the human groups, even though it's not a person?

## Validation Evidence
- Check access screenshots referenced: (filenames)
- CLI verification output referenced: (filenames)

## Known Risks / Gaps
- (e.g., no PIM / time-bound access configured yet — flag as a next step)
- (e.g., no access review cadence defined yet)

## Next Actions (if this were a real engagement)
-
```

---

## Tools & Environment Reference
Azure CLI, Azure IAM/RBAC, VNets, NSGs, Azure Load Balancer/Application Gateway, Azure VM Scale Sets, Azure SQL/Storage, Azure Monitor, Key Vault, Azure DNS overview, ARM/Bicep/Terraform overview, Git.

**Pre-lab check:** confirm learners have `az --version` ≥ 2.60, sufficient Azure AD permissions to create groups and custom roles (this may require elevated/admin access in some tenants — flag this as a dependency), and the Module 1 resource group still available.

---

## Bloom Levels & Competency Mapping
- **Apply:** creating groups, writing the custom role, creating role assignments (Tasks 2–4)
- **Analyze:** built-in role fit analysis, scope selection, troubleshooting access issues (Tasks 1, 5)
- **Evaluate:** least-privilege justification, deciding when a custom role is genuinely necessary vs. over-engineering (Task 1, review notes)
- **Competency built:** least-privilege Azure security design

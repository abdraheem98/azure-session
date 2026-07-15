# Module 2 Labs — Index

Four standalone labs, meant to be done in order. Each is self-contained (own scenario, own deliverables) so they can be split across a session if time is short, or assigned individually as practice.

| Lab | Focus | Depends On |
|---|---|---|
| [`lab1_entra_id_fundamentals.md`](lab1_entra_id_fundamentals.md) | Entra ID — users, groups, service principals, viewing directory objects | Module 1 resource group |
| [`lab2_rbac_role_assignments.md`](lab2_rbac_role_assignments.md) | RBAC — built-in vs custom roles, scoped assignments, GUI + CLI | Lab 1 groups |
| [`lab3_managed_identities.md`](lab3_managed_identities.md) | Managed Identities — system-assigned vs user-assigned, wiring an identity to Key Vault | Module 1 VMSS + Key Vault |
| [`lab4_capstone_project.md`](lab4_capstone_project.md) | Capstone — full access control redesign for RetailCo, combining all three | Labs 1–3 |

**Recommended flow for a live session:** run Labs 1–3 back to back during the hands-on block, then either compress Lab 4 into the remaining time as a guided walkthrough, or assign it as follow-up work and review the submitted evidence in a later session. Labs 1–3 alone satisfy all three learning outcomes; Lab 4 is where the outcomes get combined into one realistic deliverable.

**Shell note:** all labs use bash syntax. See `../06_troubleshooting_quickref.md` for PowerShell-specific adjustments (angle-bracket placeholders, `$RANDOM`, JSON quoting) carried over from Module 1.

**Common setup (run once, before Lab 1):**
```bash
RG=waf-lab-<yourinitials>   # your Module 1 resource group
subId=$(az account show --query id -o tsv)
echo $RG
echo $subId
```

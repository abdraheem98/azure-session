# Lab 4 — Capstone Project: Access Control Redesign for RetailCo

**Time:** ~45 minutes (or assign as take-home/follow-up work) | **Combines:** Entra ID (Lab 1), RBAC (Lab 2), and Managed Identities (Lab 3) into one realistic deliverable.

---

## Scenario

RetailCo's order-management platform (Module 1's architecture) is moving from a solo build into a real team environment. You've been asked to design and implement the **complete** access control model — human identities, service roles, and the application's own machine identity — for:

| Role / Identity | Needs |
|---|---|
| **Cloud Architect (you)** | Full control of everything — the only Owner |
| **2 App Developers** | Restart/monitor the VM Scale Set, read logs — nothing else |
| **1 Database Administrator** | Full control of SQL Database only — no access to compute, network, or Key Vault |
| **1 Security Auditor** | Read-only across the entire resource group — cannot change anything |
| **1 Ops/Support Engineer** | Start/stop/restart VMSS only — cannot resize, delete, or reconfigure |
| **The application itself (VMSS)** | Must read Key Vault secrets at runtime — no stored credentials anywhere |

**Constraint carried over from Module 1:** 3-person ops team, cost-conscious, PCI-adjacent client — the model has to be defensible in a real audit, not just "everyone gets Contributor because it's easier."

---

## Task 4.1 — Confirm Prerequisites (2 min)

If you completed Labs 1–3 in this session, the following should already exist — verify before continuing:
```bash
az ad group list --query "[?starts_with(displayName, 'retailco-')].displayName" -o table
az role definition list --custom-role-only true --query "[].roleName" -o table
az vmss identity show -g $RG -n lab-vmss --query principalId -o tsv
```

If any of these come back empty, go back and complete the relevant lab (1, 2, or 3) before proceeding — this capstone assumes all three are in place.

---

## Task 4.2 — Fit-Check the Human Roles (5 min)

Using what you learned in Lab 2, confirm each human role's built-in fit:

| Team Role | Closest Built-in Role | Exact Fit? |
|---|---|---|
| Cloud Architect | Owner | Yes |
| App Developers | Virtual Machine Contributor | Broader than ideal, accepted |
| Database Admin | SQL DB Contributor | Yes |
| Security Auditor | Reader | Yes |
| Ops/Support Engineer | *(none)* | No — custom role from Lab 2 |

**Deliverable:** one sentence justifying why App Developers' slightly-too-broad fit (VM Contributor) is accepted, while Ops/Support's gap required a custom role. (Expected reasoning: VM Contributor's extra permissions — resize, reconfigure — are things developers plausibly need occasionally during active development; Ops/Support's role is meant to be permanently minimal, so the gap was worth closing with a custom role.)

---

## Task 4.3 — Assign the DBA Role via GUI, Practicing Deliberate Scope Selection (8 min)

1. Navigate specifically to the **SQL Database resource** (not the resource group): `Home → your SQL server → your database`.
2. **Access control (IAM)** → **Add role assignment** → **SQL DB Contributor** → assign to `retailco-dba` group.
3. **Before clicking through, check the breadcrumb/scope shown on screen** — confirm it says the database's full path, not the resource group or the SQL server. This is the "implicit scope" trap from Lab 2, Task 2.2 — practice catching it deliberately here.

**Deliverable:** screenshot of the scope confirmation screen before you click "Review + assign."

---

## Task 4.4 — Assign Remaining Roles via CLI (10 min)

```bash
rgId=$(az group show -n $RG --query id -o tsv)
vmssId=$(az vmss show -g $RG -n lab-vmss --query id -o tsv)

auditGroupId=$(az ad group show -g "retailco-security-audit" --query id -o tsv)

# Security auditor: Reader across the whole resource group
az role assignment create --assignee-object-id $auditGroupId --assignee-principal-type Group \
  --role "Reader" --scope $rgId
```

*(Developers and Ops/Support assignments should already exist from Lab 2 — verify rather than re-create:)*
```bash
az role assignment list --resource-group $RG -o table
```

**Deliverable:** full `az role assignment list --resource-group $RG -o table` output showing all human-role assignments with correct roles and scopes.

---

## Task 4.5 — Confirm the Managed Identity Piece Is Wired Correctly (5 min)

This ties Lab 3 into the full model — the application identity is part of the access design, not a separate afterthought.

```bash
vmssIdentityId=$(az vmss identity show -g $RG -n lab-vmss --query principalId -o tsv)
az role assignment list --assignee $vmssIdentityId --all -o table
```

**Deliverable:** confirmation the VMSS's managed identity holds exactly **Key Vault Secrets User** — and nothing broader — alongside the human-identity assignments in the same resource group.

---

## Task 4.6 — Validate Everything with Check Access (5 min)

**GUI:** Resource group → Access control (IAM) → **Check access** → check each of: a member of `retailco-ops-support`, a member of `retailco-security-audit`, and the VMSS's managed identity. Confirm each shows exactly what was designed — no more.

**Deliverable (the most important evidence in this capstone):** Check access screenshots for all three, proving least-privilege end to end — human roles and machine identity alike.

---

## Task 4.7 — Full Access Control Diagram and Review Notes (10 min)

Produce a diagram (draw.io, Excalidraw, or hand-drawn) showing:
- The five human identities/groups on one side, the VMSS's managed identity as a distinct, visually different node (not lumped in with the human groups)
- The resource group and its resources (VMSS, SQL, Key Vault) on the other
- A labeled line from each identity to exactly the resource(s) and role it was granted

**Complete the review notes** (template in `../05_review_deliverables.md`), extended with this capstone-specific question:

> "Why does the managed identity's access model use the same RBAC mechanism as the human groups, even though it's not a person? What would go wrong if the application had instead used a stored Key Vault access key?"

---

## Full Deliverables Checklist (Capstone)

- [ ] 4.2 — built-in role fit-check table with justification
- [ ] 4.3 — DBA assignment scope-confirmation screenshot
- [ ] 4.4 — full role assignment list, all human roles correct
- [ ] 4.5 — managed identity role assignment confirmed, scoped correctly
- [ ] 4.6 — three Check access screenshots (Ops/Support, Security Auditor, managed identity)
- [ ] 4.7 — full diagram + completed review notes with the managed-identity reflection question answered

# RBAC Reference Model

This is the access model used across the theory discussion, the demo, and the hands-on project. It's mapped directly onto the Module 1 architecture (`waf-lab-<yourinitials>` resource group).

---

## Scenario

RetailCo's order-management platform is moving from a solo build into a real team environment. Five roles need access, each with a different job function and a different blast radius if over-permissioned.

## The Reference Model

| Team Role | Needs | Built-in Role Fit? | Scope |
|---|---|---|---|
| **Cloud Architect (you)** | Full control of everything, including granting access to others | Owner | Resource group |
| **App Developers (x2)** | Restart/monitor the VM Scale Set, read logs — nothing else | Virtual Machine Contributor (broader than ideal, but closest built-in fit) | VMSS resource only |
| **Database Administrator** | Full control of SQL Database only | SQL DB Contributor | SQL Database resource only |
| **Security Auditor** | Read-only across the entire resource group | Reader | Resource group |
| **Ops/Support Engineer** | Start/stop/restart VMSS only — no resize, delete, or reconfigure | **No built-in role fits** — nearest options (Virtual Machine Contributor) are too broad | Custom role required, scoped to VMSS resource only |
| **VMSS Managed Identity** (the application itself, not a person) | Read Key Vault secrets at runtime — no stored credentials anywhere | Key Vault Secrets User | Key Vault resource only |

**The row that isn't a person:** the last row is a security principal like any other from RBAC's point of view, but it's a **managed identity** — an identity Entra ID issues automatically for the VMSS resource, with no password for anyone to manage or leak. See `01_theory_content.md` Section 2.5 and `labs/lab3_managed_identities.md` for the full mechanics.

**The gap to call out explicitly:** the Ops/Support Engineer's requirement — restart only — has no clean built-in role match. This is the deliberate teaching moment: sometimes least-privilege requires writing a custom role rather than reaching for the "closest enough" built-in one.

---

## Design Principles Applied

1. **Roles assigned to groups, not individuals.** Every row above is implemented as an Azure AD (Entra ID) group (`retailco-app-developers`, `retailco-dba`, `retailco-security-audit`, `retailco-ops-support`), with the role bound to the group.
2. **Scope as narrow as the job allows.** Only the Cloud Architect and Security Auditor need resource-group-level scope (one to manage everything, one to observe everything). Everyone else is scoped to a single resource.
3. **Custom role only where a genuine gap exists.** Don't default to writing custom roles everywhere — Virtual Machine Contributor and SQL DB Contributor are accepted as reasonable fits for Developers and DBA, even though slightly broader than a perfectly minimal role would be. The custom role is reserved for the one case where the built-in options are clearly wrong (Ops/Support).
4. **No one but the Architect holds Owner.** Contributor and Owner both need justification at every scope — Owner especially, since it also grants the ability to change other people's access.
5. **Machine identities follow the same rules as human ones.** The VMSS's managed identity is scoped as narrowly as any human role — Key Vault Secrets User on one Key Vault, nothing broader — because RBAC doesn't distinguish "trustworthiness" by principal type. A narrowly-scoped machine identity is exactly as important as a narrowly-scoped human one.

---

## Discussion Prompt (run this live with the room)

"Given a 3-person ops team and a PCI-adjacent client, would you ever assign Contributor at the resource-group level to anyone other than the architect? Why or why not?"

**Expected reasoning path:** No — Contributor at resource-group scope is broader than any single job function in this scenario needs. The right answer is narrower roles at narrower (resource-level) scopes, even though that means creating more individual role assignments to manage. This trade-off — operational simplicity vs. least privilege — mirrors the compute trade-off discussion from Module 1: the "easier" option isn't automatically the right one once you weigh it against the client's actual risk profile.

---

## Custom Role Definition — VMSS Restart Operator

Used in the hands-on project to close the gap identified above.

```json
{
  "Name": "VMSS Restart Operator",
  "Description": "Can restart VM Scale Set instances only. No resize, delete, or reconfigure.",
  "Actions": [
    "Microsoft.Compute/virtualMachineScaleSets/read",
    "Microsoft.Compute/virtualMachineScaleSets/restart/action",
    "Microsoft.Insights/diagnosticSettings/read",
    "Microsoft.Insights/logDefinitions/read"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>"
  ]
}
```

**Why each `Actions` entry is there:**
- `virtualMachineScaleSets/read` — needed just to see the resource exists and view its properties
- `virtualMachineScaleSets/restart/action` — the one actual operational permission being granted
- `diagnosticSettings/read` and `logDefinitions/read` — lets the Ops/Support Engineer check logs after a restart, without granting any write/modify permission

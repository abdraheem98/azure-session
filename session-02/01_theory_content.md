# Theory Content (~48-50 minutes)

This is a deeper pass than the original outline's 35-40% split, to properly cover Entra ID and Managed Identities alongside RBAC rather than compressing them. Trim the Section 2.6 controls discussion first if you're running long — it's the most compressible section without losing core learning outcomes.

---

## 2.1 Why RBAC Matters in the TOM (5 min)

**The one idea to land:** In a mature organization, access isn't granted case-by-case by whoever's on duty — it's modeled once, applied consistently, and audited continuously.

**Script:** "Think back to the target operating model rulebook from Module 1. If the Well-Architected Review is the checkpoint for how something is built, RBAC is the checkpoint for who can operate it afterward. In a mature TOM, nobody gets access by someone quietly running `az role assignment create` for them one afternoon. Access is modeled — role by role, scope by scope — reviewed, and then applied the same way every time."

**Why this follows architecture:** "You just spent a whole module building subnets, NSGs, a scale set, a database. All of that security work is worthless the moment someone hands out Owner access on the subscription to make their life easier. RBAC is where architecture-level security either holds up in practice, or quietly falls apart."

**Real-consequence framing:** "In a real incident, the question 'how did the attacker get from a marketing intern's laptop to production database' is almost always an RBAC question. Not a firewall question. Not an encryption question. An RBAC question — some role, at some scope, was broader than it needed to be."

---

## 2.2 Three Distinct Concepts People Conflate: Entra ID, RBAC, and Managed Identities (6 min)

**Framing:** "Before we go further, I need to pull apart three things that get lumped together as 'Azure security' but actually answer three completely different questions. Confusing them is the single most common source of muddled thinking in this space."

| Concept | Question it answers | Layer |
|---|---|---|
| **Microsoft Entra ID** (formerly Azure AD) | "Who are you, and can you prove it?" | Identity platform — authentication |
| **Azure RBAC** | "Now that we know who you are, what are you allowed to do?" | Authorization |
| **Managed Identity** | "How does this *application* prove who it is, without anyone managing a password?" | A specific *type* of identity, issued by Entra ID, consumed by RBAC |

**Say this explicitly:** "Entra ID is the directory — it holds users, groups, and application identities, and it's responsible for authentication: proving you are who you claim to be, often via password + MFA. RBAC doesn't live in Entra ID — it lives in Azure Resource Manager, and it's the layer that decides what an already-authenticated identity can actually do. Managed Identity is a bridge between the two: it's an identity type that Entra ID issues automatically for an Azure resource (like a VM or an App Service), so that resource can authenticate to other Azure services — and then RBAC decides what that identity is allowed to do once authenticated."

**A one-line test to give the room:** "If the question is 'can this user log in,' that's Entra ID. If the question is 'can this already-logged-in user delete this VM,' that's RBAC. If the question is 'how does my app read a Key Vault secret without a password sitting in my code,' that's Managed Identity."

---

## 2.3 Core RBAC Concepts and Terminology (8 min)

**Framing:** "Before we touch the CLI, you need four words down cold. Everything in Azure RBAC is built from exactly four building blocks. Miss one, and the whole model stops making sense."

### The four building blocks

| Term | What it actually is | Simple analogy |
|---|---|---|
| **Security Principal** | The "who" — a user, a group, a service principal (app), or a managed identity | The person or thing asking to get in |
| **Role Definition** | The "what" — a named collection of permissions | The job description |
| **Scope** | The "where" — management group, subscription, resource group, or single resource | The building, floor, or room they're allowed into |
| **Role Assignment** | The binding — this principal + this role + this scope, glued together | The signed badge that grants entry |

**Key line:** "A role assignment is not a role. A role definition just describes a set of permissions sitting on a shelf, unused. Nothing happens until you create a role assignment — that's the moment a specific person or identity is actually granted that role at a specific scope."

### Scope hierarchy — inheritance

```
Management Group
   └─ Subscription
        └─ Resource Group
             └─ Resource (e.g., one VM, one storage account)
```

**Say explicitly:** "If you assign a role at the subscription level, it applies to every resource group and every resource underneath it — automatically, whether you meant it to or not. This is the single most common RBAC mistake in real environments: someone means to grant access to one resource group, assigns it at the subscription instead, and suddenly that person can see and touch everything in the subscription."

### Built-in roles vs. custom roles

- **Owner** — full access, including the ability to grant access to others
- **Contributor** — full access to manage resources, but cannot grant access to others
- **Reader** — can view everything, change nothing
- **Custom roles** — you define the exact permission set when no built-in role fits cleanly, especially for least-privilege scenarios

**Example:** "A built-in role like Contributor on a VM lets someone start it, stop it, resize it, delete it, reinstall the OS — everything except manage who else has access. If your actual requirement is 'this person should only be able to restart the VM and read its logs,' Contributor is already too broad. That's when you reach for a custom role."

### Deny Assignments (brief mention)

"A rarer, fifth concept — deny assignments — explicitly blocks an action regardless of any role assignment that would otherwise allow it. Mostly created by Azure itself to protect managed resources rather than something you'll hand-author often. Know the term: deny always wins over allow, no matter how permissive the role assignment is."

---

## 2.4 The RBAC Reference Model (8 min)

Full detail lives in `02_rbac_reference_model.md`. Introduce it here as: "We're going to map real team roles onto the Module 1 architecture — a Cloud Architect, App Developers, a DBA, a Security Auditor, and an Ops/Support Engineer — and figure out, for each one, whether a built-in role fits or whether we need to build a custom one. This becomes the design you implement in the hands-on project."

**Discussion prompt to run live:** "Given a 3-person ops team and a PCI-adjacent client, would you ever assign Contributor at the resource-group level to anyone other than the architect? Why or why not?" (Expected direction: no — Contributor at RG scope is almost always broader than any single job function needs; the answer is narrower roles at narrower scopes, even if it means more assignments to manage.)

---

## 2.5 Managed Identities — Deep Dive (7 min)

**Framing:** "We used Key Vault in Module 1 and I mentioned 'managed identity reads the secret at runtime' without fully unpacking it. Let's fix that now — this is one of the highest-leverage security patterns in Azure, and most teams underuse it."

### The problem Managed Identity solves

"Before Managed Identity existed, if your app needed to authenticate to Azure SQL, Key Vault, or Storage, you had exactly two bad options: hardcode a password/connection string in your app config (which then sits in source control, config files, and logs forever), or create a service principal with its own credential and rotate that credential manually forever. Both are a standing liability."

### What a Managed Identity actually is

"It's an identity that Entra ID automatically creates and manages *for an Azure resource* — a VM, a VM Scale Set, an App Service, a Function App. Azure handles the credential lifecycle entirely: creation, rotation, and — critically — you never see or handle the credential yourself. Your code just asks for a token; Azure hands it one."

### Two types — know the difference

| Type | Lifecycle | Best for |
|---|---|---|
| **System-assigned** | Created and destroyed with the resource itself — 1:1 binding | A single resource that needs its own identity, e.g. one VM reading one Key Vault |
| **User-assigned** | Created independently, exists as its own resource, can be attached to multiple resources | Shared identity across several resources — e.g. every instance in a VM Scale Set uses the same identity |

**Example to use with the room:** "Our VM Scale Set from Module 1 is a good case for user-assigned — as instances scale in and out, you want them all presenting the same identity, not each spinning up a brand-new one. A single standalone VM, on the other hand, is often simpler with system-assigned — when the VM is deleted, its identity disappears with it, no cleanup needed."

### How it actually connects to RBAC

"Here's the part that ties this whole module together: a Managed Identity is just another **security principal** — the same 'who' from our four RBAC building blocks. Once a VM has a managed identity, you assign that identity a role — Reader on a Key Vault, Contributor on a Storage Account, whatever it needs — exactly the same `az role assignment create` pattern as for a user or a group. The only difference is *what* is authenticating: a resource, not a person."

**The pattern, step by step:**
1. Enable a managed identity on the resource (VM, App Service, etc.)
2. Azure creates a corresponding identity in Entra ID
3. You grant that identity an RBAC role at the appropriate scope (e.g., "Key Vault Secrets User" on the Key Vault)
4. Your application code calls the Azure Identity SDK, which silently requests a token using the managed identity — no credentials in code, ever

**Line to land:** "This is why Module 1's Key Vault pattern worked the way it did. The app never held a password. It held an identity, and RBAC decided what that identity could read."

---

## 2.6 Security, Governance, and Operational Controls (8 min)

### Identity design patterns
- **Assign roles to groups, never individuals.** Add/remove people from an Azure AD (Entra ID) group; never hunt down every resource a departing employee had direct access to.
- **Principle of least privilege**: start from zero access, add exactly what's needed — not "start from Contributor and hope nobody notices."

### Privileged Identity Management (PIM) — introduce the concept
- PIM allows **just-in-time, time-bound** role activation instead of standing (always-on) access.
- **Example:** instead of a DBA holding SQL DB Contributor permanently, PIM lets them "activate" that role for 4 hours when they actually need it, with the activation logged and optionally requiring approval.
- **Why it matters for the TOM:** standing privileged access is a permanent attack surface; PIM shrinks that surface to only the moments it's actually being used.

### Conditional Access (brief mention, ties identity to context)
- Conditional Access policies can require MFA, block sign-in from unmanaged devices, or restrict access by location — layered on top of RBAC, not a replacement for it.
- **Key distinction to make clear:** RBAC controls *what* an authenticated identity can do. Conditional Access controls *whether* that identity can authenticate at all, under what conditions. Both matter; they answer different questions.

### Access Reviews
- Scheduled, recurring reviews where a designated approver confirms each assignment is still needed — catches "temporary" access that became permanent by neglect.
- **Example:** a contractor's SQL access from a 2-week project that never got revoked six months later — an access review is the control that catches this before an audit does.

### Break-glass accounts
- Emergency accounts, excluded from Conditional Access policies, used only when normal admin access is unavailable (e.g., MFA provider outage). Tightly monitored, rarely used, credentials stored securely offline.

### Governance tie-in (from Module 1, extended)
- Azure Policy can enforce RBAC-adjacent rules too — e.g., policies that block creation of new role assignments at the subscription level, or require that custom roles follow a naming convention.

**Line to tie it together:** "Notice the pattern again from Module 1: identity, access reviews, PIM, Conditional Access — none of these are 'configure once and forget.' RBAC is a living model, not a one-time setup task."

---

## 2.7 Failure Handling, Troubleshooting, and Validation (6 min)

Full detail with fixes lives in `06_troubleshooting_quickref.md`. Name these explicitly in the room:

1. **Scope creep** — a role assigned at subscription or management-group level when resource-group or resource scope was intended. The single most common real-world RBAC mistake.
2. **Group assignment lag** — Azure AD group membership changes can take time to propagate into effective permissions (usually within an hour, but not instant) — don't assume immediate effect when testing.
3. **Custom role JSON errors** — a malformed `AssignableScopes` array, or an `Actions` string that doesn't match a real resource provider operation, causes role creation to fail with a validation error naming the bad field.
4. **"Access denied" despite a correct-looking assignment** — often a **Conditional Access** policy blocking sign-in, not an RBAC problem at all; check sign-in logs, not just role assignments, when troubleshooting access issues.
5. **Deny assignment overriding an expected allow** — rare, but if a resource is protected by Azure (e.g., certain managed app resources), no role assignment will override it; check for deny assignments specifically if a normally-sufficient role isn't working.
6. **Managed identity token request fails** — the identity exists and has a role assigned, but the app still can't authenticate. Usually one of: the identity wasn't actually enabled on the resource (check `az vmss identity show` or equivalent), the role assignment targets the wrong identity (system-assigned vs. user-assigned have different object IDs — easy to mix up), or the role was assigned but hasn't propagated yet (same propagation delay as any other assignment).

**Validation tools:**
- **Check access** blade (Portal, IAM → Check access) — shows a specific principal's effective roles at a given scope.
- `az role assignment list --assignee <id> --all` — CLI equivalent, lists all assignments for a principal across scopes.
- **Azure AD Sign-in logs** — for diagnosing authentication/Conditional Access issues distinct from RBAC issues.

---

## 2.8 Evidence Required for Trainer/Stakeholder Review (5 min)

Preview only — full checklist in `05_review_deliverables.md`:
1. RBAC model — the design (who gets which role, at which scope, and why)
2. Implementation evidence — groups created, custom role JSON, role assignment list output
3. Review notes — least-privilege justification, validation evidence, known gaps
4. Diagram — identities/groups mapped to resources and roles

**Line to tie it together:** "Same standard as Module 1 — in a real engagement, this evidence set is what you'd hand to a client's security or compliance team, not just me."

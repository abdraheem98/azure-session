# Theory Content (~35 minutes)

---

## 2.1 Why Advanced Architecture + Well-Architected Review Matters in the TOM (5 min)

- The Well-Architected Framework (WAF) is Microsoft's structured method for evaluating workloads against five pillars: **Reliability, Security, Cost Optimization, Operational Excellence, Performance Efficiency**.
- In a target operating model, WAF reviews become a **governance gate**: no workload moves to production without a documented review and remediation plan.
- Position this module as the bridge between "I can build Azure resources" (earlier modules) and "I can defend *why* I built them this way" (this module).

**Simple analogy if the room needs it:** Imagine building a house. You could build it fast and cheap, but it might not survive a storm (reliability), a burglar could break in easily (security), the electricity bill could be huge (cost), you'd have no idea if a pipe was leaking until your basement flooded (operations), and it might be too small once your family grows (performance). A Well-Architected Review is a checklist that makes you check all five *before* you move in, not after something breaks.

**Talking point:** Architecture decisions made without a WAF lens tend to optimize for one pillar (usually cost or speed) at the expense of the others. The review process forces trade-off conversations explicitly.

**TOM framing:** "Think of the target operating model as the rulebook for how this company runs Azure long-term. Inside that rulebook, the Well-Architected Review is a checkpoint — like a code review, but for architecture. Nothing goes to production until someone has walked through all five pillars and written down the answer to 'what happens if this breaks, gets attacked, costs too much, nobody's watching it, or gets more traffic than expected.'"

---

## 2.2 Core Concepts, Terminology, Architecture (10 min)

**The five WAF pillars — memorize the question, not a definition:**

| Pillar | The question to ask |
|---|---|
| **Reliability** | "Will this survive a zone failure, a region failure, or a bad deployment?" |
| **Security** | "Who can reach this, and what happens if they shouldn't have been able to?" |
| **Cost Optimization** | "Are we paying for capacity we don't use?" |
| **Operational Excellence** | "If this breaks at 2am, how do we find out — and how fast?" |
| **Performance Efficiency** | "Does the scaling model match how people actually use this?" |

Memory hook: **RSCOP** — Reliability, Security, Cost, Operations, Performance (not an official acronym, just a recall aid).

**Reference architecture — see `02_reference_architecture.md` for the full diagram and narration.** Introduce it here briefly: a 3-tier web app (web/app/data/mgmt subnets, Application Gateway in front, NSGs everywhere, Key Vault for secrets, Azure Monitor for observability).

**Line to land:** "Every resource we build in the demo and lab today, I want you asking these five questions about it. Not as a formality — as a habit."

---

## 2.3 Implementation Standards and Design Trade-offs (8 min)

Covered in full detail in `02_reference_architecture.md` (compute including serverless, data tier, networking, IaC tool). In the room, introduce it as:

"So far everything sounds like there's one right way to build this. There isn't. Every decision is a trade-off. I want to walk you through the big ones, and then I'm going to put you on the spot with a scenario."

Run the discussion prompt from `02_reference_architecture.md` here and let 1–2 people answer before moving on — don't fill the silence yourself.

---

## 2.4 Security, Governance, and Operational Controls (7 min)

### Identity and RBAC
- Use **Azure AD groups**, not individual user role assignments — add/remove people from groups, don't hunt down every resource they had access to.
- Prefer **custom roles** scoped to exactly what's needed over the built-in **Contributor** role.
- *Example:* a developer who only needs to restart a web app and view logs shouldn't get Contributor (which could also let them delete the database) — a custom role scoped to just those actions is least-privilege in practice.

### Network Controls
- **NSGs at the subnet level**, deny-by-default, explicit allow rules.
- **No direct public IPs** on VMs in app/data tiers — the only public entry point is the Application Gateway.
- *Example:* a public IP accidentally attached to a database VM "just to test something" is an instant WAF Security finding.

### Secrets Management
- Every secret goes into **Key Vault**, referenced via **managed identity** — never hardcoded into app settings or IaC variable files.
- *Example:* a DB password in a Bicep parameter file sits in Git history forever, even after deletion. A managed identity reading from Key Vault at runtime means no password ever touches source control.

### Governance
- **Azure Policy** enforces org-wide rules automatically (required tags, allowed regions, allowed VM sizes).
- **Management Groups** apply guardrails at the subscription level so new subscriptions inherit rules automatically.
- *Example:* a required `environment` tag lets Finance report Dev vs. Prod spend without chasing anyone down.

### Operational Controls
- **Diagnostic settings** on every resource → Log Analytics workspace — if it isn't logging, you can't alert on it.
- **Alert rules** on metrics that matter (CPU, gateway 5xx rate, SQL DTU) routed through an **Action Group**.
- *Example:* without an alert on Application Gateway 5xx rate, the first sign of an outage is a customer complaint, not a notification.

**Line to tie it together:** "Notice the pattern across all five: identity, network, secrets, governance, operations — none of them are 'set once and forget.' They're all about making the safe way to do something also the easy way to do something, so people don't have to be heroically careful every single time."

---

## 2.5 Failure Handling, Troubleshooting, and Validation (3 min)

**Common failure points to name explicitly (full detail with examples in `06_troubleshooting_quickref.md`):**

1. **NSG rule ordering/priority conflicts** — rules are evaluated lowest-priority-number first; a broad deny at a lower number silently blocks an allow at a higher number.
2. **Application Gateway health probe misconfiguration** — probe checking the wrong path/port marks healthy instances as unhealthy.
3. **Private endpoint DNS resolution failures** — the most common real-world private endpoint issue; without a linked Private DNS Zone, resolution falls back to public or fails outright.
4. **Missing GatewayManager NSG rule** — Application Gateway v2 needs inbound access on ports 65200–65535 from the `GatewayManager` service tag; omit it and the Gateway fails to provision or reports unhealthy.
5. **Scale set instances failing custom script extension** — instance shows "running" but never becomes healthy.

**Validation tools:**
- `az network watcher` — connectivity troubleshooting between two points in the network.
- Application Gateway backend health blade / `az network application-gateway show-backend-health` — which backend instances are healthy and why.
- `az vmss list-instances` — per-instance provisioning state.

**Line to tie it together:** "Every one of these failure modes has one thing in common — the fix isn't 'redeploy and hope.' It's running the specific validation command that isolates which layer broke: network, gateway, DNS, or the instance itself."

---

## 2.6 Evidence Required for Trainer/Stakeholder Review (2 min)

"Last theory point before we build anything. I want you to know now what you'll need to hand me at the end — because the best evidence gets captured while you're working, not reconstructed afterward from memory."

Preview only — full checklist is in `05_review_deliverables.md`:
1. Architecture diagram matching what was actually deployed
2. Implementation evidence (CLI output/screenshots) for every major resource
3. Review notes covering all five WAF pillars against what was actually built
4. Trade-off justifications (compute choice, deny-by-default reasoning)
5. Risks, assumptions, and next actions

**Line to tie it together:** "None of this is busywork. In a real engagement, this exact set of artifacts is what you'd hand to a client's architecture review board. Today it's me reviewing it instead of them, but the standard is the same."

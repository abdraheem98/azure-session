# Module 2 — Azure RBAC, Identity and Security Controls
## Session Plan

**Duration:** 2 hours | **Split:** ~35-40% Theory (48 min) / ~60-65% Hands-on (72 min)
**Dependency:** Module 1 (Advanced Azure Architecture and Well-Architected Review) — this module reuses the same lab resource group and architecture.

---

## Timing Plan

| Segment | Time | Activity |
|---|---|---|
| Opening & context | 5 min | Connect to Module 1, frame RBAC in the TOM |
| Theory block | 48 min | Entra ID vs RBAC vs Managed Identities, concepts, reference model, controls, failure handling |
| Demo walkthrough | 20 min | Trainer-led GUI + CLI role assignment build |
| Hands-on labs | 37 min | Four short labs — see `labs/` folder |
| Wrap-up & review evidence check | 10 min | Collect deliverables, Q&A |

**Note:** theory runs longer than the original 35-40% split to give Entra ID and Managed Identities proper standalone coverage rather than compressing them into the RBAC discussion. If time is tight, Lab 4 (the capstone) can be assigned as take-home/follow-up work instead of completed live — Labs 1–3 alone cover every learning outcome.

---

## Learning Outcomes (recap for opening slide)

By the end of this module, learners will be able to:
1. Explain the role of Azure RBAC in the target operating model.
2. Apply Azure RBAC patterns through a realistic hands-on workflow.
3. Produce trainer-reviewable evidence that validates design and implementation quality.

---

## Opening Script (5 min)

"Last module, we built the architecture — the network, the compute, the data tier, all wrapped in a Well-Architected Review. But we glossed over one huge question the whole time: who's actually allowed to touch any of this? That's what today is about. Azure RBAC — Role-Based Access Control — is how we answer 'who can do what, to which resource, and why.' By the end of today, you won't just deploy resources — you'll design and defend an access model for them."

**Framing line:**

"Every RBAC decision answers three questions at once: who — which identity. What — which role, meaning which permissions. Where — which scope, meaning which slice of the environment. Get any one of those three wrong, and you've either blocked someone from doing their job, or handed them the keys to something they shouldn't touch."

**Connect explicitly to Module 1:**

"We're reusing the exact same retail order-management architecture from Module 1 — same VNet, same subnets, same VM Scale Set, same SQL database. Today we're not adding new infrastructure. We're deciding who gets to touch what we already built."

---

## File Map for This Module

| File | Contents |
|---|---|
| `00_session_plan.md` | This file — timing, outcomes, opening script |
| `01_theory_content.md` | Full theory script — Entra ID vs RBAC vs Managed Identities, terminology, TOM framing, controls, failure handling, evidence |
| `02_rbac_reference_model.md` | The reference access model mapped onto the Module 1 architecture — roles, scopes, built-in vs custom |
| `03_demo_script.md` | Trainer-led demo — GUI walkthrough + CLI equivalent, step-by-step |
| `05_review_deliverables.md` | Deliverables checklist + review notes template |
| `06_troubleshooting_quickref.md` | Common RBAC/Entra ID/Managed Identity issues and fixes — check this first if something doesn't work as expected |
| `labs/` | Four standalone hands-on labs — see `labs/README.md` for the index and recommended order |

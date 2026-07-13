# Module 1 — Advanced Azure Architecture and Well-Architected Review
## Session Plan

**Duration:** 2 hours | **Split:** ~35% Theory (42 min) / ~65% Hands-on (78 min)

---

## Timing Plan

| Segment | Time | Activity |
|---|---|---|
| Opening & context | 5 min | Frame the module in the target operating model |
| Theory block | 35 min | Concepts, WAF pillars, architecture patterns, trade-offs |
| Demo walkthrough | 20 min | Trainer-led build of reference scenario |
| Hands-on lab | 50 min | Learners build + validate capstone scenario |
| Wrap-up & review evidence check | 10 min | Collect deliverables, Q&A |

---

## Learning Outcomes (recap for opening slide)

By the end of this module, learners will be able to:
1. Explain how advanced Azure architecture and the Well-Architected Review fit into the organization's target operating model (TOM).
2. Apply Well-Architected patterns (reliability, security, cost, operations, performance) through a hands-on, realistic workflow.
3. Produce trainer-reviewable evidence — diagrams, CLI/Terraform output, screenshots, and review notes — that demonstrates design and implementation quality.

---

## Opening Script (5 min)

"Today's module bridges two things you already know how to do — building Azure resources — with something you haven't practiced yet: defending *why* you built them that way. That's what a Well-Architected Review is. Every workload in a mature Azure operating model has to pass through this review before it goes to production. So by the end of these two hours, you won't just have deployed a 3-tier app — you'll have a documented justification for every major decision in it."

**Framing line to land before moving into theory:**

"Architecture decisions made without a Well-Architected lens tend to optimize one thing — usually cost or speed — at the expense of everything else. The review process is what forces the trade-off conversation to happen out loud, instead of silently, in production, during an incident."

---

## File Map for This Module

| File | Contents |
|---|---|
| `00_session_plan.md` | This file — timing, outcomes, opening script |
| `01_theory_content.md` | Full theory script, all 8 sub-topics |
| `02_reference_architecture.md` | Architecture diagram + full design trade-offs (including serverless) |
| `03_demo_script.md` | Trainer-led demo, step-by-step commands + narration |
| `04_hands_on_lab.md` | Learner lab tasks 1–6, step-by-step commands |
| `05_review_deliverables.md` | Deliverables checklist + review notes template |
| `06_troubleshooting_quickref.md` | Known issues hit during dry-runs and their fixes — check this first if a command errors |

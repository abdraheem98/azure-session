# Reference Architecture and Design Trade-offs

This is the single architecture used across the theory discussion, the demo, and the hands-on lab. Everything downstream (commands in `03_demo_script.md` and `04_hands_on_lab.md`) builds this exact shape.

---

## The Diagram

```
Internet
   │
Azure Application Gateway (WAF_v2, public IP)
   │
Virtual Network
 ├─ Subnet: web-tier   → VM Scale Set (Standard_B2s, autoscale) behind Gateway
 ├─ Subnet: app-tier   → VM Scale Set / internal services, reachable only from web-tier
 ├─ Subnet: data-tier  → Azure SQL (private endpoint) + Storage Account, reachable only from app-tier
 └─ Subnet: mgmt-tier  → Bastion / jumpbox, admin access only
NSGs on every subnet (deny-by-default, explicit allow)
Key Vault for secrets/certs (RBAC mode)
Azure Monitor + Log Analytics + Diagnostic settings on all resources
```

**Plain-language walkthrough (use this narration with the room):**

"Traffic comes in from the internet, hits an Application Gateway — a smart traffic cop with a built-in firewall. It only forwards traffic into our network.

Inside, we've split things into four lanes, called subnets:
- **Web tier** — handles incoming requests, running as a Scale Set so it can grow or shrink with traffic
- **App tier** — the business logic, only reachable from the web tier
- **Data tier** — the database, only reachable from the app tier, never exposed to the internet at all
- **Management tier** — a private way in for admins

Every lane has a guard — a Network Security Group — that only lets in traffic from the lane before it. Nothing skips ahead.

Secrets like passwords live in Key Vault, not in code. Everything reports back to Azure Monitor."

**Mapping to the five pillars:**
- Scale set across zones → **Reliability**
- NSGs + private data tier + Key Vault → **Security**
- Small VM size + autoscale → **Cost**
- Azure Monitor everywhere → **Operations**
- Load Balancer/Gateway distributing traffic → **Performance**

---

## Design Trade-offs — Full Detail

**Framing for the room:** "Every architecture choice is a trade-off between two or more reasonable options. There's no universally correct answer — it depends on the client's team, budget, and future plans."

### Decision Area 1: Compute (full spectrum, including serverless)

| Option | What it is | Best for | Avoid when |
|---|---|---|---|
| **VM Scale Sets** | Identical VMs that auto-scale | Steady, always-on workloads; teams comfortable with VM-level ops | Team wants to avoid OS patching entirely |
| **AKS (Kubernetes)** | Container orchestration platform | Many microservices sharing infrastructure; multi-cloud portability needed | Small team with no Kubernetes experience |
| **App Service** | Fully managed web app hosting | Standard web apps/APIs, fast time-to-production | Need for container orchestration or custom OS-level control |
| **Azure Container Apps** | Serverless containers | Microservices, team wants zero Kubernetes management; bursty traffic | Extremely fine-grained cluster control required |
| **Azure Functions** | Serverless, event-driven code | Short-lived tasks triggered by events (uploads, queues, timers) | Long-running or stateful processes |

**When to reach for serverless specifically (Functions and/or Container Apps):**
1. Workload is bursty/unpredictable — don't want to pay for idle capacity between bursts.
2. Work is event-driven — something happens (upload, queue message, schedule) and code reacts.
3. Team wants to avoid infrastructure management entirely.
4. Cost efficiency at low/idle usage matters — pay only when code runs.

**Worked examples:**
- *Nightly batch job (5 min/night):* Azure Functions Timer trigger — pay for 5 minutes, not 24 hours.
- *Image resizing on upload:* Azure Functions with Blob Storage trigger — no server waits around for uploads.
- *Order confirmation emails (volume varies 10x–10,000x):* Azure Functions with Queue trigger — scales automatically, costs nothing when idle.
- *8 microservices, team doesn't want Kubernetes:* Azure Container Apps — microservice-style scaling without `kubectl`.
- *When NOT to use serverless:* a checkout service needing a live in-memory session per user, constant traffic, strict low latency — VM Scale Sets or App Service fit better than stateless, short-lived Functions.

**Teaching point:** a well-architected system often mixes compute models — Container Apps/VMSS for the steady checkout/inventory workload, Functions for bursty event-driven pieces like emails or reports. Mixing is itself good architecture.

### Decision Area 2: Data Tier

| Option | Best for | Avoid when |
|---|---|---|
| **Azure SQL (PaaS, fixed tier)** | Predictable, steady database load | Load is highly variable/seasonal |
| **Azure SQL (Serverless tier)** | Spiky/intermittent load, dev/test, apps not used 24/7 | Constant, high, predictable load |
| **SQL on a VM (IaaS)** | Legacy version/config requirements, full control needed | Team wants to avoid patching/backup management |

**Example:** internal HR tool used 9am–6pm weekdays → Azure SQL Serverless tier (auto-pauses outside business hours). High-traffic always-on order database → fixed vCore tier.

### Decision Area 3: Networking

| Option | Best for | Avoid when |
|---|---|---|
| **Application Gateway alone** | Single-region app, regional user base | Global user base, multi-region failover requirement |
| **Front Door + Application Gateway** | Global users, lowest latency per region, multi-region DR | Simple internal tool with no global reach |

**Example:** internal single-office tool → Application Gateway alone. Global customer-facing app with 99.99% SLA → Front Door in front of regional Application Gateways.

### Decision Area 4: Infrastructure-as-Code Tool

| Option | Best for | Avoid when |
|---|---|---|
| **Bicep** | Azure-only, tightest native integration, simpler syntax | Explicit multi-cloud strategy |
| **Terraform** | Multi-cloud today or planned, one consistent tool across clouds | Small Azure-only team with no multi-cloud need |

**Example:** Azure-committed client → Bicep. Client wants AWS portability in 18 months → Terraform now, despite more setup work.

---

## Discussion Prompt (run this live with the room)

"Your client: 6-person ops team, no Kubernetes experience, multi-cloud in 18 months. Their app has a steady checkout/inventory workload *and* a bursty, event-driven piece (emails, reports). Walk through your pick for compute, data tier, networking, and IaC tool — and say where serverless fits, if anywhere."

**Expected reasoning path:** Container Apps or VMSS for the steady workload + Azure Functions for the event-driven pieces (the mixing is the "aha" moment); Azure SQL PaaS, serverless tier if not always-on; Application Gateway alone unless global users are mentioned; Terraform because of the explicit multi-cloud requirement.

# Demo — Step-by-Step Walkthrough (Trainer-led, ~20 minutes)

**Goal:** Trainer builds a scaled-down version of the reference architecture live, narrating decisions against the WAF pillars. Learners watch, then repeat a fuller version in the lab.

**Shell note:** commands below are written in bash style (`\` line continuation, `$VAR` syntax). If you're demoing from PowerShell, replace `\` with a backtick `` ` `` at line-ends, and see `06_troubleshooting_quickref.md` for the specific gotchas (angle-bracket placeholders, `$RANDOM`, JSON quoting) before you present.

**Opening line to the room:** "Everything we just talked about — pillars, trade-offs, security controls — now becomes real. I'm going to build a scaled-down version of our reference architecture live, and at each step I'll call out which pillar that decision serves. Watch the pattern, because you'll repeat a fuller version yourselves in the lab right after."

---

### Step 1 — Set context and variables

```bash
az login
az account set --subscription "<your-subscription-id>"
RG=waf-demo-rg
LOC=eastus
az group create -n $RG -l $LOC
```

*Say while running:* "One resource group for the whole demo — in a real client engagement you'd likely separate network and workload into different resource groups, but for a live demo, one keeps it simple to follow."

**Pre-flight (do this once, before Step 1, if this subscription hasn't been used for these services before):**
```bash
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Sql
az provider register --namespace Microsoft.KeyVault
```

---

### Step 2 — Create the VNet and subnets

```bash
az network vnet create -g $RG -n demo-vnet --address-prefix 10.10.0.0/16 \
  --subnet-name web-subnet --subnet-prefix 10.10.1.0/24

az network vnet subnet create -g $RG --vnet-name demo-vnet -n app-subnet --address-prefix 10.10.2.0/24
az network vnet subnet create -g $RG --vnet-name demo-vnet -n data-subnet --address-prefix 10.10.3.0/24
az network vnet subnet create -g $RG --vnet-name demo-vnet -n mgmt-subnet --address-prefix 10.10.4.0/24
```

*Narrate:* "Four subnets, one per tier — this segmentation is what lets us apply NSGs and Azure Policy per tier instead of one blanket rule for everything."

---

### Step 3 — Apply NSGs (deny-by-default pattern)

```bash
az network nsg create -g $RG -n web-nsg

# Real traffic: HTTPS from the internet
az network nsg rule create -g $RG --nsg-name web-nsg -n Allow-HTTPS-Inbound \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --destination-port-ranges 443 --source-address-prefixes Internet

# REQUIRED: Application Gateway v2 needs Azure's GatewayManager service to reach
# its management ports, or the NSG's default deny-all silently blocks provisioning/health
az network nsg rule create -g $RG --nsg-name web-nsg -n Allow-GatewayManager-Inbound \
  --priority 110 --direction Inbound --access Allow --protocol Tcp \
  --destination-port-ranges 65200-65535 --source-address-prefixes GatewayManager

az network vnet subnet update -g $RG --vnet-name demo-vnet -n web-subnet --network-security-group web-nsg
```

*Narrate:* "Two allow rules, not one. The first is our real traffic — 443 from the Internet. The second is a classic gotcha: Application Gateway v2 needs Azure's own management plane to reach it on ports 65200–65535, from the GatewayManager service tag. Forget this rule and the Gateway fails to provision or shows unhealthy — and it looks like your mistake when it's really a missing NSG rule. App and data subnets get NSGs that only allow traffic from the tier above them — describe this pattern rather than building every NSG live, to save time."

---

### Step 4 — Deploy Key Vault, demonstrate managed identity pattern

```bash
KV_SUFFIX=$RANDOM   # bash only — see troubleshooting doc for the PowerShell equivalent
az keyvault create -g $RG -n waf-demo-kv$KV_SUFFIX -l $LOC --enable-rbac-authorization true
```

*Narrate:* "RBAC-mode Key Vault, not access-policy mode — this is the current recommended standard so Key Vault permissions live in the same RBAC model as everything else."

---

### Step 5 — Deploy Application Gateway (WAF SKU)

```bash
az network public-ip create -g $RG -n appgw-pip --sku Standard --allocation-method Static

az network application-gateway create -g $RG -n demo-appgw \
  --sku WAF_v2 --capacity 2 \
  --vnet-name demo-vnet --subnet web-subnet \
  --public-ip-address appgw-pip \
  --priority 100
```

*Narrate:* "WAF_v2 SKU gives us the managed rule sets (OWASP Top 10) for free at the gateway — this is Security pillar coverage before a single line of app code runs."

---

### Step 6 — Show Azure Monitor diagnostic settings

```bash
az monitor log-analytics workspace create -g $RG -n demo-law

WORKSPACE_ID=$(az monitor log-analytics workspace show -g $RG -n demo-law --query id -o tsv)
APPGW_ID=$(az network application-gateway show -g $RG -n demo-appgw --query id -o tsv)

az monitor diagnostic-settings create -g $RG \
  --resource $APPGW_ID \
  --name appgw-diag --workspace $WORKSPACE_ID \
  --logs '[{"category":"ApplicationGatewayAccessLog","enabled":true}]'
```

*Narrate:* "This is the Operational Excellence pillar — if it isn't emitting logs to a workspace, you can't alert on it, and if you can't alert on it, you find out about outages from the client instead of from Azure Monitor."

---

### Step 7 — Wrap the demo

Pull up the Well-Architected Review assessment tool (learn.microsoft.com) and map the resources just built to specific checklist items — this is the bridge into the lab, so the room sees the checklist isn't abstract, it's literally checking off what they just watched you build.

**Closing line:** "Notice I called out the pillar at every single step. That's the habit I want you building in the lab next — not just deploying things, but narrating to yourself why each piece is there."

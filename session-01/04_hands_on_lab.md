# Hands-on Lab — Step-by-Step Tasks (Learner-led, ~50 minutes)

**Shell note:** commands below use PowerShell syntax (backtick `` ` `` line continuation) since that's the common environment for this cohort. If you're on bash/Cloud Shell, replace backticks with `\` at line-ends and `$RANDOM` works natively (no `Get-Random` needed).

---

## Scenario (read this first, keep it visible the whole lab)

"You are the cloud architect for a retail client migrating an internal order-management app to Azure. The client requires: 99.9% availability, PCI-adjacent data handling (no cardholder data, but customer PII), a 3-person ops team, and a cost ceiling of $800/month for this environment. Every decision you make should be justifiable against these four constraints."

---

## Task 1 — Environment Setup (10 min)

```powershell
$RG = "waf-lab-<yourinitials>"
$LOC = "eastus"
az group create -n $RG -l $LOC
az group create -n "$RG-network" -l $LOC   # separate RG for shared network, common enterprise pattern
```

**Pre-flight (run once per subscription, before anything else):**
```powershell
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Sql
az provider register --namespace Microsoft.KeyVault

# check status until each shows "Registered" (takes ~30-60 sec)
az provider show --namespace Microsoft.Insights --query registrationState -o tsv
```

**VNet and four subnets** (use your own address space if sharing a subscription — e.g. `10.20.0.0/16` with an initials-based octet):

```powershell
az network vnet create -g $RG -n lab-vnet --address-prefix 10.20.0.0/16 `
  --subnet-name web-subnet --subnet-prefix 10.20.1.0/24

az network vnet subnet create -g $RG --vnet-name lab-vnet -n app-subnet --address-prefix 10.20.2.0/24
az network vnet subnet create -g $RG --vnet-name lab-vnet -n data-subnet --address-prefix 10.20.3.0/24
az network vnet subnet create -g $RG --vnet-name lab-vnet -n mgmt-subnet --address-prefix 10.20.4.0/24
```

**Deliverable checkpoint:** screenshot the output of:
```powershell
az network vnet show -g $RG -n lab-vnet
```

---

## Task 2 — Security and Governance Controls (10 min)

**Web-subnet NSG (public-facing rule + required GatewayManager rule):**
```powershell
az network nsg create -g $RG -n web-nsg

az network nsg rule create -g $RG --nsg-name web-nsg -n Allow-HTTPS-Inbound `
  --priority 100 --direction Inbound --access Allow --protocol Tcp `
  --destination-port-ranges 443 --source-address-prefixes Internet

az network nsg rule create -g $RG --nsg-name web-nsg -n Allow-GatewayManager-Inbound `
  --priority 110 --direction Inbound --access Allow --protocol Tcp `
  --destination-port-ranges 65200-65535 --source-address-prefixes GatewayManager

az network vnet subnet update -g $RG --vnet-name lab-vnet -n web-subnet --network-security-group web-nsg
```
*Do not skip the GatewayManager rule — Application Gateway will fail to provision or show unhealthy in Task 3 without it.*

**App-subnet NSG (only reachable from web tier):**
```powershell
az network nsg create -g $RG -n app-nsg

az network nsg rule create -g $RG --nsg-name app-nsg -n Allow-Web-Inbound `
  --priority 100 --direction Inbound --access Allow --protocol Tcp `
  --destination-port-ranges 80 443 --source-address-prefixes 10.20.1.0/24

az network vnet subnet update -g $RG --vnet-name lab-vnet -n app-subnet --network-security-group app-nsg
```

**Data-subnet NSG (only reachable from app tier):**
```powershell
az network nsg create -g $RG -n data-nsg

az network nsg rule create -g $RG --nsg-name data-nsg -n Allow-App-SQL-Inbound `
  --priority 100 --direction Inbound --access Allow --protocol Tcp `
  --destination-port-ranges 1433 --source-address-prefixes 10.20.2.0/24

az network vnet subnet update -g $RG --vnet-name lab-vnet -n data-subnet --network-security-group data-nsg
```

**Mgmt-subnet NSG (admin access only):**
```powershell
az network nsg create -g $RG -n mgmt-nsg

az network nsg rule create -g $RG --nsg-name mgmt-nsg -n Allow-Bastion-Inbound `
  --priority 100 --direction Inbound --access Allow --protocol Tcp `
  --destination-port-ranges 3389 22 --source-address-prefixes VirtualNetwork

az network vnet subnet update -g $RG --vnet-name lab-vnet -n mgmt-subnet --network-security-group mgmt-nsg
```

**Governance — enforce required tags (verify before you use any ID — see troubleshooting doc):**
```powershell
$subId = az account show --query id -o tsv

# List and read the exact display name before picking one
az policy definition list --query "[?displayName=='Require a tag on resource groups']" -o json

$policyId = az policy definition list --query "[?displayName=='Require a tag on resource groups'].id" -o tsv
$policyId   # print it — confirm it's not empty before using it

az policy definition show --name $policyId   # confirm it's a real definition

az policy assignment create --name enforce-tags `
  --policy $policyId `
  --scope "/subscriptions/$subId/resourceGroups/$RG" `
  --params '{"tagName":{"value":"environment"}}'
```
*If this keeps failing and is eating time, it's fine to defer it — note it in your review notes as a known gap and move on to Key Vault below.*

**Key Vault in RBAC mode:**
```powershell
$kvName = "lab-kv" + (Get-Random -Maximum 9999)
$kvName   # confirm the generated name before using it

az keyvault create -g $RG -n $kvName -l $LOC --enable-rbac-authorization true
az keyvault secret set --vault-name $kvName --name dummy-secret --value "test-value-123"
```

**Documentation task:** write 2–3 sentences in your review notes justifying deny-by-default over allow-by-default for this client's data sensitivity profile (customer PII, PCI-adjacent — no card numbers held, but a breach would still be reputationally/legally serious).

**Deliverable checkpoints:** `az network nsg list -g $RG -o table` (all four NSGs), policy assignment confirmation (or documented deferral), Key Vault secret set confirmation.

---

## Task 3 — Compute and Load Distribution (12 min)

**VM Scale Set:**
```powershell
az vmss create -g $RG -n lab-vmss `
  --image Ubuntu2204 `
  --vm-sku Standard_B2s `
  --instance-count 2 `
  --vnet-name lab-vnet --subnet web-subnet `
  --admin-username azureuser `
  --generate-ssh-keys `
  --upgrade-policy-mode automatic
```

**Autoscale rule:**
```powershell
az monitor autoscale create -g $RG `
  --resource lab-vmss --resource-type Microsoft.Compute/virtualMachineScaleSets `
  --name lab-autoscale --min-count 2 --max-count 4 --count 2

az monitor autoscale rule create -g $RG --autoscale-name lab-autoscale `
  --condition "Percentage CPU > 70 avg 5m" `
  --scale out 1
```

**Application Gateway (WAF_v2) in front:**
```powershell
az network public-ip create -g $RG -n lab-appgw-pip --sku Standard --allocation-method Static

az network application-gateway create -g $RG -n lab-appgw `
  --sku WAF_v2 --capacity 2 `
  --vnet-name lab-vnet --subnet web-subnet `
  --public-ip-address lab-appgw-pip `
  --priority 100
```

**Validate:**
```powershell
az network application-gateway show-backend-health -g $RG -n lab-appgw
```
*If any backend shows unhealthy, check the GatewayManager NSG rule from Task 2 first — that's the most likely cause.*

**Deliverable checkpoint:** screenshot showing backend pool members as healthy.

**Trade-off decision point (write in review notes before Task 4):** Given this client (6-person ops team, no Kubernetes experience) — would AKS have been a better fit than VMSS? 2–3 sentences, using the reasoning from `02_reference_architecture.md`.

---

## Task 4 — Data Tier and Private Connectivity (10 min)

**Azure SQL Database:**
```powershell
$sqlAdminPassword = "YourStrongP@ssw0rd123!"   # use a real strong password

az sql server create -g $RG -n "lab-sqlsrv-$kvName" `
  -l $LOC --admin-user sqladmin --admin-password $sqlAdminPassword

az sql db create -g $RG -s "lab-sqlsrv-$kvName" -n lab-orderdb --service-objective S0
```

**Disable public access, create private endpoint:**
```powershell
az sql server update -g $RG -n "lab-sqlsrv-$kvName" --set publicNetworkAccess="Disabled"

$sqlServerId = az sql server show -g $RG -n "lab-sqlsrv-$kvName" --query id -o tsv

az network private-endpoint create -g $RG -n sql-pe `
  --vnet-name lab-vnet --subnet data-subnet `
  --private-connection-resource-id $sqlServerId `
  --group-id sqlServer `
  --connection-name sql-pe-connection
```

**Private DNS Zone — do not skip this step:**
```powershell
az network private-dns zone create -g $RG -n privatelink.database.windows.net

az network private-dns link vnet create -g $RG `
  -n lab-dns-link -z privatelink.database.windows.net `
  -v lab-vnet -e false

az network private-endpoint dns-zone-group create -g $RG `
  --endpoint-name sql-pe -n sql-dns-zone-group `
  --private-dns-zone privatelink.database.windows.net `
  --zone-name sql
```

**Validate — from inside the VNet (a VM in mgmt-subnet), not your laptop:**
```powershell
nslookup "lab-sqlsrv-$kvName.database.windows.net"
```
Expect a private IP in `10.20.3.0/24` — **not** a public IP. If public, the DNS zone link didn't take, or you're not testing from inside the VNet.

**Deliverable checkpoint (the most important screenshot in the whole lab):** the `nslookup` output showing the private IP.

---

## Task 5 — Observability (5 min)

```powershell
az monitor log-analytics workspace create -g $RG -n lab-law

$workspaceId = az monitor log-analytics workspace show -g $RG -n lab-law --query id -o tsv
$vmssId = az vmss show -g $RG -n lab-vmss --query id -o tsv
$appgwId = az network application-gateway show -g $RG -n lab-appgw --query id -o tsv
$sqlDbId = az sql db show -g $RG -s "lab-sqlsrv-$kvName" -n lab-orderdb --query id -o tsv

az monitor diagnostic-settings create -g $RG --resource $appgwId `
  --name appgw-diag --workspace $workspaceId `
  --logs '[{\"category\":\"ApplicationGatewayAccessLog\",\"enabled\":true}]'

az monitor diagnostic-settings create -g $RG --resource $sqlDbId `
  --name sqldb-diag --workspace $workspaceId `
  --logs '[{\"category\":\"SQLInsights\",\"enabled\":true}]'
```

**Alert rule (Application Gateway 5xx rate):**
```powershell
az monitor action-group create -g $RG -n lab-ops-team --short-name labops `
  --action email opsadmin your-email@example.com

az monitor metrics alert create -g $RG -n appgw-5xx-alert `
  --scopes $appgwId `
  --condition "count Http5xx > 5" `
  --window-size 5m --evaluation-frequency 1m `
  --action lab-ops-team
```

---

## Task 6 — Architecture Diagram and Review Notes (3 min to finalize, ongoing throughout)

- Update/finalize the architecture diagram (draw.io, Excalidraw, or hand-drawn + photo) reflecting exactly what was deployed — subnet CIDRs, resource names, data flow direction.
- Complete the Review Notes template (see `05_review_deliverables.md`) covering all five WAF pillars against what was **actually built**, not the theoretical reference architecture.

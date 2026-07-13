# Troubleshooting Quick Reference

Check this file first whenever a command errors during the demo or lab — these are real issues hit during dry-run, in order of how they tend to surface.

---

## 1. PowerShell: angle-bracket placeholders break everything

**Symptom:**
```
The '<' operator is reserved for future use.
```
or
```
The system cannot find the file specified.
```

**Cause:** Any `<placeholder>` written in documentation means "put a real value here" — it is never meant to be typed literally. PowerShell treats `<` and `>` as file-redirection operators. Quoted or unquoted, literal angle brackets will break the command.

**Fix:** Always resolve the placeholder into a variable first, then reference the variable — never type angle brackets on the command line.
```powershell
$subId = az account show --query id -o tsv
$policyId = az policy definition list --query "[?displayName=='Require a tag on resource groups'].id" -o tsv
```

---

## 2. Missing GatewayManager NSG rule → Application Gateway fails/unhealthy

**Symptom:** Application Gateway v2 (WAF_v2/Standard_v2) fails to provision, or `show-backend-health` reports unhealthy, even though the app/VMSS itself is fine.

**Cause:** Application Gateway v2 requires inbound access on ports **65200–65535** from the **GatewayManager** service tag, for Azure's own management plane. NSGs have an implicit deny-all as the lowest-priority rule — if you only add a 443 allow rule, GatewayManager traffic gets silently blocked.

**Fix:** Always add both rules to the web-subnet NSG:
```powershell
az network nsg rule create -g $RG --nsg-name web-nsg -n Allow-HTTPS-Inbound `
  --priority 100 --direction Inbound --access Allow --protocol Tcp `
  --destination-port-ranges 443 --source-address-prefixes Internet

az network nsg rule create -g $RG --nsg-name web-nsg -n Allow-GatewayManager-Inbound `
  --priority 110 --direction Inbound --access Allow --protocol Tcp `
  --destination-port-ranges 65200-65535 --source-address-prefixes GatewayManager
```

---

## 3. `InvalidAuthenticationToken` / "please register the subscription"

**Symptom:**
```
(InvalidAuthenticationToken) Please register the subscription '<id>' with Microsoft.Insights.
```

**Cause:** The resource provider for that service (Microsoft.Insights, Microsoft.Sql, Microsoft.KeyVault, etc.) has never been registered on this subscription. Common on new or tightly-locked-down subscriptions.

**Fix:** Register up front, before starting any task:
```powershell
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Sql
az provider register --namespace Microsoft.KeyVault

# poll until "Registered" — takes ~30-60 sec
az provider show --namespace Microsoft.Insights --query registrationState -o tsv
```

---

## 4. `SecurityRuleInvalidAddressPrefix` with a stray character in the value

**Symptom:**
```
(SecurityRuleInvalidAddressPrefix) ... Value provided: GatewayManager\
```

**Cause:** A copy-paste artifact mixing bash's `\` line-continuation with PowerShell's `` ` `` — a stray backslash ends up attached to the literal value instead of being a line-continuation character.

**Fix:** Retype the last line of a multi-line command manually rather than pasting across shells; the final line of a command should have no trailing continuation character at all. Never mix `\` and `` ` `` in the same command block.

---

## 5. `az policy assignment create` fails or grabs the wrong definition

**Symptom:**
```
(PolicyDefinitionNotFound) The policy definition '<id>' could not be found.
'NoneType' object has no attribute 'get'
```

**Cause:** A `contains()` or exact-match query on `displayName` returned an empty or wrong result, and the ID was used without being checked first.

**Fix — verify before you use, every time:**
```powershell
# 1. List and actually read the candidates
az policy definition list --query "[?contains(displayName, 'tag')].{name:displayName, id:id}" -o table

# 2. Get the exact match and print it
$policyId = az policy definition list --query "[?displayName=='Require a tag on resource groups'].id" -o tsv
$policyId

# 3. Confirm it resolves to a real definition before assigning
az policy definition show --name $policyId
```
**If this keeps failing during a live session:** it's fine to defer it. Note it in the review notes as a known gap ("tag policy assignment attempted, deferred due to definition lookup issue") and move on — don't let one governance step block the rest of the lab's timing.

---

## 6. `$RANDOM` doesn't work in PowerShell

**Symptom:** A resource name like `lab-kv$RANDOM` either errors or creates a vault named literally `lab-kv` with nothing appended, causing "name already exists" conflicts.

**Cause:** `$RANDOM` is a bash-only built-in. PowerShell has no equivalent variable by that name.

**Fix:**
```powershell
$kvName = "lab-kv" + (Get-Random -Maximum 9999)
$kvName   # print and confirm before using
az keyvault create -g $RG -n $kvName -l $LOC --enable-rbac-authorization true
```

---

## 7. JSON in `--logs` / `--params` arguments fails to parse in PowerShell

**Symptom:** A `--logs '[{"category":"...","enabled":true}]'` argument that works fine in bash throws a parsing error in PowerShell.

**Cause:** PowerShell's own quote-handling can interfere with single-quoted JSON containing double quotes.

**Fix:** Escape the inner double quotes with a backslash:
```powershell
--logs '[{\"category\":\"ApplicationGatewayAccessLog\",\"enabled\":true}]'
```
or use a PowerShell here-string if the JSON gets more complex.

---

## 8. Common architecture-level failure points (conceptual, not command errors)

| Failure | Likely cause | How to validate |
|---|---|---|
| NSG seems to block traffic that should be allowed | Rule priority ordering — a lower-priority-number deny beats a higher-priority-number allow | `az network nsg rule list -g $RG --nsg-name <nsg> -o table`, check priority column |
| Application Gateway shows healthy instances as unhealthy | Health probe checking wrong path/port | Check probe config against the app's actual health endpoint |
| SQL private endpoint "works" but DNS resolves to a public IP | Private DNS Zone not linked to the VNet | `nslookup` from inside the VNet, confirm private IP range |
| VMSS instance stuck / never joins backend pool | Custom script extension failed during provisioning | `az vmss list-instances -g $RG -n <vmss> -o table`, check provisioning state |

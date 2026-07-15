# Lab 2 — RBAC Role Assignments

**Time:** ~12 minutes | **Focus:** the four RBAC building blocks in practice — built-in roles, a custom role, and scoped assignments, GUI and CLI.

**Depends on:** the four groups created in Lab 1.

---

## Task 2.1 — Explore Built-in Role Definitions

**GUI:** Resource group (`waf-lab-<yourinitials>`) → **Access control (IAM)** → **Roles** tab. Search for `Reader`, `Contributor`, `Virtual Machine Contributor`, `SQL DB Contributor`. Click into each and read the **Actions**/**NotActions** JSON, not just the name.

**CLI:**
```bash
az role definition list --query "[?roleName=='Reader' || roleName=='Contributor' || roleName=='Virtual Machine Contributor'].{Role:roleName, Description:description}" -o table
```

**Deliverable checkpoint:** one sentence per role — what it actually grants, based on reading the Actions, not assuming from the name.

---

## Task 2.2 — Assign a Built-in Role: GUI First

**Scenario:** the Security Auditor group needs Reader access across the whole resource group.

1. Resource group → **Access control (IAM)** → **+ Add** → **Add role assignment**.
2. Select **Reader** → Next.
3. **Assign access to:** User, group, or service principal → **+ Select members** → search `retailco-security-audit` → select it.
4. **Review + assign** (twice).
5. Verify: back on the **Role assignments** tab, confirm the group appears with Reader at this resource group's scope.

**Notice while doing this:** the scope was implicit — determined by which blade you were in when you clicked Add.

---

## Task 2.3 — Assign the Rest via CLI (Explicit Scope)

```bash
rgId=$(az group show -n $RG --query id -o tsv)
vmssId=$(az vmss show -g $RG -n lab-vmss --query id -o tsv)

devGroupId=$(az ad group show -g "retailco-app-developers" --query id -o tsv)

# Developers: restart + monitor VMSS, scoped to the VMSS resource only — not the whole resource group
az role assignment create --assignee-object-id $devGroupId --assignee-principal-type Group \
  --role "Virtual Machine Contributor" --scope $vmssId
```

**Verify:**
```bash
az role assignment list --resource-group $RG --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" -o table
```

**Deliverable checkpoint:** the verification output showing both assignments (Reader at RG scope, VM Contributor at VMSS scope) with visibly different scope strings.

---

## Task 2.4 — Build a Custom Role

**Scenario:** the Ops/Support Engineer needs restart-only access — no built-in role fits that narrowly.

```bash
cat << 'EOF' > vmss-restart-only-role.json
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
    "/subscriptions/SUBSCRIPTION_ID_HERE/resourceGroups/RESOURCE_GROUP_HERE"
  ]
}
EOF

sed -i "s/SUBSCRIPTION_ID_HERE/$subId/" vmss-restart-only-role.json
sed -i "s/RESOURCE_GROUP_HERE/$RG/" vmss-restart-only-role.json
cat vmss-restart-only-role.json   # confirm no placeholders remain

az role definition create --role-definition vmss-restart-only-role.json
```

**Assign it:**
```bash
opsGroupId=$(az ad group show -g "retailco-ops-support" --query id -o tsv)
customRoleId=$(az role definition list --custom-role-only true --query "[?roleName=='VMSS Restart Operator'].id" -o tsv)

az role assignment create --assignee-object-id $opsGroupId --assignee-principal-type Group \
  --role $customRoleId --scope $vmssId
```

**Deliverable checkpoint:** the custom role JSON file, plus `az role assignment list` output showing it assigned to the Ops/Support group.

---

## Task 2.5 — Validate with Check Access

**GUI:** Resource group → Access control (IAM) → **Check access** tab → search a member of `retailco-ops-support` → confirm it shows **VMSS Restart Operator** only, not Contributor or Owner.

**CLI:**
```bash
az role assignment list --assignee $opsGroupId --all -o table
```

**Deliverable checkpoint (most important in this lab):** the Check access screenshot proving least-privilege — exactly the custom role, nothing broader.

---

## What You've Learned

"You've now done all four RBAC building blocks hands-on: explored role definitions (what), created group-scoped assignments both implicitly (GUI) and explicitly (CLI) (who + where), and closed a genuine least-privilege gap with a custom role. Lab 3 adds the piece we skipped: how an *application*, not a person, authenticates — Managed Identities."

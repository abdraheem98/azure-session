# Demo — Step-by-Step Walkthrough (Trainer-led, ~20 minutes)

**Goal:** show the four RBAC building blocks (principal, role definition, scope, assignment) live — first by exploring what already exists, then by creating a real assignment two ways: through the Portal GUI, then through the CLI. Learners watch, then repeat a fuller version in the hands-on project.

**Prerequisite:** the Module 1 lab resource group (`waf-lab-<yourinitials>`) should still exist. If it doesn't, any resource group the learner owns works for this demo.

---

## Part A — Explore Existing RBAC State (5 min)

### Step 1: Who am I? (the "principal")

```bash
az ad signed-in-user show --query "{name:displayName, id:id, upn:userPrincipalName}" -o json
```

*Narrate:* "That `id` field is your object ID — the real identifier RBAC uses behind the scenes. Names and emails are just labels; assignments bind to this ID."

### Step 2: What roles exist? (the "what")

```bash
az role definition list --query "[?roleName=='Reader' || roleName=='Contributor' || roleName=='Owner'].{Role:roleName, Description:description}" -o table
```

*Narrate:* "Three very different blast radii. The description field alone should tell you what each one can and can't do, before you ever assign it to anyone."

### Step 3: What's already assigned, and where? (the "where")

```bash
RG=waf-lab-<yourinitials>
subId=$(az account show --query id -o tsv)

az role assignment list --subscription $subId --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" -o table
az role assignment list --resource-group $RG --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" -o table
```

*Narrate:* "Compare these two outputs. Anything scoped at the subscription level shows up in both — that's inheritance in action. Anything assigned directly at the resource-group level only shows up in the second query. If you see a subscription-wide Owner or Contributor assignment that surprises you, that's exactly the 'scope creep' mistake we talked about in theory."

---

## Part B — Create a Role Assignment: GUI First (7 min)

**Scenario for the demo:** grant a Reader role, scoped to just the resource group — a safe, low-risk assignment to demonstrate with.

1. Navigate to the resource group in the Portal search bar, open it.
2. Left-hand menu → **Access control (IAM)**.
3. **Look at the "Role assignments" tab first** — same data as the CLI query above, different view. Point this out explicitly.
4. Click **+ Add → Add role assignment**.
5. Search for **Reader**, select it, click **Next**. *Narrate: "This is the 'what' — the role definition."*
6. Under "Assign access to," leave **User, group, or service principal**, click **+ Select members**, search for your own account (or a demo test user), select it. *Narrate: "This is the 'who.'"*
7. **Review + assign**, twice (Azure shows a summary screen first).
8. Verify: back on the **Role assignments** tab, confirm the new entry appears.

**Key point to say out loud:** "In the GUI, scope was implicit — determined by which blade you were sitting in when you clicked Add. This is exactly how accidental subscription-wide assignments happen: someone means to be in a resource group's IAM blade but is actually one level up, and doesn't notice."

---

## Part C — Create the Same Assignment via CLI (5 min)

```bash
# Get your own object ID (the "who")
myObjectId=$(az ad signed-in-user show --query id -o tsv)

# Get the resource group's full resource ID (the "where")
rgId=$(az group show -n $RG --query id -o tsv)

# Create the assignment: who + what + where, all explicit
az role assignment create \
  --assignee-object-id $myObjectId \
  --assignee-principal-type User \
  --role "Reader" \
  --scope $rgId
```

**Verify:**
```bash
az role assignment list --resource-group $RG --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" -o table
```

**Closing line for the demo:** "Notice the difference. The GUI made scope implicit — tied to where you clicked. The CLI makes scope explicit — you had to pass `--scope` as an actual parameter, spelled out as a full resource ID. That's exactly why production environments increasingly manage RBAC through IaC or scripted CLI rather than clicking through the Portal: explicit scope in code is reviewable, auditable, and repeatable. Implicit scope from a GUI click is not."

---

## Part D — Quick Look at a Managed Identity (3 min)

**Purpose:** show that the exact same role-assignment mechanism just demonstrated for a human also applies to a machine identity — this is the bridge into Lab 3.

```bash
az vmss identity assign -g $RG -n lab-vmss
vmssIdentityId=$(az vmss identity show -g $RG -n lab-vmss --query principalId -o tsv)
echo $vmssIdentityId
```

*Narrate:* "That's it — the VM Scale Set now has its own identity in Entra ID, with no password anywhere. Watch what happens when I assign it a role — same command shape as Part C, just a different principal type."

```bash
kvId=$(az keyvault show -g $RG -n <your-keyvault-name> --query id -o tsv)
az role assignment create --assignee-object-id $vmssIdentityId --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" --scope $kvId
```

**Bridge into the hands-on labs:** "What you just watched — GUI then CLI for a human role, then the identical pattern for a machine identity — is the exact sequence you'll repeat, in more depth, across Labs 1 through 4."

# Lab 3 — Managed Identities

**Time:** ~10 minutes | **Focus:** giving an Azure resource its own identity, and using RBAC to grant that identity access — no passwords involved.

**Depends on:** the VM Scale Set and Key Vault from Module 1 (`lab-vmss`, and your Key Vault name from Module 1 — referred to below as `$kvName`).

---

## Task 3.1 — Enable a System-Assigned Managed Identity

**Scenario:** in Module 1, the app tier was supposed to read secrets from Key Vault "via managed identity" — but Module 1 never actually wired that up. This lab closes that gap.

**Via GUI:**
1. Navigate to your VM Scale Set (`lab-vmss`) in the Portal.
2. Left-hand menu → **Identity**.
3. Under the **System assigned** tab, toggle **Status** to **On** → **Save**.
4. Once saved, note the **Object (principal) ID** that appears — this is the identity's own object ID in Entra ID, distinct from the VMSS resource's own ID.

**Via CLI (same result, scriptable):**
```bash
az vmss identity assign -g $RG -n lab-vmss
```

**Verify:**
```bash
az vmss identity show -g $RG -n lab-vmss --query "{principalId:principalId, type:type}" -o json
```

**Deliverable checkpoint:** the principal ID from the verify command — you'll use it in the next task.

---

## Task 3.2 — Grant the Managed Identity Access to Key Vault

This is the exact moment "no passwords" becomes real: instead of storing a Key Vault access key or connection string anywhere, the VMSS's identity itself gets an RBAC role on the vault.

```bash
vmssIdentityId=$(az vmss identity show -g $RG -n lab-vmss --query principalId -o tsv)
kvId=$(az keyvault show -g $RG -n $kvName --query id -o tsv)

az role assignment create \
  --assignee-object-id $vmssIdentityId \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $kvId
```

**Why `--assignee-principal-type ServicePrincipal` here, not `User` or `Group`:** a managed identity is backed by a service principal in Entra ID — this is the same "security principal" concept from Lab 2's RBAC building blocks, just a different principal type.

**Verify:**
```bash
az role assignment list --scope $kvId --query "[].{Principal:principalName, Role:roleDefinitionName}" -o table
```

**Deliverable checkpoint:** the verification output showing the VMSS's identity with the Key Vault Secrets User role.

---

## Task 3.3 — Compare System-Assigned vs. User-Assigned

**Do this as a read-through, not a full second implementation, to save time — but understand the distinction concretely.**

```bash
# Create a user-assigned identity as its own standalone resource
az identity create -g $RG -n retailco-shared-identity

# Note: this identity now exists independently of any VM or Scale Set
az identity show -g $RG -n retailco-shared-identity --query "{principalId:principalId, clientId:clientId, id:id}" -o json
```

**Discussion point to write in your notes:** "If we scaled `lab-vmss` from 2 instances to 4, the system-assigned identity is still just one identity shared automatically across all instances of the *same* scale set — that part doesn't change. Where user-assigned matters is when you want the *same* identity attached to multiple *different* resources — for example, a shared identity used by both the VM Scale Set and a separate Function App, so both can be granted the same Key Vault access with a single role assignment instead of two."

**Clean up (optional, since this identity isn't attached to anything in this lab):**
```bash
az identity delete -g $RG -n retailco-shared-identity
```

---

## Task 3.4 — Validate the Full Chain

**GUI:** Key Vault → **Access control (IAM)** → **Check access** → search for the VMSS's identity (search by the VMSS name — system-assigned identities often show up searchable by their parent resource's name) → confirm **Key Vault Secrets User** appears.

**CLI:**
```bash
az role assignment list --assignee $vmssIdentityId --all -o table
```

**Deliverable checkpoint:** confirmation that the chain works end to end — VMSS has an identity, that identity has exactly one narrow role (Secrets User, not Contributor) on exactly one resource (the Key Vault).

---

## What You've Learned

"You've now completed the loop Module 1 left open: an application resource (the VMSS) has its own Entra ID identity, and RBAC grants that identity exactly the access it needs — read secrets, nothing else — with no password ever created, stored, or rotated by a human. Lab 4 combines everything from Labs 1–3 into one full access control design."

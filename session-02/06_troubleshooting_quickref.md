# Troubleshooting Quick Reference

Check this file first whenever a role assignment doesn't behave as expected during the demo or project.

---

## 1. Scope creep — assignment applied more broadly than intended

**Symptom:** a user or group can see/touch resources they shouldn't have access to.

**Cause:** a role was assigned at subscription or management-group scope when resource-group or resource scope was intended. Permissions inherit downward — a subscription-level assignment silently applies to everything underneath it.

**Fix:** always verify the `--scope` value before running `az role assignment create`, and audit existing assignments with:
```bash
az role assignment list --subscription $subId --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" -o table
```
Any row where `Scope` is just `/subscriptions/<id>` (nothing after it) is subscription-wide — review whether that's actually intended.

---

## 2. Group membership change doesn't seem to take effect immediately

**Symptom:** you added someone to a group, assigned the group a role, but the user still can't perform the expected action right after.

**Cause:** Azure AD group membership and role assignment propagation isn't always instant — it can take up to an hour in some cases (especially with nested groups or large tenants).

**Fix:** wait and retest before assuming the assignment failed. Use `az role assignment list --assignee <group-object-id> --all` to confirm the assignment itself exists correctly — if it's there, the issue is propagation delay, not a misconfiguration.

---

## 3. Custom role creation fails with a validation error

**Symptom:**
```
The value for AssignableScopes cannot be empty / invalid format
```
or
```
The specified action ... is not supported
```

**Cause:** malformed `AssignableScopes` (missing leading `/subscriptions/...`, or a placeholder never replaced), or an `Actions` string that doesn't match a real resource provider operation string.

**Fix:**
- Print the JSON file and read it before submitting — confirm no leftover placeholders (`SUBSCRIPTION_ID_HERE`, etc.):
```bash
cat vmss-restart-only-role.json
```
- Look up valid operation strings for a resource provider before hand-writing them:
```bash
az provider operation show --namespace Microsoft.Compute --query "resourceTypes[?name=='virtualMachineScaleSets'].operations[].name" -o table
```

---

## 4. "Access denied" despite a correct-looking role assignment

**Symptom:** the role assignment exists, looks correct in `az role assignment list`, but the user still gets an access-denied error.

**Cause:** this is very often **not** an RBAC problem — a Conditional Access policy (MFA requirement, device compliance, location restriction) is blocking sign-in or session validity, separate from what the role itself would allow.

**Fix:** check Azure AD **Sign-in logs** (Entra ID → Sign-in logs) for the affected user around the time of the failure, not just the role assignment. If the sign-in itself was blocked or required an action the user didn't complete, that's the actual cause, not the RBAC configuration.

---

## 5. A role that should grant access doesn't — deny assignment in play

**Symptom:** a role that normally grants an action doesn't work, on a specific resource, even though the assignment is correct everywhere else.

**Cause:** rare, but Azure itself sometimes applies deny assignments to protect certain managed resources (e.g., resources managed by an Azure Blueprint or a managed application) — deny always overrides allow, regardless of role.

**Fix:**
```bash
az rest --method get --url "https://management.azure.com/{scope}/providers/Microsoft.Authorization/denyAssignments?api-version=2022-04-01"
```
Check whether a deny assignment exists at or above the resource in question.

---

## 6. Cross-shell / placeholder issues (same as Module 1)

If you're working from PowerShell instead of bash, the same issues from Module 1's troubleshooting reference apply here too: never type angle-bracket placeholders literally, `$RANDOM` doesn't exist in PowerShell, and JSON quoting in `--role-definition` or inline JSON arguments may need escaped double quotes. See Module 1's `06_troubleshooting_quickref.md` for the full detail — it's not repeated here to avoid duplication.

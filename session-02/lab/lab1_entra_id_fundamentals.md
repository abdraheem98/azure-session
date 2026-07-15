# Lab 1 — Entra ID Fundamentals

**Time:** ~10 minutes | **Focus:** Entra ID as the identity/directory layer — distinct from RBAC, which comes in Lab 2.

## Objective

Before assigning any roles, get comfortable navigating Entra ID itself: viewing your own identity, exploring users, creating groups, and understanding service principals — the directory objects that RBAC will later attach permissions to.

---

## Task 1.1 — View Your Own Identity (GUI + CLI)

**GUI:** Portal search bar → **Microsoft Entra ID** → **Users** → search for your own account → open your profile. Note the **Object ID** field — this is what everything else in this module will reference you by, not your name or email.

**CLI — same data, scriptable:**
```bash
az ad signed-in-user show --query "{name:displayName, id:id, upn:userPrincipalName}" -o json
```

**Deliverable checkpoint:** screenshot of your Object ID from either view.

---

## Task 1.2 — Explore Directory Objects

**GUI:** In Entra ID, browse each of these blades and note what lives in each — you don't need to create anything yet, just look:
- **Users** — human identities
- **Groups** — collections of users (or other groups) for simplified access management
- **App registrations** — identities for applications, each backed by a **service principal**
- **Enterprise applications** — the service principal side of app registrations, plus third-party SaaS integrations

**CLI — list your tenant's groups (there may be few or none yet):**
```bash
az ad group list --query "[].{Name:displayName, Id:id}" -o table
```

**Key distinction to internalize:** "A user is a person. A service principal is the identity an application uses. A managed identity (covered in Lab 3) is a special kind of service principal that Azure manages for you automatically. All three are 'security principals' as far as RBAC is concerned — RBAC doesn't care whether it's a human or a piece of software asking for access."

---

## Task 1.3 — Create the Groups Used Throughout This Module

These four groups are used in Labs 2 and 4 — create them now.

**Via GUI:**
1. Entra ID → **Groups** → **New group**.
2. Group type: **Security**. Name: `retailco-app-developers`. Add yourself as a member (for testing later).
3. Repeat for `retailco-dba`, `retailco-security-audit`, `retailco-ops-support`.

**Via CLI (do at least one this way):**
```bash
az ad group create --display-name "retailco-app-developers" --mail-nickname "retailco-app-developers"

myObjectId=$(az ad signed-in-user show --query id -o tsv)
groupId=$(az ad group show -g "retailco-app-developers" --query id -o tsv)
az ad group member add --group $groupId --member-id $myObjectId
```

**Verify all four exist:**
```bash
az ad group list --query "[?starts_with(displayName, 'retailco-')].{Name:displayName, Id:id}" -o table
```

**Deliverable checkpoint:** screenshot of all four groups (GUI or CLI output), plus a one-sentence note on why groups are used instead of assigning roles to individuals directly.

---

## What You've Learned

By the end of this lab you should be able to say, in your own words: "Entra ID is the directory that holds who exists — users, groups, and application identities. It answers 'who are you,' not 'what can you do.' That second question is RBAC, which is Lab 2."

# Securing Azure Infrastructure: Cloud Governance and Security Hardening

This lab restricts a test user to view-only access with RBAC, blocks expensive VM sizes with Azure Policy, and sets a budget alert that warns before spend gets out of hand, then maps each control to the NIST Cybersecurity Framework.

## 🎬 Video Walkthrough
<!-- Replace with your Loom link after recording -->
[Loom walkthrough](https://www.loom.com/)

---

## Project Overview

Governance answers three questions in a cloud environment:

1. **Who is allowed to do what?** → Role-Based Access Control (RBAC)
2. **What configurations are allowed?** → Azure Policy
3. **How much can we spend, and who finds out when we get close?** → Cost Management budgets

A useful way to picture it: if you hire ten people, you don't hand every one of them the master key, the safe combination, and authority to sign contracts. RBAC is the key you give each person, Policy is the building code everyone has to follow, and the budget is the alarm that goes off when the electric bill spikes. Where the analogy breaks: a building code only applies when something gets built, and a budget alert only tells you there's a problem. Neither one fixes anything on its own.

Everything in this lab is scoped to a single resource group, so nothing else in the subscription is affected.

**What this lab builds:**

- A test user (Junior Developer) in Microsoft Entra ID
- A Reader role assignment scoped to one resource group
- An Azure Policy assignment that only allows `Standard_B1s` and `Standard_B1ms` VMs
- A $50 monthly budget with an 80% actual alert and a 100% forecasted alert

**Skills demonstrated:**

- Identity management in Microsoft Entra ID
- RBAC and the principle of least privilege, applied at resource group scope
- Azure Policy: definitions, assignments, parameters, scope, and the Deny effect
- Cost Management: budgets, actual vs forecasted alerts
- Verifying controls instead of assuming they work (negative and positive testing)
- Mapping cloud controls to the NIST Cybersecurity Framework

---
## Architecture Diagram

![Architecture diagram](architecture.png)

**How it works:** The Admin creates the Junior Developer in Entra ID, assigns the Reader role on the resource group, applies the VM size policy, and creates the budget. When the Junior Developer tries to create anything, RBAC returns `AuthorizationFailed`. When anyone, including the Admin, tries to deploy a VM size outside the allowed list, the policy blocks it at validation. The budget runs on its own and emails the Admin when a threshold is crossed.


---

## Key Concepts

### RBAC: who can act

RBAC bundles permissions into roles and assigns them to a person at a scope.

| Role | Can do | Cannot do |
|------|--------|-----------|
| Owner | Everything, including granting access | — |
| Contributor | Create, change, delete resources | Grant access to others |
| Reader | View everything in scope | Create, change, or delete anything |

New users start with zero permissions. This lab assigns Reader at the **resource group** scope, so the Junior Developer can see that group and nothing else in the subscription.

### Azure Policy: what configurations are allowed

| Term | Meaning |
|------|---------|
| Definition | The rule ("only these VM sizes") |
| Assignment | Turning the rule on at a scope. An unassigned definition does nothing. |
| Parameter | The values that make the rule specific (B1s, B1ms) |
| Effect | What happens on a violation. **Deny** blocks it, **Audit** logs it, **Modify/Append** changes the request. |

Policy is evaluated when a resource is created or updated. A new assignment can take up to 30 minutes to take effect.

### Budgets: awareness of spend

A budget **does not stop spending**. It doesn't delete, pause, or block anything. It sends an alert when spend crosses a threshold.

- **Actual alert:** money already spent crossed the threshold (80% of $50 = $40)
- **Forecasted alert:** Azure projects you'll cross the threshold by the end of the period based on current spend rate

### NIST CSF mapping

NIST CSF 2.0 (released February 2024) has six functions. Older material often lists five; Govern was added in 2.0, and most of this lab is Govern work.

| Function | Where it shows up in this lab |
|----------|------------------------------|
| **Govern** | Deciding who gets which role, which VM sizes are approved, and what the spend limit is |
| **Identify** | Creating the user in Entra ID before granting anything |
| **Protect** | Reader role assignment and the VM size Deny policy |
| **Detect** | Budget alerts on unexpected spend |
| **Respond** | Acting on the alert email (investigate in Cost Analysis) |
| **Recover** | Not exercised in this lab |

---

## Common Misconceptions

A few points that are often stated incorrectly about these controls:

- **NIST CSF has six functions, not five.** CSF 2.0 added Govern. Since this lab is about roles, rules, and spending limits, Govern is the most relevant function.
- **A Deny policy isn't unbypassable.** It does block an Owner from deploying a non-compliant resource. But an Owner can still edit the assignment, delete it, or create an exemption. The control works because bypassing it takes a deliberate action that shows up in the Activity Log, not because it's impossible.
- **Budget alerts aren't real-time.** Cost data lags by several hours, sometimes up to a day. For a compromised account spinning up VMs, a budget alert is a backstop. Faster signals come from Activity Log alerts and Microsoft Defender for Cloud.
- **"Azure Active Directory" was renamed** Microsoft Entra ID in 2023. Same service.

---

## Prerequisites

- An active Azure subscription where you are **Owner** (needed to assign roles and policies)
- Permission to create users in your Entra ID tenant (you have this by default on a personal subscription)
- An incognito/private browser window, to sign in as the Junior Developer without signing out of your Admin account
- Optional: Azure Cloud Shell (Bash) for the verification commands

## Naming Conventions Used

Replace `yourname` with your first name in lowercase, no spaces (for example, `rg-lab05-alex`). Replace `<yourtenant>` with your Entra ID tenant domain.

| Item | Value |
|------|-------|
| Resource group | `rg-lab05-yourname` |
| Region | East US |
| Test user UPN | `junior-dev-yourname@<yourtenant>.onmicrosoft.com` |
| Test user display name | Junior Developer |
| Policy assignment | `Restrict-VM-Sizes` |
| Allowed VM sizes | `Standard_B1s`, `Standard_B1ms` |
| Budget name | `Monthly-Lab-Budget` |
| Budget amount | $50 |

---

## Screenshot Guidelines

Before committing screenshots, blur or crop:

- Your tenant domain (`<yourtenant>.onmicrosoft.com`)
- Subscription ID and tenant ID
- Your email address in the budget alert recipients
- Object IDs on the user page

Never capture the password field, even if it looks masked.

**Workflow:** save each original to `screenshots/raw/` (git-ignored), blur the sensitive parts, then save the cleaned copy into `screenshots/` with the file name shown in each step. Only the cleaned copies get committed.

---

## Project Steps

### Phase 1: Create the Lab Resource Group

The resource group is the scope for every control in this lab.

1. Sign in at [portal.azure.com](https://portal.azure.com) with your Admin account.
2. Search for **Resource groups** and click **+ Create**.
3. Resource group name: `rg-lab05-yourname`
4. Region: **East US**
5. Click **Review + create**, then **Create**.

**Verify (Cloud Shell):**
```bash
az group show --name rg-lab05-yourname --output table
```
<img width="643" height="230" alt="Screenshot 2026-10-06 150253" src="https://github.com/user-attachments/assets/0b347c59-eaf4-4f08-a6cf-e0dc424521fb" />

---

### Phase 2: Create the User and Assign the Reader Role

#### Step 1: Create the user in Entra ID (Identify)

1. Search for **Microsoft Entra ID** → **Users** → **+ New user** → **Create new user**.
2. User principal name: `junior-dev-yourname`
3. Display name: `Junior Developer`
4. Uncheck **Auto-generate password** and set one you'll remember for Phase 3.
5. Click **Review + create**, then **Create**.

**Verify:**
```bash
az ad user show --id junior-dev-yourname@<yourtenant>.onmicrosoft.com \
  --query "{name:displayName, upn:userPrincipalName}" --output table
```

<img width="660" height="415" alt="Screenshot 2026-10-06 150936" src="https://github.com/user-attachments/assets/b2170f45-f190-401c-8dd4-bd837f2b5537" />


#### Step 2: Assign Reader at the resource group scope (Protect)

1. Open `rg-lab05-yourname` → **Access control (IAM)**.
2. **+ Add** → **Add role assignment**.
3. **Role** tab: search **Reader**, select it, click **Next**.
4. **Members** tab: **+ Select members** → search **Junior Developer** → **Select**.
5. **Review + assign** twice.

**Verify:**
```bash
az role assignment list --resource-group rg-lab05-yourname --output table
```
Look for the Junior Developer with role `Reader` and a scope ending in `/resourceGroups/rg-lab05-yourname`.

<img width="653" height="337" alt="Screenshot 2026-10-06 150902" src="https://github.com/user-attachments/assets/e3a09e36-6470-48ca-8cf7-c04100a51d63" />


### Phase 3: Verify Access (the "Permission Denied" Test)

A control you haven't tested is a control you can't trust.

#### Step 1: Sign in as the Junior Developer

1. Open an incognito window and go to [portal.azure.com](https://portal.azure.com).
2. Sign in as `junior-dev-yourname@<yourtenant>.onmicrosoft.com`. Complete the MFA setup if prompted.
3. Go to **Resource groups**. Only `rg-lab05-yourname` should appear.

<img width="1804" height="542" alt="Screenshot 2026-10-06 at 3 12 45 PM" src="https://github.com/user-attachments/assets/ad89f06f-d0f0-49db-a7a1-65eee2218cc0" />


#### Step 2: Try to create a resource

1. Open the resource group → **+ Create** → **Storage account** → **Create**.
2. Fill in any name and click **Review + create**.
3. Expand the error message.

**Expected:** `AuthorizationFailed` or "You do not have permission to perform this action."

<img width="1916" height="955" alt="Screenshot 2026-10-06 at 3 11 47 PM" src="https://github.com/user-attachments/assets/efd4e3e4-3733-43d5-be72-ccc26cc2af37" />

---

### Phase 4: Assign the VM Size Policy

The scenario: an engineer means to pick `Standard_D2s_v3` (2 cores) and picks `Standard_D32s_v3` (32 cores) instead. Without a policy, that runs until someone sees the bill.

#### Step 1: Create the policy assignment

1. Search **Policy** → **Authoring** → **Assignments** → **Assign policy**.
2. **Basics tab:**
   - **Scope:** click **...**, select your subscription, then `rg-lab05-yourname`. Click **Select**.
   - **Policy definition:** click **...**, search `Allowed virtual machine size SKUs`, select it, click **Add**.
   - **Assignment name:** `Restrict-VM-Sizes`
3. **Parameters tab:**
   - Uncheck **Only show parameters that need input or review**.
   - **Allowed Size SKUs:** select `Standard_B1s` and `Standard_B1ms`.
4. Click **Review + create**, then **Create**.

**Verify:**
```bash
az policy assignment list --resource-group rg-lab05-yourname --output table
```


#### Step 2: Confirm the allowed sizes

1. Click `Restrict-VM-Sizes` → **View assignment**.
2. Scroll to **Parameters**.

> ⏱️ **Wait 15–30 minutes before Phase 5.** New assignments take time to apply. If the test passes when it should fail, wait and retry before troubleshooting.


<img width="942" height="317" alt="Screenshot 2026-10-06 150427" src="https://github.com/user-attachments/assets/a6ca4ea3-179c-48b1-84af-aa900144431a" />

---

### Phase 5: Test the Policy in Both Directions

You're testing that the policy blocks what it should **and** allows what it should. Do this in the portal only. Don't use `az vm create` for the negative test: if the policy hasn't applied yet, it creates a real VM that costs money.

#### Step 1: Try a blocked size

1. Open `rg-lab05-yourname` → **+ Create** → **Virtual machine** → **Create**.
2. VM name: `vm-policy-test`, Image: **Ubuntu Server**.
3. **Size:** **See all sizes** → `Standard_D2s_v3`.
4. Click **Review + create** and expand the error.

**Expected:** "Validation failed" with "Policy check failed" naming `Restrict-VM-Sizes`. In API responses, the error code is `RequestDisallowedByPolicy`.

📸 **Screenshot `08-policy-denied-d2s.png`:** The expanded validation error showing the policy assignment name as the reason.

![Policy denied D2s_v3](screenshots/08-policy-denied-d2s.png)

#### Step 2: Try an allowed size

1. Go back to **Basics** and change the size to `Standard_B1s`.
2. Click **Review + create**.

**Expected:** validation passes. **Don't click Create.** Passing validation is the proof.

📸 **Screenshot `09-policy-allowed-b1s.png`:** The "Validation passed" banner with `Standard_B1s` visible in the summary.

![Policy allowed B1s](screenshots/09-policy-allowed-b1s.png)

---

### Phase 6: Create the Budget and Alerts

#### Step 1: Create the budget with alerts

1. Open `rg-lab05-yourname` → **Cost Management** → **Budgets** → **+ Add**.
2. **Create budget tab:**
   - Name: `Monthly-Lab-Budget`
   - Reset period: **Billing month**
   - Expiration date: one year from today
   - Amount: **50**
3. Click **Next**.
4. **Alert conditions tab:** add both conditions (use **+ Add alert condition** for the second).

   | Alert | Type | % of budget | Fires when |
   |-------|------|-------------|-----------|
   | 1 | Actual | 80 | $40 has already been spent this month |
   | 2 | Forecasted | 100 | Azure projects spend will pass $50 by month end |

5. **Alert recipients:** your email address.
6. Click **Create**.

**Verify:**
```bash
az consumption budget list --resource-group rg-lab05-yourname --output table
```

#### Step 2: Confirm the alert conditions

1. Click `Monthly-Lab-Budget` → **Edit budget** → **Alert conditions**.
2. Check both thresholds and the recipient are saved. Close without changing anything.

<img width="679" height="458" alt="Screenshot 2026-10-06 150558" src="https://github.com/user-attachments/assets/1d3aa482-cd8e-457a-908b-207e7b11e183" />


---

## Results

<!-- Fill this in after you run the lab. Only mark what you actually observed. -->

| Test | Expected | Observed |
|------|----------|----------|
| Junior Developer sees only the lab resource group | Only `rg-lab05-yourname` listed | ☐ |
| Junior Developer creates a Storage Account | `AuthorizationFailed` | ☐ |
| VM with `Standard_D2s_v3` | Policy check failed: `Restrict-VM-Sizes` | ☐ |
| VM with `Standard_B1s` | Validation passed | ☐ |
| Budget created | $50, billing month reset | ☐ |
| Alerts configured | 80% Actual, 100% Forecasted | ☐ |
| Clean-up | Resource group and user gone | ☐ |

**How the three controls cover different failures:**

| Scenario | Control that catches it |
|----------|------------------------|
| Junior Developer tries to delete a resource | RBAC: Reader has no write permissions |
| Engineer picks a 32-core VM by mistake | Policy: blocked at validation, before any cost |
| Load balancer left running over a weekend | Budget: alert when spend crosses 80% |
| Compromised account deploys VMs for crypto mining | Policy blocks large sizes; if attacker uses allowed sizes, the budget alert eventually fires (with a delay) |

---

## Security Decisions

- **Reader at resource group scope, not subscription.** Least privilege applies to scope as well as role. The Junior Developer can't see anything outside the lab.
- **Deny instead of Audit.** Audit would log the expensive VM and let it run. For cost control, the mistake needs to be stopped before it bills.
- **Two alert types.** Actual gives ground truth; Forecasted gives time to act before the limit.
- **Testing both directions.** A policy that blocks everything also "passes" the negative test. The B1s test proves the policy is selective.
- **Deleting the test user.** Unused accounts are attack surface, even in a lab tenant.

---

## Troubleshooting / Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Junior Developer can still create resources | Role assigned at the wrong scope, or a different role was picked | IAM → Role assignments on the resource group. Confirm Reader is listed there. Check the subscription IAM for any extra assignment. |
| Junior Developer sees no resource groups | Role assignment not propagated yet | Wait 5 minutes, sign out of the incognito window, and sign back in |
| D2s_v3 passes validation | Policy hasn't applied yet | Wait 15–30 minutes. Confirm the assignment scope is the lab resource group. |
| Policy definition not found | Search term doesn't match | Search `virtual machine size` and pick the one about allowed SKUs |
| Budgets not visible in the resource group menu | Some subscription offer types hide it there | Search **Cost Management** → **Budgets** and set the scope to the resource group |
| Budget creation fails with a permissions error | Account lacks Cost Management write access | Confirm you have Owner or Cost Management Contributor on the subscription |
| MFA prompt on first sign-in | Security defaults are on in the tenant | Complete setup with an authenticator app. This is expected behavior. |
| `az ad user show` returns nothing | Wrong tenant domain in the UPN | Run `az ad signed-in-user show --query userPrincipalName` to see your domain |

---

## Cleanup

1. **Delete the resource group.** `rg-lab05-yourname` → **Overview** → **Delete resource group**. Type the name to confirm. This removes the policy assignment scoped to it.
2. **Delete the test user.** Entra ID → Users → `junior-dev-yourname` → **Delete**.
3. **Confirm the budget is gone.** Search **Cost Management** → **Budgets**. If `Monthly-Lab-Budget` is still listed, delete it.

Or from Cloud Shell:
```bash
az group delete --name rg-lab05-yourname --yes
az ad user delete --id junior-dev-yourname@<yourtenant>.onmicrosoft.com

# Both should confirm the resources are gone
az group exists --name rg-lab05-yourname        # returns false
az ad user show --id junior-dev-yourname@<yourtenant>.onmicrosoft.com   # returns an error
```

📸 **Screenshot `12-cleanup-verified.png`:** Cloud Shell output showing `false` from `az group exists`, or the Resource groups list without the lab group.

![Cleanup verified](screenshots/12-cleanup-verified.png)

---

## Key Takeaways

- **RBAC controls people; Policy controls configurations.** Even an Owner gets blocked by a Deny policy, though an Owner can change the policy itself.
- **Scope matters as much as the role.** Reader on the subscription and Reader on one resource group are very different levels of access.
- **A budget is a smoke detector, not a sprinkler.** It tells you about spend; it doesn't stop it. Automatic response needs an Action Group wired to a function or runbook.
- **Test controls in both directions.** Blocking the bad case isn't enough. You also need proof the good case still works.
- **Governance maps directly to NIST CSF.** Govern sets the rules, Identify and Protect enforce them, Detect and Respond catch what slips through.

---



---

**Author:** Manuel Yannick Armah

**Project:** Securing Azure Infrastructure: Cloud Governance and Security Hardening

**Difficulty:** Beginner

**Time to Complete:** 60–75 minutes (plus 15–30 minutes waiting for policy to apply)

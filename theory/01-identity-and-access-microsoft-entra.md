today 31 july 2026. First Module : 

# Manage and implement authentication methods in Microsoft Entra ID

<img width="577" height="171" alt="image" src="https://github.com/user-attachments/assets/ccc77e3c-8065-4a94-9563-324505d2ea0f" />


































































Standing privilege
Blast radius

The blast radius extends further when you factor in the control plane

Just-in-time (JIT) access as the response to standing privilege
Just-in-time (JIT) access provides elevated permissions that you don't hold persistently. You activate them for a specific task, and they expire after a defined period. JIT access reduces the risk that standing privilege creates through three properties.

You don't hold access by default. Without a standing role assignment, there are no persistent credentials for an attacker to discover or steal between sessions.

You activate access intentionally. You must take a deliberate action to elevate your permissions. Every use of privilege becomes an explicit, auditable event rather than an ambient condition.

Access expires automatically. Even if a session is compromised, the attacker's window closes when it ends—not when you happen to notice a problem.

Microsoft Entra Privileged Identity Management (PIM) is the service that implements just-in-time access for both Microsoft Entra ID roles and Azure resource roles.

Activation controls
When you activate an eligible assignment, PIM can require you to satisfy one or more conditions before granting the role. These controls create a checkpoint between the entitlement and the live session, raising the cost of unauthorized use and ensuring that a stolen credential alone isn't enough to obtain elevated permissions.

Multifactor authentication (MFA) verification—confirms your identity before elevation, so stolen credentials alone aren't enough to gain the role.
Justification—written rationale you must supply before PIM grants the session, creating an auditable record of why access was requested.
Approval—a delegated approver must confirm the request before PIM activates the role, adding a human checkpoint for sensitive assignments.
Activation duration—the configured maximum time window, measured in hours (from 1 to 24 hours), after which PIM automatically removes the role, limiting the exposure window for any single session.





<img width="563" height="900" alt="image" src="https://github.com/user-attachments/assets/d7ec996b-46e6-45eb-90ed-ff83653cccb2" />
Microsoft Entra Privileged Identity Management (PIM) implements just-in-time access through a specific set of controls.

eligible assignment

 An active assignment grants the role directly, making access immediate without any activation step. PIM can time-bound active assignments, making them suitable for temporary standing access with a defined expiration.

 Every activation generates an audit log entry recording who activated the role, which role was activated, when the session started, how long it lasted, and the justification the user provided. These entries accumulate into a durable record of all privileged activity across your tenant.



PIM also supports access reviews, a periodic process to validate that eligible assignments are still appropriate. At regular intervals, designated reviewers examine each eligible assignment and confirm or deny whether it should continue.

 # Implement just-in-time access for Microsoft Entra roles


### PIM requires a Microsoft Entra ID P2 or Microsoft Entra ID Governance license for each user whose access it manages.



Global Administrator	
Privileged Role Administrator	
Security Administrator	
Exchange Administrator	
Application Administrator
Authentication Policy Administrator	


Assign eligible access to a Microsoft Entra role
When you assign a user to a role as eligible, you convert them from always holding the role to holding only the entitlement. The role grants no active permissions until the user initiates an activation. The procedure makes that conversion for any Microsoft Entra ID role in PIM.

1.Sign in to the Microsoft Entra admin center and open ID Governance > Privileged Identity Management.

2.Under Manage, select Microsoft Entra roles.

3.Select the target role—for example, Exchange Administrator.

4.Select Assignments > Add assignments.

5.Set Assignment type to Eligible.

6.Select the member and set the assignment duration (permanent or time-bound).

7.Select Assign to complete the assignment.

This change converts standing access to a JIT entitlement—the role now appears in the user's eligible assignments but grants no active privileges until they complete an activation.


<img width="1036" height="222" alt="image" src="https://github.com/user-attachments/assets/d60916a7-f916-4ae5-8258-caac550194a6" />

Activate an eligible Microsoft Entra role
From your perspective, activation is the step that converts an eligible assignment into a live, time-limited session. That session grants the role's full permissions for the duration you specify, capped by the maximum set in role settings.

1.In PIM, select My roles > Eligible assignments.

2.Find the role and select Activate.

3.Set the activation duration (within the maximum configured in role settings).

4.Enter a justification describing the task requiring elevated access.

5.If approval is required, submit the request and wait for approver action.

6.Confirm the role appears under Active assignments with a duration countdown.



 ### Role settings
**Role Settings are the per-role configuration in PIM that governs the conditions any eligible user must satisfy before PIM grants access to that role. For example, settings determine whether MFA verification is enforced, whether the user must enter a justification, whether an approver must authorize the request, and how long an activation can last.**

> <img width="573" height="381" alt="image" src="https://github.com/user-attachments/assets/794d4331-0dcc-482a-bbd3-8b2989c187a8" />

<img width="865" height="256" alt="image" src="https://github.com/user-attachments/assets/bb7f05ab-0089-4ea9-85fc-6c1e1f69d3b4" />

>To reach the settings for a specific role, navigate to PIM > Microsoft Entra roles, select the role, select Role settings, and select Edit.
---
### Understand the Azure resource scope in Privileged Identity Management (PIM)
Azure RBAC operates across a nested scope hierarchy: management group, subscription, resource group, and individual resource. Assignments at a broader scope inherit downward. An Owner assignment at the subscription level grants ownership over every resource group and resource within it. PIM surfaces this same hierarchy, letting you make a user eligible at the subscription, resource group, or individual resource level.

 ID Governance > Privileged Identity Management > Azure resources. Here, you're managing RBAC assignments on Azure control-plane objects

 **The updated PIM experience uses the latest Azure Resource Manager API and automatically surfaces Azure resources in your tenant—no manual onboarding step is required. The table summarizes the key operational differences between the two planes.**

<img width="319" height="304" alt="image" src="https://github.com/user-attachments/assets/7d8cf665-bdd2-4def-a401-6397773e5a39" />
Not every Azure resource carries the same risk from a compromised Owner or Contributor. The right activation controls depend on four factors: data sensitivity, blast radius, regulatory exposure, and the reversibility of any damage. A misconfigured dev/test resource group is annoying and recoverable. 



### Risk tier Resource examples	Risk reason	Recommended activation controls

Critical	
---
>Production subscription, Key Vault, Azure AI services	Compromise grants broad lateral movement; Key Vault secrets unlock downstream systems; AI model weights and training data represent exfiltratable IP	multifactor authentication (MFA) required, justification required, approval required, 1–2 hour max
---
High
>Production resource group, Azure SQL, Azure Kubernetes Service	Narrower scope but high data sensitivity or service continuity risk	MFA required, justification required, approval optional, 4–8 hour max
---
Standard	
>Dev/test resource groups, sandbox subscriptions	Low data sensitivity; mistakes are reversible	MFA required, justification required, no approval, up to 8 hours

To activate an eligible Azure resource role, navigate to My roles in PIM, select the Azure resources tab to see your eligible assignments, and select Activate—the activation steps follow the same flow as Microsoft Entra roles.

**One important nuance: you configure role settings for Azure resource roles independently per role per resource scope. The same Contributor role can have different activation controls at the subscription level versus the resource-group level. This flexibility lets you apply stricter requirements—shorter duration, mandatory approval—on production scopes while keeping lower-friction settings for development scopes.**

How PIM work for groups? 


Privileged Identity Management (PIM) for Groups shifts the eligible resource from a role to a group membership. Instead of making each engineer eligible for the Key Vault Contributor role directly, you create a group, assign that group the Key Vault Contributor role, and make each engineer eligible for membership in that group.

For groups that govern access to sensitive resources, using a role-assignable group is a security best practice. A standard Microsoft Entra ID group is modified by its owners, or by lower-privileged administrators such as a Groups Administrator, without triggering an approval or audit entry in PIM. A role-assignable group raises the privilege bar—membership changes require at least a Privileged Role Administrator—reducing the risk that someone bypasses your PIM workflow by modifying the group directly.

Within PIM for Groups, you make a principal eligible as a Member—granting them the group's assigned roles—or as an Owner, which confers control over the group itself. Most JIT scenarios use Member eligibility.


Why group-based JIT enforces policy more consistently ? 

here is why:


<img width="820" height="381" alt="image" src="https://github.com/user-attachments/assets/ac1f70b5-70af-4578-a4d7-b4dd557fcb5c" />

### When to choose group-based JIT
Group-based just-in-time access is the right model when two or more of these conditions are true: your team has more than a few users who need the same access pattern, that access recurs regularly, or the required permissions span multiple roles or resources that logically belong together.







### Applying JIT access to AI workloads, agents, and applications
The group-based model you built in the previous unit scales directly to the engineers who manage your AI services. Create a role-assignable group, assign it the AI control-plane roles your platform team needs, and make each engineer eligible for group membership—the same pattern, the same policy control, the same audit trail. That part transfers without modification. The complication surfaces when you look past the humans configuring those services to the identities that run them. Azure OpenAI and Azure Machine Learning aren't accessed only by engineers working through activation workflows. They're also accessed by applications, automation pipelines, and AI agents executing inference at machine speed, none of which can complete an MFA prompt or wait for an approver. That difference in identity type produces two distinct access tracks, and understanding where each applies is what this unit builds toward. 
---
### Applying PIM and groups to AI control-plane roles
The fix is the same one that works for Key Vault and production resource groups. Assign the Cognitive Services OpenAI Contributor role and any Azure Machine Learning workspace roles your platform team requires to a role-assignable group, and make each engineer eligible for membership through Privileged Identity Management (PIM) for Groups.

Engineers who need eligible access to deploy models to endpoints, modify model configurations, initiate fine-tuning runs, or access training datasets directly. The list is typically a small, well-defined set of people. Eligible membership through one role-assignable group enforces one activation policy—one maximum duration, one approvers list, one audit log—across all of them. Adding a new engineer means adding eligible membership to the group. There are no new activation settings to configure, no separate approvers list to maintain, and no new configuration surface to drift out of compliance 

The principal boundary—humans versus workload identities
Here, the architecture diverges from everything discussed so far. A human engineer activating an eligible role works through an interactive session—MFA, justification, and approval if policy requires it. PIM can gate that access because there's a session to gate.

A managed identity—the Microsoft Entra ID identity type assigned directly to Azure resources such as container apps, virtual machines, and functions—authenticates non-interactively. It acquires an OAuth 2.0 access token by proving its own identity to Microsoft Entra ID using a platform-managed credential. There's no sign in prompt, no session, no MFA challenge, and no point where an approval workflow can pause execution.

Requiring activation isn't a policy gap—it's a mechanism constraint. PIM requires interactive activation because activation is what temporarily elevates an eligible assignment to an active one. A managed identity can't complete interactive activation, so PIM can't govern its access. There's no PIM configuration that makes a managed identity eligible; that workflow doesn't exist in the platform.



The right model for workload identities is permanent role-based access control (RBAC) assignment, scoped as narrowly as possible to the specific resource and operation, with least privilege enforced by scope rather than time.
<img width="476" height="620" alt="image" src="https://github.com/user-attachments/assets/f4c0211d-dbea-4d65-835c-ecf9b2fa3240" />



### Choosing the right access model
Consider this scenario: your AI pipeline agent needs to write inference outputs to Azure Blob Storage. Should its identity be eligible for the Storage Blob Data Contributor role in PIM?

No. The managed identity can't complete PIM's interactive activation flow. Assign it a permanent Storage Blob Data Contributor role scoped to the specific container it writes to. PIM governs the human engineer who configures that RBAC assignment—who holds an eligible role that permits modifying access on that container—not the managed identity performing the writes.

**The clean two-track rule: PIM governs the humans who configure, deploy, and manage AI services. RBAC governs the workload identities that run them.**

The two-track model is clear in isolation, but applying it consistently across an organization—with mixed teams, shared AI services, and evolving workload patterns—requires explicit design decisions. Unit 8 examines the principles that connect all the patterns covered in this module into a coherent privileged access strategy.


### JIT design patterns and best practices
Knowing how each control works isn't the same as knowing which ones to reach for. Across Microsoft Entra roles, Azure resource roles, groups, and AI workloads, the same underlying question recurs: what level of friction, scope, and duration is right for this workload, at this risk level, for this team?

---

###The break-glass exception
Break-glass accounts are excluded from JIT not because they're exempt from governance, but because PIM's activation path—which depends on multifactor authentication (MFA), approver availability, and service health—can itself fail during the outage conditions that require emergency access. When the identity platform is partially degraded, the last thing you need is access gated behind a service that isn't responding. The governance model shifts accordingly: instead of time-limited activation, the controls become zero normal use, maximum alerting, and a documented break-glass procedure that is rehearsed and audited on a fixed schedule.

**Break-glass accounts must never be used for routine tasks. Any activation should trigger an immediate investigation to determine whether the use was authorized and whether underlying access gaps need to be addressed.**


<img width="511" height="856" alt="image" src="https://github.com/user-attachments/assets/285eb0c8-f705-4566-b6ad-d90721cdc087" />



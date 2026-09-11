Security Copilot workspaces are powered by Security Compute Units (SCUs).

Pay-as-you-go capacity requires an Azure subscription. You configure hourly SCUs through the Azure portal or during workspace creation. With pay-as-you-go, you set a baseline (for example, 5 SCUs) that runs continuously and optionally configure overage units that activate during usage spikes

Lisanses: 
**Microsoft 365 E5 inclusion capacity provides 400 SCUs per month per 1,000 E5 licenses. With the inclusion model, a default Security Copilot Capacity appears in your tenant automatically. You don't create hourly capacity; instead, usage deducts from the monthly SCU allocation. Unused SCUs don't roll over. This model simplifies budgeting—you use existing E5 investments without separate Azure billing.**

Provisioned SCUs
= Always available baseline capacity
 
Overage SCUs
= Auto-scale capacity for demand spikes
 
Steady workload
= More provisioned, less overage
 
Spiky workload
= Lower baseline, more overage
 
Overage
= Pay only when consumed

<img width="859" height="300" alt="image" src="https://github.com/user-attachments/assets/22acf27b-28d9-4594-a16a-fda1a87ef971" />

### data residency and prompt evaluation locations

**Security Copilot requires two geographic location decisions during workspace creation: data storage location and prompt evaluation location.**

Data storage location determines where Security Copilot stores session data at rest—prompts, responses, and workspace configuration. Options include Australia (ANZ), Europe (EU), Switzerland (CH), United Kingdom (UK), and United States (US). This setting is immutable after workspace creation.

Prompt evaluation location determines where GPU resources process prompts. You can match the data storage location or select "evaluate anywhere with available capacity." Evaluating anywhere improves responsiveness by routing to the least-busy datacenter but can process prompts outside your chosen data region. Evaluating in a specific region maintains stricter geographic control but can encounter higher latency during regional peak usage.



For workspace creation, you need a supported Security Copilot role (Security Administrator, or a Microsoft Entra/Purview role such as Compliance Administrator, or Purview Organization Management). To set capacity during creation, you also need Azure Owner or Contributor access to the Azure subscription where capacity resources are created.

For workspace configuration, workspace Owners can configure settings, assign roles, manage plugins, and deploy agents within that specific workspace. The workspace Owner role is workspace-specific—being Owner of one workspace doesn't grant ownership of others. Azure Contributor access to the capacity resource enables associating or switching capacity allocations.

For workspace usage, workspace Contributors can use Security Copilot features—submit prompts, run promptbooks, view session history—but can't configure workspace settings or manage access. Contributors represent your typical end users.

Integrated Microsoft Security agents (Defender XDR, Purview, Intune, Microsoft Entra) route traffic to a designated workspace per product. This tenant-wide setting determines which workspace receives embedded experience interactions. Assigning agents thoughtfully ensures operational teams access Security Copilot seamlessly from their primary security portals.


## Configure workspace fundamentals

>Workspace name must be unique within your tenant and follow Azure resource naming conventions.
>Capacity association connects your workspace to Security Compute Units. Select from available capacity resources in your subscription, or select Create new capacity inline.

note : *Capacity resources can only be used in one workspace at a time. If all existing capacity is already allocated, you must create new capacity.*

>Data storage location determines where Security Copilot stores workspace session data at rest. Options include Australia (ANZ), Europe (EU), Switzerland (CH), United Kingdom (UK), and United States (US). This setting is immutable after workspace creation—verify it carefully before proceeding. (**Cannot change after creation**)

>Prompt evaluation location determines where GPU resources process prompts. You can match the data storage location or select Evaluate anywhere with available capacity to route prompts to the least-busy datacenter globally, improving responsiveness at the cost of strict geographic control.
>Data sharing preferences control whether Microsoft can use prompts and responses to improve the service. Organizations with strict data governance policies typically opt out for production workspaces and opt in for sandbox environments.


<img width="2375" height="1207" alt="image" src="https://github.com/user-attachments/assets/e60d8d76-f310-4f8e-a48c-6d7121662e8c" />


1.Sign in to Security Copilot at https://securitycopilot.microsoft.com.
2.Select the workspace name in the breadcrumb, then select New workspace.
3.Enter the workspace name.
4.Select Create new capacity and configure the capacity settings (subscription, resource group, capacity name, prompt evaluation location, provisioned SCUs, overage SCUs).
5.Review the estimated monthly cost, then return to workspace creation.
6.Select the data storage location.
7.Set the prompt evaluation location.
8.Set the data sharing preference.
9.Acknowledge the terms and conditions, then select Create.
Workspace creation takes several minutes as Security Copilot configures resources.


*Capacity association can be changed at any time, but a workspace must always have an assigned capacity to function. Workspace names cannot be changed after creation. To rename a workspace, contact Microsoft Support for assistance.*

### Configure owner settings
---

**Owner settings control workspace behavior for capacity, data sharing, file uploads, and audit logging. All settings apply specifically to the workspace being configured, with one exception: audit logging.**

Capacity management - view current capacity association, adjust provisioned and overage Secure Compute Units (SCUs), and switch to different capacity resources. Changes apply only to the current workspace.

File upload permissions - specify who can upload files to use as knowledge sources during sessions: owners only, or all contributors. The SOC workspace might allow all analysts to upload threat intelligence reports, while the compliance workspace restricts uploads to compliance officers only.

**Audit logging - enables logging of Security Copilot activities to Microsoft Purview. Unlike all other owner settings, audit logging applies tenant-wide across all workspaces and only is changed by a Security Administrator.**

### agent workspace routing
---

<img width="2346" height="1223" alt="image" src="https://github.com/user-attachments/assets/0f015286-cf3c-4d44-81ed-f1479b41e63d" />

**New agent setups in the destination workspace can't access to previous workspace-specific data such as feedback or session memories. Plan migrations carefully to avoid disrupting operational workflows.**

## Monitor and manage workspace capacity

*SCUs can't be shared between workspaces. Each workspace has available and overage SCUs are independent. If a workspace exhausts its allocation, other workspaces can't contribute capacity to prevent throttling.*

<img width="2372" height="791" alt="image" src="https://github.com/user-attachments/assets/3d5c887f-3805-41f6-9854-b12023235c16" />

### Delete or disassociate capacity

**Deleting capacity and its internal data is a permanent action and can't be undone. Session history, feedback, and workspace-specific data are lost. A Security Administrator role is required to delete capacity**

*To temporarily disable a workspace without losing data, disassociate the capacity resource and remove user permissions instead. No billing occurs without assigned capacity, and you can reassign capacity and restore access later.*

Q&A


1.Your organization needs to ensure that prompts and session data for the compliance team remain within the European Union for regulatory reasons. Which workspace configuration setting addresses this requirement?
-Data storage location

2. You need to assign the SOC team lead permissions to configure plugins and manage user access within the SOC workspace. Which Security Copilot role provides these capabilities?
-Owner

3. Your organization sets 5 Security Compute Units (SCUs) for the sandbox workspace. During load testing, all SCUs are consumed and operations are throttled. You configured 3 overage units during creation. What happens next?
-Security Copilot automatically allocates up to 3 extra SCUs on-demand to handle the spike

4.You assign the Defender XDR agent to the SOC workspace. What must you do before switching the agent assignment to a different workspace?
- Turn off any scheduled or automatic agent triggers in the current workspace

5. You update provisioned SCU capacity from 3 to 8 units for the compliance workspace. When does the new capacity take effect?
-Within 30 minutes of the change



Data stored in EU
→ Data storage location
 
Prompt processed in EU
→ Prompt evaluation location
 
Manage plugins and workspace users
→ Owner
 
Submit prompts and use Copilot
→ Contributor
 
5 provisioned + 3 overage
→ Up to 8 SCUs available
→ Overage used automatically and billed when consumed
 
Move agent to another workspace
→ Turn off scheduled/automatic triggers first
 
Change provisioned SCUs
→ Takes effect within 30 minutes
 
Delete workspace
→ Export needed session history first
 
Temporarily disable workspace
→ Disassociate capacity and remove user access




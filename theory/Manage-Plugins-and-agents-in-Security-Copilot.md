# Configure plugin settings in Security Copilot

Plugin governance operates at two distinct levels: user scope and organization scope. 
User scope controls who can add and manage custom plugins for their own sessions—a personal catalog of tools accessible only to the person who added them. 
Organization scope controls who can publish custom plugins that become available to all Security Copilot users in the organization.

<img width="764" height="464" alt="image" src="https://github.com/user-attachments/assets/ad2e8108-98ff-4fed-838b-b1213e3d5161" />

## Configure user-scope permissions
The first permission you configure is who can add custom plugins for themselves:

Owners only: only owners can add and manage their own custom plugins
Owners and Contributors: owners and contributors can both add and manage custom plugins for their own sessions

<img width="876" height="400" alt="image" src="https://github.com/user-attachments/assets/7ba9759a-25e3-46b9-b326-4c43779ea500" />

## Configure organization-scope permissions
Setting user-scope permissions to Owners and Contributors unlocks a second control. Now you can configure who can publish custom plugins for everyone in the organization:

-Owners only: only owners can make custom plugins available organization-wide
-Owners and Contributors: contributors can also publish plugins to the organization

## Restrict preinstalled plugin availability
Restricted plugins affect embedded experiences. If the Microsoft Defender XDR or Natural Language to Kusto Query Language (KQL) plugins are restricted, analysts working in the Defender portal see a degraded or unavailable Copilot experience within that portal. 

## setting up Microsoft-built agents

* Create an agent identity: a dedicated nonhuman identity used exclusively by this agent. Microsoft recommends this option because it improves auditability and limits the blast radius if the identity is ever compromised.
* Assign an existing user account: use the credentials of an existing user to run the agent. Appropriate when the user account already has the required permissions and a dedicated identity isn't needed.
<img width="833" height="384" alt="image" src="https://github.com/user-attachments/assets/494c1d0a-58b3-44ff-ba90-1c7c3ff3f8af" />

## configure partner agents from Security Store
---

### Access the Security Store
*Removing an agent from Security Copilot doesn't cancel a Security Store subscription. Manage subscription billing in Security Store separately from agent operations in Security Copilot.*

### Understand the billing model
Security Store subscriptions and Security Compute Unit (SCU) consumption are separate billing streams:

Security Store subscription: the licensing fee paid to the partner for using the agent. Managed and billed through Security Store.
SCU consumption: the Security Copilot compute cost incurred each time the agent runs. Billed through your Security Copilot capacity.

**When a partner agent requires access to Microsoft product data—such as Defender, Microsoft Sentinel, Intune, or Microsoft Entra—the Set up button is disabled until a Global Administrator approves the required permissions.**

## Manage Security Copilot agents

*Control agent execution*

Agents can run automatically based on a configured trigger or manually on demand:

>Automatic (on trigger): the agent runs on its configured schedule or in response to the defined event. When the trigger is active, the agent runs without manual intervention.
>Manual (one time): select One time to run the agent immediately, regardless of its trigger schedule. Use "one time" to validate a configuration change or run an ad-hoc investigation.

Q&A

1.A Security Copilot owner wants contributors to be able to publish custom plugins for use by everyone in the organization. What must the owner configure first?
>Set the user-scope permission to Owners and Contributors.

2.You restrict the Microsoft Defender XDR plugin to Owners only in your Security Copilot workspace. Which statement best describes the result?
>Analysts using Security Copilot capabilities in the Microsoft Defender portal see a restricted or degraded embedded experience.

3.When setting up a Microsoft-built Security Copilot agent, which identity approach does Microsoft recommend?
>Create a dedicated agent identity.

4. A Security Copilot owner begins setting up a partner-built agent that requires access to Microsoft Defender data. The Setup button is disabled and a banner is displayed. What should the owner do?
>Copy the approval link and share it with a Global Administrator.

5.An organization removes a partner agent from Security Copilot after purchasing a subscription in Security Store. What happens to the Security Store subscription?
> The subscription remains active; billing must be managed separately in Security Store.


User scope first
Contributors publish plugins organization-wide
→ Enable user-scope permission first
→ Then enable organization-scope permission

Owners-only means restricted
Plugin restricted to Owners only
→ Contributors lose access
→ Embedded Defender experience becomes restricted

Agent gets dedicated identity
Microsoft-built agent identity
→ Create a dedicated agent identity
→ Avoid personal user accounts

Global Admin approves partner agents
Partner agent requires approval
→ Copy the approval link
→ Send it to a Global Administrator

Removing agent does not cancel billing
Partner agent removed from Copilot
→ Store subscription remains active
→ Billing must be canceled separately in Security Store

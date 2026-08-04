Three settings govern your vault's security posture from the moment it exists: the SKU, the deletion protection properties, and public network access. 
Azure Key Vault comes in two tiers: Standard and Premium. Both tiers protect keys and secrets at rest and in transit, but they differ in how key material is stored and processed.

Standard tier protects keys using software-based encryption. The key material never leaves Azure, but isn't backed by dedicated hardware. For most workloads—application secrets, connection strings, API keys—Standard provides appropriate protection.

Premium tier backs keys with a hardware security module (HSM) validated to Federal Information Processing Standard (FIPS) 140-3 Level 3. The private key material is created inside the HSM and never exported in unencrypted form. If your compliance framework—PCI DSS, HIPAA, FedRAMP High, or a heavily regulated industry standard—mandates HSM-protected keys for cryptographic operations, Premium is the required choice. The decision isn't a performance consideration; it's a compliance boundary

Soft delete preserves deleted vault objects—keys, secrets, and certificates—in a recoverable state for a configurable period of 7 to 90 days. The default retention is 90 days. During that window, deleted objects remain recoverable, giving your team time to detect accidental or malicious deletion and restore what was lost. Soft delete is enabled by default on all new vaults and can't be disabled after creation. The retention period can only be set at creation time, so consider your recovery SLAs before accepting the default.

Purge protection is the harder enforcement layer. A purge is a permanent, irrecoverable deletion of a soft-deleted object or vault. Without purge protection, any identity with sufficient permissions can delete an object and then immediately purge it—bypassing the "soft delete" retention window entirely. With purge protection enabled, no one can purge a soft-deleted object until the retention period expires, regardless of their permissions. Even subscription owners are blocked. The block matters for encryption-at-rest scenarios: if the key protecting your data is permanently destroyed, the data becomes unreadable.

**Purge protection can't be disabled once enabled.**

Network protection Public network access defaults to enabled on new vaults, which means any internet IP address can attempt to authenticate. Authentication still requires a valid Microsoft Entra token and the appropriate role assignment, so the vault isn't unprotected—but exposing the endpoint to the public internet unnecessarily expands your attack surface. You lock down network access in a later unit, but the vault-level setting is where you set the direction: disabled means private-only by default, with controlled exceptions.

Okay well, How we should create a az keyvault? 

```bash
az keyvault create --name <vault-name> --resource-group <resource-group> --location <location> --sku <premium-standart> --retention-days 90 --enable-purge-protection true
```
Soft delete is always on for new vaults and can't be disabled. --retention-days sets the retention window; --enable-purge-protection locks against permanent deletion and has no off switch after creation.

Azure Policy provides two built-in policies that address required settings directly:

-Key vaults should have soft delete enabled 
-Key vaults should have deletion protection enabled 




<img width="500" height="300" alt="image" src="https://github.com/user-attachments/assets/f731691b-76e2-4868-b8bc-013886f54aba" />

cli prefered:


```bash
az policy assignment create --name <value-name> --scope "/subscriptions/$(az account show --query id -o tsv)" --policy "" --params '{"effect": {"value": "Deny"}}'
```


more information :
https://learn.microsoft.com/en-us/azure/key-vault/policy-reference
---
### Understand the Key Vault access model 

**Azure Key Vault separates access into two distinct planes: the control plane and the data plane.**
The control plane governs the vault itself. Control-plane operations use Azure Resource Manager (ARM) and include creating or deleting the vault, configuring network rules, and viewing vault metadata. The data plane governs items stored inside the vault—keys, secrets, and certificates. Both planes authenticate through Microsoft Entra ID, but authorization is independent on each plane.

This separation is a deliberate security boundary. An identity with permission to manage the vault's configuration—a Platform Engineer updating network rule, for example—doesn't automatically get access to the secrets the vault holds. Conversely, an application identity that reads connection strings from the data plane doesn't need control-plane permissions. Design your role assignments to respect this boundary: grant control-plane access only to identities managing the vault as infrastructure, and data-plane access only to identities or workloads that need the content

in schema: 

<img width="450" height="125" alt="image" src="https://github.com/user-attachments/assets/b4a0873c-eab0-4d1b-b434-be58a3c7821d" />

**Key Vault supports two authorization models for the data plane: Azure role-based access control (RBAC) and legacy vault access policies.**

Microsoft.Authorization/roleAssignments/write permission, which is separate from Contributor and not implicitly held by most operational roles. With RBAC, the person managing the vault's network settings can't silently grant themselves secrets access.

There's also a functional limitation to access policies: they don't support Privileged Identity Management. If you want to apply just-in-time elevation for Key Vault operations—which you do—access policies can't participate in that model. Use Azure RBAC for all new vaults. Starting with API version 2026-02-01, Azure RBAC is the default permission model for newly created Key Vaults, consistent with the portal experience.


when we assing rbac controls 
**Owner, Contributor** have to **Control plane**	because The vault resource itself—create, configure, and delete via ARM template
**Key Vault Administrator, Secrets User, Certificates Officer, Purge Operator**	**Data plane**	The vault content—keys, secrets, certificates
Key Vault Administrator grants full data-plane access to all keys, secrets, and certificates in the vault. Key Vault Administrator is the most powerful role in Key Vault and should never be a permanent assignment for humans. Reserve it for break-glass scenarios activated through a privileged access process, and for automation that genuinely requires cross-object access.

Key Vault Secrets User grants read-only access to secret values. Secrets user is the correct role for application service principals and managed identities that retrieve database connection strings, API keys, or other runtime secrets. The identity can read the secret content but can't list, create, update, or delete secrets. Assign this role to your workload identity, not Key Vault Administrator.

The same applies to AI agent workload identities. If you're deploying agents built on Azure Foundry, Microsoft Copilot Studio, or Agent 365, Microsoft Entra Agent ID (currently in preview) provides a dedicated service principal type for agents. Agent identities support the same Azure RBAC role assignments as any other service principal. However, because agent identities are service principals, they cannot hold eligible PIM assignments—the JIT activation flow requires interactive human steps such as MFA verification and approval requests that an autonomous agent can't perform. To time-bound an agent's Key Vault access, use a time-limited active role assignment with a defined start and end date instead. Assign Key Vault Secrets User—not Key Vault Administrator—regardless of whether the agent authenticates with a managed identity or an agent identity blueprint credential.

Key Vault Certificates Officer allows full lifecycle management of certificates—creating, importing, updating, and deleting—without touching secrets or keys. Assign this role to certificate management automation or the team responsible for certificate renewal workflows.

Key Vault Purge Operator allows permanent deletion of soft-deleted vault objects. This role should be tightly controlled and used only in explicit recovery or decommissioning scenarios, where purging a soft-deleted object is intentional. Don't assign it broadly.


Microsoft Entra Privileged Identity Management (PIM) replaces permanent role assignments with eligible assignments. An eligible user doesn't hold the role. When they need elevated Key Vault access, they request activation through PIM, which enforces the controls you configure: justification, approval from a designated approver, MFA reverification, and a maximum activation duration. After the window closes, the role is automatically removed and the next request starts the same process.

Beyond PIM, Microsoft Entra Conditional Access lets you apply policy-based controls to Key Vault access. You can require MFA for access from unmanaged or noncompliant devices. Then you can block access from risky sign-in conditions, and limit Key Vault access to requests that originate from locations or network paths you trust. Conditional Access policies apply to agent identities in the same way they apply to human identities, giving you a second enforcement layer that operates independently of role assignments

---

 ## Configure Key Vault firewall and network settings

When you create a new Key Vault, the firewall is disabled and public network access is enabled. Any internet IP address can resolve your vault's public DNS name and send authentication requests to it. The Microsoft Entra token and role assignment requirements you configured in earlier units still apply—but that only means an attacker needs valid credentials, not that the endpoint is unreachable. Every request, legitimate or adversarial, arrives at your vault directly across the public internet.

The first level of network hardening is enabling the Key Vault firewall and adding explicit allow rules. When the firewall is enabled without any configured rules, all traffic is blocked by default. You then add the specific IPv4 addresses or Classless Inter-Domain Routing (CIDR) ranges that should have access.

This model works well for defined use cases. A CI/CD pipeline running on build agents with static, known IP addresses is a good fit—you add the agents' CIDR range to the allow list, and the pipeline continues to reach Key Vault to read deployment secrets. On-premises systems connecting over site-to-site VPN with a known egress address are another valid scenario.

### Configure the firewall in the Azure portal
I'll introduction to firewall management with az cli 

```bash
az keyvault network-rule add --name <'keyvault-name'> --resource-group <'resource-group-name'> --ip-adress <'ip-adrress-or-cidr'>
```



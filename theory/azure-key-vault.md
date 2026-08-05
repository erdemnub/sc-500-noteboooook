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

another note : 

**Firewall rules apply only to data plane operations—reading secrets, keys, and certificates. Control plane operations (creating or configuring the vault via ARM template) aren't subject to firewall restrictions. Also, private IP address ranges (RFC 1918: 10.x, 172.16–31.x, 192.168.x) can't be used in IP allow list rules**


Route virtual network traffic over the Azure backbone (Azure Microsoft’s global wide area network (WAN) using service endpoints
Virtual network service endpoints give workloads inside an Azure virtual network a more structured alternative to IP-based rules. When you enable the <'Microsoft.KeyVault'> service endpoint on a virtual network subnet and add that subnet to your Key Vault network rules, traffic from that subnet travels over the Azure backbone network rather than the public internet.

to addining :
```bash
az network vnet subnet update \
  --resource-group "<rg-name>" \
  --vnet-name "<vnet-name>" \
  --name "<subnet-name>" \
  --service-endpoints "Microsoft.KeyVault"
```
or 

```bash
az keyvault network-rule add
```

isolate Key Vault with private endpoints 

Two components are required for private endpoint connectivity to work correctly.

The first is the private endpoint resource - a network interface in your virtual network subnet, bound to Key Vault via Azure Private Link, that receives a private IP address.

The second is a private DNS zone. Without DNS integration, clients resolve your vault's public IP regardless of the private endpoint. Configure the privatelink.vaultcore.azure.net zone and link it to the virtual network so that <vault-name>.vault.azure.net resolves to the private IP. The Azure portal offers automatic DNS integration during private endpoint creation.

Once the private endpoint is deployed and DNS is configured, disable public network access on the vault. That combination—private endpoint active, public access disabled—means the vault is fully isolated from the internet.



**Disabling public network access affects some Microsoft-managed services that need to reach Key Vault—including Azure Monitor and Azure Backup. Use the Allow trusted Microsoft services to bypass this firewall exception for services in this category, or ensure those services connect through a path in your private network.**

choosing right isolation model : 

<img width="650" height="300" alt="image" src="https://github.com/user-attachments/assets/b46670b3-7563-4cd7-b2a1-632fb3c48367" />

let the explain this schema 

Private endpoint with public access disabled - the recommended architecture for production workloads storing sensitive data. Provides complete network isolation with no internet-facing endpoint.
VNet service endpoints - appropriate when private endpoints aren't feasible due to virtual network architecture constraints. Eliminates internet routing for controlled workloads, but the public endpoint persists.
IP-based firewall rules - a valid transitional state for tightly bounded use cases such as CI/CD pipelines from build agents with static IPs. Not a long-term production architecture.
Network Security Perimeter (NSP) - a GA option for organizations that need to define a logical isolation boundary across multiple PaaS resources (Key Vault, Storage, SQL Database) outside your virtual network perimeter. NSP uses publicNetworkAccess: SecuredByPerimeter and supports inbound/outbound access rules. Note the setting overrides the trusted Microsoft services bypass—services relying on that bypass, such as Azure Monitor and Azure Backup, are blocked if NSP is active.
Trusted services bypass - needed when Microsoft-managed services like Azure Monitor, Azure Backup, or Azure Site Recovery require Key Vault access that can't be routed through your private network. Not applicable when NSP is in use.

I'v been alredy added more sources about configured vnet networks https://learn.microsoft.com/en-us/azure/key-vault/general/network-security

**For agents built with Microsoft Copilot Studio, two supported access patterns are available. The first uses Power Platform Virtual Network support in a Managed Environment: when virtual network support is enabled, the agent calls Key Vault directly over a private link using an HTTP Request node, with all traffic staying on your private network. The second uses environment variable secret references: Copilot Studio Service is granted to the Key Vault Secrets User role on the vault, and the agent accesses secrets through Power Platform environment variables—no direct network path to Key Vault is required from the agent itself. Review these patterns before deciding whether IP allow lists or public access are appropriate for agents in your design.**

Q&A

Your organization's compliance policy requires that Key Vault objects can't be permanently deleted during a 90-day retention window—even by subscription owners. Which Key Vault setting enforces retention?
>Purge Protection

Contoso's application needs to retrieve secrets from Azure Key Vault without storing credentials in code or configuration files. What is the recommended approach?
>Assign the Key Vault Secrets User role to a managed identity


 A security engineer needs application servers in an Azure virtual network to access Key Vault with no public internet exposure. Which solution meets the requirement?
>Configure a private endpoint for Key Vault
>










---

# Manage keys and secrets in Azure Key Vault
lifecycle gaps
Key rotation limits the damage window if a key is ever exposed. An unrotated key in use for three years was potentially accessible to every system, person, and process that touched it across that entire period. Configuring an automatic rotation policy doesn't close that historical window—but it caps all future ones. From the moment rotation is active, the maximum exposure window equals the rotation interval.

Secret expiry works differently but addresses the same underlying risk. An expiry date doesn't rotate credentials automatically—it forces them to become invalid, creating an operational requirement to replace them. Without expiry, a compromised database connection string remains usable for as long as the secret exists. The credential becomes a permanently open door rather than one that closes on a schedule.

Rotation and Expiry

 ---

## Manage cryptographic keys in Azure Key Vault
for example A key unrotated for three years isn't just a policy violation—it's an exposure window. If the key was ever visible to an unauthorized party, attackers had three years to use it. 

Azure Key Vault supports three cryptographic key types:
>RSA keys are asymmetric keys used for encryption, decryption, digital signatures, and key wrapping. Key Vault supports sizes of 2,048 bits, 3,072 bits, and 4,096 bits. RSA-2048 is the minimum for production use; RSA-4096 is appropriate for long-term data protection or where extended assurance is required.

How it works: It uses a pair of keys: a Public Key to encrypt (lock) and a Private Key to decrypt (unlock).

Key Sizes: 2048, 3072, 4096 bits.

Analogy: A public padlock. Anyone can snap the padlock shut on a box (Public Key), but only you hold the key to open it (Private Key).

Use Cases:

Encrypting small data or secret tokens.

Key Wrapping: Encrypting symmetric keys so they can be transferred safely.

Digital signatures.

>Elliptic curve (EC) keys are asymmetric keys used for digital signatures. The supported curves are P-256, P-384, P-521, and P-256K (secp256k1). EC keys produce smaller key material at equivalent security strength compared to RSA, making them efficient for certificate operations and signing workloads.

Elliptic Curve (EC) Keys (Asymmetric / Asimetrik)
How it works: Like RSA, it is asymmetric (uses public/private key pairs), but it is based on complex algebraic curves instead of huge prime numbers.

Supported Curves: P-256, P-384, P-521, and P-256K (secp256k1).

Why use EC over RSA? EC produces much smaller key sizes with the same level of security as RSA. Smaller key size = faster performance, less storage, and lower computational power needed.

Analogy: A lighter, high-tech titanium padlock. It offers the exact same strength as a heavy iron padlock (RSA), but weighs a fraction of the size.

Use Cases:

Digital Signatures: Verifying identity quickly.

SSL/TLS Certificates: Securing web traffic efficiently.

Blockchain/Web3: P-256K (secp256k1) is used in Bitcoin and Ethereum wallet signing operations.

>Symmetric (octet) keys are used for symmetric encryption. Software-protected octet keys support 128-bit, 192-bit, and 256-bit sizes. Unlike RSA and EC keys, octet keys can't be HSM-backed in the Key Vault Premium service—they're supported as HSM keys only on Azure Key Vault Managed HSM. (**Symmetric / Octet Keys (Symmetric / Simetrik)
How it works: It uses a single shared key to both encrypt and decrypt data. (This is where standard algorithms like AES fit in Supported Sizes: 128-bit, 192-bit, and 256-bit.**)


In hsm's: 

HSM-protected keys are generated, stored, and processed entirely within hardware security modules (HSMs) validated to FIPS 140-3 Level 3. (**Key operations execute inside the HSM boundary; the private key material never exists in software memory.**)


**HSM-protected keys require the Premium SKU for Azure Key Vault. The Standard SKU supports software-protected keys only.** And RSA-HSM or EC-HSM keys. 


### BYOK ( Bring Your Own Key ) SCENERIOS.

BYOK scenario, you generate the key material inside your own on-premises or external HSM. And transferring that key to azure key vault services ,  Key Exchange Key (KEK). 

The proceess schema : 

1.You generate a KEK as an RSA-HSM key inside your Key Vault Premium instance. The KEK must have only the import key operation permitted—the import operation is mutually exclusive with all other key operations.
2.You download the KEK public key as a .pem file to a computer connected to your on-premises HSM.
3.On the offline computer, you use your HSM vendor's BYOK tool to encrypt (wrap) your Customer Key using the KEK public key, producing a BYOK file.
4.You upload the BYOK file to Key Vault. Inside the Key Vault HSM, the KEK private key unwraps the Customer Key, and the key material is imported directly into HSM protection.

BYOK is supported for RSA-HSM and EC-HSM key types with supported HSM vendors including Thales, Entrust (nShield), Fortanix, IBM, Marvell, Utimaco, and others. 

---

### Key autotoration.

A rotation policy on a key specifies:

Rotation time: how frequently Key Vault creates a new version. Set rotation time based on your compliance requirements and operational risk tolerance.

Expiry time: the lifetime of each key version after which Key Vault marks it as expired. For fully automated rotation, expiry must also be set.

Notification time: the number of days before expiry at which Key Vault publishes a near-expiry event to Azure Event Grid.



When Key Vault creates a new key version through rotation, applications that reference the key by its versionless URI (using only the vault URL and key name, without a specific version) automatically receive the latest version. No application configuration change is needed at rotation time.

for example : 

>Versioned URI (Not be should)
(https://myvault[.]vault.azure.net/keys/mykey/a1b2c3d4e5f6)

>Versionless URI
(https://myvault[.]vault.azure.net/keys/mykey) (No key id no another path. )



example can apply scenerio :

```bash
az keyvault key rotation-policy update --vault-name <vault-name> --name <key-name> --value ./rotation-policy.json
```

the output is : 
```
{
  "lifetimeActions": [
    {
      "trigger": { "timeAfterCreate": "P18M" },
      "action": { "type": "Rotate" }
    },
    {
      "trigger": { "timeBeforeExpiry": "P30D" },
      "action": { "type": "Notify" }
    }
  ],
  "attributes": {
    "expiryTime": "P2Y"
  }
}
```
**This policy rotates the key 18 months after creation and notifies 30 days before the two-year expiry date and after 2 year that will be expire.**

<img width="325" height="650" alt="image" src="https://github.com/user-attachments/assets/0f1384c4-6bc3-4bf3-8c62-a118bdd256a4" />

Applications using the versionless URI always encrypt new data with the latest version. However, for data encryption scenarios—where a data encryption key (DEK) is wrapped by the Key Vault key—store the versioned URI alongside the encrypted data so decryption always uses the exact key version that wrapped the DEK. Existing data remains encrypted under its original version until you explicitly re-encrypt it. Re-encrypting existing data to bring it under the new key version is a separate, planned activity—it isn't automatic, and it isn't required for decryption to continue working.

In addition to autorotation, you can set NotBefore and Expires attributes on individual key versions to control their operational window. A key version with a future NotBefore date isn't usable until that date—useful for pregenerating a replacement key during a planned cutover. A key version with a past Expires date signals to consuming services that it should no longer be used for new operations.

Microsoft.KeyVault.KeyNearExpiry

on cli :

```bash
az eventgrid event-subscription create \
  --name "evs-key-near-expiry-notification" \
  --source-resource-id "/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.KeyVault/vaults/<keyvault-name>" \
  --included-event-types "Microsoft.KeyVault.KeyNearExpiry" \
  --endpoint-type "azurefunction" \
  --endpoint "/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.Web/sites/<function-app-name>/functions/<function-name>"
```
---

## Manage secrets in Azure Key Vault

In Azure Key Vault, a secret is any sensitive string value: connection strings, API keys, passwords, storage account keys, tokens, and similar credentials. Secrets are intentionally distinct from keys and certificates. Keys perform cryptographic operations (signing, encrypting, wrapping). Certificates assert identity through a Public Key Infrastructure (PKI) chain. Secrets are simply values—stored securely, accessed through a controlled API.

Every time you update a secret value, Key Vault creates a new version of that secret. The old version is retained—you can retrieve it, audit access to it, and disable it.

You can set two important attributes on any secret version:

Enabled/Disabled: A disabled secret version can't be retrieved. Disabling (rather than deleting) a version is the correct way to deprecate a credential while retaining the audit trail. If an incident investigation later requires knowing when a specific credential was active, the disabled version and its access log are still available.
Expires: Set an expiry date on each secret version to establish a maximum credential lifetime. When a version expires, Key Vault still allows retrieval—expiry is informational, not a hard enforcement block. The attribute signals to applications and downstream services that the credential should no longer be used for new operations, and it triggers Event Grid lifecycle events that drive rotation automation. The version and its access history remain in the vault.

lets review when will expire our keyvault secret.
```bash
az keyvault secret set-attributes --vault-name <vault-name> --name <secret-name> --expires 2026-10-14T00:00:00Z
```

Key Vault references in Azure App Service and Azure Functions makes seamless application key usage possible.

Instead of storing a connection string directly in an app setting, you store a reference:

```
@Microsoft.KeyVault(SecretUri=https://<vault-name>.vault.azure.net/secrets/<secret-name>)
```
Versionless URI - references the secret by name only, without a pinned version. When the application starts, it resolves this reference to the latest enabled version of the secret. When you rotate the secret and add a new version, the application picks up the new value automatically within 24 hours. 

---

### credential rotation


What is a credential?: Any secret key used for authentication, such as a database password, API key, or connection string.

What is single-credential rotation?: The simplest rotation pattern where a single secret is updated to a new value in a single atomic step.

Where does the risk come from?: There is a brief transition delay between updating the target system and updating the secret in Key Vault.

What is the impact on the application?: If the application reads the old credential from Key Vault during that brief window, authentication fails because the target system already expects the new one.

Where is it acceptable?: It is acceptable for non-critical workloads with retry logic, but unacceptable for high-availability (HA) production environments.


Implement dual-credential rotation
Dual-credential rotation eliminates the transition window by maintaining two valid credentials simultaneously in the target system. The pattern accommodates the full rotation cycle without a moment when both the application and the target system agree on a valid credential.

Here's the pattern in detail, using a database as the example:

Set up: The database has two sign-in credentials - db-user-a and db-user-b. Both are valid. The vault's active secret contains the password for db-user-a. The application uses the secret (versionless URI) to authenticate.

Rotate credential B first: Generate a new password for db-user-b. Update the database sign-in for db-user-b with the new password. The application is still using db-user-a - nothing disrupts it.

Store credential B as the new secret version: Add the new db-user-b password to Key Vault as a new version of the secret. The versionless reference in the application now resolves to db-user-b.

Verify the application uses credential B: Confirm the application is authenticating with db-user-b. You can validate this through query logs, connection tracing, or diagnostic logs.

Rotate credential A: Now that the application runs on db-user-b, generate a new password for db-user-a. Update the database sign-in. The application isn't using db-user-a, so this change causes zero disruption.

Store credential A as the next secret version: Add the new db-user-a password as the next version. The cycle is complete.

Credentials alternate on each rotation cycle. At no point, does a rotation event leave the application without a valid credential in the target system.




Implementing dual-credential rotation with Azure Functions: This pattern is typically automated using an Azure Function that triggers on the Key Vault SecretNearExpiry Event Grid event. When the secret approaches its expiry date:

The Microsoft.KeyVault.SecretNearExpiry Event Grid event calls the function app endpoint via HTTP POST.
The function identifies which credential isn't* currently stored in the latest vault version (the alternating credential).
The function regenerates that credential in the target system.
The function stores the new credential as a new version in Key Vault.
The function app's managed identity needs the Key Vault Secrets Officer role on the vault and the appropriate permission on the target resource (for example, the Storage Account Key Operator Service Role for storage account keys, or database admin credentials for SQL sign-in rotation).

example schema :



<img width="150" height="532" alt="image" src="https://github.com/user-attachments/assets/51a5f81d-2d22-442d-8baf-39a502b773c7" />

### Q&A
A regulatory compliance assessment requires that encryption keys protecting patient record archives meet FIPS 140-3 Level 3 hardware security requirements. Which Azure Key Vault configuration satisfies this requirement?
>Create RSA-HSM keys in an Azure Key Vault Premium vault.

 A security engineer configures an autorotation policy for an RSA encryption key in Azure Key Vault. What happens to data that was encrypted with the previous key version after rotation occurs?
>Previously encrypted data remains accessible because Key Vault retains all key versions.

 An Azure Function uses a database connection string stored as a Key Vault secret. A security engineer wants to rotate the connection string without causing application downtime. What rotation pattern should they use?
>Dual-credential rotation using two alternating valid connection strings.


 ---


 # Manage certificates and monitor Azure Key Vault


































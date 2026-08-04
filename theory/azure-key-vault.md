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
az keyvault create --name <vault-name> --resource-group <resource-group> --location <location> --sku premium --retention-days 90 --enable-purge-protection true
```

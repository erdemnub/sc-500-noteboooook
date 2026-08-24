## Enable soft delete and immutable vaults

Azure Backup provides layered protections that convert single-point-of-failure scenarios into defense-in-depth architectures.

**Enhanced soft delete and Vault immutability.**

Azure Backup's tiered security posture model addresses the problem by adding recovery windows, immutability guarantees, and multi-administrator approval requirements. The goal: make backup destruction impossible for a single compromised account, even if that account holds the Backup Contributor role.

### Enhanced soft delete
>Soft delete creates a safety net for deleted backup items. When backup data is deleted—whether by an administrator, an attacker, or accidental user action—the recovery points enter a soft-deleted state instead of being immediately purged. During the soft delete retention period, you can recover the data at no extra cost. The data remains in Azure storage; only the deletion is reversed.

Azure Backup enforces soft delete by default across all global Azure regions as part of its Secure by default platform commitment. In regions where this enforcement is generally available, soft delete can't be disabled from the Azure portal for any vault.



### Within this baseline, you configure the retention period and the always-on setting:

>Soft delete retention: configurable from 14 to 180 days. The default is 14 days. Longer periods provide more time to detect advanced persistent threats before the recovery window expires.

**Always-on soft delete: makes the retention period itself irreversible. Once enabled, no user—including Global Administrators—can reduce or disable the retention period on that vault.**

Soft delete coverage includes Azure VMs, SQL Server in Azure VMs, SAP HANA in Azure VMs, Azure Files (vaulted backup), and on-premises servers protected by the MARS agent.




## Vault Immutability :

Vault immutability prevents backup policies from being modified in ways that reduce recovery point retention, and prevents backup data from being deleted before the retention period expires. This implements WORM (Write Once, Read Many) storage behavior at the vault level. Recovery points, once written, can't be modified or deleted until their retention period expires naturally.


Immutability operates in three states:

Disabled: No protection—the default state for existing vaults. Administrators can modify policies and delete data freely.
Enabled (reversible): Immutability is active. Recovery points can't be deleted, and policies can't be modified to reduce retention. This setting can be disabled if organizational requirements change.
Enabled and Locked (irreversible): Immutability is permanently active and can't be disabled by any user, including Global Administrators. Permanently active is the WORM compliance state.


**The Enabled and Locked state is irreversible by any user, including subscription owners and Global Administrators. Before locking, verify that your vault's backup policies and retention settings meet long-term organizational requirements. After locking, you can increase retention periods but never decrease them. Only lock production vaults after confirming that policy configurations are finalized.**




### Azure Backup security posture rating :

The Excellent tier requires two elements: a deletion protection mechanism (either locked immutability or always-on soft delete), and Multi-User Authorization. The deletion protection mechanisms—soft delete and immutability—protect recovery points after an attack begins. Multi-User Authorization prevents a single compromised credential from disabling those protections.


### Resource guard : 
 Resource Guard is a separate Azure resource that acts as a gatekeeper for critical backup operations. The vault administrator configures backup policies and manages day-to-day backup operations. The Resource Guard owner (a separate security administrator) approves requests to perform operations that could weaken the vault's security posture.

The key architectural principle: the vault administrator must not have Contributor, Backup MUA Admin, or Backup MUA Operator permissions on the Resource Guard. Any of these roles allow self-approval of critical operations, defeating the two-administrator requirement. Effective MUA deployment requires role separation.


Resource Guard protects two mandatory operations for Recovery Services vaults that can never be excluded:

>Disable soft delete or security features: Prevents removal of the soft delete recovery window or other security settings.
>Remove MUA protection: Prevents disabling MUA itself, so an attacker can't remove the gatekeeper before executing other destructive operations.

**MUA configuration involves three roles: the security administrator who creates and manages the Resource Guard, the vault administrator who manages backup operations, and the Resource Guard owner who approves critical operation requests. The same person can assume multiple roles in small organizations, but they must use separate accounts for each role to maintain technical role separation.**

Step 1: Create the Resource Guard (security administrator account)

The security administrator creates the Resource Guard in a subscription separate from the Recovery Services vaults:

1.Sign in to the Azure portal with the security administrator account
2.Search for Resource Guards and select the service
3.Select + Create
4.Choose a subscription different from the one containing your Recovery Services vaults
5.Select or create a resource group—use a resource group dedicated to security infrastructure
6.Provide a name for the Resource Guard (for example, rg-backup-security-prod)
7.Select the region—must match the Recovery Services vault's region
8.Under Protected operations, review the default selections—all optional operations except Restore are enabled by default; deselect any that don't require two-administrator approval in your environment
9.To require approval for restore operations, select Restore (disabled by default)
10.Select Review + create, then Create

Step 2: Assign the vault admin as Reader on the Resource Guard

The vault administrator needs read access to the Resource Guard to see that it exists and to submit approval requests. They must not have Contributor, Backup MUA Admin, or Backup MUA Operator permissions on the Resource Guard—any of these roles allow self-approval of critical operations.

1.Navigate to the Resource Guard you created
2.Select Access control (IAM)
3.Select + Add role assignment
4.On the Role tab, select Reader
5.On the Members tab, select the vault administrator account or the service principal used for backup operations
6.Select Review + assign
7.Step 3: Configure the vault to use MUA

The vault administrator configures the vault to enforce MUA using the Resource Guard:

1.Sign in as the vault administrator
2.Navigate to the Recovery Services vault
3.Under Settings, select Properties
4.Under Multi-User Authorization, select Update
5.Select Protect with Resource Guard
6.Use the cross-subscription resource picker to select the Resource Guard created in Step 1
7.Select Save

After this configuration, any attempt by the vault administrator to perform a protected operation generates a request that requires Resource Guard owner approval. The request includes context about the operation and the identity of the user requesting it.


**When the vault administrator needs to perform a protected operation, the approval flow uses Microsoft Entra Privileged Identity Management (PIM). The security administrator creates an eligible assignment of the Backup MUA Operator role on the Resource Guard for the vault administrator. When a protected operation is needed, the vault administrator activates their eligible assignment in PIM, which raises an approval request to the security administrator. After approval, the vault administrator holds the Backup MUA Operator role for the approved time window, and Azure Resource Manager validates that role membership when the critical operation is attempted. PIM automatically revokes the role when the approved period ends. For organizations without PIM, the security administrator can manually grant and revoke the Backup MUA Operator role via Access control (IAM) on the Resource Guard.**



### Backup RBAC Roles
>Azure Backup provides three built-in RBAC roles with different privilege levels

-Backup Contributor: Two named administrators who configure vaults and policies (accounts that submit MUA approval requests)
-Backup Operator: The service account used by automated backup scripts and Azure Automation runbooks
-Backup Reader: The SOC monitoring dashboard service principal, compliance auditors, and helpdesk staff





## Key encryption for recovery points : 
By default, Azure Backup encrypts all recovery points using platform-managed keys. Microsoft manages the encryption keys, handles key rotation, and maintains the encryption infrastructure. Platform managed keys provides AES-256 encryption at rest with no configuration required.

**Customer-managed key (CMK) encryption lets you use encryption keys stored in Azure Key Vault that your organization controls. You manage key lifecycle, rotation policy, and access permissions. CMK provides more data sovereignty assurance for organizations with requirements to control cryptographic key material used to encrypt backup data.**

**CMK encryption for Recovery Services vaults is a one-way commitment. Once you enable CMK encryption on a vault, you can't revert to platform-managed keys.**
**Azure Backup also only supports CMK on new vaults that have no registered backup items. You can't migrate an existing vault with backup data from platform-managed keys to customer-managed keys.**

If CMK encryption is a compliance requirement, configure it during vault creation before registering any servers or VMs for backup. Configure the Key Vault, create (or import) the encryption key. Then grant the vault's managed identity permission to access the key, then enable CMK in the vault's Properties > Encryption settings.

# Layers of Encryption (Envelope Encryption)

**Data Level (DEK — Data Encryption Key): Storage or Database services encrypt data at rest using a high-performance, symmetric key (DEK) directly on the disk.**

**Key Level (KEK — Key Encryption Key / CMK): The DEK cannot remain exposed in plain text; it is wrapped (encrypted) by the Customer-Managed Key (CMK) stored in Azure Key Vault.**

**Hardware Level (HSM — Hardware Security Module): The CMK itself resides inside FIPS-compliant, dedicated physical hardware security modules (HSM) within Key Vault, ensuring the key material never leaves the cryptographic boundary unencrypted.** (Maximum)

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




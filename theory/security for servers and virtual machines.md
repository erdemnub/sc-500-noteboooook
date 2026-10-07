## Azure provides four disk encryption options for virtual machines.

<img width="903" height="374" alt="image" src="https://github.com/user-attachments/assets/0a49651f-79f7-424e-a0e9-9b4cb341c192" />

### Server-side encryption (SSE) runs automatically on all Azure managed disks. When you create a virtual machine, Azure encrypts the OS and data disks at the storage platform level using AES-256 encryption.This protection requires no configuration and provides defense against physical disk theft or improper disposal.
SSE doesn't encrypt temporary disks or disk caches.


### Encryption at host: The current recommendation
Encryption at host extends SSE coverage to include temporary disks and disk caches.

With encryption at host enabled, all data leaving the VM undergoes encryption before reaching Azure Storage. This includes:

OS disk read/write operations
Data disk I/O
Temporary disk storage (D: drive on Windows, /dev/sdb1 on Linux)
Disk controller cache operations
You enable encryption at host as a VM property during creation or by stopping an existing VM, enabling the feature, then restarting. The encryption layer sits between the VM and the storage subsystem, requiring no changes to guest OS configurations or application code. 

### Azure Disk Encryption:
Azure Disk Encryption (ADE) uses BitLocker (Windows) or DM-Crypt (Linux) to encrypt disks from within the guest operating system.Microsoft announced that ADE retires on September 15, 2028.

Organizations using ADE must migrate to encryption at host or another supported option before the retirement date. The migration involves:

Creating snapshots of encrypted disks
Creating new VMs with encryption at host enabled
Restoring data from snapshots to the new encrypted environment
Validating application functionality
Decommissioning the ADE-protected VMs
Unlike encryption at host, ADE requires guest OS integration, consumes VM compute resources for encryption operations, and doesn't encrypt temporary disks. These limitations make encryption at host the superior replacement for most workloads.

### Confidential disk encryption: Hardware-isolated protection
Confidential virtual machines provide hardware-based isolation using AMD SEV-SNP (Secure Encrypted Virtualization - Secure Nested Paging) technology. When you create a confidential VM, you can enable confidential disk encryption, which binds the OS disk encryption key to the VM's virtual Trusted Platform Module (vTPM)


Confidential VMs use DCasv5 or ECasv5 series sizes and require the security type set to "Confidential virtual machine" at creation time. You can't convert existing standard VMs to confidential VMs. 

---
## Configure encryption at host with customer-managed keys
---

### Platform-managed keys versus customer-managed keys

Encryption at host supports two key management approaches. Platform-managed keys (PMK) use encryption keys that Microsoft generates, stores, and rotates automatically. This approach requires no configuration beyond

Customer-managed keys (CMK) give your organization control over the encryption key lifecycle. You generate and store keys in Azure Key Vault or Azure Key Vault Managed HSM. Then you define rotation policies, set expiration dates, and maintain audit logs of every key access operation
---
### Enable encryption at host on new VMs
---
```bash
az vm create 
--resource-group company-mfg-rg \
--name plc-mgmt-vm-01 --image Ubuntu2204 \
--size Standard_D4s_v5 \
--encryption-at-host true\
--os-disk-encryption-set /subscriptions/{subscription-id}/resourceGroups/company-security-rg/providers/Microsoft.Compute/diskEncryptionSets/company-mfg-des-eastus2\
--admin-username azureuser\
--generate-ssh-keys
```

### Enable encryption at host on existing VMs
Stop and deallocate the VM
```bash
az vm deallocate --resource-group contoso-mfg-rg --name legacy-factory-vm-03
```
 Enable encryption at host
```bash
az vm update --resource-group contoso-mfg-rg --name legacy-factory-vm-03 --set securityProfile.encryptionAtHost=true
```
 Update OS disk to use customer-managed key
```bash
az vm update --resource-group contoso-mfg-rg --name legacy-factory-vm-03 --set storageProfile.osDisk.managedDisk.diskEncryptionSet.id=/subscriptions/{subscription-id}/resourceGroups/contoso-security-rg/providers/Microsoft.Compute/diskEncryptionSets/contoso-mfg-des-eastus2
```
 Start the VM
```bash
az vm start --resource-group contoso-mfg-rg --name legacy-factory-vm-03
```

### Verify encryption is active

```bash
az vm show --resource-group contoso-mfg-rg --name plc-mgmt-vm-01 --query securityProfile.encryptionAtHost
```
A true response confirms encryption at host is enabled.

check the encryption settings on the OS disk:

```bash
az disk show --resource-group contoso-mfg-rg --name plc-mgmt-vm-01_OsDisk --query [encryption.type,encryption.diskEncryptionSetId] --output table
```

The output displays EncryptionAtRestWithCustomerKey and the Disk Encryption Set resource ID, confirming customer-managed key protection.





## confidential disk encryption to confidential virtual machines

Azure confidential VMs run on DCasv5-series and ECasv5-series hardware with AMD EPYC processors supporting Secure Encrypted Virtualization-Secure Nested Paging (SEV-SNP). This technology encrypts VM memory at the hardware level using a key inaccessible to the hypervisor, Azure operators, or other VMs on the same physical host.

Unlike standard VMs where Azure operators can theoretically access memory during maintenance operations, confidential VMs maintain cryptographic isolation enforced by the CPU. The processor generates attestation reports proving the VM runs in a protected environment without tampering or unauthorized access


### Security types in Azure

<img width="917" height="309" alt="image" src="https://github.com/user-attachments/assets/2a120c4b-9786-4048-8f5f-eb6d4f2655c3" />

You can't convert a Standard or Trusted launch VM to a Confidential VM after deployment.  This restriction exists because confidential computing requires specific CPU features and firmware configurations initialized during VM provisioning.

### How confidential disk encryption works

Confidential disk encryption binds the OS disk encryption key to the VM's vTPM. The vTPM is a virtualized Trusted Platform Module that stores cryptographic keys in a hardware-protected environment isolated from the guest operating system and Azure management plane.

When the VM boots, the vTPM releases the disk encryption key only after verifying the boot chain integrity. If an attacker modifies boot components, copies the disk to another VM, or attempts offline access to the encrypted volume, decryption fails because the vTPM key binding validation fails.

This protection differs fundamentally from encryption at host. Encryption at host encrypts data flows between the VM and storage but uses keys accessible to the Azure storage platform. Confidential disk encryption uses keys that only the specific VM instance can access, providing defense against disk theft scenarios including unauthorized copies by privileged insiders or government agencies with legal access to datacenter infrastructure

### Supported disk types and configuration

Confidential OS disk encryption: The OS disk uses a vTPM-bound key. This provides the strongest protection but limits some platform features like disk snapshots and some backup solutions that require offline disk access.

Confidential OS disk encryption with customer-managed key: The vTPM-bound key is itself encrypted with a customer-managed key stored in Azure Key Vault. This configuration gives you organizational control over the root key while maintaining vTPM binding for the active encryption key.

For data disks attached to confidential VMs, use encryption at host. Confidential disk encryption doesn't extend to data disks, but encryption at host on a confidential VM provides comprehensive protection: hardware memory encryption (SEV-SNP) + vTPM-bound OS disk + encrypted data disks and temp storage.

---
### Create a confidential VM with confidential disk encryption
---
```bash
az vm create \
--resource-group contoso-research-rg \
--name formula-vault-vm-01 \
--image Ubuntu2204 \
--size Standard_DC4as_v5\
--security-type ConfidentialVM\
--os-disk-security-encryption-type DiskWithVMGuestState\
--enable-vtpm true \
--enable-secure-boot true\
--encryption-at-host true \
--admin-username azureuser \
--generate-ssh-keys
```
### Limitations and considerations

Confidential VMs with confidential disk encryption have specific operational constraints:

No disk snapshots: Because the encryption key binds to the vTPM, you can't create snapshots of confidential OS disks using standard Azure snapshot tools. Plan backup strategies using application-level backups or Azure Backup solutions that support confidential VMs.

No cross-VM disk movement: You can't detach a confidential OS disk from one VM and attach it to another. The vTPM binding ensures only the original VM can decrypt the disk.

Regional availability: Confidential VMs require specific hardware. Verify your target Azure region supports DCasv5 or ECasv5 series before planning deployments.

Size selection: Confidential VM sizes typically cost more than equivalent standard sizes due to specialized hardware. Balance security requirements against budget constraints.

### When to use confidential disk encryption

Regulatory protection against privileged access: Compliance frameworks like ITAR (International Traffic in Arms Regulations) or specific healthcare regulations can require technical controls preventing cloud provider access to sensitive data.

Defense against physical theft: Industries with extremely high-value intellectual property (pharmaceutical research, defense applications, financial algorithms) benefit from hardware-enforced protection that remains effective even if encrypted disks fall into adversary possession.

Attestation requirements: Applications needing cryptographic proof that they run in an uncompromised environment use the attestation capabilities built into confidential VMs.

Zero Trust architecture: Organizations implementing Zero Trust principles apply confidential VMs to protect high-value assets from both external attackers and insider threats, reducing the trusted perimeter to only the workload itself

### Q&A
Which disk encryption approach is recommended for new Azure virtual machines and provides end-to-end encryption that covers temp disks, disk caches, and data flows to storage?
> Encryption at host

You need to use your organization's own keys to encrypt Azure VM managed disks, with the keys managed in Azure Key Vault. What Azure resource must you create to associate the customer-managed key with the VM's disks?
>Disk Encryption Set

A confidential VM uses which mechanism to bind disk encryption keys, ensuring that protected disk content is accessible only to that specific VM?
>The VM's virtual Trusted Platform Module (vTPM)


## Identify Trusted Launch components and VM security types

<img width="918" height="264" alt="image" src="https://github.com/user-attachments/assets/f5cdf0d6-ed19-4323-899f-9173eaba274f" />

### Compare VM security types

Standard security represents the traditional VM configuration with no built-in boot integrity protection. Gen1 virtual machines use Standard security by default and can't be upgraded to other security types without first migrating to Gen2. Standard VMs rely entirely on operating system and application-layer security controls—the boot process remains unverified and unmonitored.

Trusted Launch security provides boot integrity verification through Secure Boot, vTPM, and integrity monitoring. This security type is now the default for new Gen2 virtual machines created in the Azure portal. Trusted Launch VMs measure and verify the boot process but don't encrypt memory or provide confidential computing guarantees.

Confidential VM security extends Trusted Launch protections by adding memory encryption and isolation using AMD SEV-SNP or Intel TDX technology. Confidential VMs protect data in use, not just at boot time. This security type requires specific VM sizes and incurs higher compute costs.

<img width="1383" height="1376" alt="image" src="https://github.com/user-attachments/assets/1ae4878b-bc73-48b8-8573-979f9c74472f" />

**Secure Boot**: This protection blocks unsigned or maliciously modified boot components from executing. Rootkits and boot kits that modify the boot loader, kernel, or early-loading drivers fail signature validation and can't compromise the system

### Virtual Trusted Platform Module (vTPM)
The virtual Trusted Platform Module provides a dedicated, hardware-backed secure vault for each VM. The vTPM is fully compliant with TPM 2.0 specifications and operates independently from the guest operating system.

These measurements are stored in Platform Configuration Registers (PCRs) inside the vTPM. Because the vTPM is isolated from the guest OS, malware can't tamper with the measurements. The cryptographic record provides irrefutable evidence of what code executed during boot.
The vTPM enables remote attestation, a process where the VM cryptographically proves to an external verifier that it booted with authorized components. 

### Integrity monitoring
Integrity monitoring connects the vTPM measurements to Microsoft Defender for Cloud by installing the Guest Attestation extension on the VM. This extension continuously retrieves boot measurements from the vTPM and sends them to Azure's attestation service for validation.

When boot integrity fails—for example, if Secure Boot is disabled or an unmeasured component loads—the attestation service detects the mismatch between actual and expected measurements. Defender for Cloud generates a security alert that appears in the Azure portal and triggers any configured alert actions such as email notifications or Logic App workflows.

Integrity monitoring transforms the vTPM's local measurements into actionable security intelligence. Without this component, boot integrity failures would remain invisible to security operations teams.

**The three Trusted Launch components create a layered defense against boot-level threats. Secure Boot prevents unauthorized code from executing. vTPM measures what executed during boot. Integrity monitoring surfaces failures to security teams.**

Trusted Launch is now the default security type for new Gen2 virtual machines created through the Azure portal. When you create a Gen2 VM, Secure Boot and vTPM are enabled by default.

**Gen1 virtual machines use Standard security and require migration to Gen2 before Trusted Launch can be enabled. Azure doesn't support upgrading a Gen1 VM to Trusted Launch while remaining Gen1—you must migrate the VM to Gen2 architecture first.**

### Trusted Launch on new and existing Gen2 VMs

<img width="910" height="202" alt="image" src="https://github.com/user-attachments/assets/47a978fb-d7f7-493d-882c-2ac7375bc26d" />

<img width="1088" height="431" alt="image" src="https://github.com/user-attachments/assets/e21faedb-4743-4a32-a401-3dbcf77e617e" />

### Upgrade an existing Gen2 VM to Trusted Launch
 If the VM uses Azure Backup, verify that it uses an Enhanced backup policy rather than a Standard backup policy. 


### Migrate a Gen1 VM to Gen2 with Trusted Launch

Gen1 virtual machines require migration to Gen2 architecture before Trusted Launch can be enabled
Azure doesn't support migrating Gen1 to Gen2 without enabling Trusted Launch—the two changes must happen together.

Gen1 VMs use a Main Boot Record (MBR) disk layout, but Gen2 requires GUID Partition Table (GPT) with an Extensible Firmware Interface (EFI) system partition. On Windows, use the built-in MBR2GPT.exe utility to convert the disk layout before initiating the upgrade. Linux VMs require equivalent GPT conversion steps.

**Windows Server 2016 doesn't include MBR2GPT.exe and isn't supported for the Gen1 to Trusted Launch upgrade path. If your VM runs Windows Server 2016, first perform an in-place OS upgrade to Windows Server 2019 or 2022, then run MBR2GPT conversion. Azure Linux and Debian are also excluded from the Gen1 to Trusted Launch upgrade path.**

**The Gen1 to Trusted Launch upgrade can't be rolled back to Gen1 configuration. If rollback is needed, you must restore from a backup or restore point taken before the upgrade**



### Configure individual security components

<img width="843" height="623" alt="image" src="https://github.com/user-attachments/assets/4485ced9-bb58-4618-ac0f-5aa29f68c8fa" />

### Enforce Trusted Launch adoption with Azure Policy

<img width="910" height="336" alt="image" src="https://github.com/user-attachments/assets/d9ab41df-be97-485f-901e-b9e4039c1a05" />


### Use the Trusted Launch eligibility policy

The "Disks and OS image should support TrustedLaunch" policy identifies Gen2 virtual machines that have compatible operating systems and VM sizes for Trusted Launch upgrade. This eligibility assessment policy evaluates each VM's current configuration against Trusted Launch requirements without making assumptions about intent.

The policy checks:

VM generation (must be Gen2)
Operating system compatibility (current OS supports Trusted Launch)
VM size compatibility (current size supports Trusted Launch security features)
VMs that meet all criteria appear as Compliant in the policy compliance report. VMs that fail one or more checks appear as Non-compliant. The policy uses an Audit effect, which means it evaluates and reports compliance but doesn't block VM creation or modification.



### Use the Trusted Launch security configuration policy
The "Virtual Machine should have TrustedLaunch enabled" policy audits whether VMs actually have Trusted Launch configured as their security type. This policy focuses on security configuration rather than eligibility—it identifies VMs that could have Trusted Launch but currently use Standard security.

The policy evaluates the VM's security type property. VMs with security type set to TrustedLaunch appear as Compliant. VMs with security type set to Standard appear as Non-compliant.

This policy supports two effects:

Audit: Evaluate and report compliance without blocking actions
Disabled: Don't evaluate the policy
In Audit mode, the policy evaluates VMs and reports noncompliance but doesn't block VM creation or modification. This allows you to identify which VMs in your estate are using Standard security and are candidates for upgrade, without disrupting ongoing deployments.


**Critical consideration: VMs must be stopped and deallocated for remediation to succeed. Remediation tasks don't automatically stop running VMs—if the VM is running when the task executes, remediation fails.**


Q&A 
A security engineer notices that a Gen2 VM has Trusted Launch enabled as its security type, but Microsoft Defender for Cloud isn't generating boot integrity alerts even after a suspected boot kit infection. Which Trusted Launch component is most likely not configured correctly?
>Integrity monitoring

An administrator needs to enable Trusted Launch on an existing Generation 1 virtual machine that runs a supported Windows Server workload. What is the correct approach
>Migrate the VM to Generation 2 using the in-place upgrade process. The process converts the disk from Main Boot Record (MBR) to GUID Partition Table (GPT) and updates the firmware to UEFI.

A compliance team wants to ensure that all new Gen2 VMs deployed to a subscription are created with Trusted Launch enabled. They also want to identify existing Gen2 VMs that are eligible but not yet configured. Which combination of Azure Policy effects achieves both goals?
>Apply the Audit effect to the eligibility policy and the Deny effect to the configuration policy




## Implement Azure Bastion

**Azure Bastion provides browser-based and native client connectivity through encrypted TLS 443 connections, eliminating the need for public IP addresses on virtual machines and open management ports in network security groups.** RDP or SSH session from the Bastion host directly to the target VM over the Azure backbone network.

### Bastion SKU

<img width="906" height="490" alt="image" src="https://github.com/user-attachments/assets/020ee1a3-dfc7-43cb-b144-6db3e24d3b4c" />

he Developer SKU provides no-cost access for individual users during development and testing. This tier supports a single concurrent user and lacks scaling capabilities, making it unsuitable for production team environments.

The Basic SKU offers browser-based access through its fixed allocation of 2 host units, supporting up to 40 concurrent RDP sessions or 80 concurrent SSH sessions. Users connect through the Azure portal using the browser-based RDP or SSH client. This tier works well for small teams that don't require native RDP client features or significant scaling.

The Standard SKU adds native client support, IP-based connections, and configurable scaling from 2 to 50 host units. Native client support allows users to connect using their existing RDP clients (such as Microsoft Remote Desktop or Windows App) and SSH tools through an Azure CLI tunnel command. IP-based connections enable administrators to connect to virtual machines by specifying an IP address rather than selecting from the portal's VM list, which proves essential for hub-spoke architectures where Bastion in the hub virtual network provides access to spoke VNets through peering. Shareable links provide time-limited URL-based access for users without Azure portal credentials, useful for contractor or vendor scenarios.

The Premium SKU adds session recording capabilities that capture complete RDP and SSH session activity to an Azure Storage account for compliance and security auditing. Private-only mode removes the public IP address requirement from the Bastion host itself for air-gapped or highly restricted network environments.

### network architecture

Azure Bastion deploys into a dedicated subnet within an Azure virtual network. This subnet must be named exactly AzureBastionSubnet and sized with a minimum /26 prefix (64 IP addresses). The /26 minimum ensures sufficient address space for the Bastion service infrastructure and future scaling requirements.

Organizations with hub-spoke network topologies face a deployment choice: deploy Bastion in the hub virtual network to serve all spoke VNets, or deploy separate Bastion instances in each spoke virtual network. A hub deployment reduces costs (one Bastion host instead of many) and simplifies management, but requires Standard or Premium SKU with IP-based connection capability. The Bastion host in the hub connects to VMs in spoke VNets through VNet peering relationships. With IP-based connections enabled, administrators specify the target VM's IP address directly rather than selecting from the portal's resource list.

Per-VNet deployment provides isolation and removes dependencies on virtual network peering, but multiplies costs and administrative overhead. This approach makes sense for VNets with strict isolation requirements or when different teams manage separate VNets independently.

**Azure Bastion requires a Standard SKU static public IP address for inbound connectivity from user clients.**

The AzureBastionSubnet requires a network security group with specific inbound and outbound rules. Inbound rules must allow HTTPS (port 443) from the internet to permit user connections, and allow port 443 from the GatewayManager service tag to enable Azure control plane operations. Outbound rules must allow traffic to target VMs on ports 3389 (RDP) and 22 (SSH), and allow port 443 to the AzureCloud service tag for Azure service dependencies.

These NSG requirements differ from typical subnet NSGs because they accommodate both user-facing traffic (inbound 443 from Internet) and backend service communication (GatewayManager, AzureCloud). Restrictive NSGs that block these required flows prevent Bastion from functioning.

### Deploy and Configure Azure Bastion

The deployment process requires a dedicated subnet named exactly AzureBastionSubnet with a minimum /26 address prefix. This subnet name is case-sensitive and mandatory—the Bastion deployment fails if the subnet has any other name.



```bash
az network bastion tunnel
```

inbound rules:

<img width="897" height="263" alt="image" src="https://github.com/user-attachments/assets/e42eb267-958b-4fde-b400-675efd85bbf8" />

outbound rules: 

<img width="914" height="256" alt="image" src="https://github.com/user-attachments/assets/7ba5a744-dfc7-40b2-9db0-8b996510331f" />


### Connect to VMs through Azure Bastion
```bash
az extension add --name bastion
```

after : 

```bash
az network bastion tunnel \
 --name bastion-hub-prod\
 --resource-group rg-network-prod\
--target-resource-id /subscriptions/<subscription-id>/resourceGroups/<vm-rg>/providers/Microsoft.Compute/virtualMachines/<vm-name> \
 --resource-port 3389\
 --port 50001\
```

For ssh connections , 
```bash
az network bastion ssh
--name bastion-hub-prod\
--resource-group rg-network-prod\
--target-resource-id /subscriptions/<subscription-id>/resourceGroups/<vm-rg>/providers/Microsoft.Compute/virtualMachines/<vm-name> \
 --auth-type password\
 --username azureuser
```


 ### use shareable links

The portal generates a URL that provides direct browser-based access to the VM through Bastion.

Configure the shareable link properties:

Expiration time: Set how long the link remains valid (maximum 90 days)
Allowed credentials: Optionally restrict which VM credentials can authenticate through the link
Description: Add notes about the link's purpose for audit records
Share the generated URL with the intended recipient. When they open the URL, they see a connection form requesting VM credentials. After authentication, the browser establishes a Bastion session identical to the standard browser-based connection experience.


Revoked links immediately stop working, terminating any active sessions using those links.


Q&A 

A security engineer needs to deploy Azure Bastion to support native RDP client connections and IP-based connections to VMs, but doesn't require session recording. Which SKU meets these requirements at the lowest cost?
>Standard


What is the minimum subnet size required for the AzureBastionSubnet in an Azure virtual network?
>/26

An organization wants to provide a contractor with browser-based RDP access to a specific development VM, but the contractor doesn't have an Azure portal account. Which Azure Bastion feature enables this without granting Azure access?
>Shareable link

### SKU selection balances capability requirements against cost:

Developer: Individual use, no host unit charges, browser-only
Basic: Small teams, fixed two host units, browser-only, no native client
Standard: Enterprise scale, configurable host units, native client support, IP-based connections
Premium: Compliance environments add session recording and private-only mode


---






## Manage Security for Arc-Enabled hybrid servers

Use RBAC to control Arc server management.

Azure provides two specialized built-in roles for Arc-enabled server management. 

The Azure Connected Machine Onboarding role grants the minimum privilege needed to connect servers to Azure Arc. Users or service principals with this role can create new Arc server resources but can't reonboard, delete, or modify existing resources. 

The Azure Connected Machine Resource Administrator role grants full control over Arc-enabled servers, including the ability to deploy and remove extensions. Because extensions run with elevated privileges on the target machine, this role effectively grants root or administrator access to the underlying server.








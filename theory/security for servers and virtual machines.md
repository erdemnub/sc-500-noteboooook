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


## Onboard servers to Defender for Servers

<img width="908" height="661" alt="image" src="https://github.com/user-attachments/assets/5851779e-5b51-43ff-a319-ace22eadd390" />

Plan 1 provides essential protection when Microsoft Defender for Endpoint integration and agent-based vulnerability assessment meets your needs. Organizations with cost constraints or simple server workloads often start with Plan 1. The plan delivers core endpoint detection and response (EDR) capabilities and continuous vulnerability scanning through the MDE sensor installed on each machine.

Plan 2 adds agentless scanning, which analyzes server disk contents without requiring agent deployment or consuming machine resources. With agentless scanning, you gain software inventory, vulnerability assessment, secrets scanning, and malware detection capabilities that run offline every 24 hours. Plan 2 also unlocks just-in-time VM access for securing management ports, File Integrity Monitoring for detecting unauthorized changes, and OS configuration assessment through Machine Configuration.

30-day trial period

You can override the subscription-level plan setting at the resource group or individual resource level when different environments require different protection tiers. For example, you might apply Plan 1 to development and test resource groups while maintaining Plan 2 for production workloads. 

For Plan 2 subscriptions, agentless scanning activates automatically and begins its 24-hour scanning cycle. The first agentless scan completes within 24 hours of enablement, after which scans repeat on a daily schedule

### vulnerability scanning with Defender Vulnerability Management

Defender for Servers provides two complementary scanning methods

<img width="912" height="281" alt="image" src="https://github.com/user-attachments/assets/49c58784-113c-4315-a41f-f3beff7739dd" />

### Use agent-based vulnerability scanning for real-time detection

Agent-based vulnerability scanning runs through the Microsoft Defender for Endpoint sensor installed on each protected machine. The sensor continuously monitors installed software, compares it against Microsoft's vulnerability intelligence database, and reports findings in near real-time. When a new vulnerability disclosure affects software running on your servers, the agent detects it within minutes and surfaces the finding in Defender for Cloud.

The agent-based approach provides the fastest detection because the sensor runs locally on each machine and doesn't depend on scheduled scans. 

Agent-based scanning is available with both Plan 1 and Plan 2, making it the baseline vulnerability assessment method for all Defender for Servers deployments

### Use agentless scanning to eliminate agent deployment overhead

Agentless vulnerability scanning takes a fundamentally different approach. Instead of installing software on your VMs, agentless scanning creates a snapshot of each VM's disk, then analyzes that snapshot in an isolated Azure environment completely outside your virtual machine. The VM itself experiences no performance issues because the analysis happens on a copy of the disk data.

Here's how the technical process works. Once every 24 hours, Defender for Cloud takes a point-in-time snapshot of each running VM's disk. The snapshot process uses Azure's native snapshot capabilities, which complete in seconds without pausing or interrupting the VM. Defender for Cloud then mounts the snapshot disk in a secure analysis environment, scans the file system to build a software inventory, and compares installed packages against vulnerability databases. After analysis completes, the snapshot is immediately deleted. Your VM never knows the scan happened.

**Agentless scanning has technical limits based on disk size, disk count, and encryption type. The maximum total disk size that can be scanned is 4 TB, calculated as the sum of all attached disks. If a VM has six disks of 1 TB each (6 TB total), only the OS disk is scanned, provided the OS disk alone is under 4 TB. The maximum number of disks per VM is six—if a VM has more than six disks, the scan skips that VM entirely.**

**Disk encryption affects scan eligibility. Agentless scanning supports unencrypted disks, disks encrypted with SSE using platform-managed keys, and disks encrypted with SSE using customer-managed keys (CMK). However, certain disk types are unsupported: UltraSSD_LRS, PremiumV2_LRS, and AKS Ephemeral OS Disks can't be scanned because they use storage architectures incompatible with the snapshot-based scanning process.**

**Only running VMs are scanned. If a VM is powered off or deallocated when the 24-hour scan cycle starts, the scan skips that VM until the next cycle.**

Manage the Microsoft Defender for Endpoint integration

The sensor runs as a lightweight service on Windows and Linux machines, monitoring process behavior, network connections, and file system changes to detect suspicious activity in real-time.

The extension appears in the Azure portal with the name MDE.Windows on Windows VMs and MDE.Linux on Linux VMs.


 ### Configure agentless scanning capabilities for Plan 2

 


























## isolate Azure workloads using network security controls

 NSGs, ASGs (Application Security Group), Azure Virtual Network Manager, and Network Watcher

**Two default rules—AllowVNetInBound and AllowVNetOutBound—**

**The threat model consequence is clear: a compromised VM becomes a foothold to reach everything else in the virtual network. An attacker who gains access to one workload doesn't need to breach another perimeter**

###  lateral movement patterns in Azure

north-south  (the internet and your virtual network) traffic and east-west (traffic flowing between resources inside your virtual network) traffic.

Common Azure lateral movement paths:

**Web tier to app tier: The web VM sends application requests to backend services. If compromised, the same path lets an attacker probe the app tier for vulnerabilities.**

**App tier to database tier: Application servers need database access. A compromised app server can exfiltrate data, modify records, or escalate privileges using database credentials.**

**VM to Azure platform services: VMs in a virtual network can reach Azure SQL, Azure Storage, and Key Vault if those services are accessible on public endpoints or connected via service endpoints. A compromised VM inherits those permissions.**

**VM to VM on administrative ports: If remote desktop protocol (RDP) on port 3389 or secure shell (SSH) on port 22 are open between subnets, an attacker can spread by brute-forcing credentials or exploiting unpatched services.**

Network segmentation and NSG

### network segmentation
attack surface

Are workloads with different trust levels sharing a subnet? Production and development VMs shouldn't coexist in the same subnet without controls. A compromised dev VM becomes a path to production resources.

Are NSGs attached at the subnet level or the network interface (NIC) level? Resources without an associated NSG inherit the default-allow virtual network rules.
What are the effective security rules on each subnet? Azure evaluates NSG rules in priority order. If no deny rules exist between tiers, the default-allow rules permit all traffic.

Are administrative ports open between tiers? RDP (3389), SSH (22), and SQL Server (1433) are common targets. If these ports are accessible from workloads that don't need them, you created an attack path.

Are PaaS services exposed on public endpoints? Azure SQL Database, Azure Storage, and Key Vault are accessible from the internet by default. If they're not protected by firewall rules or private endpoints, any resource with outbound internet access can reach them.

**The goal is to enforce a default-deny stance**

Network Watcher's effective security rules 

<img width="705" height="414" alt="image" src="https://github.com/user-attachments/assets/747b3eaa-1629-4063-98eb-30385ed61ac6" />


The critical-risk path is the web-to-database connection. An attacker who compromises the web tier can exfiltrate customer data, modify records, or use SQL Server stored procedures to execute code on the database server

### Control traffic with network security groups











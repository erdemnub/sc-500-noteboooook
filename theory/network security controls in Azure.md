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

Each NSG rule specifies a source, destination, protocol, port range, and action (Allow or Deny).

**Every NSG includes default rules that you can't remove. These defaults allow traffic within the virtual network (AllowVNetInBound at priority 65000), allow traffic from Azure's load balancer (AllowAzureLoadBalancerInBound at priority 65001), and deny all other inbound traffic (DenyAllInBound at priority 65500)**


A deny rule at priority 100 blocks traffic before Azure evaluates the allow-rule at priority 200. This means you should reserve the 100-200 priority range for critical deny rules that enforce hard security boundaries.



### rule properties to enforce least privilege

Each NSG rule requires six properties that together determine what traffic to allow or deny.

Source and destination accept an IP address, CIDR (Classless Inter-Domain Routing) range, service tag, or application security group. Service tags like AzureLoadBalancer, Internet, and virtual network let you scope rules without managing IP address lists. For example, using the AzureLoadBalancer tag as the source allows traffic from Azure's infrastructure load balancer (168.63.129.16) without hard-coding that IP address.

Protocol specifies TCP, UDP, or Any. Be specific—choosing "Any" creates unnecessary risk. If your application uses TCP port 1433 for SQL Server connections, the rule should specify TCP, not Any.

Port accepts a specific port number (such as 1433, 22, 443, or 3389) or a range. Avoid broad ranges like 0-65535 in allow rules because they undermine the principle of least privilege. With augmented security rules, you can specify multiple ports, source addresses, or destination addresses in a single rule using comma-separated values, which reduce your total rule count.

Action is either Allow or Deny. Use Deny rules to create hard security boundaries and Allow rules to open specific, documented pathways.

Priority ranges from 100 to 4096, with lower numbers evaluated first. Leave gaps between priority numbers so future administrators can insert rules without renumbering your entire ruleset.


### common NSG misconfigurations

Allow Any → Any rules nullify the security benefit of NSGs entirely. If you have a documented business reason for allowing all traffic, the environment probably doesn't need NSG-level segmentation at all.

**if your critical deny rule is at priority 200 and your allow rule is at priority 300, someone can insert an allow rule at priority 150 that bypasses your deny rule. Reserve the 100-199 range for deny rules and start allow rules at 200 or higher.**


for example : 

100-199   Critical Deny

200-499   Allow Business Traffic

500-999   Admin / Management

1000+     Special Rules


NSG flow logs feed into tools like Microsoft Defender for Cloud and Microsoft Sentinel to detect anomalies.

The least-privilege principle applies to NSG rules: if you don't have a documented business reason for a rule, it shouldn't exist. Review your ruleset quarterly and remove rules that no longer serve active workloads.

IP-based CIDR rules work in stable environments, but as workloads scale and IPs change, maintaining accurate source and destination CIDRs becomes error-prone.


Stale rules create security risks. A rule that was accurate last quarter can permit traffic from VMs that teams repurposed—or can block VMs that now need access. Manual tracking doesn't scale, and IP-based documentation becomes outdated in the moment documented.


ASG (application security groups)

**The NSG rule references the ASG as a source or destination. When a NIC joins the ASG, traffic from that NIC automatically inherits the rules that reference that ASG. When you remove a NIC from the ASG, the rules no longer apply to that NIC's traffic—no rule edits required.**

**ASGs don't replace NSGs. They make NSG rule sources and destinations easier to maintain. The NSG still enforces the rules; ASGs provide a dynamic, membership-based way to identify traffic sources and destinations.**



tip 

>Associate all NICs to the appropriate ASG before updating NSG rules to reference those ASGs. An NSG rule that references an ASG only applies to NICs that are members of that ASG—if you update the rule first, traffic can be blocked until NIC associations are complete.


Before (CIDR-Based)

<img width="690" height="207" alt="image" src="https://github.com/user-attachments/assets/d5976730-bf1b-4cc3-86a1-d19601e07db0" />


After (ASG-based)

<img width="704" height="198" alt="image" src="https://github.com/user-attachments/assets/238772de-d00a-4c15-af9e-4be6352120c3" />

**ASGs also support complex scenarios. A single NIC can be a member of multiple ASGs, allowing you to grant different types of access based on overlapping group memberships. ASGs can serve as both source and destination in the same rule, as long as both ASGs are in the same virtual network.**

### ASG scope and limitations
ASGs simplify rule management, but they have scope limits. ASGs are scoped to a single virtual network—you can't use an ASG created in VNet-A as a source in an NSG rule applied to resources in VNet-B. ASGs also don't span subscriptions for NSG rule purposes

**ASGs don't create subnets or change routing. They only affect NSG rule matching. An ASG has no challenge on traffic flow unless an NSG rule references it. Without the rule, the ASG is just an empty container.**

**Azure enforces limits on ASG usage. Each subscription supports up to 3,000 ASGs, and each NIC supports up to 20 ASG associations**






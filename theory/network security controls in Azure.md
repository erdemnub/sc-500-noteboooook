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



### policy with Azure Virtual Network Manager

Azure Policy can restrict NSG rule creation by denying rules that match certain patterns, but it operates on resource creation events, not on the effective network posture in real time.

Azure Virtual Network Manager (AVNM) addresses this gap directly. AVNM lets you define network policy at the organization or management group level and push it to virtual networks across subscriptions. Unlike NSGs, which any resource owner can modify, AVNM configurations require centralized administrative permissions and override local network rules.

### How Azure Virtual Network Manager works

AVNM scope defines which management groups or subscriptions the AVNM instance manages. An AVNM instance deployed at the management group level can govern all child subscriptions, giving you organization-wide control without deploying separate instances per subscription.

Network groups are logical collections of virtual networks managed by AVNM. You add virtual networks (VNets) to a network group either manually or using Azure Policy conditional expressions. Think of network groups as application security groups for virtual networks—they let you target policies at sets of networks rather than configuring each virtual network individually.

Configurations define what you apply to a network group. AVNM supports two configuration types: connectivity configurations create hub-spoke or mesh topologies, while security admin configurations enforce traffic rules. For security engineering, security admin configurations are the critical tool.

Security admin rules are rules in a security admin configuration that apply before any NSG rules are evaluated. They can Allow, Always Allow, or Deny traffic, regardless of what NSG rules exist on the affected resources. This evaluation order is what gives AVNM its enforcement power.

### Security admin rules vs. NSG rules

<img width="687" height="428" alt="image" src="https://github.com/user-attachments/assets/cc3ecc9b-253c-482a-941e-ea6230dcab10" />

The security consequence of this design is powerful: a security admin rule with the Deny action on port 3389 from the Internet service tag blocks all inbound RDP from the internet on every virtual network in the network group, even if a team has an NSG Allow rule on port 3389. The admin rule wins.


Create a network group and security admin rule
1. Create an AVNM instance:
2. Create a network group:
3. Create a security admin configuration:
4. Add a rule collection
5. Add a security admin rule
Name: deny-rdp-from-internet
Priority: 100 (lower numbers are evaluated first)
Protocol: TCP
Source: Service tag Internet
Source port: Any (*)
Destination: Any
Destination port: 3389
Action: Deny

6.Deploy the configuration:

**Deploying a security admin rule is a network-wide change that takes effect immediately on all VNets in the network group. Test configurations in a nonproduction network group before deploying to production VNets. Changes to security admin configurations don't take effect until you explicitly deploy them.** 


### When to use AVNM security admin rules

Block dangerous ports organization-wide: Use Deny rules to block RDP from the internet (port 3389), SSH from the internet (port 22), and unencrypted management protocols.

**Always Allow for required management traffic: If your organization uses Azure Bastion for management access, create an Always Allow rule for Bastion subnet traffic so teams can't accidentally block management connectivity with their own NSGs. This pattern also works for centralized monitoring agents, backup services, and other infrastructure services that must reach all workloads.**

Standard port enforcement: If your organization has a policy that all internal services communicate on specific approved ports only, security admin rules enforce that requirement at the platform level. Teams can still create NSG rules for application-specific traffic, but the baseline port policy remains immutable.


### Verify effective network security rules with Network Watcher












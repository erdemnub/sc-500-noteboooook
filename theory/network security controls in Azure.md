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

**How do we know these controls are actually working together as intended?**

Azure evaluates network traffic against multiple layers of security controls in a specific sequence: AVNM security admin rules first, then subnet NSG rules, then NIC NSG rules. A single virtual machine (VM) might have three or more rule sources affecting its traffic, and the effective security rules are the aggregated result of all these layers.


### Network Watcher to view effective security rules

<img width="702" height="318" alt="image" src="https://github.com/user-attachments/assets/2c1e1807-b8af-402c-a099-5b1a01943691" />



Network Watcher's effective security rules tool shows all NSG rules affecting a specific NIC, aggregated, and sorted by evaluation order. 

You access this tool through the Azure portal in two ways: navigate to Network Watcher → NSG diagnostic, or go directly to a VM's NIC screen and select Effective security rules. 

The output shows : 
Source: which configuration the rule comes from (AVNM security admin configuration, subnet NSG, or NIC NSG)
Priority: the rule's priority within its rule set (lower numbers evaluate first)
Action: Allow or Deny
Port range and protocol: what traffic the rule matches
Inheritance indicator: whether the rule was inherited from a subnet NSG, applied at the NIC level, or enforced via AVNM

The result displays Allow or Deny along with the specific rule name that made the decision. This pinpoints exactly which rule in your multi-layer configuration is controlling each traffic path.


Important:
**IP flow-verify evaluates NSG rules and security admin configurations based on their configuration and doesn't require the Network Watcher agent VM extension to be installed on the VM. The extension is required for features such as packet capture and Connection Monitor, not for IP flow verify.**


### Handle unexpected verification results

When IP flow-verify returns unexpected results, the effective security rules view helps you diagnose the problem. If a path is denied that should be allowed, check for priority conflicts. A broad deny rule with a lower priority number (which evaluates earlier) might catch the traffic before your allow-rule processes it.

Verify that application security groups (ASGs) are correctly associated with NICs. Rules that reference an ASG only apply to NICs that are members of that ASG. A NIC that isn't assigned to the referenced ASG is excluded from those rules, even if the rule configuration appears correct.

If a path is allowed that should be denied, look for missing deny rules. Azure's default rules include AllowVNetInBound, which permits traffic between resources in the same virtual network unless you explicitly deny it. Check the AVNM deployment status—if you created a security admin configuration but haven't deployed it to the target network group, those admin rules don't take effect.

When you identify the rule causing unexpected behavior in the effective security rules list, you have several remediation options. Adjust rule priorities to ensure evaluation order matches your intent. Add explicit deny rules to block traffic that default allow rules currently permit. Verify ASG assignments on NICs. Deploy pending AVNM configurations to activate admin rules.

### Complete the verification workflow 

The results confirm all three layers of their security controls appear correctly:

AVNM admin rule: deny-rdp-from-internet (priority 100, source: Internet, port: 3389, action: Deny) ✓
Subnet NSG rule: deny-web-to-db-1433 (priority 100, source: asg-web-tier, destination port: 1433, action: Deny) ✓
Subnet NSG rule: allow-app-to-db-1433 (priority 200, source: asg-app-tier, destination port: 1433, action: Allow) ✓


then runs IP flow verify for each critical path identified in their security assessment:

Web VM → Database VM, port 1433: Deny—blocked by rule deny-web-to-db-1433 ✓
App VM → Database VM, port 1433: Allow—permitted by rule allow-app-to-db-1433 ✓
Internet → Any VM, port 3389: Deny—blocked by AVNM admin rule deny-rdp-from-internet ✓


Q&A

Company's security team discovers that a compromised web-tier VM can initiate connections directly to the database tier on port 1433. No NSG is attached to the database subnet. What is the most effective first step to close this lateral movement path?

-Create and attach an NSG to the database subnet with a deny-all inbound rule, then add an allow rule for port 1433 scoped to the web tier only.

A team uses IP-based NSG rules to control access between 40 application-tier VMs and 20 database-tier VMs. When new VMs are added, rules frequently break because IP addresses change. What change resolves this maintenance problem while preserving the security boundary?

-Use application security groups to group application-tier and database-tier VMs, then write NSG rules referencing the ASGs instead of individual IPs.


Company wants to ensure that no team in any subscription can create an NSG rule that allows RDP (port 3389) inbound from the internet. They want to block this even if they have Owner permissions on their subscription. Which Azure Virtual Network Manager capability enforces this?

-A security admin rule with action Always Deny on destination port 3389 from source Any applied to a network group covering all subscriptions.

**While NSGs and ASGs control lateral movement within your network, Azure DDoS Protection provides defense against volumetric attacks from the internet. Consider enabling DDoS Protection to complement your network segmentation strategy.**

## Centralize and enforce traffic inspection using Azure Firewall

<img width="690" height="302" alt="image" src="https://github.com/user-attachments/assets/0857e869-8cf4-44a6-964d-8369c328b83a" />

### What NSGs can and can't do

NSGs evaluate traffic based on source IP, destination IP, protocol, and port number. They operate at Layer 3 and Layer 4 of the network stack and apply rules in a stateless manner to each packet.

NSGs excel at basic network segmentation. You can block or allow specific ports and IP ranges, use service tags to represent Azure services without memorizing IP addresses, and group virtual machines with application security groups (ASGs) for dynamic rule management. An NSG rule can permit HTTPS traffic from your web tier to your database tier while blocking all other protocols.

An NSG can't distinguish between legitimate HTTPS traffic to api.github.com and malicious traffic to a command-and-control server on port 443. Because NSGs only see IP addresses, they can't block traffic to evil.example.com unless you manually add every IP that domain might resolve to. 



Five threat classes drive the need for Azure Firewall in enterprise environments.

<img width="682" height="597" alt="image" src="https://github.com/user-attachments/assets/4ae6ea94-5a32-4979-80c6-456f9a2f533d" />

**An NSG rule permitting outbound HTTPS doesn't distinguish between approved Azure OpenAI endpoints and an attacker-controlled API endpoint on the same port. With Azure Firewall application rules, you create an FQDN allow-list that permits contoso-openai.openai.azure.com while blocking all other destinations on port 443.**

Tip:
*Enable threat intelligence-based filtering in Alert and deny mode for production environments. Alert mode logs suspicious traffic without blocking it, which is useful during initial deployment but leaves your environment exposed.*


### Azure Firewall capabilities

Azure Firewall provides four types of rules, each addressing different threat scenarios.

Network rules filter traffic based on IP address, port, and protocol, similar to NSGs but with two advantages: they're stateful (tracking connection state reduces rule complexity), and they're managed centrally through Azure Firewall Policy. A network rule permitting outbound DNS on port 53 automatically allows return traffic without a separate inbound rule.

Application rules filter outbound HTTP and HTTPS traffic by fully qualified domain name (FQDN). These rules require the DNS proxy feature, which allows Azure Firewall to intercept DNS queries and resolve FQDNs before applying rules. An application rule can permit traffic to github.com and *.nuget.org while blocking all other outbound HTTPS, even though all destinations share port 443.

DNAT rules (destination network address translation) translate inbound traffic from a public IP to an internal resource. Unlike network and application rules that focus on outbound and east-west traffic, DNAT rules handle internet-to-resource scenarios. A DNAT rule can map your firewall's public IP on port 443 to an internal web server at 10.1.2.5:443.

Threat intelligence-based filtering blocks traffic to or from IP addresses and FQDNs associated with known malicious activity. This capability draws from Microsoft's Intelligent Security Graph, which aggregates threat signals from across Microsoft's global infrastructure. Threat intelligence operates in three modes: Off (disabled), Alert (log only), or Alert and deny (block and log). Unlike custom network or application rules that require manual updates, threat intelligence filtering updates automatically as Microsoft identifies new threats.

Azure Firewall also complements Azure Web Application Firewall (WAF), which protects inbound HTTP/S traffic against layer 7 threats like SQL injection and cross-site scripting. WAF sits at the ingress point on Application Gateway or Azure Front Door, while Azure Firewall controls outbound and east-west traffic. These controls work together: WAF defends public-facing applications, and Azure Firewall prevents compromised workloads from exfiltrating data or communicating with C2 infrastructure.


### Choose the right Azure Firewall SKU
Azure Firewall offers three SKUs: Basic, Standard, and Premium. All three provide stateful network filtering, but they differ significantly in threat intelligence enforcement capability and advanced inspection features.

<img width="686" height="416" alt="image" src="https://github.com/user-attachments/assets/f70004f0-b75e-48e3-924c-6c527dcc8393" />


**Transport Layer Security (TLS) inspection decrypts outbound HTTPS traffic, inspects the content, and re-encrypts it before forwarding. This capability addresses threats hidden in encrypted traffic, such as malware downloads over HTTPS or data exfiltration to legitimate cloud services. Regulated industries often require TLS inspection to meet compliance mandates, but it introduces certificate management complexity and potential privacy concerns**

Basic supports threat intelligence in alert-only mode—it logs suspicious traffic but can't block it. Standard and Premium can enforce threat intelligence in Alert and deny mode, actively blocking connections to known malicious IPs and domains. For production environments where blocking is required, Basic isn't a suitable choice.


IDPS analyzes network traffic patterns to detect and block exploits, malware propagation, and protocol violations. Unlike threat intelligence filtering, which relies on known-bad indicators, IDPS uses signature-based and anomaly based detection to identify suspicious behavior. IDPS operates in three modes: Off, Alert, or Alert and deny.

URL filtering extends application rules by inspecting the full URL path, not just the FQDN. Standard tier application rules permit or deny example.com entirely, but Premium URL filtering can allow example.com/api/* while blocking example.com/admin/*.

For most enterprise environments, Standard tier addresses the core threat classes outlined earlier. Premium tier becomes necessary when regulatory requirements mandate TLS inspection, when advanced threat detection justifies the extra cost, or when granular URL-level control is required.


### Configure Azure Firewall rules and policies

<img width="709" height="456" alt="image" src="https://github.com/user-attachments/assets/ac349ab5-a806-478f-92e1-a7f3da9fd9fe" />


Azure Firewall operates as a centralized enforcement point in a hub-spoke network topology. The firewall sits in a dedicated subnet within the hub virtual network (virtual network) and inspects traffic flowing between spokes, from spokes to the internet, and from the internet to spoke workloads.

The deployment requires a subnet named AzureFirewallSubnet in the hub virtual network. This subnet must be at least /26 in size, though Microsoft recommends /24 to accommodate future scaling. The firewall receives a private IP address from this subnet, which becomes the next-hop target for user-defined routes (UDRs) applied to spoke subnets.

With this hub-spoke pattern, you configure UDRs on each spoke subnet to route all internet-bound traffic (0.0.0.0/0) to the firewall's private IP address. Traffic from spoke workloads flows to the hub firewall, where Firewall Policy rules inspect and either allow or deny the connection. East-west traffic between spokes can also route through the hub firewall if you configure spoke-to-spoke UDRs for that purpose.

This architecture provides a single choke point for policy enforcement. Without the firewall, spoke virtual networks (VNets) would route directly to the internet through Azure's default system routes, bypassing centralized inspection and logging.

### Firewall Policy hierarchy

Azure Firewall supports two configuration methods: classic rules and Firewall Policy. Classic rules are stored directly on the firewall resource and can't be shared across multiple firewalls. Firewall Policy is a standalone Azure resource that supports rule reuse, policy inheritance, and integration with Azure Firewall Manager for centralized governance.

important : 

**Always use Firewall Policy for new deployments. Classic rules are a legacy option and don't support advanced features like rule collection groups, parent-child policy inheritance, or global policy management.**



Firewall Policy organizes rules into a four-level hierarchy:

Policy: The top-level resource (for example, policy-contoso-security)
Rule collection group: A container with a priority value (100, 200, 300, and so on). Lower numbers are evaluated first.
Rule collection: A set of rules with a shared action (Allow or Deny) and priority within the group
Rules: Individual traffic-matching criteria (source IP, destination FQDN, port, protocol)



---


Firewall Policy supports three rule collection types, and Azure Firewall evaluates all traffic in a fixed priority order:

Threat intelligence rules: highest priority, evaluated before all custom rules. When enabled in Alert and deny mode, threat intelligence can block traffic before any DNAT, network, or application rule is evaluated.
Destination network address translation (DNAT) rule collection: Translates inbound public IP addresses to private IP addresses for workloads behind the firewall
Network rule collection: Filters traffic by IP address, protocol, and port (stateful inspection)
Application rule collection: Filters outbound traffic by fully qualified domain name (FQDN) using HTTP/HTTPS inspection



The first step is to enable the DNS proxy on the Firewall Policy. Application rules rely on FQDN filtering, which requires the firewall and clients to resolve domain names to the same IP address. When DNS proxy is enabled, the fire

---

To configure Firewall Policy and rule collections in the Azure portal:

Deploy Azure Firewall by selecting the hub virtual network, creating the AzureFirewallSubnet (minimum /26), choosing Firewall Policy (create new: policy-contoso-security), and selecting the appropriate SKU (Standard for most scenarios, Premium for Transport Layer Security (TLS) inspection and intrusion detection).

Open the Firewall Policy resource (policy-contoso-security) and navigate to DNS Settings. Enable DNS Proxy and configure custom DNS servers if your environment uses private DNS zones.

Create an application rule collection to allow approved outbound HTTPS traffic:

Collection name: allow-approved-outbound
Priority: 200
Action: Allow
Rule 1 (Microsoft 365 access): Source = spoke virtual network CIDR ranges, destination FQDNs = use the WindowsVirtualDesktop FQDN tag or list specific Microsoft 365 endpoints, protocol = HTTPS:443
Rule 2 (Azure OpenAI access for AI agents): Source = AI agent subnet CIDR, destination FQDN = *.openai.azure.com, protocol = HTTPS:443
Create a network rule collection to block inbound management ports:

Collection name: deny-mgmt-ports-inbound
Priority: 100
Action: Deny
Rule: Source = Any, destination = spoke virtual network ranges, protocol = TCP, destination ports = 3389, 22




Azure AI agents running in spoke VNets make outbound HTTPS calls to model endpoints. Network security groups (NSGs) see these requests as ordinary port 443 traffic and can't distinguish between approved and unapproved AI services. Firewall application rules solve this problem by filtering on FQDN. Configure an application rule that allows only *.openai.azure.com from the AI agent subnet. The firewall's default-deny stance blocks any AI endpoint not explicitly listed, preventing agents from calling unauthorized external APIs or exfiltrating data through model interactions.


**Network rules block high-risk inbound management ports even if Azure Virtual Network Manager (AVNM) policies fail. Application rules allow only approved FQDNs for outbound HTTPS, and the stance to deny by default blocks everything else.**


### enable threat intelligence filtering 

Threat intelligence supports three modes:

Off: Disables threat intelligence filtering entirely. Use this mode only when you need a baseline traffic view during initial deployment.
Alert only: Logs suspicious connections but doesn't block them. Use this mode when testing to understand potential false positives before enforcing.
Alert and deny: Blocks and logs suspicious connections. Use this mode in production environments.


### Secure a Virtual WAN hub with Azure Firewall

<img width="678" height="309" alt="image" src="https://github.com/user-attachments/assets/1b10ed88-bf80-4b07-9ed3-c2abb0db82e9" />

Azure Virtual WAN provides managed hub connectivity for branches (site-to-site VPN), remote users (point-to-site VPN), and spoke VNets. By default, the Virtual WAN hub routes traffic directly between connected branches and spokes—no inspection in the middle.

The security requirement is clear: all inter-spoke, branch-to-spoke, and internet-bound traffic must pass through an inspection point. Azure Firewall deployed into the Virtual WAN hub solves this problem.


### Secured Virtual Hub architecture and routing intent
A Secured Virtual Hub is a Virtual WAN hub with Azure Firewall deployed into it. Deploying Azure Firewall into a Virtual WAN hub converts it to a Secured Virtual Hub. The firewall integrates with Virtual WAN routing to become the next hop for all traffic types when routing intent is enabled.

Azure Firewall Manager is the management plane for Secured Virtual Hubs. It applies Firewall Policy to the hub-deployed firewall and configures routing intent. Unlike the hub-spoke virtual network pattern, you don't create an AzureFirewallSubnet manually—the platform manages the subnet automatically when you deploy Azure Firewall into the hub.

### How routing intent works

Private traffic: branch-to-spoke, spoke-to-spoke, and branch-to-branch traffic
Internet traffic: all internet-bound traffic from branches and spokes

**When routing intent is enabled for private traffic, Virtual WAN automatically programs the route tables in all connected branches and spoke VNets to route via the hub firewall. You don't create or update user-defined routes (UDRs) manually—the platform handles route propagation for you.**

**When routing intent is enabled for internet traffic, all internet-bound flows route through the firewall before egress. You can enable routing intent for one or both traffic types, depending on your security requirements.**



**Enabling routing intent on a production hub reroutes all traffic through the firewall immediately. Test with a nonproduction hub first and verify all required traffic is permitted in the Firewall Policy before enabling routing intent in production environments.**

### Hub-spoke virtual network vs. Secured Virtual Hub

<img width="685" height="368" alt="image" src="https://github.com/user-attachments/assets/ee9b1264-8dfb-4838-9973-d88581397cd8" />



Q&A

Company's security team needs to prevent Azure-hosted VMs from accessing malware command-and-control domains, even if those domains use dynamically generated hostnames. Which Azure Firewall capability addresses the requirement?
>Threat-intelligence-based filtering set to Deny mode, which blocks traffic to and from known malicious IPs and domains maintained by Microsoft.

A Company Azure AI agent running in a spoke virtual network makes outbound HTTPS calls to Azure OpenAI endpoints. The security team wants to ensure the agent can only reach authorized Azure OpenAI endpoints and is blocked from reaching any other external AI APIs. Which Azure Firewall rule type enforces blocking external AI APIs?
>An application rule collection with FQDN rules that allow the specific Azure OpenAI endpoint hostnames and deny all other external AI service FQDNs.


Company deploys Azure Virtual WAN with hub-spoke topology across three regions. The CISO requires all inter-spoke and internet-bound traffic to pass through a central inspection point. What configuration achieves the outcome?
>Convert each Virtual WAN hub to a Secured Virtual Hub by deploying Azure Firewall into the hub, then enable routing intent for private and internet traffic.

**Azure Firewall controls outbound and east-west traffic. For protecting inbound HTTP/S web application traffic from OWASP threats, use Azure Web Application Firewall on Application Gateway or Azure Front Door.**










in azure policy :

With the Audit effect, Azure evaluates every resource against the policy rule and records the compliance state. Noncompliant resources appear in the compliance dashboard with a reason code, but the resources remain operational. This approach provides visibility into the current state without breaking existing workloads.

With the Deny effect, Azure blocks the resource creation or update request entirely. When a developer attempts to deploy a storage account with HTTP enabled, the ARM template deployment fails immediately with an error message referencing the policy. The policy prevents new noncompliant resources from entering the environment.

The DeployIfNotExists effect combines evaluation with automated remediation. For example, a policy that requires diagnostic logs on Azure Key Vault uses DeployIfNotExists to check whether a diagnostic settings resource exists. If it doesn't, Azure deploys the diagnostic settings automatically during the next policy evaluation cycle.

A related effect, AuditIfNotExists, behaves like Audit but evaluates associated resources rather than the resource itself. For example, a policy that audits whether a VM has a specific extension installed uses AuditIfNotExists because the extension is a separate resource.
A related effect, AuditIfNotExists, behaves like Audit but evaluates associated resources rather than the resource itself. For example, a policy that audits whether a VM has a specific extension installed uses AuditIfNotExists because the extension is a separate resource.

Modify	Automatically adds, updates, or removes a tag or property on a resource at creation or update time	Enforcing resource tagging standards or default configurations

Policy assignments inherit through the Azure resource hierarchy: management group → subscription → resource group → resource.

Azure Security Benchmark 

Assigning at the subscription scope only covers resources within that subscription. New subscriptions created later aren't covered automatically. This scope is appropriate for policies that apply only to a specific workload or business unit.

o to Policy → Assignments → + Assign initiative. After the assignment completes, Azure begins evaluating all resources within the scope during the next policy evaluation cycle. This process runs automatically every 24 hours.



Newly assigned policies can’t show compliance results immediately. Trigger an on-demand evaluation for a specific resource group using the Azure CLI:  

``` bash
az policy state trigger-scan --resource-group <rg-name> 
```

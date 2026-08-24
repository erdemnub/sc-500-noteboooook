## Azure Policy supports several effects, but four are essential for security governance workflows:

>With the Audit effect, Azure evaluates every resource against the policy rule and records the compliance state. Noncompliant resources appear in the compliance dashboard with a reason code, but the resources remain operational. This approach provides visibility into the current state without breaking existing workloads.

>With the Deny effect, Azure blocks the resource creation or update request entirely. When a developer attempts to deploy a storage account with HTTP enabled, the ARM template deployment fails immediately with an error message referencing the policy. The policy prevents new noncompliant resources from entering the environment.

>The DeployIfNotExists effect combines evaluation with automated remediation. For example, a policy that requires diagnostic logs on Azure Key Vault uses DeployIfNotExists to check whether a diagnostic settings resource exists. If it doesn't, Azure deploys the diagnostic settings automatically during the next policy evaluation cycle.

>A related effect, AuditIfNotExists, behaves like Audit but evaluates associated resources rather than the resource itself. For example, a policy that audits whether a VM has a specific extension installed uses AuditIfNotExists because the extension is a separate resource.


### Policy hierarchy : management group → subscription → resource group → resource.

Azure Security Benchmark : Benchmark recommendations provide a starting point for selecting specific security configuration settings and facilitate risk reduction. (https://marketplace.microsoft.com/en-us/product/azure-applications/azuresentinel.azure-sentinel-solution-azuresecuritybenchmark?tab=Overview)


**Assigning at the subscription scope only covers resources within that subscription. New subscriptions created later aren't covered automatically. This scope is appropriate for policies that apply only to a specific workload or business unit.**

to to Policy → Assignments → + Assign initiative. After the assignment completes, Azure begins evaluating all resources within the scope during the next policy evaluation cycle. This process runs automatically every 24 hours.

Newly assigned policies can’t show compliance results immediately. Trigger an on-demand evaluation for a specific resource group using the Azure CLI:  

``` bash
az policy state trigger-scan --resource-group <rg-name> 
```

To view compliance results:

Go to Policy → Compliance to open the compliance dashboard.

The compliance dashboard organizes these findings by policy definition and subscription. Each noncompliant resource includes a reason code (for example, "Resource doesn't match policy rule") and a link to the resource's properties in the Azure portal. 

When built-in definitions aren't enough
Built-in policy definitions cover common security controls like encryption, network access, and identity management, but they operate at a general level. Organizations need more granular control to match their specific security standards and compliance frameworks

Custom policy definitions let you write the exact 'policyRule' logic that implements these requirements. You define the condition that triggers evaluation, the parameters that make the definition reusable across environments, and the effect that enforces compliance.

A custom policy definition uses a JSON (JavaScript Object Notation) structure with three core components: mode, parameters, and policyRule. Understanding each component helps you build definitions that enforce exactly what your organization requires.

```json
{
  "properties": {
    "displayName": "Storage accounts must use the approved Log Analytics workspace",
    "description": "Ensures diagnostic settings on storage accounts send logs to the Contoso central Log Analytics workspace.",
    "mode": "All",
    "parameters": {
      "approvedWorkspaceId": {
        "type": "String",
        "metadata": {
          "displayName": "Approved Log Analytics workspace ID",
          "description": "The resource ID of the required Log Analytics workspace."
        }
      },
      "effect": {
        "type": "String",
        "defaultValue": "AuditIfNotExists",
        "allowedValues": ["AuditIfNotExists", "Disabled"]
      }
    },
    "policyRule": {
      "if": {
        "field": "type",
        "equals": "Microsoft.Storage/storageAccounts"
      },
      "then": {
        "effect": "[parameters('effect')]",
        "details": {
          "type": "Microsoft.Insights/diagnosticSettings",
          "existenceCondition": {
            "field": "Microsoft.Insights/diagnosticSettings/workspaceId",
            "equals": "[parameters('approvedWorkspaceId')]"
          }
        }
      }
    }
  }
}
```

The mode property determines which resource types the policy evaluates. Use "All" to evaluate all resource types, including those without tags and location support, such as diagnostic settings and network security rules. Use "Indexed" to evaluate only resource types that support tags and location, which is appropriate for policies that check tag compliance or regional restrictions.

Parameters make the definition reusable across different environments and scopes. The approvedWorkspaceId parameter lets you specify a different Log Analytics workspace for each assignment without modifying the definition. Always include an effect parameter with allowedValues so the assignment can toggle between audit mode during testing and enforcement mode in production.

The policyRule contains two sections: if and then. The if section defines the condition that triggers evaluation, using the field keyword to access resource properties like type, location, tags, or SKU. The then section specifies the action when the condition is true, referencing the effect parameter. For AuditIfNotExists and DeployIfNotExists effects, the details.existenceCondition section checks whether the required associated resource or property exists.






Configure a remediation task for existing noncompliant resources 

Custom policies with DeployIfNotExists effects identify noncompliant resources but don't automatically fix them. Remediation tasks apply the required configuration to existing resources that were created before the policy assignment or that became noncompliant due to configuration drift.

Remediation tasks require a managed identity assigned to the policy assignment. The managed identity must have the role-based access control (RBAC) permissions needed to deploy the required resource. For a policy that deploys diagnostic settings, the managed identity needs the "Monitoring Contributor" role. For a policy that configures network security rules, the managed identity needs "Network Contributor."


### Policy exemptions

Azure Policy supports two exemption categories. A waiver exemption indicates the organization accepts the risk identified by the policy. No compensating control is in place, but leadership reviewed the risk and decided it's acceptable for this specific resource, often due to business constraints or legacy system limitations. A mitigated exemption indicates a compensating control addresses the same security objective that the policy targets. The resource doesn't technically comply with the policy definition, but the underlying security requirement is satisfied through an alternative mechanism

. When the exemption expires, the resource becomes subject to the policy assignment again, and compliance evaluation resumes

Azure Policy prevents noncompliant resources from being created, but critical resources that already exist and are correctly configured still need protection from accidental or malicious deletion.


 ## Implement resource locks 

you learn how to configure resource locks. Locks prevent deletion or modification of critical resources, understand lock inheritance behavior, and control who can create and remove locks.

Use a Delete lock to prevent accidental deletion on active workloads while allowing modifications, and reserve a ReadOnly lock strictly for frozen, static resources where all write and update operations must be blocked.

Control who can create and remove locks
Resource locks provide separation of duties between users who manage resources and users who control governance. Creating or deleting a resource lock requires the Microsoft.Authorization/locks/write and Microsoft.Authorization/locks/delete permissions, which are granted by the Owner and User Access Administrator built-in roles.

The Contributor role grants permission to create, update, and delete resources but doesn't include permission to manage locks. This design ensures that developers with Contributor access can manage their resources but can't remove locks applied by the security or governance team.


 Use a built-in or custom Azure Policy definition with a DeployIfNotExists effect to automatically deploy a Delete lock on resources that match specific criteria, such as all Recovery Services vaults or all production virtual networks tagged Environment: Production.


Protecting a critical resource with a lock requires just a few steps in the command line : 

```bash
az resource lock create \
 --lock-type ReadOnly \
 --name myLock \
 --resource-group MyResourceGroup \
 --resource myVnet \
 --resource-type Microsoft.Network/virtualNetworks
```
with (DeleteLock)
```bash
az resource lock create \
 --lock-type CanNotDelete \
 -n protectStorage \
 -g MyResourceGroup \
 --resource mystorageacct \
 --resource-type Microsoft.Storage/storageAccounts
```
Applying locks manually works for known critical resources, but new resources provisioned without locks reintroduce the risk. Use a built-in or custom Azure Policy definition with a DeployIfNotExists effect to automatically deploy a Delete lock on resources that match specific criteria, such as all Recovery Services vaults or all production virtual networks tagged Environment: Production.





Q&A
1.A security engineer assigns an Azure Policy definition with the DeployIfNotExists effect. Existing resources in the subscription don't meet the policy requirement. What must the engineer do to bring those resources into compliance?
>Create a remediation task that triggers the DeployIfNotExists deployment against the noncompliant resources.

2.A security engineer applies a Delete lock to a production Azure virtual network. A network administrator with the Owner role attempts to delete the network. What is the result?
>The delete is blocked. Resource locks override all RBAC role assignments, including Owner.

3.Company's security team receives a request to exempt a storage account from a policy that requires HTTPS-only access. The account uses a legacy application, and the network team confirmed that a compensating network control routes all traffic through an encrypted gateway. Which exemption category should the security engineer select?
>Mitigated—an alternative control addresses the same risk that the policy is designed to prevent.


## Microsoft Security DevOps (MSDO) 
to scan infrastructure as code (IaC) templates in the CI/CD pipeline before deployment, catching misconfigurations while they're still in source control. You also learn how to use Azure Policy to enforce compliance as a platform-level backstop—even for deployments that bypass the pipeline entirely. Together, these two layers form a complete defense-in-depth model for IaC security.

## Connecting Github and Azure DevOps to defender for Cloud

To enable the connection, go to Defender for Cloud → Environment settings → + Add environment → choose GitHub or Azure DevOps. For GitHub, the process requires a GitHub App installation on the organization or selected repositories. The installation creates a connector resource in Azure that links Defender for Cloud to your GitHub environment. For Azure DevOps, you authorize access using an OAuth authorization or Azure DevOps personal access token, which creates a similar connector resource.

After connection, Defender for Cloud ingests repository metadata and begins generating DevOps security recommendations. Recommendations include exposure of secrets in code, open-source component vulnerabilities, and IaC misconfigurations. All findings appear under the Recommendations screen in Defender for Cloud, filtered to the DevOps category.

### Microsoft Security DevOps (MSDO) extension

For GitHub workflows, the action name is microsoft/security-devops-action. You use it as a step in the workflow YAML file. For Azure Pipelines, the task name is MicrosoftSecurityDevOps@1, and you use it as a step in the pipeline YAML file. 

**By default, MSDO runs multiple tool categories: secrets scanning, code analysis, container scanning, and IaC scanning.**

Github Actions example : 

```yaml
- name: Run Microsoft Security DevOps
  uses: microsoft/security-devops-action@v1
  with:
    categories: 'IaC'
```
Azure DevOps Pipeline example:

```yaml
- task: MicrosoftSecurityDevOps@1
  inputs:
    categories: 'IaC'
```

### the scanning engines 

>Template Analyzer **analyzes ARM templates and Bicep files for Azure-specific security misconfigurations.** It checks for common issues like storage accounts without secure transfer required, Transport Layer Security (TLS) minimum version not set to 1.2, diagnostic logs not enabled, public access on storage accounts, and network rules allowing all traffic.

>PSRule for Azure is also available as a discrete tool within MSDO.**You can run it independently by specifying it in the tools parameter alongside or instead of Template Analyzer. It applies Azure Well-Architected Framework and CAF rules to Bicep and ARM templates**, providing a complementary rule set to Template Analyzer's security-focused checks.

>Checkov is an **open-source static analysis tool that supports Bicep, ARM template, Terraform, Kubernetes manifests, Dockerfiles, and Helm charts. It's useful in mixed-IaC environments** where teams use multiple template languages across different cloud providers.


**MSDO orchestrates multiple scanning tools internally. Two are most relevant for IaC template analysis: Template Analyzer and Checkov.**


## agentless scanning 

For teams that can't modify pipelines—because of legacy pipeline configurations, vendor-managed repositories, or organizational constraints—Defender for Cloud offers agentless IaC scanning.

**Agentless scanning works differently from pipeline scanning. Instead of running as a pipeline task, Defender for Cloud scans GitHub and Azure DevOps repositories directly, without any pipeline extension or code change. The scan runs every 24 hours automatically, versus being triggered on every commit like pipeline scanning.**


### Enterprise Policy as Code (EPAC) framework : 

MSDO scanning protects the pipeline path. Azure Policy with the Deny effect protects EVERY deployment path—it operates at the Azure Resource Manager layer and intercepts all write operations before they're committed to resource state. Together, MSDO and Azure Policy form two independent defense layers that address different threat surfaces. One catches violations during code review. The other catches violations at the moment of deployment, no matter where the deployment originates.



Azure Policy with the Deny effect blocks noncompliant resource creation or modification at the Resource Manager level. The deployment fails immediately with an error message identifying the violated policy. The key distinction from Module 1 is that Module 1 covered policy enforcement for runtime resources already deployed. In contrast, you learn here how to author and promote policy definitions for IaC governance—a pattern often called policy-as-code.



### Policy-as-code workflow : 
Policy-as-code means managing Azure Policy definitions, initiatives, and assignments as source-controlled code rather than as portal-only configurations. The workflow mirrors the same CI/CD practices used for infrastructure templates.

Here's the typical workflow:

Author - write the policy definition JSON (or Bicep equivalent) in source control alongside the IaC templates it governs
Test in audit mode - assign the policy in a dev or test environment with effect: Audit - observe compliance reports without blocking deployments
Validate - confirm the policy catches the intended violations and doesn't produce false positives
Promote to Deny - update the assignment parameter to effect: Deny for production environments (subscriptions or management groups)
Monitor - use Azure Policy compliance reports to track ongoing compliance



## Enterprise Policy as Code (EPAC) : 

For organizations with multiple management groups, subscriptions, and teams, manually maintaining hundreds of policy assignments becomes unsustainable. Enterprise Policy as Code (EPAC) is an open-source PowerShell module that manages Azure Policy at management group scale through a CI/CD pipeline.

EPAC provides several key capabilities. You define policy assignments in JSON files that map to management group scopes. EPAC then deploys, updates, and removes assignments consistently across environments using a single pipeline. It tracks desired state—EPAC detects and reports drift between the defined in code goal and the currently assigned in Azure configuration. It also supports GitHub Actions and Azure Pipelines natively, making it easy to integrate into existing workflows


### Secure Bicep authoring patterns:

**First, use the @secure() decorator on parameters that hold secrets or credentials. The decorator prevents values from being logged in deployment history, reducing the risk of credential exposure through deployment logs.**

**Second, reference Key Vault secrets using the existing keyword and getSecret() to avoid hardcoding sensitive values in parameter files. Secret references keep secrets centralized in Key Vault instead of scattered across parameter files in source control.**

**Third, assign managed identities to resources instead of service principal credentials. Managed identities eliminate the need to pass credentials through templates entirely, removing a common source of secret exposure in IaC templates.**


### Q&A
1.A security engineer wants to use the Microsoft Security DevOps (MSDO) GitHub Action to scan only infrastructure as code files in a repository, excluding code scanning. Which configuration achieves this?
>Add categories: 'IaC' in the with: block of the Microsoft Security DevOps GitHub Action

2.A Bicep template passes the IaC scan in a pipeline. The deploying engineer bypasses the pipeline and uses Azure CLI to deploy the template directly to production. Which control prevents the noncompliant resource from being created?
>Azure Policy with a Deny effect assigned at management group scope blocks the noncompliant resource at the platform level.

3.What is the recommended first step when introducing a new Azure Policy definition in a policy-as-code workflow?
>Assign the policy definition in Audit effect in a development or test environment and review compliance results before promoting to Deny.

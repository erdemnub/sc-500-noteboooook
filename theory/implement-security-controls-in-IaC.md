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


### Enterprise Policy as Code (EPAC) framework.




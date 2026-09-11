<img width="2273" height="1118" alt="image" src="https://github.com/user-attachments/assets/6b840f9c-5dd4-45ad-9d51-394fa4dff43e" />

Other audit action groups provide more granular control:

DATABASE_LOGOUT_GROUP: Records when users disconnect from the database
DATABASE_CHANGE_GROUP: Tracks database configuration changes like altering compatibility levels
SCHEMA_OBJECT_ACCESS_GROUP: Records access to specific tables and views
DATABASE_OBJECT_PERMISSION_CHANGE_GROUP: Tracks permission changes on database objects
DATABASE_PERMISSION_CHANGE_GROUP: Records changes to database-level permissions

## Differences between server-level and database-level 
Azure SQL auditing operates at two different scopes, and the choice affects both performance and log organization. Server-level auditing captures events from all databases on a logical server, while database-level auditing tracks events for a single database only.

The key technical difference lies in how extended event sessions are created. Server-level auditing uses a single extended event session for all databases on the logical server, while database-level auditing creates a separate session for each audited database. 

## Configure retention settings 
Retention settings control how long audit logs are preserved in storage. The default retention value is RETENTION_DAYS = 0, which means unlimited retention and never automatically deletes audit logs.

**When the admin combines SQL auditing with immutable storage (WORM), the SQL retention setting must be longer than the immutable storage lock period. For example, if immutable storage has a 90-day lock period, RETENTION_DAYS must be set to at least 90 days to ensure audit logs remain available for the entire immutability window.**


## Configure audit destinations for Azure SQL Database

<img width="2419" height="1334" alt="image" src="https://github.com/user-attachments/assets/457f1cd3-85fb-4b20-be2a-fb12cc513012" />


**Azure Blob Storage provides long-term, tamper-resistant audit log storage that meets compliance requirements. Audit logs are written as .xel files (SQL Server Extended Events format), which you can read using SQL Server Management Studio (SSMS) or SQL Server Profiler.**

**To configure blob storage as an audit destination, open the Azure portal and navigate to your SQL server resource. Under Security, select Auditing, then toggle Enable Azure SQL Auditing to on. Select Storage as the destination type, then specify the storage account URL in the format https://<StorageName>.blob.core.windows.net/<ContainerName>.**

*For authentication, managed identity is the recommended approach. Both system-assigned managed identity (SMI) and user-assigned managed identity (UMI) are supported. The portal selects the primary user-assigned identity by default. If no identity is assigned to the server, it creates and uses a system-assigned identity automatically. Assign the chosen identity the Storage Blob Data Contributor role on the target storage account. This approach eliminates the need to manage or rotate storage keys. Alternatively, you can use storage access keys for authentication, but keys require periodic rotation and poses a greater security risk if keys are compromised.*

**Azure Blob Storage supports two immutable policy types: time-based retention and legal hold. For financial compliance scenarios, use time-based retention with a period matching your regulatory requirement (typically 90–365 days). Once you lock a time-based retention policy, you can only extend it, not shorten it. Locking retention policies ensures audit logs remain tamper-proof even if an administrator's credentials are compromised.**

Using Log analytics
**To enable Log Analytics as an audit destination, return to the Auditing screen for your SQL server. Select Log Analytics as a destination and choose an existing workspace or create a new one. Audit events appear in the AzureDiagnostics table with Category == "SQLSecurityAuditEvents".**

This type of real-time detection is difficult to achieve with blob storage, which stores audit logs as files rather than queryable records.

Use this KQL query to review high-risk audit events:

```kql
AzureDiagnostics
| where Category == "SQLSecurityAuditEvents"
| where statement_s contains "DROP" or statement_s contains "TRUNCATE"
| project TimeGenerated, server_instance_name_s, database_name_s, client_ip_s, statement_s
| order by TimeGenerated desc
```
The statement_s column contains the SQL statement executed, while client_ip_s identifies the source IP address. 

Log Analytics retention governs the workspace retention setting, which you configure separately from SQL auditing. Workspaces support up to 730 days of interactive (queryable) retention. Beyond the 730 days, you can configure long-term archival retention of up to 12 years at a lower storage cost—archived data isn't immediately queryable but can be retrieved via search jobs when needed. 

**When Log Analytics or Event Hubs is configured as an audit destination, Azure automatically creates a diagnostic settings resource named SQLSecurityAuditEvents_XXXX-XXXX-XXX. If this resource is deleted—intentionally or by automation—audit logs stop flowing with no error or alert raised. The auditing screen continues to show auditing as enabled, but no events are written. To protect against the scenario, create an Azure Monitor activity log alert that fires when a diagnostic settings resource is deleted from your SQL server resource.**

## In event hubs 

Azure Event Hubs provides near-real-time streaming of audit events to downstream consumers. Unlike blob storage and Log Analytics, which stores audit logs for later analysis, Event Hubs forwards events immediately to a SIEM platform (such as Microsoft Sentinel, Splunk, or QRadar) or custom processing pipelines.

To configure Event Hubs as an audit destination, select Event Hub in the Auditing screen and provide the connection string for your Event Hubs namespace. Event Hubs uses SAS-based authentication, so you need to secure and periodically rotate the connection string.




### SQL Managed Instance
<img width="2406" height="1172" alt="image" src="https://github.com/user-attachments/assets/5e1c2214-1e74-4866-8660-52b2eba28e3f" />

SQL Managed Instance requires a server audit and server audit specification to send logs to blob storage. Unlike Azure SQL Database, which uses portal toggles for all audit destinations, SQL MI relies on T-SQL commands to create and configure audits.

To route audit logs to blob storage, you create a server audit with the TO URL clause that points to a storage container. This approach mirrors how SQL Server audits work on-premises, making it familiar to database administrators migrating workloads to Azure.

**SQL Managed Instance authenticates to the storage account using a Shared Access Signature (SAS) token, not a managed identity. You must create a SQL credential that stores the SAS token, with the credential name matching the storage URL:**

```sql
CREATE CREDENTIAL [https://companystorage.blob.core.windows.net/sqlmi-audit]
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = '<SAS-token-value-without-?>';
```


The TO EXTERNAL_MONITOR clause signal that logs flow through Azure's diagnostic pipeline rather than directly to storage. After creating this audit, you configure the destination in the Azure portal:

Navigate to the SQL Managed Instance resource
Select Monitoring > Diagnostic settings
Select Add diagnostic setting
Enable the SQLSecurityAuditEvents log category
Choose a destination: Log Analytics workspace or Event Hubs

This support operations audit provides compliance evidence that even Microsoft's privileged access is monitored and recorded. In the Azure portal, navigate to your SQL Managed Instance resource and select Security > Auditing. When creating or editing a server audit, you see a Microsoft support operations option that enables auditing of activities performed by Microsoft engineers during support sessions.

Storage-based audit logs appear as .xel (extended events) files in the blob container, which you open using SQL Server Management Studio's Extended Events viewer. Both storage and external monitor audits can run simultaneously, providing redundancy in case one destination becomes unavailable.



<img width="2412" height="1255" alt="image" src="https://github.com/user-attachments/assets/1f46ffb2-c689-47ed-857a-9234ee247e3a" />

**With Azure Blob Storage and immutable WORM policies, you create the tamper-resistant compliance record the financial regulator reviews. This destination prioritizes long-term retention, immutability, and availability for audit review. The regulator doesn't query these logs frequently, but when they do, the record must be complete and unaltered.**

**With Azure Monitor Log Analytics, you create the operational monitoring stream the security team queries daily. This destination prioritizes real-time alerting, correlation with other security signals, and investigative queries. The security operations center monitors failed authentication attempts, unusual query patterns, and permission changes using KQL queries against this workspace.**

The built-in policy "Auditing on SQL server should be enabled" uses the Audit effect to identify SQL servers where auditing isn't configured. This policy surfaces noncompliant resources in the Azure Policy compliance dashboard, making gaps visible to the security team. However, it doesn't automatically remediate the issue. The security team must manually enable auditing on flagged resources.

The built-in policy "Configure SQL servers to have auditing enabled to Log Analytics workspace" uses the DeployIfNotExists effect to automatically enable auditing and route logs to a Log Analytics workspace when new SQL servers are created. This policy creates a remediation task for existing noncompliant resources and prevents new resources from being deployed without auditing.

Q&A
- Company's Azure SQL Database server hosts a high-volume Online Transaction Processing (OLTP) banking application. The security team reports that server-level auditing is causing performance degradation during peak transaction hours. What change resolves this issue?
>Switch to database-level auditing so each database writes audit logs to its own folder independently

- A financial regulator requires that Contoso's database audit logs can't be altered or deleted after they're written. Which audit destination configuration meets this requirement?
>Azure Blob Storage with immutable blob storage (WORM) policies configured on the audit container

A cloud security engineer needs to configure SQL Managed Instance auditing to route logs to both Azure Monitor and an Event Hubs. Which configuration method is required for these nonstorage destinations?
>T-SQL CREATE SERVER AUDIT with TO EXTERNAL_MONITOR specified as the destination



SQL MI
 
Blob Storage
→ TO URL
 
Log Analytics
→ TO EXTERNAL_MONITOR
 
Event Hubs
→ TO EXTERNAL_MONITOR



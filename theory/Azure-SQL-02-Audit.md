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




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




Both databases allow SQL authentication with shared passwords, including credentials used by the AI service. The SQL servers have public endpoints accessible from the internet with only basic firewall protection. Transparent data encryption uses service-managed keys, but regulators require customer-managed keys for financial personal data. Query results expose full account numbers and credit card data to all database users regardless of their role.


Microsoft Entra to provide authentication and replace SQL credentials with managed identities for AI workloads


https://learn.microsoft.com/en-us/training/wwl-sci/configure-azure-sql-platform-security/media/managed-identity-authentication-flow.png#lightbox,



Set Microsoft Entra admin   	Assign a user or group as the Microsoft Entra administrator
Enable Entra-only auth     	Disable SQL Server authentication at the server level
Create contained users	   Map Microsoft Entra identities or managed identities to database users
Grant permissions	        Assign roles using standard T-SQL GRANT statements


Microsoft Entra ID–only authentication disables SQL logins and SQL Server authentication at the server level. 

When you enable this mode, traditional username-password authentication stops working, and all connections must authenticate using Microsoft Entra credentials.
Before enabling Entra-only authentication, you must set a Microsoft Entra admin on the logical SQL server or SQL Managed Instance. This admin has full control over the database and can create more Microsoft Entra users.Before enabling Entra-only authentication, you must set a Microsoft Entra admin on the logical SQL server or SQL Managed Instance. This admin has full control over the database and can create more Microsoft Entra users.


We have two different options to enable it. 

in azure portal settings > microsoft entra id choosing set admin.

also azure cli : 

```bash
az sql server ad-admin create \
--resource-group CompanyRG \
--server-name company-sql-server \   
--display-name "SQL Administrators" \
--object-id <group-object-id>
```


Once enabled, SQL authentication is prevented from connecting at the server level—existing SQL authentication logins and users remain in the system but can't establish connections. New SQL authentication logins can be created by Microsoft Entra accounts with proper permissions, but those accounts also can't connect while Entra-only mode is active. All successful connections must authenticate through Microsoft Entra ID.


This configuration brings three security benefits: it eliminates password sprawl by removing local SQL credentials, enables MFA enforcement through Microsoft Entra authentication policies, and allows Conditional Access policies to control access based on location, device compliance, or risk level.

## Attention

**Enabling Entra-only authentication immediately disables all SQL authentication logins. Ensure you have a Microsoft Entra admin configured and tested before enabling this mode in production environments.**


 ## Create contained database users for managed identities

Microsoft Entra users and managed identities are added to databases as contained database users, not server logins. A contained database user exists within the database itself and authenticates directly against Microsoft Entra ID. This approach simplifies permission management and aligns with modern cloud identity patterns. 




. System-assigned managed identities are automatically created and lifecycle-tied to the resource—when you delete the Function, the identity is deleted


to grant the managed identity access connect to the database as the Microsoft Entra admin and run T-SQL : 

```sql 
CREATE USER [FraudDetectionFunction] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [FraudDetectionFunction];
```


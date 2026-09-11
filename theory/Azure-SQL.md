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
The first statement creates a contained database user mapped to the managed identity. The name in brackets must match the name of the Azure resource with the managed identity.
The second statement grants read-only access by adding the user to the db_datareader role. For write access, use db_datawriter, or grant specific permissions using standard GRANT statements.

You can also create user for Microsoft Entra groups, which simplifies permission management when multiple users or services need the same access:

```sql
CREATE USER [SecurityEngineers] FROM EXTERNAL PROVIDER;
GRANT VIEW DATABASE STATE TO [SecurityEngineers];
```
After you create the database user, the application connection string uses Authentication=Active Directory Managed Identity. No passwords or secrets are needed—the Azure platform handles token acquisition and rotation automatically.

## Attention

**Test managed identity access from the application before disabling SQL authentication.**
Use **Azure Monitor** or query diagnostics to verify successful authentication events.


# Azure RBAC roles for SQL security management
Azure RBAC roles control who can manage SQL resources, but they don't grant access to data inside databases. This separation ensures that management permissions don't automatically grant data access—a principle of least privilege


**The SQL Security Manager** role (ID: 056cd41c-7e88-42e1-933e-88ba6a50c9c3) grants permissions to manage security policies including firewall rules, encryption settings, auditing configuration, dynamic data masking, and row-level security. This role is designed for security engineers who configure security controls but don't need to read or modify data. With this role, you configure authentication settings like the Microsoft Entra admin and enable Entra-only authentication.

**the SQL Server Contributor role** manages SQL servers and databases but doesn't grant security policy management. **The SQL DB Contributor role** manages individual databases but also lacks security policy permissions. 

**Passing authentication doesn't bypass network rules, and passing network rules doesn't bypass authentication.**

# Network Izolatıon


<img width="2116" height="978" alt="image" src="https://github.com/user-attachments/assets/99d65d1e-a158-4227-a7e7-44423015f35d" />



Private endpoints + disable public access  

use case : Financial services, regulated workloads requiring no public endpoint exposure

Development environments where public endpoint is acceptable with restricted access  approach:
Virtual network service endpoints + firewall rules


## Configure private endpoints for Azure SQL

Private endpoints bring Azure SQL Database into your virtual network address space, eliminating the need for public endpoint access. When you deploy a private endpoint, Azure creates a network interface with a private IP address from your subnet and maps it to your SQL server's fully qualified domain name (FQDN).

 In the Azure portal, navigate to your SQL server, select Security > Networking, and choose Private access. When you add a private endpoint, specify the target subnet and enable automatic private DNS zone integration. Azure creates the privatelink.database.windows.net DNS zone and adds a record that maps your server name to the private IP address.

 The connection policy determines how traffic flows after a private endpoint is deployed. The Proxy policy routes all traffic through port 1433 and is the simplest option for private endpoint deployments—it requires no extra firewall changes and works with existing private endpoints that use the Default policy. The Redirect policy provides lower latency by establishing a direct connection to the database node, but requires clients to open ports 1433 to 65535 for both inbound and outbound communication on the virtual network hosting the private endpoint.

**Existing private endpoints using the Default connection policy fallback to Proxy mode (port 1433 only) to avoid disrupting client traffic. To use Redirect with a private endpoint, explicitly set the connection policy to Redirect after provisioning the endpoint—toggling the policy can be required if it was set before the private endpoint was created.**

**Private endpoint creation requires approval from the SQL administrator. After the network administrator creates the endpoint, the SQL administrator must approve the connection in the SQL server's Private endpoint connections list before it becomes active.**


## Disable public network access 


The private endpoint alone doesn't prevent public access—both endpoints remain active until you explicitly disable public network access.

SQL server's Networking page and select Public access > Disable. This action blocks all connections from the internet, even if firewall rules permit specific IP addresses. The "Allow Azure services and resources to access this server" setting, which creates a special rule for Azure's internal IP ranges (0.0.0.0–0.0.0.0), no longer applies because the public endpoint is turned off.

**Firewall rule changes can take up to five minutes to propagate across Azure's infrastructure. If you need immediate effect after modifying rules, connect to the database and run the following command:**
```sql
DBCC FLUSHAUTHCACHE
```

## network isolation to SQL Managed Instance

*Unlike Azure SQL Database, SQL MI doesn't require a private endpoint—it's already VNet-native.*

*SQL MI offers an optional public endpoint on port 3342, but it's disabled by default.*

The key difference: Azure SQL Database requires you to add private endpoints and disable public access as separate steps, while SQL Managed Instance starts with private-only connectivity built in.

test connectivity using the same FQDN:
```bash
sqlcmd -S contoso-transactions.database.windows.net -U sqladmin -P <password> -Q "SELECT @@VERSION"
```




 Defender for Azure SQL Databases at subscription scope

<img width="1920" height="920" alt="image" src="https://github.com/user-attachments/assets/0e13e942-0be1-49a3-805f-a037f0f9a64d" />

## Compare Defender for Databases plans 

Defender for Databases is a bundle within Microsoft Defender for Cloud that contains four independently priced subplans: 
>Defender for Azure SQL Databases,
>Defender for SQL Servers on Machines,
>Defender for Open-Source Relational Databases, 
>Defender for Azure Cosmos DB.

---

Azure SQL / SQL MI
→ Defender for Azure SQL Databases
 
SQL Server VM / Arc
→ Defender for SQL Servers on Machines
 
PostgreSQL / MySQL
→ Defender for Open-Source Relational Databases
 
Cosmos DB
→ Defender for Cosmos DB


<img width="1943" height="958" alt="image" src="https://github.com/user-attachments/assets/c8274387-45ad-43de-ba82-c41c6f7cd508" />



<img width="868" height="331" alt="image" src="https://github.com/user-attachments/assets/e5990f1f-893a-4a8e-a36c-bf1d05440ff9" />


**if you run SQL Server on virtual machines or on-premises servers with Arc, you need the Azure SQL plan. If you only use managed PostgreSQL or MySQL services, the open-source plan provides the appropriate protection.**

For Azure SQL workloads, Defender detects SQL injection attacks by identifying when applications construct SQL statements that include malicious user input. The detection logic identifies both successful injection attempts and vulnerability indicators. Even if the attacker hasn't yet escalated privileges, you receive an alert that the database is vulnerable.  

**Defender for Databases uses behavioral analytics to establish a baseline of normal database activity and detects anomalies such as unusual access patterns, geographic locations, query volumes, and brute-force attacks. It correlates suspicious events into meaningful alerts and maps them to MITRE ATT&CK tactics, helping security teams understand attack stages and prioritize incident response.**

---

Vulnerability assessment is included in Defender for Azure SQL Databases as an integrated feature, not a separate product. The assessment engine automatically scans your databases for security misconfigurations and known vulnerabilities, then generates findings categorized by severity level: High, Medium, and lower-severity best practice recommendations.

With Express configuration (the recommended mode), Microsoft manages scan result storage and no storage account configuration is required. Findings appear directly in Defender for Cloud's recommendations view without requiring you to configure storage accounts or scan agents. You can mark accepted findings—such as configurations that are intentional for your environment—so only new deviations surface as open issues. This baseline approach reduces alert fatigue and focuses your attention on configuration drift



<img width="1920" height="920" alt="image" src="https://github.com/user-attachments/assets/e6e5a382-49ce-40cc-816b-7ddacb32b032" />

steps to configure defender:

<img width="836" height="272" alt="image" src="https://github.com/user-attachments/assets/96821a5a-0d96-430d-8a50-0509cdfc0372" />


### Enable protection at the subscription level

When you enable Defender for Azure SQL Databases at the subscription level, protection applies instantly to all SQL Database instances, elastic pools, SQL Managed Instances, and Synapse Analytics dedicated SQL pools in that subscription

**To enable subscription-level protection, navigate to Microsoft Defender for Cloud, select Environment settings, choose the target subscription, and locate the Databases plan. Enable Defender for Azure SQL Databases with a single toggle. This activation protects all existing SQL resources immediately and extends coverage to any resources created later. Development teams can deploy new SQL databases without requiring other security configuration steps.**

To assign the policy, navigate to Azure Policy, select Definitions, and search for "Defender SQL" to locate the appropriate policy. Assign it at the management group level that contains your production subscriptions. This approach ensures Company's production subscription maintains protection even if configuration changes occur, and automatically applies the same protection standard to new subscriptions created for other business units or regions.

### Enable Defender for open-source relational databases

<img width="1961" height="878" alt="image" src="https://github.com/user-attachments/assets/4fb7eb50-b9ba-4d06-b578-586e781ac2d9" />

Not covered by this plan : 
>SQL Server on Virtual Machines
>PostgreSQL Single
>Other Cloud

**The plan protects Azure Database for PostgreSQL Flexible Server and Azure Database for MySQL Flexible Server across all pricing tiers. Azure Database for MariaDB isn't covered by this plan. In preview, the plan also protects Amazon RDS instances running Aurora PostgreSQL, Aurora MySQL, PostgreSQL, MySQL, and MariaDB. This RDS coverage has geographic limitations that you should verify before relying on it for production workloads.**

**The plan doesn't extend to SQL Server running on virtual machines or Arc-connected servers. If you need threat detection for SQL Server instances outside Azure SQL Database, you use the Defender for Azure SQL Databases plan with Arc enablement, which you learned about in the previous unit.**


Enablement is independent from the Azure SQL Database toggle. Turning on protection for Azure SQL databases doesn't activate open-source database protection, and vice versa. You need to enable both toggles to get comprehensive database threat detection across your subscription.

<img width="839" height="264" alt="image" src="https://github.com/user-attachments/assets/b31550df-910b-4478-93f3-802956e8349d" />

 Defender for Azure SQL Databases extends to on-premises and multicloud SQL Server instances through Azure Arc, while the open-source plan doesn't support Arc connectivity. If you're running PostgreSQL or MySQL on on-premises servers or in other clouds, you can't use Defender for Cloud to protect them. Only Azure-native and AWS RDS instances receive coverage.

The pricing model also differs. Defender for Azure SQL Databases charges per SQL server (covering all databases on that server), while the open-source plan charges per database server. This difference affects cost planning when you run multiple database engines across your environment.



### Vulnerability Assesment
<img width="2031" height="896" alt="image" src="https://github.com/user-attachments/assets/11a86659-9a8e-408e-bdf5-6bc368ecebcb" />

>High-severity findings identify critical misconfigurations that represent active security risks. Examples include SQL logins enabled when Entra-only authentication is required, or the sysadmin role granted to nonadministrator accounts. These findings demand immediate attention because they create exploitable attack vectors.

>Medium-severity findings indicate configuration gaps that reduce security posture but don't represent immediate exploitation risks. These findings often relate to defense-in-depth controls or operational best practices that strengthen overall security.

>Lower-severity findings provide best practice recommendations that improve security hygiene over time. While not urgent, these findings help you maintain a strong security baseline.

Express configuration is the recommended approach for vulnerability assessment because Microsoft manages the baseline automatically and no storage account is required.

<img width="844" height="332" alt="image" src="https://github.com/user-attachments/assets/79b44f64-2d73-4649-b318-08cfb64cc176" />


**To enable express configuration, navigate to Microsoft Defender for Cloud, select your SQL server or SQL managed instance resource, open the Defender for Cloud screen, and select Vulnerability assessment. Select Configure, then select Express.**

## Configure alert routing 
<img width="2082" height="888" alt="image" src="https://github.com/user-attachments/assets/21b467ab-2ee1-4f0f-8c5a-b144fc2f94fb" />

### Configure email notifications for immediate awareness
>Email notifications provide the fastest path to awareness when Defender for Databases detects a threat
### Connect to Microsoft Sentinel for centralized incident management
>While email provides immediate awareness, Microsoft Sentinel offers centralized incident management that the SOC team uses to track investigations across multiple alert sources




Q&A

1.Company's cloud security team needs to protect both Azure SQL Managed Instance and Azure Database for MySQL. Which Defender for Databases plan selection correctly covers both services?
>Defender for Azure SQL Databases for SQL Managed Instance, and Defender for open-source relational databases for Azure Database for MySQL

2.A security engineer enables Defender for Azure SQL Databases at subscription scope. Which statement correctly describes the resulting coverage behavior?
>All existing and future Azure SQL resources in the subscription receive protection automatically

3. A cloud security engineer runs a vulnerability assessment on Contoso's Azure SQL Database and sees several findings. Some findings represent accepted configurations, such as broad read permissions for an internal reporting service account. What is the correct action to prevent these known findings from appearing as open issues?
>Set a baseline for the findings to mark the accepted configurations as known and expected


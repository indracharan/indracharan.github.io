---
layout: post
title: "Solved! Data security in Azure Synapse"
date: 2023-05-15 00:00:00 +0530
categories: [azure, data engineering]
permalink: /blogs/synapse_security
---

A new team, a new challenge!
At this time my organization had slowly started to embrace cloud technologies, with Azure as a preferred partner.

Our team was exploring Azure Synapse, a managed cloud analytics service with dedicated SQL pools for data warehousing.

## Challenge
Our organization has different categories of users, who have different levels of data access. For the sake of explanation, let's divide them into four buckets: Level 1, 2, 3, and 4.

Level 1 users have the highest level of access and can see all data in the tables.
Level 2 users have less permission than Level 1, so they cannot see the most sensitive data but can see some sensitive data.
Level 3 users have very limited access to sensitive data.
Level 4 users cannot see the higher-classified sensitive data.
All levels can access non-sensitive data.

Note: If you want to understand more about the data categories an organization could have, I can explain with an example. Hit me up!

The challenge was to have a data security framework where lower-clearance users could not read data above their level, while higher-clearance users could access data at lower levels.

To put it in SQL terms, imagine a table with five columns: `Level1_A`, `Level2_B`, `Level3_C`, `Level4_D`, and `Public_E`. Level 1 can see all columns. Level 2 should not see the value in `Level1_A`, and so on. In this example, `Level4_D` is the least-sensitive classified field and is available to Level 4; `Public_E` is available to everyone.

## Solution
### 1. Put users in groups
The first task was to put each user in a group based on their level of access.

Microsoft Entra ID (previously Azure Active Directory) has security groups containing the users who belong to that group. Microsoft Identity Manager (MIM) is an identity management product, not the name of these groups. If MIM is used to manage or provision group membership, it can remain part of that process; the SQL database is assigned permissions through the corresponding Microsoft Entra security groups.

Think of a group as an Outlook distribution list: it has the email identities of the users belonging to it.

So we created four distinct security groups: Level1, Level2, Level3, and Level4, and added each user to the appropriate group.

Here is a simplified example of mapping the Entra groups to database roles in a dedicated SQL pool. Replace the group names with the display names in your tenant:

```sql
CREATE ROLE [Level1];
CREATE ROLE [Level2];
CREATE ROLE [Level3];
CREATE ROLE [Level4];

CREATE USER [Level1-Entra-Group] FROM EXTERNAL PROVIDER;
CREATE USER [Level2-Entra-Group] FROM EXTERNAL PROVIDER;
CREATE USER [Level3-Entra-Group] FROM EXTERNAL PROVIDER;
CREATE USER [Level4-Entra-Group] FROM EXTERNAL PROVIDER;

ALTER ROLE [Level1] ADD MEMBER [Level1-Entra-Group];
ALTER ROLE [Level2] ADD MEMBER [Level2-Entra-Group];
ALTER ROLE [Level3] ADD MEMBER [Level3-Entra-Group];
ALTER ROLE [Level4] ADD MEMBER [Level4-Entra-Group];
```

### 2. Map the data rules
Now that users are categorized, how do we ensure each of them gets only the access they are supposed to have?

Dedicated SQL pools support dynamic data masking (DDM). A mask is configured on a column, and users without `UNMASK` permission see masked values in query results. `UNMASK` can be granted to a role for a whole table or selected columns, which lets the roles reveal different columns. DDM masks values; it does not remove columns from the result set.

For example, the following statements mask the classified columns and let every level query the table. `dbo.Customer` is an example table name; use your actual schema and table:

```sql
ALTER TABLE dbo.Customer
ALTER COLUMN Level1_A ADD MASKED WITH (FUNCTION = 'default()');

ALTER TABLE dbo.Customer
ALTER COLUMN Level2_B ADD MASKED WITH (FUNCTION = 'default()');

ALTER TABLE dbo.Customer
ALTER COLUMN Level3_C ADD MASKED WITH (FUNCTION = 'default()');

ALTER TABLE dbo.Customer
ALTER COLUMN Level4_D ADD MASKED WITH (FUNCTION = 'default()');

GRANT SELECT ON OBJECT::dbo.Customer TO [Level1];
GRANT SELECT ON OBJECT::dbo.Customer TO [Level2];
GRANT SELECT ON OBJECT::dbo.Customer TO [Level3];
GRANT SELECT ON OBJECT::dbo.Customer TO [Level4];

-- Level 1 can see every column without masking.
GRANT UNMASK ON OBJECT::dbo.Customer TO [Level1];

-- Higher-clearance roles can unmask the columns allowed by the policy.
GRANT UNMASK ON OBJECT::dbo.Customer (Level2_B, Level3_C, Level4_D) TO [Level2];
GRANT UNMASK ON OBJECT::dbo.Customer (Level3_C, Level4_D) TO [Level3];
GRANT UNMASK ON OBJECT::dbo.Customer (Level4_D) TO [Level4];
```

In this example, a Level 4 user querying `Level1_A` sees its masked value, while the user can still see `Level4_D` and `Public_E`. A query using `SELECT *` still returns the masked columns. If the requirement is that a user must not be able to query a column at all, use column-level `SELECT` grants instead of relying on DDM. For example, grant only the columns Level 2 may query, and do not also grant that role table-, schema-, or database-level `SELECT` permission:

```sql
GRANT SELECT ON OBJECT::dbo.Customer
	(CustomerId, Level2_B, Level3_C, Level4_D, Public_E)
TO [Level2];
```

DDM is useful to reduce accidental exposure, but it should not be treated as a complete defense against users who can run arbitrary queries and try to infer masked values. For data that must be inaccessible, use least-privilege permissions such as column-level grants and test access using accounts in each group.

At the database level, we created roles and a mapping table that recorded each table's columns and their access levels. A stored procedure read that metadata and applied the required masks and role permissions.

Stored procedure? Just for four roles?
Not really. Our organization had hundreds of these mappings, so a stored procedure made more sense. In production, the procedure should validate the metadata and generate only the required `ALTER TABLE`, `GRANT`, and `REVOKE` statements.

### 3. Link users to the SQL roles
We had already added users to the respective Entra groups, so SQL roles were applied to groups and not individual users. Managing access was easier: as long as a user was a member of the right group, they inherited that role's permissions.

That's it!
Hope you liked how I solved it!

For more details, see Microsoft's documentation on [dynamic data masking for Azure Synapse Analytics](https://learn.microsoft.com/en-us/azure/azure-sql/database/dynamic-data-masking-overview) and [database engine permissions](https://learn.microsoft.com/en-us/sql/relational-databases/security/permissions-database-engine).
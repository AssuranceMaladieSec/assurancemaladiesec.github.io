---
layout: post
title:  "FoxTerrier : On the trail of vulnerable Active Directory objects and a report"
date:   2021-12-07 11:00:00 +0200
lang: en
ref: foxterrier
author: acp 
categories: ActiveDirectory
---

{% include toc %}

# The FoxTerrier origin story

Like many companies, we are using the famous [BloodHound](https://github.com/BloodHoundAD/BloodHound) (thanks so much [@_wald0](https://twitter.com/@_wald0), [@CptJesus](https://twitter.com/@CptJesus) and [@harmj0y](https://twitter.com/@harmj0y)!) to identify vulnerable objects and dangerous paths on our Active Directory (AD).

But during our recurrent audits on our AD, we faced a limitation in our use of the tool. Indeed, We needed:

- A report containing all vulnerable objects
- A synthesis of the vulnerable objects
- The ability to use multiple start nodes (objects) in input
- The ability to use de BloodHound databases (Neo4j) without the BloodHound GUI

> **Neo4j**: Neo4j is a graph database management system. The BloodHound environment uses Neo4j database to: 
- store the Active Directory data retrieved by `SharpHound`
- request these data. 

It is possible to use multiple start nodes by crafting and executing your own Cypher Queries in the BloodHound `Raw Query bar`. 

> **Cypher Queries**: Cypher is [Neo4j’s](https://neo4j.com) graph query language. A [Cypher Query](https://neo4j.com/developer/cypher/) is a query in the Cypher language.

However, Cypher Queries are complex and crafting requests manually everytime we need results is not an option in our case.


# Does BloodHound already provide reports? 

Unfortunately, no. BloodHound can provide a json of your current results or a png of it. And that's all. BloodHound is simply not designed for this purpose. 

# What is FoxTerrier

FoxTerrier is a Free Software tool written in Python and ***working in the BloodHound environment***. Source code and cooperation available on [its repository under the GitHub of the Assurance Maladie IT Security team](https://github.com/AssuranceMaladieSec/FoxTerrier).

> FoxTerrier can be seen as a **more flexible version without GUI of BloodHound `OUTBOUND CONTROL RIGHTS` and `EXECUTION RIGHTS (RDP only)` features**. 
> 
> In addition, FoxTerrier provides **all the results in a `csv` file and a `.txt` summary of it**.

FoxTerrier allows you to :

* **set multiple starting points**: identify, from a predefined list of user/groups, all the vulnerable objects (GPO, OU, User, Group, machine with RDP connection available for the object) that can be compromised by them.
* **set the type of the desired vulnerable objects**: unlike the BloodHound `OUTBOUND CONTROL RIGHTS` feature, FoxTerrier allows you to request the type of object you want for specifics users/groups. 
  For instance, if you want to retrieve only the vulnerable GPO that can be compromised by a list of predefined users/groups, you can :)
* **use regexp**: predefined users and groups can be expressed as regular expressions. It can be handy if, for instance, you have generic groups with pattern in the name and want to target all of them.

> **Prerequisite:** FoxTerrier relies on the Neo4j databases already filled with Active Directory information provided by SharpHound.

# How FoxTerrier works

Like the BloodHound GUI, FoxTerrier works with Active Directory information retrieved by SharpHound and loaded in the Neo4j database.

So, before using FoxTerrier, make sure that:
- an Active Directory scan has been performed by SharpHound
- its scan results have been loaded in the Neo4j database

FoxTerrier creates specific Cypher Queries by using the data provided in the input file called `template.json` and requests the Neo4j database with these Cypher Queries. 

![FoxTerrier Schema](/images/posts/requests.png)

> BloodHound and FoxTerrier can be used simultaneously.

To work, FoxTerrier needs 2 files: `conf.ini` and `template.json`

## template.json: The input file

The file `template.json` contains the data used by FoxTerrier to create the specific Cypher Queries. 

In this file 6 parameters can be used:
- **node_start_type**	: <span style="color:red">*Mandatory*</span>. Must be `"User"` or `"Group"`.
- **node_start_name**	: <span style="color:red">*Mandatory*</span>. Can be the full name or a regex (cf.`is_node_start_regex`). Example `"JOHN-DOE-825@MYDOMAIN.LOCAL"` or `"JOHN-DOE-\\d{3}@MYDOMAIN.LOCAL"`.
- **is_node_start_regex** : If regex are used in `node_start_name`, the value must be `true` (be careful to use the value `true` and not the string `"true"`). *Default value : **false***
- **mode** : Relation between start node and objects can be direct or indirect (permissions inherited from a group membership). You can set the mode of your choice by choising between the value `"direct"`, `"indirect"` or `"all"` (`all` is "direct"+"indirect"). *Default value : **"direct"***
 - **objects_type** : The target objects can be `"GPO"`, `"OU"`, `"User"`, `"Group"`,`"RDP"`. **The values must be within a list**. *Default value : **["GPO", "OU" ,"User", "Group", "RDP"]***
- **exclude_node** : When using regex the results can be overwhelming. It's possible to exclude specific nodes from the queries. **The values must be within a list**. Example : `["GENERIC-GROUP-11111111@MONDOMAIN.LOCAL","GENERIC-GROUP-22222222@MONDOMAIN.LOCAL"]`

Here is an example of a `template.json` file:

 ```
 {	
	"queries": 
	[	
		{
			"node_start_type": "Group",
 			"node_start_name": "GENERIC-GROUP-\\d{8}.*@MYDOMAIN.LOCAL",
			"is_node_start_regex": true,
			"mode": "all",
			"objects_type": ["GPO", "OU" ,"User", "Group","RDP"],
			"exclude_node": ["GENERIC-GROUP-11111111@MYDOMAIN.LOCAL","GENERIC-GROUP-22222222@MYDOMAIN.LOCAL"]
		},
		{
			"node_start_type": "User",
			"node_start_name": "MY-AD-ACCOUNT@MYDOMAIN.LOCAL",
			"is_node_start_regex": false,
			"mode": "direct",
			"objects_type": ["GPO"],
		}		
	]
 }
 ```
## conf.ini : The configuration file

The file `conf.ini` contains the neo4j credentials, the file name of the summary and the report and the template file name. If you want to change the name of the file generated or the input file, it's here.

 ```
[neo4j_credentials]
username=PutYourNeo4jLoginHere
password=PutYourNeo4jPasswordHere
address=127.0.0.1
port=7687

[files]
template_file=template.json
csv_report=my_report.csv
txt_summary=my_summary.txt
 ```

# A FoxTerrier output

## A `summary.txt` example:

```
--- Load JSON File C:\Users\XXX\Documents\Tool\FoxTerrier\template.json
--- JSON file C:\Users\XXX\Documents\Tool\FoxTerrier\template.json Loaded ---

--- Executed queries ---

Match p=(m:Group)-[r]->(n:GPO) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' and r.isacl=true return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r1:MemberOf*1..]->(g2:Group)-[r2]->(n:GPO) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' and r2.isacl=true return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r]->(n:OU) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' and r.isacl=true return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r1:MemberOf*1..]->(g2:Group)-[r2]->(n:OU) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' and r2.isacl=true return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r]->(n:User) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' and r.isacl=true return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r1:MemberOf*1..]->(g2:Group)-[r2]->(n:User) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' and r2.isacl=true return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r]->(n:Group) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' and r.isacl=true return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r1:MemberOf*1..]->(g2:Group)-[r2]->(n:Group) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' and r2.isacl=true return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r:CanRDP]->(n:Computer) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' return m.name AS start_name, n.name AS end_name

Match p=(m:Group)-[r1:MemberOf*1..]->(g2:Group)-[r2:CanRDP]->(n:Computer) WHERE m.name =~ 'MONGROUPE_GENERIQUE-\d{11}.*@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-11111111111@MONDOMAIN.LOCAL' AND NOT m.name = 'MONGROUPE_GENERIQUE-22222222222@MONDOMAIN.LOCAL' return m.name AS start_name, n.name AS end_name

--- Summary of vulnerable object per User or Group ---

MONGROUPE_GENERIQUE-33333333333@MONDOMAIN.LOCAL : 1
MONGROUPE_GENERIQUE-66666666666@MONDOMAIN.LOCAL : 1
MONGROUPE_GENERIQUE-99999999999@MONDOMAIN.LOCAL : 1
MONGROUPE_GENERIQUE-00000000000@MONDOMAIN.LOCAL : 1
MONGROUPE_GENERIQUE-77777777777@MONDOMAIN.LOCAL : 1
MONGROUPE_GENERIQUE-44444444444@MONDOMAIN.LOCAL : 2

--- Summary of CanRDP machines per User or Group ---

MONGROUPE_GENERIQUE-33333333333@MONDOMAIN.LOCAL : 3289
MONGROUPE_GENERIQUE-66666666666@MONDOMAIN.LOCAL : 521
MONGROUPE_GENERIQUE-99999999999@MONDOMAIN.LOCAL : 141
MONGROUPE_GENERIQUE-00000000000@MONDOMAIN.LOCAL : 35
MONGROUPE_GENERIQUE-77777777777@MONDOMAIN.LOCAL : 5
MONGROUPE_GENERIQUE-44444444444@MONDOMAIN.LOCAL : 84
MONGROUPE_GENERIQUE-33333333366@MONDOMAIN.LOCAL : 300
MONGROUPE_GENERIQUE-99333333366-TOTO@MONDOMAIN.LOCAL : 459
MONGROUPE_GENERIQUE-55444444444@MONDOMAIN.LOCAL : 459
MONGROUPE_GENERIQUE-82333333366@MONDOMAIN.LOCAL : 697

--- Summary of vulnerable object per Type ---

GPO : 4
OU : 1
User : 1
Group : 1
CanRDP : 5990

--- Results available in My_Report.csv ---
```

## A `report.csv` example:


| Start Object 				| Vulnerable Object 				|Distinguished Name Vulnerable Object 													| Type |
| -----------  				| 	----------- 					| ----------- 																			|  -----------  |
| MY-USER@MONDOMAIN.LOCAL 	| GPO-169519@MONDOMAIN.LOCAL  		|CN={12345678-1234-5678-9123-012345678944},CN=Policies,CN=System,DC=mondomain,DC=local	| GPO 			|
| MY-USER@MONDOMAIN.LOCAL 	| GPO-1881@MONDOMAIN.LOCAL 			|CN={12345678-1234-5678-9123-012345678955},CN=Policies,CN=System,DC=mondomain,DC=local	| GPO 			|
| MY-USER@MONDOMAIN.LOCAL 	| GPO-94984@MONDOMAIN.LOCAL 		|CN={12345678-1234-5678-9123-012345678977},CN=Policies,CN=System,DC=mondomain,DC=local	| GPO 			|
| MY-USER@MONDOMAIN.LOCAL 	| Berlin@MONDOMAIN.LOCAL  			|OU=Berlin,OU=Germany,OU=My Big Company,DC=mondomain,DC=local							| OU 			|
| MY-USER@MONDOMAIN.LOCAL 	| GPO-71188@MONDOMAIN.LOCAL  		|CN={12345678-1234-5678-9123-012345678966},CN=Policies,CN=System,DC=mondomain,DC=local	| GPO 			|
| MY-USER@MONDOMAIN.LOCAL 	| GPO-72618@MONDOMAIN.LOCAL  		|CN={12345678-1234-5678-9123-012345678922},CN=Policies,CN=System,DC=mondomain,DC=local	| GPO 			|
| MY-USER@MONDOMAIN.LOCAL 	| New-York@MONDOMAIN.LOCAL 			|OU=New-York,OU=USA,OU=My Big Company,DC=mondomain,DC=local								| OU 			|
| MY-USER@MONDOMAIN.LOCAL 	| GPO-4514@MONDOMAIN.LOCAL  		|CN={12345678-1234-5678-9123-012345678999},CN=Policies,CN=System,DC=mondomain,DC=local	| GPO 			|
| MY-USER@MONDOMAIN.LOCAL 	| GPO-72967@MONDOMAIN.LOCAL  		|CN={12345678-1234-5678-9123-012345678933},CN=Policies,CN=System,DC=mondomain,DC=local	| GPO 			|
| MY-USER@MONDOMAIN.LOCAL 	| GROUP-1644@MONDOMAIN.LOCAL  		|CN=GROUP-1644,OU=Berlin,OU=Germany,OU=My Big Company,DC=mondomain,DC=local				| Group 			|
| MY-USER@MONDOMAIN.LOCAL 	| GPO-2561@MONDOMAIN.LOCAL 			|CN={12345678-1234-5678-9123-012345678911},CN=Policies,CN=System,DC=mondomain,DC=local	| GPO 			|


# Conclusion

So according to us, FoxTerrier can help you because it is:

- Fast: Because FoxTerrier doesn't use a GUI interface and doesn't try to display all the results, it's more faster when requesting data for multiple start objects.
- Flexible: When you use in BloodHound the `Outbound Control Rights` feature on a object, all vulnerable objects related to it are displayed. With FoxTerrier you can choose the object type you want to target.
- Multiple start objects in input: You can specify, multiple start objects in the input file. You can also use regex to target start objects with pattern on their name.

But, the only way to be sure FoxTerrier can help you during your AD security assessment is to [download it](https://github.com/AssuranceMaladieSec/FoxTerrier) and use it!

Issues, questions, cooperation will be welcomed on [FoxTerrier GitHub repository](https://github.com/AssuranceMaladieSec/FoxTerrier) hosted under the Assurance Maladie IT Security team organization on GitHub :)

**Last but not least:** nothing would have been possible without the work of **some giants** aka [@_wald0](https://twitter.com/@_wald0), [@CptJesus](https://twitter.com/@CptJesus) and [@harmj0y](https://twitter.com/@harmj0y). **Warm thanks for creating Bloodhound and distributing it under a Free Software license!**

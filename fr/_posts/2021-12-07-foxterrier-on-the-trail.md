---
layout: post
title:  "FoxTerrier : Sur la trace des objets Active Directory vulnérables et d'un rapport"
date:   2021-12-07 11:00:00 +0200
lang: fr 
ref: foxterrier
author: acp 
categories: ActiveDirectory
---

{% include toc %}

# The FoxTerrier origin story

Like many companies, we are using the famous [BloodHound](https://github.com/BloodHoundAD/BloodHound) to identify vulnerable objects and dangerous path on our Active Directory.

We faced limitation in our use of the tool. Indeed, We needed:

- A report containing all vulnerable objects
- A synthesis of the vulnerable objects
- The ability to use multiple start nodes (objects) in input
- The ability to use de BloodHound databases (Neo4j) without the BloodHound GUI

> **Neo4j**: Neo4j is a graph database management system. The BloodHound environment uses Neo4j database to: 
- Store the Active Directory data retrieve by `SharpHound`
- Request these data. 

It is possible to use multiple start nodes by crafting and executing your own Cypher Queries in the BloodHound `Raw Query bar`. 

> **Cypher Queries**: Cypher is [Neo4j’s](https://neo4j.com) graph query language. A [Cypher Query](https://neo4j.com/developer/cypher/) is a query in the Cypher language.

However, Cypher Queries are complex and crafting requests manually everytime we need results is not an option in our case.


# Does BloodHound already provide reports? 

Unfortunately, no. BloodHound can provide a json of your current results or a png of it. And that's all. BloodHound is not design for this purpose. 

# What is FoxTerrier

FoxTerrier is a tool written in Python and ***working in the BloodHound environment***.

FoxTerrier can be seen as a more flexible version without GUI of BloodHound `OUTBOUND CONTROL RIGHTS` and `EXECUTION RIGHTS (RDP only)` features. In addition, FoxTerrier provides all the results in a `csv` file and a `.txt` summary of it.

FoxTerrier allows to identify, from a predefine list of user/groups, all the vulnerable objects (GPO, OU, User, Group, machine with RDP connection available for the object) that can be compromised by them.

But unlike the BloodHound `OUTBOUND CONTROL RIGHTS` feature, FoxTerrier allows you to request the type of object you want for specifics users/groups. 

For instance, If you want to retrieve only the vulnerable GPO that can be compromise by a list of predefine users/groups. You can !

Finally, predefine users and groups can be regex. It can be handy if, for instance, you have generic groups with pattern in the name and want to target all of them.

> **FoxTerrier relies on the Neo4j databases already filled with Active Directory information provided by SharpHound.**

# How FoxTerrier works

Like the BloodHound GUI, FoxTerrier works with Active Directory information retrieved by SharpHound and loaded in the Neo4j database.

Before using FoxTerrier make sure that:
- An Active Directory scan was performed by SharpHound
- The scan results are loaded in Neo4j

FoxTerrier creates specific Cypher Queries by using the data provided in the input file called `template.json` and request the Neo4j database. 

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
 - **objects_type** : The target objects can be `"GPO"`, `"OU"`, `"User"`, `"Group"`,`"RDP"`. **The values must be within a list**. *Default value : **["GPO", "OU" ,"User", "Group","RDP"]***
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





# The advantages of Fox Terrier

blablabla
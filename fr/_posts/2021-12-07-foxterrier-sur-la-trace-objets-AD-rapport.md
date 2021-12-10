---
layout: post
title:  "FoxTerrier : Sur la trace des objets Active Directory vulnérables et d'un rapport"
date:   2021-12-07 11:00:00 +0200
lang: fr 
ref: foxterrier
author: acp 
categories: ActiveDirectory
---
**TL;DR:** Vous utilisez Sharphound/BloodHound pour collecter/auditer votre AD et aimeriez pouvoir générer un rapport sur les objets vulnérables de votre AD ? Vous devriez jeter un oeil à FoxTerrier !

{% include toc %}

# Les origines de FoxTerrier

Comme beaucoup d'entreprises, nous utilisons le très connu [BloodHound](https://github.com/BloodHoundAD/BloodHound) (merci beaucoup [@_wald0](https://twitter.com/@_wald0), [@CptJesus](https://twitter.com/@CptJesus) et [@harmj0y](https://twitter.com/@harmj0y)!) afin d'identifier de potentiels objets vulnérables et chemins de compromission au sein de notre Active Directory (AD).

Cependant, au fil de nos audits AD, nous nous sommes heurtés à certaines limitations de l'outil. En effet, nous avions besoins de :
- rapports contenant l'ensemble des objets vulnérables ,
- synthèses des objets vulnérables ,
- utiliser plusieurs `start nodes` pour nos recherches
- et effectuer des recherches dans la base contenant les données de BloodHound (Neo4j) sans utiliser la GUI .

> **Neo4j**: Neo4j est un système de gestion de base de données basé sur les graphes. L'environnement Bloodhound utilise Neo4j pour: 
- stocker les données Active Directory récupérées par `SharpHound`
- requêter ces données 

Il est important de préciser qu'il est techniquement possible d'utiliser plusieurs `start nodes` pour ses recherches au sein de BloodHound. Cela demande de créer et exécuter ses propres `Cypher Queries` au sein de la `Raw Query bar` de BloodHound.

> **Cypher Queries**: Cypher est le langage de requête pour graphe de [Neo4j](https://neo4j.com). Une [Cypher Query](https://neo4j.com/developer/cypher/) est donc une requête du langage Cypher.

Cependant, les Cypher Queries sont complexes à appréhender. Et créer de toutes pièces ce type de requêtes chaque fois que cela était nécessaire n'était pas une option dans notre cas :-)

# BloodHound fournit-il des rapports ? 

Malheureusement non. BloodHound peut fournir un fichier json des résultats courants dans l'interface ou une image (png) de ceux-la.  BloodHound n'est pas conçu pour fournir des rapports de ces résultats. 

Vous pouvez via le [script `bloodhoundanalytics.py`](https://github.com/BloodHoundAD/BloodHound-Tools/blob/master/bloodhoundanalytics.py) du [dépôt officiel `BloodHound-Tools`](https://github.com/BloodHoundAD/BloodHound-Tools) récupérer des informations intéressantes et cela au format `xlsx`. Cependant, ce script n'est pas utile pour notre cas de figure (retrouver tous les objets vulnérables pour des utilisateurs/groupes spécifiques).

# FoxTerrier. Qui est-il ? Quels sont ses réseaux ?

FoxTerrier est un Logiciel Libre écrit en Python et ***fonctionnant au sein de l'écosystème BloodHound***. Le code source est disponible et les contributions sont possibles sur [le dépot GitHub de l'équipe sécurité de l'Assurance Maladie](https://github.com/AssuranceMaladieSec/FoxTerrier).

> FoxTerrier peut être vu comme **une version plus flexible et sans GUI des fonctionnalités `OUTBOUND CONTROL RIGHTS` and `EXECUTION RIGHTS (RDP only)` de BloodHound**. 
> 
> De plus, FoxTerrier fournit tous ses résultats au sein d'un fichier `csv` ainsi qu'une synthése de ceux-là dans un fichier `.txt`

FoxTerrier permet de :

* **définir plusieurs `Start node` (sans avoir à écrire de Cypher Queries)** :  Il identifie tous les objets vulnérables (GPO, OU, Utilisateur, Groupe, Machine où il est possible de se RDP) liés à une liste pré-définie d'utilisateurs/groupes.
* **définir le(s) type(s) d'objet(s) vulnérable(s) que l'on souhaite obtenir** : Contrairement à la fonctionnalité `OUTBOUND CONTROL RIGHTS` de Bloodhound, FoxTerrier permet d'affiner sa recherche sur des types spécifiques d'objets vulnérables.
 Par exemple, si vous souhaitez ne cibler que les GPOs vulnérables pour certains utilisateurs/groupes, vous pouvez :)
* **utiliser des regexp** : Des expressions régulières peuvent être utilisées au sein des noms de `start node`. Cela peut être pratique si vous souhaitez, par exemple, cibler en `start node` l'ensemble des utilisateurs/groupes répondant à un motif spécifique dans leur nom.

> **Pré-requis:** FoxTerrier nécessite une base Neo4j préalablement peuplée avec les données Active Directory recueillies par SharpHound.

# Comment FoxTerrier fonctionne-t-il

A l'instar de la GUI BloodHound, FoxTerrier travaille avec les données Active Directory récupérées préalablement par SharpHound et chargées au sein de la base Neo4j. 

Avant d'utiliser FoxTerrier, assurez vous :
- d'avoir un scan Active Directory effectué par SharpHound en votre possession ,
- d'avoir chargé ce scan dans la base Neo4j .

FoxTerrier créé **pour vous** des Cypher Queries spécifiques en utilisant les données fournies dans le fichier d'entrée nommé `template.json` et les utilise afin de requêter la base Neo4j.

![FoxTerrier Schema](/images/posts/requests.png)

> BloodHound et FoxTerrier peuvent être utilisé en simultané.

Pour fonctionner, FoxTerrier a besoin de 2 fichiers : `conf.ini` et `template.json`

## template.json: Le fichier d'entrée

Le fichier `template.json` contient l'ensemble des données utilisées par FoxTerrier pour créer des Cyper Queries personnalisées.

Au sein de ce fichier, 6 paramètres peuvent être utilisé:
- **node_start_type**	: <span style="color:red">*Obligatoire*</span>. Doit être `"User"` ou `"Group"`.
- **node_start_name**	: <span style="color:red">*Obligatoire*</span>. Peut être le nom complet ou une regexp (cf.`is_node_start_regex`). Exemple `"JOHN-DOE-825@MYDOMAIN.LOCAL"` ou `"JOHN-DOE-\\d{3}@MYDOMAIN.LOCAL"`.
- **is_node_start_regex** : Si une regexp est utilisé dans le paramètre `node_start_name`,la valeur ici doit être `true` (attention à bien utiliser `true` et non la chaîne avec des double quotes `"true"`). *Valeur par défaut : **false***
- **mode** : Les relations entre les `start node` et les objets peuvent être directes (permissions explicites sur un objet) ou indirectes (permissions héritées via l'appartenance à un groupe). Vous pouvez définir le mode de votre choix en choisissant les valeurs `"direct"`, `"indirect"` ou `"all"` (`all` correspond à "direct"+"indirect"). *Valeur par défaut : **"direct"***
 - **objects_type** : Les objets cibles peuvent être de type `"GPO"`, `"OU"`, `"User"`, `"Group"`,`"RDP"`. **Les valeurs définies doivent être au sein d'une liste**. *Valeur par défaut : **["GPO", "OU" ,"User", "Group", "RDP"]***
- **exclude_node** : Lors de l'utilisation d'une regexp au sein d'un `start node`, on peut être submergé par les résultats. Il est possible d'exclure des noms de `start node` spécifiques. **Les valeurs définies doivent être au sein d'une liste**. Exemple : `["GENERIC-GROUP-11111111@MONDOMAIN.LOCAL","GENERIC-GROUP-22222222@MONDOMAIN.LOCAL"]`

Voici un exemple de fichier `template.json` :

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
## conf.ini : Le fichier de configuration

Le fichier `conf.ini` contient les données de connexion à la base neo4j, les noms qui seront données au fichier de synthèse et au rapport et enfin le nom du fichier d'entrée à utiliser.
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
## Installation 

Pour "installer" FoxTerrier, vous avez seulement besoin de cloner le dépôt et installer le module `neo4j` pour python.

```
git clone https://github.com/AssuranceMaladieSec/FoxTerrier
pip install neo4j
```

## Utilisation

L'utilisation de FoxTerrier nécessite que **la base Neo4j soit démarrée et peuplée avec les données récupérées par SharpHound**.

Une fois vos fichiers `conf.ini` et `template.json` prêts, vous avez juste à lancer le script !

```
python FoxTerrier.py
```

# Exemple de résultats de FoxTerrier

## Un exemple de fichier de synthèse `summary.txt` :

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

## Un aperçu du contenu de `report.csv` :

Les résultats sont affichés ici dans un tableau pour des raisons de présentation. Bien évidemment, les données stockées dans le fichier de rapport sont en csv.

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

Selon nous, FoxTerrier peut vous être utile car il est :
- rapide : Puisque FoxTerrier n'utilise pas la GUI BloodHound et n'essaie pas d'afficher les résultats de ses requêtes. Il est plus rapide lors de l'utilisation de Cypher Queries utilisant plusieurs `start node`.
- flexible : Lorsque l'on utilise la fonctionnalité `Outbound Control Rights` sur un objet, tous les objets vulnérables (directe et indirecte) sont affichés. Avec FoxTerrier, vous pouvez choisir le type d'objet que vous souhaitez cibler.
- permet d'utiliser plusieurs `start node` : Vous pouvez définir plusieurs `start node` au sein du fichier d'entrée. Vous pouvez également utiliser des regexp pour cibler des `start node` avec des motifs au sein de leur nom.

Mais la seule façon d'être sur que FoxTerrier peut vous aider lors de vos audits d'Active Directory est de [le télécharger](https://github.com/AssuranceMaladieSec/FoxTerrier) et de l'utiliser ! 

Problémes, questions, envie de contribuer ? N'hésitez à faire tout cela sur [le dépôt GitHub de FoxTerrier](https://github.com/AssuranceMaladieSec/FoxTerrier) :)

Enfin, ce projet n'aurait pas vu le jour sans l'énorme travail déjà existant des developpeurs de BloodHound aka [@_wald0](https://twitter.com/@_wald0), [@CptJesus](https://twitter.com/@CptJesus) et [@harmj0y](https://twitter.com/@harmj0y). Merci d'avoir créer cet outil et de le distribuer sous une licence Libre !

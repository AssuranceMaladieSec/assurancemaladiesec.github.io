---
layout: post
title:  "implémentation security.txt à l'Assurance Maladie"
date:   2021-09-22 13:33:19 +0200
lang:   fr
ref:    securitytxtatassurancemaladie
author: cbrocas
categories: secops
---
**TL;DR:** Trouver comment rapporter des vulnérabilités détectées sur des sites ou des services web a toujours été un défi pour les chercheurs en Sécurité. Mais, publier les informations de contact de manière consistante au travers de tous leurs sites et services en ligne a toujours été aussi un défi pour les équipes de Sécurité défensive. Voyons si l'initiative `security.txt` peut nous aider !

{% include toc %}

# Rapporter les vulnérabilités de Sécurité : le(s) problème(s)

* **Du côté des chercheurs en Sécurité :** ils souhaitent trouver un moyen de rapporter les vulnérabilités qu'ils ont trouvées. Jusqu'à récemment, c'était uniquement possible de manière manuelle et insconsistante entre les sites : 
  * vérifier si un contact Sécurité figure quelque part sur le site concernée, 
  * utiliser une des adresses email généralement disponibles au sein des entreprises (abuse, admin etc) mais sans aucune assurance que leur rapport serait traité (correctement).
* **Du côté des équipes défensives :** dans les entreprises, trouver la bonne manière de mettre en place un canal de remontée et réussir à dépasser toutes les sortes de résistance qui peuvent exister (culturelle, organisationnelle, technique) ne sont pas des tâches aisées. 

**Impacts :**
* Pour les entreprises, cela entraine une **perte de remontées de vulnérabilités** en provenance des chercheurs en Sécurité.
* Pour ces chercheurs, **il s'agit d'une situation pénible** qui gagnerait grandement à être facilitée et fluidifiée.

# Défis rencontrés par l'Assurance Maladie

Comme beaucoup de grandes entreprises, nous opérons un grand nombre de sites web et de services web à destination de nos différents types d'utilisateurs (citoyens, professionels de santé et employeurs).

Ces services sont :
* hébergés/opérés dans différents endroits par différents fournisseurs (en interne, SaaS, cloud etc)
* propulsés par des piles applicatives très variées
* **important :** pilôtés par de très nombreuses structures de gestion de projets.

**Défis rencontrés:**
* **couverture des services :** comment afficher les instructions de remontée de vulnérabilités sur (presque) tous ces services ?
* **informations à jour :** comment être sûr d'afficher une version à jour des instructions de remontée de vulnérabilités sur (presque) tous ces services ?
* **accord du management des projets de la publication de ces instructions sur leur service :** comment convaincre les chefs de projets de tous ces services d'afficher ces instructions de remontée de vulnérabilités ? 
* **affichage consolidée entre les services :** si nous arrivons à convaincre tous ces chefs de projets de publier ces instructions, nous devons aussi nous assurer qu'ils vont le faire de manière consistante entre tous leurs services.

# La proposition : `security.txt`

[Security.txt](https://securitytxt.org/) est une proposition visant à standardiser la manière dont les entreprises documente sur chacun de leurs sites web comment ils veulent recevoir les signalements de vulnérabilités et comment ils vont les gérer.

Security.txt en détail:
* Il s'agit d'un [Draft Internet](https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt)
* Cette RFC définit notamment l'URI de type well-known `.well-known/security.txt` (et l'URI de repli `/security.txt`)
* Ces URI pointent vers le fichier `security.txt`.
* Le fichier `security.txt` contient plusieurs enregistrements standardisés permettant aux entreprises de publier, entre autres, leurs informations de contact, leur politique de gestion des rapports de vulnérabilités ouune éventuelle clé GPG à utiliser.

<u>Exemple:</u> nous voyons sur cette capture d'écran le contenu du fichier `security.txt` du [site web de Deepl](https://www.deepl.com/). Son contenu est affiché au travers de l'[extension Firefox security.txt](https://addons.mozilla.org/fr/firefox/addon/security-txt/) :

![security.txt example](/images/posts/securitytxt-example.png)

Tous les détails de l'initiative `security.txt` (URI, enregistrements etc) sont documentés  sur [https://securitytxt.org/](https://securitytxt.org/).

# Implémentation de `security.txt` par l'Assurance Maladie

Voyons comment nous l'avons déployé au sein de l'Assurance Maladie. 

## Situation initiale

Comme de nombreuses entreprises, nous ne disposions d'aucune information consistante au travers de nos sites et services web pour indiquer aux chercheurs en Sécurité comment rapporter les vulnérabilités qu'ils ont découvertes. Encore pire, par le passé, nous (équipe Sécurité) avons essayé d'afficher sur notre site principal l'adresse email de notre SOC (centre de Sécurité opérationnelle) à destination des personnes qui rencontreraient un problème de Sécurité sur le site. Et devinez quoi ? Notre SOC a immédiatement croulé sous les sollicitations concernant non pas la Sécurité mais seulement le métier. Un retour arrière a été effectué au bout de quelques heures. Le mode _"Apprentissage par l'échec"_ était activé ;-)

Du coup, quand l'initiative `security.txt` est apparue, nous l'avons examinée avec beaucoup d'attention.

## La manière naïve
Une manière simple mais néanmoins naïve de déployer un fichier `security.txt` est de demander à tous les chefs de projets de prendre ce fichier et de l'incorporer dans les livrables de leur site ou de leur service web.

![security.txt N copies](/images/posts/n-copies-of-security-txt.png)

Votre liste de tâches liée à `security.txt` va vite ressembler à :
* **Tâche n°1: essayer de convaincre & espérer atteindre l'exhaustivité.** L'équipe Sécurité va devoir se battre pour convaincre une majorité de ces chefs de projet.
* **Tâche n°2: atteindre & dépendre.** L'équipe Sécurité attend jusqu'à ce que le fichier soit déployé. Chaque projet va le déployer à son propre rythme et selon son propre agenda. 
* **Tâche n°3: demander la mise à jour & prier.** Vous allez avoir à refaire les mêmes actions à chaque mise à jour du fichier !

Comme vous pouvez le constater, un des pires cauchemars des équipes Sécurité n'est plus très loin (lire : _"la mise en oeuvre de la Sécurité dépendant de l'approbation par les projets"_) !

## La manière (un peu plus) intelligente

![security.txt N copies](/images/posts/centralized-security-txt.png)

Nous opérons quelqu'uns des sites les plus consultés en France (notre principal service a autour de 40 millions d'utilisateurs, des millions de connexions quotidiennes).

Du coup, nos services en ligne sont opérés derrière des équilibreurs de charge et des reverse proxies que **nous gérons**. Nous avons donc jugé intéressant d'utiliser ces composants centralisés de notre chaine de liaison pour notre problématique.

Plus précisément, nous avons décidé :
* **d'intercepter** les requêtes demandant les 2 URI `/security.txt` et `/.well-known/security.txt` sur tous les services derrière nos équilibreurs de charge/reverse proxies.
* **de récupérer le contenu du fichier `security.txt`** depuis un serveur centralisé (OK, depuis 2 serveurs pour la haute disponibilité) et de le renvoyer aux clients aillant solliciter ce contenu.

Beaucoup d'avantages émergent de cette décision technique :
* **Aucun impact sur nos sites ou services web**. Aucun déploiement n'est nécessaire sur les services, aucune mise à jour à gérer sur les serveurs web etc.
* **La mise en oeuvre de cette mesure de Sécurité ne dépend d'aucune décision des chefs de projets applicatifs**. Et _"c'est une Bonne Chose (tm)!"_.
* **Un fichier unique pour tous les services**. Par ce choix, vous gérez les informations liées à la remontée de vulnérabilités au travers d'un seul fichier. Son déploiement et ses éventuelles mises à jour seront simples et sous votre plein et seul contrôle.

**Bonus :** 
* **Moyen: signature électronique.** Comme nous fournissons une clé publique GnuPG aux chercheurs en Sécurité, nous avons décidé de signer électroniquement notre fichier `security.txt` avec la clé privée correspondante. En effet, la compromission des fichiers fait [partie du modèle de menaces](https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt#section-6.1) inclus dans la RFC.  
* **Impact : cela nous permet de publier le fichier `security.txt` d'une manière sécurisée sur des sites externes.** Nous pouvons ainsi demander à nos hébergeurs externes de déployer notre fichier `security.txt` (exemple pour [www.ameli.fr](https://www.ameli.fr/security.txt)) sans se soucier de sa modification silencieuse car nous pouvons en vérifier son intégrité à tout moment.

## Détails d'implémentation 

En regardant notre schéma actuel de remontée de vulnérabilités, nous avons décidé d'utiliser les enregistrements suivants au sein de notre [fichier security.txt](https://assure.ameli.fr/.well-known/security.txt):
* **`Contact:`** au travers de cet enregistrement, nous communiquons l'adresse email abuse@ en tant que point de contact principal.
* **`Preferred-Languages:`** les langages que nous parlons/comprenons.
* **`Encryption:`** URL de la clé publique GnuPG de l'adresse email abuse@.
* **`Policy:`** URL de notre politique de remontée de vulnérabilités.
* **`Acknowledgments:`** URL de la page où nous publions et remercions les chercheurs en Sécurité qui nous ont remonté de manière responsable des vulnérabilités.

Et comme indiqué précédemment, ce fichier est signé électroniquement avec la clé privée GnuPG de notre adresse abuse@.

**Pour héberger les ressources** pointées par les 3 derniers enregistrements, **nous avons décidé d'utiliser Github** car il s'agit d'un hébergement externe (qui peut être utile en cas de compromission massive sur notre infrastructure locale). 

Il s'agit aussi d'un hébergement sûr car nous avons décidé de **signer tous nos commits**, non seulement pour notre code mais aussi pour toutes nos ressources statiques comme notre politique de remontée de vulnérabilités ou notre fichier de clé publique GnuPG.

## Bénéfices sur l'aspect remontée de vulnérabilités

Au moment où nous écrivons ces lignes (fin septembre 2021), nous avons une année et demi de recul sur le déploiement du fichier `security.txt` sur la plupart de nos sites/services web (qui a eu lieu lors du premier trimestre 2020).

Au final, la vraie question est : _"Et au final, vous avez fait tout cela **pour quels résultats** ?"_

* **Avant le déploiement :** nous avions reçu **très peu voire aucun signalement externe** pendnat de nombreuses années.

* **Un an et demi après : nous avons reçu une douzaine de signalements de vulnérabilités** au travers de ce canal de remontée. Certains d'entre eux nous ont amené à nous pencher sur certains aspects de l'infrastructure de nos services qui n'avaient pas fait l'objet d'une attention poussée auparavant (composants logiciels, manière d'exposer certains services etc). Un autre point intéressant : nous n'avons reçu aucune sollicition inopportune (bruit) au travers de ce canal. Aucune fausse alerte, aucune sollicitation inadéquate. Merci à tous les chercheurs impliqués pour cela !

* **Pourquoi cela fonctionne-t-il ?** Après avoir demandé aux chercheurs en question (en général des professionnels en Sécurité ou des participants à des programmes de bug bounties), il s'avère que le fichier`security.txt` est utilisé par leurs soins de manière quotidienne comme moyen principal de contacter les équipes Sécurité afin de remonter les vulnérabilités découvertes.

* **Remerciements des chercheurs en Sécurité :** Après validation des vulnérabilités trouvées, et leur correction si besoin, **nous créditons les chercheurs** sur [notre page de remerciement](https://assurancemaladiesec.github.io/abuse/thanks/).

## Limites

Bien entendu, ce déploiement centralisé n'assure pas une couverture à 100% de nos services car nous avons aussi des hébergements externes ou en mode SaaS. Mais, vu que cela est loin de représenter une majorité de nos sites, nous pouvons arriver à un bon niveau de couverture global sans fournir un effort démesuré pour déployer un fichier `security.txt`sur ces services externes.

Il faut aussi ajouter que même si on déploie un fichier `security.txt`, **il ne s'agit pas d'une baguette magique !**

Avant d'y avoir recours, vous devez mettre en place en interne un circuit de traitement au sein de votre entreprise des vulnérabilités qui seront remontées. Sans ce travail, rien de miraculeux n'arrivera. 

Dans notre cas, mettre par écrit et en libre accès notre politique de remontée nous a aidé à clarifier le canal utilisé, les rôles de chaque entité ou comment nous souhaitons reconnaitre les chercheurs Sécurité par exemple.

# Conclusion

L'intiative `Security.txt` est une solution simple et peu coûteuse à mettre en oeuvre pour améliorer la situation d'un des défis pour les équipes Sécurité sur Internet : la publication de la méthode de remontée des vulnérabilités trouvées sur nos services. 

Utiliser _de manière intelligente_ votre pile technique pour déployer le fichier `Security.txt` sur la plupart de vos services est un moyen d'éviter le risque de blocage principal (organisationnel) que les équipes Sécurité rencontrent en entreprise, à savoir obtenir l'accord des projets sur une mesure Sécurité et leur participation active à sa mise en oeuvre.

# Resources
* Notre fichier security.txt : [https://assure.ameli.fr/.well-known/security.txt](https://assure.ameli.fr/.well-known/security.txt)
* Le site web de l'initiative security.txt : [https://securitytxt.org/](https://securitytxt.org/)
* La RFC Security.txt : [https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt-12](https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt-12)
* Tous les schémas ont été réalisé via le site [draw.io](https://app.diagrams.net/).
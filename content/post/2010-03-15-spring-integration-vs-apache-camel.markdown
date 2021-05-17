---
title: "Spring Integration vs. Apache Camel"
date: 2010-03-15 23:29:48 +0100
comments: true
sharing: true
footer: true
tags: 
- apache camel
- eip
- java
- jbi
- soa
- spring integration
---
![left-small](http://4.bp.blogspot.com/_XLL8sJPQ97g/S56PashF4SI/AAAAAAAAAJA/OVagPG4D-g0/s320/apache_spring.png)
Lors d'un [post précédent](/2009/12/eip-quest-ce-que-cest.html), j'avais parlé des [EIPs](/blog/2009/12/13/eip-quest-ce-que-cest/) (__Enterprise Integration Patterns__) en expliquant qu'il s'agissait de Patterns permettant de normaliser les échanges de messages dans un système asynchrone.

Dans cet article, je vais tenter de présenter succinctement deux de ses implémentations : [Spring Integration](http://projects.spring.io/spring-integration/) et [Apache Camel](https://camel.apache.org/.

En fait, pour être plus précis, je vais plutôt tenter de présenter la vision que j'en ai ainsi que la façon dont je les ai compris.

Ainsi, ce post n'a pas pour objectif de les détailler de manière exhaustive car ils sont trop complets pour cela et qu'un seul post ne pourrait suffire à les aborder tous les deux (leurs documentations font d'ailleurs, pour Spring Integration, plus de 130 pages, et pour Apache Camel, plus de 580 pages... cqfd... ;-) ) mais juste, comme je l'ai dit précédemment dit, d'aider à comprendre leurs différences (quelles soient conceptuelles ou structurelles).

Je ne reviendrai pas sur les concepts des EIPs ni de JBI que j'utiliserai dans la suite et pour cela, je vous renvoie sur internet ou sur mes posts précédents ([ici](/2009/12/eip-quest-ce-que-cest.html) et [là](/2009/12/jbi-une-solution-enterree.html)).

Concernant les versions utilisées, cela n'a pas vraiment son importance ici car je m'intéresserai surtout aux principes de ces deux frameworks mais à titre indicatif, il s'agit des versions 2.0 pour Apache Camel et 1.3.0 pour Spring Integration (il me semble qu'il n'y a pas de modifications flagrantes dans les versions courantes qui sont 2.2.0 pour Camel et 2.0.0.M2 pour Spring Integration).

<!-- more -->
# Les concepts
## Apache Camel
Apache Camel est un framework open-source qui est une implémentation des EIPs et qui propose pour sa configuration d'utiliser indifféremment :

* un fichier de configuration Spring (Spring DSL), 
* un DSL interne (Java DSL), 
* ou un DSL externe (Scala DSL).

Cependant, il définit ses propres concepts qui diffèrent de ceux des EIPs mais qui restent assez similaires.

Ainsi, Apache Camel s'appuie sur la notion de routes (__Route__) qui relient deux points d'accès (__Endpoint__). Sur cette route transite des messages (__Message__) au travers d'échanges (__Exchange__). En fait un Exchange est le conteneur du message durant la phase de routage.

Ces points d'accès peuvent être de différents types et peuvent supporter différentes technologies telles que :

* une destination (Topic ou Queue) JMS,
* un service web,
* un fichier se trouvant sur le système de fichier,
* un serveur FTP,
* une adresse mail,
* ou encore un POJO (_Plain Old Java Object_).

Si l'on fait un parallèle avec JBI (_Java Business Integration_) (cf. [ici](/2009/12/jbi-une-solution-enterree.html)), un __Endpoint__ peut être vu comme une instance d'un __Binding Component__.

Une __Route__ peut, quant à elle, être vue comme le chemin qu'emprunte le message d'un bout à l'autre de la chaîne de médiation. Ce chemin pouvant être entrecoupé par ce qu'on appelle des processeurs (__Processor__).

Apache Camel introduit également la notion de Composants (__Component__) à partir desquels les __Endpoint__ sont issus (toujours pour faire un parallèle avec JBI, un __Component__ peut être vu comme un __Composant JBI__ en mode fournisseur ou consommateur).

Enfin, par composition à un __Endpoint__, Apache Camel utilise également les notions de __Producer__ et __Consumer__ qui permettent d'émettre (respectivement, recevoir) un message vers (resp. d') une application extérieure et qui peuvent être vu comme des __Binding Component__ en mode fournisseur (resp. consommateur).

# Spring Integration
Spring Integration appartient au portfolio de Spring Framework et étend le modèle de programmation de Spring mais dans le domaine de la messagerie. Il supporte l'architecture basée sur les messages (_Message-Driven Architecture_) où l'inversion de contrôle est utilisée pour les problématiques d'exécution comme, par exemple, quand doivent être appelés les composants métiers ou encore où doivent être envoyées les réponses. En outre, il offre des mécanismes de routage et de transformation de messages afin de permettre aisément l'utilisation de protocoles de transport et des types de messages hétérogènes. Il propose pour sa configuration d'utiliser soit un fichier de configuration Spring soit le mécanisme d'annotations.

Concernant ses concepts, ils collent parfaitement à ceux des EIPs puisque Spring Integration utilise la notion de :

* __Message__ dont la définition est identique à celle des EIPs, 
* __Message Channel__ dont la définition est identique à celle des EIPs,
* et de __Message Endpoint__ qui regroupe :
	* les Message Routing dont la définition est identique à celle de Routing des EIPs
	* les Message Transformation dont la définition est identique à celle de Transformation des EIPs
	* et les Message Endpoint dont la définition est identique à celle des EIPs

# Mon avis avant utilisation
## Apache Camel

Apache Camel est un framework issu de l'implémentation JBI Apache ServiceMix et qui est également utilisé dans OpenESB. Cela peut expliquer pourquoi ses concepts sont si proches de ceux de JBI (ou du moins que les modèles sont si facilement transposables). Cependant, à mon sens, même si ses concepts sont proches de ceux de JBI, les termes utilisés sont différents et, pour ceux qui connaissent JBI, un effort d'apprentissage supplémentaire doit être fait. En outre, la documentation d'Apache Camel est loin d'être aisée à lire et retrouver de l'information dans le wiki qui lui sert de documentation s'achèvent généralement avec des cheveux en moins, une souris torturée ou un écran ébréché... le tout accompagné par une flopée d'injures...

Ainsi, à mon avis, pour un produit qui se veut être une implémentation des EIPs, il faut :

* d'une part, lire ou se familiariser avec les EIPs
* et d'autre part, connaitre JBI ou, au moins, se familiarisé avec les concepts d'Apache Camel (comprendre la différence entre un composant, un endpoint qui est lui-même un processeur, les éléments pipeline et multicast qui sont eux-même des processeurs, etc, etc, etc...). Rien d'insurmontable mais il est quand même nécessaire de se faire quelques nœuds au cerveau...

Par contre, par rapports aux EIPs, Apache Camel masque la notion de Channel, ce qui peut s'avérer plaisant.

## Spring Integration
Spring Integration est un projet du portfolio Spring donc très propre, avec une documentation très bien faite et pas trop longue... En outre, pour quelqu'un qui est déjà familiarisé avec les EIPs, la prise en main de Spring Integration est immédiate. Le fait qu'il s'appuie sur Spring (tout comme Apache Camel d'ailleurs) permet une courbe d'apprentissage rapide.

# Mon avis après utilisation
## Apache Camel
Apache Camel s'avère ardu à prendre en main malgré un forum actif. Cela en raison de sa documentation un peu (complètement?) fouillis mais également parce qu'il est possible de faire la même chose de multiples manières, ces manières ayant chacunes ses limitations. Si on utilise le framework simplement avec une description XML et des POJOs (qui sont, parfois, fortement couplé au framework Apache Camel au travers de son API (Exchange, Endpoint, ...) ... pas très propre tous ça... :( ), on peut arriver sans trop de douleur à faire ce que l'on veut. Par contre, si on se lance dans les DSLs internes ou externes (et même si j'aime bien le principe des DSLs...), cela n'exclue pas l'adhérence des POJOs à l'API d'Apache Camel et il est souvent nécessaire de faire intervenir la notion d'Exchange dans les signatures des méthodes.

En outre, on se rend rapidement compte que la majorité des classes de l'API héritent de l'interface `Processor`. Pourquoi cela me pose problème? et bien parce que j'estime qu'il est bizarre et peu naturel que certaines classes comme Pipeline ou MulticastProcessor implémente (directement ou indirectement) Processor ou alors il aurait, au moins, été préférable de mettre ce type de classes dans un autre package que celui où se trouvent des classes comme Aggregator, FileProcessor, Resequencer ou encore RecipientList. Bon, je comprends qu'Apache Camel a été pensé dès le départ pour offrir une configuration simple à base de DSL et que, du coup, il a été nécessaire de faire certaines concessions mais lorsque l'on rentre dans le code du framework, ce n'est pas (et cela ne concerne que moi) une impression de propreté qui en ressort...

Un autre point qui m'interpelle est le fait que lors de l'écriture des règles de médiations, il n'y a pas de différences sémantiques entre les différentes notions qu'offrent les EIPs : les Message Transformation, Message Routing ou Service Activator se représentent tous comme des Processor ou des Bean. Je trouve cela dommage de perdre la classification (qu'on aime ou pas) fournit par les EIPs et qui, à mon sens, permettait justement de clarifier toutes les notions utiles pour gérer les échanges de messages dans un système asynchrone.

Par contre, comme je l'ai dit précédemment, il est vraiment appréciable de ne pas à avoir à déclarer ses propres Channels qui sont masqués par la notion de Route.

## Spring Integration
Spring Integration est extrèmement simple à prendre en main et son forum est très actif. La documentation est claire et c'est vrai qu'il est appréciable de pouvoir utiliser les notions des EIPs directement.

Les annotations sont parlantes et les éléments (service-activator, splitter, filter, resequencer, aggregator, transformer, router, ...) à utiliser dans le fichier de configuration Spring permettent de classifier rapidement les différents éléments se trouvant dans la chaine de médiations.

En outre, le code interne du framework est propre et permet de masquer la complexité des différents Message Endpoint.

# Conclusion
Vous l'aurez compris, j'ai une petite préférence pour Spring Integration...

Cependant, je dirai que le choix dépend aussi grandement de l'utilisation qu'on veut en faire (bon ok, je ne me mouille pas trop... ;-) ) :

* dans un ESB type ServiceMix ou OpenESB, il peut être préférable d'utiliser Apache Camel en raison de sa capacité à être configurée via un DSL qui peut être plus aisé que de manipuler des Channels et des fichiers XML,
* par contre, pour les autres cas, je conseillerai plutôt Spring
Integration pour son modèle de programmation et ses concepts plus
simple...

mais là, je conseillerai à chacun de se faire sa propre idée...

Un petit mot en plus sur les points non abordés ci-dessus :

* les mécaniques offertes pour manipuler des messages au format XML (XPath, ...),
* la gestion des erreurs,
* le nombre de connecteurs fournis par ces derniers.

Sur ces deux derniers points, l'avantage va indéniablement à Apache Camel même s'il est aisé (modulo la connaissance du protocole utilisé) d'en redéfinir avec Spring Integration en utilisant un Service Activator mais, dans ce cas, il faut le faire à la main.

Pour le premier point, je ne dirai rien car je n'ai pas eu l'occasion de l'utiliser...

Enfin, il ne faut pas oublier que ces frameworks ont pour but principal d'offrir un moyen de faire de la médiation technique et non de gérer une orchestration de processus métier et que pour cette raison, ils doivent être utilisés à bon escient.

# Pour aller plus loin...

* __SOA : le guide de l’architecte__ de X. Fourner-Morel, P. Grojean, G. Plouin et C. Rognon chez Dunod
* __Enterprise Integration Patterns__ de G. Hohpe et B. Woolf chez Addisson Wesley
* Site de Spring Integration : http://www.springsource.org/spring-integration
* API de Spring Integration : http://static.springframework.org/spring-integration/apidocs/
* Manuel de référence de Spring Integration : http://docs.spring.io/spring-integration/docs/4.0.0.M3/reference/html/
* Site d'Apache Camel : http://camel.apache.org/
* API d'Apache Camel : http://camel.apache.org/javadoc.html
* Manuel de référence d'Apache Camel : http://camel.apache.org/manual.html
* Spécification de JBI : http://jcp.org/aboutJava/communityprocess/edr/jsr208/index.html
* Blog de Xebia sur les ESB légers : http://blog.xebia.fr/2007/12/17/spring-integration-lavenement-des-lightweight-esb/
* Blog de Xebia sur une revue de presse d'un article sur Spring Integration et Apache Camel : http://blog.xebia.fr/2010/01/04/revue-de-presse-xebia-141/#SpringIntegrationetApacheCamel

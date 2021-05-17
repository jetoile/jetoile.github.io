---
title: "JMX pour les nuls... - Les concepts - Partie 1"
date: 2010-10-27 12:01:35 +0100
comments: true
tags: 
- java
- jmx 
---
![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)
Comme promis lors de mon [article précédent](/2010/10/jmx-pour-les-nuls-introduction.html) (et pour éviter de vous faire languir ;-) ) la première partie de cette série d'article sur JMX!

<!-- more -->

#Table des matières

* JMX, qu'est ce que c'est?
	* [Généralités](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#generalite)
	* [Architecture JMX](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#architecture)
		* [Niveau instrumentation](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#instrumentation)
		* [Niveau agent](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#agent)
		* [Niveau service distribué](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#distribue)
	* [Composants JMX](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#composant)
		* [MBeans](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#mbean)
		* [Modèle de notifications](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#notification)
		* [Classe de métadonnées de MBeans](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#metadonnee)
		* [Serveur de MBeans](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#serveur)
		* [Service d'agents](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html#service)
* Spécifications
	* [JMX Instrumentation](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html)
		* [MBean](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html#mbean)
			* [MBean Standard](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html#mbean_standard)
			* [Dynamic MBean](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html#mbean_dynamic)
		* [Notification](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html#notification)
		* [Open MBean](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html#mbean_open)
		* [Model MBean](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html#mbean_model)
	* [Agent JMX](/2010/11/jmx-pour-les-nuls-les-agents-jmx-partie.html#agent)
	* [Concepts](/2010/11/jmx-pour-les-nuls-les-classes-de-base.html)
		* [ObjectName](/2010/11/jmx-pour-les-nuls-les-classes-de-base.html#objectName)
		* [ObjectInstance](/2010/11/jmx-pour-les-nuls-les-classes-de-base.html#objectInstance)
		* [Attribute et AttributeList](/2010/11/jmx-pour-les-nuls-les-classes-de-base.html#attribute)
		* [Les Exceptions](/2010/11/jmx-pour-les-nuls-les-classes-de-base.html#exception)
	* [MBean Server](/2010/11/jmx-pour-les-nuls-le-mbean-server.html#mbean_server)
	* [Chargement dynamique des MBeans](/2010/12/jmx-pour-les-nuls-chargement-dynamique.html#mbean_dynamic)
	* [Les services JMX](/2010/12/jmx-pour-les-nuls-les-services-jmx.html)
		* [Service Monitoring](/2010/12/jmx-pour-les-nuls-les-services-jmx.html#monitoring)
		* [Service Timer](/2010/12/jmx-pour-les-nuls-les-services-jmx.html#timer)
		* [Service Relation](/2010/12/jmx-pour-les-nuls-les-services-jmx.html#relation)
		* [Service Sécurité](/2010/12/jmx-pour-les-nuls-les-services-jmx.html#securite)
	* [Les Connecteurs](/2010/12/jmx-pour-les-nuls-les-connecteurs.html#connector)


#JMX, qu'est ce que c'est?

<a name="generalite"></a>
##Généralités
_Java Management eXtensions_ (JMX) définit une architecture, un design pattern, une API et les services permettant de superviser et d’administrer une application au travers du langage Java.

JMX permet, en effet :

* d’administrer et de superviser une application Java sans avoir à investir dans de couteux serveurs : l’architecture JMX repose sur un serveur d’objets de base qui peut être vu comme un agent de supervision et d’administration et qui peut fonctionner sur presque toutes les JVM.
* d’offrir une architecture de supervision et d’administration scalable : chaque agent JMX est un module indépendant pouvant se brancher à l’agent de supervision et d’administration de l’application à surveiller.
* de s’interfacer avec une autre solution de supervision et d’administration : les agents JMX peuvent être supervisés et administrés aux travers de différents protocoles (HTTP, SNMP, …) et l’API JMX permet de définir le sien.  
* de s’intégrer avec les autres technologies Java présentes dans le système telles que JNDI, JDBC, JINI, SLP, …

<a name="architecture"></a>
##Architecture JMX
L’architecture JMX peut se décomposer en trois niveaux :

* le niveau instrumentation (__instrumentation level__),
* le niveau agent (__agent level__),
* et le niveau service distribué (__distributed services level__).

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TMdQmiH7InI/AAAAAAAAALY/rXNS-iCKg6A/s1600/jmx-01.png)

<a name="instrumentation"></a>
###Niveau instrumentation
Le niveau instrumentation spécifie comment implémenter des ressources administrable et supervisable par JMX ; une ressource pouvant être une application, une implémentation d’un service, un périphérique, ou autre tant qu’elle est développée en Java (ou qu’un wrapper existe) et qu’elle a été instrumentée pour être compatible JMX.

L’instrumentation d’une ressource donnée est fournit par la présence d’un ou plusieurs Beans supervisables et administrables (ou MBeans pour _Managed Beans_) qui peuvent être de plusieurs sortes :

* __MBean standard__,
* __MBean dynamique__.

Ainsi, l’instrumentation d’une ressource permet de la rendre administrable et supervisable au travers du niveau agent. Cela permet aux MBean de s’abstraire du niveau agent et ainsi que de les rendre plus simple et évolutifs. 

<a name="agent"></a>
###Niveau agent

Le niveau agent spécifie comment implémenter les agents qui permettent le contrôle direct des ressources et qui les rend administrable et supervisable d’une application de supervision et d’administration distante. Les agents se trouvent être généralement sur la même machine que les ressources qu’elles contrôlent mais cela n’est pas un pré-requis.

Ce niveau s’appuie et utilise le niveau d’instrumentation pour définir des agents standards utilisés pour se superviser et s’administrer elle-même.

L’__agent JMX__ est constitué d’un serveur de MBeans (__MBeanServer__) et d’un ensemble de services permettant de manipuler les MBeans. En outre, le MBeanServer nécessite au moins un connecteur ou adaptateur.
L’agent JMX peut être embarqué dans la machine virtuelle Java qui accueille les ressources JMX à administrer et superviser, ou peut être instancié dans un élément de médiation quand la ressource n’est pas une ressource Java.

Enfin, l’agent JMX n’a pas besoin de connaitre les ressources qui doivent être administrées et supervisées.
Le manageur accède donc au MBeans de l’agent et utilise les services fournis par ce dernier au travers un adaptateur de protocoles ou un connecteur. Celui permet à l’agent JMX d’être indépendant et faiblement couplé à l’application de management.

<a name="distribue"></a>
###Niveau service distribué

Le niveau service distribué spécifie comment implémenter l’interface d’administration et de supervision JMX. Ce niveau permet de définir les interfaces d’administration et de supervision et les composants qui exploitent les agents ou la hiérarchie d’agents JMX. Ces composants peuvent :

* fournir une interface d’administration et de supervision des applications via leurs agents JMX au travers d’un connecteur,
* exposer une vue d’administration et de supervision d’un agent JMX et de ses MBeans en associant leur signification sémantique avec un protocole haut niveau (comme HTML ou SNMP),
diffuser les informations de supervision d’une plate-forme de supervision et d’administration à différents agents JMX,
* agréger et consolider les informations de supervision et d’administration en provenance de différents agents JMX dans une vue logique à destination de l’utilisateur final,
* gérer et sécuriser l’administration et la supervision du système.

En fait, les composants d’administration et de supervision peuvent interagir les uns les autres au sein du réseau afin de fournir des fonctions d’administration et de supervision scalables et distribuées.

<a name="composant"></a>
##Composants JMX

JMX s’appuie sur les composants et notions suivants :

* __MBeans__ (standard, dynamic, open et model)
* __Modèle de notifications__ (Notification Model)
* __Classe de métadonnées de MBeans__ (MBean metadata classes)
* __Serveur de MBeans__ (MBean Server) 
* __Services d’agents__ (Agent Services)

<a name="mbean"></a>
###MBeans

Un MBean est un objet Java qui implémente une interface spécifique et qui respecte un design pattern particulier. Cela permet de normaliser la représentation de l’interface des ressources à gérer dans le MBean. Cette dernière est composée de l’ensemble des informations et des méthodes nécessaires à l’administration et à la supervision de l’application.

L’interface de gestion d’un MBean peut être composée :

* d’attributs dont les valeurs peuvent être accédées,
* d’opérations qui peuvent être invoquées,
* de notifications qui peuvent être émises,
* de constructeurs (au sens Java du terme)

En fait, un MBean encapsule les attributs et les opérations au travers de leurs méthodes publiques : par exemple, un attribut en lecture-seule dans un MBean standard n’aura qu’un _getter_ alors qu’un attribut accessible en lecture-écriture disposera d’un _getter_ et d’un _setter_.

Ainsi, n’importe quel objet qui implémentera un MBean et qui sera déclaré auprès de l’agent JMX pourra être géré.

Les autres types de composants JMX comme les services d’agents sont des MBeans complètement instrumentés leur permettant de bénéficier de l’infrastructure JMX.

La spécification JMX définit quatre types de MBean :

* __Standard MBean__ : c’est l’implémentation de MBean la plus simple. Elle fournit une représentation statique des ressources à gérer et à superviser. Son interface de gestion est décrite par le nom de ses méthodes.
* __Dynamic MBean__ : ce MBean doit implémenter une interface spécifique mais il expose ses interfaces de gestion de manière dynamique à l’exécution.
* __Open MBean__ : ce MBean est une extension du Dynamic MBean mais repose sur une représentation normée de ses types de données afin de permettre plus d’interopérabilité. Pour ce faire, il s’auto-décrit. 
* __Model MBean__ : ce MBean est une extension du Dynamic MBean mais est complètement configurable et s’auto-décrit à l’exécution. Il est à noter qu’il fournit une classe générique de MBean qui définit son comportement par défaut qu’il est possible de redéfinir.

<a name="notification"></a>
###Modèle de notifications

JMX définit un modèle de notifications génériques qui se base sur le modèle d’événements Java. Les notifications peuvent être émises par une instance de MBean mais aussi par le serveur de MBean lui-même. 

<a name="metadonnee"></a>
###Classe de métadonnées de MBeans

JMX définit comment les classes doivent décrire les interfaces de gestion des MBean. Ces dernières servent à construire la structure des informations à publier qui est utilisée par le serveur de MBeans. Ainsi, ces classes de métadonnées contiennent la structure qui décrit tout ce dont il a besoin : ses attributs, ses opérations, ses notifications et ses constructeurs mais aussi les caractéristiques de chacun de ces éléments comme la signature des méthodes ou le niveau d’accessibilité des attributs.   

<a name="serveur"></a>
###Serveur de MBeans

Le serveur de MBean peut être vu comme un registre des objets qui sont gérables au travers de l’agent. Ainsi, chaque objet enregistré dans le serveur de MBean est visible de l’application de gestion. 

Cependant, il n’expose jamais les objets directement mais seulement leurs interfaces même s’il fournit une interface standardisée d’accès aux MBeans présent dans la même JVM.

Un MBean doit donc être enregistré de manière unique au sein du serveur de MBean et peut l’être par :

* un autre MBean,
* l’agent lui-même,
* une application de gestion distante au travers d’un service distribué. 

Les opérations alors disponibles sur un MBean sont :

* la découverte de l’interface de gestion du MBean,
* la lecture et l’écriture de la valeur des attributs,
* l’invocation des opérations offertes par l’interface de gestion du MBean,
* la récupération des notifications émises par le MBean.

Pour permettre à un agent d’être accessible depuis une application extérieure, le MBean Server s’appuie sur les adaptateurs de protocoles et les connecteurs.

<a name="service"></a>
###Service d’agents

Les services d’agents sont les objets qui gèrent les différentes opérations de gestion des MBean enregistrés dans le serveur de MBeans. 

Ces services d’agents sont généralement eux-mêmes des MBeans et permettent ainsi au serveur de MBeans de les gérer.

Par défaut, doivent être présents les services suivants : 

* __Management Applet__ (m-let) qui permet le chargement dynamique de classes
* __Service Timers__ qui permet de déclencher des actions sur les MBeans de façon régulière
* __Service Relation__ qui permet d’associer des MBeans entre eux
* __Service Monitor__ qui permet de scruter certaines valeurs d’attribut afin de notifier d’autres objets lors d’un changement

#Le mot de la fin de cette partie

Dans cet article, nous avons vu (ou revu) les concepts de bases de JMX. Dans les articles suivants, nous entrerons plus dans les entrailles des spécifications en nous calquant sur les spécifications JMX pour tenter d'y voir un peu plus clair...
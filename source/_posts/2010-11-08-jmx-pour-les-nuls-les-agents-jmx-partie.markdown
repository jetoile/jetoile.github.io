---
layout: post
title: "JMX pour les nuls... - Les agents JMX - Partie 3"
date: 2010-11-08 15:50:22 +0100
comments: true
categories: 
- java
- jmx
---

![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)
Cette troisième partie sur JMX reste dans la continuité de la série d'article sur ce sujet (cf. [introduction](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/), [partie 1](/blog/2010/11/03/jmx-pour-les-nuls-dot-dot-dot-les-differents-mbeans-et-la-notion-de-notification-partie-2/) portant sur les généralités et partie 2 portant sur les différents MBeans et le concept de Notification) en introduisant plus précisément ce qu'est un agent JMX et à quoi il sert.

Les articles qui feront suite préciseront les concepts manipulés par un agent JMX beaucoup plus en profondeur.

<!-- more -->

#Table des matières

* JMX, qu'est ce que c'est?
	* [Généralités](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#generalite)
	* [Architecture JMX](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#architecture)
		* [Niveau instrumentation](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#instrumentation)
		* [Niveau agent](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#agent)
		* [Niveau service distribué](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#distribue)
	* [Composants JMX](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#composant)
		* [MBeans](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#mbean)
		* [Modèle de notifications](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#notification)
		* [Classe de métadonnées de MBeans](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/metadonnee)
		* [Serveur de MBeans](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#serveur)
		* [Service d'agents](/blog/2010/10/27/jmx-pour-les-nuls-dot-dot-dot-les-concepts-partie-1/#service)
* Spécifications
	* [JMX Instrumentation](/blog/2010/11/03/jmx-pour-les-nuls-dot-dot-dot-les-differents-mbeans-et-la-notion-de-notification-partie-2/)
		* [MBean](/blog/2010/11/03/jmx-pour-les-nuls-dot-dot-dot-les-differents-mbeans-et-la-notion-de-notification-partie-2/#mbean)
			* [MBean Standard](/blog/2010/11/03/jmx-pour-les-nuls-dot-dot-dot-les-differents-mbeans-et-la-notion-de-notification-partie-2/#mbean_standard)
			* [Dynamic MBean](/blog/2010/11/03/jmx-pour-les-nuls-dot-dot-dot-les-differents-mbeans-et-la-notion-de-notification-partie-2/#mbean_dynamic)
		* [Notification](/blog/2010/11/03/jmx-pour-les-nuls-dot-dot-dot-les-differents-mbeans-et-la-notion-de-notification-partie-2/#notification)
		* [Open MBean](/blog/2010/11/03/jmx-pour-les-nuls-dot-dot-dot-les-differents-mbeans-et-la-notion-de-notification-partie-2/#mbean_open)
		* [Model MBean](/blog/2010/11/03/jmx-pour-les-nuls-dot-dot-dot-les-differents-mbeans-et-la-notion-de-notification-partie-2/#mbean_model)
	* [Agent JMX](/blog/2010/11/08/jmx-pour-les-nuls-dot-dot-dot-les-agents-jmx-partie-3/#agent)
	* [Concepts](/blog/2010/11/21/jmx-pour-les-nuls-dot-dot-dot-les-classes-de-base-partie-4/)
		* [ObjectName](/blog/2010/11/21/jmx-pour-les-nuls-dot-dot-dot-les-classes-de-base-partie-4/#objectName)
		* [ObjectInstance](/blog/2010/11/21/jmx-pour-les-nuls-dot-dot-dot-les-classes-de-base-partie-4/#objectInstance)
		* [Attribute et AttributeList](/blog/2010/11/21/jmx-pour-les-nuls-dot-dot-dot-les-classes-de-base-partie-4/#attribute)
		* [Les Exceptions](/blog/2010/11/21/jmx-pour-les-nuls-dot-dot-dot-les-classes-de-base-partie-4/#exception)
	* [MBean Server](/blog/2010/11/29/jmx-pour-les-nuls-dot-dot-dot-le-mbean-server-partie-5/#mbean_server)
	* [Chargement dynamique des MBeans](/blog/2010/12/06/jmx-pour-les-nuls-dot-dot-dot-chargement-dynamique-de-mbeans-partie-6/#mbean_dynamic)
	* [Les services JMX](/blog/2010/12/13/jmx-pour-les-nuls-dot-dot-dot-les-services-jmx-partie-7/)
		* [Service Monitoring](/blog/2010/12/13/jmx-pour-les-nuls-dot-dot-dot-les-services-jmx-partie-7/#monitoring)
		* [Service Timer](/blog/2010/12/13/jmx-pour-les-nuls-dot-dot-dot-les-services-jmx-partie-7/#timer)
		* [Service Relation](/blog/2010/12/13/jmx-pour-les-nuls-dot-dot-dot-les-services-jmx-partie-7/#relation)
		* [Service Sécurité](/blog/2010/12/13/jmx-pour-les-nuls-dot-dot-dot-les-services-jmx-partie-7/#securite)
	* [Les Connecteurs](/blog/2010/12/20/jmx-pour-les-nuls-dot-dot-dot-les-connecteurs-partie-8/#connector)

<a name="agent"></a>
#Agent JMX

Un agent JMX est une entité de management qui s’exécute sur une machine virtuelle Java et qui permet de faire la liaison entre les MBean et l’application de management. Un agent JMX est composé d’un MBean server, d’un ensemble de MBean représentant les ressources à administrer et superviser, d’un ensemble minimal de services d’agent qui se présentent sous forme de MBean et d’au moins un adaptateur de protocol ou connecteur.

L’architecture d’un agent JMX peut donc se définir comme tel :

* Les MBeans qui représentent les ressources à administrer et superviser
* Le MBean server qui est la clé de voute de l’architecture d’un agent JMX
* Les services d’agent qui peuvent être des composants pré-définis ou des services spécifiques et  qui peuvent être classés de la manière suivante :
	* Services d’agent dynamiques qui permettent à l’agent d’instancier des MBeans qui utilisent indifféremment des classes Java ou des librairies natives qui peuvent être téléchargées dynamiquement sur le réseau
	* Services d’agent de management utilisés pour superviser la valeur des attributs des MBean : le service notifie sous certaines conditions ses écouteurs lors du changement de valeur d’un attribut du MBean
	* Services timer d’agent qui permet d’émettre périodiquement des notifications
	* Services de relation d’agent qui définit les associations entre les MBeans et qui est garant de la consistence des relations

Les applications de management peuvent accéder à l’agent JMX aux travers de différents adaptateurs de protocole et connecteurs. Ces objets font parti intégrantes de l’application agent mais ne sont pas définis par les spécifications JMX.

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TNHQb2x4MpI/AAAAAAAAANg/OPcRpH9yrII/s1600/jmx24.png)

L’architecture JMX permet aux objets d’invoquer les opérations suivantes sur un agent JMX :

* Gérer les MBeans existants en :
	* Récupérant la valeur de ses attributs
	* Changeant la valeur de ses attributs
	* Invoquant des opérations sur ce dernier
* Récupérer les notifications émises par n’importe quel MBean
* Instancier et enregistrer des MBeans à partir de :
	* Classes Java déjà présentent dans la JVM de l’agent JMX
	* Classes Java téléchargées d’une autre machine ou présentent sur le réseau
* Utiliser les services d’agent pour implémenter une stratégie de management qui impactera les MBean existants.

Il est à noter que ces objets peuvent être indifférement du coté de l’agent ou dans une application de management distante.

Les adaptateurs de protocole et les connecteurs rendent l’agent accessible d’une application de management distante et offrent une vue, au travers d’un protocole spécifique, des MBeans instanciés et enregistrés dans le MBean Server.

Les connecteurs sont utilisés pour connecter un agent JMX à une application de management qui supporte JMX, c'est-à-dire une application développée en utilisant des services distribués tels que définis par la spécification JMX. Ce type de communication implique une couche serveur dans l’agent et une couche cliente dans l’application de management. 

Un connecteur est spécifique à un protocole donné mais l’application de management peut posséder plusieurs connecteurs indifféremment puisqu’ils ont la même interface.

Un adaptateur de protocole fournit une interface de management de l’agent JMX au travers un protocole donné. Il adapte les opérations du MBean et du MBean Server en une représentation de ce protocole éventuellement avec un modèle différent des informations (comme dans le cas de SNMP). Dans ce cas, l’application de gestion est généralement spécifique au protocole utilisé : elles accèdent à l’agent JMX non à travers une représentation de MBean Server distant mais au travers des opérations qui s’interfacent à celles offertes par le MBean Server.
Les serveurs de connecteurs et les adaptateurs de protocole utilisent les services du MBean Server pour invoquer les opérations de l’application de management aux MBeans et pour transmettre les notifications des MBeans aux applications.

Il est à noter que pour qu’un agent soit gérable, il doit inclure au moins un adaptateur de protocole ou un serveur de connecteurs. Enfin, ces connecteurs et adaptateurs doivent être implémentés comme des MBeans afin qu’ils puissent également être gérable et plus particulièrement afin qu’ils puissent être chargés ou déchargés dynamiquement lorsque cela est nécessaire.
---
title: "JMX pour les nuls... - Les agents JMX - Partie 3"
date: 2010-11-08 15:50:22 +0100
comments: true
tags: 
- java
- jmx
---

![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)
Cette troisième partie sur JMX reste dans la continuité de la série d'article sur ce sujet (cf. [introduction](/2010/10/jmx-pour-les-nuls-introduction.html), [partie 1](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html) portant sur les généralités et [partie 2](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html) portant sur les différents MBeans et le concept de Notification) en introduisant plus précisément ce qu'est un agent JMX et à quoi il sert.

Les articles qui feront suite préciseront les concepts manipulés par un agent JMX beaucoup plus en profondeur.

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


<a name="agent"></a>
#Agent JMX

Un agent JMX est une entité de management qui s’exécute sur une machine virtuelle Java et qui permet de faire la liaison entre les MBean et l’application de management. Un agent JMX est composé d’un MBean server, d’un ensemble de MBean représentant les ressources à administrer et superviser, d’un ensemble minimal de services d’agent qui se présentent sous forme de MBean et d’au moins un adaptateur de protocol ou connecteur.

L’architecture d’un agent JMX peut donc se définir comme tel :

* Les MBeans qui représentent les ressources à administrer et superviser
* Le MBean server qui est la clé de voute de l’architecture d’un agent JMX
* Les services d’agent qui peuvent être des composants pré-définis ou des services spécifiques et  qui peuvent être classés de la manière suivante :
	* __Services d’agent dynamiques__ qui permettent à l’agent d’instancier des MBeans qui utilisent indifféremment des classes Java ou des librairies natives qui peuvent être téléchargées dynamiquement sur le réseau
	* __Services d’agent de management__ utilisés pour superviser la valeur des attributs des MBean : le service notifie sous certaines conditions ses écouteurs lors du changement de valeur d’un attribut du MBean
	* __Services timer d’agent__ qui permet d’émettre périodiquement des notifications
	* __Services de relation d’agent__ qui définit les associations entre les MBeans et qui est garant de la consistence des relations

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
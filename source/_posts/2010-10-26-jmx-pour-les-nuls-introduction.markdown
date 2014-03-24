---
layout: post
title: "JMX pour les nuls... - Introduction"
date: 2010-10-26 11:52:53 +0100
comments: true
categories: 
- java
- jmx
---
![left-small](http://1.bp.blogspot.com/_XLL8sJPQ97g/S-HXgq4HI7I/AAAAAAAAAJ4/LzJKs3wTNZM/s200/jmx4.png)
Certains l'auront peut être remarqué mais cela fait un moment que je n'ai rien posté sur mon blog. En fait, j'ai été un peu occupé à faire quelque chose qui me tenait à coeur depuis un moment... : la lecture complète des spécifications JMX (ouais, ça va, on s'amuse comme on veut/peut... ;-) ). Du coup, je me suis dit que cela pouvait également intéresser d'autres personnes qui aurait pu être découragées par la [version agrégée](http://download.oracle.com/javase/6/docs/technotes/guides/jmx/JMX_1_4_specification.pdf) de la spécification (ie. contenant la [JSR 3](http://www.jcp.org/en/jsr/detail?id=3) - __Java Management Extensions (JMX) Specification__ - en version 1.4 et la [JSR 160](http://jcp.org/en/jsr/detail?id=160) - __Java Management Extensions (JMX) Remote API__ - en version 1.0) qui fait très exactement 290 pages et qui se veut être, à ce jour, la dernière en date.

<!-- more -->

J'ai donc initié un document qui plagie plus ou moins les spécifications JMX mais qui, je l'espère, permettra à ceux que cela intéressent de comprendre les entrailles de cette technologie indispensable dans toute architecture qui se veut un minimum industrialisé... ;-) (cf. [mon post précédent](/blog/2010/05/05/jmx-ou-comment-administrer-et-superviser-son-systeme-dot-dot-dot/)) et ainsi de pouvoir briller en société en expliquant la différence entre un MBean Standard et un MBean Dynamique (ce que tout le monde connait ;-) ) mais aussi en précisant qu'il existe deux MBean dynamiques particuliés que sont le Model MBean et le Open MBean... la classe! ;-) 

~~Par contre, je ne sais pas encore comment découper mon document en article donc j'improviserai... sachant que toute remarque est la bienvenue ;-).~~

Cette série suivra le plan suivant :

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

En outre, il est à noter que ce document (et donc les articles qui en découleront) ne se veut pas exhaustif et qu'il ne s'agit que d'une interprétation de ma part des spécifications JMX. De plus, le ton appliqué se voudra plus "professionel" que dans mes autres articles car il s'agit plus d'un document de travail.

Ainsi, ce petit article me permet d'introduire la série d'article qui va suivre.

Bonne lecture ;-)
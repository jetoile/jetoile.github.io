---
title: "JMX pour les nuls... - Introduction"
date: 2010-10-26 11:52:53 +0100
comments: true
tags: 
- java
- jmx
---
![left-small](http://1.bp.blogspot.com/_XLL8sJPQ97g/S-HXgq4HI7I/AAAAAAAAAJ4/LzJKs3wTNZM/s200/jmx4.png)
Certains l'auront peut être remarqué mais cela fait un moment que je n'ai rien posté sur mon blog. En fait, j'ai été un peu occupé à faire quelque chose qui me tenait à coeur depuis un moment... : la lecture complète des spécifications JMX (ouais, ça va, on s'amuse comme on veut/peut... ;-) ). Du coup, je me suis dit que cela pouvait également intéresser d'autres personnes qui aurait pu être découragées par la [version agrégée](http://download.oracle.com/javase/6/docs/technotes/guides/jmx/JMX_1_4_specification.pdf) de la spécification (ie. contenant la [JSR 3](http://www.jcp.org/en/jsr/detail?id=3) - __Java Management Extensions (JMX) Specification__ - en version 1.4 et la [JSR 160](http://jcp.org/en/jsr/detail?id=160) - __Java Management Extensions (JMX) Remote API__ - en version 1.0) qui fait très exactement 290 pages et qui se veut être, à ce jour, la dernière en date.

<!-- more -->

J'ai donc initié un document qui plagie plus ou moins les spécifications JMX mais qui, je l'espère, permettra à ceux que cela intéressent de comprendre les entrailles de cette technologie indispensable dans toute architecture qui se veut un minimum industrialisé... ;-) (cf. [mon post précédent](/2010/05/jmx-ou-comment-administrer-et.html)) et ainsi de pouvoir briller en société en expliquant la différence entre un MBean Standard et un MBean Dynamique (ce que tout le monde connait ;-) ) mais aussi en précisant qu'il existe deux MBean dynamiques particuliés que sont le Model MBean et le Open MBean... la classe! ;-) 

~~Par contre, je ne sais pas encore comment découper mon document en article donc j'improviserai... sachant que toute remarque est la bienvenue ;-).~~

Cette série suivra le plan suivant :

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


En outre, il est à noter que ce document (et donc les articles qui en découleront) ne se veut pas exhaustif et qu'il ne s'agit que d'une interprétation de ma part des spécifications JMX. De plus, le ton appliqué se voudra plus "professionel" que dans mes autres articles car il s'agit plus d'un document de travail.

Ainsi, ce petit article me permet d'introduire la série d'article qui va suivre.

Bonne lecture ;-)
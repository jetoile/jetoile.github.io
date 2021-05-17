---
title: "JMX pour les nuls... - Les classes de base - Partie 4"
date: 2010-11-21 16:03:56 +0100
comments: true
tags: 
- java
- jmx
---

![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)
Cette quatrième partie sur JMX (cf. [introduction](/2010/10/jmx-pour-les-nuls-introduction.html, [partie 1](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html) portant sur les généralités, [partie 2](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html) portant sur les différents MBeans et le concept de Notification et [partie 3](/2010/11/jmx-pour-les-nuls-les-agents-jmx-partie.html) sur les agents JMX) permettra de présenter les classes de bases de JMX c'est-à-dire les classes qui sont manipulées en interne par les API de JMX à savoir :

* `ObjectName`
* `ObjectInstance`
* `Attribute`
* `AttributeList`

<!-- more -->

# Table des matières

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


<a name="objectName"></a>
# ObjectName
Un nom d'objet permet d'identifier un MBean dans le MBean Server de manière unique. L'application de supervision et d'administration utilise ce nom unique pour identifier le MBean cible.

La classe `ObjectName` représente ce nom unique dans le MBean Server et est constitué :

* d'un nom de domaine,
* d'un ensemble non ordonné d'une ou plusieurs clés-propriétés.

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TOk6Iw1WyvI/AAAAAAAAAOA/xCsPcxlmq_U/s1600/jmx39.png)

Le nom de domaine est une chaîne de caractères sensible à la casse qui fournit la structure d'un espace de nommage dans l'agent JMX. Il peut être optionnel car le MBean Server est capable d'en fournir un par défaut.

Il peut contenir n'importe quel caractère mis à part les caractères ":", "*" et "?" et prend généralement la forme d'un nom de domaine DNS inversé (ex : `com.jetoile.myDomain`). Il est déconseillé qu'il contienne la chaine "//".

La liste de clés-propriétés permet de fournir un nom unique au MBean dans un domain donné. Une clé-propriété est, en fait, un pair de clé/valeur où la valeur est une chaîne de caractères qui ne doit pas contenir les caractères suivants : ":", """, " ", "=", "*", "?". Une propriété doit obligatoirement être présente.

Un objectName aura donc la forme suivante :
```text
[domainName]:property=value[,property=value]*
```

Exemple :

```text
MyDomain:description=Printer,type=laser
MyDomain:description=Disk,capacity=2
DefaultDomain:description=Disk,capacity=1
DefaultDomain:description=Printer,type=ink
DefaultDomain:description=Printer,type=laser,date=1993
Socrates:description=Printer,type=laser,date=1993
```

En outre, JMX propose un moyen pour interroger l'annuaire de MBeans présent dans le MBean Server.

Ainsi, pour l'exemple donné ci-dessus, il est possible d'avoir :

* "*:*" renverra tous les objets du MBean Server. Un objet null ou une chaîne de caractère est équivalente à *.*
* ":*" renverra tous les objets du MBean Server présents dans le domaine par défaut
* "MyDomain:*" ne renverra rien
* "??Dom*:*" renverra tous les objets présent dans MyDomain
* "*:description=Printer,type=laser,*" renverra les objets suivants :
	* MyDomain:description=Printer,type=laser
	* DefaultDomain:description=Printer,type=laser,date=1993
	* Socrates:description=Printer,type=laser,date=1993
* "*Domain:description=Printer,*" renverra les objets suivants :
	* MyDomain:description=Printer,type=laser
	* DefaultDomain:description=Printer,type=ink
	* DefaultDomain:description=Printer,type=laser,date=1993
* "*Domain:description=P*,*" renverra les mêmes résultats que la requête précédente.

<a name="objectInstance"></a>
# ObjectInstance

La classe `ObjectInstance` permet de lier le nom d'un objet MBean à sa classe Java. C'est la seule description possible d'un MBean dans le MBean Server puisque l'accès à un MBean par sa référence n'est pas autorisé.

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/TOk7jhfsxWI/AAAAAAAAAOE/izV9iyFNQmc/s1600/jmx40.png)
<a name="attribute"></a>
## Attribute et AttributeList

Les classes `Attribute` et `AttributeList` représente les attributs et leurs valeurs d'un MBean. Elles contiennent le nom des attributs sous forme de chaîne de caractères et leurs valeurs sous forme d'objet de type `ObjectInstance`.
![center](http://4.bp.blogspot.com/_XLL8sJPQ97g/TOk722T85fI/AAAAAAAAAOI/HTXxjiZt06o/s1600/jmx41.png)

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/TOk8Go3ihYI/AAAAAAAAAOM/kE0Z7RzO6NQ/s1600/jmx42.png)

<a name="exception"></a>
## Les exceptions
JMX propose un ensemble d'exceptions qui peuvent principalement être levées par le MBean Server ou les services de l'agent JMX qui effectuent les opérations sur le MBean lorsque le code de ce dernier lève une exception.

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TOk89656tOI/AAAAAAAAAOU/mGlxu_K6NJo/s1600/jmx73.png)
![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TOk8v-NM4II/AAAAAAAAAOQ/m7Edr965FKk/s1600/jmx72.png)


# Le mot de la fin de cette partie

Nous avons vu dans cette partie les objets de base manipulés par JMX. Dans les parties suivantes, nous verrons plus précisément la notion de MBean Server.

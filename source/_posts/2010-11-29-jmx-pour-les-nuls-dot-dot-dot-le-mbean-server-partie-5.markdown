---
layout: post
title: "JMX pour les nuls... - Le MBean Server - Partie 5"
date: 2010-11-29 16:27:13 +0100
comments: true
categories: 
- java
- jmx
---

![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)
Cet article reste dans la lignée de ma petite série d'articles sur JMX (cf. introduction, partie 1 portant sur les généralités, partie 2 portant sur les différents MBeans et le concept de Notification, partie 3 sur les agents JMX et partie 4 sur les classes de base) et portera sur le MBean Server.

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


<a name="mbean_server"></a>
#MBean Server

Le MBean Server est un annuaire de MBeans se trouvant dans l'agent JMX. Il permet de fournir les services que manipulent les MBeans et toutes les opérations d'administration et de supervision, qui sont fournies par le MBean, le sont au travers de l'interface MBeanServer :

* Les MBeans qui représentent une ressource managée qui peuvent être n'importe quelle application, des ressources système ou réseaux tant qu'elles sont représentés par une interface Java ou un wrappeur Java.
* Les MBeans qui ajoutent des fonctionnalités de management qui peuvent être génériques (par exemple, des fonctionnalités de journalisation ou de monitoring) ou plus spécifique à une technologie ou à un domaine d'application.
* Les composants de l'infrastructure comme les serveurs de connecteurs et les adaptateurs de protocole.

Pour ce faire, l'agent JMX propose une classe factory pour trouver ou créer un MBean Server qui peut ne pas êre unique dans un agent JMX.

L'interface MBeanServer définie les opérations accessible sur un agent JMX.

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TOlMH2RYqlI/AAAAAAAAAOY/rcYE9XfKwrQ/s1600/jmx43.png)

La classe MBeanServerFactory dispose de méthodes statiques permettant de retourner une instance de l'implémentation de l'interface MBeanServer.

Lorsque l'implémentation du MBean Server est créée, il est possible d'indiquer le nom du domaine par défaut utilisé dans l'agent JMX qu'il représente. Ce sont ces méthodes qui sont également utilisées par l'agent JMX pour créer un ou plusieurs MBean Server.

Dans le cas où ce sont des méthodes de recherche et de récupération du MBean Server déjà créé dans l'agent JMX, les classes chargées dans la JVM peuvent accéder aux MBean Server déjà existant sans avoir à connaitre l'agent JMX.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Depuis Java 5, toute application Java possède un MBean Server qui peut être obtenu via la méthode java.lang.management.ManagementFactory.getPlatformMBeanServer()
</ul></td></tr>
</tbody></table>

![center](http://3.bp.blogspot.com/_XLL8sJPQ97g/TOlM8DBItVI/AAAAAAAAAOc/GK-kgJY2Hos/s1600/jmx46.png)

L'accès aux méthodes statiques de la classe MBeanServerFactory est controlé par la classe MBeanServerPermission. Cette classe étend les classes de permissions basiques de Java et permet l'accès aux opérations suivantes :

* createMBeanServer
* findMBeanServer
* newMBeanServer
* releaseMBeanServer

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TOlQlzvrRwI/AAAAAAAAAPM/6MVsoLxzkxM/s1600/jmx47.png)


L'enregistrement des MBeans au sein du MBean Server peut être fait soit via l'agent JMX, soit via un autre MBean. Pour ce faire, l'interface MBeanServer propose deux types d'enregistrement :

* l'instanciation d'un nouveau MBean et son enregistrement en une seule opération. Dans ce cas, le chargement de la classe Java du MBean peut être fait en utilisant le classe loader par défaut ou en en explicitant un,
* l'enregistrement d'un MBean déjà existant.

Dans ces deux cas, un ObjectName doit être associé au MBean lors de son enregistrement afin de l'identifier de manière unique au sein du contexte du MBean Server.

Le développeur du MBean peut, de plus, avoir la main sur l'enregistrement et le désenregistrement du MBean au sein du MBean Server en implémentant l'interface MBeanRegistration. Pour ce faire, le MBean Server vérifie dynamiquement avant (resp. après) l'exécution de ces opérations si le MBean implémente l'interface MBeanRegistration. Si c'est le cas, alors les opérations de callbacks adéquates sont invoquées.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
L'implémentation de l'interface MBeanRegistration est la seule manière de renseigner le MBean sur son environnement et plus particulièrement de lui fournir des informations sur le MBean Server au sein duquel il est enregistré. Cela lui permet, entre autre, d'obtenir des informations sur les autres MBean présent dans le MBean Server.
</ul></td></tr>
</tbody></table>

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TOlRJXtN5FI/AAAAAAAAAPQ/kWtb8jvKMsU/s1600/jmx48.png)

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TOlRaaRexrI/AAAAAAAAAPU/ho2Ok7wJLyE/s1600/jmx49.png)

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TOlRdOCQS3I/AAAAAAAAAPY/qYOspXtPJx0/s1600/jmx50.png)

Ainsi, l'interface MBeanServer offre des opérations pour :

* récupérer un MBean à partir de son ObjectName,
* récupérer une collection de MBeans à partir d'une expression régulière sur son nom en appliquant éventuellement un filtre sur ses attributs,
* récupérer la valeur d'un ou plusieurs attributs d'un MBean,
* invoquer une opération sur un MBean,
* d'introspecter un MBean pour récupérer l'interface de management d'un MBean, c'est-à-dire ses attributs et ses opérations,
* s'enregistrer afin de recevoir les notifications émises par un MBean.

Il est à noter que ces méthodes sont génériques, c'est pourquoi elles se basent toutes sur l'objectName du MBean. Le rôle du MBean Server est donc, entre autre :

* de résoudre l'objectName,
* de déterminer si l'opération invoquer sur le MBean peut être consistante (ie. résolue par le MBean),
* de renvoyer le résultat de l'opération au demandeur si les deux points précédents ont été vérifiés.

Cependant, il est également possible, plutôt que d'utiliser l'interface MBeanServer, d'utiliser un proxy. Un proxy est un objet Java implémentant la même interface que le MBean et dont une méthode de son interface est routé au travers du MBean Server vers le MBean. Pour ce faire, la classe statique JMX offre les méthodes utilitaires.

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/TOlSItzCdaI/AAAAAAAAAPc/MYHC7W_7ORs/s1600/jmx51.png)

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le MBean Server défini un domaine particulier appelé JMXImplementation dans lequel un MBean de la classe MBeanServerDelegate est enregistré. Cet objet identifie et décrit le MBean Server dans lequel il est enregistré. Il est également utilisé pour émettre les notifications émises par le MBean Server : ce MBean agit comme un delegate du MBean Server qu'il représente.<br/>
Son ObjectName est JMXImplementation:type=MBeanServerDelegate et il possède que des attributs en lecture-seule.
<br/>
<img src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TOlStTeaChI/AAAAAAAAAPg/DEI2Md4dIO0/s1600/jmx52.png" alt="medium"/>
</ul></td></tr>
</tbody></table>

Les méthodes queryMBeans(ObjectName, QueryExp) et queryNames(ObjectName, QueryExp) de récupération d'un MBean permettent de fournir une requête afin de minimiser la recherche au sein du MBean Server. Pour ce faire, ces méthodes prennent en paramètre :

* un ObjectName sous forme de pattern qui correspond à la portée de la requête et qui permet de spécifier le nom de l'ObjectName des MBeans à rechercher,
* un critère de filtre sur la valeur des attributs qui s'applique aux MBeans trouvés par le premier paramètre.

Dans le cas où la requête ne trouve pas de MBeans ou qu'aucun MBean trouvé ne répond aux critères fournis, le résultat ne contiendra pas d'éléments.

Dans le cas où le pattern de l'ObjectName est null, alors la portée de la recherche se fera dans tout le MBean Server.

Dans le cas où le critère de filtre est null, alors le résultat n'est pas filtré et correspond à tous les éléments se trouvant dans la portée de la requête.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le MBeanServerDelegate est accessible lors de la recherche de MBeans.
</ul></td></tr>
</tbody></table>

La portée de la requête doit avoir la forme telle que celle vu dans la quatrième partie de cette série de post sur JMX.

Le critère de filtre doit utiliser les classes et interfaces suivantes :

* L'interface QueryExp qui permet d'identifier les objets qui forme les expressions de la requête. Ces objets peuvent être utilisés dans une requête ou peuvent composés une requête plus complexe.
* Les interfaces ValueExp et StringValueExp qui permettent d'identifier les objets qui représentent les valeurs numériques ou de type chaîne de caractères.
* L'interface AttributeValueExp qui permet d'identifier les objets qui représentent les attributs.
* La classe Query qui permet la construction d'une requête. Elle propose des méthodes statiques qui retournent des objets de type QueryExp et ValueExp.
* La classe ObjectName qui implémente l'interface QueryExp et qui peut donc être utilisé dans un critère de filtre.

Dans la pratique, il n'est pas nécessaire d'avoir à utiliser directement les interfaces ValueExp et QueryExp. En effet, la classe Query fonctionne comme une fabrique d'objets de type QueryExp.

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TOlUQAX1lSI/AAAAAAAAAPk/ILENCvAgkPQ/s1600/jmx53.png)

Ainsi, on pourra, par exemple, avoir le code suivant :
```java
QueryExp exp = Query.and(
   Query.geq(Query.attr("age"),
   Query.value(20)),
   Query.match(Query.attr("name"),
   Query.value("G*ling")));
 
QueryExp exp = Query.eq(
   Query.classattr(),
   Query.value("managed.device.Printer"));
```

L'interface MBeanServerConnection étendue par l'interface MBeanServer permet de fournir une interface commune d'accès au MBean Server qu'il soit distant ou local.

Enfin, il est possible de fournir sa propre implémentation du MBean Server en renseignant la propriété système javax.management.builder.initial : lorsque les méthodes createMBeanServer() ou newMBeanServer() sur les MBeanServerFactory sont appelées, alors, si la variable système est renseignée, c'est le MBean Server qui est renseigné qui est utilisé, sinon, c'est le MBean Server par défaut.

Ce MBean Server doit être une classe publique qui étend la classe MBeanServerBuilder. Il est à noter que cette classe doit également pouvoir créer une instance de MBeanServerDelegate (qui peut être standard ou pas).

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/TOlUoxk6UlI/AAAAAAAAAPo/oPr-jYiJNIw/s1600/jmx54.png)

---
layout: post
title: "JMX pour les nuls... - Chargement dynamique de MBeans - Partie 6"
date: 2010-12-06 16:40:48 +0100
comments: true
categories: 
- java
- jmx
---

![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)

Cet article toujours dans la lignée de ma petite série d'articles sur JMX (cf. [introduction](/2010/10/jmx-pour-les-nuls-introduction.html), [partie 1](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html) portant sur les généralités, [partie 2](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html) portant sur les différents MBeans et le concept de Notification, [partie 3](/2010/11/jmx-pour-les-nuls-les-agents-jmx-partie.html) sur les agents JMX, [partie 4](/2010/11/jmx-pour-les-nuls-les-classes-de-base.html) sur les classes de base et [partie 5](/2010/11/jmx-pour-les-nuls-le-mbean-server.html) sur le MBeanServer) sera plus accès sur les capacités de chargement dynamique des MBeans. Cependant, il n'abordera que succinctement cette partie car il est nécessaire  d'avoir une très bonne connaissance du fonctionnement du classloader et qu'un simple article ne saurait l'aborder comme il se doit.

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


<a name="mbean_dynamic"></a>
#Chargement dynamique des MBeans

JMX propose un service de chargement dynamique de MBeans qui s'appuie sur les fonctionnalités du classloader Java pour récupérer et instancier les MBeans qui peuvent utilisés indifféremment des classes Java ou des librairies natives. Cette fonctionnalité existe car lorsque le serveur JMX est démarré, il peut ne pas avoir connaissance de tous les MBeans et devoir en télécharger. Pour ce faire, un service particulier chargé de gérer des applets de management (m-let) est utilisé pour instancier les MBeans obtenus d'une URL distante sur le réseau. En fait, le service de m-let permet d'instancier et d'enregistrer des MBeans dans le MBean Server en chargeant un fichier texte, dont la localisation est spécifiée par une URL, qui contient les informations des MBeans à obtenir. Quand ce fichier texte est chargé, toutes les classes spécifiés par le tag MLET sont téléchargées et une instance de chaque MBean spécifié est crée et enregistré. Le service de m-let étant lui-même un MBean enregistré dans le MBean Server, il peut être utilisé par d'autres MBeans, par d'autres applications ou par d'autres applications d'administration et de supervision distantes.

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TPKDftVvX6I/AAAAAAAAAPw/3wQmopnzDw0/s1600/jmx74.png)

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Depuis Java 5, toute application Java possède un MBean Server qui peut être obtenu via la méthode java.lang.management.ManagementFactory.getPlatformMBeanServer()
</ul></td></tr>
</tbody></table>

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TPKEiiLkb-I/AAAAAAAAAP0/xpRHd_cDe80/s1600/jmx56.png)

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
La classe MLet étend la classe URLClassLoader, ce qui implique qu'il s'agit également d'un classe loader.
</ul></td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Dans le cas générique, l'appel à la méthode <i>getMBeansFromURL()</i> du service de m-let permet de :</div><br />
<ul><li style="text-align: justify;">télécharger le fichier texte contenant le descriptif des MBeans (un MBean par tag MLET),</li>
<li style="text-align: justify;">télécharger les classes nécessaires,</li>
<li style="text-align: justify;">charger les classes dans le classe loader du MBean Server,</li>
<li style="text-align: justify;">instancier les MBeans,</li>
<li style="text-align: justify;">enregistrer les MBeans instanciés préalablement,</li>
<li style="text-align: justify;">renvoyer les instances des MBeans.</li>
</ul></td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Dans le cas où le MBean contient de méthodes natives et que des librairies natives sont chargées en utilisant la méthode <i>System.loadLibrary</i>, alors la méthode <i>findLibrary</i> du classe loader du m-let est appelé pour trouver la librairie. Cette méthode surcharge celle de la classe <i>ClassLoader</i> et, si la librairie est trouvée, copie cette dernière à l'emplacement retourné par la méthode <i>getLibraryDirectory</i> de la classe <i>MLet</i>.
</ul></td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le MBean Server maintient une liste de classe loader dans le repository de classe loader (aussi appelé <i>default loader Repository</i>). <br/>Ce classe loader <i>Repository</i> est utilisé dans les méthodes createMBean et instantiate de l'interface MBean Server : le repository loader repository est utilisé pour trouver et charger les classes.<br/>Si un m-let ne trouve pas les classe à l'URL, il essaie de chargeer les classes du classe loader repository.
</ul></td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Dans le cas où plusieurs MBean Server sont présent dans une même JVM, ils disposent chacun de leurs propres classe loader repository qui sont indépendants les uns des autres.
</ul></td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
L'ordre des classes loader dans le repository est important : quand une classe est chargée dans le repository, chaque classe loader est interrogé pour charger la classe. Si un loader réussi à charger la classe alors la recherche s'arrête, sinon (s'il lève une exception <i>ClassNotFoundException</i>) la recherche continue jusqu'à ce qu'il n'y en ait plus (dans ce cas l'exception <i>ClassNotFoundException</i> est levée).<br/>Le premier loader dans la classe loader repository est celui qui a chargé l'implémentation du MBean Server. Se trouve ensuite les loaders des MBeans, l'ordre des loaders étant celui du chargement des MBeans.<br/>Ainsi, on aura, par exemple : <br/>Un MBean m1 apparait avant le MBean m2 si la méthode <i>createMBean</i> ou  <i>registerMBean</i> qui a enregistrée m1 s'est terminé avant le démarrage de l'opération qui a enregistré m2. Si la méthode d'enregistrement de m2 a démarré avant la fin de la méthode d'enregistrement de m1, alors l'ordre des loaders est indéterminé.
</ul></td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Un m-let est un classe loader et, comme l'indique le comportement d'un classe loader standard, il charge une classe en utilisant la méthode <i>loadClass</i> :<br/>En premier lieu, il envoie une requête au classe loader parent pour charger la classe (le classe loader parent est celui spécifié lors le m-let est créé (par défaut le classe loader système)), puis, dans un second temps, si le classe loader parent n'a pas pu charger la classe, le m-let tente de la charger dans sa liste d'URL. Enfin, si ces deux essais se soldent par un échec, le m-let tente de charger la classe dans le classe loader repository (dans ce cas, il est dit qu'il délègue au classe loader repository).<br/>Si ces trois tentatives échouent, l'exception <i>ClassNotFoundException</i> est levée.<br />
<br/>
Pour avoir d'autres informations sur le chargement de classe ainsi que sur les problèmes qui peuvent apparaitre, voir les chapitres <i>8.4 The Class Loader Repository</i> et <i>8.5 Using the Correct Class Loader for Parameters</i> des spécifications JMX.
</ul></td></tr>
</tbody></table>
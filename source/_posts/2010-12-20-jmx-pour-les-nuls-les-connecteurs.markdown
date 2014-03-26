---
layout: post
title: "JMX pour les nuls... - Les connecteurs - Partie 8"
date: 2010-12-20 17:25:15 +0100
comments: true
categories: 
- java
- jmx
---

![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)

Ce huitième et dernier article sur JMX clôture cette petite série de posts sur JMX (cf. [introduction](/2010/10/jmx-pour-les-nuls-introduction.html), [partie 1](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html) portant sur les généralités, [partie 2](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html) portant sur les différents MBeans et le concept de Notification, [partie 3](/2010/11/jmx-pour-les-nuls-les-agents-jmx-partie.html) sur les agents JMX, [partie 4](/2010/11/jmx-pour-les-nuls-les-classes-de-base.html) sur les classes de base, [partie 5](/2010/11/jmx-pour-les-nuls-le-mbean-server.html) sur le MBeanServer, [partie 6](/2010/12/jmx-pour-les-nuls-chargement-dynamique.html) sur le chargement dynamique des MBeans et [partie 7](/2010/12/jmx-pour-les-nuls-les-services-jmx.html) sur les services JMX). Il abordera succinctement la notion de connecteur.

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


<a name="connector"></a>
#Les connecteurs

La spécification JMX défnit la notion de connecteurs : un connecteur est attaché à l'API JMX d'un MBean Server et le rend accessible de clients Java distants, le connecteur client possèdant une interface similaire à celle du MBean Server.

Un connecteur est donc constitué d'une partie cliente et d'une partie serveur :

* Le connecteur serveur est attaché au MBean Server et écoute les requêtes de connexion des clients
* Le connecteur client a pour rôle de trouver le server et d'étable une connexion avec ce dernier. Un connecteur client se trouve généralement dans une JVM différente du connecteur serveur et s'exécutera même généralement sur une machine différente.

Il est à noter que contrairement à un connecteur serveur, le connecteur client ne pourra se connecter qu'à un et un seul connecteur serveur (mais une application cliente pourra posséder plusieurs connecteurs clients).

JMX permet d'avoir de multiples connecteurs qui peuvent s'appuyer sur différents protocoles de communication entre le client et le serveur. Cependant, il définit le protocole standard et obligatoire sur tous serveurs JMX s'appuyant sur RMI (_Remote Method Invocation_) et le protocole optionel s'appuyant sur des sockets TCP et appelé JMXMP (_JMX Messaging Protocol_). 

Les connecteurs distinguent deux notions distinctes qui sont la __session__ et la __connexion__. Un connecteur client voit une session qui peut, tout au long de son cycle de vie de la session, avoir plusieurs connexions successives à un connecteur serveur. Il est à noter que, dans certains cas, il est possible d'avoir une connexion par requête (comme cela peut être le cas avec le protocole UDP ou JMS).

Une session possède un état sur le client mais pas nécessairement sur le connecteur serveur. Un connecteur, ne possède, quant à lui, pas forcément d'état.

Pour se connecter au connecteur serveur, le connecteur client peut utiliser une adresse du type `service:jmx:jmxmp://host1:9876`. Si la requête de connexion aboutie, le point d'accès client du connecteur client est retourné en réponse. 

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TQ0JWJXGzAI/AAAAAAAAAQo/b1hs_49t0Ys/s1600/jmx75.png)

Du point de vue de la connexion cliente, le code de l'utilisateur peut obtenir un objet qui implémente l'interface `MBeanServerConnection`. Cette interface étant similaire à l'interface `MBeanServer`, il devient transparent pour le code de l'application cliente d'interagir avec un MBean Server ne se trouvant pas dans la même JVM. 

En fait, l'interface `MBeanServerConnection` est étendue par l'interface `MBeanServer` et contient donc toutes les méthodes principales sauf celles qui n'ont de sens que pour une utilisation locale.

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TQ0Js7mndfI/AAAAAAAAAQs/puPa9Bkgz1Y/s1600/jmx68.png)

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Lors de l'enregistrement d'un écouteur auprès du MBeanServerConnection, le connecteur s'assure de la transmission de la notification de la partie serveur du connecteur à la partie cliente, puis à l'écouteur.<br/>
Les détails de la mise en œuvre de la transmission de la notification sont dépendants du protocole.
</ul></td></tr>
</tbody></table>

<br/>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Lors de l'utilisation d'un filtre de notifications, le filtrage des notifications reçues peut être effectué soit par la partie cliente du connecteur, soit par le MBean Server suite à la transmission du filtre.<br/>
Il est, bien sûr, à noter que le filtrage des notifications coté serveur évite la transmission des notifications sur le réseau.<br/>
Cependant, il est conseillé de rendre les filtres indépendants de leur lieu d'exécution (pour rappel, l'interface NotificationFilter étend l'interface Serializable).
</ul></td></tr>
</tbody></table>

Toutes les implémentations des connecteurs coté serveur doivent posséder un buffer de notifications qui correspond à la liste de toutes notifications émises par les MBeans à destination du MBean Server. Il est à noter que les connecteurs doivent supporter les requêtes concurrentes pour des raisons évidentes. Concernant l'adresse du serveur connecteur, il est conseillé, pour la générer, de passer par la classe `JMXServiceURL`.

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TQ0J6oZ__RI/AAAAAAAAAQw/HbReQRkRJtk/s1600/jmx69.png)

Afin de créer la partie cliente d'un connecteur, il existe deux possibilités :

* si l'adresse du serveur est connue, la méthode `connect(JMXServiceURL)` de la classe statique `JMXConnectorFactory` peut être utilisée,
* sinon, il est possible d'interroger le serveur pour obtenir un stub de `JMXConnector`.

Du point de vue de la connexion serveur, la connexion serveur est représentée par une sous-classe de `JMXConnectorServer` qui peut être obtenue :

* soit à l'aide de  la classe statique JMXConnectorServerFactory en lui indiquant le paramètre de type JMXServiceURL qui lui permet de connaitre les classes à instancier,
* soit en instanciant directement la sous-classe de JMXConnectorServer.

Le connecteur serveur doit être attaché à un MBean Server (ce qui ne signifie pas qu'il doit s'enregistré en son sein) et doit être actif pour être utilisable. Il peut être attaché auprès du MBean Server de deux manières :

* Le MBean Server auquel est attaché le connecteur serveur lui est spécifié lors de sa construction
* Le connecteur serveur est enregistré auprès du MBean Server comme s'il s'agissait d'un MBean

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TQ0KE210zHI/AAAAAAAAAQ0/uN68yvrtjr4/s1600/jmx71.png)

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TQ0KUwpcisI/AAAAAAAAAQ4/oWKoFdTIM4k/s1600/jmx70.png)

Exemple de création et d'attachement d'un connecteur serveur auprès d'un MBean Server

```java
MBeanServer mbs = MBeanServerFactory.createMBeanServer();
JMXServiceURL addr = new JMXServiceURL("jmxmp", null, 0);
JMXConnectorServer cs = JMXConnectorServerFactory.newJMXConnectorServer(addr, null, mbs);
cs.start();
```
Exemple de création et d'enregistrement d'un connecteur serveur dans un MBean Server

```java
MBeanServer mbs = MBeanServerFactory.createMBeanServer();
JMXServiceURL addr = new JMXServiceURL("jmxmp", null, 0);
JMXConnectorServer cs = JMXConnectorServerFactory.newJMXConnectorServer(addr, null, null);
ObjectName csName = new ObjectName(":type=cserver,name=mycserver");
mbs.registerMBean(cs, csName);
cs.start();
```

JMX définit également comment un agent peut publier son serveur de connexion dans une infrastructure de recherche et de découverte ; ce qui permet à l'API cliente distante de savoir où trouver un serveur et ainsi de pouvoir si connecter. Cela couvre :

* SLP (_Service Location Protocol_) en décrivant comme un agent enregistre le service URL d'un connecteur serveur avec SLP
* Jini en décrivant comment un agent enregistre un stub du connecteur serveur dans le service de recherche Jini
* JNDI (_Java Naming and Directory Interface_) en décrivant un agent enregistre le service URL d'un connecteur dans un annuaire LDAP

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Un connecteur client peut spécifier un class loader par défaut lorsqu'il effectue sa connexion au serveur. Ce class loader est alors utilisé lors de la déserialisation des objets reçus du serveur que ce soit les valeurs retournées, les exceptions ou les notifications.<br/>
Le class loader par défaut est positionné par la valeur de l'attribut jmx.remote.default.class.loader définie par l'environnement JMXConnector, sinon, il s'agit du class loader du contexte.
</ul></td></tr>
</tbody></table>

<br/>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le class loader utilisé par le connecteur serveur pour déserialiser les paramètres reçus par le connecteur client est dépendant de l'opération : en effet, parfois, il peut s'agir de celui du MBean cible (si les types de paramètres ne sont pas définis par l'API JMX), parfois, de celui configuré pendant la création du connecteur server (s'il est prévu de l'utiliser dans une application de gestion particulière qui peut définir ses propres sous classes de types de paramètres de MBean ou <em>NotificationFilter</em>).<br />
Le connecteur serveur possède un class loader par défaut qui est déterminé lors de son démarrage comme suit :<br />
<ul><li>Si l'attribut <em>jmx.remote.default.class.loader</em> de l'environnement existe, sa valeur détermine le class loader à utiliser</li>
<li>Si l'attribut <em>jmx.remote.default.class.loader.name</em> de l'environnement existe, sa valeur détermine l'ObjectName du MBean à utiliser comme class loader par défaut. Cette fonctionnalité permet à un connecteur serveur d'être créé avec un class loader qui est un m-let dans le même MBean Server</li>
<li>Sinon, c'est le class loader du thread contextuel qui est utilisé</li>
</ul>Pour plus d'informations sur l'utilisation de class loader étendu, il est recommandé de se référer au chapitre 13.11 Class Loading de la spécification JMX.
</ul></td></tr>
</tbody></table>

<br/>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Afin d'éviter les difficultés avec le fonctionnement des class loader Java, il est conseillé de n'utiliser que les types standards Java ou fournis par l'API JMX ou les Open MBeans qui permettent des structures plus complexes. De même, pour les notifications, il est préconisé de n'utiliser que les notifcations suivantes :<br />
<ul><li><em>NotificationFilterSupport</em></li>
<li><em>MBeanServerNotificationFilter</em></li>
<li><em>AttributeChangeNotificationFilter</em></li>
</ul>
</ul></td></tr>
</tbody></table>

Le connecteur serveur est, en outre, capable d'authentifier le client distant. Par exemple, pour le connecteur RMI, cela est fait fournissant une implémentation de l'interface `JMXAuthenticator` lors de la création du connecteur serveur. Dans le cas du connecteur JMXMP, cela est fait via SASL. Dans ces deux cas, le résultat de l'authentification renvoie un Subject JAAS qui représente l'identité authentifiée. Les requêtes alors reçues du client sont exécutées en utilisant cette identité. Avec JAAS, il est possible de définir les permissions liées à une identité. En particulié, il est possible de restreindre l'accès aux opérations du MBean Server en utilisant la classe `MBeanPermission`. Bien sûr, pour que cela fonctionne, il est nécessaire d'avoir un `SecurityManager`. Dans le cas où le connecteur serveur ne supporte pas l'authentification ou qu'il n'est pas configuré, les requêtes clientes sont exécutées en utilisant la même identité que celle qui a été créée avec le connecteur serveur. Comme alternative à JAAS, il est possible de controler l'accès aux opérations du MBean Server en utilisant un `MBeanServerForwarder`. Cet objet est une implémentation de l'interface `MBeanServer` et transmet les méthodes à un autre objet `MBeanServer` mais en ajoutant des opérations avant et après la transmission. Cela permet, en l'occurrence d'y faire des controles d'accès. Pour ajouter un `MBeanServerForwarder`, la méthode `setMBeanServerForwarder` peut être utilisée. De plus, pour chaque connexion à un connecteur serveur, au moins un Subject authentifié est néccessaire. Cela signifie que si un client effectue des opérations qui nécessitent plusieurs identités, des connexions différentes doivent être établies. Cependant, il est possible aux connecteurs (tels que les connecteurs RMI et JMXMP) d'utiliser la notion de délégation de Subject. Pour ce faire, avec chaque requête, le client spécifie un Subject. La requête est alors exécutée en utilisant une identité pour chaque requête si cela est possible. La permission a utilisé doit alors être de type `SubjectDelegationPermission`. Dans ce cas, pour chaque Subject délégué, le client obtient un `MBeanServerConnection` du JMXConnector pour un Subject authentifié. Les requêtes utilisées par le `MBeanServerConnection` sont émises avec un Subject délégué. Il est, de plus, à noter que ces objets de type `MBeanServerConnection` peuvent être obtenus du même JMXConnector et peuvent être utilisés simultanément. Ainsi, pour résumer, il est possible de configurer les permissions d'un connecteur serveur de deux manières :

* toutes les permissions pour les opérations de chaque client distant doivent être configurées,
* un `SubjectDelegationPermission` pour chaque `Principal` utilisé par les clients distants doit être configuré.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le connecteur RMI est le seul connecteur qui doit être présent dans les implémentations de JMX. Il utilise l'infrastructure RMI (Remote Method Invocation) pour la communication entre le client et le serveur.<br />
RMI définit deux couches de transport standard :<br />
<ul><li><em>Java Remote Method Protocol</em> (JRMP) qui est le protocole de transport par défaut</li>
<li><em>Internet Inter-ORB Protocol</em> (IIOP) qui est le protocole défini par CORBA et qui permet l'interopérabilité avec un autre langage de programmation</li>
</ul>Pour chaque serveur de connexion RMI, un objet distant implémentant l'interface distante RMIServer existe. Un client voulant communiquer avec le connecteur serveur doit obtenir une référence distante (ou stub) connecté à cet objet distant. Via RMI, chaque méthode appelée sur un stub sera transféré à l'objet distant. Ainsi, le client possédant un stub sur l'objet <em>RMIServer</em> pourra appeler obtenir le même résultat que s'il avait appelé l'objet du serveur.<br />
De plus, il existe un objet distant pour chaque connexion cliente qui permet de créer un nouvel objet distant qui implémente l'interface distante <em>RMIConnection</em>.<br />
Enfin, il est à noter que le code de l'utilisateur n'interagit généralement pas directement avec les objets du <em>RMIServer</em> ou du <em>RMIConnection</em> : en général, le stub <em>RMIServer</em> est obtenu au travers d'un objet <em>RMIConnector</em> qui implémente l'interface <em>JMXConnector</em>.<br />
L'obtention de cet objet <em>RMIConnector</em> peut être fait de trois manières :<br />
<ul><li>En fournissant un <em>JMXServiceURL</em> au <em>JMXConnectorFactory</em> qui spécifie le protocol RMI ou IIOP.</li>
<li>En obtenant le stub <em>JMXConnector</em> de " quelque part " (annuaire LDAP, Service de lookup JNDI, &#8230;).</li>
<li>En obtenant un stub <em>RMIServer</em> de " quelque part " et qui peut être utilisé comme paramètre du constructeur de <em>RMIConnector</em>.</li>
</ul>Les notions de sécurité et de versionnement sont également décrites dans le spécification JMX (et plus particulièrement la JSR160). Cependant, ce document n'abordera pas ces points.</td></tr>
</tbody></table>

<br/>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
La spécification JMX (et plus précisément la JSR160) décrit comment implémenter un connecteur générique en précisant comment configurer par plugin son protocole de transport ainsi que la manière dont il encapsule les objets transitant du client au serveur et plus particulièrement sa gestion du classloader du MBean cible.<br/>
Cependant, ce document n'abordera pas ces points. Aussi, pour plus d'informations, il est conseillé de se référer à la JSR160 ou aux chapitres 15 - Generic Connector et 16 - Defining a new transport des spécifications agrégées JMX.
</td></tr>
</tbody></table>

<br/>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le connecteur JMXMP est une configuration particulière du connecteur générique où le protocole de transport s’appuie sur TCP et l’encapsulation des objets est la sérialisation native Java.
</td></tr>
</tbody></table>

<br/>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
La spécification JMX ne spécifie pas comment trouver l'adresse d'un serveur de connecteurs attaché à un agent connu ou comment découvrir les agents présents dans le système ainsi que ce qu'ils proposent. Cependant, la JSR160 propose trois infrastructures permettant de répondre à ces besoins :<br />
<ul><li>SLP (<em>Service Location Protocol</em>)</li>
<li>JINI</li>
<li>JNDI (<em>Java Naming and Discovery Interface</em>) utilisé au travers un annuaire LDAP</li>
</ul><br />
Bien sûr, les APIs pour utiliser ces différentes technologies diffèrent cependant, les principes généraux restent identiques :<br />
<ul><li>L'agent créé un ou plusieurs serveurs de connecteurs</li>
<li>Pour chacun des connecteurs à exposer, il expose soit le <em>JMXServiceURL</em> (dans le cas de SLP ou JNDI), soit le stub <em>JMXConnector</em> (dans le cas de JINI ou JNDI) qu'il enregistre dans le service de recherche (en spécifiant au besoin d'autres méta-données)</li>
<li>Le client interroge le service de recherche pour obtenir un ou plusieurs <em>JMXServiceURL</em> ou stub <em>JMXConnector</em></li>
<li>Le client utilise soit le factory <em>JMXConnectorFactory</em> pour obtenir un <em>JMXConnector</em> connecté au serveur s'il possède un <em>JMXServiceURL</em>, soit se connecte directement au serveur en utilisant le stub <em>JMXConnector</em></li>
</ul>Ce document ne fournira pas plus d'informations sur ces points. Aussi, pour plus d'informations, il est conseillé de se référer à la JSR160 ou au chapitres 17 -<em>&nbsp;Bindings to lookup services</em> des spécifications agrégées JMX.
</td></tr>
</tbody></table>

#Le mot de la fin
Voilà, ce dernier article met fin à cette série. Si vous avez réussi à tenir jusque là et à tout lire, félicitation! Maintenant, vous devriez avoir (enfin, j'espère) une vision plus précise sur ce que contiennent les spécifications JMX. Bien sûr, cette série n'était qu'un condensée (qui vaut ce qu'il vaut) permettant de comprendre les concepts sans pour autant lire les 290 pages de spécification et ne se veut pas être suffisante pour tout maîtriser.
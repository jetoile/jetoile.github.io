---
title: "JMX pour les nuls... - Les services JMX - Partie 7"
date: 2010-12-13 17:01:18 +0100
comments: true
tags: 
- java
- jmx
---

![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)

Ce septième et avant dernier article sur JMX (cf. [introduction](/2010/10/jmx-pour-les-nuls-introduction.html), [partie 1](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html) portant sur les généralités, [partie 2](/2010/11/jmx-pour-les-nuls-les-differents-mbeans.html) portant sur les différents MBeans et le concept de Notification, [partie 3](/2010/11/jmx-pour-les-nuls-les-agents-jmx-partie.html) sur les agents JMX, [partie 4](/2010/11/jmx-pour-les-nuls-les-classes-de-base.html) sur les classes de base, [partie 5](/2010/11/jmx-pour-les-nuls-le-mbean-server.html) sur le MBeanServer et [partie 6](/2010/12/jmx-pour-les-nuls-chargement-dynamique.html) sur le chargement dynamique des MBeans) abordera les différents types de services JMX.

Le MBean Server propose, par défaut, un ensemble de fonctionnalités qui se présente sous forme de services JMX (ie. de MBeans), à savoir :

* __Service Monitoring__
* __Service Timer__
* __Service de relation__
* __Service de sécurité__

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


<a name="monitoring"></a>
# Service Monitoring

Il existe une famille de MBeans de monitoring qui se présentent qui permettent de scrupter les variations au cours du temps de la valeur des attributs des autres MBeans et qui permet d'émettre des notifications à intervalle régulier. Ils sont aussi classifiés comme des services de monitoring.

En fait, un service de monitoring (et donc un MBean de monitoring) permet d'observer la valeur d'un attribut donné présent dans un ou plusieurs MBeans (on dit qu'il s'agit alors du __MBean observé__) à un intervalle à régulier. Cette valeur peut être indifféremment la valeur d'un attribut ou la valeur contenue dans la valeur d'un attribut s'il est de type complexe. Pour chaque MBean observé, le moniteur interpole une seconde valeur qui est appelé le _pas dérivé_ (__derived gauge__) et qui correspond, soit à la valeur observée, soit à la différence entre deux valeurs numériques obtenues consécutivement.

Un type de notification spécifique est alors émis à chaque service de monitoring lorsque le pas dérivé répond à un ou un ensemble de conditions (ou lorsque certaines erreurs se produisent) qui peuvent être initialisées lors de l'initialisation du moniteur ou dynamiquement au travers de l'interface d'administration du MBean moniteur. 

Il existe trois types de moniteurs :

* __CounterMonitor__ qui permet de scrupter des valeurs de type entier (Byte, Integer, Short, Long) et qui se comporte comme un compteur :
	* les valeurs ne peuvent être inférieures strictement à 0,
	* les valeurs ne peuvent qu'être incrémentées,
	* les valeurs peuvent être tournantes (dans ce cas un modulo doit être défini).
* __GaugeMonitor__ qui permet de scrupter des valeurs de types entier ou flottantes (Float, Double) et qui se comporte comme un pas
* __StringMonitor__ qui permet de scrupter des valeurs de type String.

Tous ces moniteurs doivent étendre la classe abstraite `Monitor` alors que les types des valeurs surveillées doivent être supportés par le type de moniteur utilisé. Cependant, il est à noter que le moniteur vérifie le type de la valeur de l'instance retournée et non le te type de l'attribué déclaré dans les méta-données du MBean observé.

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TPKLxHRkZAI/AAAAAAAAAP4/eZfErdqRqWE/s1600/jmx58.png)

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TPKMExdR2sI/AAAAAAAAAP8/LvN1aDxsNIc/s1600/jmx59.png)

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TPKMQ5MDK8I/AAAAAAAAAQA/sszqSpNWdeM/s1600/jmx60.png)

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TPKNgMBobwI/AAAAAAAAAQI/NyLwm_9d4oc/s1600/jmx76.png)

Pour les notifications, une sous classe spécifique de `Notification` est définie pour être utilisée par tous les services de monitoring. Il s'agit de la classe `MonitorNotification`. 

Ainsi, elle est utilisée pour rapporter les cas suivants :

* Une condition de déclenchement d'un moniteur est détectée (par exemple, si le seuil d'une valeur est atteinte)
* Une erreur s'est produite durant la scrutation d'un attribut (par exemple, si un MBean est désenregistré)

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TPKOjDQNWDI/AAAAAAAAAQM/qL7w59mPIcM/s1600/jmx77.png)

La figure suivante représente tous les types de notifications qui peuvent être générées par un service Monitor.

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TPKO0P3FAEI/AAAAAAAAAQQ/FSwyMlB4ryI/s1600/jmx57.png)

<a name="timer"></a>
# Service Timer

Le service Timer permet de déclencher des notifications à des dates et heures spécifiques ou à intervalles réguliers. Les notifications sont émises à tous les objets déclarés comme étant interessés par les notifications émises par le timer (patron __Observer__). 

Le service Timer est un MBean qui peut être managé en permettant, entre autre, aux applications de modifier la configuration de son ordonnanceur (__scheduler__).

En fait, la classe Timer gère une liste de notifications datées qui sont émises quand leur date et heure arrivent. Les méthodes de cette classe permettent d'ajouter et de supprimer des notifications de sa liste. En fait, les types de notifications sont fournis par l'utilisateur avec la date et optionnellement la fréquence et le nombre de répétitions.

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TPKPT0nrpwI/AAAAAAAAAQU/_P4QW-Dln5I/s1600/jmx64.png)

Pour les notifications, une sous classe spécifique de `Notification` est définie pour être utilisée par le service __Timer__. Il s'agit de la classe `TimerNotification`. 

Ainsi, elle est utilisée pour rapporter les cas suivants :

* Les notifications ne sont déclenchées qu'une seule fois
* Les notifications sont répétées à une période définie et/ou un nombre d'occurrences

Ces paramètres sont définis lorsque la notification est ajoutée à la liste de notifications du timer. En outre, à chacune des notifications émises est associée un identifiant unique.

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TPKPwGfdiTI/AAAAAAAAAQY/YNQBbw48Vsw/s1600/jmx61.png)

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Si la notification ajoutée au service Timer possède une date antérieure à la date courante, alors la méthode addNotification se comporte comme si la date indiquée était la date courante.
</ul></td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Si la date d'émission d'une notification est mise à jour alors qu'elle est déjà existante, alors aucune notification de mise à jour n'est émise.
</ul></td></tr>
</tbody></table>

Le service Timer MBean est un émetteur standard de notifications (qui possèdent un type et une date définis dans la liste de notifications ajoutées à l'aide de la méthode `addNotification()`). Tous les écouteurs d'un MBean Timer donnés reçoivent toutes les notifications. Les écouteurs interessés par un type de notifications Timer particulier doivent spécifier un objet filtre lors de leur enregistrement.

Lorsqu'un timer est actif et que la date d'une notification de type Timer arrive à son terme, le service Timer émet cette notification en fournissant son type, message, données utilisateurs ainsi que l'identifiant de la notification. S'il s'agit d'une notification périodique disposant d'un nombre d'occurrence, il est décrémenté.

Enfin, si la notification n'a plus besoin d'être émise (notification non récurente ou notification périodique disposant d'un nombre d'émission prédéfini), elle est supprimée de la liste des notifications.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Si une notification dispose d'une date d'émission antérieure à son démarrage, il est possible de l'émettre ou de l'ignorer à l'aide de la méthode sendPastNotification(). Par défaut, la valeur de cet attribut est mise à false.
</ul></td></tr>
</tbody></table>

<a name="relation"></a>
# Service Relation

JMX définit un modèle de relation entre les MBeans. Une relation est définie par l'utilisateur et se présente comme une association n-aire entre les MBeans. 

Pour ce faire, JMX propose un ensemble de classes à utiliser pour décrire les relations, ainsi qu'un service qui permet de centraliser toutes les opérations de mise en relation. 

Toutes les relations permettent d'associer un type de relations (qui fournit des informations sur les rôles qu'il contient comme la multiplicité) avec le nom de classe du MBean qui satisfait le rôle. 

Le service de relations permet également de chercher un MBean au travers de ses relations.

Il est garant de la consistance des relations en vérifiant les opérations et le désenregistrement des MBean afin de s'assurer que les relations sont toujours conformes. Si une relation n'est plus valide, elle est supprimée du service de relations même si ses MBeans membres restent présents.

Une relation est composée de rôles nommés qui disposent d'une liste de MBeans associés à ce rôle. Cette liste doit être consistante avec les informations de multiplicité ainsi qu'avec les classes des MBean qui représentent ce rôle. 

Un ensemble de définition de rôles constitue un type de relation.

Un type de relation est un patron pour toutes les instances de relations qui associent les MBeans représentant ses rôles.

Par la suite, le terme de relation signifiera une instance spécique d'une relation associant des MBeans existants aux rôles dans un type de relations défini.

Le tableau ci-dessous décrit les concepts nécessaires à la compréhension du modèle de relations défini par JMX :

Concept | Description
-- | --
Role information | Décrit un des rôles dans une relation. L'information sur un rôle est composée du nom du rôle, de sa multiplicité, du nom de la classe qui participe à ce rôle, d'une propriété en lecture-seule sur les permissions et d'une description.
Relation type |	Méta-donnée d'une relation, composée d'un ensemble de Role information. Permet de décrire les rôles qu'une relation satisfait et permet de servir de patron pour créer et maintenir des relations.
Relation |	Association courante entre des MBeans qui satisfont une relation donnéee. Une relation peut avoir des propriétés et méthodes qui ne s'appliquent qu'à ses MBeans.
Role Value |	Liste de MBeans qui satisfont un rôle donné dans une relation.
Unresolved Role	Résultat d'un accès illégal d'une opération sur un rôle. Contient la raison du refus de l'opération.
Support Classes	| Classes internes utilisées pour représenter des types de relations et des instances de relations.
Relation Service |	MBean qui peut accéder et maintenir la consistance de tous les types de relations et toutes les instances de relations dans un agent JMX. Fournit des opérations pour trouver un MBean et ses rôles dans une relation.

En JMX, le type de relation est un ensemble statique de rôles. Les types de relations peuvent être définis à l'exécution mais, une fois défini, ces rôles et informations sur les rôles ne peuvent être modifiés. 

<u>Exemple de relations :</u>
Dans cet exemple, des livres (_Books_) et les propriétaires (_Owner_) représente des rôles : 

* Books représente le nombre de livres que possède la classe d'un MBean donnée. 
* Owner représente un propriétaire de livres d'une autre classe d'un MBean.

Un type de relations (_Personal Library_) peut être défini comme étant la notion d'appartenance de livres.

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TPKRennmH-I/AAAAAAAAAQg/l79ynOUIw8k/s1600/jmx65.png)

En fait, le service de relations empêche la création de types de relations invalides, de relations invalides ou la modifiation de valeurs de rôles invalides.

Quand une relation est supprimée du service de relations, ses MBeans membres ne sont pas impactés.
Quand un type de relation est supprimé, toutes les relations référencées sont au préalable supprimées. Cependant, c'est à l'appelant de gérer la consistance lors de la suppression d'un type de relations.

En outre, puisque les relations ne sont définies qu'au niveau des MBeans enregistrés, en supprimer un de l'annuaire est susceptible de modifier la relation. Le service de relations écoute, pour tous les MBean, le server de notifications qui indique quand un membre d'une relation est désenregistré. Le MBean correspondant est alors supprimé de tous rôles où il se trouve. Dans le cas où la nouvelle cardinalité du rôle n'est pas consistante avec son type de relation, alors la relation est supprimée du service de relation.

Le service de relations émet des notifications après l'exécution d'opérations qui modifient une instance de relations (création, mise à jour ou suppression). Cette notification fournit des informations sur la modification comme l'identifiant de la relation et la nouvelle valeur d'un rôle.

De plus, il est à noter qu'il existe deux types de relations et relations :

* Les types de relations et relations internes : les types de relations et leurs instances sont créés par le service de relations et ne sont accessibles qu'aux travers ses opéraitons. 
* Les types de relations et relations externes : les types de relations et leurs instances sont instanciés en dehors du service de relations puis lui sont ajoutés. Dans ce cas, ils peuvent être accessibles directement et, en l'occurrence, peuvent être enregistré comme des MBeans. Il est cependant à noter que ces types de relation sont immuables et qu'elles ne doivent pas être modifiées une fois enregistré dans le service de relation.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
En raison de la complexité de ce service, pour plus d'informations, il est préférable de se référer aux chapitres 11.2 Relation Service Classes, 11.3 Interfaces and Support Classes et 11.4 Role Description Classes des spécifications agrégées JMX.
</ul></td></tr>
</tbody></table>

<a name="securite"></a>
# Service securité
Un MBean Server JMX peut avoir accès à des informations sensibles et peut être susceptible d'exposer des opérations sensibles. Pour ce faire, JMX propose un mécanisme d'accès à de telles opérations en s'appuyant sur le modèle de sécurité de Java : il est possible de définir des permissions qui permettent de contrôler l'accès au MBean Server et à ses opérations.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
La sécurisation du MBean Server s'appuyant sur le modèle standard de sécurité Java, il est nécessaire de disposer du gestionnaire de sécurité Java (security manager). La méthode statique System.getSecurityManager() permet de le vérifier.
</ul></td></tr>
</tbody></table>

<br/>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Pour rappel, le modèle de sécurité de Java s'appuie sur le principe de permissions : pour qu'une opération soit invocable, l'appelant doit posséder la/les permissions.<br />
En effet, à un point donné de l'exécution d'un programme correspond un ensemble de permissions appliqué à l'exécution d'un thread. <br />
En fait, une opération est autorisée si le thread qui l'invoque dispose des permissions nécessaires.<br />
<br />
Les permissions sont des objets java qui étendent la classe <i>Permission</i>.<br />
<div class="separator" style="clear: both; text-align: center;"><a href="http://1.bp.blogspot.com/_XLL8sJPQ97g/TPKS5FhDvOI/AAAAAAAAAQk/QbB4KTZsUXA/s1600/jmx66.png" imageanchor="1" style="margin-left: 1em; margin-right: 1em;"><img border="0" height="263" src="http://1.bp.blogspot.com/_XLL8sJPQ97g/TPKS5FhDvOI/AAAAAAAAAQk/QbB4KTZsUXA/s320/jmx66.png" width="320" /></a></div>
</ul></td></tr>
</tbody></table>

En fait, JMX défini, par défaut, sa propre permission MBeanServerPermission qui peut être configurée comme suit :

* `MBeanServerPermission` qui prend en paramètre de son constructeur une chaine de caractère représentant la liste des opérations autorisées sur le MBeanServer (ex : `MBeanServerPermission("createMBeanServer")` qui permet de controler l'accès aux méthodes `MBeanServerFactory.createMBeanServer` )
* `MBeanPermission` qui prend en paramètre deux chaines de caractères :
	* la première représente la cible et peut être composée du nom canonique de la classe ou d'une expression régulière, du membre (c'est-à-dire le nom de l'attribut ou de l'opération qui doit être accédé), et de l'ObjetName du MBean cible. 

<u>Exemples de cible :</u>
```text
com.example.Resource#Name[com.example.main:type=resource]
com.example.Resource#*[com.example.main:type=resource]
*#Name[com.example.main:type=resource]
```
<ul><li><ul>
<li>la  deuxième représente l'action une ou plusieurs méthodes du MBean Server parmi les suivantes :</li>
<li><ul>
	<li>`addNotificationListener`</li>
	<li>`getAttribute`</li>
	<li>`getClassLoader`</li>
	<li>`getClassLoaderFor`</li>
	<li>`getClassLoaderRepository`</li>
	<li>`getMBeanInfo`</li>
	<li>`getObjectInstance`</li>
	<li>`instantiate`</li>
	<li>`invoke`</li>
	<li>`isInstanceOf`</li>
	<li>`queryMBeans`</li>
	<li>`queryNames`</li>
	<li>`registerMBean`</li>
	<li>`removeNotificationListener`</li>
	<li>`setAttribute`</li>
	<li>`unregisterMBean`</li>
</ul>
</li>
</ul></li></ul>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Les opérations suivantes ne sont pas soumises à la sécurisation du MBean Server :<br />
<ul><li><i>isRegistered</i></li>
<li><i>getMBeanCount</i></li>
<li><i>getDefaultDomain</i></li>
</ul>
</ul></td></tr>
</tbody></table>

De plus, JMX propose la permission `MBeanTrustPermission` qui permet de faire confiance à un certain utilisateur ou code source signé. Si le signataire ou le code source possède cette permission, le MBean est considéré comme étant de confiance. Seuls les MBeans de confiance peuvent alors être enregistrés dans le MBean Server.

Une permission `MBeanTrustPermission` est configurée soit avec la valeur "register", soit avec la valeur  "*" qui signifient toutes deux la même chose dans la version courante de la version JMX.

Ainsi, utilisé en complément de la permission `MBeanPermission`, la permission `MBeanTrustPermission` permet une gestion de la sécurité plus fine sur l'enregistrement des MBeans : la permission `MBeanPermission` vérifie qui enregistre alors que la permission `MBeanTrustPermission` vérifie ce qui est enregistré.

<u>Exemples de fichier de policy :</u>

```text
grant {
    permission javax.management.MBeanPermission "*", "*";
};
 
grant codeBase "file:${user.dir}${/}appl1.jar" {
    permission javax.management.MBeanPermission "", "getClassLoaderRepository";
};
 
grant codeBase "file:${user.dir}${/}appl2.jar" {
    permission javax.management.MBeanPermission "[d1:*]", "isInstanceOf, getObjectInstance";
};
 
grant codeBase "file:${user.dir}${/}appl3.jar" {
    permission javax.management.MBeanServerPermission "findMBeanServer";
    permission javax.management.MBeanPermission "JMImplementation:*", "queryNames";
};
 
grant {
    permission javax.management.MBeanTrustPermission "register";
};
 
grant signedBy "Gorey" {
    permission javax.management.MBeanTrustPermission "register";
};
```

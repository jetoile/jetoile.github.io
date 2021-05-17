---
title: "JMX pour les nuls... - Les différents MBeans et la notion de Notification - Partie 2"
date: 2010-11-03 14:09:21 +0100
comments: true
tags: 
- java
- jmx
---
![left](http://1.bp.blogspot.com/_XLL8sJPQ97g/TMdPuTpRY8I/AAAAAAAAALQ/R_w0PiLpvwo/s200/jmx-cover.png)
Mon [précédent article](/2010/10/jmx-pour-les-nuls-les-concepts-partie-1.html) portant sur la présentation générale de JMX a permis de poser les bases quant aux concepts fondamentaux. Cette seconde partie ainsi que les suivantes consistent en une descente plus en profondeur dans les entrailles des spécifications en reprenant les différentes notions vues précédemment. 

Ainsi, cette partie traitera des différents MBeans et du concept notification.

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


<a name="mbean"></a>
# MBean
Un MBean est une classe java concrète qui :

* doit implémenter sa propre interface MBean ou implémenter l’interface `DynamicMBean`,
* peut implémenter l’interface `NotificationBroacaster`

Une classe qui implémente sa propre interface de `MBean` est un MBean Standard. Cette manière de procéder est la plus simple. Dans ce cas, les opérations et attributs accessibles sont déterminés statiquement grâce à l’introspection de l’interface du MBean.

Pour un Dynamic MBean (et donc une classe implémentant l’interface `DynamicMBean`), les opérations et les attributs sont exposés de manière indirecte à l’agent JMX qui invoque alors des méthodes pour déterminer le nom et la nature des opérations et attributs. 

Un MBean n’est pas obligatoirement une classe publique Java. Cependant, elle doit implémenter une interface publique (que ce soit ses propres opérations pour un MBean standard ou l’interface `DynamicMBean` pour un MBean dynamique). 

En outre, le MBean peut avoir plusieurs constructeurs qui doivent être publiques s’il doit être instancié par un agent.

<a name="mbean_standard"></a>
## MBean Standard
Comme il a été vu précédemment que, pour être gérable via un agent JMX, un MBean Standard doit explicitement définir son interface de gestion et doit être conforme à certaines règles de nommage qui sont inspirées du modèle des JavaBean.

Ainsi, l’interface de gestion d’un MBean Standard peut être composée de :

* ses constructeurs : seuls les constructeurs publics sont exposés,
* ses attributs : les propriétés qui sont exposées le sont au travers des getter et setter,
* ses opérations : les méthodes qui sont exposées sont toutes les méthodes qui ne sont pas des getter ou des setter,
* ses notifications.

En outre, l’interface qui est implémenté par le MBean doit porter le même nom que ce dernier mais avec le suffixe `MBean` (par exemple, si la classe à gérer se nomme `MyClass`, alors l’interface devra se nommer `MyClassMBean`).

Une classe peut aussi être administrable par son héritage si elle respecte un des modèles suivants :

Modèles | Commentaires 
--- | --- 
![small](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCNQHfzV3I/AAAAAAAAALc/YTeLXLYZLSM/s1600/jmx02.png)|Les opérations exposées sont a1 et a2.
![small](http://4.bp.blogspot.com/_XLL8sJPQ97g/TNCNQQW_w2I/AAAAAAAAALg/shMqH7XIyfk/s200/jmx03.png)|Si B étend A sans implémenter sa propre interface, alors B hérite de l’interface de management AMBean et les opérations exposées sont a1 et a2.
![small](http://4.bp.blogspot.com/_XLL8sJPQ97g/TNCNQpfTdAI/AAAAAAAAALk/aXy_rHOAVbY/s1600/jmx04.png)|B hérite de A et implémente sa propre interface de management BMBean. L’opération exposée est b2.
![small](http://2.bp.blogspot.com/_XLL8sJPQ97g/TNCNRII-pTI/AAAAAAAAALo/P97HDUlqIYw/s200/jmx05.png)|B hérite de A et implémente son interface de management qui hérite de celle de A. Les opérations exposées sont a1, a2 et b2. A noter que si B n’étendait pas A, les opérations exposées seraient similaires.
![small](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCNR6tYsnI/AAAAAAAAALw/2vLb7jJai6s/s1600/jmx07.png)|B implémente son interface de management qui hérite de l’interface AMBean. Les opérations exposées sont a1, a2 et b2.

Les attributs sont les champs ou les propriétés des MBean implémentés (ou étendus). Ils sont déterminés par leurs noms (la méthode `Character.isJavaIdentifierStart` (resp. `Character.isJavaIdentifierPart`) doit renvoyer vrai pour sa première lettre (resp. ses autres lettres)) et par leurs types et sont accédés au travers de leurs accesseurs. Aussi, si un attribut n’a pas d’accesseurs, il ne sera pas visible puisque, pour rappel, ses accesseurs permettent également de déterminer son niveau d’accessibilité (lecture seule, écriture seule ou lecture-écriture). Cependant, il est à noter qu’un attribut ne doit avoir qu’un seul _getter_ et/ou _setter_ et peut être de n’importe quel type Java (cela inclus également les tableaux).


<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Un attribut doit suivre la règle de nommage suivante :
<ul><li>Sa première lettre doit renvoyée vrai à la méthode <i>Character.isJavaIdentifierStart</i></li>
<li>Ses autres lettres doivent renvoyées vrai à la méthode <i>Character.isJavaIdentifierPart</i></li>
</ul></td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Les attributs et opérations sont case sensitives.<br/>En outre, un attribut ne peut avoir qu’un seul getter et/ou setter.<br/>
Par exemple, les attributs suivants ne sont pas valides :

```java
boolean valid ;
int number ;
boolean isValid() {return this.valid ;}
boolean getValid() {return this.valid ;}
void setNumber(int number) {this.number = number ;}
void setNumber(String number) {this.number = Integer.valueOf(number) ;}
```
</ul></td></tr>
</tbody></table>

Les opérations JMX peuvent avoir différentes fonctions (comme effectuer une opération interne sur la ressource managée ou renvoyer une valeur) et sont des méthodes Java qui doivent être spécifiées dans l’interface de management (MBean). Ainsi, toute opération qui ne répond pas au design pattern défini par la définition d’attribut JMX est vue comme une opération.

<a name="mbean_dynamic"></a>
## Dynamic MBean
Alors que les MBean Standard permettent de gérer des ressources bien définies, les Dynamics MBean offrent une solution plus flexible. En effet, les Dynamics MBean sont des ressources qui sont instrumentés au travers d’une interface prédéfinie qui n’expose les attributs et les opérations qu’à l’exécution. En effet, au lieu d’exposer ses opérations et attributs au travers de méthodes déterminées à l’avance, les Dynamics MBean implémentent une méthode qui retourne tous ses attributs ainsi que la signature de toutes ses méthodes, permettant ainsi de rendre l’exposition dynamique. 

En fait, un MBean qui implémente l’interface `DynamicMBean` ne fonctionne pas par introspection comme les MBean Standard mais appelle des méthodes qui lui renvoient les attributs et opérations gérables.

Ainsi, un MBean Dynamic est une classe Java qui implémente directement ou indirectement (ie. par héritage) l’interface publique `DynamicMBean`.

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCUxCaOtDI/AAAAAAAAAL8/xrRlOQfuDAo/s1600/jmx13.png)

En fait, cette interface définie les méthodes suivantes :

* `getMBeanInfo` qui retourne une instance d’un MBeanInfo et qui contient la définition de l’interface de management du MBean, c'est-à-dire la liste des ses attributs associés à leurs types et leurs propriétés (lecture-seule, écriture, lecture-écriture), de ses opérations associées à leurs signatures, la liste des notifications ainsi que la description des constructeurs offerts par le MBean.

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TNCNStck85I/AAAAAAAAAL0/FeAhq8knqNM/s1600/jmx08.png)

* `getAttribute` et `getAttributes` qui retournent le ou les attributs demandés.
* `setAttribute` et `setAttributes` qui permettent de renseigner les attributs.
* `invoke` qui permet d’invoquer toutes les opérations offertes par le MBean.

En outre, tout comme pour les MBean Standard, les Dynamic MBean peuvent hériter leur instrumentation d’une superclasse, mais il est à noter que l’interface de management ne peut être composée d’un arbre d’héritage puisque l’interface de management est fournie par le `MBeanInfo`. 

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCVlbEX3AI/AAAAAAAAAMA/_H914xn8sP8/s1600/jmx15_1.png)

Ainsi, dans l’exemple suivant, si la classe D étend la classe C sans redéfinir la méthode `getMBeanInfo`, alors D aura la même interface de management que C. Cependant, D pourra, en redéfinissant les accesseurs redéfinir l’implémentation de ses attributs.

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCNR6tYsnI/AAAAAAAAALw/2vLb7jJai6s/s1600/jmx07.png)

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le serveur MBean ne teste ni ne valide la description du Dynamic MBean : c’est au développeur de vérifier si l’interface de management est conforme à l’implémentation interne.
</td></tr>
</tbody></table>

<br/>

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
De par sa nature, un Dynamic MBean est susceptible d’évoluer à l’exécution de l’application. Cependant, seules des applications de management propriétaires peuvent prendre en compte ces changements puisqu’il est nécessaire de les notifier en cas de changements.
</td></tr>
</tbody></table>

><a name="notification"></a>
# Notification

Les interfaces de gestion des MBean permettent à un agent de récupérer ou de modifier la valeur des attributs des ressources ainsi que d’invoquer des opérations sur ces dernières. Cependant, JMX permet également de notifier les applications de gestion en cas de changement d’état du système (et plus particulièrement des ressources).

Ainsi, JMX propose un modèle pour émettre des événements (appelés __Notification__) fonctionnant sur le principe d’écouteurs : les applications de managements et les autres objets enregistrés peuvent s’enregistrer comme des écouteurs (`NotificationListener`) du MBean émetteur de notifications (`Notification`). Il est cependant à noter qu’il ne leur est possible de ne s’enregistrer qu’une seule fois auprès de l’émetteur. 

![center](http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCYqpaFn1I/AAAAAAAAAME/hy9yqTxYLjU/s1600/jmx11_1.png)

Les notifications sont encapsulées dans une implémentation de l’interface `Notification` qui est composée :

* du type de notification (de type `String`) qui est formatée à la façon des propriétés java (ie. une chaine de caractère utilisant des points. Par exemple : `vendorName.resourceA.eventA1`). Il est à noter que les types de notification préfixés par JMX. sont réservées aux notifications émises par l’infrastructure JMX (par exemple,  `JMX.mbean.registered`).
* d’un numéro de séquence qui permet d’identifier une instance particulière de notification dans un contexte de broadcast.
* du timestamp qui permet de savoir quand la notification a été générée.
* du message émis (de type `String`) qui peut permettre d’expliciter la notification à l’utilisateur.
* d’information utilisateur (de type `Object`) qui peut contenir d’autres types d’information. 

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCZPDKjyTI/AAAAAAAAAMM/4AH_JVk_0Rk/s1600/jmx14_1.png)

Du coté de l’émetteur, il doit implémenter soit l’interface `NotificationBroadcaster`, soit l’interface `NotificationEmitter`.

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/TNCZnN_WiiI/AAAAAAAAAMQ/U3qrTB-431c/s1600/jmx12_1.png)

En outre, il est également possible de filtrer les notifications reçues en implémentant l’interface `NotificationFilter`.

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/TNCZ3MbN7WI/AAAAAAAAAMU/kM4ERUGeCBg/s1600/jmx10_1.png)

<a name="mbean_open"></a>
# Open MBean

Les Open MBean sont un-sous type des Dynamic MBean. En fait, il s’agit de Dynamic MBean mais dont le type des attributs et des signatures des opérations a été normalisé afin de permettre une meilleure interopérabilité entre le MBean et l’interface de management. Ainsi, l’application de management et les Open MBean peuvent partager et utiliser les données de management (attribut) et invoquer les opérations sans nécessiter une recompilation, un ré-assemblement ou un linkage dynamique, opérations qui nécessiteraient que l’application de management ait accès aux classes Java de l’agent si les types n’étaient pas normalisés. 

Cependant, les Open MBean ne sont que des Dynamic MBean, c'est-à-dire que c’est à la charge du développeur de vérifier qu’ils n’utilisent que les types définis supportés par les Open MBean. 

Aussi, la méthode `getMBeanInfo()` doit retourner une instance d’un `OpenMBeanInfoSupport` plutôt qu’une instance d’un `MBeanInfo` (pour rappel, un Dynamic MBean doit implémenter l’interface `DynamicMBean` qui possède la méthode `getMBeanInfo()` et qui renvoie un objet de type `MBeanInfo`). 

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TNCaO8HRaJI/AAAAAAAAAMY/dAkSqsvCRbQ/s1600/jmx17_1.png)

Les types supportés par les Open MBean sont :

| Type de données sypportées par les Open MBean ||
| :----: | :----: |
| java.lang.Boolean | java.lang.Float |
| java.lang.Byte |	java.lang.Integer |
| java.lang.Character	| java.lang.Long |
| java.lang.Double | java.lang.Short |
| boolean[] | float[] |
| byte []	| int[] |
| char[]	| long[] |
| double[] |	short[] |
| java.lang.String	| java.lang.Void (retour d’opérations seulement) |
| java.math.BigDecimal |	java.math.BigInteger |
| java.util.Date	| javax.management.ObjectName |
| javax.management.openmbean.CompositeData (interface) | javax.management.openmbean.TabularData (interface) |

Les types `CompositeData` et `TabularData` sont utilisés pour agréger les données basiques (tels que les String ou les tableaux de int) et fournissent un mécanisme pour fournir des structures de données complexes. Bien que ces types soient des types simples (au sens JMX du terme), elles peuvent contenir d’autres `CompositeData` ou `TabularData`.

En fait, une implémentation d’un `CompositeData` est équivalente à une `Map`, alors qu’une instance d’un `TabularData` est équivalente à un tableau de `CompositeData`.

Il est à noter qu’un `CompositeData` est une structure immuable. Ainsi dès qu’il est instancié, il n’est pas possible de le modifier. Le `TabularData` peut, quant à lui, être modifié.

![center](http://4.bp.blogspot.com/_XLL8sJPQ97g/TNCb7nMHwgI/AAAAAAAAAMc/2_IcnEmbYAU/s1600/jmx17_bis.png)

JMX fournit une implémentation de ces interfaces qu’il est possible d’utiliser. Pour le `CompositeData`, il s’agit de `CompositeDataSupport` et, pour le `TabularData`, `TabularDataSupport` (classe qui définie une structure de tableau contenant des CompositeData qui doivent avoir le même `CompositeType`).

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Les classes GCInfo, MemoryNotifInfoCompositeData, MemoryUsageCompositeData, MonitorInfoCompositeData, StackTraceElementCompositeData, ThreadInfoCompositeData et VMOptionCompositeData implémentent également l’interface CompositeDataSupport.
</td></tr>
</tbody></table>

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCcUpilkdI/AAAAAAAAAMg/GnDbK9JVRqo/s1600/jmx18_1.png)

Cependant, le fait d’utiliser des types de données complexes nécessite que l’Open MBean doit décrire la structure des données. Pour cela, les Open MBean proposent une classe abstraite `OpenType` qui décrit aussi bien ses types basiques que ses types complexes. Cette classe abstraite est donc étendue par des classes qui décrivent chacun de ses types cités ci-dessus en spécifiant le nom du type, sa description ainsi que la classe qui la spécifie. Cela permet, pour les types complexes, de fournir une description et un nom de chaque élément qui le compose.

![large](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCccuJTwzI/AAAAAAAAAMk/tAsptdhxVj8/s1600/jmx20.png)


De plus, comme indiqué précédemment, un Open MBean étant un Dynamic MBean, il doit s’auto décrire via l’interface MBeanInfo. Pour cela, JMX propose, par défaut, des sous-classes des classes permettant de décrire un Dynamic MBean (`MBeanInfo`, `MBeanAttributeInfo`, `MBeanOperationInfo`, `MBeanParameterInfo`, `MBeanConstructorInfo`) et dont le nom est préfixé par Open et postfixé par Support (`OpenMBeanInfoSupport`, `OpenMBeanAttributeInfoSupport`, `OpenMBeanOperationInfoSupport`, `OpenMBeanParameterInfoSupport`, `OpenMBeanConstructorInfoSupport`).

Les notifications sont, quant à elle, standard.
![large](http://2.bp.blogspot.com/_XLL8sJPQ97g/TNCczbc-JCI/AAAAAAAAAMo/ivfzCGCGR0A/s1600/jmx21_1.png)

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCdCn-DxlI/AAAAAAAAAMs/61pQhkzTh4c/s1600/jmx22_1.png)

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TNCdZywnsaI/AAAAAAAAAMw/pWaoBFBfLR4/s1600/jmx23_1.png)

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TNCdaX_4lPI/AAAAAAAAAM0/D_ck5n0AsvE/s1600/jmx25_1.png)

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCdtEsWmrI/AAAAAAAAAM4/gE6SvL-vWVI/s1600/jmx26_1.png)

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Pour développer un Open MBean, il faut :<br />
<ul><li>Que la ressource à manager implémente l'interface <i>DynamicMBean</i></li>
<li>Que tous les attributs et les signatures des méthodes soient des types supportés par les OpenMBean</li>
<li>Que l'implémentation de la méthode <i>getMBeanInfo</i> retourne une instance de l'interface <i>OpenMBeanInfo</i> (par exemple, une instance d'un <i>OpenMBeanInfoSupport</i>) qui doit retourner des objets de type <i>OpenMBeanXXXInfo</i></li>
<li>Que les méthodes suivantes retournent des objets valides et non null :  <ul><li><i>OpenMBeanInfo.getDescription</i></li>
<li><i>OpenMBeanOperationInfo.getDescription</i></li>
<li><i>OpenMBeanConstructorInfo.getDescription</i></li>
<li><i>OpenMBeanParameterInfo.getDescription</i></li>
<li><i>OpenMBeanAttributeInfo.getDescription</i></li>
<li><i>MBeanNotificationInfo.getDescription</i></li>
</ul></li>
<li>Que la méthode <i>OpenMBeanOperationInfo.getImpact</i> retourne une des constantes suivantes :  <ul><li><i>ACTION</i>,</li>
<li><i>INFO</i>,</li>
<li><i>ACTION_INFO</i>.</li>
</ul></li>
</ul>
</td></tr>
</tbody></table>

<a name="mbean_model"></a>
# Model MBean
Un Model MBean est un MBean générique configurable ayant pour objectif de fournir un patron de MBean pouvant être utilisé simplement par n’importe quelles ressources. Il s’agit, en fait, d’un MBean Dynamic particulier dont l’interface définie une structure qui, lorsqu’elle est implémentée, fournie un MBean avec un comportement par défaut. Ces Model MBean devant être supportés par les agents JMX, ils peuvent être utilisés par n’importe quels applications, ressources et services pour créer un objet gérable à l’exécution : les utilisateurs n’ont qu'a instancier un Model MBean, configurer son comportement par défaut et l’enregistrer au sein de l’agent JMX.

En fait un Model MBean est constitué d’un ensemble d’interfaces et de classes concrètes fournit par l’agent JMX (qui doit fournir une implémentation de la classe `RequiredModelMBean` et qui a pour objectif de fournir un comportement par défaut).

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCd8pEaVUI/AAAAAAAAAM8/zp_uL_-y7p8/s1600/jmx29.png)

Un MBean Server est donc un repository et factory de Model MBean qui peuvent être obtenus au travers de l’agent JMX afin de rendre une ressource gérable : le développeur n’a pas besoin de fournir une implémentation de cette classe mais juste la configurer à l’exécution afin d’exposer l’interface d’administration et de supervision nécessaire à la ressource.

La ressource à administrer et à superviser ajoute ses attributs, ses opérations et ses notifications à l’objet basique Model MBean en s’interfaçant avec l’agent JMX et à son ou ses Model MBean.

Comme précisé précédemment, un Model MBean est un Dynamic MBean et doit donc implémenter l’interface `DynamicMBean`.

Il est également à noter que l’implémentation du `RequiredModelMBean` est dépendante de l’environnement et plus précisément de la JVM : elle peut fournir des mécanismes de persistance, de transaction, de cache, de scalabilité ou de fonctionnement distant en fonction des besoins. Ainsi, le développeur n’a pas à se soucier de problématiques comme celles de persistance en se fiant à l’implémentation de la JVM sur lequel fonctionne son application (par exemple, un environnement J2ME n’a pas besoin d’offrir des propriétés de persistance ou d’accès distant) : il peut le déléguer au Model MBean exposé.

Une implémentation d’un Model MBean doit implémenter l’interface `ModelMBean` qui étend les interfaces `DynamicMBean`, `PersistentMBean` et `ModelMBeanNotificationBroadcaster`. 

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCeTxNgCPI/AAAAAAAAANA/P68PBgsxiEY/s1600/jmx27_1.png)

En outre, le Model MBean doit exposer ses méta-données dans un objet de type `ModelMBeanInfoSupport` qui étend la classe `MBeanInfo` et qui implémente l’interface `ModelMBeanInfo`.

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCekBf1QzI/AAAAAAAAANE/uRwZx4k-7CU/s1600/jmx28_1.png)


De plus, une instance d’un Model MBean peut émettre des notifications lors de changements de la valeur de ses attributs et possède un constructeur par défaut et un constructeur qui prend en paramètre une instance d’un `ModelMBeanInfo`. Il est à noter que chaque attribut, constructeur, opération et notification peut fournir sa description. Ces descriptions peuvent être de différentes natures comme la politique de connexion, la politique de notifications, … 

Enfin, les descriptions d’un Model MBean fournissent un mapping entre les attributs et les opérations de l’interface de gestion et les méthodes utilisées par la ressource à gérer (méthodes set, get et invoke). Il est à noter que ces ressources peuvent être indifféremment sur une autre JVM si le Model MBean a été configuré. 

Concrètement, une ressource managée donc doit utiliser l’interface `ModelMBeanInfo` pour exposer son interface d’administration et de supervision. A l’initialisation, la ressource managée créée ou trouve via le MBean Server une ou des instances d’un ModelMBean qui peuvent alors être configurées avec son interface de management (via une implémentation de l’interface `ModelMBeanInfo`).

De la même manière, l’application de management (l’application cliente des ressources à administrer et superviser) via le serveur MBean peut accéder aux descriptions, aux opérations et aux notifications du ModelMBean afin de pouvoir les exposer à l’administrateur en lui fournissant le maximum d’information.
Afin de permettre au Model MBean de fournir la description de ses différentes opérations, notifications et attributs, JMX propose, par défaut, des sous-classes des classes permettant de décrire un ModelMBean : `ModelMBeanInfoSupport`, `ModelMBeanAttributeInfo`, `ModelMBeanOperationInfo`, `ModelMBeanNotificationInfo`, `ModelMBeanConstructorInfo`).

Ces classes implémentent l’interface `DescriptorAccess` qui permet de remplacer l’interface `Descriptor` pour chaque attribut, constructeur, opération et notification dans l’interface de management. La description est alors accédée au travers des méta-données de chaque composant.

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TNCfZU2SohI/AAAAAAAAANI/OXAP8J6klxE/s1600/jmx32.png)

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TNCfieyUY9I/AAAAAAAAANM/r9J5xW6yD1Y/s1600/jmx35.png)

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCfphXk6qI/AAAAAAAAANQ/HyNPv6cxZy8/s1600/jmx34.png)

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TNCfzx5EXdI/AAAAAAAAANU/sypPZS0mmy8/s1600/jmx36.png)

L’interface `NotificationBroadcaster`, qui permet lorsqu’elle est implémentée par une MBean d’émettre des notifications génériques ou spécifiques et/ou lorsqu’un changement de valeur d’un attribut se produit à des MBean déclarés comme écouteur, est implémentée pour les Model MBean par l’interface `ModelMBeanNotificationBroadcaster`.

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCgZrU3ClI/AAAAAAAAANY/7OggAyAVTas/s1600/jmx36_bis.png)

L’interface `Descriptor` fournit les descriptions des opérations, attributs et notifications du Model MBean qui ne sont pas accédées directement mais au travers de l’interface `DescriptorAccess` qui définit comment récupérer et positionner la valeur des différents champs qui constituent la description.

Par défaut, JMX fournit deux implémentations de l’interface `Descriptor` : `ImmutableDescriptor` et `DescriptorSupport`.

![small](http://4.bp.blogspot.com/_XLL8sJPQ97g/TNCgiOsFsOI/AAAAAAAAANc/JzGI0sNlKj8/s1600/jmx37.png)

En fait, l’interface `ModelMBeanInfo` publie des méta-données sur les attributs, les opérations et les notifications déclarés dans l’interface d’administration et de supervision. Les descripteurs du Model MBean fournissent, quant à eux, le comportement et les règles de ces attributs, opérations et notifications. Un descripteur se présente sous forme d’un ensemble de clé/valeur où la clé est de type String et la valeur est de type Object qui peut être modifié (ajout, modification et suppression des champs) par la ressource managée ou par l’application de gestion. Il est à noter que certains noms sont réservés pour des propriétés prédéfinies telles que la gestion du cache ou de la persistance. Les descripteurs contiennent également le nom des opérations des getter et setter utilisées pour lire et écrire la valeur des attributs. 

Comme indiqué précédemment, les descripteurs sont des objets qui implémentent l’interface `Descriptor` et qui sont accessibles au travers des méthodes définies par l’interface `DescriptorAccess` dont les méthodes doivent être définies par les implémentations concrètes des interfaces `ModelMBeanAttributeInfo`, `ModelMBeanOperationInfo`, `ModelMBeanConstructorInfo` et `ModelMBeanNotificationInfo` (qui sont accessibles via l’instance de `ModelMBeanInfo`).

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Si le descripteur d’un attribut ne possède pas les signatures de ses méthodes, alors l’application ne peut gérer l’attribut : lorsque la méthode setAttribute est invoquée, la valeur est enregistrée dans le descripteur et une notification de type AttributeChangeNotification est émise à destination des écouteurs enregistrés comme intéressés par cet attribut.
<br/>
Si la méthode getAttribute est invoquée, la valeur est renvoyée par le descripteur mais sa valeur ne peut pas être rafraichie par la ressource managée.
</td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le Model MBean ne loguera les notifications que si le champ logfile dans le descripteur associé à la notification est renseigné.<br/>
La journalisation des notifications ne se fera que si l’implémentation du Model MBean ou de l’agent JMX le supporte. Dans le cas contraire, les champs log et logfile seront ignorés.
</td></tr>
</tbody></table>

<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
La politique de persitance peut être implémentée optionnellement. Cependant, même si elle est géré par le Model MBean, il n’est pas obligé que ce soit ce dernier qui l’implémente et elle peut être offerte par l’agent JMX qui peut proposer en différents niveaux.<br/>
Lorsqu’aucune politique n’est implémentée, l’objet est transient par défaut. La politique de persistance peut être une implémentation simple offerte en sérialisant dans un fichier les données.<br/>
Les champs prédéfinis persistLocation et persistName peuvent être utilisés pour définir l’emplacement des données persistées.
</td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
La politique de cache d’une donnée (opération ou attribut) est définie par une valeur par défaut présente dans le descripteur de la donnée. En général, l’adaptateur accède au Model MBean de l’application au travers de l’agent JMX mais une interrogation systématique n’est pas toujours nécessaire (ou voulue) : c’est pourquoi JMX offre la possibitilié de définir un cache d’accès à la ressource.<br/>
Pour ce faire, la description d’un attribut possède les clés prédéfinies currencyTimeLimit et lastUpdatedTimeStamp.
</td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Le comportement par défaut offert par un Model MBean est généralement suffisant à la plupart des besoins. Cependant, JMX offre la possibilité de relier le Model MBean de l’application à un modèle de données différent. Cela peut se produire dans le cas, par exemple, où un MIB spécifique existe dans le système : le générateur de MIB peut alors intéragir directement avec l’agent JMX et créer un fichier MIB qui sera chargé par un système de supervision SNMP.<br/>
Pour cela, le descripteur d’un attribut ou d’une opération propose le champ protocolMap qui doit alors contenir une référence à une instance de la classe implémentant l’interface Descriptor.
</td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Si l’implémentation d’un agent JMX supporte les opérations dans un environnement multi-agent JMX, alors il peut avoir besoin d’avertir de son existance et de sa disponibilité un service de recherche. Il peut également avoir besoin d’enregistrer en son sein des MBean qui peuvent être localisés sur d’autres agents JMX sans avoir à connaitre ces derniers.<br/>
Pour ce faire, le descripteur de son ModelMBeanInfo propose le champ prédéfini export dont la valeur peut être le nom ou l’objet requis pour exporter le MBean. S’il vaut F ou false, alors le MBean ne sera pas exporté.
</td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Il peut parfois être nécessaire d’avoir différent niveau de lisibilité sur les différents MBeans du système.
Pour cela, il est possible d’utiliser le champ visibility du descripteur. Il peut prendre une valeur entière de 1 à 4 et peut s’appliquer au niveau du MBean, d’un attribut ou d’une opération.<br/>
Dans ce cas, c’est à l‘implémentation de l’adaptateur de protocole, du connecteur ou de l’interface de gestion de filtrer les différentes informations à remonter.
</td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Dans le cas où l’interface de gestion doit etre dynamique (ie. qu’elle doit générer son interface en fonction des MBeans découverts ou présents) il est possible d’utiliser le champ presentationString du descripteur.
Il doit contenir une chaine de caractères au format XML.<br/>
Il est à noter, qu’actuellement, il n’existe pas de standardisation de la représentation de ce champ.
</td></tr>
</tbody></table>
<br/>
<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Une liste exhaustive des champs prédéfinis des descripteurs peut être trouvée dans la version agrégée de la spécification <a href="http://download.oracle.com/javase/6/docs/technotes/guides/jmx/JMX_1_4_specification.pdf">JMX au chapitre 4.5 Predefined Descriptor Fields</a>.
</td></tr>
</tbody></table>

# Le mot de la fin de cette partie

Ainsi s'achève cette partie qui a permis de voir les notions de MBeans (en particulier les différences entre les différents MBeans) et de Notification.

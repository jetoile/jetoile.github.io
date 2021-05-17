---
title: "JGroups : tour d'horizon"
date: 2010-12-28 17:42:18 +0100
comments: true
tags: 
- java
- jgroups
---

![left-small](http://2.bp.blogspot.com/_XLL8sJPQ97g/TRtn71JxZPI/AAAAAAAAATI/0vNqJjOXKlY/s1600/jgroups_logo_450px.png)
Actuellement, le besoin de rendre les systèmes ou les applications interopérables est indéniable. Pour ce faire, de nombreuses technologies ont vues le jour. Il peut s'agir des MOM (_Message-Oriented Middleware_), des protocoles de communication entre les applications clientes et serveurs (REST, Soap, RMI, ...), de solutions propriétaires utilisées pour permettre la communication entre les serveurs pour des problématiques de réplication ou de synchronisation ou même de solutions basées sur le paradigme NoSQL.

Cependant, il est important de ne pas oublier que nos couches basses restent les protocoles TCP ou UDP et que répondre par un grand concept tel que MOM, réplication de serveurs ou cache distribué sur une problématique ne permet pas de savoir ce qui se trame derrière... 

Un produit souvent méconnu est utilisé en interne pour les problématiques de communication entre serveurs et même si, à ce jour, je ne connais pas d'implémentations de MOM qui l'utilise, il pourrait être un très bon candidat pour ces dernières en interne. Il s'agit de JGroups.

Aussi, cet article va tenter de donner un rapide tour d'horizon sur [JGroups](http://www.jgroups.org/) en présentant ses concepts. Il s'appuiera sur sa version stable courante, à savoir la version 2.11.0.

A noter que ce tour d'horizon s'appuie très fortement sur le manuel de référence de JGroups.
<!-- more -->

Cet article suivra le plan suivant :

* [Un peu d'histoire](#histoire)
* [Présentation des concepts](#concept)
	* [Canal](#canal)
	* [Bloc de construction](#bloc)
	* [Pile de protocoles](#pile)
	* [Entête](#entete)
	* [Evénement](#evenement)
* [Présentation des APIs de JGroups](#api)
	* [Les interfaces](#interface)
		* [MessageListener](#messageListener)
		* [MembershipListener](#membershipListener)
		* [ChannelListener](#channelListener)
		* [Receiver](#receiver)
		* [ReceiverAdapter](#receiverAdapter)
	* [L'interface Address](#address)
	* [La classe Message](#message)
	* [La classe View](#view)
		* [La classe ViewId](#viewId)
		* [La classe MergeView](#mergeView)
		* [La classe JChannel](#jchannel)
* [Les blocs de construction](#construction)
	* [MessageDispatcher](#messageDispatcher)
	* [RpcDispatcher](#rpcDispatcher)
	* [ReplicatedHashMap](#replicationHashMap)
	* [NotificationBus](#notificationBus)
* [Liste des protocoles](#protocole)
* [Conclusion](#conclusion)

<a name="histoire"></a>
#Un peu d'histoire...

Créé par Bela Ban, JGroups a vu le jour suite à son travail universitaire dans les équipe de Ken Birman sur le framework _Ensemble_ qui, en 1998-1999, était un prototype pour la troisième génération de communication de groupes. _Ensemble_ faisait suite à _Horus_ (écrit par Robbert VanRenesse) qui faisait lui-même suite à _ISIS_ (écrit par Keb Birman) et était écrit en OCaml. _Ensemble_ proposant une interface pour s'interfacer avec Java mais s'appuyant sur son coeur, cela a été le début de JGroups (enfin pour être plus précis le début de _JChannel_ puis de la partie _ProtocolStack_) qui avait pour objectif de fournir une solution full Java de Ensemble.

Ainsi, JGroups a pris son essor en 2000 avec le départ de Bela Ban du pôle de recherche de l'université de Cornell pour enfin être utilisé par JBoss dès 2002 avec, entre autre _JBossCache_.

<a name="concept"></a>
#Présentation des concepts

La communication entre groupes utilise les termes de groupes et de membres qui font partis intégrantes des groupes. En fait, un membre peut être vu comme un noeud et un groupe comme un cluster.

En fait, un noeud est un processus qui réside sur une machine hôte et un cluster est constitué d'un ou de plusieurs noeuds, sachant que plusieurs noeuds peuvent résider sur le même hôte et qu'ils peuvent appartenir à un ou plusieurs clusters.

JGroups est un framework permettant la communication fiable entre groupes et permets aux processus :

* de rejoindre un groupe,
* d'envoyer des messages à tout ou partie des membres du groupe,
* et de recevoir les messages des membres du groupe.

En outre, JGroups permet de garder de notifier les membres de chaque groupe lors de l'arrivé d'un nouveau membre ou d'un départ ou arrêt brutal de l'un d'eux.

Un __groupe__ est identifié par son nom et n'a pas besoin d'être créé explicitement puisqu'il est créé automatiquement.

En fait, l'architecture de JGroups s'articule autour de trois points :

* les canaux (__channel__) utilisés par les applications pour leurs permettre de prendre part au groupe de communication fiable,
* les blocs de construction (__building blocks__) qui sont des couches au dessus des canaux qui permettent d'abstraire ces derniers de la pile de protocoles à utiliser,
* la pile de protocoles qui implémentent les propriétés d'un canal donné.  

En fait, un canal est connecté à une pile (_stack_) de protocoles et lorsqu'une application émet un message, le canal le transmet au travers de cette pile pour effectuer séquentiellement des traitements jusqu'à la couche réseau. De manière similaire, pour qu'une application reçoive un message, ce dernier doit repasser dans cette pile de protocole (dans l'ordre inverse des traitements qu'il a subi lors de son émission) jusqu'à arriver à canal qui gère la file des messages jusqu'à leurs consommations par l'application.

Il est à noter que lorsqu'une application se connecte à un canal, la pile de protocole est démarrée et que lorsqu'elle se déconnecte, cette même pile est arrêtée. Quand le canal est fermé, la pile est détruite permettant ainsi de libérer ses ressources.

<a name="canal"></a>
##Canal (Channel)
Pour joindre un groupe et émettre des messages, un processus doit créer un canal et s'y connecter en utilisant le nom du groupe puisque tous les canaux qui possèdent le même nom forment un même groupe. 

Une fois connecté, un membre peut émettre (resp. recevoir) des messages aux (resp. des) autres membres du groupe.

Il quitte le groupe en se déconnectant du canal permettant ainsi sa réutilisation. Cependant, un canal ne gère qu'un seul client simultanément.

Le client a également la possibilité de signaler qu'il ne désire plus utiliser un canal en le fermant. Dans ce cas, le canal ne peut plus être utilisé.

Chaque canal dispose d'une adresse unique et est toujours connu des autres membres de son groupe : une liste d'adresse des membres peut être récupérée d'un canal et est appelé vue (__view__).

Un processus peut alors sélectionner une adresse dans cette liste et émettre un message à destination de l'unique membre associé à cette adresse (mais également à lui-même puisqu'il appartient à la vue) ou émettre un message à tous les membre de la vue (et donc au groupe).

Pour gérer les vues :

* lorsqu'un membre rejoint le groupe ou qu'il s'arrête (déconnexion ou détection de crash), JGroups envoie une nouvelle vue à tous les membres du groupes.
* lorsqu'un membre est suspecté comme ayant potentiellement crashé, un message de suspicion est émis à destination de tous les membres non fautif.

Ainsi, un canal peut recevoir trois types de messages ;

* les messages normaux (ie. un message émis par une application),
* les messages de vues lors de l'ajout ou du retrait d'un de ses membres,
* les messages de suspicion.  

Bien sûr, un client peut choisir s'il accepte de recevoir les messages de types vue ou suspicion.
En fait, les canaux sont similaires à des sockets BSD ; les messages sont stockés dans le canal jusqu'à temps qu'ils soient consommés par un client (approche _pull_). Dans le cas où il n'y a pas de messages, le client est bloqué. Cependant, JGroups propose également la notion de push pour recevoir les messages en se basant sur la notion de _callback_ (ou, du point de vue client, de _Listener_). Il est à noter que le fonctionnement en mode _pull_ tend à être supprimé en version 3 de JGroups : il est donc déconseillé de l'utiliser.

Afin de positionner les propriétés sur un canal, JGroups s'appuie sur une configuration XML.

<a name="bloc"></a>
##Bloc de construction (Building Blocks)
Le canal est un concept simple qui permet de fournir la fonctionnalité basique d'un groupe de communication en proposant une douzaine de méthodes qui permettent la communication entre applications via la transmission de messages asynchrones sur le même modèle que UDP. Ainsi, par exemple, le canal de base ne gère pas l'ordonnancement des messages et il est alors à la charge de l'application de réordonnancer les messages. En outre, le fonctionnement en mode _pull_ nécessite généralement au client d'avoir à gérer un thread pour éviter de bloquer l'application lorsqu'il attend un message.

Ainsi, JGroups propose la notion de blocs de construction (__Building Blocks__) qui offre une API plus sophistiquée et qui s'appuie, en interne, sur les canaux. 

Cela permet aux applications de communiquer en utilisant ces derniers en offrant une plus grande facilité d'utilisation du groupe de communication (par exemple, certains blocks permettent de gérer nativement les identifiants des corrélations).

<a name="pile"></a>
##Pile de protocoles (Protocol Stack)
La pile de protocoles contient de nombreuses couches de protocoles qui sont bidirectionnels. Tous les messages émis et reçus par le canal doivent passer à travers ces protocoles qui peuvent :

* modifier,
* réordonnancer,
* supprimer ou laisser passer un message,
* ajouter un entête au message,
* découper le message en messages plus petit,
* réaggréger un ensemble de messages,
* ...

La composition de la pile de protocoles (ie. ses couches) est déterminée par le créateur du canal via un fichier XML qui définit et paramètres les différentes couches à utiliser en lui associant un nom.
Cette séparation des concepts permet ainsi d'abstraire l'application, qui utilise un canal, de comment le message est émis et reçu.

Si aucune pile de protocoles n'est fournie lors de la création du canal, alors c'est sa configuration par défaut qui est utilisé.

<a name="entete"></a>
##Entête (Header)

Un entête (__Header__) est un ensemble d'informations personnalisables qui peut être ajouté à chaque message.

Il est à noter que JGroups utilise cette fonctionnalité dans de nombreux cas de figures comme, par exemple, pour savoir connaitre l'ordonnancement d'envoi des messages (via les entêtes _NAKACK_ et _UNICAST_).

<a name="evenement"></a>
##Evénement (Event)

Les évènements (__Event__) permettent à JGroups de savoir quels protocoles peuvent communiquer avec quel autre. Ainsi, contrairement aux messages qui transitent au travers du réseau entre les membres d'un groupe, les évènements transitent au travers la pile.

<a name="api"></a>
#Présentation générale des APIs de JGroups
<a name="interface"></a>
##Les interfaces
JGroups propose un certain nombre d'interfaces principales sur lesquelles s'appuient les différentes implémentations :

* `MessageListener`
* `MembershipListener`
* `ChannelListener`
* `Receiver`
* `ReceiverAdapter`

<a name="messageListener"></a>
###MessageListener
L'interface `MessageListener` permet d'être notifier lors de la réception d'un message lorsque le mode de fonctionnement est de type push.

Pour ce faire, la méthode `receive(Message msg)` est invoquée. Les méthodes `getState()` et `setState()` permettent, quant à elles, de récupérer l'état du groupe.
![center](http://3.bp.blogspot.com/_XLL8sJPQ97g/TQ4yCVvoq6I/AAAAAAAAAQ8/P05famfiSL4/s1600/jgroups01.png)

<a name="membershipListener"></a>
###MembershipListener

L'interface `MembershipListener` permet d'être notifiée lorsqu'une nouvelle vue, un message de suspicion ou un évènement de bloc est reçu. 

Dans la plupart des cas, c'est la méthode `viewAccepted()` qui est invoquée puisqu'elle notifie lorsqu'un nouveau membre a rejoint le groupe ou que l'un d'entre eux s'est déconnecté ou s'est arrêté brutalement. La méthode `suspect()` permet d'être informé lorsqu'un membre du groupe est suspecté de s'être arrêté brutalement mais qu'il n'a pas encore été exclu. La méthode `block()` est invoquée pour indiqué que le membre courant est sur le point d'être d'être exclu des membres qui ont le droit d'émettre des messages.

![center](http://4.bp.blogspot.com/_XLL8sJPQ97g/TQ4yKn49KiI/AAAAAAAAARA/w8UlwBfI1lc/s1600/jgroups02.png)

<a name="channelListener"></a>
###ChannelListener

L'interface `ChannelListener` est utilisée comme interface de _callback_ lorsque l'application souhaite recevoir des informations sur les changements d'état du canal (fermeture, déconnexion ou ouverture).

![center](http://3.bp.blogspot.com/_XLL8sJPQ97g/TQ4ydDBkt5I/AAAAAAAAARE/kEWjYvTnDFs/s1600/jgroups03.png)

<a name="receiver"></a>
###Receiver

L'interface `Receiver` peut être utilisée pour recevoir des messages et les changements de vue dans le mode _push_. A noter que cette interface est marquée comme dépréciée et sera supprimé en version 3. L'utilisation de `MessageListener` et `MembershipListener` est préconisée.

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/TRoGqAZ288I/AAAAAAAAASg/EshWFLsfHjA/s1600/jgroups04.png)

<a name="receiverAdapter"></a>
###ReceiverAdapter

Cette classe permet de fournir une implémentation par défaut de l'interface `Receiver` en ne permettant d'avoir à implémenter que les méthodes nécessaires (dont la méthode `receive()`).

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/TRoG6cQ9F5I/AAAAAAAAASk/orJ7pVnADug/s1600/jgroups05.png)

<a name="address"></a>
##L'interface Address

Chaque membre d'un groupe dispose d'une adresse qui l'identifie de manière unique. Cette adresse est représentée par l'interface Address qui require une implémentation concrète afin de fournir ses méthodes de comparaison, de trie et afin de permettre de différencier une adresse de type multicast.

Cependant, une implémentation de l'interface `Address` ne doit jamais être utilisée directement et l'interface `Address` doit être vue comme une couche d'abstraction lui permettant d'identifier le noeud dans le cluster. Ses implémentations sont généralement générées par la couche la plus basse du protocole (ie. UDP ou TCP). 

Puisque l'adresse identifie de manière unique un canal et plus précisément un membre du groupe, elle peut être utilisée pour émettre des message.

Ainsi, par exemple, dans le cas de `JChannel`, l'adresse, qui est implémentée par la classe `IpAddress`, correspond à l'IP de l'hôte sur laquelle se trouve le membre et de son port sur lequel sont reçus les messages.
![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TRoHFjviIMI/AAAAAAAAASo/nsGhSwGXwzw/s1600/jgroups06.png)

<a name="message"></a>
##La classe Message

Les échanges de données émis entre les membres se fait au travers de messages qui sont représentés par la classe `Message`. Un message peut être émis, par un membre, aussi bien à un membre ou à tous les membres du groupe du canal.

Il est composé de cinq parties :

* L'adresse de destination : si elle vaut `null`, le message est émis à tous les membre du groupe courant.
* L'adresse de l'émetteur : ce champ est optionnel mais sera renseigné par le protocole de transport avant que le message n'atteigne la couche réseau.
* Les drapeaux (__flags__) qui est codé sur un octet et qui peut valoir l'une des valeurs suivantes : `OOB`, `LOW_PRIO` ou `HIGH_PRIO`.
* Le contenu (__payload__) : codé sous forme de buffers d'octets, la classe `Message` doit contenir les méthodes permettant de sérialiser et désérialiser l'information transmise.
* Les entêtes (__headers__) qui représentent les informations additionnelles au contenu.

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/TQ4y5ZcL1RI/AAAAAAAAARU/3ayXiOTHOIY/s1600/jgroups07.png)

<a name="view"></a>
##La classe View

Une vue représente la liste des membres d'un groupe à un instant donné. Elle consiste en un objet de type `ViewId` qui représente de manière unique la vue ainsi que la liste de ses membres.

Les vues sont gérées automatiquement par la couche de protocoles lorsqu'un nouveau membre rejoint le groupe ou qu'un membre le quitte. Tous les membres d'un groupe voient la même séquence de vues qui sont ordonnées.

Ainsi, généralement, le premier membre d'une vue est le coordinateur (son rôle consiste à émettre les nouvelles vues aux autres membres du groupe). Cela permet, si les membres du groupes changent, de déduire le coordinateur facilement sans avoir à contacter les autres membres du groupe.

A noter que lorsqu'une application est notifiée que le groupe a changé (ie. qu'une nouvelle vue a été reçue), la vue est déjà à sa disposition dans le canal.

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TRoHVP9nFLI/AAAAAAAAASs/bdb-orAyhxo/s1600/jgroups08.png)

<a name="viewId"></a>
###La classe ViewId

La classe `ViewId` est utilisée pour numéroter les vues et consiste en l'adresse du créateur de la vue et en son numéro de séquence.

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TRoHeWcBzbI/AAAAAAAAASw/aSNJllb0Q3o/s1600/jgroups09.png)

<a name="mergeView"></a>
###La classe MergeView

Lorsqu'un groupe se retrouve découpé en sous-groupe (par exemple, lors d'un partitionnement du réseau puis de son ré-aggrégement), les vues peuvent nécessiter un merge.

Dans ce cas, un objet de type `MergeView` qui étend la classe `View` est reçu par l'application. Cette classe contient, en plus de View, la liste des vues qui ont été mergés.

Ainsi, par exemple, si un groupe est décrit par la vue _V1:(p,q,r,s,t)_, qu'il subit un découpage en deux sous-groupe _V2:(p,q,r)_ et _V2:(s,t)_, et que sa vue mergée est _v3:(p,q,r,s,t)_, alors l'objet `MergegView` contiendra la liste des deux vues suivantes : _v2:(p,q,r)_ et _v2:(s,t)_.

![center](http://3.bp.blogspot.com/_XLL8sJPQ97g/TQ4zP2h5POI/AAAAAAAAARg/SUrNkuEtuR8/s1600/jgroups10.png)

<a name="jchannel"></a>
###La classe JChannel

Afin de pouvoir joindre un groupe et émettre des messages (ou en recevoir), un processus doit créer un canal. Un canal peut s'assimiler à une Socket. Quand le client se connecte à un canal, il doit donner le nom du groupe qu'il veut rejoindre. En effet, un canal est (dans son mode connecté) toujours associé à un groupe donné. La pile de protocoles a alors à sa charge de vérifier que le canal se joint au groupe du même nom, résultant un nouvelle vue à installer sur chaque membre du groupe.

Si aucun membre existe, le groupe est créé.

A noter que lors de sa création, un canal est dans un état déconnecté. Pour qu'il puisse commencer à traiter des opérations, il doit être dans un état connecté.

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/TQ40li7dxMI/AAAAAAAAARs/0kqgw2nAHfs/s1600/jgroups12.png)

<a name="construction"></a>
#Les blocs de construction (Building blocks)

Les blocs de construction sont les couches qui se trouvent au dessus des canaux. La plupart ne nécessite pas d'un canal mais seulement d'implémenter l'interface `Transport` (qui est également implémentée par la classe `Channel`). Cela leurs permet de fonctionner pour n'importe quel type de transport qui implémente cette interface. Les blocs de construction peuvent donc être utilisés à la place des canaux si cela est nécessaire. Ainsi, alors que les canaux peuvent être vus comme des sortes de sockets, les blocs de construction peuvent bénéficier d'une interface beaucoup plus sophistiquée.

Dans la suite, seuls les blocs de construction les plus importants seront décrits (à noter que le bloc de construction `PushPullAdapter` étant déprécié, il ne sera pas non plus décrit).

<a name="messageDispatcher"></a>
##MessageDispatcher

Les canaux sont de simple patron pour émettre et recevoir des messages de manière asynchrone. Cependant, il peut être nécessaire de disposer d'un moyen de communiquer de manière synchrone.

Le `MessageDispatcher` offre cette possibilité en fournissant la possibilité d'émettre un message (ou requête) en positionnant un identifiant de corrélation afin de pouvoir associer la réponse avec la requête.

Il offre également la possibilité de recevoir un message en mode push.

Pour l'utiliser, il doit être créé associé avec un canal et peut être utilisé coté émetteur ou récepteur.
S'il est utilisé du coté serveur, alors à chaque requête reçu, la méthode `handle()` provenant de l'interface `RequestHandler` est invoquée.

Cette méthode retourne le message (qui doit être sérialisable) ou lève une exception qui sont transmises au client.

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TQ43hC993TI/AAAAAAAAASE/2mgpujE4eL0/s1600/jgroups14.png)

S'il est utilisé du coté client, il est possible d'utiliser deux méthodes :

* `castMessage()` qui permet de requêter un ensemble de destinataires (dans ce cas, si le destinataire est positionné sur l'entête du message, il sera écrasé) et qui renvoie un objet de type `RspList` (qui implémente l'interface `Map`) 
* `sendMessage()` qui permet de requêter un seul destinataire (dans ce cas, le destinataire doit être positionné sur l'entête du message)

Ces deux méthodes peuvent positionner un certain nombre d'options lors de l'émission du message encapsuler dans un objet de type `RequestOptions` tels que :

* un timeout qui permet d'indiquer le temps qu'il faut attendre la réponse. 
* le type de requête à émettre :
	* dès qu'une réponse arrive, la méthode castMessage() rend la main au programme (`GroupRequest.GET_FIRST`)
	* attend la réponse de chaque destinataire avant de rendre la main au programme (`GroupRequest.GET_ALL`)
	* attend la majorité des réponses (`GroupRequest.GET_MAJORITY`)
	* attend la majorité absolue des réponses (`GroupRequest.GET_ABS_MAJORITY`)
	* n'attend aucune réponse (`GroupRequest.GET_NONE`)
* un filtre pour les réponses attendues à l'aide d'un objet de type `RspFilter`

![center](http://4.bp.blogspot.com/_XLL8sJPQ97g/TRoHzicdvtI/AAAAAAAAAS0/OjoJmHdwLHU/s1600/jgroups13.png)

A noter que si un membre destinataire se déconnecte du canal (dans le cas d'un arrêt brutal par exemple), alors l'objet RspList retourner par la méthode castMessage() contiendra une réponse indiquée comme en échec.

<a name="rpcDispatcher"></a>
##RpcDispatcher

La classe RpcDispatcher est dérivée de `MessageDispatcher` et permet donc au programmeur d'invoquer des méthodes distantes sur un ou tous les membres du groupe et, optionnellement, attendre leurs réponses. Cependant, contrairement à l'utilisation d'un `MessageDispatcher`, l'utilisation d'un `RpcDispatcher` est plus simple pour un besoin de type requête/réponse puisqu'il ne nécessite pas d'avoir à implémenter l'interface `RequestHandler` (et donc la méthode `handle()`).

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TQ42co-e7II/AAAAAAAAASA/arjovzck9cY/s1600/jgroups15.png)

En effet, il est possible d'indiquer lors de l'émission des requêtes (qui se fait à l'aide des méthodes `callRemoteMethod()` et `callRemoteMethods()`) quelles seront les méthodes (ainsi que leurs paramètres) qui seront invoqués lors de la réception des réponses.

Les autres paramètres de ces méthodes sont similaires à la méthode `castMessage()` de la classe `MessageDispatcher`.

De plus, il est à noter qu'il est possible d'utiliser les méthodes `callRemoteMethodWithFuture()` et `callRemoteMethodsWithFuture()` qui, comme leur nom l'indique, permettent de ne pas bloquer l'application lors de l'appel à des méthodes.

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/TQ44DbehK6I/AAAAAAAAASM/SEJXTcIvnZY/s1600/jgroups16.png)

<a name="replicatedHashMap"></a>
##ReplicatedHashMap
La classe `ReplicatedHashMap` a été écrite comme une classe de démonstration afin de montrer comme les états pouvaient être partagés entre les différents noeuds du cluster. Aussi, elle n'a pas été entièrement testée et n'a pas pour objectif d'être utilisée en production.

Une ReplicatedHashMap utilise une HashMap concurrente en interne et permet de créer différentes instances d'objets de type `HashMap` dans différents processus. Toutes ces instances possèdent exactement le même état. Lorsqu'une instance de `ReplicatedHashMap` est créée, un nom de groupe permet de déterminer quel groupe de `ReplicatedHashMap` est rejoint et la nouvelle instance interroge alors les autres membres existants pour connaitre l'état courant, le met à jour et se démarre.

Les modifications de la `ReplicatedHashMap` (méthodes `put()`, `clear()` ou `remove()`) sont propagées de manière ordonné dans tout le groupe alors que les lectures (méthode `get()`) sont faites sur la copie locale.

![medium](http://2.bp.blogspot.com/_XLL8sJPQ97g/TQ438y8jKhI/AAAAAAAAASI/QUvyWEKXEXY/s1600/jgroups17.png)

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TQ44U-8aDBI/AAAAAAAAASQ/8P6jK1PT8j8/s1600/jgroups18.png)

<a name="notificationBus"></a>
##NotificationBus

La classe `NotificationBus` permet d'émettre et de recevoir des notifications. Ainsi, cela permet à une application de maintenir un cache local répliqué avec toutes les autres instances. La classe `NotificationBus` s'appuie sur un canal en interne.

En fait, le bloc de construction NotificationBus s'apparente à JMS dans son mode _publish/subscribe_. Cependant, il peut également être utilisé pour faire du point à point en précisant l'adresse du destinataire.

![medium](http://1.bp.blogspot.com/_XLL8sJPQ97g/TQ44eSOwW3I/AAAAAAAAASU/l8hNwFLTXYU/s1600/jgroups19.png)

<a name="protocole"></a>
#Liste des protocoles

Ce paragraphe fournit à titre indicatif une liste non exhaustive de protocoles fournies par JGroups. Pour plus d'informations sur la configuration de chacun, il est conseillé de se référer à la documentation [officielle](http://www.jgroups.org/ug.html) et/ou au [wiki](http://community.jboss.org/wiki/JGroups) dont est issu ce paragraphe.

Catégorie | Nom | Description
----------|-----|-------------
Transport |	UDP	| UDP utilise le protocole multicast IP pour émettre des messages à tous les membres du groupe et des datagrams UDP pour les messages unicast (ie. à destination de membres particuliers). Quand il et démarré, il ouvre un socket multicast et unicast : le socket unicast est utilisé pour émettre et recevoir les messages unicast alors que le socket multicast pour les messages multicast. L'adresse du canal est l'adresse et le port du socket unicast.<br/><br/>Une pile de protocoles  utilisant UDP comme protocole de transport est généralement utilisé avec les groupes dont les membres s'exécute sur le même hôte ou de manière distribué sur le LAN. Il est nécessaire de s'assurer que le multicast IP est supporté entre les sous-réseau, ce qui n'est souvent pas le cas. Dans ce dernier cas de figure, il est préférable d'utiliser le protocole de transport TCP.
| TCP | TCP permet d'indiquer à JGroups qu'il doit s'appuyer sur le protocole de transport TCP pour émettre des messages à travers les membres du groupe. Dans ce cas, les membres du groupe créé un réseau de connexions TCP.
|TCP_NIO	| TCP_NIO est une implémentation de TCP ne bloquant pas les I/O. Cependant, il n'est pas conseillé de l'utiliser en production mais d'utiliser plutôt le protocole TCP.
| TUNNEL	| TUNNEL est un protocole de transport qui fonctionne en ouvrant une connexion TCP sur un routeur de message (tel que le Gossip Router). Tous les membres d'un groupe utilisant le protocole TUNNEL doivent être connectés au même routeur qui a pour rôle de transmettre les messages.<br/><br/>Il peut être utilisé dans le cas où des firewalls sont en place.
Découverte	| PING |	Le protocole PING permet de récupérer les membres du groupe. Il est utilisé pour détecter le coordinateur (le membre le plus vieux) soit en utilisant une requête PING multicast à l'adresse IP MCAST, soit en se connectant au GossipRouteur : chaque membre du groupe répond alors avec un paquet {C, A} où C est l'adresse du coordinateur et A est sa propre adresse. Cela permet de déterminer le coordinateur en fonction des réponses, permettant ainsi d'émettre une requête de type JOIN au coordinateur.<br/><br/>Si personne ne répond alors on considère qu'il s'agit du premier membre du groupe.<br/><br/>A noter que, contrairement à TCPPING, PING permet une découverte dynamique et n'a donc pas besoin d'avoir la connaissance des autres membres du groupe.
| TCPPING |	Le protocole TCPPING permet de récupérer les membres initiaux du groupe en contactant directement (ie. en point-à-point) les autres membres du groupe suite à la réception d'un message de type FIND_INITIAL_MBRS émit par le protocole GSM. Les réponses permettent de déterminer qui est le coordinateur du groupe.<br/><br/>A noter que, si le noeud courant est le serveur (ie. après avoir reçu un message de type BECOME_SERVER), alors il répondra aux requêtes de type TCPPING avec un message de type TCPPING.<br/><br/>TCPPING requière une configuration statique qui permet de savoir à l'avance où trouver les autres membre du groupe.<br/><br/>Pour permettre une découverte dynamique des autres membres avec une pile de protocole qui s'appuie sur TCP, il est possible d'utiliser soit le protocole MPING (qui utilise un découverte multicast), soit le protocole TCPGOSSIP (qui contacte le GossipRouteur pour obtenir les membres initiaux).
| TCPGOSSIP |	Le protocole TCPGOSSIP permet de récupérer les membres initiaux du groupe (utilisé par le protocole GMS et initié lors de la réception d'un message de type FIND_INITIAL_MBRS).<br/><br/>Cela s'effectue en contactant le ou les GossipRouteur qui doivent se trouver à des adresses/ports connus. Les réponses reçues doivent permettre de déterminer le coordinateur qui doit être contacté dans le cas où le noeud courant souhaite rejoindre le groupe.<br/><br/>A noter que, si le noeud courant est le serveur (ie. après avoir reçu un message de type BECOME_SERVER), alors il répondra aux requêtes de type TCPGOSSIP avec un message de type TCPGOSSIP.
| MPING |	MPING (pour Multicast PING) utilise les IP multicast pour déterminer les membres initiaux du groupe. Il peut être utilisé avec tous les types de transport mais il s'applique plus généralement avec le protocole TCP.<br/><br/>
| S3_PING |	S3_PING utilise le protocole S3 d'Amazon pour déterminer les membres du groupe. Il a été conçu spécialement pour les membres qui s'exécute sur Amazon EC2 qui ne supporte pas les échanges multicast.
Merge |	MERGE2 |	Les protocoles MERGE et MERGE2 permettent, dans le cas d'une scission du groupe (par exemple dans le cas d'un partitionnement réseau), de fusionner les sous-groupes en un seul. Il est seulement exécuté par le coordinateur du groupe (ie. le membre le plus ancien du groupe) qui rappelle périodiquement sa présence par un message muticast. Si un autre coordinateur du même groupe reçoit le message émis par un autre coordinateur, cela signifie qu'il doit engager un processus de fusion.<br/><br/>A noter que si une fusion des sous-groupe {A, B} et {C, D, E} a lieu et donne {A, B, C, D, E}, l'état n'est pas fusionner mais laisse à l'application le soin de le faire.
Détection d'échec |	FD	| Le protocole FD (pour Failure Detection) s'appuie sur des messages de heartbeat. Si aucune réponse n'est reçu dans un délai imparti, le membre est déclaré comme suspect et sera exclu par le protocole GSM.<br/><br/>Si le protocole FD_SOCK est utilisé, alors aucun message de heartbeat n'est émis mais un socket TCP est créé et un membre est déclaré mort quand le socket est fermé.
| FD_ALL |	FD_ALL permet la détection d'échec en s'appuyant sur un simple protocole de heartbeat : chaque membre diffuse (multicast) périodiquement un message de type heartbeat. Chaque membre devant maintenir la table de tous les membres, dès qu'un message n'est pas reçu d'un des membres dans un délai imparti, il est considéré comme suspect.
| FD_PING |	FD_PING utilise un script ou une commande qui reçoit comme paramètre l'hôte à pinger et qui doit renvoyer l'entier 0 en cas de succès et 1 sinon.<br/><br/>A noter que la commande par défaut est /sbin/ping (resp. ping.exe).
| FD_ICMPL |	FD_ICMPL utilise en interne la méthode InetAddress.isReachable() afin de déterminer si l'hôte donné est présent ou pas.<br/><br/>A noter qu'il est préférable, avant d'utiliser ce protocole, de s'assurer au préalable du mode de fonctionnement de cette méthode pour le système d'exploitation cible.
| FD_SOCK |	Le protocole FD_SOCK s'appuie sur une topologie en anneau au niveau des sockets TCP créé entre les membres du groupe.<br/><br/>Ainsi, par exemple, le membre B est suspecté par son voisin A s'il observe une fermeture du socket TCP. Par contre, c'est au membre B de prévenir son voisin dans le cas d'un départ du groupe volontaire.<br/><br/>Cependant, dans le cas de l'interruption d'un serveur et/ou arrêt brutal d'un switch (et donc une non fermeture du socket TCP), ce protocole est inefficace. Aussi, il est conseillé de l'utiliser conjointement avec le protocole de FD (qui s'appuie sur un fonctionnement en heartbeat). 
| VERIFY_SUSPECT |	Le protocole VERIFY_SUSPECT vérifie l'état d'un membre suspect en lui envoyant un ping.<br/><br/>A noter qu'il doit être utilisé entre les couches FD et GMS sur la pile de protocole. 
Messages fiables |	pbcast.NAKACK |	Le protocole pbcast.NAKACK permet de s'assurer de la fiabilité de réception des messages en utilisant une file de message de type FIFO conjointement à des ack.<br/><br/>Ainsi, par exemple, si le noeud a reçu les messages P:1, P:3, P:4, alors il demandera une retransmission du message 2.
| pbcast.STABLE |	Le protocole pbcast.STABLE permet de calculer les messages émis qui sont dans un état stable, c'est-à-dire qui ont été reçus par tous les membres. Cela permet au protocole NAKACK de supprimer les messages qui sont considérés comme ayant été reçu par tous les membres du groupe. 
| UNICAST |	Le protocole UNICAST fournit un protocole unicast fiable d'émission en mode FIFO. 
Fragmentation | 	FRAG |	Le protocole FRAG permet de fragmenter les messages trop gros lors de l'émission et de les ré-assembler lors de la réception.<br/><br/>Ce protocole fonctionne aussi bien pour les messages émis en multicast ou en unicast.<br/><br/>Il sérialise chaque message afin de le morceler, et cela, quel que soit son type (même s'il s'agit d'octets) afin de pouvoir prendre en compte avec précision la taille de l'entête et les propriétés du message.
| FRAG2 |	Le protocole FRAG2 permet, tout comme le protocole FRAG, de fragmenter les messages.<br/><br/>Cependant, il offre un autre algorithme lui permettant de ne pas avoir à sérialiser les messages pour les découper.<br/><br/>A noter que le protocole FRAG2 est préférable au protocole FRAG. 
Gestion des membres du groupe | 	pbcast.GMS	| Le protocole pbcast.GMS permet de gérer le départ ou l'enregistrement des membres dans le groupe. Il a également à sa charge de gérer les membres suspects et de les exclure du groupe. En fait, c'est lui qui émet les vues à tous les autres membres du groupe quand un changement se produit.
| VIEW_SYNC |	Le protocole VIEW_SYNC permet aux membres du groupe de s'échanger périodiquement leurs vues et, si une incohérence entre vue est constatée, alors cela permet au protocole pbcast.GMS de mettre à jour la vue "officielle". En fait, ce protocole est le pendant du coordinateur mais au niveau des noeuds et garanti, en cas d'anomalie sur le coordinateur pendant une dissémination des vues, une consistance des vues.
Contrôle de flux |	FC |	Le protocole FC (Flow Control) permet de gérer le flux des messages. En effet, si l'émetteur émet plus vite que les destinataires ne peuvent consommer les messages, alors il y a risque de contension et c'est pourquoi les émetteurs ne doivent pas émettre les messages plus vite que leur capacité à être absorbés par le système.<br/><br/>Pour ce faire, un émetteur possède N crédits pour chaque membres. Quand il émet un message, il décrémente le nombre d'octets émis au crédit du récepteur. Quand le crédit d'un récepteur est inférieur que ce que requière le message à émettre, l"émetteur se bloque et attend que l'émetteur dispose de plus de crédits.<br/><br/>Le récepteur décrémente également le nombre d'octets reçu et émet son nombre de crédit à l'émetteur quand il est en dessous d'un certain seuil.
| SFC |	Le protocole SFC correspond à une version simplifiée du protocole FC qui peut entraîner une surconsommation chez le récepteur lorsqu'il existe de la latence dans la couche de transport.<br/><br/>En fait, ce protocole s'appuie sur un nombre d'octet maximal de crédits (configurable) qui est vérifié coté émetteur et récepteur. Il se marrie très bien avec le protocole pbcast.STABLE qui permet de "nettoyer" les messages déjà traités.
Transfert d'état |	pbcast.STATE_TRANSFER	 | Le protocole pbcast.STATE_TRANSFER permet à un nouveau membre de récupérer l'état du groupe du coordinateur.<br/><br/>Il fonctionne comme suit :<br/><ul><li>le nouveau membre demande au coordinateur l'état</li><li>Le coordinateur vérifie deux choses : un digest et l'état de l'application, le digest étant un vecteur contenant le plus haut et le plus bas numéro de séquence des messages reçu pour chaque membre</li><li>Les messages dans le digest font déjà parti de l'état du groupe et ne doivent pas être reçus</li><li>Le coordinateur émet l'état et le digest au membre arrivant</li><li>Le membre arrivant positionne son état et positionne son digest avec celui reçu permettant ainsi de savoir les messages qui n'ont pas encore été reçu</li></ul>
| pbcast.STREAMING_STATE_TRANSFER |	Le protocole pbcast.STREAMING_STATE_TRANSFER permet de pallier aux problèmes de gros messages non traités par le protocole pbcast.STATE_TRANSFER.<br/><br/>Il permet de gérer des états de plus d'un gigaoctet.
Synchronisation virtuelle	| pbcast.FLUSH |	Le protocole FLUSH, comme son nom l'indique, force les membres du groupe à flusher leurs messages en cours plutôt que de les bloquer.<br/><br/>Il peut être utilisé dans les cas suivants :<br/><ul><li>Transfert d'état ; quand un membre demande l'état, le coordinateur indique à tous les membres d'arrêter d'émettre des message et attends leur réponse. Lorsque le nouveau membre a reçu toutes les informations requises, le coordinateur indique alors à tous les membres qu'ils peuvent reprendre leurs activités</li><li>Changement du vue (c'est-à-dire opération de join) : avant d'installer une nouvelle vue, l'opération de flush permet de s'assurer que tous les messages émis dans la vue précédente ont été traités</li></ul>
Authentification et cryptage |	AUTH |	Le protocole AUTH est utilisé pour le couche d'authentification de JGroups en permettant de déterminer si un noeud est autorisé à rejoindre un groupe. AUTH s'appuie sur le protocole GMS et écoute les messages de type JOIN REQUEST : quand un message de type JOIN REQUEST est reçu, il cherche l'objet AuthHeader dans lequel doit se trouver une implémentation de la classe AuthToken.<br/><br/>AuthToken est une classe abstraite dont les implémentations doivent fournir le mécanisme d'authentification. JGroups fournit des implémentations basiques telles que :<br/><ul><li>SimpleToken,</li><li>MD5Token</li><li>et X50Token.</li></ul><br/><br/>Ces implémentations encrypte un chaîne de caractère se trouvant dans la configuration de JGroups et la passe au message JOIN REQUEST.<br/><br/>Quand l'authentification est réussie, le message est passé à la pile qui s'occupe du protocole GMS, sinon, un message de type JOIN RESPONSE contenant un message d'erreur est renvoyé au noeud. Le client lève alors une exception de type SecurityException.
| ENCRYPT |	Le protocole ENCRYPT permet de crypter les messages. Par défaut, il n'encrypte que le corps du message (ie. que tous les entêtes, les adresses des destinataires et de la source ne sont pas cryptés par défaut).
Synchronisation |	BARRIER |	Le protocole BARRIER peut être utilisé pour suspendre les messages à émettre. 

<a name="conclusion"></a>
#Conclusion

Comme nous avons pu le voir, JGroups présente une API pour le développeur qui est très agréable à utiliser.
Cependant, la configuration des canaux peut s'avérer très ardue car elle est demande une bonne connaissance de la couche réseau mais aussi de ce qui est ciblé :

* Est-il acceptable de faire du multicast IP sur notre réseau? A défaut, l'utilisation de TCP est-elle suffisante?
* Faut-il crypter les messages?
* Faut-il que les messages émis sur le canal X soient ordonnancés?
* Comment veut-on s'assurer que les messages soient bien transmis?
* ...

Pour moi, c'est ce type de questions qu'il est important de se poser afin de configurer les différents canaux de JGroups. En outre, par défaut, JGroups peut s'avérer très bavard (notion de ping, signalisation en interne sur un merge, ...) et, là encore, plutôt que de faire hurler votre administrateur réseau, essayer de voir avec lui ce qui est acceptable.

Ainsi JGroups s'adresse aux personnes qui veulent avoir la main sur les couches basses (contrairement à JMS qui a tendance à être plus abstrait sur les couches basses puisque cela dépend du provider choisi).

#Pour aller plus loin...

* Site officiel de JGroups : http://www.jgroups.org/
* Wiki de JGroups : http://community.jboss.org/wiki/JGroups
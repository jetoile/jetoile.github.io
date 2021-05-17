---
title: "Petits retours sur Cassandra"
date: 2012-03-13 10:19:01 +0100
comments: true
tags: 
- architecture
- java
- cassandra
- réflexion
- nosql
---
![left-small](http://2.bp.blogspot.com/-IvTZebKJhAc/T1hsV7nG8nI/AAAAAAAAAjw/qFiNLqvXgnY/s1600/cassandra01.png)

Suite à de nombreuses présentations de Cassandra (faites, entre autre, par [Michaël Figuière](http://twitter.com/mfiguiere)) et à une opportunité de regarder plus précisément ce qui se cachait réellement derrière cette implémentation d'une solution de la famille des produits NoSQL orienté colonnes, je vais, dans cet article, tenter de décrire ce que j'ai aimé et ce que je n'ai pas aimé sur [Apache Cassandra](http://cassandra.apache.org/).

Je tiens toutefois à préciser que je n'ai aucune expérience réelle sur le produit et que je ne m'appuierai donc que sur sa [documentation officielle](http://www.datastax.com/doc-source/pdf/cassandra10.pdf) en version 1.0 qui date du 2 mars 2012.

En outre, je ne m'attarderai que sur les points qui m'ont semblé intéressants et marquants.

Pour ceux qui sont coutumiés de ce blog, je ne changerai pas mes habitudes et je me contenterai seulement de traduire de manière libre les passages qui m'ont intéressés ;-). A noter que de nombreux points sont rappelés à différents endroits dans ce document mais c'est également le cas dans la documentation officielle.

<!-- more -->

#Introduction et concepts

Apache Cassandra est un système permettant de gérer une grande quantité de données de manière distribué. Ces dernières peuvent être structurées, semi structurées ou non.

En fait, Cassandra est été conçu pour être hautement scalable sur un grand nombre de serveurs tout en ne présentant pas de _Single Point Of Failure_.

Enfin, Cassandra fourni un schéma de données dynamique afin d'offrir un maximum de flexibilité et de performance.
Derrière ce jolie discour se cachent quelques termes qu'il est important de connaitre afin de mieux appréhender les différents concepts manipulés par Cassandra :

* __Keyspace__ : c'est l'équivalent d'une database dans le monde des bases de données relationnel. A noter qu'il est possible d'avoir plusieurs Keyspaces sur un même serveur. 
* __Une famille de colonnes__ : c'est l'objet principal de données et peut être assimilé à une table dans le monde des bases de données relationnel.

#Architecture de Cassandra

Une instance Cassandra est un collection de noeuds indépendants qui sont configurés ensembles pour former un cluster.

Dans un cluster, tous les noeuds sont égaux, ce qui signifie qu'il n'y a pas de noeud maître ou un processus centralisant leur gestion.

En fait, Cassandra utilise un protocole appelé __Gossip__ afin de découvrir la localisation et les information sur l'état des autres noeuds du cluster. Le protocole __Gossip__ est un protocole de communication de type _peer-to-peer_ dans lequel les noeuds échangent périodiquement des informations sur leur état mais également sur ce qu'ils savent des autres noeuds.

Pour être plus précis, le processus s'exécute toutes les secondes afin d'échanger les messages avec au plus trois autres noeuds du cluster. De plus, une version est associée à ces messages afin de permettre d'écraser les informations plus anciennes.

Ainsi, quand un noeud démarre, il regarde, dans un premier temps, son fichier de configuration qui le renseigne sur le nom de son cluster ainsi que les autres noeuds compris dedans. Cela lui permet de les contacter afin de récupérer leur état.

Cependant, afin d'éviter un partitionnement, tous les noeuds du cluster doivent disposer de la même liste de noeuds dans leurs fichiers de configuration.

La détection des échecs est une méthode pour déterminer localement si un autre noeud est accessible ou pas. En outre, les informations récoltées par ce mécanisme permet à Cassandra d'éviter d'émettre des requêtes aux noeuds qui ne sont plus accessibles.

En fait, ce mécanisme fonctionne sur le principe de _heartbeat_ soit de manière directe (ie. en récoltant les informations directement des noeuds) soit de manière indirecte (ie. en récoltant les informations par l'intermédiaire de la connaissance des autres noeuds - par transitivité quoi... ;-) - ).

Je ne décrirai pas plus précisément cette partie si ce n'est une petite précision. En fait, lorsqu'un noeud est déclaré comme inaccessible, les autres noeuds stockent messages susceptibles d'avoir été manqué par ce noeud. Cependant, il peut arriver qu'entre le moment où le noeud devient inaccessible et le moment où sa perte est détectée, un laps de temps s'écoule et qu'ainsi, les réplicas ne soient pas conservés. En outre, si le noeud vient à être indisponible pendant une période trop importante (par défaut, une heure), alors les messages ne sont plus stockés. C'est pour cette raison qu'il est conseillé d'exécuter régulièrement l'outils de réparation des données.

#Partitionnement des données avec Cassandra

Concernant cette partie, je ne m'étendrai pas dessus puisque la documentation est suffisante à elle-même.

Cependant, juste un point qui m'a chagriné.

En effet, il est possible de configurer le partitionnement pour une famille de colonnes en précisant que l'on veut que cela soit géré avec une stratégie de type __Ordered Partitioners__.

Ce mode peut, en effet, avoir un intérêt si l'on souhaite récupérer une plage de lignes comprise entre deux valeurs (chose qui n'est pas possible si le _hash MD5_ des clés des lignes est utilisé).

Cependant, il est conseillé d'utiliser plutôt une clé deuxième clé d'indexation positionnée sur la colonne contenant les informations voulues. En effet, utiliser la stratégie __Ordered Partitionners__ a les conséquences suivantes :

* L'écriture séquentielle peut entraîner des _hotspots_ : si l'application tente d'écrire ou de mettre à jour un ensemble séquentiel de lignes, alors l'écriture ne sera pas distribué dans le cluster.
* Un _overhead_ accru pour l'administration du _load balancer_ dans le cluster : les administrateurs doivent calculer manuellement les plages de jetons afin de les répartir dans le cluster.
* Répartition inégale de charge pour des familles de colonnes multiples.

#La répartition dans Cassandra

La réplication est le processus permettant de stocker des copies des données sur de multiples noeuds afin de permettre leur fiabilité et la tolérance à la panne. Quand un __keyspace__ est créé dans Cassandra, il lui est affecté la stratégie de distribution des réplicas, c'est à dire le nombre de réplicas et la manière dont ils sont répliqués dans le cluster. La stratégie de réplication repose sur la configuration du cluster snitch afin de déterminer la localisation physique des noeuds ainsi que leur proximité par rapport aux autres.

Il est souvent fait référence au facteur de réplication (__replication factor__) pour parler du nombre total de réplicas dans le cluster.

Aussi :

* Un facteur de réplication de 1 signifie qu'il n'y a qu'une seule copie de chaque ligne.
* Un facteur de réplication de 2 signifie qu'il existe deux copie de chaque ligne.
* ...

Tous les réplicas possèdent la même importance : il n'y a pas de réplicas principaux ou maîtres du point de vu de la lecture et de la lecture.

Ainsi, la règle générale est que le facteur de réplication ne doit pas être supérieur au nombre de noeuds dans le cluster. Cependant, il est possible d'augmenter le facteur de réplication puis d'ajouter le nombre de noeuds désiré à postériori. A noter toutefois que lorsque le facteur de réplication est supérieur au nombre de noeuds, alors les écritures ne sont plus prises en compte tandis que l'opération de lecture reste viable tant que le degrés de consistance est respecté.

Concernant les stratégies de distribution des réplicas, cela permet de jouer sur la façon dont les réplicas sont répartis dans le cluster pour un keyspace. Cette stratégie est à renseigner lors de la création de la __keyspace__.

De plus, il est possible de configurer la manière dont les noeuds sont groupés ensemble dans la topology du réseau global. Cet élément de configuration correspond au snitch et est associé au cluster. Cassandra utilise alors cette information pour router les requêtes entre les noeuds.

Je n'en dirai pas plus pour cette partie mais plus d'informations sont accessibles dans la documentation officielle.

#Cassandra du point de vue de l'application Client

Tous les noeuds de Cassandra sont égaux. Ainsi, une demande de lecture ou d'écrire peut interroger indifféremment n'importe quel noeud du cluster. Quand un client se connecte à un noeud et demande une opération d'écriture ou de lecture, le noeud courant sert de coordinateur du point de vue du client.

Le travail du coordinateur est de se comporter comme un proxy entre le client de l'application et les noeuds qui possèdent la données. C'est lui qui a la charge de déterminer quels noeuds de l'anneau devra recevoir la requête.

![medium](http://4.bp.blogspot.com/-Y_ErJ1Xzjwk/T1jQ9lh4oDI/AAAAAAAAAj4/Y_MOdH2_2mk/s1600/cassandra02.png "crédit photo: Cassandra")

Concernant les requêtes d'écriture, le coordinateur émet la requête à tous les réplicas qui possèdent la ligne à modifier. Aussi longtemps qu'il sont disponibles, ils reçoivent les demandes d'écriture quelque soit le niveau de consistance demandé par le client. Le niveau de consistance d'écriture détermine le nombre de noeuds qui doivent acquitter l'écriture afin de considérer l'écriture comme ayant réussi.

Il est à noter que, dans le cas où il existe plusieurs _data center deployments_, Cassandra optimise les performances d'écriture en choisissant un noeud coordinateur dans chaque _data center_ distant afin de traiter les requêtes des réplica dans le data center. Le noeud coordinateur contacté par l'application cliente n'a alors qu'à transmettre les requêtes d'écriture à un seul noeud de chaque _data center_ distant.

Si le niveau de consistance choisi est __ONE__ ou __LOCAL_QUORUM__, alors seuls les noeuds du même data center que le noeud coordinateur doivent acquitter l'écriture.

![medium](http://1.bp.blogspot.com/-_TFMEXBbdSo/T1jRBazh_wI/AAAAAAAAAkA/7uBwOjhPb5s/s1600/cassandra03.png "crédit photo: Cassandra")

Concernant la lecture, il y a deux types de requêtes de lecture qu'un coordinateur peut émettre à un réplica :

* Une requête de lecture directe. Dans ce cas, le nombre de réplicats contactés par une demande de lecture directe est déterminé par le niveau de consistance spécifié par le client,
* Une requête de réparation de lecture en tâche de fond. Dans ce cas, elle est envoyée à tous les réplicas additionels qui n'ont pas reçu de requête directe. Ce type de requête permet de vérifier que la ligne est consistante par rapport aux autres réplicas.

Ainsi, dans un premier temps, le coordinateur contacte les réplicas en fonction du niveau de consistance qui a été spécifié. Ces réplicas sont choisis en fonction de leurs capacités à répondre rapidement. Ces derniers répondent avec la donnée demandé et, s'il existe plusieurs réponses, elles sont comparées en mémoire afin de vérifier leur consistance. Si ce n'est pas le cas, alors c'est le réplica qui est le plus récent (en se basant sur le timestamp) qui est utilisé par le coordinateur pour répondre au client.

En outre, afin de s'assurer que tous les réplicas ont la version la plus récente, le coordinateur les contacte en tâche de fond et compare la donnée de tous les réplicas restant qui possède la ligne. Il demande ensuite, éventuellement, une opération d'écriture afin de mettre à jour la donnée. C'est cette opération qui est appelé __read repair__ et qui peut être configuré par famille de colonne (par défaut, elle est désactivée).

![medium](http://4.bp.blogspot.com/-uK_LgOrv6w8/T1jUQ2yvdDI/AAAAAAAAAkI/GQewQ8NmdTc/s1600/cassandra04.png "crédit photo: Cassandra")

#Le modèle de données de Cassandra

Le modèle de données de Cassandra s'appuie sur un schéma dynamique, avec un modèle de données orienté colonne.
Cela signifie que, contrairement à une base de données relationelle, il n'est pas nécessaire de modéliser toutes les colonnes puisqu'une ligne n'a, potentiellement, pas le même ensemble de colonnes.

Les colonnes et leurs méta données peuvent être ajoutées par l'application lorsque cela s'avère nécessaire.
En fait, le modèle de données de Cassandra a été pensé pour répondre à des problématiques de données distribuées et diffère complètement d'un modèle classique de base de données relationnel où les données sont stockées dans des tables qui sont, dans la plupart des cas, en relation entre elles. En outre, dans un modèle relationnel, les données sont généralement normalisées afin d'éviter la redondance et des jointures sont faites entre les tables sur des clés communes afin de satisfaire les requêtes.

Dans Cassandra, le __Keyspace__ est le conteneur des données de l'application (un peu comme une database ou un schéma pour une base de données relationnel). Dans ces keyspaces se trouvent une ou plusieurs familles de colonnes (qui correspondent aux tables en base de données relationnelle).

Ces familles de colonnes contiennent des colonnes ainsi qu'un ensemble de colonnes connexes qui sont identifiées par un clé de ligne. En outre, chaque ligne d'une famille de colonnes ne dispose pas nécessairement des mêmes colonnes qu'une autre ligne.

Enfin, Cassandra n'impose pas de relations entre les familles de colonnes au sens base de données relationnelle : il n'y a pas de clés étrangères et les jointures entre familles de colonnes ne sont pas supportées.

Ainsi, pour faire simple, chaque famille de données dispose de son ensemble de colonnes qui sont destinées à être consulté ensemble pour satisfaire les requêtes spécifiques à l'application. 

En outre, les familles de colonnes peuvent définir les méta données des colonnes mais pour une colonne donnée correspondant à une ligne, cela est à la charge de l'application. De plus, chaque ligne peut disposer d'un ensemble de colonnes différentes.

Par contre, bien que les familles de colonnes puissent être flexibles, dans la pratique, il est conseillé d'y associer une sorte de schéma (à une famille de colonnes, il est préférable de n'y mettre qu'un même type de données).

En fait, il y a deux types de familles de colonnes :

* __Les familles de colonnes statiques__ : elles utilisent un ensemble statique de nom de colonnes et sont très similaires à une table d'une base de données relationnel même si toutes les colonnes n'ont pas à être obligatoirement renseignées. A noter, qu'en général, les méta données des colonnes sont définies, dans ce cas, pour chaque colonne.
* __Les familles de colonnes dynamiques__ : elles permettent, par exemple, de pré calculer un ensemble de résultats et de les stocker dans une même ligne afin de pouvoir y accéder plus tard. Chaque ligne correspond donc à un snapshot de données correspondant à un requête donnée. A noter que, pour une famille de colonnes dynamique, plutôt que d'avoir des méta données pour chaque colonne, le type des informations par colonnes (tels que le comparateur et les validateurs) est défini par l'application lorsque la donnée est insérée.

Pour toutes les familles de colonnes, chaque ligne est identifiée de manière unique par sa clé (un peu comme la notion de clé primaire pour les bases de données relationnelles). Une famille de colonnes est, quant à elle, toujours partitionnée par ses clés de ligne qui sont, implicitement, indexées.

En fait,  dans Cassandra, une colonne est le plus petit incrément de données. Elle est modélisée par un tuple qui contient le nom, la valeur et un _timestamp_. Ce timestamp est utilisé par Cassandra pour déterminer la mise à jour la plus récente dans la colonne (et, dans ce cas, c'est la plus récente qui est prioritaire lors d'une requête). 

![center](http://3.bp.blogspot.com/-a0aThYoPwxU/T1nHhEkkeiI/AAAAAAAAAkQ/N5n7HZJ1Gb8/s1600/cassandra05.png "crédit photo: Cassandra")

Une colonne doit avoir un nom qui peut être un label statique (comme "nom" ou "email") ou peut être mis dynamiquement lorsque la colonne est créée par l'application.

Les colonnes peuvent être indexées par leurs noms (en utilisant un second index).

Cependant, une limitation de l'indexation des colonnes est le non support des requêtes qui nécessitent un ordonnancement comme cela peut être le cas avec une collection de données de temps. En effet, dans ce cas, un second index sur la colonne de timestamp ne serait pas suffisant car il n'est pas possible d'avoir la main sur l'ordonnancement de la colonne avec un index secondaire. Pour palier à ce manque, il est toujours possible de maintenir manuellement une famille de colonnes qui ferait office d'index.

Il est à noter qu'il n'est pas obligatoire pour une colonne d'avoir une valeur puisque, parfois, le nom de cette dernière lui suffit à elle-même.

En plus des colonnes dites "classiques", Cassandra offre 3 autres types de colonnes (que je détaillerai pas plus) :

* __Expiring Columns__ qui disposent d'un TTL (_Time To Live_).
* __Counter Columns__ qui permettent de stocker un compteur.

![center](http://4.bp.blogspot.com/-_NU2ZTQ6yN0/T1201ywQkBI/AAAAAAAAAkY/zfvptCJ7dxM/s1600/cassandra06.png)

* __Super Columns__ qui permettent de gérer une meta colonne pouvant contenir d'autres colonnes

![center](http://3.bp.blogspot.com/-RyNcDkPN5og/T1209KAYDHI/AAAAAAAAAkg/K42iHZ7csaU/s1600/cassandra07.png)

Concernant, le type de données exploitables par Cassandra, il existe deux notions :

* Le __validator__ qui est le type de données pour la valeur d'une colonne (ou la clé d'une ligne).
* Le __comparator__ qui est le type de données pour le nom d'une colonne.

En fait, il est possible de définir un type de données lors de la création d'une famille de colonnes et que son schéma est précisé (cf. les familles de colonnes statiques). Cassandra stocke alors, par défaut, le nom et les valeurs des colonnes en tableau d'hexadécimal (`BytesType`).

Ci-dessous, la liste des valeurs possibles pour les _validator_ et _comparator_ (sauf pour le type `CounterColumnType` qui ne peut être utilisé que comme valeur de colonne) :
![medium](http://2.bp.blogspot.com/-ZwnUrailUWs/T13LTaRvLBI/AAAAAAAAAko/Ul9Erfto4gI/s1600/cassandra08.png)

Plus précisément sur les validator, pour toutes les familles de colonnes, il est conseillé de définir un _validator_ par défaut pour une clé de ligne (qui peut être modifié ou ajouté au besoin). Dans ce cas, une vérification est effectué lors de l'insertion ou la mise à jour de données.

Concernant le _comparator_, il est à noter que, dans une ligne, les colonnes sont toujours stockées de manière ordonné par rapport au nom des colonnes. Le comparator précise le type de données pour le nom d'une colonne mais ne peut être modifié.

#L'indexation dans Cassandra

Un index est une structure de données qui permet un accès rapide ainsi qu'une recherche de données par rapport à un ensemble de critères donnés.

Dans les bases de données classique, une clé primaire est une clé unique utilisée pour identifier chaque ligne d'une table. Cela permet, comme tous les index, d'accélérer l'accès aux données dans une table. La clé primaire garantie également l'unicité et mais peut également se charger de l'ordonnancement des données.

Dans Cassandra, l'index primaire d'une famille de colonnes correspond à l'index de ses clés de lignes. Puisque chaque noeud connait la plage de ses clés par noeuds gérés, les requêtes sur les lignes sont plus aisées à localiser en scannant l'index des lignes sur un noeud donnée.

Avec un partitionnement aléatoire des clés de lignes (qui est la configuration par défaut), les clé de lignes sont partitionnés par le _hash MD5_ et ne peuvent donc pas gérer l'ordonnancement (au sens ordre des données). Il est à noter qu'utiliser un partitionnement ordonné n'est pas conseillé puisque cela implique une maintenance accrue pour la distribution des données sur les noeuds.

Cassandra propose également un index secondaire qui se fait sur la valeur des colonnes. Il permet d'effectuer des opérations disposant de prédicats d'égalité (par exemple,  èèwhere column x = value y`). Ainsi, les requêtes sur des valeurs indexées peuvent appliquer une sorte de filtres.

#L'accès aux données dans Cassandra

##L'écriture

Cassandra est optimisé pour permettre une écriture rapide et hautement disponible des données avec un débit élevé. Pour ce faire, les données sont écrites dans un premier temps dans un __journal de commit__ (pour s'assurer de la durabilité), puis dans une structure en mémoire appelé la __Memtable__. Une écriture est considérée comme ayant réussi si les deux actions précédentes ont réussis. Cela permet d'avoir peu d'entrée/sortie disque au moment de l'écriture. Ainsi, les données sont mises en mémoire et périodiquement écrite sur le disque dans une structure appelé une __SSTable__ (pour _Sorted String Table_). Les __Memtables__ et __SSTables__ sont maintenus par famille de colonnes : les __Memtables__ sont organisées de manière ordonné par clé de ligne et sont flushé dans les __SSTables__ séquentiellement.

A noter également que les __SSTables__ sont immuables, ce qui signifie qu'une ligne peut, typiquement, être stockée dans plusieurs fichiers de __SSTable__. Ainsi, pour répondre à une requête de lecture, il est nécessaire de combiner toutes les __SSTables__ se trouvant sur le disque pour trouver la valeur d'une ligne. Pour optimiser ce processus, Cassandra utilise un structure en mémoire appelé __Bloom Filter__ : chaque __SSTable__ dispose de son propre __Bloom Filter__ qui est utilisé pour vérifier si la clé d'une ligne existe lors d'une requête.

De plus, en tâche de fond, Cassandra merge périodiquement les __SSTables__ ensembles dans un __SSTables__ d'une taille supérieure. Ce processus est appelé __Compaction__. Plus précisément, il regroupe les fragments de lignes ensemble, supprime les colonnes qui ont été marquées comme devant être supprimé et reconstruit les index primaires et secondaires. Bien sûr, ce processus a des impacts sur les _disk I/O_.

Contrairement aux bases de données relationnelles classique, Cassandra n'offre pas des transactions dites ACID : il n'y a pas de locks ou de dépendances transactionnelles lors d'une mise à jour concurrente de multiples lignes ou de familles de colonnes.

Pour rappel, ACID est l'acronyme de :

* __Atomic__ : tout ce qu'il y a dans une transaction doit être réussi ou sinon un rollback est effectué.
* __Consistent__ : une transaction ne peut pas laisser la base de données dans un état incohérent.
* __Isolated__ : les transactions ne peuvent pas interagir les unes avec les autres.
* __Durable__ : les transactions effectuées persistent même après une  défaillance du serveur.

Pour sa part, Cassandra ne traite que les concepts d'isolation et d'atomicité. En fait, pour être plus précis, une écriture est atomique au niveau des lignes (ie. l'insertion ou la mise à jour des colonnes d'une ligne donnée est traitée comme une écriture unique). Ainsi, Cassandra ne supporte pas les transactions dans le sens de mise à jour de lignes multiples. De même, il n'y a pas de rollback lorsque l'écriture réussi sur un noeud mais échoue sur les autres (dans ce cas, il n'est possible de n'avoir qu'un rapport d'erreur mais pas de rollback).

Cassandra utilise les _timestamps_ pour déterminer la mise à jour la plus récente d'une colonne ; ce _timestamp_ étant fourni par l'application cliente. Ainsi, la donnée avec le dernier timestamp est toujours celle qui prévaut lors d'une requête. C'est pour cela que, si de multiples clients mettent à jour la même donnée de manière concurrente, alors c'est la donnée la plus récente qui sera persistée.

Du coté de la durabilité, toutes les écritures sur un réplica sont enregistrés à la fois en mémoire et dans le journal de commit avant que l'acquittement ne soit émis. Si un échec se produit avant l'écriture sur disque, alors le journal de commit est rejoué du début afin de récupérer les données qui n'ont pas encore été écrites.

##L'insertion et la mise à jour

Plusieurs colonnes peuvent insérées en même temps. Ainsi, lorsque des colonnes sont insérées ou mise à jour dans une famille de colonnes, l'application cliente précise la clé de ligne pour identifier l'enregistrement à mettre à jour. La clé de la ligne peut donc être vu come une clé primaire puisqu'elle doit être unique pour chaque ligne dans une famille de colonnes données. Cependant, à la différence d'une clé primaire, il est possible d'insérer des données à une clé primaire déjà présente, mais dans ce cas, Cassandra répondra comme une mise à jour (si elle n'existe pas, elle sera créé).

De plus, les colonnes sont écrasées si le _timestamp_ d'une autre version de colonne est plus récente. Il est, cependant, à noter que le timestamp est fourni par le client. Aussi, ces derniers doivent disposer d'une horloge synchronisée par NTP (_Network Time Protocol_).

##La suppression

Lorsqu'une ligne ou une colonne est supprimé dans Cassandra, il est important de comprendre les points ci-dessous :

* la suppression de la donnée du disque n'est pas immédiate (pour rappel, une données insérée est écrite dans le SSTable qui se trouve sur le disque). En effet, le __SSTable__ étant immuable, un marqueur appelé __tombstone__ est écrit pour renseigner le nouveau status de la colonne. Les colonnes ainsi marquées persistent pendant un certain laps de temps configurable puis sont réellement supprimées lors du processus de __compaction__.
* une colonne supprimé peut réapparaitre si la routine de réparation de noeud n'est pas exécuté. En effet, marquer une colonne d'un tombstone implique qu'un réplica qui était dans un état non atteignable lors de la suppression ne recevra l'information que lorsqu'il sera de nouveau dans un état joignable. Cependant, si un noeud est injoignable plus longtemps que le temps configuré de conservation des tombstones, alors le noeud peut complètement manquer la suppression et répliquer les données supprimées lors de son retour dans le cluster. 
* La clé de ligne pour une ligne supprimée apparait toujours dans la plage de résultats d'une requête. En effet, quand une ligne est supprimée dans Cassandra, ses colonnes correspondantes sont marquées par un __tombstone__. Ainsi, tant que les données marquées par un tombstone ne sont pas réellement supprimées par le processus de __compaction__, la ligne reste vide (ie. sans colonne associées).

##La lecture

Pour rappel, lorsqu'une requête de lecture d'une ligne est reçue par un noeud, la ligne doit être combinée de toutes les __SSTables__ du noeud qui contiennent les colonnes de la ligne en question ainsi que de toutes les Memtables. Pour optimiser ce processus, Cassandra utilise une structure en mémoire appelée les __bloom filter__ : chaque __SSTable__ dispose d'un __bloom filter__ associé qui est utilisé pour vérifier s'il existe des données dans la ligne avant de faire une recherche (et ainsi de faire des entrées/sorties disque).

#La consistance des données dans Cassandra

Lorsqu'il est fait question de consistance dans Cassandra, cela s'applique à la façon dont est maintenue la cohérence des données et comment les lignes de données sont synchronisées entre tous les réplicas.

Cassandra étend le concept de cohérence éventuelle en offrant un consistance réglable.

Ainsi, pour toutes opérations de lecture ou d'écriture, l'application cliente peut décider du degrés de consistance que doit avoir la requête. En plus de cela, Cassandra dispose d'un mécanisme de réparation afin de s'assurer de la consistance des données entre les différents réplicas.

##Consistance réglable pour les requêtes clientes

Le niveau de consistance dans Cassandra peut être associé à n'importe quelle requête de lecture ou d'écriture. Cela permet aux développeurs de l'application de trouver le juste milieu entre le temps d'exécution de leurs requêtes et la consistance qu'ils souhaitent avoir sur les résultats obtenus.

Lors d'une écriture, le niveau de consistance précise sur combien de réplicas doit être écrite la donnée avant d'être acquitté du succès de l'opération.

Les niveaux de consistance disponibles sont les suivants :

![medium](http://1.bp.blogspot.com/-x5hTsbj3l_g/T18dPz_XiHI/AAAAAAAAAkw/CKBq0LoZ6pE/s1600/cassandra11.png)

En fait, pour être plus précis, un quorum est calculé de la manière ci-dessous :
![center](http://1.bp.blogspot.com/-vL7EP1n4Jgs/T18f1aw36II/AAAAAAAAAk4/AshD58ObrgE/s1600/cassandra12.png)

Ainsi, par exemple, si le facteur de réplication est de 3, alors le quorum est de 2 (c'est à dire que l'on tolère qu'un seul réplica est indisponible).

De même, si le facteur de réplication est de 6, alors le quorum des de 4 (c'est à dire que 2 réplicas peuvent être indisponibles).

Lors d'une lecture, le niveau de consistance spécifie combien de réplicas doivent répondre avant que le résultat ne soit fourni au client.

Cassandra contacte le nombre de réplicas indiqué afin d'obtenir la donnée la plus récente (en se basant sur le timestamp).

Les niveaux de consistance disponibles sont les suivants :
![medium](http://2.bp.blogspot.com/-l5GRGCM3d2Y/T18l1f7VGDI/AAAAAAAAAlA/jPAqImzGCWs/s1600/cassandra13.png)

Le niveau quorum est calculé de la même manière que pour l'écriture.

Ainsi, par exemple, pour un facteur de réplication de 3, le quorum est de 2 (c'est à dire que l'on tolère qu'un seul réplica est indisponible).

De même, pour un facteur de réplication de 6, le quorum est de 4 (c'est à dire que 2 réplicas peuvent être indisponibles).

Concernant le choix du niveau de consistance sur l'écriture et la lecture, cela dépend, bien sûr, des besoins mais impact fortement le comportement de Cassandra.

Ainsi, si la latence est une priorité, un niveau de consistance à ONE peut être envisageable. Bien sûr, dans ce cas, il y a une forte probabilité que les données lues ne soient pas cohérente.

A l'inverse, si c'est l'écriture qui importe, alors une consistance à un niveau ANY peut répondre au besoin. Dans ce cas, ce sont les données lues qui risquent de ne pas être consistantes.

A noter que si c'est la consistance qui est importante (c'est à dire que les données lues sont les plus proches possibles de ce qui a été écrit), alors la formule suivante peut être appliquée :

![center](http://1.bp.blogspot.com/-19RglOrHOBI/T19Zlpb7TBI/AAAAAAAAAlI/Xa7Kqfi3dpY/s1600/cassandra14.png)

#Conclusion

Dans cet article, j'ai essayé de mettre en avant les points qui m'ont marqués (en bien ou en mal) (je tiens quand même à re-préciser que mon avis n'est que théorique et que je n'ai pas réellement utilisé le produit). 
Ainsi au niveau des points positifs, j'ai aimé les choix techniques et les concepts (très enrichissants d'ailleurs... ;-) ).

Cependant, coté points négatifs, je ne suis pas fan des dégradations coté consistance qui peuvent se produire (le coup de l'élément supprimé qui revit dans le système... comment dire... mais WTF???!! ;-)  - bon ok, la raison est viable mais quand même... -). 

Bien sûr, cela est configurable et il s'agit d'un choix technique mais j'avoue avoir été déçu par Cassandra. En outre, le fait de laisser à l'application cliente le choix du niveau de consistance, même s'il est louable, me semble dangereux au sens maintenabilité et compréhension du fonctionnement du système.

En tout cas, la documentation du produit est très claire et transparente quant au fonctionnement interne et aux risques encourus, et pour ça, chapeau! ;-)

Pour conclure, je ne ferai pas de super mot de la fin puisque je ne me suis borné qu'à traduire quelques bouts par ci par là de la documentation et que je trouve que cela suffit à lui-même ;-).

Par contre, pour tous ceux que ça intéresse, je vous invite à jeter un oeil à la documentation officielle très complète (aussi bien au niveau configuration que explication).

Enjoy! ;-)
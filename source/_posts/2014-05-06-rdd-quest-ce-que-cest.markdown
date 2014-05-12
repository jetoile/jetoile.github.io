---
layout: post
title: "RDD : qu'est ce que c'est"
date: 2014-05-12 09:21:35 +0200
comments: true
categories: 
- spark
- big data
- rdd
---
![left-small](/images/rdd/spark.png)

[Spark](http://spark.apache.org/) est un framework qui a de plus en plus le vent en poupe et le fait qu'il ait été [promu](https://blogs.apache.org/foundation/entry/the_apache_software_foundation_announces50) en _top-level project_ par la fondation Apache qu'il a rejoint récemment (en juin 2013) montre bien de l'intérêt qu'il succite (cela est d'aileurs confirmé par son intégration avec des solutions comme celles de [DataStax](http://www.datastax.com/) (cf. [ici](http://databricks.com/blog/2014/05/08/Databricks-and-Datastax.html)) ou [mapR](http://www.mapr.com/) (cf. [ici](http://www.mapr.com/products/apache-spark)).

Un des points central de Spark est son utilisation des RDDs (_Resilient Distributed Datasets_). 

Cet article tentera d'expliquer un peu plus précisément ce que sont ces fameux RDDs (enfin, pour être plus précis, il ne s'agit (comme à mon habitude) que d'une pseudo-traduction du [papier](https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf) de recherche expliquant ses tenants et aboutissants).

<!-- more -->


#Introduction

Les frameworks de cluster de calcul tels que [_Dryad_](http://cs.brown.edu/~debrabant/cis570-website/papers/dryad.pdf) et ceux basés sur [_MapReduce_](http://static.googleusercontent.com/media/research.google.com/fr//archive/mapreduce-osdi04.pdf) ont largement été adoptés pour les analyses de données à grande échelle. Ces systèmes permettent aux utilisateurs d'écrire des calculs parallèles en utilisant des opérateurs de haut niveau sans avoir à se soucier de la distribution des unités de calcul ni de la tolérance aux pannes.

Bien que ces frameworks actuels fournissent de nombreuses abstractions pour accéder aux ressources de calcul du cluster, ils n'en fournissent pas pour accéder à la mémoire partagée. Cela les rend inefficace pour une importante classe d'applications émergeantes : celle qui doit réutiliser les résultats intermédiaires entre leurs différents calculs. La réutilisation de données est courante dans de nombreux algorithmes _itératifs_ de _machine learning_ et de graphe (comme le __PageRank__, __K-Means__ et la régression logique) mais également dans des algorithmes d'extraction de données _interactive_ où un utilisateur a besoin d'exécuter de multiples requêtes sur un même sous ensemble de données. 

Malheureusement, dans la plupart des frameworks actuels, la seule manière de réutiliser des données entre les calculs (ie. entre deux _MapReduce_ job) est de les écrire dans un système de stockage externe (ie. un système de fichiers distribué). Cela engendre un _overhead_ dû à la réplication de données, aux I/O disques et à la sérialisation.

C'est pour palier à ces problèmes que des chercheurs ont développés des framework spécialisés pour les applications qui nécessitent la réutilisation de données. Par exemple, [_Pregel_](http://kowshik.github.io/JPregel/) est un système pour le calcul itératif de graphe qui garde les données intermédiares en mémoire, alors que [_HaLoop_](https://code.google.com/p/haloop/) offre une version itérative de _MapReduce_. 

Cependant, ces frameworks ne supportent que des _patterns_ de calculs précis (ie. une série d'étapes _MapReduce_) et s'appuient implicitement sur le partage de données. Ils ne fournissent pas l'abstraction pour une réutilisation plus globale.

C'est dans ce contexte qu'une nouvelle abstraction appelée _Resilient Distributed Datasets_ (__RDD__s) permettant de réutiliser efficacement les données dans une large famille d'applications a été proposée. Les RDDs sont tolérants à la panne et proposent des structures de données parallèles qui laissent les utilisateurs :

- persister explicitement les données intermédiaires en mémoire,
- controller leur partitionnement afin d'optimiser l'emplacement des données,
- manipuler les données en utilisant un ensemble important d'opérateurs.

La principale difficulté dans la conception des RDDs a été de définir une interface de programmation qui pouvait palier efficacement à la panne. Les abstractions de stockage en mémoire pour les clusters telles que les mémoires partagées distribuées, les stockages clé/valeur, les bases de données et Piccolo offrent une interface basée sur de petites mises à jours d'état mutables. Avec ces interface, les seules manières de fournir de la tolérance à la panne est de répliquer les données entre les machines ou de logger les mise à jours entre les machines. Ces deux approches sont consommatrices pour une charge de travail intensive sur des données puisque cela nécessite un fort transfert de données lors de la copie des informations entre les serveurs (le réseau est beaucoup plus lent que la RAM).

RDDs fournit, pour sa part, une interface basée sur des transformations "grosses mailles" (ie. _map_, _filter_ et _join_) qui appliquent les mêmes opérations aux données. Cela permet d'être résiliant à la panne en loggant les transformations utilisées pour construire l'ensemble de données plutôt que les données réelles. Si une partition de RDD est perdue, le RDD dispose de suffisament d'informations sur la manière dont il a été produit pour recalculer la partition manquante. Ainsi les données perdues peuvent être récupérées souvent rapidement sans avoir à recourir aux mécanismes de réplication souvent couteux.

Bien qu'une interface basée sur des transformations "grosses mailles" peut sembler limité, RDDs convient pour beaucoup d'applications parallèles qui appliquent naturellement les mêmes opérations à de multiples éléments de données. En effet, RDDs peut répondre efficacement à de nombreux modèles de programmations de type cluster qui ont, jusque là, été traités comme des systèmes séparés. 

#Resilient Distributed Datasets (RDDs)

Un RDD est une collection partitionnée d'enregistrements en lecture seule qui ne peut être créée que par des opérations déterministes :

-  soit à partir de données présentes dans un stockage stable,
-  soit à partir d'autres RDDs.

Ces opérations sont appelées __transformations__ pour les différentier des autres opérations. Parmi ces dernières il y a : `map`, `filter` et `join`.

Les RDDs n'ont pas besoin d'être matérialisés tout du long puisqu'un RDD dispose de suffisamment d'informations sur la façon dont il a été produit à partir d'un autre ensemble de données pour pouvoir être recalculé. Ainsi, un programme ne peut faire référence à un RDD s'il n'est pas capable de le reconstruire suite à une panne.

Enfin, les utilisateurs peuvent controler deux autres aspects des RDDs :

- la persistence : il est possible de préciser quels sont les RDDs réutilisés ainsi que la stratégie de stockage,
- le partitionning : les éléments des RDDs peuvent être partitionnés entre les machines en se basant sur une clé dans chaque enregistrement.

Par exemple, Spark expose les RDDs au travers d'une API où chaque ensemble de données est représenté comme un objet sur lequel des transformations sont appliquées. Les développeurs définissent un ou plusieurs RDDs au travers de transformations de données présente dans un stockage stable (ie. `map` ou `filter`) puis ils peuvent les utiliser dans des __actions__ (comme `count`, `collect` ou `save`) qui retournent une valeur à l'application ou qui exporte des données sur un stockage du système. A cela peut s'ajouter l'opération `persist` qui indique quels RDDs pourront être réutilisés dans les opérations futures (à noter qu'il est possible d'indiquer que l'opération de persistance doit être faite entre plusieurs machines).

Pour comprendre les avantages des RDDs comme abstraction de mémoire distribuée, une comparaison avec les mémoires partagées distribuées (_DSM_) a été faite.

![center](/images/rdd/comparaisonRdd.png "crédit photo : https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf")

Dans un système à base de mémoire partagée distribuée, les applications lisent et écrivent de manière aléatoire dans un espace d'adressage global. La principale différence entre les RDDs et DSM est que les RDDs ne peuvent créés qu'au travers de transformations _macros_ alors que DSM permet la lecture et l'écriture sur chaque allocation mémoire. Cela réduit l'utilisation des RDDs aux applications qui nécessitent de l'écriture en bloc mais cela permet une meilleure tolérance aux pannes. En outre, une autre particularité des RDDs est sa nature immuable qui permets de mieux gérer les cas où des noeuds peuvent être lents en effectuant des copies des tâches lentes comme dans MapReduce. Avec DSM, cela est difficilement implémentable car deux copies d'une tâche doivent potentiellement accéder aux mêmes allocations mémoires et que cela peut créer des conflits avec des mises à jour éventuelle.

Cependant, les RDDs s'appliquent mieux à des applications de type batch qui doivent exécuter les mêmes opérations sur tous les éléments de l'ensemble de données : dans ce cas, chaque transformation peut être vu comme une étape dans le graphe d'origine et la récupération des partitions perdues sans avoir à logger une grosse quantité d'information est plus adaptée. Dans le cas d'applications qui nécessitent une mise à jour d'un état partagé, les RDDs ne sont pas optimals.

Son intégration avec _Spark_ se fait au travers d'un API intégré au framework. Le driver se connecte à une cluster de _workers_ et permet de définir un ou plusieurs RDDs tout en y exécutant un ensemble d'action. Le driver permet également de suivre les modifications. Par exemple, l'utilisateur fournit des arguments aux opérations appliquées au RDD en passant des closures. En scala, chaque closure est représentée comme un objet Java qui peuvent être sérialisés et chargés sur un autre noeud. 

Les principales transformations et actions disponibles dans Spark sont les suivantes :

![center](/images/rdd/spark_operation_action.png "crédit photo : https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf")

En fait, les __transformations__ sont des opérations _lazy_ qui permette de définir un nouveau RDD alors que les __actions__ permettent d'exécuter un calcul et de retourner une valueur au programme ou de l'écrire sur un stockage externe. En plus de ces opérations, un RDD peut être persisté (méthode `persist`). En outre, il est possible d'obtenir l'ordre des partitions d'un RDD. Les opérations telles que `groupByKey`, `reduceByKey` et le `sort` fournissent un hash ou un interval de RDD partitionné.

Le choix de la représentation des RDDs doit permettre de retrouver la situation initiale au travers des transformations appliquées. Pour ce faire, une représentation sous forme de graphe a été choisie. En fait, chaque RDD est représenté par une interface commune qui expose différents types d'informations :

- un ensemble de partitions,
- un ensemble de dépendances aux RDDs parents,
- une fonction pour calculer l'ensemble de données en se basant sur ses parents,
- et des métadonnées sur ses plans de distribution ainsi que sur l'emplacement des données.

![center](/images/rdd/rdd_interface.png "crédit photo : https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf")

Ainsi, par exemple, un RDD qui représente un fichier HDFS a une partition pour chaque bloc de fichiers et connait l'emplacement de la machine où se trouve ce dernier. De même, le résultat d'une opération `map` sur ce RDD dispose des mêmes partitions mais applique la fonction _map_ aux données parentes.

Un autre point intéressant concernant le _design_ de cette interface est la manière de représenter les dépendances entres les RDDs. Cela a été résolu en définissant 2 types de dépendances :

- les dépendances _étroites_ où chaque partition d'un parent RDD est utilisé par au plus une partition d'un RDD enfant,
- les dépendances _larges_ où plusieurs partitions filles peuvent dépendre d'une partition donnée.

Par exemple, la fonction `map` peut engendrer des dépendances étroites alors que la fonction `join` peut produire des dépendances larges.

Ces distinctions sont importantes car :

- une dépendance étroite permet l'exécution en _pipeline_ sur un seul noeud du cluster. A l'inverse, une dépendance large nécessite que les données de toutes les partitions parentes soient présentes et il convient donc de les déplacer entre les noeuds en utilisant une opération de type MapReduce
- la récupération après un noeud en échec est plus efficace avec une dépendance étroite puisque seules les partitions parentes perdues doivent être recalculées et que cela peut être fait en parallèle sur différents noeuds. A l'inverse, avec une dépendance large, un simple échec d'un noeud peut entrainer la perte de plusieurs partitions sur tous les ancêtres d'un RDD entraînant une réexécution complète des opérations.

![center](/images/rdd/rdd_dependencies.png "crédit photo : https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf")

#Implémentation

Spark est un moteur rapide et général pour le traitement de données à grande échelle. Il permet, entre autre, de lire des données d'Hadoop (HDFS ou HBase) en utilisant les API de lecture existantes d'Hadoop. 

Ce paragraphe revient sur quelques-unes des parties techniques du système.

##L'ordonnanceur de job

L'ordonnanceur de Spark utilise la représentation des RDDs présentée précédemment.

Lorsqu'un utilisateur exécute une action sur un RDD, l'ordonnanceur examine le graphe d'origine des RDD pour construire un DAG ([_Directed Acyclic Graph_](http://en.wikipedia.org/wiki/Directed_acyclic_graph)) des étapes à exécuter. Chaque étape (déterminée, soit par les opérations de déplacements de données nécessaires définies par les dépendances larges, soit par une partition déjà calculée qui peut être court circuité par le calcul d'un RDD parent) contient une série de transformations chainées avec autant de dépendances étroites que possible. L'ordonnanceur exécute alors les tâches pour calculer les partitions manquantes sur chacune des étapes jusqu'à l'optention du RDD cible.

L'ordonnanceur assigne les tâches aux machines en fonction des données locales. Si une tâche nécessite une partition qui est disponible en mémoire sur un noeud, cette dernière est rappatriée. Sinon, si une tâche calcule une partition pour laquelle le RDD fourni une localisation souhaitée, alors la partition est envoyée à ce noeud.

Pour les dépendances larges, les enregistrements intermédiaires sont matérialisés sur les noeuds hébergeant la partition parente afin de simplifier la reprise sur erreur.

Si la tâche échoue, elle est réexécutée sur un autre noeud tant que l'étape parente est disponible. Si certaines étapes deviennent inaccessibles, la tâche est re-soumise en parallèle afin de recalculer la partition manquante.

##Gestion mémoire

Spark fournit 3 options de stockage pour les RDDs persistants :

- stockage en mémoire comme objet java désérialisé,
- stockage en mémoire comme objet java sérialisé,
- stockage sur disque.

Le stockage en mémoire comme objet java désérialisé est le plus rapide car la JVM peut accéder nativement à chaque élément du RDD.

La seconde option permet aux utilisateurs de choisir une représentation plus efficace de la mémoire lorsque l'espace est limité. Bien sûr, la performance est moindre que pour la première option.

La troisième option est utile lorsque les RDDs sont trop larges pour tenir en RAM.

Afin de gérer la mémoire limitée disponible, la politique d'éviction LRU est utilisée pour les RDDs. Lorsqu'une nouvelle partition RDD est calculée mais qu'il n'y a pas suffisamment de place pour la stocker, la partition du RDD le plus anciennement accédée est évincée à moins qu'il ne s'agisse du même RDD que celui qui doit recevoir la nouvelle partition. Dans ce cas, la vieille partition est conservée en mémoire. Cela est important car la plupart des opérations exécutent des tâches sur un RDD entier. Aussi, la probabilité est forte pour qu'une partition déjà en mémoire soit utilisée ultérieurement.

##Point de contrôle
Bien que l'état initial peut toujours être utilisé pour recalculer les RDDs suite à un échec, cela peut être couteux surtout si la chaine est longue. Ainsi, il peut être utile de disposer d'un point de controle des RDD (_checkpoint_) sur disque.

En général, les points de controle sont utiles pour des RDDs qui disposent d'un graphe contenant des dépendances larges. Dans les autres cas, cela peut être inutile. Ce point de controle peut être controlé manuellement via le flag `REPLICATE` de la méthode `persist`.

#Conclusion

On a vu dans ce court article une première explication de ce qu'était les RDDs qui sont la pierre angulaire de Spark. 

Pour ceux qui souhaiteraient avoir plus d'informations là-dessus, je les renverrai vers le papier officiel dont est extrait cet article disponible [ici](https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf) ;-)


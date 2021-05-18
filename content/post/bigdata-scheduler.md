---
title: "Big Data et Scheduler : Reflexion"
date: 2021-05-18 18:20:00 +0200
comments: true
tags: 
- bigdata
- scheduler
- architecture
---
![left-small](/images/hadoop-all.png)
Dans le monde Big Data, rapidement se pose un certain nombre de besoins. 

Besoins qui, prit unitairement, sont déjà complexes à résoudre mais qui, mis bout à bout, s'avèrent encore plus difficile à intégrer.

Cet article n'a pas pour objectif de fournir une solution clé en main mais plutôt de poser différentes réflexions que j'ai pu avoir afin de savoir si elles avaient un sens...

<!-- more -->

# Besoins

Lorsque l'on est confronté au Big Data, les besoins qui en découlent sont les suivants :
- avoir un __scheduler__ scalable permettant d'exécuter et ordonnancer les différents jobs (qu'ils soient MapReduce, Spark, Hive ou autres) mais également exécuter des actions qui leurs sont associées (déplacement de fichiers/répertoires, création/suppression/modification de tables hive, création de fichiers \_SUCCESS, etc.)
- avoir un __catalogue de données__ (afin de connaitre ses datasets) pour pouvoir retrouver l'information
- disposer d'un __lineage de données__ (afin de pouvoir savoir à partir de quel dataset est issu tel autre)
- avoir de la __qualité de données__

Si nous revenons plus précisément sur certains de ces points, __le lineage de données__ et la __qualité de données__ méritent qu'on s'y arrête un peu plus longtemps.

Concerant le __lineage de données__, cela est intéressant de savoir à partir de quel dataset un dataset est issu mais il peut également être utile de disposer d'un __lineage de traitement__, c'est à dire savoir quel job a produit quel dataset. En effet, cela permet de facilement savoir, si un dataset a, par exemple, mal été produit, quel est le job dans la chaine qui est fautif. 

De même, dans un contexte de véracité ou d'audit, pouvoir prouver que tel traitement produit bien le résultat voulu peut être primordial.

En outre, en plus d'avoir un __lineage de données__ portant sur le dataset, il peut être intéressant d'avoir un lineage de données portant sur le contenu du dataset. En effet, connaitre la propagation d'une information (ou d'un champ) de manière plus fine peut être nécessaire.

Pour ce qui concerne la __qualité de données__, cela lève différentes interrogations :
- que faire lorsque la donnée produite ne respecte pas le contrat (car oui, il s'agit bien d'un contrat au sens pré-condition/post-condition) :
	- doit-on arrêter la chaine de traitement?
	- doit-on lever une alerte et continuer le traitement? et si oui, que faire de la donnée ignorée/mal produite (doit-on faire une _dead letter queue_ sur la donnée en amont ou en aval?)?
- doit-on être pro-actif (ie. faire une verification avant/après l'exécution de la transformation) ou le faire de manière lazy/asynchrone?
- comment doit être exprimé le contrat et qui (au sens traitement) doit vérifier la véracité de ce dernier?

Bref, plus de question que de réponses... mais qu'il est important de se poser.

Aussi, le besoin peut maintenant être exprimé de la manière suivante :
- avoir un __scheduler__ 
- avoir un __catalogue de données__ 
- disposer d'un __lineage de données__ au niveau du dataset
- disposer d'un __lineage de données__ au niveau du champ
- disposer d'un __lineage de traitement__
- avoir de la __qualité de données__ :
	- disposer d'un moyen pour exprimer le contrat à respecter
	- disposer d'un moyen pour exprimer le comportement à avoir si le contrat n'est pas respecté


# Constat

## Scheduler

Si on revient sur la partie __scheduler__, un constat peut être fait concernant les solutions les plus souvent utilisées/connues (airflow ou oozie pour n'en citer que quelques unes) : elles sont verbeuses (au sens _don't repeat yourself_).

Afin de permettre à l'orchestrateur de calculer/représenter son _DAG_ (Directed Acyclic Graph) d'exécution, il faut :
- indiquer/représenter les input et output des jobs
- ajouter les pre/post actions à exécuter (à noter que la vérification des conditions de l'exécution d'un job peut être vu comme un pré-action)

Pourtant, devoir indiquer/représenter les input/output d'un job est déjà exprimé dans le job en lui-même (par exemple, si on prend un job Spark, afin de pouvoir consommer/produire un RDD/DataSet/DataFrame, il faut effectuer une opération de lecture/écriture explicite. De même pour une requête SQL où ces informations apparaissent). 

Concernant les pre/post actions, une grande majorité d'entre elles peuvent être déduite du job : par exemple, on ne peut lire une donnée non présente. Ainsi devoir à l'exprimer au niveau du __scheduler__ est redondant. 

De même si on _sait_ que le job _doit_ créer/ajouter la partition hive pour la donnée produite, devoir le repréciser pour chaque job au niveau du scheduler est dommageable. 

Si un alter table hive doit être effectué dans le cas où un nouveau champ venait à être produit, nous y reviendrons un peu plus tard... :P 

Bref, on peut donc constater que la majorité des choses qui doivent être exprimées dans le scheduler sont redondantes. 


## Catalogue de données

Le catalogue de données permet de connaitre l'ensemble des datasets du système. Afin de le peupler, plusieurs approches sont possibles :
- utiliser directement le metastore hive comme étant le catalogue de données
- dissocier le catalogue de données du métastore hive. Dans ce cas, il est possible d'avoir :

Dissocier le catalogue de données du métastore hive apporte plusieurs avantages :
- le découplage
- la possibilité d'ajouter d'autres types d'informations plus méta que ce qui est permis via le metastore (documentation, ...)

Cependant, cela demande à avoir à le maintenir/synchroniser et, pour cela, plusieurs approches sont possibles :
- une approche _lazy_ c'est à dire un mécanisme allant scruter le metastore hive (que cela soit fait en positionnant un _listener_ directement sur ce dernier ou via un job allant l'interroger)
- une approche plus _invasive_ qui consiste à coupler fortement les jobs en leur demandant d'aller renseigner le catalogue de données
- une approche _entre deux_ qui consiste à s'appuyer sur l'observabilité des jobs

En outre, vient se poser la question de la _véracité_ du métastore hive. En effet, le metastore hive est une projection du dataset qui peut être incomplète puisqu'on est en fonctionnement _schema on read_. Le dataset réel peut donc est plus complet que ce qui est déclaré dans le metastore (le cas des _view_ n'est, ici, pas pris en compte) (à noter que l'utilisation de SparkSQL permet de réduire la chance de diverger).


## Linéage de données

Le __lineage de données__ est une problématique non triviale et est parfois faite de manière déclarative (et donc non à jour...).

Cependant, les jobs disposent de l'information et il est donc possible de réaggréger cette dernière de manière pro-active ou non afin de disposer d'un linéage (au moins le lineage de dataset). 

Bien sûr disposer de nombreuses technologies de traitement rend cela plus complexe. 

Une autre approche pourrait consister à corréler les accès au metastore hive et le plan d'exécution des jobs mais il se pose alors la même question de la complétude de l'information que pour le __catalogue de données__.

Concernant le linéage de données au niveau champ, cela s'avère assez compliqué.

## Linéage de traitement

Disposer d'un __linéage de traitement__ complet (ie. pas seulement avec une vision d'un _workflow_ donné) peut s'avérer complexe avec les scheduler actuel. 

En outre, si plusieurs technologies de scheduler sont utilisées ou si en existe plusieurs instances, le manque d'API/interface commune rend l'extraction et la corrélation difficile.

## Qualité de données

La __qualité de données__ est souvent effectuée a posteriori sur les datasets avec présentation des résultats sous forme de dashboard et/ou possibilité d'envoie d'alertes. 

Cela permet d'éviter d'avoir un arrêt de la chaine de traitement mais cela induit qu'il faut re-traiter la donnée toute la donnée corrompue. 

Pour ce faire, il devient alors utile/indispensable de disposer du __linéage de données__ (au minimum dataset). 

Une fois l'ensemble des datasets à re-produire en main, il faut alors en déduire les jobs qu'il faut relancer et re-scheduler ces derniers (après une éventuelle correction) ou un sur-ensemble si le scheduler ne permet pas une exécution unitaire.

Si la __qualité de données__ est effectuée de manière défensive (ie. qu'un job ne s'exécute que si ses pre/post conditions sont respectées) cela peut permettre :
- une consommation moindre de ressource de calcul (le dataset est lu et traité moins souvent)
- d'éviter d'avoir à re-processer un ensemble de datasets corrompus

Cependant, il se pose la question des données de qualité. En effet, il s'agit souvent de données aggrégées (somme, moyenne, nombre d'occurence, ...). Il convient alors de les stocker et de les rendre accessible aux jobs en ayant besoin afin d'éviter de les recalculer.

Une autre question qui doit être posée est le couplage entre la génération de ces métriques de qualité et le traitement du job. Faut-il que le job produise ces données ou peut-on la voir comme une décoration du job?


# Proposition

On a pu constater un certain nombre de points dans le paragraphe précédent (de manière totalement subjective et biaisé :P).

Dans cette partie, on va tenter de réassembler les morceaux au moins d'un point de vue conceptuel...

Concernant le __scheduler__, on peut le voir de manière centrale. En effet, il a connaissance de nombreuses informations modulo qu'il soit mono-tenant ou du moins qu'ils soient interopérables dans le cas contraire.

Pour la partie duplication d'informations entre les jobs et leurs actions qui leur sont associées, si il était capable d'intropecter le job, il serait presque possible de n'avoir aucune information superflux à lui donner. 

Cette introspection pourrait être faite via un système de SPI (_Service Provider Interface_) couplé à un mini wrapper sur les opérations de lecture/écriture par exemple.

Si l'ajout _magique_ d'une action n'était pas souhaitable, il est tout à fait possible d'imaginer un mécanisme de décorateur (au sens _design pattern_) qui permettrait d'ajouter un comportement au job et qui irait l'introspecter afin de découvrir la majorité des paramètres nécessaire à son fonctionnement.

Les informations dont le __scheduler__ disposeraient permettrait alors de répondre :
- au __linéage de données__ (du point de vue dataset)
- au __linéage de traitement__

Il est donc succeptible (afin de séparer les _concerns_) de remonter ces informations à des services tièrces chargés d'aggréger et de mettre à disposition ces dernières.

Concernant le __catalogue de données__, si il est vu découplé du metastore hive et si on considère que le schéma réel du dataset n'est pas porté par le metastore hive mais par un autre format exhausif (comme parquet, avro, protobuf), on obtient alors la notion de __schema registry__. 

Ce __schema registry__ deviendrait alors la source de vérité sur les datasets et chaque dataset alors consommé/produit devrait être connu de ce dernier et associé au format choisi (parquet, avro, protobuf) mais également à ses meta-données (localisation hive, etc). Le __scheduler__ ou  le job devraient alors y accèder et il deviendrait alors possible de connaitre les champs consommés et produits. En effet, si chaque input/output du job dispose de sa propre déclaration au niveau du __schema registry__, alors par construction, on connait les champs consommés/produits. Il devient alors possible de disposer du __linéage de données avec la granularité champ__. 

Pour les mêmes raisons qu'avec le __catalogue de données__, cette remonté d'accès aux datasets pourraient être fait un à service tierce via un système de _hook_, _listener_ ou autre.

Enfin pour la __qualité de données__, si on considère que les pré/post condition ne sont que le résultat de la vérification d'un contrat sur le dataset (ce dont il s'agit réellement), alors cette information devrait être associée aux metadonnées du dataset et donc portée et exprimée au niveau du __schema registry__. 

Les metriques aggrégés de qualités pourraient également être enregistrées en tant que metadonnées associés au dataset (?).

D'un point de vue exécution, ce calcul de données aggrégées et la décision de savoir quel comportement adopté dans le cas où le contrat n'est pas respecté pourrait être du ressort du __scheduler__ en décorant le job au même titre qu'il est décorré par les actions comme l'ajout de partition hive.

Enfin, concernant le design du __scheduler__ en lui même, il doit bien sûr être _scalable_, fournir les métriques nécessaires à son monitoring/supervision. 

Si il est multi-tenant, il doit être capable d'éviter les phénomènes de famine en disposant de _scheduler_ permettant la préemption d'exécution de jobs. 

Il devrait également disposer d'exécuteurs indépendants et autonomes pilotés par un _manager_ afin de permettre la résilience des exécutions et et la bonne exécution des jobs/actions (en gérant les retry ou en tuant une action bloquée par exemple)

# Conclusion

Dans cet article, j'ai essayé de poser un certain nombre de besoins qui sont généralement présents dans les projets Big Data ainsi que les constats que j'ai pu faire et une proposition (qui vaut ce qu'elle vaut...) pour réassembler ces différents points. 

L'approche que j'ai aimé avec ma proposition est que, je trouve, qu'au final, il y avait une certaine forme de cohérence un peu comme les pièces d'un puzzle qu'on emboite... avec en son centre le __scheduler__ :P  

Bien sûr tous ces points sont discutables et c'est d'ailleurs la raison pour laquelle j'ai posé mes réflexions dans cet article afin de fournir un matériel à partir duquel il sera possible de débattre/échanger/discuter si le coeur vous en dit :P 


---
layout: post
title: "Sqoop et Parquet : Mode d'emploi"
date: 2016-11-28 11:15:19 +0100
comments: true
categories: 
- Hadoop
- Sqoop
- Parquet
---
![left-small](/images/Apache_Sqoop_logo.svg.png) Dans le monde du _BigData_ (en l'occurence avec Hadoop), il est parfois utile de pouvoir importer le contenu d'une base de données dans son Datalake.

Pour ce faire, [Apache Sqoop](http://sqoop.apache.org/) est une des alternatives pour le faire (peut être pas la meilleure mais bon...). 
 
En effet, Sqoop permet d'importer (et exporter également) les données d'une base de données dans :

- hdfs au format _plain text_, sequencefile, avro ou parquet
- hive
- hbase
 
En outre, il permet d'avoir un mode incrémental afin de gérer le mode delta.

Cependant, comme on le verra dans cet article, Sqoop n'est pas aussi trivial qu'il peut le paraitre.

C'est ce qui sera détaillé dans cet article : à savoir une sorte de mini retour d'expérience... et heureux en plus ;)

<!--more-->

#Cas d'usage

Techniquement, voilà ce qui est souhaité concernant l'import des données dans hdfs : 

- pouvoir ingérer le contenu d'une base de données dans hdfs au format parquet ou avro,
- pouvoir ingérer les données de la table en mode incrémental,
- pouvoir préciser la requête avec ```--query``` en raison que la colonne utilisée par le ```--split-by``` dispose d'un index partitionné.

Pour être plus précis sur les besoins, la volonté de disposer des données au format parquet ou avro est liée au besoin d'effectuer des traitements Spark simplement (par exemple, pouvoir repartitionner les données en fonction d'un certain nombre de colonne).

En outre, concernant le mode incrémental, cela est lié au fait que des données sont reçues quotidiennement et que des traitements sont effectués à l'aide de PL/SQL (_no comment..._). Heureusement, il peut être considérer que les informations reçues/calculées sont seulement ajoutées aux tables.

Enfin, afin de permettre la parallélisation du traitement, plusieurs _mappers_ seront utilisés. Afin de permettre le calcul des _ranges_ affectés à chacun des _mappers_, il est nécessaire de préciser l'option ```--split-by```. Cependant, dans notre cas d'usage, la colonne utilisée pour le split-by dispose d'un index mais est partitionné. Etant donné qu'il n'est pas possible (toujours dans notre cas précis) de préciser l'option ```--boundary-query```, c'est donc le comportement par défaut qui est appliqué (à savoir un ```select min(<split-by>), max(<split-by>)```). Cependant, attaquer la table sans préciser la partition implique que notre temps d'exécution du min/max dure plusieurs heures, chose non acceptable dans le cas du fonctionnement incrémental. 

Aussi, il a été décidé d'ingérer la table partition par partition en précisant la partition à chaque fois avec l'option ```--query``` (à noter que l'option ```--where``` aurait pu suffire mais que cela n'a pas été testé).

#Mise en oeuvre

Afin de mettre en oeuvre notre cas d'usage, plusieurs expérimentations on été effectuées.

Dans ce chapitre, il sera expliqué ce qui a pu être constaté ainsi que les _workaround_ qui ont été trouvées lorsqu'un soucis se présentait.

##Import au format parquet ou avro

En fait, lorsque la commande ```sqoop import``` est utilisée avec les options ```--as-avrodatafile``` ou ```--as-parquetfile```, les données récupérées sont toutes de type string. Aussi, il est nécessaire de préciser le type de chaque colonne avec l'option ```--map-column-java```. Pour trouver le typage de chaque colonne, une simple requête jdbc a été effectuée afin de trouver les colonnes et leurs types.

##Import incrémental

Le mode incrémental ne supportant pas le format avro, il a donc été écarté et l'import s'est fait au format parquet.

##Parallélisation de l'import

En fait, le fait de préciser la requête d'import avec sqoop 1.4.6 en mode parquet est buggé...
En effet, il existe 2 _issues_ qui traitent de ce problème :

- [SQOOP-2582](https://issues.apache.org/jira/browse/SQOOP-2582) - Query import won't work for parquet
- [SQOOP-2408](https://issues.apache.org/jira/browse/SQOOP-2408) - Sqoop doesnt support --as-parquetfile with -query option.

Après avoir appliqué les 2 patchs et recompilé Sqoop (avec un joli ```ant package -Dhadoopversion=210```, le verdict tombe... : une erreur kite.

Heureusement, une montée de version de kite directement dans le lib de sqoop permet de faire fonctionner l'import (ie. remplacer ```kite-data-core-1.0.0.jar``` par ```kite-data-core-1.1.0.jar```, ```kite-data-hive-1.0.0.jar``` par ```kite-data-hive-1.1.0.jar```, ```kite-data-mapreduce-1.0.0.jar``` par ```kite-data-mapreduce-1.1.0.jar``` et ``````kite-hadoop-compatibility-1.0.0.jar``` par ```kite-hadoop-compatibility-1.1.0.jar```).

#Conclusion

Ce mini retour d'expérience sur l'utilisation de Sqoop (dans sa version 1.4.6) a permis de voir qu'il était tout à fait possible d'importer des données d'une base de données dans hdfs au format parquet.

Cependant, il semble que, pour l'instant, Sqoop manque _un peu_ de maturité avec Parquet et trouver des contournements pour faire fonctionner le tout n'est pas des plus trivial mais heureusement, cela fonctionne ;)

A noter que dans notre cas d'usage, il n'y avait pas, bien sûr, qu'une seule table mais plusieurs centaines.
---
title: "Hadoop Unit 1.3"
date: 2016-05-16 14:20:51 +0200
comments: true
tags:
- Hadoop
---

![left-small](/images/hadoop-all.png) Si vous êtes un lecteur assidu (ou pas ;)), vous avez pu vous rendre compte que j'avais posté précédemment sur un composant au doux nom d'Hadoop-Unit.

J'ai le plaisir de vous annoncer qu'il a été releasé en version 1.3 et qu'il est également disponible sur maven central.

Il intègre dans sa nouvelle version :

* support d'ElasticSearch 5.0.0-alpha2
* correction de bugs : la variable d'environnement HADOOP_UNIT n'est plus nécessaire que pour les utilisateurs de Windows (merci Florent ;))
* passage en version 0.1.6 de [Hadoop Mini Cluster](https://github.com/sakserv/hadoop-mini-clusters)

A noter que pour utiliser Hadoop Unit en mode standalone, il est maintenant nécessaire de choisir entre Hadoop-Unit version SolR et Hadoop-Unit version ElasticSearch.

Cela est dû à un conflit de jars (Lucene pour ne pas le citer...) qui oblige à gérer ces composants indépendamment...

Pour les téléchargements, ça se passe ici :

* [Hadoop-Unit compatible ElasticSearch](http://search.maven.org/remotecontent?filepath=fr/jetoile/hadoop/hadoop-unit-standalone-elasticsearch/1.3/hadoop-unit-standalone-elasticsearch-1.3.tar.gz)
* [Hadoop-Unit compatible SolR](http://search.maven.org/remotecontent?filepath=fr/jetoile/hadoop/hadoop-unit-standalone-solr/1.3/hadoop-unit-standalone-solr-1.3.tar.gz)


Enjoy ;)

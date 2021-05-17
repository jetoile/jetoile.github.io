---
title: "Logstash : tour d'horizon sur les stratégies de déploiement"
date: 2014-04-07 11:20:26 +0200
comments: true
tags: 
- java
- supervision
- logstash
---

![left-small](/images/logstash/logstash.png)
Cet article fera un rapide tour d'horizon sur les différentes stratégies qui peuvent être utilisées pour [Logstash](http://logstash.net/).

Pour ce faire, je m'appuierai sur le très bon [livre officiel](http://www.logstashbook.com/) que je me suis procuré (moyennant environ 10€) et qui fournit une très bonne vision sur ce qui est possible de faire ainsi que sur les différents concepts mais également sur les différentes stratégies de déploiement. 

Même si je résumerai succinctement quelques-uns des concepts afin que cet article soit un minimum compréhensible, cet article traitera surtout sur la façon dont il est possible de déployer les agents Logstash.

[_ndlr_ : par contre, je ne ferai, comme à mon habitude, que retranscrire ce qui est présent dans le livre...]

<!-- more -->

# Les concepts

Logstash est écrit en JRuby et fonctionne dans une JVM. Son architecture est orientée messages et est très simple. Plutôt que de séparer le concepts d'agents et de serveurs, Logstash se présente comme  un simple agent qui est configuré pour combiner différentes fonctions avec d'autres composants open souce.

L'écosystème de Logstash est constitué de 4 composants :

* __Shipper__ qui envoie des événements à Logstash.  
* __Broker__ et __Indexer__ qui reçoivent et indexent les événements.
* __Search__ et __Stockage__ qui permettent de rechercher et de stocker les événements.
* __Web Interface__ qui est une interface web appelée [__Kibana__](http://www.elasticsearch.org/overview/kibana/).

Les serveurs Logstash sont constitués d'un ou de plusieurs de ces composants indépendamment, ce qui permet de les séparer offrant ainsi la possibilité de _scaler_  mais également de les combiner en fonction du besoin.

Dans le plupart des cas, Logstash sera déployé de la manière suivante :

* Les hôtes exécutant les agent Logstash comme des __Shipper__ qui émettent, comme des événements, les logs des applications, services et hôte à un serveur central Logstash. Ces hôtes n'ont besoin de disposer que d'agents Logstash.
* Le serveur central Logstash qui aura à sa charge l'exécution du __Broker__, __Indexer__, __Search__, __Storage__ et __Web Interface__ afin de recevoir, _processer_ et stocker les logs.

![center](/images/logstash/archi01.png)

En fait, une configuration typique de Logstash est la suivante :

```text
input { 
  stdin { } 
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}

output {
  elasticsearch { host => localhost }
  stdout { codec => rubydebug }
}
```

où :

* `input` peut prendre en valeur des _plugins_ qui correspondent à ce que peut prendre en entrée l'agent (comme, par exemple, l'entrée standard ou le contenu d'un fichier).
* `filter` peut prendre en valeur des _plugins_ qui permettent de manipuler l'événement en le _parsant_, filtrant ou en ajoutant des informations issues du parsing ou non.
* `output` peut prendre en valeur des _plugins_ qui permettent de préciser où seront envoyés les événements (comme, par exemple, la sortie standard ou ElasticSearch).


# Les différentes stratégies de déploiement possibles

## Le mode de déploiement _classique_

Dans l'architecture de déploiement _classique_, on retrouve la _stack_ préconisée qui est la suivante :

* Les agents Logstash se trouvant sur les machines hôtes collectent et émettent les logs (sous forme d'événements) au système central.
* Une instance d'un système de bufferisation (comme [__Redis__](http://redis.io/) ou autre, comme une implémentation d'[__AMQP__](http://www.amqp.org/)) reçoit les événement sur le serveur central et joue le rôle de buffer.
* Un agent Logstash extrait les événements de logs du buffer et les traite.
* L'agent Logstash envoie les événements d'index dans ElasticSearch.
* ElasticSearch stocke et rend les événements cherchable.
* Kibana permet la recherche et le rendu des événements indexés dans ElasticSearch.

![center](/images/logstash/archi02.png)

En fait, le __broker__ permet de servir de buffer entre les agents et le serveur Logstash. Cela est essentiel pour les raisons suivantes :

* Cela permet d'améliorer les performances de l'environnement Logstash en fournissant une buffer de cache pour les événements de log.
* Cele permet de fournir de la résiliance. Si l'indexation Logstash échoue, alors les événements sont mise en fils d'attente afin d'éviter la perte d'informations.

On observe donc, dans cette configuration, que les agents Logstash présents sur les machines hôtes ne font que transmettre sans intelligence réelle au buffer les différents événements de log et qu'ils n'ont pas _vraiment_ de logique (ie. ils n'ont pas de section __filter__ mais juste les sections __input__ et __output__).

## Le mode de déploiement sans agent
### A la mode système
Comme on a pu voir dans le paragraphe précédent, les machines hôtes disposent d'un agent Logstash complet. Cependant, ils n'ont pas vraiment de logique puisqu'ils ne font que transmettre les événements de logs au broker dont le rôle est de servir de buffer.

Cependant, parfois, il peut être intéressant de ne pas à avoir besoin d'installer un agent Logstash sur les machines hôtes :

* si la JVM déployé sur la machine hôte est limitée,
* si la machine hôte est un périphérique qui dispose de peu de ressource et qu'il n'est pas possible d'y installer une JVM ou d'exécuter un agent,
* s'il n'est pas possible d'installer n'importe quel logiciel sur la machine hôte.

Pour répondre à cette problématique, il est possible d'utiliser des outils systèmes comme __Syslog__. 

Dans ce cas, le serveur Logstash n'aura qu'à déclarer un _input_ supplémentaire permettant d'écouter des événéments (dans notre cas, Syslog).

A titre informatif, il est possible d'utiliser un _Appender_ syslog dans log4j ou logback (entre autre).


![center](/images/logstash/archi03.png)

### A la mode agent
Dans le cas où ni un agent Logstash ni Syslog ne sont envisageables, il est possible d'utiliser [Logstash Forwarder](https://github.com/elasticsearch/logstash-forwarder) (anciennement Lumberjack).

Il s'agit d'un client légé permettant d'envoyer des messages à Logstash en offrant un protocole maison intégrant de la sécurité (encryption SSL) ainsi que de la compression.

Il a été conçu pour être petit avec une faible emprunte mémoire tout en étant rapide. Il a été écrit en [Go](http://golang.org/).

Dans ce cas, il suffit d'exécuter logstash-forwarder avec les _bons_ fichiers de configuration spécifiant l'adresse du serveur cible ainsi que l'emplacement du certificat et les fichiers à scruter.

Du coté serveur, il suffit, tout comme pour le mode sans agent à base de Syslog, de déclarer un _input_ lumberjack.

A noter que d'autres _shipper_ sont également disponibles tels que :

* [Beaver](https://github.com/josegonzalez/beaver)
* [Woodchuck](https://github.com/danryan/woodchuck)

# Les filtres

Logstash vient avec un système de filtre qu'il est possible de configurer via la section __filter__. 

Ces filtres permettent de filtrer mais également de modifier (via __mutable__) le contenu de l'événement. Ils permettent également de _parser_ les événements (via __grok__) afin de les rajouter lors de la phase d'indexation (et donc de stockage). Cela permet ainsi de pouvoir rechercher des événements de manière plus ciblé.

Il existe plusieurs stratégies lors de l'utilisation de filtres :

* filtrer les événements sur l'agent,
* filtrer les événements sur le serveur central,
* émettre les événements au bon format.

Le plus simple est encore d'émettre les logs au bon format, cependant, cela n'est pas toujours possible (trop de log différents, systèmes hétérogènes, code legacy, ...).

Une autre manière de faire est d'exécuter le filtrage localement (ie. directement sur l'agent). Cela permet de réduire la charge de traitement du serveur central et d'être sûr que seuls les événements propres et structurés seront stockés. Cependant, cela oblige à maintenir une configuration plus complexe sur chaque agent.

A l'inverse, si le filtrage est effectué sur le serveur central, cela permet de centraliser les filtres et permet donc une administration plus simple. Cependant, cela demande des ressources supplémentaires pour effectuer le filtrage sur un plus grand nombre d'événements.

# La scalabilité et Logstash
Une des grande force de Logstash est qu'il est possible de le composer avec différents composants : Logstash lui-même, Redis comme _broker_, ElasticSearch et bien d'autres éléments qu'il est possible de composer via la configuration de Logstash. 

Ainsi, il est possible de jouer à plusieurs niveaux pour répondre à telles ou telles problématiques comme la perte de messages, le fait d'avoir un SPOF (_Single Point Of Failure_) ou d'avoir un point de contention dans le système.

Par exemple, si Redis est utilisé comme broker entre les agents Logstash et le serveur central, il peut être intéressant de passer Redis en mode _failover_ afin d'éviter une perte d'événements lors de la transmission de ces derniers. Pour ce faire, il suffit de configurer le plugin __redis__ de la section __output__  avec l'option `shuffle_hosts` pour indiquer à l'agent Logstash de n'utiliser qu'un seul noeud Redis lors de sa phase d'écriture. Du coté du serveur central, il suffit d'ajouter (et de configurer) autant de plugin __redis__ de la section __input__ que de noeud.

![center](/images/logstash/archi04.png)

Afin de permettre à la partie stockage/indexation d'être scalable, il suffit de configurer ElasticSearch en mode cluster, ce qui est natif chez lui.

Enfin, il est possible de rendre le serveur central Logstash robuste à la panne en en créant d'autres instances (mode _failover_) qui partageront la même configuration. 

![center](/images/logstash/archi05.png)


# Conclusion

En conclusion de cet article où je ne suis pas rentré dans les détails (mais ce n'est pas ce qui m'intéressait...), on peut constater qu'il existe moultes façons de configurer Logstash (et son écosystème) qui dépendent à chaque fois des besoins. 

Cela est rendu possible par l'architecture et la conception modulaire de Logstash et le fait qu'il est très simple de le _plugger_ à différentes solutions.

Même si cela est évident, je trouvais utile de le marquer noir sur blanc dans un court article... ;-)

# Pour aller plus loin...

* http://logstash.net/
* http://www.logstashbook.com/
* http://blog.xebia.fr/2013/12/12/logstash-elasticsearch-kibana-s01e02-analyse-orientee-business-de-vos-logs-applicatifs/

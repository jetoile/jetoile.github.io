---
title: "Hadoop ecosysteme bootstrap"
date: 2016-01-18 10:00:00 +0100
comments: true
tags:
- Hadoop
---
![left-small](/images/hadoop-all.png)
Comme je l'ai déjà dit dans un article [précédent](/2015/10/hadoop-et-son-ecosysteme.html), la force d'Hadoop n'est plus dans Hadoop en lui-même mais plutôt dans son écosystème.

Du coup, même lorsqu'on développe avec Spark, il est courant de vouloir s'interfacer avec le métastore de Hive ou encore avec SolR si l'objectif est de vouloir indexer dans ce dernier.

Parfois encore, l'objectif est de lire ou d'écrire de Kafka via Spark (ou en Spark Streaming).

Ainsi, lorsqu'on fait du développement avec des composants "BigData", on a souvent besoin d'un écosystème complet.

Alors, bien sûr, dans un monde idéal, il est possible de disposer de tout l'écosystème via Docker ou via une machine virtuelle. Pourtant, parfois, il n'est pas possible de disposer de la pleine puissance de son poste parce qu'on est, parfois (sic...), dans un contexte où disposer des droits administrateurs de son poste (et pire quand il s'agit d'un poste sous Windows...) est impossible.

Pour ce faire, il existe quelques solutions comme [Hadoop-mini-clusters](https://github.com/sakserv/hadoop-mini-clusters). Cependant, chaque composant doit alors être démarré dans le bon ordre et il faut avouer que c'est un peu verbeux...

En outre, les couches clientes fonctionnent principalement dans la même JVM.

L'objectif de ce post est de proposer une alternative beaucoup moins verbeuse et plus pratique.

Il ne s'agit que d'une simple surcouche à Hadoop-mini-clusters mais la solution proposée offre également d'autres possibilités qui ne sont pas offertes par la solution :

* possibilité de disposer d'un SolR qu'il soit en mode embedded ou en mode cloud
* possibilité d'avoir un oozie accessible par un client externe à la JVM

De plus, la solution proposée dans cet article (enfin plutôt sur mon [github](https://github.com/jetoile/hadoop-bootstrap)...) propose :

* de démarrer l'écosystème nécessaire dans un contexte de Test d'intégration
* de démarrer l'écosystème nécessaire en mode standalone avec une simple commande

<!-- more -->

#Contexte et proposition

Comme je l'ai dit précédemment, le besoin était de pouvoir disposer de l'écosystème nécessaire à l'exécution de tests d'intégration mais également de pouvoir démarrer l'écosystème nécessaire en mode _standalone_.

Bien sûr, le tout capable de fonctionner sous Linux mais aussi sous Windows... (_ndlr_ : oui... on n'a pas toujours la main sur ce que nous impose le client... :'( )

[Hadoop-mini-clusters](https://github.com/sakserv/hadoop-mini-clusters) est un composant très pratique mais un peu verbeux. En outre, devoir configurer chacun des composants alors que, souvent, une configuration par défaut est largement suffisante me semblait un peu inutile.

Enfin, comme je l'ai déjà dit, [Hadoop-mini-clusters](https://github.com/sakserv/hadoop-mini-clusters) a été conçu pour les TU/TI mais dans un cas de foncionnement en mode _standalone_ il faut alors accéder aux composants de manière _remote_.

Parmi les besoins, les composants suivants :

* HDFS
* Zookeeper
* HBase
* Hive (ie. HiveMetastore et Hiveserver2)
* Kafka
* Oozie
* SolRCloud
* SolR en mode _embedded_

Concernant les parties Spark ou MapReduce, ces frameworks disposant déjà d'un mode "développement", il m'était inutile de les proposer dans ma solution.

La surcouche proposée (que j'ai décidé d'appeler __Hadoop-bootstrap__) se trouve [ici](https://github.com/jetoile/hadoop-bootstrap)

Elle nécessite cependant quelques pré-requis...

#Prérequis

Pour fonctionner, Hadoop-bootstrap nécessite d'avoir téléchargé et décompressé Hadoop sur son poste. Dans mon cas, j'ai juste récupéré la version d'[Apache](http://www.apache.org/dyn/closer.cgi/hadoop/common/hadoop-2.7.1/hadoop-2.7.1.tar.gz) que j'ai décompressée dans ```/opt/hadoop```

Il faut ensuite positionner la variable d'environnement ```HADOOP_HOME``` à l'endroit où la version d'Hadoop a été décompressée.

Pour l'utilisation de Oozie, il est nécessaire de télécharger les ```sharelib``` de Oozie qui contient les librairies permettant de lancer des actions Hive, Spark, Shell ou autre : http://s3.amazonaws.com/public-repo-1.hortonworks.com/HDP/centos6/2.x/updates/2.3.2.0/tars/oozie-4.2.0.2.3.2.0-2950-distro.tar.gz

Il suffit alors de modifier le fichier ```default.properties``` en y renseignant les variables ```HADOOP_HOME``` et ```oozie.sharelib.path``` avec les bonnes valeurs.

#Fonctionnement

Une fois la partie prérequis réalisée, il ne reste plus qu'à générer :

* le jar si le besoin est d'utiliser Hadoop-bootstrap dans un contexte de tests d'intégration
* le tar.gz si le besoin est d'utiliser Hadoop-bootstrap dans son mode _standalone_

Pour ce faire, exécuter simplement la commande :
```bash
mvn install
```

##Pour un fonctionnement en test d'intégration ou unitaire

Pour ce mode, il est possible de faire de 2 manières différentes :

* démarrer les composants nécessaires manuellement
* utiliser un wrapper qui prend en paramètre les composants à démarrer

Attention cependant, dans les 2 cas, il est faut démarrer/arrêter les composants dans le bon ordre...

###Zookeeper

```java
private static Bootstrap zookeeper;

BeforeClass
public static void setup() {
  zookeeper = ZookeeperBootstrap.INSTANCE.start();
}

@AfterClass
public static void tearDown() {
  zookeeper.stop();
}
```

ou

```java
static private HadoopBootstrap hadoopBootstrap;

@BeforeClass
public static void setup() throws BootstrapException {
  hadoopBootstrap = new HadoopBootstrap(Component.ZOOKEEPER);
  hadoopBootstrap.startAll();
}


@AfterClass
public static void tearDown() throws BootstrapException {
    hadoopBootstrap.stopAll();
}
```

###HDFS

```java
private static Bootstrap hdfs;

BeforeClass
public static void setup() {
  hdfs = HdfsBootstrap.INSTANCE.start();
}

@AfterClass
public static void tearDown() {
  hdfs.stop();
}
```

ou

```java
static private HadoopBootstrap hadoopBootstrap;

@BeforeClass
public static void setup() throws BootstrapException {
  hadoopBootstrap = new HadoopBootstrap(Component.HDFS);
  hadoopBootstrap.startAll();
}


@AfterClass
public static void tearDown() throws BootstrapException {
    hadoopBootstrap.stopAll();
}
```

###HBase

```java
static private Bootstrap zookeeper;
static private Bootstrap hdfs;
static private Bootstrap hbase;

BeforeClass
public static void setup() {
  hdfs = HdfsBootstrap.INSTANCE.start();
  zookeeper = ZookeeperBootstrap.INSTANCE.start();
  hbase = HBaseBootstrap.INSTANCE.start();
}

@AfterClass
public static void tearDown() {
  hbase.stop()
  zookeeper.stop()
  hdfs.stop()
}
```

ou

```java
static private HadoopBootstrap hadoopBootstrap;

@BeforeClass
public static void setup() throws BootstrapException {
  hadoopBootstrap = new HadoopBootstrap(Component.HDFS, Component.ZOOKEEPER, Component.HBASE);
  hadoopBootstrap.startAll();
}


@AfterClass
public static void tearDown() throws BootstrapException {
    hadoopBootstrap.stopAll();
}
```

###Hive

```java
static private Bootstrap zookeeper;
static private Bootstrap hiveMetastore;
static private Bootstrap hiveServer2;

@BeforeClass
public static void setup() throws Exception {
    zookeeper = ZookeeperBootstrap.INSTANCE.start();
    hiveMetastore = HiveMetastoreBootstrap.INSTANCE.start();
    hiveServer2 = HiveServer2Bootstrap.INSTANCE.start();
}

@AfterClass
public static void tearDown() throws Exception {
    hiveServer2.stop();
    hiveMetastore.stop();
    zookeeper.stop();
}
```

ou

```java
static private HadoopBootstrap hadoopBootstrap;

@BeforeClass
public static void setup() throws BootstrapException {
  hadoopBootstrap = new HadoopBootstrap(Component.ZOOKEEPER, Component.HIVEMETA, Component.HIVESERVER2);
  hadoopBootstrap.startAll();
}


@AfterClass
public static void tearDown() throws BootstrapException {
    hadoopBootstrap.stopAll();
}
```


###Kafka

```java
static private Bootstrap zookeeper;
static private Bootstrap kafka;

@BeforeClass
public static void setup() throws Exception {
    zookeeper = ZookeeperBootstrap.INSTANCE.start();
    kafka = KafkaBootstrap.INSTANCE.start();
}

@AfterClass
public static void tearDown() throws Exception {
    kafka.stop();
    zookeeper.stop();
}
```

ou

```java
static private HadoopBootstrap hadoopBootstrap;

@BeforeClass
public static void setup() throws BootstrapException {
  hadoopBootstrap = new HadoopBootstrap(Component.ZOOKEEPER, Component.KAFKA);
  hadoopBootstrap.startAll();
}


@AfterClass
public static void tearDown() throws BootstrapException {
    hadoopBootstrap.stopAll();
}
```


###SolRCloud

La difficulté pour intégrer ce composant était que le mode SolR Cloud n'existait pas. Dans mon contexte d'utilisation, la cible était SolR Cloud qui impose l'utilisation de Zookeeper. Concernant l'interaction d'un client externe, il doit donc récupérer la configuration de SolR au sein de Zookeeper : le mode de fonctionnement est donc différent d'un SolR en _standalone_.

Pour faire fonctionner le mode SolR Cloud, il a fallu interagir directement avec Zookeeper en y écrivant les informations sur la collection cible car le mode embarqué n'offrait pas les API suffisantes pour créer une collection.

Actuellement, ce mode ne fonctionne qu'avec une collection qui doit être connue et renseignée préalablement (schema.xml, ...).

```java
static private Bootstrap zookeeper;
static private Bootstrap solr;

@BeforeClass
public static void setup() throws Exception {
    zookeeper = ZookeeperBootstrap.INSTANCE.start();
    solr = SolrCloudBootstrap.INSTANCE.start();
}

@AfterClass
public static void tearDown() throws Exception {
    solr.stop();
    zookeeper.stop();
}
```

ou

```java
static private HadoopBootstrap hadoopBootstrap;

@BeforeClass
public static void setup() throws BootstrapException {
  hadoopBootstrap = new HadoopBootstrap(Component.ZOOKEEPER, Component.SOLRCLOUD);
  hadoopBootstrap.startAll();
}


@AfterClass
public static void tearDown() throws BootstrapException {
    hadoopBootstrap.stopAll();
}
```


###Oozie

La difficulté pour intégrer ce composant était que [Hadoop-mini-clusters](https://github.com/sakserv/hadoop-mini-clusters) ne permettait pas à un client extérieur à la JVM ayant démarré Oozie d'intéragir avec le serveur.

En outre, pour utiliser les action Oozie tierce, il était nécesssaire de télécharger oozie. J'ai juste simplifié ce mode en obligeant l'utilisateur à disposer déjà de oozie sur son poste pour accélérer le mode de fonctionnement.

Il est donc nécessaire de télécharger les ```sharelib``` de Oozie qui contiennet les librairies permettant de lancer des actions Hive, Spark, Shell ou autre (en fait, pour être plus précis, les sharelib sont dans la distribution d'Oozie : on télécharge donc Oozie...) : http://s3.amazonaws.com/public-repo-1.hortonworks.com/HDP/centos6/2.x/updates/2.3.2.0/tars/oozie-4.2.0.2.3.2.0-2950-distro.tar.gz

Il suffit ensuite de modifier le fichier ```default.properties``` en y renseignant la variable ```oozie.sharelib.path``` avec la bonne valeur.


```java
static private Bootstrap hdfs;
static private Bootstrap oozie;

@BeforeClass
public static void setup() throws Exception {
    hdfs = HdfsBootstrap.INSTANCE.start();
    oozie = OozieBootstrap.INSTANCE.start();
}

@AfterClass
public static void tearDown() throws Exception {
    oozie.stop();
    hdfs.stop();
}
```

ou

```java
static private HadoopBootstrap hadoopBootstrap;

@BeforeClass
public static void setup() throws BootstrapException {
  hadoopBootstrap = new HadoopBootstrap(Component.HDFS, Component.OOZIE);
  hadoopBootstrap.startAll();
}


@AfterClass
public static void tearDown() throws BootstrapException {
    hadoopBootstrap.stopAll();
}
```

##Pour un fonctionnement en mode _standalone_

Pour ce mode de fonctionnement, il suffit de récupérer le tar.gz généré lors de la compilation. Il est _autoporteur_ et il suffit de le décompresser puis de démarrer Hadoop-bootstrap.

Ses fichiers de configuration se trouvent dans le répertoire ```conf```.

Afin de pouvoir choisir les composants à démarrer, il suffit de modifier le contenu du fichier ```hadoop.properties``` se trouvant dans le répertoire ```conf```.

Pour démarrer l'écosystème, il suffit d'exécuter la commande suivante :

```bash
./bin/hadoop-bootstrap start
```

Pour arrêter hadoop-bootstrap, exécuter la commande suivante :
```bash
./bin/hadoop-bootstrap stop
```

Pour démarrer hadoop-bootstrap en mode interactif, exécuter la commande :
```bash
./bin/hadoop-bootstrap console
```


<div>
<iframe allowfullscreen="" frameborder="0" height="315" src="https://www.youtube.com/embed/ef_9Y7a2ar8" width="560"></iframe></div>


A noter que ces commandes fonctionnent également sous Windows.

Concernant macOS, je n'ai pas testé mais cela ne devrait pas poser de difficultées.


#Conclusion

[Hadoop-bootstrap](https://github.com/jetoile/hadoop-bootstrap) n'est pas un composant révolutionnaire et ne fait que wrapper ce qui a déjà été fait dans [Hadoop-mini-clusters](https://github.com/sakserv/hadoop-mini-clusters).

Cependant, il est plus simple d'utilisation car limite plus l'utilisation des composants.

Il permet, en outre, d'utiliser l'écosystème Hadoop en mode _standalone_ (écosystème fonctionnant, bien sûr, en mode dégradé...) et d'interagir avec avec des clients externes (par exemple, ```hdfs dfs``` fonctionne bien avec le hdfs démarré). Pour plus d'information, il suffit d'aller voir la classe ```ManualIntegrationBootstrapTest``` dans le package ```fr.jetoile.sample.integrationtest```.

Alors, oui, ce n'est pas une révolution mais, au moins, ca permettra à l'utilisateur de ne pas à avoir à se battre avec les conflits de jar que j'ai pu avec avec SolR et Hadoop... ;)

Pour plus d'informations, les [tests unitaires](https://github.com/jetoile/hadoop-bootstrap/tree/master/src/test/java/fr/jetoile/sample/component) et les [tests d'_intégration_](https://github.com/jetoile/hadoop-bootstrap/blob/master/src/test/java/fr/jetoile/sample/integrationtest/IntegrationBootstrapTest.java).

A noter que pour faire fonctionner certains composants, je n'ai pas fait dans la finesse... et j'ai allègrement _hacké_ certaines classes...

A noter également que j'ai simplement copier/coller les tests de [Hadoop-mini-clusters](https://github.com/sakserv/hadoop-mini-clusters) sauf dans le mode intégration où je les ai un peu modifié pour permettre une connexion _remote_.

Voilà, ce n'est pas le plus beau code que j'ai pu faire, mais j'espère que ca pourra aider certains et, au pire, n'hésitez pas à contribuer ou à forker pour en faire ce que vous en voulez ;)

##Limitations connues

* phoenix ne fonctionne pas (voir la branche phoenix)
* une seule collection possible avec SolR
* des optimisations sur Oozie sont possibles (notamment sur la partie téléchargement/décompression)

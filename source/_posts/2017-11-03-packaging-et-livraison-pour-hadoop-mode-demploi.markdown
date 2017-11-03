---
layout: post
title: "Packaging, test et livraison pour Hadoop : Mode d'emploi"
date: 2017-11-03 16:16:05 +0100
comments: true
categories: 
- Livrable
- Test
- Hadoop
---
![left-small](/images/hadoop-all.png) Hadoop et son écosystème est un monde complexe où beaucoup de nos paradigmes de développeur Java / JavaEE (EE4J?) sont chamboulés.

D'une part les technologies utilisées diffèrent mais, en plus, d'autres questions telles que l'architecture, les tests (unitaires, intégrations, ...), la gestion des logs (debug, audit, pki, ...), les procédures de livraison, la gestion de la configuration de l'application, etc. viennent s'y ajouter.

Cet article va montrer comment il est possible de concilier simplement les tests d'intégration mais aussi le déploiement afin de tendre vers la philosophie de _continuous deployment_.

Une solution sera proposée et, même si elle est discutable et peut paraitre naïve, elle montrera comment il peut être simple de concilier ces deux points.

Concernant les technologies utilisées, la solution proposée utilisera :

* Spark 2.2.0
* Oozie
* Knox
* ElasticSearch 5.6.3
* Hive
* Scala 2.11 pour le langage mais Java pourrait également être utilisé
* Maven 3.5.0 pour la partie de _build_

Bien sûr, il est facilement possible d'ajouter d'autres technologies comme HBase, Sqoop, Hive (avec exécution de hql) ou autre.

A noter qu'il sera utilisé les composants Hortonworks (HDP 2.6.2) et c'est pourquoi toute la partie exécution des jobs se fera au travers d'Oozie qui est, le plus souvent quand on utilise une distribution du marché, la solution par défaut.

Ainsi il sera traité les points suivants :

* Description du cas d'usage et implémentation
* Anatomie d'un livrable
* Mise en oeuvre

<!-- more -->

# Description du cas d'usage et implémentation

L'exemple qui sera utilisée par la suite est simple mais il ne s'agit que d'un prétexte.

Il s'agira de lire des donnnées au format orc, de les indexer dans ElasticSearch et de créer une table Hive dessus. Le tout avec un job Spark.

Ainsi, le code est le suivant :

`Classe Main`
```java
object Main {

  lazy val logger = LoggerFactory.getLogger(this.getClass.getName)


  private def logArguments(reader: String, configPath: String, persister: String) = {
  }

  def parseArgs(args: Array[String]): (String, String, String) = {
    val jCommander = new JCommander(CommandLineArgs, args.toArray: _*)

    if (CommandLineArgs.help) {
      jCommander.usage()
      System.exit(0)
    }
    (CommandLineArgs.inputPath, CommandLineArgs.index, CommandLineArgs.docType)

  }

  def main(args: Array[String]): Unit = {

    val sparkSession = SparkSession.builder.appName("simpleApp").config("es.index.auto.create", "true").enableHiveSupport.getOrCreate

    val (inputPath, index, docType) = parseArgs(args)

    val job = new SimpleJob(sparkSession)
    val dataFrame = job.read(inputPath)
    dataFrame.cache()
    job.write(dataFrame, index, docType)
    job.createExternalTable(dataFrame, "tmp", inputPath + "/tmp")
  }
}
```


`Classe SimpleJob`
```scala
class SimpleJob(sc: SparkSession) {
  val sparkSession = sc

  def read(path: String): DataFrame = {
    sparkSession.read.format("orc").load(path)
  }

  def write(dataFrame: DataFrame, index: String, docType: String): Unit = {
    import sparkSession.implicits._
    val rdd = dataFrame.map(t => new FooData(t.getAs("id"), t.getAs("value")))

    import org.elasticsearch.spark._
    rdd.rdd.saveToEs(index + "/" + docType)
  }

  def createExternalTable(dataframe: DataFrame, hiveTableName: String, location: String) = {
    dataframe.registerTempTable("my_temp_table")
    sparkSession.sql("CREATE EXTERNAL TABLE " + hiveTableName + " (id STRING, value STRING) STORED AS ORC LOCATION '" + location + "'")
    sparkSession.sql("INSERT INTO " + hiveTableName + " SELECT * from my_temp_table")
  }

  def shutdown(): Unit = {
    if (sparkSession != null) {
      sparkSession.stop()
    }
  }
}
```

> On notera que la création de la session Spark a été dissociée du job afin de permettre la réalisation de tests d'intégration.

Afin de réaliser des tests d'intégration, [Hadoop Unit](https://github.com/jetoile/hadoop-unit) est utilisé avec le mode plugin maven en mode embedded.

Ainsi les tests d'intégrations donnent cela:
`Classe SimpleJobIntegrationTest`
```scala
class SimpleJobIntegrationTest extends FeatureSpec with BeforeAndAfterAll with BeforeAndAfter with GivenWhenThen {

  var configuration: Configuration = _
  val inputCsvPath: String = "/input/csv"
  val inputOrcPath: String = "/input/orc"
  val index: String = "test_index"
  val docType: String = "foo"
  var DROP_TABLES: Operation = _

  override protected def beforeAll(): Unit = {
    HadoopUtils.INSTANCE

    configuration = new PropertiesConfiguration(HadoopUnitConfig.DEFAULT_PROPS_FILE)

    DROP_TABLES = sequenceOf(sql("DROP TABLE IF EXISTS default.toto"));
  }

  before {
    val fileSystem = HdfsUtils.INSTANCE.getFileSystem

    val hdfsPath = "hdfs://" + configuration.getString(HDFS_NAMENODE_HOST_KEY) + ":" + configuration.getInt(HDFS_NAMENODE_PORT_KEY) + "/"

    val created = fileSystem.mkdirs(new Path(hdfsPath + inputCsvPath))

    fileSystem.copyFromLocalFile(new Path(SimpleJobIntegrationTest.this.getClass.getClassLoader.getResource("simplefile.csv").toURI), new Path(hdfsPath + inputCsvPath + "/simplefile.csv"))

    val sparkSession = SparkSession.builder.appName("test").master("local[*]").enableHiveSupport.getOrCreate

    val dataFrame = sparkSession.read.format("com.databricks.spark.csv")
      .option("header", "true")
      .option("delimiter", ",")
      .load(hdfsPath + inputCsvPath + "/simplefile.csv")

    dataFrame.write.option("orc.compress", "ZLIB")
      .mode(SaveMode.Append)
      .orc(hdfsPath + inputOrcPath + "/simplefile.orc")

    sparkSession.stop()
  }

  after {
    new HiveSetup(HiveConnectionUtils.INSTANCE.getDestination, sequenceOf(DROP_TABLES)).launch()
    HdfsUtils.INSTANCE.getFileSystem().delete(new Path("/input"))
  }

  feature("simple test") {
    scenario("read data") {

      Given("a local spark conf")
      val sparkSession = SparkSession.builder.appName("test").master("local[*]").enableHiveSupport.getOrCreate

      And("my job")
      val job = new SimpleJob(sparkSession)

      When("I read an orc")
      val hdfsPath = "hdfs://" + configuration.getString(HDFS_NAMENODE_HOST_KEY) + ":" + configuration.getInt(HDFS_NAMENODE_PORT_KEY) + "/"
      val dataFrame = job.read(hdfsPath + inputOrcPath + "/simplefile.orc")

      Then("I have the right schema")
      assertThat(dataFrame.schema.fieldNames).contains("id", "value")

      job.shutdown()
    }

    scenario("write into es") {

      Given("a local spark conf")
      val sparkSession = SparkSession.builder
        .appName("test")
        .master("local[*]")
        .config("spark.driver.allowMultipleContexts", "true")
        .config("es.index.auto.create", "true")
        .config("es.nodes", configuration.getString(ELASTICSEARCH_IP_KEY))
        .config("es.port", configuration.getString(ELASTICSEARCH_HTTP_PORT_KEY))
        .config("es.nodes.wan.only", "true")
        .enableHiveSupport
        .getOrCreate

      And("my job")
      val job = new SimpleJob(sparkSession)

      When("I read an orc")
      val hdfsPath = "hdfs://" + configuration.getString(HDFS_NAMENODE_HOST_KEY) + ":" + configuration.getInt(HDFS_NAMENODE_PORT_KEY) + "/"
      val dataFrame = job.read(hdfsPath + inputOrcPath + "/simplefile.orc")

      And("I call write method")
      job.write(dataFrame, index, docType)
      job.shutdown()

      Then("data is indexed into ES")
      val client = new PreBuiltTransportClient(Settings.EMPTY).addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(configuration.getString(ELASTICSEARCH_IP_KEY)), configuration.getInt(ELASTICSEARCH_TCP_PORT_KEY)))

      client.admin.indices.prepareRefresh(index).execute.actionGet

      val response = client.prepareSearch(index)
        .setTypes(docType)
        .setSize(0)
        .setQuery(QueryBuilders.queryStringQuery("*")).get().getHits().getTotalHits();

      assertThat(response).isEqualTo(3)
    }

    scenario("create external table") {

      Given("a local spark conf")
      val sparkSession = SparkSession.builder.appName("test").master("local[*]").enableHiveSupport.getOrCreate

      And("my job")
      val job = new SimpleJob(sparkSession)

      When("I read an orc")
      val hdfsPath = "hdfs://" + configuration.getString(HDFS_NAMENODE_HOST_KEY) + ":" + configuration.getInt(HDFS_NAMENODE_PORT_KEY) + "/"
      val dataFrame = job.read(hdfsPath + inputOrcPath + "/simplefile.orc")

      And("I call createExternalTable method")
      job.createExternalTable(dataFrame, "default.toto", hdfsPath + "input/titi")
      job.shutdown()

      Then("my external table is created")
      val stmt = HiveConnectionUtils.INSTANCE.getConnection.createStatement
      val resultSet = stmt.executeQuery("SELECT * FROM default.toto")
      while (resultSet.next) {
        val id = resultSet.getInt(1)
        val value = resultSet.getString(2)
        assertThat(id).isNotNull
        assertThat(value).isNotNull
      }
    }
  }
}
```

> On peut noter que la partie _before_ charge un fichier csv dans hdfs et le transforme en orc avec un job Spark (un simple read/write). En effet, il est plus aisé de lire un csv qu'un fichier orc pour un être humain... ;)
> 
> On peut également noter la suppression de la table Hive créée sur la partie _after_ qui utilise DbSetup _wrappé_ par Hadoop Unit.
> 
> Enfin, on peut constater que les job Spark utilisés ici sont bien en mode __local__.

Afin de permettre le démarrage d'Hadoop Unit en phase de pré-integration test et son arrêt en phase de post-integration test, la configuration maven est la suivante :
`pom.xml`
```xml

  <profiles>
    <profile>
      <id>windows</id>
      <activation>
        <os>
          <family>windows</family>
        </os>
      </activation>
      <properties>
        <suffix.test>"(?&lt;!Integration)(Test|Case|Suite|Spec)"</suffix.test>
        <suffix.it>"(?&lt;=Integration)(Test|Case|Suite|Spec)"</suffix.it>
      </properties>
    </profile>
    <profile>
      <id>default</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <suffix.test>(?&lt;!Integration)(Test|Case|Suite|Spec)</suffix.test>
        <suffix.it>(?&lt;=Integration)(Test|Case|Suite|Spec)</suffix.it>
      </properties>
    </profile>
  </profiles>

  <build>
    <plugins>
      <plugin>
        <groupId>org.scalatest</groupId>
        <artifactId>scalatest-maven-plugin</artifactId>
        <version>${scalatest.maven.plugin.version}</version>
        <executions>
          <execution>
            <id>IntegrationTest</id>
            <goals>
              <goal>test</goal>
            </goals>
            <phase>integration-test</phase>
            <configuration>
              <suffixes>${suffix.it}</suffixes>
            </configuration>
          </execution>
          <execution>
            <id>UnitTest</id>
            <goals>
              <goal>test</goal>
            </goals>
            <phase>test</phase>
            <configuration>
              <suffixes>${suffix.test}</suffixes>
            </configuration>
          </execution>
        </executions>
      </plugin>

      <plugin>
        <artifactId>hadoop-unit-maven-plugin</artifactId>
        <groupId>fr.jetoile.hadoop</groupId>
        <version>${hadoop-unit.version}</version>
        <executions>
          <execution>
            <id>start</id>
            <goals>
              <goal>embedded-start</goal>
            </goals>
            <phase>pre-integration-test</phase>
          </execution>
          <execution>
            <id>stop</id>
            <goals>
              <goal>embedded-stop</goal>
            </goals>
            <phase>post-integration-test</phase>
          </execution>
        </executions>
        <configuration>
          <components>
            <componentArtifact implementation="fr.jetoile.hadoopunit.ComponentArtifact">
              <componentName>HDFS</componentName>
              <artifact>fr.jetoile.hadoop:hadoop-unit-hdfs:${hadoop-unit.version}</artifact>
            </componentArtifact>
            <componentArtifact implementation="fr.jetoile.hadoopunit.ComponentArtifact">
              <componentName>ZOOKEEPER</componentName>
              <artifact>fr.jetoile.hadoop:hadoop-unit-zookeeper:${hadoop-unit.version}</artifact>
            </componentArtifact>
            <componentArtifact implementation="fr.jetoile.hadoopunit.ComponentArtifact">
              <componentName>HIVEMETA</componentName>
              <artifact>fr.jetoile.hadoop:hadoop-unit-hive:${hadoop-unit.version}</artifact>
            </componentArtifact>
            <componentArtifact implementation="fr.jetoile.hadoopunit.ComponentArtifact">
              <componentName>HIVESERVER2</componentName>
              <artifact>fr.jetoile.hadoop:hadoop-unit-hive:${hadoop-unit.version}</artifact>
            </componentArtifact>
            <componentArtifact implementation="fr.jetoile.hadoopunit.ComponentArtifact">
              <componentName>ELASTICSEARCH</componentName>
              <artifact>fr.jetoile.hadoop:hadoop-unit-elasticsearch:${hadoop-unit.version}</artifact>
              <properties>
                <elasticsearch.version>${elasticsearch.version}</elasticsearch.version>
                <elasticsearch.download.url>
                  https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/zip/elasticsearch/${elasticsearch.version}/elasticsearch-${elasticsearch.version}.zip
                </elasticsearch.download.url>
              </properties>
            </componentArtifact>
          </components>
        </configuration>
      </plugin>

      <plugin>
        <artifactId>maven-shade-plugin</artifactId>
        <version>${maven-shade-plugin.version}</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <filters>
                <filter>
                  <artifact>*:*</artifact>
                  <excludes>
                    <exclude>META-INF/*.SF</exclude>
                    <exclude>META-INF/*.DSA</exclude>
                    <exclude>META-INF/*.RSA</exclude>
                  </excludes>
                </filter>
              </filters>
              <shadedClassifierName>uber</shadedClassifierName>
              <shadedArtifactAttached>false</shadedArtifactAttached>
            </configuration>
          </execution>
        </executions>
      </plugin>

    </plugins>
  </build>
```

> On peut noter 3 choses dans ce `pom.xml` :
>
> * la configuration de tous ce qui est post-fixé par Integration(Test|Case|Suite|Spec) dans le scope d'integration test. Cela permet de séparer les tests unitaires des tests d'intégration
> * l'utilisation d'un _profile_ afin de corriger un bug sous windows obligeant à _quoter_ le suffixe lors de l'utilisation de scalatest
> * l'utilisation du plugin shade afin de générer un _fat jar_ ayant le nom de l'artéfact

> Ainsi, l'exécution de `mvn test` n'exécutera que les tests unitaires alors que les tests d'intégration seront exécutés dès la phase integration-test (par exemple avec `mvn install` ou `mvn verify`).

# Anatomie d'un livrable

Dans le paragraphe précédent, il a été montré comment il était possible de réaliser des tests d'intégrations sur un job Spark nécessitant la présence de Hdfs, du métastore Hive et d'ElasticSearch. 

Un _fat jar_ a également été produit.

Dans ce paragraphe, nous allons voir de quoi nous avons besoin pour permettre de le déployer sur le cluster.

En fait, lorsqu'il est décidé de tout orchestrer via Oozie, il devient nécessaire de pousser les jars (ou les scripts) dans un répertoire Hdfs donné mais également le workflow oozie (et éventuellement son/ses coordinateur(s)). Il est alors possible de _submitter_ le workflow (ou le coordinateur) au travers du serveur Oozie en lui précisant un fichier `job.properties`.

Ainsi, dans notre cas, le livrable sera constitué de :

* un jar (job Spark)
* un workflow.xml référençant le jar dans un répertoire Hdfs
* un coordinateur référençant le workflow dans un répertoire Hdfs
* un job.properties permettant d'exécuter le coordinateur
* un job.properties permettant d'exécuter le workflow (parce qu'on ne sait jamais si le job doit être exécuté manuellement)

De plus, si il existe différent environnement (ie. developpement, recette ou production), il est préférable de variabiliser les choses spécifiques aux environnements (ex: répertoire Hdfs, nom de database Hive, nom d'index ElasticSearch, ...)

Ainsi, il peut être imaginé le format suivant :

```bash
├── dist
│   ├── lib
│   │   └── bigdata-sample-job-1.0-SNAPSHOT.jar
│   └── oozie
│       ├── coordinator
│       │   └── sample
│       │       └── coordinator.xml
│       ├── run
│       │   ├── coordinator
│       │   │   └── sample
│       │   │       └── job.properties
│       │   └── workflow
│       │       └── sample
│       │           └── job.properties
│       └── workflow
│           └── sample
│               └── workflow.xml
```

où le `workflow.xml` pourrait être le suivant :

```xml
<workflow-app name="sample" xmlns="uri:oozie:workflow:0.5">
  <global>
    <configuration>
      <property>
        <name>mapred.job.queue.name</name>
        <value>queue</value>
      </property>
    </configuration>
  </global>
  <start to="start"/>
  <kill name="kill">
    <message>Action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
  </kill>
  <action name="start">
    <spark xmlns="uri:oozie:spark-action:0.2">
      <job-tracker>${jobTracker}</job-tracker>
      <name-node>${nameNode}</name-node>
      <master>yarn</master>
      <mode>cluster</mode>
      <name>sampleSpark</name>
      <class>fr.jetoile.hadoop.sample.Main</class>
      <jar>${nameNode}//user/myuser/projects/share/lib/bigdata-sample-job-1.0-SNAPSHOT.jar</jar>
      <arg>-p</arg>
      <arg>hdfs://namenode/mypath</arg>
      <arg>-i</arg>
      <arg>sampleIndex</arg>
      <arg>-d</arg>
      <arg>sampleDoc</arg>
    </spark>
    <ok to="End"/>
    <error to="kill"/>
  </action>
  <end name="End"/>
</workflow-app>
```

et le `job.properties` servant à l'exécuter :

```bash
oozie.use.system.libpath=True
send_email=False
dryrun=False
security_enabled=False
nameNode=hdfs://namenode
jobTracker=jobtracker:8050
oozie.wf.application.path=hdfs://namenode/user/myuser/projects/oozie/workflow/sample/
```

Par contre, savoir ce qui doit être déployé sur Hdfs est intéressant mais il faut ensuite réaliser le déploiement en lui même.

Ainsi, dans notre exemple, il faut :

* copier le fat jar dans le répertoire Hdfs `/user/myuser/projects/share/lib/` au travers de Knox (l'accès WebHdfs ou le client Hdfs n'étant souvent pas accessible pour des raisons de sécurité)
* copier le workflow.xml dans le répertoire Hdfs `/user/myuser/projects/oozie/workflow/sample` au travers de Knox (l'accès WebHdfs ou le client Hdfs n'étant souvent pas accessible pour des raisons de sécurité)
* exécuter le `workflow.xml` via la submission du fichier `job.properties` au Server Oozie au travers de Knox (l'accès au client Oozie ou à l'API Rest de Oozie n'étant souvent pas accessible pour des raisons de sécurité)

Bien sûr, il est possible d'exécuter une suite de `curl` afin de permettre cela. Cependant, cela peut vite s'avérer réberbatif et, surtout, source d'erreur... 

> Pour rappel, par exemple, la copie d'un fichier sur Hdfs au travers de WebHdfs ou Knox se fait au travers 2 appels http (un premier PUT avec l'option CREATE qui renvoie dans le header une _location_ ainsi qu'un code retour 307 puis un second PUT avec le fichier à l'url retournée dans le header _location_ de l'appel précédent). 

Ainsi, il peut être intéressant de regarder le composant __gateway shell__ fournit par Knox et qui offre la possibilité d'écrire un script Groovy (ou un programme Java/Scala) permettant de piloter ces opérations de manière un peu plus _lisible_ (une copie de fichier sur Hdfs devient `Hdfs.put(session).file(<fichier>).to(<path hdfs>).now`). 

> Il devient alors possible de fournir un ensemble de méthodes statiques utilisables, par exemple, par un script groovy où serait décrit la procédure de déploiement. Ces méthodes statiques pourraient alors faire un ensemble de vérification (comme, par exemple, vérifier qu'il n'existe pas déjà les fichiers dans Hdfs et faire un renommage dans le cas contraire)

Ce script de déploiement pourrait être le suivant :
```groovy
deployOnHdfs("dist/lib", "/user/myuser/projects/share/lib/", config, options)
deployOnHdfs("dist/oozie", "/user/myuser/projects/oozie/", config, options)

runOozieJobs("dist/oozie/run/workflow/sample/job.properties", config, options)

//deleteOnHdfs("/user/myuser/projects/share/lib", config, options)

//def w = checkOozieCoordinatorStatus(config, options, "status=RUNNING")
//killOrExitCoordinators("SAMPLE COORD", w, options, config)
```

Ainsi, il a été défini ce qu'était un livrable qui est donc constitué de :

* un jar 
* un ensemble de fichiers oozie
* un script de déploiement indiquant quoi copier où et ce qu'il doit exécuter.

> A noter que Knox propose la même API que WebHdfs et que Oozie. Il est donc possible, au besoin, d'attaquer directement WebHdfs et le serveur Oozie (pour rappel, le client Oozie ne fait qu'invoquer des _endpoints_ Rest).

# Mise en oeuvre

Afin de mettre en oeuvre ce qui a été décrit dans les paragraphes précédents mais aussi de rendre réutilisable et un peu plus industrialisable les choses, il est possible de découper les choses en 3 repositories :

* un repository (__parent__) ne contenant que la configuration maven qui sera hérité par les projets (versions des archetypes configurées en _dependencyManagement_, configuration des plugins (scala en l'occurence et séparation tests unitaires/tests d'intégrations), ...)
* un repository (__commons__) contenant 2 modules maven :
  * un module (__commons-conf__) contenant la configuration globale (namenode, resource manager, knox, hivemetastore, ...) du cluster hadoop par environnement (developpement, recette, production)
  * un module (__commons-deploy__) contenant les méthodes statiques groovy utilisées dans les scripts de déploiement
* un repository (__sample__) qui contenant au moins 2 modules maven :
  * un module (__sample-conf__) contenant les configurations  spécifiques projet (configuration, workflow, coordinator, ...) ainsi que les scripts de déploimenet permettant un déploiement sur le cluster
  * un ou des module(s) (__sample-job__) contenant le code des jobs spark ou autre

De plus, afin de permettre d'exécuter le script de déploiment (écrit dans groovy dans notre cas), il faut proposer un script shell (type sh) qui appelle un programme Java encapsulant le moteur Groovy. En effet, il est rare que Groovy soit installé sur les différents environnements.

La configuration Maven reste simple pour tous les repositories si ce n'est pour la partie __sample-conf__ qui a la responsabilité de construire le livrable.

Pour ce faire, ce module doit :

* dépendre de __commons-deploy__ afin de permettre au script de déploiement d'avoir les méthodes statiques encapsulant la __gateway shell__ de Knox
* récupérer __commons-conf__, dézipper le jar et mettre à disposition son contenu afin de pouvoir remplacer les variables des différents fichiers
* remplacer les variables des fichiers par les configurations spécifiques projet et global
* générer le script sh appelant la classe Java permettant d'exécuter le script groovy
* générer le livrable (au format tar.gz par exemple)

Enfin, cerise sur le gateau, il peut également être utile de positionner les __start_date__ des coordinateurs Oozie à la date de génération du livrable.

Pour ce faire, une simple combinaison des bons plugins maven permet d'obtenir le résultat escompté :

```xml
  <build>
    <resources>
      <resource>
        <directory>src/main/resources</directory>
        <filtering>true</filtering>
      </resource>
    </resources>
    <plugins>
      <plugin>
        <artifactId>maven-dependency-plugin</artifactId>
        <executions>
          <execution>
            <id>unpack</id>
            <phase>generate-resources</phase>
            <goals>
              <goal>unpack</goal>
            </goals>
            <configuration>
              <artifactItems>
                <artifactItem>
                  <groupId>fr.jetoile.hadoop.sample</groupId>
                  <artifactId>bigdata-commons-conf</artifactId>
                  <type>jar</type>
                  <overWrite>false</overWrite>
                  <outputDirectory>${project.build.directory}/global-conf</outputDirectory>
                </artifactItem>
              </artifactItems>
            </configuration>
          </execution>
        </executions>
      </plugin>

      <plugin>
        <groupId>org.codehaus.gmaven</groupId>
        <artifactId>groovy-maven-plugin</artifactId>
        <dependencies>
          <dependency>
            <groupId>org.codehaus.groovy</groupId>
            <artifactId>groovy-all</artifactId>
            <version>${groovy-all.version}</version>
          </dependency>
        </dependencies>
      </plugin>

      <plugin>
        <artifactId>maven-resources-plugin</artifactId>
        <executions>
          <execution>
            <id>dev-resources</id>
            <phase>process-resources</phase>
            <goals>
              <goal>resources</goal>
            </goals>
            <configuration>
              <outputDirectory>${project.build.outputDirectory}/dev</outputDirectory>
              <filters>
                <filter>${basedir}/src/main/filters/dev/conf.properties</filter>
                <filter>${project.build.directory}/global-conf/dev/conf.properties</filter>
              </filters>
            </configuration>
          </execution>
          <execution>
            <id>prod-resources</id>
            <phase>process-resources</phase>
            <goals>
              <goal>resources</goal>
            </goals>
            <configuration>
              <outputDirectory>${project.build.outputDirectory}/prod</outputDirectory>
              <filters>
                <filter>${basedir}/src/main/filters/prd/conf.properties</filter>
                <filter>${project.build.directory}/global-conf/prd/conf.properties</filter>
              </filters>
            </configuration>
          </execution>
          <execution>
            <id>test-resources</id>
            <phase>process-resources</phase>
            <goals>
              <goal>resources</goal>
            </goals>
            <configuration>
              <outputDirectory>${project.build.outputDirectory}/test</outputDirectory>
              <filters>
                <filter>${basedir}/src/main/filters/test/conf.properties</filter>
                <filter>${project.build.directory}/global-conf/test/conf.properties</filter>
              </filters>
            </configuration>
          </execution>
        </executions>
      </plugin>

      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>appassembler-maven-plugin</artifactId>
        <version>1.10</version>
        <executions>
          <execution>
            <goals>
              <goal>assemble</goal>
            </goals>
            <phase>package</phase>
          </execution>
        </executions>
        <configuration>
          <configurationDirectory>conf</configurationDirectory>
          <programs>
            <program>
              <mainClass>Main</mainClass>
              <name>main</name>
            </program>
          </programs>
          <repositoryLayout>flat</repositoryLayout>
          <binFileExtensions>
            <unix>.sh</unix>
          </binFileExtensions>
        </configuration>
      </plugin>

      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <executions>
          <execution>
            <id>deploy-dev</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
            <configuration>
              <descriptors>
                <descriptor>src/main/assembly/descriptor-deploy-run-dev.xml</descriptor>
              </descriptors>
            </configuration>
          </execution>
          <execution>
            <id>deploy-prd</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
            <configuration>
              <descriptors>
                <descriptor>src/main/assembly/descriptor-deploy-run-prd.xml</descriptor>
              </descriptors>
            </configuration>
          </execution>
          <execution>
            <id>deploy-test</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
            <configuration>
              <descriptors>
                <descriptor>src/main/assembly/descriptor-deploy-run-test.xml</descriptor>
              </descriptors>
            </configuration>
          </execution>

        </executions>
      </plugin>


      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>buildnumber-maven-plugin</artifactId>
        <version>1.4</version>
        <executions>
          <execution>
            <phase>validate</phase>
            <goals>
              <goal>create-timestamp</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <timestampFormat>yyyy-MM-dd'T'HH:mmZ</timestampFormat>
          <timestampPropertyName>build.date</timestampPropertyName>
        </configuration>
      </plugin>

    </plugins>
  </build>
```

> A noter la configuration du plugin _buildnumber-maven-plugin_ afin d'avoir la valeur de __start_date__ au format attendu par Oozie.

> A noter l'utilisation du plugin _appassembler-maven-plugin_ permettant la génération du script shell et qui permet de gérer, entre autre, automatiquement la récupération des dépendances nécessaires à l'exécution de la classe Java.

Afin de permettre la génération tar.gz du livrable, un assembly maven est utilisable :

```xml
<?xml version="1.0"?>
<assembly xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2 http://maven.apache.org/xsd/assembly-1.1.2.xsd">

  <id>deploy-dev</id>

  <formats>
    <format>tar.gz</format>
  </formats>

  <fileSets>
    <fileSet>
      <!-- all files to deploy have to be into dist directory -->
      <directory>${project.build.outputDirectory}/dev/</directory>
      <outputDirectory>/dist/</outputDirectory>
      <excludes>
        <exclude>deploy.properties</exclude>
      </excludes>
      <lineEnding>unix</lineEnding>
    </fileSet>

    <fileSet>
      <directory>${project.build.directory}/appassembler</directory>
      <outputDirectory>/</outputDirectory>
      <fileMode>750</fileMode>
      <directoryMode>750</directoryMode>
    </fileSet>
  </fileSets>

  <dependencySets>
    <dependencySet>
      <!-- all files to deploy have to be into dist directory -->
      <outputDirectory>/dist/lib/</outputDirectory>
      <includes>
        <include>
          fr.jetoile.hadoop.sample:bigdata-sample-job
        </include>
      </includes>
      <useProjectArtifact>false</useProjectArtifact>
      <scope>provided</scope>
      <unpack>false</unpack>
    </dependencySet>
  </dependencySets>

  <files>
    <file>
      <source>src/main/groovy/deploy.groovy</source>
      <outputDirectory>bin</outputDirectory>
    </file>

    <file>
      <source>${project.build.directory}/classes/dev/deploy.properties</source>
      <outputDirectory>/</outputDirectory>
    </file>
  </files>

</assembly>
```

> A noter que, afin de récupérer le jar contenant le job Spark, la dépendance a été mis en scopte _provided_. En effet, le plugin _appassembler-maven-plugin_ prend tout ce qu'il trouve en scope compile et aurait embarqué, par défaut, le jar du job entrainant des conflits de classpath.

# Conclusion

On a vu dans cet article comme il était simple de tester son code et de définir et construire un livrable juste en structurant son code et en utilisant quelques bons plugin maven.

Bien sûr, si cela est faisable en Maven, il est tout à fait possible d'utiliser un autre outils de build pour le faire. 

Alors, oui, l'approche est peut être un peu simpliste et naïve mais elle a au moins le mérite d'exister et de proposer à moindre coût quelque chose de facilement généralisable au sein des différents projets/équipes réalisant des développements dans l'écosystème Hadoop.

En outre, le script de déploiement peut également être utilisé pour tester rapidement son code sur le cluster de développement.

Enfin, les 2 autres avantages non négligeables sont que :

* le livrable est versionné dans le _repository manager_ (tel que Nexus ou Archiva)
* une chaine CI classique (tel que Jenkins) peut s'occuper du déploiement au besoin

> A noter que le code se trouve dans le [github ci-joint](https://github.com/jetoile/bigdata-sample-parent) et qu'il a été mis, pour des questions de simplicité, dans un seul repository (cela explique qu'on ait un pom parent qui ne fait que déclarer les modules fils mais que ces derniers ne le référencent pas).

> A noter également qu'un archetype permettant de générer la structure du projet sample est présent (__bigdata-archetype__).


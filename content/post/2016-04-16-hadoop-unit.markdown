---
title: "Hadoop Unit"
date: 2016-04-16 18:16:01 +0200
comments: true
tags:
- Hadoop
---
![left-small](/images/hadoop-all.png)
Dans mon dernier [post](/2016/01/hadoop-ecosysteme-bootstrap.html), j'avais parlé d'une surcouche que j'avais développé afin de faciliter l'utilisation de quelques-uns des composants de l'écosystème Hadoop, à savoir :

* Hdfs,
* Zookeeper,
* HiveMetastore,
* Hiveserver2,
* SolR,
* SolRCloud,
* Oozie,
* Kafka,
* HBase,
* MongoDB [New \o/ ],
* et Cassandra [New \o/ ].

Il s'appelait alors Hadoop-Bootstrap mais il s'agissait aussi d'une première version qui a, bien sûr, évolué.

Cet article présentera donc quels ont été les améliorations qui ont été apportées.

_Disclaimer_ : je tiens à repréciser que Hadoop-unit n'est qu'une solution de contournement permettant de simuler une partie de l'écosystème Hadoop afin de permettre de disposer en local d'un ersatz de distribution afin de fluidifier le développement mais proposant aussi d'effectuer des tests d'intégration dans un environnement dégradé. Cela peut également permettre d'éviter de monter un cluster Hadoop dédié aux tests d'intégration.

<!-- more -->

#Description des changements/évolutions

Alors que la première version d'Hadoop Unit (aka. Hadoop Bootstrap) était monolithique, le projet a été découpé en modules permettant ainsi de ne démarrer que les composants qui sont présents dans le classpath de l'environnement de test (en maven le scope de test).

Ainsi, par exemple, si l'application nécessite HBase (qui nécessite lui-même Hdfs et Zookeeper), il n'y a qu'à ajouter au classpath de test la dépendance à HBase.

Dans ce cas, grâce à la transitivité Maven, les modules Zookeeper et Hdfs seront automatiquement ajoutés au classpath de test et les composants démarrés avant HBase.

En outre, cela permet de ne pas à avoir à se soucier de l'ordre de démarrage.

Pour information, afin de permettre ce mécanisme, il y a dernier un simple SPI.

Ainsi dans un exemple, cela peut donner :

```xml
<dependency>
    <groupId>fr.jetoile.hadoop</groupId>
    <artifactId>hadoop-unit-hbase</artifactId>
    <version>1.2-SNAPSHOT</version>
    <scope>test</scope>
</dependency>
```

avec dans le test d'intégration :
```java
@BeforeClass
public static void setup() {
    HadoopBootstrap.INSTANCE.startAll();
}

@AfterClass
public static void tearDown() {
    HadoopBootstrap.INSTANCE.stopAll();
}

@Test
public void hBaseShouldStart() throws Exception {

    String tableName = configuration.getString(Config.HBASE_TEST_TABLE_NAME_KEY);
    String colFamName = configuration.getString(Config.HBASE_TEST_COL_FAMILY_NAME_KEY);
    String colQualiferName = configuration.getString(Config.HBASE_TEST_COL_QUALIFIER_NAME_KEY);
    Integer numRowsToPut = configuration.getInt(Config.HBASE_TEST_NUM_ROWS_TO_PUT_KEY);

    org.apache.hadoop.conf.Configuration hbaseConfiguration = HBaseConfiguration.create();
    hbaseConfiguration.set("hbase.zookeeper.quorum", configuration.getString(Config.ZOOKEEPER_HOST_KEY));
    hbaseConfiguration.setInt("hbase.zookeeper.property.clientPort", configuration.getInt(Config.ZOOKEEPER_PORT_KEY));
    hbaseConfiguration.set("hbase.master", "127.0.0.1:" + configuration.getInt(Config.HBASE_MASTER_PORT_KEY));
    hbaseConfiguration.set("zookeeper.znode.parent", configuration.getString(Config.HBASE_ZNODE_PARENT_KEY));

    LOGGER.info("HBASE: Creating table {} with column family {}", tableName, colFamName);
    createHbaseTable(tableName, colFamName, hbaseConfiguration);

    LOGGER.info("HBASE: Populate the table with {} rows.", numRowsToPut);
    for (int i = 0; i < numRowsToPut; i++) {
        putRow(tableName, colFamName, String.valueOf(i), colQualiferName, "row_" + i, hbaseConfiguration);
    }

    LOGGER.info("HBASE: Fetching and comparing the results");
    for (int i = 0; i < numRowsToPut; i++) {
        Result result = getRow(tableName, colFamName, String.valueOf(i), colQualiferName, hbaseConfiguration);
        assertEquals("row_" + i, new String(result.value()));
    }
}
```

Bien sûr, il reste possible de préciser les composants à démarrer mais, dans ce cas, il est à la charge du développeur de vérifier que les modules sont bien dans son classpath et qu'il démarre bien les composants dans le bon ordre.

```java
@BeforeClass
public static void setup() throws NotFoundServiceException {
    HadoopBootstrap.INSTANCE
        .start(Component.ZOOKEEPER)
        .start(Component.HDFS)
        .start(Component.HBASE)
        .startAll();
}

@AfterClass
public static void tearDown() throws NotFoundServiceException {
    HadoopBootstrap.INSTANCE
        .stopAll();
}
```
Enfin, il a été ajouté un plugin Maven permettant de démarrer les composants de l'écosystème :

* soit dans le même classloader que le test,
* soit de démarrer hadoop unit dans une autre jvm modulo que hadoop-unit-standalone ait été installé sur le poste en local.

Ce dernier mode est celui à privilégier dans les tests d'intégration car cela évite les problèmes de conflits de dépendances engendrés par le fait que tous les composants d'Hadoop Unit s'éxécute dans la même JVM (chose qui bien sûr n'est pas le cas dans le vraie vie...).

Ainsi, en mode embedded, le plugin peut s'utiliser de la manière suivante :
```xml
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.11</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>fr.jetoile.hadoop</groupId>
        <artifactId>hadoop-unit-hdfs</artifactId>
        <version>1.2-SNAPSHOT</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>fr.jetoile.hadoop</groupId>
        <artifactId>hadoop-unit-client-hdfs</artifactId>
        <version>1.2-SNAPSHOT</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>**/*IntegrationTest.java</exclude>
                </excludes>
            </configuration>
            <executions>
                <execution>
                    <id>integration-test</id>
                    <goals>
                        <goal>test</goal>
                    </goals>
                    <phase>integration-test</phase>
                    <configuration>
                        <excludes>
                            <exclude>none</exclude>
                        </excludes>
                        <includes>
                            <include>**/*IntegrationTest.java</include>
                        </includes>
                    </configuration>
                </execution>
            </executions>
        </plugin>

        <plugin>
            <artifactId>hadoop-unit-maven-plugin</artifactId>
            <groupId>fr.jetoile.hadoop</groupId>
            <version>1.2-SNAPSHOT</version>
            <executions>
                <execution>
                    <id>start</id>
                    <goals>
                        <goal>embedded-start</goal>
                    </goals>
                    <phase>pre-integration-test</phase>
                </execution>
            </executions>
            <configuration>
                <values>
                    <value>HDFS</value>
                </values>
            </configuration>
        </plugin>

    </plugins>
</build>
```

avec :
```java
public class HdfsBootstrapIntegrationTest {

    static private Configuration configuration;

    @BeforeClass
    public static void setup() throws BootstrapException {
        try {
            configuration = new PropertiesConfiguration("default.properties");
        } catch (ConfigurationException e) {
            throw new BootstrapException("bad config", e);
        }
    }

    @Test
    public void hdfsShouldStart() throws Exception {
        FileSystem hdfsFsHandle = HdfsUtils.INSTANCE.getFileSystem();

        FSDataOutputStream writer = hdfsFsHandle.create(new Path(configuration.getString(Config.HDFS_TEST_FILE_KEY)));
        writer.writeUTF(configuration.getString(Config.HDFS_TEST_STRING_KEY));
        writer.close();

        // Read the file and compare to test string
        FSDataInputStream reader = hdfsFsHandle.open(new Path(configuration.getString(Config.HDFS_TEST_FILE_KEY)));
        assertEquals(reader.readUTF(), configuration.getString(Config.HDFS_TEST_STRING_KEY));
        reader.close();
        hdfsFsHandle.close();

        URL url = new URL(String.format( "http://localhost:%s/webhdfs/v1?op=GETHOMEDIRECTORY&user.name=guest",
                        configuration.getInt( Config.HDFS_NAMENODE_HTTP_PORT_KEY ) ) );
        URLConnection connection = url.openConnection();
        connection.setRequestProperty( "Accept-Charset", "UTF-8" );
        BufferedReader response = new BufferedReader( new InputStreamReader( connection.getInputStream() ) );
        String line = response.readLine();
        response.close();
        assertThat("{\"Path\":\"/user/guest\"}").isEqualTo(line);
    }
}
```
En utilisation remote, cela donne :
```xml
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.11</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>fr.jetoile.hadoop</groupId>
        <artifactId>hadoop-unit-client-hdfs</artifactId>
        <version>1.2-SNAPSHOT</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
    </configuration>
    <executions>
        <execution>
            <id>integration-test</id>
            <goals>
                <goal>test</goal>
            </goals>
            <phase>integration-test</phase>
            <configuration>
                <excludes>
                    <exclude>none</exclude>
                </excludes>
                <includes>
                    <include>**/*IntegrationTest.java</include>
                </includes>
            </configuration>
        </execution>
    </executions>
</plugin>

<plugin>
    <artifactId>hadoop-unit-maven-plugin</artifactId>
    <groupId>fr.jetoile.hadoop</groupId>
    <version>1.2-SNAPSHOT</version>
    <executions>
        <execution>
            <id>start</id>
            <goals>
                <goal>start</goal>
            </goals>
            <phase>pre-integration-test</phase>
        </execution>
    </executions>
    <configuration>
        <hadoopUnitPath>/home/khanh/tools/hadoop-unit-standalone</hadoopUnitPath>
        <exec>./hadoop-unit-standalone</exec>
        <values>
            <value>HDFS</value>         
        </values>
        <outputFile>/tmp/toto.txt</outputFile>
    </configuration>
</plugin>

<plugin>
    <artifactId>hadoop-unit-maven-plugin</artifactId>
    <groupId>fr.jetoile.hadoop</groupId>
    <version>1.2-SNAPSHOT</version>
    <executions>
        <execution>
            <id>stop</id>
            <goals>
                <goal>stop</goal>
            </goals>
            <phase>post-integration-test</phase>
        </execution>
    </executions>
    <configuration>
        <hadoopUnitPath>/home/khanh/tools/hadoop-unit-standalone</hadoopUnitPath>
        <exec>./hadoop-unit-standalone</exec>
        <outputFile>/tmp/toto.txt</outputFile>
    </configuration>
</plugin>
```

#Trucs et astuces

Il est possible d'intéragir avec Hadoop Unit (et même si il s'agit d'un cluster Hadoop dégradé) avec les moyens standards, à savoir :

* hbase shell
* kafka-console commande
* hdfs commande
* hive shell

Ainsi, il devient possible de faire des choses comme :
```bash
kafka-console-consumer --zookeeper localhost:22010 --topic topic
hdfs dfs -ls hdfs://localhost:20112/
```
ou lancer un hbase shell ou hive shell pour créer des tables ou des tables hive et/ou peupler et/ou requêter ces derniers.


Hadoop Unit propose également des classes utilitaires (à n'utiliser que dans le contexte des tests bien sûr!) qui lisent directement la configuration d'Hadoop Unit (le fichier `default.properties`) et qui permettent de bénéficier simplement :

* d'un FileSystem Hdfs
* d'avoir une syntaxe fluent pour créer les tables hive via jdbc (cette partie est directement tirée de [DBSetup](http://dbsetup.ninja-squad.com/) de [Ninja Squad](http://ninja-squad.com/))
* des bonnes dépendances pour utiliser Spark 1.6.0 (cf. conflit de lib)
* des bonnes dépendances pour utiliser Spark-solr (cf. conflit de lib)


#Conclusion
Comme je l'ai déjà dit maintes et maintes fois, Hadoop Unit n'est qu'une solution de contournement (ou la solution du pauvre au choix... ;) ) mais cela peut permettre d'accélérer les phases de développement (en utilisant hadoop-unit-standalone par exemple) et peut permettre d'effectuer quelques tests d'intégration.

Hadoop Unit s'appuie sur le travail de [Shane Kumpf](https://github.com/sakserv/hadoop-mini-clusters) et est lié aux versions des composants d'Hortonworks (d'ailleurs, une grosse partie des tests est directement un copier/coller de ce qu'à fait Shane Kumpf).

Il ne propose ni Spark ni Map/Reduce car ces derniers n'ont pas besoin d'être simulés puisqu'ils proposent un mode de développement.

Enfin, pour plus d'informations sur l'utilisation d'Hadoop-Unit, je vous invite à aller sur la page du projet [github](https://github.com/jetoile/hadoop-unit) et/ou de faire des Pull Request si vous avez des remarques/bugs et/ou à forker si cela vous convient pas.

Pour conclure, et même si cela se trouve sur github, voilà un exemple de ce qu'il est possible de faire avec Hadoop Unit :

* Exemple 1
  * émettre des données dans un topic Kafka
  * consommer les données en utilisant Spark Streaming
  * écrire les résultats dans HBase et/ou SolrCloud et/ou HDFS

* Exemple 2
  * upload dans hdfs un fichier au format csv
  * créer une table hive associée
  * transformer via Spark SQL le csv en parquet (en utilisant le métastore de Hive)
  * indexer les données au format parquet dans SolrCloud en utilisant Spark

soit :

```java
public class SparkSolrIntegrationTest {
    static private Logger LOGGER = LoggerFactory.getLogger(SparkSolrIntegrationTest.class);
    private static Configuration configuration;

    public static Operation CREATE_TABLES = null;
    public static Operation DROP_TABLES = null;

    @BeforeClass
    public static void setUp() throws BootstrapException, SQLException, ClassNotFoundException, NotFoundServiceException {
        try {
            configuration = new PropertiesConfiguration("default.properties");
        } catch (ConfigurationException e) {
            throw new BootstrapException("bad config", e);
        }

        CREATE_TABLES =
                sequenceOf(sql("CREATE EXTERNAL TABLE IF NOT EXISTS default.test(id INT, value STRING) " +
                                " ROW FORMAT DELIMITED FIELDS TERMINATED BY ';'" +
                                " STORED AS TEXTFILE" +
                                " LOCATION 'hdfs://localhost:" + configuration.getInt(Config.HDFS_NAMENODE_PORT_KEY) + "/khanh/test'"),

        DROP_TABLES =
                sequenceOf(sql("DROP TABLE IF EXISTS default.test"));
    }

    @Before
    public void before() throws IOException, URISyntaxException {
        FileSystem fileSystem = HdfsUtils.INSTANCE.getFileSystem();

        fileSystem.mkdirs(new Path("/khanh/test"));
        fileSystem.mkdirs(new Path("/khanh/test_parquet"));
        fileSystem.copyFromLocalFile(new Path(SparkSolrIntegrationTest.class.getClassLoader().getResource("test.csv").toURI()), new Path("/khanh/test/test.csv"));

        new HiveSetup(HiveConnectionUtils.INSTANCE.getDestination(), Operations.sequenceOf(CREATE_TABLES)).launch();
    }

    @After
    public void clean() throws IOException {
        new HiveSetup(HiveConnectionUtils.INSTANCE.getDestination(), Operations.sequenceOf(DROP_TABLES)).launch();

        FileSystem fileSystem = HdfsUtils.INSTANCE.getFileSystem();
        fileSystem.delete(new Path("/khanh"), true);
    }


    @Test
    public void spark_should_read_parquet_file_and_index_into_solr() throws IOException, SolrServerException {
        //given
        SparkConf conf = new SparkConf()
                .setMaster("local[*]")
                .setAppName("test");

        JavaSparkContext context = new JavaSparkContext(conf);

        //read hive-site from classpath
        HiveContext hiveContext = new HiveContext(context.sc());

        DataFrame sql = hiveContext.sql("SELECT * FROM default.test");
        sql.write().parquet("hdfs://localhost:" + configuration.getInt(Config.HDFS_NAMENODE_PORT_KEY) + "/khanh/test_parquet/file.parquet");

        context.close();

        //when
        context = new JavaSparkContext(conf);
        SQLContext sqlContext = new SQLContext(context);

        DataFrame file = sqlContext.read().parquet("hdfs://localhost:" + configuration.getInt(Config.HDFS_NAMENODE_PORT_KEY) + "/khanh/test_parquet/file.parquet");
        DataFrame select = file.select("id", "value");

        JavaRDD<SolrInputDocument> solrInputDocument = select.toJavaRDD().map(r -> {
            SolrInputDocument solr = new SolrInputDocument();
            solr.setField("id", r.getInt(0));
            solr.setField("value_s", r.getString(1));
            return solr;
        });

        String collectionName = configuration.getString(SolrCloudBootstrap.SOLR_COLLECTION_NAME);
        String zkHostString = configuration.getString(Config.ZOOKEEPER_HOST_KEY) + ":" + configuration.getInt(Config.ZOOKEEPER_PORT_KEY);
        SolrSupport.indexDocs(zkHostString, collectionName, 1000, solrInputDocument);

        //then
        CloudSolrClient client = new CloudSolrClient(zkHostString);
        SolrDocument collection1 = client.getById(collectionName, "1");

        assertNotNull(collection1);
        assertThat(collection1.getFieldValue("value_s")).isEqualTo("value1");

        client.close();
        context.close();
    }

}
```

---
title: "Des tests d'intégration avec Elasticsearch"
date: 2017-07-11 10:05:51 +0200
comments: true
tags: 
- Elasticsearch
- Test
---
![left-small](/images/logo-elastic.png) La version 5.0.0-alpha4 a signé la fin du support du mode embedded d'Elasticsearch.

Cela a été annoncé [là](https://www.elastic.co/blog/elasticsearch-the-server#_embedded_elasticsearch_not_supported) et la classe `NodeBuilder` permettant de démarrer un noeud programmatiquement a été supprimée.

Cependant, même si la raison de l'arrêt du support de ce mode est compréhensible, cela pose le problème des tests d'intégration puisqu'il n'est plus possible de démarrer un Elasticsearch pendant la phase de test.

Oui, Elastic propose officiellement une [alternative](https://www.elastic.co/guide/en/elasticsearch/reference/current/testing-framework.html) via l'utilisation de ESIntegTestCase mais personnellement, je ne suis pas très fan de cette approche...

Cet article va tenter de dresser un panorama non exhaustif de ce que j'ai pu trouver d'intéressant pour permettre de réaliser des tests d'intégration avec Elasticsearch.

<!--more-->

Parmi les solutions intéressantes et simples que j'ai trouvés pour faire des tests d'intégration, il y a surtout 2 projets que j'ai retenus.

#Plugin maven permettant de télécharger, installer et démarrer Elasticsearch

Le premier se trouve être celui de [alexcojocaru](https://github.com/alexcojocaru/elasticsearch-maven-plugin).

Il s'agit d'un plugin maven s'appuyant sur [maven-resolver](https://github.com/apache/maven-resolver) qui permet sur les phases de pré-intégration et post-intégration de démarrer et d'arrêter Elasticsearch.

```xml
<plugin>
    <groupId>com.github.alexcojocaru</groupId>
    <artifactId>elasticsearch-maven-plugin</artifactId>
    <version>5.7</version>
    <configuration>
        <clusterName>elasticsearch</clusterName>
        <transportPort>9300</transportPort>
        <httpPort>9200</httpPort>
        <autoCreateIndex>true</autoCreateIndex>
    </configuration>
    <executions>
        <execution>
            <id>start-elasticsearch</id>
            <phase>pre-integration-test</phase>
            <goals>
                <goal>runforked</goal>
            </goals>
        </execution>
        <execution>
            <id>stop-elasticsearch</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>stop</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

##Exemple

Code java de test :
```java
public class ElasticsearchIntegrationTest {

    @Test
    public void transportClient_should_success() throws IOException, InterruptedException {
        TransportClient client = new PreBuiltTransportClient(Settings.EMPTY)
                .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("localhost"), 9300));

        ObjectMapper mapper = new ObjectMapper();

        Sample sample = new Sample("value", 0.33, 3);

        String jsonString = mapper.writeValueAsString(sample);

        // indexing document
        IndexResponse ir = client.prepareIndex("test_index", "type").setSource(jsonString).setId("1").execute().actionGet();
        client.admin().indices().prepareRefresh("test_index").execute().actionGet();

        assertNotNull(ir);

        GetResponse gr = client.prepareGet("test_index", "type", "1").execute().actionGet();

        assertNotNull(gr);
        assertEquals(gr.getSourceAsString(), "{\"value\":\"value\",\"size\":0.33,\"price\":3.0}");
    }

    @Test
    public void restClient_should_success() throws IOException, JSONException {

        RestClient restClient = RestClient.builder(
                new HttpHost("localhost", 9200, "http")).build();

        org.elasticsearch.client.Response response = restClient.performRequest("GET", "/",
                Collections.singletonMap("pretty", "true"));
        System.out.println(EntityUtils.toString(response.getEntity()));

        // indexing document
        HttpEntity entity = new NStringEntity(
                "{\n" +
                        "    \"user\" : \"kimchy\",\n" +
                        "    \"post_date\" : \"2009-11-15T14:12:12\",\n" +
                        "    \"message\" : \"trying out Elasticsearch\"\n" +
                        "}", ContentType.APPLICATION_JSON);

        org.elasticsearch.client.Response indexResponse = restClient.performRequest(
                "PUT",
                "/twitter/tweet/1",
                Collections.<String, String>emptyMap(),
                entity);

        response = restClient.performRequest("GET", "/_search",
                Collections.singletonMap("pretty", "true"));

        String result = EntityUtils.toString(response.getEntity());
        System.out.println(result);
        JSONObject obj = new JSONObject(result);
        int nbResult = obj.getJSONObject("hits").getInt("total");
        assertThat(nbResult).isEqualTo(1);

        restClient.close();
    }

}

@EqualsAndHashCode
@Data
@AllArgsConstructor
class Sample implements Serializable {
    private String value;
    private double size;
    private double price;
}
```

Dépendence maven (façon gradle) :
```plain
dependencies {
    compile group: 'org.apache.logging.log4j', name: 'log4j-to-slf4j', version:'2.7'
    compile group: 'org.slf4j', name: 'slf4j-api', version:'1.7.12'
    compile group: 'ch.qos.logback', name: 'logback-classic', version:'1.1.3'
    testCompile group: 'org.elasticsearch.client', name: 'transport', version:'5.4.3'
    testCompile group: 'org.elasticsearch.client', name: 'rest', version:'5.4.3'
    testCompile group: 'com.fasterxml.jackson.core', name: 'jackson-core', version:'2.7.1'
    testCompile group: 'com.fasterxml.jackson.core', name: 'jackson-databind', version:'2.7.1'
    testCompile group: 'org.codehaus.jettison', name: 'jettison', version:'1.3.8'
    testCompile group: 'org.assertj', name: 'assertj-core', version:'3.8.0'
    testCompile group: 'junit', name: 'junit', version:'4.11'
    compile(group: 'org.projectlombok', name: 'lombok', version:'1.16.6') {
       /* This dependency was originally in the Maven provided scope, but the project was not of type war.
       This behavior is not yet supported by Gradle, so this dependency has been converted to a compile dependency.
       Please review and delete this closure when resolved. */
    }
}
```

Plugins maven utilisés :
```xml
<plugin>
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
    <groupId>com.github.alexcojocaru</groupId>
    <artifactId>elasticsearch-maven-plugin</artifactId>
    <version>5.7</version>
    <configuration>
        <clusterName>elasticsearch</clusterName>
        <transportPort>9300</transportPort>
        <httpPort>9200</httpPort>
        <autoCreateIndex>true</autoCreateIndex>
    </configuration>
    <executions>
        <execution>
            <id>start-elasticsearch</id>
            <phase>pre-integration-test</phase>
            <goals>
                <goal>runforked</goal>
            </goals>
        </execution>
        <execution>
            <id>stop-elasticsearch</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>stop</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```


#Téléchargement, installation et démarrage d'Elasticsearch

Le deuxième projet est celui d'[Allegro Tech](https://github.com/allegro/embedded-elasticsearch).

Contrairement à la solution précédente, ce projet permet programmatiquement de télécharger, installer et démarrer/arrêter Elasticsearch.

En outre, l'avantage de cette solution est qu'il n'est pas nécessaire de configurer la partie test d'intégration dans maven. Ainsi, utiliser un autre outils de build est possible.

```java
final embeddedElastic = EmbeddedElastic.builder()
    .withElasticVersion("5.4.3")
    .withSetting(PopularProperties.TRANSPORT_TCP_PORT, 9300)
    .withSetting(PopularProperties.CLUSTER_NAME, "elasticsearch")
    .withPlugin("analysis-stempel")
    .withIndex("cars", IndexSettings.builder()
        .withType("car", getSystemResourceAsStream("car-mapping.json"))
        .build())
    .withIndex("books", IndexSettings.builder()
        .withType(PAPER_BOOK_INDEX_TYPE, getSystemResourceAsStream("paper-book-mapping.json"))
        .withType("audio_book", getSystemResourceAsStream("audio-book-mapping.json"))
        .withSettings(getSystemResourceAsStream("elastic-settings.json"))
        .build())
    .build()
    .start()
```

#Exemple

Code java :
```java
public class ElasticsearchTest {

    private static EmbeddedElastic elasticsearchCluster;

    @BeforeClass
    public static void setup() throws IOException, InterruptedException {
        elasticsearchCluster = EmbeddedElastic.builder()
                .withElasticVersion("5.4.3")
                .withSetting(PopularProperties.TRANSPORT_TCP_PORT, 9300)
                .withSetting(PopularProperties.HTTP_PORT, 9200)
                .withSetting(PopularProperties.CLUSTER_NAME, "elasticsearch")
                .withSetting("network.host", "localhost")
                .withCleanInstallationDirectoryOnStop(true)
                .build()
                .start();
    }

    @AfterClass
    public static void teardown() throws IOException, InterruptedException {
        elasticsearchCluster.stop();
    }

    @Test
    public void transportClient_should_success() throws IOException, InterruptedException {
        TransportClient client = new PreBuiltTransportClient(Settings.EMPTY)
                .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("localhost"), 9300));

        ObjectMapper mapper = new ObjectMapper();

        Sample sample = new Sample("value", 0.33, 3);

        String jsonString = mapper.writeValueAsString(sample);

        // indexing document
        IndexResponse ir = client.prepareIndex("test_index", "type").setSource(jsonString).setId("1").execute().actionGet();
        client.admin().indices().prepareRefresh("test_index").execute().actionGet();

        assertNotNull(ir);

        GetResponse gr = client.prepareGet("test_index", "type", "1").execute().actionGet();

        assertNotNull(gr);
        assertEquals(gr.getSourceAsString(), "{\"value\":\"value\",\"size\":0.33,\"price\":3.0}");
    }

    @Test
    public void restClient_should_success() throws IOException, JSONException {

        RestClient restClient = RestClient.builder(
                new HttpHost("localhost", 9200, "http")).build();

        org.elasticsearch.client.Response response = restClient.performRequest("GET", "/",
                Collections.singletonMap("pretty", "true"));
        System.out.println(EntityUtils.toString(response.getEntity()));

        // indexing document
        HttpEntity entity = new NStringEntity(
                "{\n" +
                        "    \"user\" : \"kimchy\",\n" +
                        "    \"post_date\" : \"2009-11-15T14:12:12\",\n" +
                        "    \"message\" : \"trying out Elasticsearch\"\n" +
                        "}", ContentType.APPLICATION_JSON);

        org.elasticsearch.client.Response indexResponse = restClient.performRequest(
                "PUT",
                "/twitter/tweet/1",
                Collections.<String, String>emptyMap(),
                entity);

        response = restClient.performRequest("GET", "/_search",
                Collections.singletonMap("pretty", "true"));

        String result = EntityUtils.toString(response.getEntity());
        System.out.println(result);
        JSONObject obj = new JSONObject(result);
        int nbResult = obj.getJSONObject("hits").getInt("total");
        assertThat(nbResult).isEqualTo(1);

        restClient.close();
    }

}

@EqualsAndHashCode
@Data
@AllArgsConstructor
class Sample implements Serializable {
    private String value;
    private double size;
    private double price;
}
```

Dépendence maven (façon gradle) :
```plain
dependencies {
    compile group: 'org.apache.logging.log4j', name: 'log4j-to-slf4j', version:'2.7'
    compile group: 'org.slf4j', name: 'slf4j-api', version:'1.7.12'
    compile group: 'ch.qos.logback', name: 'logback-classic', version:'1.1.3'
    testCompile group: 'pl.allegro.tech', name: 'embedded-elasticsearch', version:'2.2.0'
    testCompile group: 'org.elasticsearch.client', name: 'transport', version:'5.4.3'
    testCompile group: 'org.elasticsearch.client', name: 'rest', version:'5.4.3'
    testCompile group: 'com.fasterxml.jackson.core', name: 'jackson-core', version:'2.7.1'
    testCompile group: 'com.fasterxml.jackson.core', name: 'jackson-databind', version:'2.7.1'
    testCompile group: 'org.codehaus.jettison', name: 'jettison', version:'1.3.8'
    testCompile group: 'org.assertj', name: 'assertj-core', version:'3.8.0'
    testCompile group: 'junit', name: 'junit', version:'4.11'
    compile(group: 'org.projectlombok', name: 'lombok', version:'1.16.6') {
       /* This dependency was originally in the Maven provided scope, but the project was not of type war.
       This behavior is not yet supported by Gradle, so this dependency has been converted to a compile dependency.
       Please review and delete this closure when resolved. */
    }
```    

#Conclusion

En conclusion, on a pu voir deux solutions qui permettent de faire des tests d'intégration avec Elasticsearch.

Personnellement, j'ai une préférence pour la deuxième solution qui me permet d'avoir la main sur la récupération et installation d'Elasticsearch.

Pour aller plus loin, quelques liens en vrac.

* http://david.pilato.fr/blog/2016/10/18/elasticsearch-real-integration-tests-updated-for-ga/
* https://www.elastic.co/guide/en/elasticsearch/reference/current/testing-framework.html
* https://gquintana.github.io/2016/11/30/Testing-a-Java-and-Elasticsearch-50-application.html
* https://github.com/alexcojocaru/elasticsearch-maven-plugin
* https://github.com/allegro/embedded-elasticsearch

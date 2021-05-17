---
title: "Des tests d'intégration avec Cassandra"
date: 2017-07-21 10:22:37 +0200
comments: true
tags: 
- Cassandra
- Test
---
![left-small](/images/1200px-Cassandra_logo.svg.png) Parce que je suis parti sur ma lancée des articles _des tests d'intégration avec ..._, à la demande de [Duyhai](https://twitter.com/doanduyhai), voilà que je me retrouve à faire un article pour Apache Cassandra... ;)

Plus sérieusement, faire des tests d'intégration avec Apache Cassandra est beaucoup plus simple qu'avec Redis ou Elasticsearch mais il existe cependant 2 projets qui simplifient énormément les tests d'intégration avec Cassandra :

* [cassandra-unit](https://github.com/jsevellec/cassandra-unit) créé par [Jérémy Sevellec](https://github.com/jsevellec)
* [achille](https://github.com/doanduyhai/Achilles) créé par [Duyhai Doan](https://twitter.com/doanduyhai) 

Ce petit article résume comment utiliser ces 2 solutions.

<!-- more -->
#Classe de test

La classe utilisée comme exemple dans les 2 cas est la suivante :

```java
import com.datastax.driver.core.Cluster;
import com.datastax.driver.core.ResultSet;
import com.datastax.driver.core.Row;
import com.datastax.driver.core.Session;
import lombok.AllArgsConstructor;
import lombok.Data;

import java.util.List;
import java.util.stream.Collectors;

public class UserDao {

    private Cluster cluster;
    private Session session;

    public UserDao(String localhost, int port) {
        cluster = Cluster.builder()
                .addContactPoints(localhost)
                .withPort(port)
                .build();

        session = cluster.connect();
    }

    public List<User> getUsers() {
        ResultSet execute = session.execute("select * from test.user");

        List<Row> res = execute.all();

        return res.stream().map(r -> new User(r.getString("lastName"), r.getString("firstName"))).collect(Collectors.toList());
    }
}

@Data
@AllArgsConstructor
class User {
    private String lastName;
    private String firstName;
}
```

Au niveau dépendance, elles sont les suivantes :
```plain
dependencies {
    compile group: 'com.datastax.cassandra', name: 'cassandra-driver-core', version:'3.3.0'
    compile(group: 'org.projectlombok', name: 'lombok', version:'1.16.18') 
}
```

Rien de bien compliqué mais il s'agit d'un cas d'exemple ultra simple... ;)

#Cassandra-Unit

Pour utiliser Cassandra-Unit, il suffit de déclarer sa dépendance en scope de test:
```xml
<dependency>
    <groupId>org.cassandraunit</groupId>
    <artifactId>cassandra-unit</artifactId>
    <version>3.1.3.2</version>
    <scope>test</scope>
</dependency>
```

La classe de test ressemble alors à :
```java
import com.datastax.driver.core.Cluster;
import com.datastax.driver.core.Session;
import org.apache.thrift.transport.TTransportException;
import org.cassandraunit.utils.EmbeddedCassandraServerHelper;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.io.IOException;
import java.util.List;

import static org.junit.Assert.assertEquals;

public class UserDaoIntegrationTest {

    private Session session;
    private Cluster cluster;

    @Before
    public void setUp() throws InterruptedException, IOException, TTransportException {
        EmbeddedCassandraServerHelper.startEmbeddedCassandra("cassandra-sample.yaml");

        cluster = Cluster.builder()
                .addContactPoints("localhost")
                .withPort(9000)
                .build();

        session = cluster.connect();

        session.execute("create KEYSPACE test WITH replication = {'class': 'SimpleStrategy' , 'replication_factor': '1' }");
        session.execute("CREATE TABLE test.user (lastName text, firstName text, PRIMARY KEY (lastName))");
    }

    @After
    public void teardown() {
        session.close();
        EmbeddedCassandraServerHelper.stopEmbeddedCassandra();
    }


    @Test
    public void user_should_be_returned() {
        session.execute("insert into test.user(lastName, firstName) values('lastName1', 'firstName1')");

        UserDao cassandraJob = new UserDao("localhost", 9000);

        List<User> users = cassandraJob.getUsers();

        assertEquals(users.size(), 1);
        assertEquals(users.get(0).getFirstName(), "firstName1");
        assertEquals(users.get(0).getLastName(), "lastName1");
    }

}
```

A noter que le fichier `cassandra-sample.yaml` correspond au fichier de configuration par défaut de cassandra avec les spécificités suivantes :
```yaml
native_transport_port: 9000
data_file_directories:
    - target/embeddedCassandra/data
commitlog_directory: target/embeddedCassandra/commitlog
hints_directory: target/embeddedCassandra/hints
cdc_raw_directory: target/embeddedCassandra/cdc_raw
saved_caches_directory: target/embeddedCassandra/saved_caches

#multithreaded_compaction: false
#memtable_flush_queue_size: 4
#compaction_preheat_key_cache: true
#in_memory_compaction_limit_in_mb: 64
```

A noter également que Cassandra-Unit propose également des loader permettant d'initialiser les données dans le cluster ainsi que d'autres facilités d'utilisation telles que des Rules ou un support de Spring. Pour avoir plus d'informations, je vous invite à aller voir la documentation.

#Achille-Embedded

Pour utiliser Achille-embedded, il suffit de déclarer sa dépendance en scope de test:
```xml
<dependency>
    <groupId>info.archinnov</groupId>
    <artifactId>achilles-embedded</artifactId>
    <version>5.2.1</version>
    <scope>test</scope>
</dependency>
```

La classe de test ressemble alors à :
```java
import com.datastax.driver.core.Session;
import info.archinnov.achilles.embedded.CassandraEmbeddedServerBuilder;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.List;

import static org.junit.Assert.assertEquals;

public class UserDaoIntegrationTest {

    private Session session;

    @Before
    public void setUp() {
        session = CassandraEmbeddedServerBuilder.builder()
                .withCQLPort(9000)
                .withClusterName("Test Cluster")
                .cleanDataFilesAtStartup(true)
                .buildNativeSession();

        session.execute("create KEYSPACE test WITH replication = {'class': 'SimpleStrategy' , 'replication_factor': '1' }");
        session.execute("CREATE TABLE test.user (lastName text, firstName text, PRIMARY KEY (lastName))");
    }

    @After
    public void teardown() {
        session.close();
    }


    @Test
    public void user_should_be_returned() {
        session.execute("insert into test.user(lastName, firstName) values('lastName1', 'firstName1')");

        UserDao cassandraJob = new UserDao("localhost", 9000);

        List<User> users = cassandraJob.getUsers();

        assertEquals(users.size(), 1);
        assertEquals(users.get(0).getFirstName(), "firstName1");
        assertEquals(users.get(0).getLastName(), "lastName1");
    }
}
```

A noter également qu'Achille-Embedded propose également des Rules et offre des facilités pour loader des données pour les tests. Pour avoir plus d'informations, je vous invite à aller voir les différents tests du projet.

#Conclusion

On a donc vu dans ce rapide article comment il était possible de faire des tests d'intégration avec Cassandra.

A noter que le code montré dans cet article est extrêmement simple mais je voulais juste montrer un rapide cas d'usage, du coup, pas grand chose à ajouter puisque les exemples parlent d'eux-même ;) . 

Concernant le choix de l'un ou de l'autre, j'avoue avoir une préférence pour Achille-Embedded car il permet plus simplement d'avoir la main sur la configuration et ne pas avoir à maintenir un fichier de configuration en plus. 

A noter qu'Achille-Embedded a été utilisé dans [Hadoop-Unit](https://github.com/jetoile/hadoop-unit) et qu'il est possible d'utiliser Cassandra en test d'intégration des manière suivantes ( #autopromo ;) ) :

* En utilisant le démarrage de [manière programmatique](https://github.com/jetoile/hadoop-unit/blob/master/hadoop-unit-cassandra/src/test/java/fr/jetoile/hadoopunit/component/CassandraBootstrapTest.java)
* En utilisant le plugin maven en [phase de pré-integration](https://github.com/jetoile/hadoop-unit/blob/master/sample/spark-streaming-cassandra)
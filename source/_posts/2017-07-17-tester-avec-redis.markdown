---
layout: post
title: "Des tests d'intégration avec Redis"
date: 2017-07-17 11:10:30 +0200
comments: true
categories: 
- Redis
- Test
---
![left-small](/images/redis.png) Redis est écrit en C et faire des tests d'intégration en Java peut s'avérer compliquer. En outre, le fait que Redis doive être compilé lors de son installation rend les choses encore moins aisées.

Bien sûr, il est possible d'utiliser Docker ou de l'installer préalablement sur son poste mais cette deuxième option casse un peu les bonnes pratiques des tests.

Il existe également de nombreux projets permettant de faire des tests avec Redis mais, souvent, les solutions proposées embarquent le binaire de Redis ou on besoin qu'il soit déjà présent et installer/compiler sur le poste (https://github.com/kstyrc/embedded-redis, https://github.com/lordofthejars/nosql-unit, https://github.com/ishiis/redis-unit). Les solutions qui intègrent le binaire ne sont malheureusement souvent pas à jour et laisse assez peu la main sur la version.

Pour ceux qui n'aurait pas envie d'utiliser Docker, cet article va montrer comment il est possible de piloter programmatiquement l'installation de Redis afin de permettre les tests d'intégration.

<!-- more -->

En fait, le code ci-dessous se charge de télécharger Redis et de le compiler dans le répertoire `$USER_HOME/.redis`. Pour ce faire une tache ant est utilisée car un appel direct à la commande make échoue.

```java
import org.apache.commons.io.FileUtils;
import org.apache.commons.io.IOUtils;
import org.apache.tools.ant.BuildException;
import org.apache.tools.ant.DefaultLogger;
import org.apache.tools.ant.Project;
import org.apache.tools.ant.ProjectHelper;
import org.codehaus.plexus.archiver.ArchiverException;
import org.codehaus.plexus.archiver.tar.TarGZipUnArchiver;
import org.codehaus.plexus.logging.console.ConsoleLogger;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.URL;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Arrays;
import java.util.List;

import static org.apache.commons.io.FileUtils.getFile;

public class RedisInstaller {

    private static final Logger LOGGER = LoggerFactory.getLogger(RedisInstaller.class);
    private static final String REDIS_PACKAGE_PREFIX = "redis-";
    private static final List<String> REDIS_EXECUTABLE_FILE = Arrays.asList("redis-server", "redis-sentinel");

    private static final Path REDIS_INSTALLATION_PATH = Paths.get(System.getProperty("user.home") + "/.redis");

    private final String downloadUrl;
    private final String version;
    private final boolean forceCleanupInstallationDirectory;
    private final String tmpDir;

    RedisInstaller(String version, String downloadUrl, boolean forceCleanupInstallationDirectory, String tmpDir) {
        this.downloadUrl = downloadUrl;
        this.version = version;
        this.forceCleanupInstallationDirectory = forceCleanupInstallationDirectory;
        this.tmpDir = tmpDir;
        REDIS_INSTALLATION_PATH.toFile().mkdirs();
    }

    File getExecutableFile() {
        return fileRelativeToInstallationDir("src", "redis-server");
    }

    private File fileRelativeToInstallationDir(String... path) {
        return getFile(getInstallationDirectory(), path);
    }

    private File getInstallationDirectory() {
        return getFile(REDIS_INSTALLATION_PATH.toFile(), REDIS_PACKAGE_PREFIX + version);
    }

    void install() throws IOException, InterruptedException {
        if (forceCleanupInstallationDirectory) {
            FileUtils.forceDelete(getInstallationDirectory());
        }
        installRedis();
        makeRedis();
        applyRedisPermissionRights();
    }


    private void installRedis() throws IOException {
        Path downloadedTo = download(new URL(downloadUrl + REDIS_PACKAGE_PREFIX + version + ".tar.gz"));
        install(downloadedTo);
    }

    private File getAntFile() throws IOException {
        InputStream in = RedisInstaller.class.getClassLoader().getResourceAsStream("build.xml");
        File fileOut = new File(tmpDir, "build.xml");

        LOGGER.info("Writing redis' build.xml to: " + fileOut.getAbsolutePath());

        OutputStream out = FileUtils.openOutputStream(fileOut);
        IOUtils.copy(in, out);
        in.close();
        out.close();

        return fileOut;
    }

    private void makeRedis() throws IOException, InterruptedException {
        LOGGER.info("> make");
        File makeFilePath = getInstallationDirectory();

        DefaultLogger consoleLogger = getConsoleLogger();

        Project project = new Project();
        File buildFile = getAntFile();
        project.setUserProperty("ant.file", buildFile.getAbsolutePath());
        project.addBuildListener(consoleLogger);

        try {
            project.fireBuildStarted();
            project.init();
            ProjectHelper projectHelper = ProjectHelper.getProjectHelper();
            project.addReference("ant.projectHelper", projectHelper);
            project.setProperty("redisDirectory", makeFilePath.getAbsolutePath());
            projectHelper.parse(project, buildFile);
            project.executeTarget("init");
            project.fireBuildFinished(null);
        } catch (BuildException buildException) {
            project.fireBuildFinished(buildException);
            throw new RuntimeException("!!! Unable to compile redis !!!", buildException);
        }
    }

    private DefaultLogger getConsoleLogger() {
        DefaultLogger consoleLogger = new DefaultLogger();
        consoleLogger.setErrorPrintStream(System.err);
        consoleLogger.setOutputPrintStream(System.out);
        consoleLogger.setMessageOutputLevel(Project.MSG_INFO);

        return consoleLogger;
    }

    private Path download(URL source) throws IOException {
        File target = new File(REDIS_INSTALLATION_PATH.toString(), source.getPath());
        if (!target.exists()) {
            LOGGER.info("Downloading : " + source + " to " + target + "...");
            FileUtils.copyURLToFile(source, target);
            LOGGER.info("Download complete");
        } else {
            LOGGER.info("Download skipped");
        }
        return target.toPath();
    }

    private void install(Path downloadedFile) throws IOException {
        LOGGER.info("Installing Redis into " + REDIS_INSTALLATION_PATH + "...");
        try {
            final TarGZipUnArchiver ua = new TarGZipUnArchiver();
            ua.setSourceFile(downloadedFile.toFile());
            ua.enableLogging(new ConsoleLogger(1, "console"));
            ua.setDestDirectory(REDIS_INSTALLATION_PATH.toFile());
            ua.extract();
            LOGGER.info("Done");
        } catch (ArchiverException e) {
            LOGGER.info("Failure : " + e);
            throw new RuntimeException("!!! Unable to download and untar redis !!!", e);
        }
    }

    private void applyRedisPermissionRights() throws IOException {
        File binDirectory = getFile(getInstallationDirectory(), "src");
        for (String fn : REDIS_EXECUTABLE_FILE) {
            File executableFile = new File(binDirectory, fn);
            LOGGER.info("Applying executable permissions on " + executableFile);
            executableFile.setExecutable(true);
        }
    }

}
```

Ainsi, en wrappant l'appel dans un builder, il devient facile d'invoquer l'installeur.
```java
import lombok.Builder;
import lombok.Data;

import java.io.File;
import java.io.IOException;

@Data
@Builder
public class EmbeddedRedisInstaller {

    private String downloadUrl;
    private String version;
    private String tmpDir;
    private boolean forceCleanupInstallationDirectory;

    RedisInstaller redisInstaller;

    public EmbeddedRedisInstaller install() throws IOException, InterruptedException {
        installRedis();
        return this;
    }

    private void installRedis() throws IOException, InterruptedException {
        redisInstaller = new RedisInstaller(version, downloadUrl, forceCleanupInstallationDirectory, tmpDir);
        redisInstaller.install();
    }

    public File getExecutableFile() {
        return redisInstaller.getExecutableFile();
    }
}
```

Utilisation de l'installeur :
```java
EmbeddedRedisInstaller.builder()
                .version("4.0.0")
                .downloadUrl("http://download.redis.io/releases/")
                .forceCleanupInstallationDirectory(false)
                .tmpDir("tmp/redis")
                .build()
        .install();
```

Enfin, en le couplant, avec le projet [redis-unit](https://github.com/ishiis/redis-unit), il est possible de choisir son mode d'utilisation tout en rendant configurable les ports :
```
EmbeddedRedisInstaller installer = EmbeddedRedisInstaller.builder()
                .downloadUrl(downloadUrl)
                .forceCleanupInstallationDirectory(cleanupInstallation)
                .version(version)
                .tmpDir(tmpDir)
                .build();

try {
    installer.install();
} catch (IOException | InterruptedException e) {
    LOGGER.error("unable to install redis", e);
}

switch (type) {
    case SERVER:
        redisServer = new RedisServer(
                new RedisServerConfig.ServerBuilder(masterPort)
                        .redisBinaryPath(installer.getExecutableFile().getAbsolutePath())
                        .build()
        );
        break;
    case CLUSTER:
        redisCluster = new RedisCluster(
                slavePorts.stream().map(p ->
                        new RedisClusterConfig.ClusterBuilder(p)
                                .redisBinaryPath(installer.getExecutableFile().getAbsolutePath())
                                .build()
                ).collect(Collectors.toList())
        );
        break;
    case MASTER_SLAVE:
        redisMasterSlave = new RedisMasterSlave(
                new RedisMasterSlaveConfig.MasterBuilder(masterPort)
                        .redisBinaryPath(installer.getExecutableFile().getAbsolutePath())
                        .build(),
                slavePorts.stream().map(p ->
                        new RedisMasterSlaveConfig.SlaveBuilder(p, masterPort)
                                .redisBinaryPath(installer.getExecutableFile().getAbsolutePath())
                                .build()
                ).collect(Collectors.toList())
        );
        break;
    case SENTINEL:
        redisSentinel = new RedisSentinel(
                new RedisMasterSlaveConfig.MasterBuilder(masterPort)
                        .redisBinaryPath(installer.getExecutableFile().getAbsolutePath())
                        .build(),
                slavePorts.stream().map(p ->
                        new RedisMasterSlaveConfig.SlaveBuilder(p, masterPort)
                                .redisBinaryPath(installer.getExecutableFile().getAbsolutePath())
                                .build()
                ).collect(Collectors.toList()),
                sentinelPorts.stream().map(p ->
                        new RedisSentinelConfig.SentinelBuilder(p, masterPort)
                                .redisBinaryPath(installer.getExecutableFile().getAbsolutePath())
                                .build()
                ).collect(Collectors.toList())
        );
        break;
}
```

# Conclusion

On a pu voir dans cet article comment il était possible simplement d'utiliser Redis lors de ses tests d'intégration sans utiliser Docker et tout en pilotant la chose programmatiquement.

Pour retrouver le code, il est disponible à l'[url suivante](https://github.com/jetoile/hadoop-unit/tree/master/hadoop-unit-redis).

A noter qu'il a été utilisé dans [Hadoop-Unit](https://github.com/jetoile/hadoop-unit) et qu'il est possible d'utiliser Redis en test d'intégration des manière suivantes ( #autopromo ;) ) :

- En utilisant le démarrage de [manière programmatique](https://github.com/jetoile/hadoop-unit/blob/master/hadoop-unit-redis/src/test/java/fr/jetoile/hadoopunit/RedisBootstrapTest.java): 
```
@BeforeClass
public static void setup() throws BootstrapException {
    HadoopBootstrap.INSTANCE.startAll();
}

@AfterClass
public static void tearDown() throws BootstrapException {
    HadoopBootstrap.INSTANCE.stopAll();
}


@Test
public void testStartAndStopServerMode() throws InterruptedException {
    Jedis jedis = new Jedis("127.0.0.1", 6379);
    Assert.assertNotNull(jedis.info());
    System.out.println(jedis.info());
    jedis.close();
}
```

avec le pom suivant :
```xml
<dependency>
    <groupId>fr.jetoile.hadoop</groupId>
    <artifactId>hadoop-unit-redis</artifactId>
    <version>2.2</version>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
    <scope>test</scope>
</dependency>
```

et le fichier de configuration `hadoop-unit-default.properties` suivant:
```plain
# Redis
redis.port=6379
redis.download.url=http://download.redis.io/releases/
redis.version=4.0.0
redis.cleanup.installation=false
redis.temp.dir=/tmp/redis
redis.type=SERVER
#redis.type=CLUSTER
#redis.type=MASTER_SLAVE
#redis.type=SENTINEL
#redis.slave.ports=6380
#redis.sentinel.ports=36479,36480,36481,36482,36483
```

- En utilisant le plugin maven en [phase de pré-integration](https://github.com/jetoile/hadoop-unit/tree/master/sample/redis): 
```xml
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
    </executions>
    <configuration>
        <components>
            <componentArtifact implementation="fr.jetoile.hadoopunit.ComponentArtifact">
                <componentName>REDIS</componentName>
                <artifact>fr.jetoile.hadoop:hadoop-unit-redis:${hadoop-unit.version}</artifact>
            </componentArtifact>
        </components>
    </configuration>
</plugin>
```
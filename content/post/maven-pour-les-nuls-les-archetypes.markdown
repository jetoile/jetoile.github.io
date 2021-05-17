---
title: "Maven pour les nuls... les archetypes"
date: 2011-06-05 21:48:02 +0100
comments: true
tags: 
- maven
- java
- archetype
---
![left-small](http://1.bp.blogspot.com/-l-NZQRxE0vI/TevA5LfJQ_I/AAAAAAAAAXI/IP8zRk4wbeg/s1600/maven-logo-2.gif)

Dans le cadre d'une problématique d'une usine logiciel, il peut s'avérer utile de posséder un patron ou template de projet qui fournit les "bonnes" pratiques que doivent respecter l'ensemble des applications du projet.

Bien sûr, notre cher IDE est capable de générer un joli patron de projet. Cependant, il reste toujours nécessaire de modifier la configuration du projet pour y mettre, par exemple :

* la version de jdk,
* les libs utilisées (comme mockito par exemple),
* optionnellement, la configuration de plugins,
* optionnellemet, l'url du SCM,
* ou tout simplement, la référence à un projet parent qui permet de définir, par exemple, la version des librairies ou des plugins utilisés. 

Le hic, c'est que, généralement, cela fini par de jolis copier/coller dont le résultat diffère, bien sûr, en fonction du projet qui a servi de template. Le résultat : le syndrome du téléphone arabe sachant qu'en plus, on n'est même plus capable de savoir qu'elle était la référence du départ... embêtant tout ça... surtout pour un projet se voulant industriel...

En outre, posséder un patron de projet peut également s'avérer utile si vous êtes amené à POCer des frameworks ou faire des projets perso chez vous...

Vous l'aurez compris, cet article se focalisera sur les projets construits sur maven où les notions de dépendances, de plugins, d'informations projet sont présentes dans le pom.

Cet article a donc pour objectif de montrer comment il est possible de créer un archetype maven qui permet de répondre à ce problème.

Le use case sera simple puisque je m'appuierai sur la création d'un archetype simple pour mes besoins personnels afin de me fournir un patron normalisé (selon mes normes) me permettant de démarrer un POC rapidement pour un artifact de type jar et cela dans le seul but de ne pas avoir à faire moultes copier/coller... ;-)

A noter qu'il n'y a rien de révolutionnaire dans cet article pour toutes les personnes qui ont quelques connaissances de base en maven mais qu'il peut s'avérer utile (enfin j'espère) pour les autres... ;-)

<!-- more -->

# Méthodologie
Afin de créer un archetype, il existe plusieurs méthodes :

* la première consiste à créer l'archetype directement, chose pas trop complexe mais qui demande d'être rodée à la chose,
* la deuxième consiste à créer le projet maven tel que l'on souhaiterait que l'archetype nous le créer et de demander à maven de nous générer l'archetype en lui-même. Suite à cela, il convient de modifier l'archetype généré afin de le rendre configurable (nom des packages, ...).

Suite à cela, il convient d'installer (ou de déployer) notre archetype au sein de notre repository (global ou distant) afin de le rendre accessible aux autres membres de l'équipe ou tout simplement à soi-même.

Dans mon cas, j'utiliserai la deuxième méthode qui consiste donc à générer l'archetype à partir du projet maven que je souhaite avoir.

# Création du projet de base

Comme je l'ai dit précédemment, mes besoins sont purement personnels et mon projet devra répondre aux points suivants :

* un seul projet,
* de type jar,
* utilisant un jdk 6,
* déclarant les librairies suivantes : slf4j, logback-classic, commons-lang
* déclarant les librairies suivantes pour mes TUs : JUnit, mockito, powermock
* déclarant et configurant le plugin checkstyle et dont les rules seront dans le projet en lui-même
* déclarant et configurant le plugin assembly afin de me permettre de fournir, au besoin, un livrable
* déclarant le plugin source afin de me permettre de générer un jar de source

Bien sûr, il tentera de suivre au mieux les bonnes pratiques maven...

Cette partie n'étant pas bien intéressante et étant spécifique aux besoins de chacun, je la détaillerai donc peu...

Sa structure :

![center](http://4.bp.blogspot.com/-ActIqf-Gkv8/TevBObqDojI/AAAAAAAAAXM/TRHrtsoB8L0/s1600/archetype01.png)

Son pom :

```xml
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0" xsi:schemalocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"><build><plugins><project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
 <modelVersion>4.0.0</modelVersion>
 <groupId>fr.jetoile</groupId>
 <artifactId>project-template</artifactId>
 <version>0.0.1-SNAPSHOT</version>
 
 <properties>
  <java.version>1.6</java.version>
 
  <junit.version>4.8.1</junit.version>
  <slf4j.version>1.6.1</slf4j.version>
  <logback.version>0.9.27</logback.version>
  <commons-lang.version>2.5</commons-lang.version>
  <powermock.version>1.4.7</powermock.version>
  <mockito.version>1.8.5</mockito.version>
 </properties>
 
 <!-- distributionManagement>
  <repository>
   <id>nexus</id>
   <url>http://localhost:8080/nexus/content/repositories/releases</url>
  </repository>
  <snapshotRepository>
   <id>nexus</id>
   <name>Nexus snapshot Repository</name>
   <url>http://localhost:8080/nexus/content/repositories/snapshots</url>
  </snapshotRepository>
 </distributionManagement>
 <scm>
  <developerConnection>scm:svn:svn://localhost/projet/trunk/TODO</developerConnection>
 </scm-->
 
 
 <developers>
  <developer>
   <email>kmx.petals@gmail.com</email>
   <id>khanh</id>
   <roles>
    <role>developer</role>
   </roles>
  </developer>
 </developers>
 
 <dependencies>
  <dependency>
   <groupId>ch.qos.logback</groupId>
   <artifactId>logback-classic</artifactId>
   <version>${logback.version}</version>
  </dependency>
  <dependency>
   <groupId>org.slf4j</groupId>
   <artifactId>slf4j-api</artifactId>
   <version>${slf4j.version}</version>
  </dependency>
  <dependency>
   <groupId>commons-lang</groupId>
   <artifactId>commons-lang</artifactId>
   <version>${commons-lang.version}</version>
  </dependency>
  <dependency>
   <groupId>org.mockito</groupId>
   <artifactId>mockito-all</artifactId>
   <version>${mockito.version}</version>
   <scope>test</scope>
  </dependency>
  <dependency>
   <groupId>org.powermock</groupId>
   <artifactId>powermock-module-junit4</artifactId>
   <version>${powermock.version}</version>
   <scope>test</scope>
  </dependency>
  <dependency>
   <groupId>junit</groupId>
   <artifactId>junit</artifactId>
   <version>${junit.version}</version>
   <scope>test</scope>
  </dependency>
  <dependency>
   <groupId>org.powermock</groupId>
   <artifactId>powermock-api-mockito</artifactId>
   <version>${powermock.version}</version>
   <scope>test</scope>
  </dependency>
 </dependencies>
 
 
 <build>
  <plugins>
   <plugin>
    <artifactId>maven-deploy-plugin</artifactId>
    <version>2.6</version>
   </plugin>
 
   <plugin>
    <artifactId>maven-release-plugin</artifactId>
    <version>2.1</version>
   </plugin>
 
   <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>2.3.2</version>
    <configuration>
     <source>${java.version}</source>
     <target>${java.version}</target>
    </configuration>
   </plugin>
 
   <plugin>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>2.6</version>
    <configuration>
     <configLocation>src/config/custom-checkstyle.xml</configLocation>
     <includeTestSourceDirectory>false</includeTestSourceDirectory>
     <logViolationsToConsole>true</logViolationsToConsole>
    </configuration>
    <executions>
     <execution>
      <phase>compile</phase>
      <goals>
       <goal>check</goal>
      </goals>
     </execution>
    </executions>
   </plugin>
 
   <plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>2.2.1</version>
    <configuration>
     <descriptors>
      <descriptor>src/assembly/src.xml</descriptor>
     </descriptors>
    </configuration>
    <executions>
     <execution>
      <id>make-assembly</id>
      <phase>install</phase>
      <goals>
       <goal>single</goal>
      </goals>
     </execution>
    </executions>
   </plugin>
 
   <plugin>
    <artifactId>maven-source-plugin</artifactId>
    <version>2.1.2</version>
    <executions>
     <execution>
      <phase>package</phase>
      <goals>
       <goal>jar</goal>
      </goals>
     </execution>
    </executions>
   </plugin>
  </plugins>
 </build>
</project>  </plugins>
 </build>
</project>
```

Son fichier assembly :
```xml
<assembly>
</assembly><assembly>
 <id>bin</id>
 <formats>
  <format>zip</format>
 </formats>
 <includeBaseDirectory>false</includeBaseDirectory>
 <dependencySets>
  <dependencySet>
   <scope>compile</scope>
   <useProjectArtifact>true</useProjectArtifact>
   <useTransitiveDependencies>true</useTransitiveDependencies>
   <outputDirectory>lib</outputDirectory>
  </dependencySet>
 </dependencySets>
 
 <fileSets>
  <fileSet>
   <directory>target/</directory>
   <outputDirectory>sources</outputDirectory>
   <includes>
    <include>**sources.jar</include>
   </includes>
  </fileSet>
  <fileSet>
   <directory>bin/</directory>
   <outputDirectory>bin</outputDirectory>
  </fileSet>
 </fileSets>
 
</assembly>
```

# Création de l'archetype

La création de l'archetype est simple puisqu'il suffit de lancer la commande suivante :
```bash
mvn archetype:create-from-project
```

Suite à cela, notre archetype se trouve dans le répertoire target/generated-sources/archetype et a la structure suivante :

![center](http://4.bp.blogspot.com/-QGyUib6583A/TevDZZVezbI/AAAAAAAAAXQ/lHGC3y6oLew/s1600/archetype02.png)

Son pom :
```xml
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0" xsi:schemalocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
</project><project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
 
  <groupId>fr.jetoile</groupId>
  <artifactId>project-template-archetype</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>maven-archetype</packaging>
 
  <name>project-template-archetype</name>
 
  <build>
    <extensions>
      <extension>
        <groupId>org.apache.maven.archetype</groupId>
        <artifactId>archetype-packaging</artifactId>
        <version>2.0</version>
      </extension>
    </extensions>
 
    <pluginManagement>
      <plugins>
        <plugin>
          <artifactId>maven-archetype-plugin</artifactId>
          <version>2.0</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```
On constate qu'il y a deux points à noter dans cette structure d'archetype générée :

* le patron de notre template de projet qui sera utilisé par maven lors de l'appel au goal archetype:generate se trouve dans le répertoire src/main/resources/archetype-resources/src,
* dans le répertoire src/main/resources/META-INF/maven, se trouve le fichier archetype-metadata.xml qui contient le descriptif de ce qui doit être généré :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<archetype-descriptor xsi:schemaLocation="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0 http://maven.apache.org/xsd/archetype-descriptor-1.0.0.xsd" name="project-template"
    xmlns="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
     
  <fileSets>
    <fileSet filtered="true" packaged="true" encoding="UTF-8">
      <directory>src/main/java</directory>
      <includes>
        <include>**/*.java</include>
      </includes>
    </fileSet>
    <fileSet filtered="true" packaged="true" encoding="UTF-8">
      <directory>src/test/java</directory>
      <includes>
        <include>**/*.java</include>
      </includes>
    </fileSet>
    <fileSet filtered="true" encoding="UTF-8">
      <directory>src/config</directory>
      <includes>
        <include>**/*.xml</include>
      </includes>
    </fileSet>
    <fileSet filtered="true" encoding="UTF-8">
      <directory>src/assembly</directory>
      <includes>
        <include>**/*.xml</include>
      </includes>
    </fileSet>
    <fileSet filtered="true" encoding="UTF-8">
      <directory></directory>
      <includes>
        <include>README.txt</include>
      </includes>
    </fileSet>
  </fileSets>
</archetype-descriptor>
```

A cette étape, il est tout de suite possible d'installer l'archetype dans le repository maven (cf. Mise à disposition de l'archetype) et de l'invoquer (cf. Utilisation et résultat de l'archetype).
A noter qu'ici, il peut être intéressant de supprimer le support des .project, .classpath et .settings si le template a été géré avec eclipse.

# Transformation de l'archetype

Dans le cadre de mon petit use case, je n'ai pas beaucoup besoin de customiser mon archetype. En effet, je souhaite seulement pouvoir demander à l'utilisateur (en l'occurence moi ;-) ), quel sera le nom de ma classe à me générer.

Pour ce faire, il n'y a qu'à ajouter une propriété dans le fichier src/main/resources/META-INF/maven/archetype-metadata.xml :
```xml
<requiredProperties>
    <requiredProperty key="className">
      <defaultValue>MyClass</defaultValue>
    </requiredProperty>
</requiredProperties>
```

Modifier le nom des fichiers java :

* `Test.java` en `__className__.java`
* `TestTest` en `__className__.java`

Et remplacer dans les fichiers respectifs :
```java
public class Test {
```
en
```java
public class ${className} {
```
et 
```java
public class TestTest {
```
en
```java
public class ${className}Test {
```

En outre, il faut également modifier le fichier src/test/resources/projects/basic/archetype.properties pour y ajouter la ligne suivante :
```text
className=MyClass
```
Enfin, il est possible de figer la version de l'archetype en modifiant le pom du projet ainsi que son artifactId et son groupId. Dans mon cas, il restera identique.

# Mise à disposition de l'archetype

Une fois que l'archetype a été modifié pour répondre aux besoins, il n'y a plus qu'à exécuter la commande mvn install pour déployer l'archetype dans le repository local et la commande mvn deploy pour le déployer dans le repository partagé.

# Utilisation et résultat de l'archetype

Une fois l'archetype déployé dans le repository adéquate, il n'y a plus qu'à l'invoquer via la commande :
```bash
mvn archetype:generate \
 -DarchetypeGroupId=fr.jetoile \
 -DarchetypeArtifactId=project-template-archetype \
 -DarchetypeVersion=0.0.1-SNAPSHOT \
 -DgroupId=fr.jetoile \
 -DartifactId=test-archetype
```

Ainsi, le résultat obtenu est le suivant :
![center](http://3.bp.blogspot.com/-IyLOW6pbUgw/TevFCch0TRI/AAAAAAAAAXU/-8aUatC2M5c/s1600/archetype04.png)

# Conclusion

Nous avons vu dans cet article qu'il était extrêmement simple de créer un archetype maven, action qui peut s'avérer très utile pour gagner du temps que ce soit pour des besoins personnels ou dans un cadre d'industrialisation du processus d'initialisation d'un projet dans une équipe.

[update] Je viens de mettre sur github le code :

* du projet utilisé pour créer l'archetype ainsi que le code de l'archetype : https://github.com/jetoile/project-template 
* du projet permettant de créer l'archetype : https://github.com/jetoile/project-template-archetype


Cependant, je n'ai pas invoqué ici le cas de projets multimodules mais le principe reste identique, même s'il est possible d'utiliser d'autres templates (tel que `__rootArtifactId__`) si l'on souhaite préfixer le nom des modules avec l'artifactId du projet généré ou si l'on souhaite customiser plus finement un nom de répertoire (ou package) : dans ce cas, il peut être souhaitable d'ajouter d'autres propriétés dans le fichier archetype-metadata.xml :

```xml
<module id="${rootArtifactId}-core" dir="__rootArtifactId__-core" name="${rootArtifactId}-core">
      <fileSets>
        <fileSet filtered="true" encoding="UTF-8" packaged="true">
          <directory>src/main/java</directory>
          <includes>
            <include>**/*.java</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8" packaged="true">
          <directory>src/test/java</directory>
          <includes>
            <include>**/*.java</include>
          </includes>
        </fileSet>
      </fileSets>
    </module>
    <module id="${rootArtifactId}-client" dir="__rootArtifactId__-client" name="${rootArtifactId}-client">
      <fileSets>
        <fileSet filtered="true" encoding="UTF-8" packaged="true">
          <directory>src/main/java</directory>
          <includes>
            <include>**/*.java</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8" packaged="true">
          <directory>src/main/resources</directory>
          <includes>
            <include>**/*.xml</include>
            <include>**/*.properties</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8" packaged="true">
          <directory>src/test/resources</directory>
          <includes>
            <include>**/*.xml</include>
            <include>**/*.properties</include>
          </includes>
        </fileSet>
      </fileSets>
    </module>
```

![center](http://1.bp.blogspot.com/-VL7re_vr97E/TevFPbO9msI/AAAAAAAAAXY/tXBaVB_kG-0/s1600/archetype03.png)

# Pour aller plus loin...

* Apache Maven de N. De Loof, A. Héritier chez Pearson
* Better Builds with Maven : http://repo.exist.com/dist/maestro/1.7.0/BetterBuildsWithMaven.pdf
* Maven. Definitive Guide : http://www.sonatype.com/products/maven/documentation/book-defguide
* Site de maven : http://maven.apache.org/

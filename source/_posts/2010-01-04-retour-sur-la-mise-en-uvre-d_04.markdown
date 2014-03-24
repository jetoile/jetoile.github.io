---
layout: post
title: "Retour sur la mise en oeuvre d'un environnement de développement"
date: 2010-01-04 20:17:40 +0100
comments: true
sharing: true
footer: true
categories: 
- cargo
- java
- jetspeed
- maven
---
![left](http://maven.apache.org/images/maven-logo-2.gif)
Cet article présente la mise en œuvre que j'ai appliquée lors de la mise en place d'un environnement de développement devant s'interfacer, dans une démarche pseudo-agile, avec des outils d'intégration continue.
Il présentera, dans un premier temps, le contexte et les problématiques puis, dans un second temps, comment j'ai tenté de répondre à ces problématiques que ce soit d'un point de vue méthodologique que d'un point de vue technique. Bien sûr, les choix et les implémentations utilisés sont discutables, mais c'est aussi pour cela que j'ai créé ce blog (afin de faire partager mon expérience et d'avoir un retour) ;-)

Il ne présentera ni l'utilité des outils d'intégration continue ni celle d'une démarche agile (ce projet ne fonctionnait pas en agile mais mettait en œuvre quelques-unes de ses pratiques) qui sont très bien expliqués sur d'autres sites ou blogs. Enfin, il est à noter que ce projet possédait un existant et que mon rôle n'était pas de remettre en cause les solutions retenues.

<!-- more -->
#Présentation du contexte et des problématiques
##Contexte
Le projet concerné avait pour but, entre autre, de fournir un site web à destination du grand public. Il s'appuyait sur le portail Jetspeed 2 (qui pour rappel est un portail permettant d'héberger des portlets 2 - JSR 286 à l'image de Liferay ou GateIn) dont les fonctionnalités d'édition avaient été désactivées. Aussi, le portail Jetspeed n'était utilisé que pour ses capacités à "modularisé" les composants d'affichage à l'aide de portlets qui étaient figés dans les pages. Des services web étaient également utilisés pour permettre de s'interconnecter avec un progiciel utilisé pour implémenter le cœur du métier.

Les outils de build utilisés étaient maven 2 et plus précisément le plugin Jetspeed fournit par ce dernier.
Enfin, le cycle d'itération des développements étaient de 2 semaines... 2 semaines qui incluaient la définition de ce qu'allait embarquer la livraison (ou si on peut l'appeler comme cela, ce qu'il y aurait dans le sprint), les tests fonctionnels ainsi que la mise en production du livrable... 

L'équipe incluait 12 développeurs et, dans la globalité, un environnement de développement hétérogène (la faute n'incubant pas foncièrement aux développeurs mais plutôt à des délais beaucoup trop serrés ainsi qu'à une très grosse charge de travail).

##Problématiques
Au vu du contexte, un certain nombre de constatation pouvait être fait rapidement :

* des cycles de développement très court,
* un grand nombre de développeurs dont il fallait intégrer le code,
* un environnement très hétérogène (eclipse, netbeans, intelliJ idea pour la partie IDE, des PCs sous Windows, * des PCs sous Linux et des macs pour la partie matérielle),
* une procédure de livraison difficile...

En effet, pour revenir sur ce dernier point, les développeurs utilisaient, pour développer, pluto (le moteur de portlet de référence) en raison de machines pas suffisamment véloces et déployaient en local en utilisant directement le plugin mis à disposition par Jetspeed qui construisait le portail, créait sa base de données dans derby, associait son schéma et la peuplait à l'aide de son mécanisme utilisant Torque. Les fichiers de configurations utilisés par les applications web et contenant des paramètres dépendant de l'environnement où elles étaient déployées étaient embarqués dans les jars qui étaient eux-mêmes embarqués dans les applications web (portlets et services web) (ndlr : pour rappel, Jetspeed ainsi que ses portlets sont packagés sous forme de war).

En outre, afin de permettre une meilleure évolutivité du système, les portlets n'étaient pas déployés via le mécanisme préconisé par Jetspeed (qui est de les déposer dans son répertoire deploy) mais directement déployés comme une application web normale.

En conséquence, la procédure de livraison sur le serveur de production incluait l'utilisation d'une base de données Derby, et une recompilation/redéploiement complet du portail et de ses portlets à l'aide du plugin maven Jetspeed 2 effectués après récupération des sources de notre SCM, modification des fichiers de configuration et lancement du goal install de maven 2 :
```bash
cd project
mvn install
mvn jetspeed:mvn -Pall
```

#Méthodologie et objectifs
Suite à ces constatations, mes objectifs étaient (par ordre de priorité) :

* Définir un environnement propre, c'est-à-dire ne faisant plus de choses obscures (comme la modification des web.xml des portlets, le peuplement de la base de données derby ou l'ajout dans notre conteneur de Servlets d'un contexte où était déclarée la ressource jndi pour la base de données derby). Cela était un prérequis à la mise en place d'outils d'intégration continue.
* Externaliser les fichiers de configuration se trouvant dans les jars afin de permettre à notre équipe de production de ne pas avoir à recompiler et redéployer les jars à chaque fois qu'une modification devait être effectuée sur ces derniers.
* Mettre en place un processus pour générer un livrable (par exemple sous forme de zip) à destination de notre équipe de production afin ne pas avoir à recompiler le projet (oui, oui, je sais... c'est mal mais c'est la vie! ;-) ) et surtout permettre la reproductivité de l'installation des environnements.
* Fournir, via les outils d'intégration continue, une photo journalière des développements de la veille déployées et opérationnelles (c'est à dire avec une réinitialisation et une injection de données de contenu se trouvant dans une base de données type MySQL).

De plus, en parallèle, j'avais pour objectif de fournir aux développeurs un processus de travail plus rigoureux où chaque étape était distincte afin d'avoir une meilleure maitrise des environnements :

* installation d'un conteneur de servlet spécifique au projet (c'est-à-dire avec les bon jars dans son classpath et sa configuration prédéfinie)
* création des bases de données, des schémas et injection des données
* déploiement de toutes les applications web de manière automatique
* au besoin, déploiement unitaire d'une application web

#Description de l'arborescence Maven
```bash
parent
   |-- portail
   |-- portlets
          |-- portlet1
          |-- portlet2 
   |-- ws
          |-- ws1
          |-- ws2
   |-- config
   |-- assembly
   |-- src
          |-- main
          |-- resources
                     |-- sql
```

#Mise en oeuvre
## Un environnement de développement "propre" avec Maven 2
Le but de ce point était de proposer une meilleure façon de faire pour générer les wars (ie. modules portail, portlet1, portlet2, ws1 et ws2), à savoir, n'avoir à faire qu'un mvn install pour avoir une application web qu'il était possible de déployer automatiquement dans notre conteneur de Servlet (et non plus un obscur mvn jetspeed:mvn -Pdeploy). En effet, le plugin maven2 de Jetpseed permettait de compiler, générer les wars (le sien et les portlets), et de les déployer dans le conteneur de servlet indiqué. En outre, il permettait également de créer et peupler la base de données utile à son fonctionnement ; la base de données (ie. ses accès et son type) pouvant être indiquée via un fichier de configuration. Pour ce dernier point, le peuplement était fait via la mécanique interne de Jetspeed, à savoir, l'utilisation du framework Torque.

La première tache a été de dumper la base de données afin d'obtenir un fichier sql pouvant être utilisé sur la base de données cible et afin de s'abstraire de Torque.

Une fois la base de données créée et peuplée à l'aide de ce fichier sql, elle nécessitait d'être renseignée dans le contexte de l'application web du portail. Afin d'éviter un couplage fort entre l'application web et son contexte (et ainsi permettre un déploiement à chaud), la datasource a été renseignée directement dans le fichier conf/context.xml du conteneur de servlet tomcat (bon, pour ce coup, on a parlé implémentation parce que je ne sais pas si le fonctionnement est identique pour tous les conteneurs de Servlet...) partagé par l'ensemble des applications web :
```xml
<!--?xml version='1.0' encoding='utf-8'?-->
<Context antiJARLocking="true">
 <WatchedResource>WEB-INF/web.xml</WatchedResource>
 <resource name="jdbc/jetspeed" auth="Container" factory="org.apache.commons.dbcp.BasicDataSourceFactory" type="javax.sql.DataSource" username="xxx" password="xxx" driverclassname="xxx" url="jdbc:xxx://xxx:xxx/j2" maxactive="100" maxidle="30" maxwait="10000"></resource>
</Context>
```
Ce fichier a également été mis dans le module config dans le répertoire src/main/resources.

La seconde tache a été de mettre le plugin Jetspeed dans le cycle de vie maven utilisé par défaut lors de l'exécution du goal maven compile du module portail afin de générer un war qu'il était possible de copier directement dans le répertoire adéquate du conteneur de Servlet afin de permettre son déploiement :
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemalocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
 <parent>
  ...
 </parent>
 
 <modelVersion>4.0.0</modelVersion>
 <prerequisites>
  <maven>2.0.9</maven>
 </prerequisites>
 <groupId>com.xxx</groupId>
 <artifactId>portail</artifactId>
 <name>${project.artifactId}</name>
 <url>${urlRacineSite}/${project.artifactId}</url>
 <version>0.0.1</version>
 <packaging>war</packaging>
 
 <dependencies>
  <!-- jetspeed war dependencies -->
  <dependency>
   <groupId>org.apache.portals.jetspeed-2</groupId>
   <artifactId>jetspeed-dependencies</artifactId>
   <version>${org.apache.portals.jetspeed.version}</version>
   <type>pom</type>
  </dependency>
  <dependency>
   <groupId>org.apache.portals.jetspeed-2</groupId>
   <artifactId>jetspeed</artifactId>
   <version>${org.apache.portals.jetspeed.version}</version>
   <type>war</type>
  </dependency>
  <dependency>
   <groupId>org.apache.portals.jetspeed-2</groupId>
   <artifactId>jetspeed-api</artifactId>
   <version>${org.apache.portals.jetspeed.version}</version>
   <scope>provided</scope>
  </dependency>
  <dependency>
   <groupId>org.apache.portals.pluto</groupId>
   <artifactId>pluto-container-api</artifactId>
   <version>${org.apache.pluto.version}</version>
   <scope>provided</scope>
  </dependency>
 </dependencies>
 
 <build>
  <finalName>${project.artifactId}</finalName>
  <plugins>
 
   <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
   </plugin>
 
   <plugin>
    <groupId>org.apache.portals.jetspeed-2</groupId>
    <artifactId>jetspeed-deploy-maven-plugin</artifactId>
    <version>${org.apache.portals.jetspeed.version}</version>
    <executions>
     <execution>
      <id>deploy-jetspeed-layouts</id>
      <goals>
       <goal>deploy</goal>
      </goals>
      <phase>process-resources</phase>
      <configuration>
       <targetBaseDir>
        ${project.build.directory}/${project.build.finalName}</targetBaseDir>
       <destinations>
        <local>WEB-INF/deploy/local</local>
       </destinations>
       <deployments>
        <deployment>
         <artifact>
          org.apache.portals.jetspeed-2:jetspeed-layouts:war
         </artifact>
         <destination>local</destination>
        </deployment>
       </deployments>
      </configuration>
     </execution>
    </executions>
    <dependencies>
     <dependency>
      <groupId>org.apache.portals.jetspeed-2</groupId>
      <artifactId>jetspeed-layouts</artifactId>
      <version>${org.apache.portals.jetspeed.version}</version>
      <type>war</type>
     </dependency>
    </dependencies>
   </plugin>
 
   <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
    <configuration>
     <warName>${pom.artifactId}</warName>
     <overlays>
      <overlay>
       <id>jetspeed1</id>
       <groupId>org.apache.portals.jetspeed-2</groupId>
       <artifactId>jetspeed</artifactId>
      </overlay>
      <overlay>
       <id>jetspeed2</id>
       <groupId>org.apache.portals.jetspeed-2</groupId>
       <artifactId>jetspeed</artifactId>
      </overlay>
     </overlays>
    </configuration>
   </plugin>
  </plugins>
 </build>
</project>
```

De la même manière, les portlets (et donc les modules portlet1 et portlet2) devaient pouvoir être générés avec l'exécution du goal maven2 compile (n'oublions pas que les portlets devaient pouvoir être enregistrés dans le portail Jetspeed sans avoir à être déposés dans son répertoire deploy mais juste dans le répertoire de déploiement du conteneur de Servlet). Pour ce faire, le projet maven n'avait qu'à être indiqué comme étant un type war et le web.xml contenu dans le répertoire src/main/webapps/WEB-INF n'avait qu'à être renseigné comme utilisant le Servlet controleur de Jetspeed :

```xml
<!--?xml version="1.0" encoding="UTF-8"?-->
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemalocation="http://java.sun.com/xml/ns/javaee
http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" version="2.5" xmlns="http://java.sun.com/xml/ns/javaee" id="xxx">
 
 <filter-mapping>
  <filter-name>encoding-filter</filter-name>
  <url-pattern>/*</url-pattern>
 </filter-mapping>
 
 <servlet>
  <description>
   MVC Servlet for Jetspeed Portlet Applications
  </description>
  <display-name>Jetspeed Container</display-name>
  <servlet-name>JetspeedContainer</servlet-name>
 
  <servlet-class>
   org.apache.jetspeed.container.JetspeedContainerServlet
  </servlet-class>
 
  <init-param>
   <param-name>contextName</param-name><param-value>xxx</param-value></init-param>
  <load-on-startup>0</load-on-startup>
 </servlet>
 
 <servlet-mapping>
  <servlet-name>JetspeedContainer</servlet-name>
  <url-pattern>/container/*</url-pattern>
 </servlet-mapping>
 
 <jsp-config>
  <taglib>
   <taglib-uri>http://java.sun.com/portlet</taglib-uri>
   <taglib-location>/WEB-INF/tld/portlet.tld</taglib-location>
  </taglib>
  <taglib>
   <taglib-uri>http://java.sun.com/portlet_2_0</taglib-uri>
   <taglib-location>
    /WEB-INF/tld/portlet_2_0.tld
   </taglib-location>
  </taglib>
 </jsp-config>
</web-app>
```
Grâce à ces actions simples, la simple exécution du goal maven 2 compile était suffisant pour générer les war directement exploitables dans le conteneur de Servlet.
Il est à noter que la base de données originellement peuplée par le portail ne l'est pas ici. Ce point est traité par le paragraphe suivant.

##Une gestion maitrisée des bases de données
Comme il a été indiqué dans le paragraphe précédent la base de données utilisée par Jetspeed a été dissociée de son processus de génération. Afin de permettre sa création et son peuplement simplement, le plugin maven sql associé à un profile particulié a été utilisé :
```xml
<profile>
 <id>initdb</id>
 <build>
  <plugins>
 
   <plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>sql-maven-plugin</artifactId>
    <dependencies>
     <!-- specify the dependent jdbc driver here -->
     <dependency>
      <groupId>xxx</groupId>
      <artifactId>xxx</artifactId>
      <version>${xxx.version}</version>
     </dependency>
    </dependencies>
    <configuration>
     <driver>xxx.Driver</driver>
     <url>jdbc:xxx://xxx:xxxx</url>
     <username>xxx</username>
     <password>xxx</password>
     <settingsKey>sensibleKey</settingsKey>
     <!--all executions are ignored if -Dmaven.test.skip=true -->
     <skip>${maven.test.skip}</skip>
    </configuration>
    <executions>
 
     <execution>
      <id>drop-db-after-test-j2</id>
      <phase>test-compile</phase>
      <goals>
       <goal>execute</goal>
      </goals>
      <configuration>
       <autocommit>true</autocommit>
       <sqlCommand>drop database j2</sqlCommand>
       <onError>continue</onError>
      </configuration>
     </execution>
 
     <execution>
      <id>create-db-j2</id>
      <phase>test</phase>
      <goals>
       <goal>execute</goal>
      </goals>
      <configuration>
       <url>jdbc:xxx://xxx:xxxx:j2</url>
       <autocommit>true</autocommit>
       <sqlCommand>create database j2</sqlCommand>
      </configuration>
     </execution>
 
     <execution>
      <id>create-schema-j2</id>
      <phase>test</phase>
      <goals>
       <goal>execute</goal>
      </goals>
      <configuration>
       <url>jdbc:xxx://xxx:xxxx/j2</url>
       <autocommit>true</autocommit>
       <srcFiles>
        <srcFile>src/sql/create-schema.sql</srcFile>
       </srcFiles>
      </configuration>
     </execution>
 
     <execution>
      <id>create-data-j2</id>
      <phase>pre-integration-test</phase>
      <goals>
       <goal>execute</goal>
      </goals>
      <configuration>
       <url>jdbc:xxx://xxx:xxxx/j2</url>
       <autocommit>true</autocommit>
       <srcFiles>
        <srcFile>src/sql/create-data.sql</srcFile>
       </srcFiles>
      </configuration>
     </execution>
    </executions>
   </plugin>
 
  </plugins>
 </build>
</profile>
```

Ici, les fichiers sql utilisés pour peupler la base de données Jetspeed sont issus du dump de la base alors que les fichiers de création du schéma sont issus des sources de Jetspeed (à noter qu'il aurait également été possible de les dumper) et ont été mis dans le répertoire src/main/sql du module parent.

Il est à noter également que différentes phases du cycle de vie ont été utilisés et qu'elles ne sont pas foncièrement très logique... (une création d'une base de données sur la phase test n'est pas très adéquate...). Cependant, cela a été fait en raison d'un bug sur maven qui indique qu'il n'est pas possible de prédire  l'ordonnancement de l'exécution des plugins dans une même phase. Il s'agit donc d'un contournement (certes pas très propre mais qui fonctionne... ;-) ).

##Un déploiement automatisé avec Cargo

Le plugin Jetspeed permettait un déploiement automatique dans le conteneur de Servlet indiqué. Afin de fournir un comportement plus ou moins iso-fonctionnel, j'ai utilisé le plugin maven Cargo pour permettre un déploiement sur le conteneur de Servlet cible (dans notre cas, en local). Pour ce faire, un profile maven a été utilisé :

```xml
<profiles>
 <profile>
  <id>deployer</id>
  <build>
   <plugins>
 
    <plugin>
     <groupId>org.codehaus.cargo</groupId>
     <artifactId>cargo-maven2-plugin</artifactId>
     <version>1.0</version>
     <executions>
      <execution>
       <id>undeploy</id>
       <phase>pre-integration-test</phase>
       <goals>
        <goal>undeploy</goal>
       </goals>
      </execution>
 
      <execution>
       <id>start</id>
       <phase>integration-test</phase>
       <goals>
        <goal>deploy</goal>
       </goals>
      </execution>
 
     </executions>
 
     <configuration>
      <wait>true</wait>
      <container>
       <containerId>xxx</containerId>
       <type>remote</type>
      </container>
      <configuration>
       <type>runtime</type>
       <properties>
        <cargo.remote.username>xxx</cargo.remote.username>
        <cargo.remote.password>xxx</cargo.remote.password>
       </properties>
      </configuration>
 
      <deployer>
       <type>remote</type>
       <deployables>
        <deployable>
         <groupId>xxx</groupId>
         <artifactId>xxx</artifactId>
         <type>war</type>
         <properties>
          <context>xxx</context>
         </properties>
        </deployable>
       </deployables>
      </deployer>
     </configuration>
    </plugin>
 
   </plugins>
  </build>
 </profile>
</profiles>
```

Suite à ces actions, le portail, les portlets et les autres applications web étaient déployables via les commandes :
```bash
mvn install -Pinitdb
mvn install -Pdeployer
```
En outre, la commande mvn install permettait également de générer un war directement exploitable par un conteneur de Servlet (et donc directement utilisable dans un livrable à destination de la production).
#Externaliser les fichiers de configuration

Comme il a été indiqué dans l'introduction, les fonctions métiers (disponible au travers de jars fournis via des modules maven) utilisées par les applications web étaient dépendantes de variables liées à l'environnement, ces variables étant contenues dans un fichier de configuration présent dans le jar. Afin de permettre une meilleure maitrise et une meilleure évolutivité du système, il était indispensable de sortir ces fichiers de configuration des jars et même des applications web. 

Pour ce faire, il a été décidé de les mettre dans le classpath serveur. Cependant, Tomcat 6.0 (arf, pour ce coup, je vais devoir parler implémentation... ;-) ) semble avoir désactivé par défaut cette fonctionnalité. Aussi, il a été nécessaire d'éditer le fichier catalina.properties se trouvant dans le répertoire conf de notre Tomcat et d'indiquer comme valeur à la clé shared.loader l'emplacement de nos fichiers de configuration (ici, ${catalina.home}/lib/conf).

Ces fichiers ont, bien sûr, été sortis des jars (et mis dans le répertoire src/main/resources du module config) et ont été templatisés pour permettre de modifier facilement leur contenu en fonction de l'environnement d'exécution.

En outre, les fichiers utilisés par l'application et par les tests unitaires mais dont le contenu ne pouvait pas être templatisé simplement (par exemple, la déclaration de la datasource utilisant, lorsque les jars étaient exécutés au sein du conteneur de Servlet, une référence à un annuaire jndi -et donc une ligne de configuration- et, lorsque les jars étaient utilisés unitairement, une référence directe -et donc 4 lignes de configuration-) ont été dupliqués (modulo les modifications adéquates) dans le répertoire src/test/resources du module config. Cela a permis de créer une dépendance de scope test pour les projets qui avait besoin de cette configuration pour exécuter les tests unitaires :

```xml
<dependency>
 <groupId>com.xxx</groupId>
 <artifactId>config</artifactId>
 <classifier>tests</classifier>
 <type>test-jar</type>
 <scope>test</scope>
</dependency>
```
Afin de templatiser ces fichiers de configuration, la fonctionnalité de filtrage de maven a été utilisée en complément de l'utilisation de profiles maven :

```xml
<build>
 <resources>
  <resource>
   <filtering>true</filtering>
   <directory>src/main/resources</directory>
  </resource>
 </resources>
 <filters>
  <filter>src/main/filters/filter-${environment}.properties</filter>
 </filters>
 ...
</build>
```
où les profiles étaient :
```xml
<profiles>
 <profile>
  <id>dev</id>
  <activation>
   <activeByDefault>true</activeByDefault>
  </activation>
  <properties>
   <environment>dev</environment>
  </properties>
 </profile>
 
 <profile>
  <id>dev-integ</id>
  <properties>
   <environment>dev-integ</environment>
  </properties>
 </profile>
</profiles>
```
et où les fichiers src/main/filters/filter-${environment}.properties contenaient les association clé/valeur en fonction de l'environnement.

##Mise en place d'un livrable
Suite au travail décrit dans les paragraphes précédents, cela avait permis de mettre en place un environnement de développement et de build propre. Une réflexion sur la génération d'un livrable à destination de la production était désormais possible.

Pour cela, nous avons décidé d'utiliser le plugin assembly de maven en créant un module supplémentaire (module assembly) et dont le rôle était de générer notre livrable. 

Ci-joint notre fichier assembly se trouvant dans le répertoire src/main/assembly du module assembly :
```xml
<assembly>
 <id>1.0</id>
 <formats>
  <format>zip</format>
 </formats>
 <includeBaseDirectory>false</includeBaseDirectory>
 <dependencySets>
 
  <dependencySet>
   <outputDirectory>webapps</outputDirectory>
   <outputFileNameMapping>
    ${artifact.artifactId}.${artifact.extension}
   </outputFileNameMapping>
   <includes>
    <include>
     *:war
    </include>
   </includes>
  </dependencySet>
 
  <dependencySet>
   <scope>provided</scope>
   <!-- notre pom déclarait les jars à mettre dans le classpath serveur avec 
    un scope provided -->
   <useProjectArtifact>false</useProjectArtifact>
   <useTransitiveDependencies>false</useTransitiveDependencies>
   <outputDirectory>tomcat_bin/lib</outputDirectory>
   <includes>
    <include>
     *:jar
    </include>
   </includes>
  </dependencySet>
 </dependencySets>
 <fileSets>
  <fileSet>
   <directory>../config/target/classes</directory>
   <outputDirectory>tomcat_parameters/lib/conf</outputDirectory>
   <includes>
    <include>
     **
    </include>
   </includes>
  </fileSet>
 
  <fileSet>
   <directory>../src/main/sql/j2</directory>
   <outputDirectory>portal-sql-db</outputDirectory>
   <includes>
    <include>
     **
    </include>
   </includes>
  </fileSet>
 </fileSets>
</assembly>
```
et où le pom.xml utilisant le plugin était :
```xml
<build>
 <plugins>
  <plugin>
   <artifactId>maven-assembly-plugin</artifactId>
   <configuration>
    <descriptors>
     <descriptor>src/main/assembly/src.xml</descriptor>
    </descriptors>
   </configuration>
   <executions>
    <execution>
     <id>make-assembly</id>
     <phase>package</phase>
     <goals>
      <goal>single</goal>
     </goals>
    </execution>
   </executions>
  </plugin>
 </plugins>
</build>
```
##Compilation, génération et déploiement automatisé
Cette phase faisait suite aux travaux décrits dans les paragraphes précédents et avait pour objectif de fournir une version de la veille de ce qui était présent sur le trunk du SCM, et cela, dans l'idée de permettre aux développeurs de se voir avancer et de tester de manière intégré leur code.

Pour ce faire, le plugin maven cargo a été utilisé conjointement à un profile particulier. En outre, c'est le mode remote qui a été choisi afin de permettre un déploiement des applications web au sein du conteneur de servlet local, ce conteneur de servlet ayant préalablement été modifié afin d'avoir en son sein les variables de configuration dans son classpath ainsi que ses fichiers de configuration modifiés.

```xml
<profile>
 <id>dev-integ</id>
 <build>
  <plugins>
   <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <executions>
 
     <execution>
      <id>stop-tomcat</id>
      <phase>generate-test-sources</phase>
      <goals>
       <goal>run</goal>
      </goals>
      <configuration>
       <tasks>
        <exec dir="/home/dev/tomcat/bin" executable="sh">
         <arg line="shutdown.sh">
         </arg>
         <sleep seconds="20">
         </sleep>
        </exec>
       </tasks>
      </configuration>
     </execution>
 
     <execution>
      <id>start-tomcat</id>
      <phase>integration-test</phase>
      <goals>
       <goal>run</goal>
      </goals>
      <configuration>
       <tasks>
        <exec dir="/home/dev/tomcat/bin" executable="sh">
         <arg line="startup.sh">
         </arg>
        </exec>
       </tasks>
      </configuration>
     </execution>
    </executions>
   </plugin>
 
   <plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>sql-maven-plugin</artifactId>
    <dependencies>
     <dependency>
      <groupId>bdd</groupId>
      <artifactId>bdd</artifactId>
      <version>${bdd.version}</version>
     </dependency>
    </dependencies>
 
    <configuration>
     <driver>org.bdd.Driver</driver>
     <url>jdbc:xxx://localhost:xxx</url>
     <username>xxx</username>
     <password>xxx</password>
     <settingsKey>sensibleKey</settingsKey>
     <skip>${maven.test.skip}</skip>
    </configuration>
 
    <executions>
     <execution>
      <id>drop-db-after-test-j2</id>
      <phase>test-compile</phase>
      <goals>
       <goal>execute</goal>
      </goals>
      <configuration>
       <autocommit>true</autocommit>
       <sqlCommand>drop database j2</sqlCommand>
       <onError>continue</onError>
      </configuration>
     </execution>
 
     <execution>
      <id>create-db-j2</id>
      <phase>test</phase>
      <goals>
       <goal>execute</goal>
      </goals>
      <configuration>
       <url>jdbc:postgresql://localhost:xxx:j2</url>
       <autocommit>true</autocommit>
       <sqlCommand>create database j2</sqlCommand>
      </configuration>
     </execution>
 
     <execution>
      <id>create-schema-j2</id>
      <phase>test</phase>
      <goals>
       <goal>execute</goal>
      </goals>
      <configuration>
       <url>jdbc:postgresql://localhost:xxx/j2</url>
       <autocommit>true</autocommit>
       <srcFiles>
        <srcFile>
         ${project.basedir}/../src/main/sql/j2/create-schema.sql
        </srcFile>
       </srcFiles>
      </configuration>
     </execution>
 
     <execution>
      <id>create-data-j2</id>
      <phase>pre-integration-test</phase>
      <goals>
       <goal>execute</goal>
      </goals>
      <configuration>
       <url>jdbc:postgresql://localhost:5432/j2</url>
       <autocommit>true</autocommit>
       <srcFiles>
        <srcFile>
         ${project.basedir}/../src/main/sql/j2/postgresql-data.sql
        </srcFile>
       </srcFiles>
      </configuration>
     </execution>
    </executions>
   </plugin>
 
   <plugin>
    <groupId>org.codehaus.cargo</groupId>
    <artifactId>cargo-maven2-plugin</artifactId>
    <version>1.0</version>
    <executions>
 
     <execution>
      <id>undeploy</id>
      <phase>integration-test</phase>
      <goals>
       <goal>undeploy</goal>
      </goals>
     </execution>
 
     <execution>
      <id>start</id>
      <phase>integration-test</phase>
      <goals>
       <goal>deploy</goal>
      </goals>
     </execution>
    </executions>
 
    <configuration>
     <wait>true</wait>
     <container>
      <containerId>tomcat6x</containerId>
      <type>remote</type>
     </container>
 
     <configuration>
      <type>runtime</type>
      <properties>
       <cargo.remote.username>xxx</cargo.remote.username>
       <cargo.remote.password>xxx</cargo.remote.password>
      </properties>
     </configuration>
 
     <deployer>
      <type>remote</type>
      <deployables>
       <deployable>
        <groupId>xxx</groupId>
        <artifactId>portail</artifactId>
        <type>war</type>
       </deployable>
       <deployable>
        <groupId>xxx</groupId>
        <artifactId>portlet1</artifactId>
        <type>war</type>
       </deployable>
       <deployable>
        <groupId>xxx</groupId>
        <artifactId>ws1</artifactId>
        <type>war</type>
       </deployable>
       ...
      </deployables>
     </deployer>
    </configuration>
   </plugin>
  </plugins>
 </build>
</profile>
```

Il est à noter qu'ici, les phases utilisés ne sont pas foncièrement cohérente... en effet, j'ai supprimé quelques petites actions afin de ne pas géner à la visibilité ;-)
#Conclusion
Il a été vu dans les paragraphes précédents que maven 2 a servi de point central à la mise en place de l'environnement de développement même s'il est vrai que je n'ai pas utilisé pleinement ses fonctionnalités (les goals deploy ou release par exemple)...

En outre, je n'ai pas mentionné les outils d'intégration continue qui, bien sûr, exécutait périodiquement certaines de ces phases ainsi qu'une phase transverse où était exécutés des tests d'intégration et d'acceptance sur une instance de type embedded déployée via le plugin cargo.

De même, les outils mis en place de type Sonar n'ont pas été mentionnés ici : cet article n'avait pour but que de présenter une mise en oeuvre d'un environnement de développement (bonne ou mauvaise, je vous laisse en juger...) qui j'espère donnera quelques idées...

#Pour aller plus loin...

* Apache Maven de N. De Loof, A. Héritier chez Pearson
* Better Builds with Maven : http://repo.exist.com/dist/maestro/1.7.0/BetterBuildsWithMaven.pdf
* Maven. Definitive Guide : http://www.sonatype.com/products/maven/documentation/book-defguide
* Site de maven : http://maven.apache.org/
* Site de Cargo : http://cargo.codehaus.org/Maven2+plugin
* Site du plugin SQL maven : http://mojo.codehaus.org/sql-maven-plugin/
* Site de Tomcat : http://tomcat.apache.org/
* Site de Jetspeed 2 : http://portals.apache.org/jetspeed-2/
* Blog de Xebia sur Cargo et Maven : http://blog.xebia.fr/2008/11/05/lintegration-continue-avec-cargo/
* Article sur Cargo : http://www.waltercedric.com/java-j2ee-mainmenu-53/361-maven-build-system/1555-deploy-to-tomcat-6-using-maven.html
* Présentation de Maven 2 par le BreizhJUG : http://sites.google.com/a/breizhjug.org/home/2008/Maven
* Présentation de l'intégration continue par le BreizhJUG : http://sites.google.com/a/breizhjug.org/home/2008/IntegrationContinue
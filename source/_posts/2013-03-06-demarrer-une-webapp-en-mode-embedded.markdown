---
layout: post
title: "Démarrer une webapp en mode embedded avec Maven"
date: 2013-03-06 15:26:29 +0100
comments: true
categories: 
- intégration continue
- java
- jetty
- maven
- tomcat
---

![left-small](http://4.bp.blogspot.com/-2c2Ie9Tla54/UTaEjXrNaiI/AAAAAAAAA20/MWhTcyyJvMg/s1600/image.png)

La mouvance actuelle dit que tout projet qui se veut un minimum industrialisé doit pouvoir détecter les anomalies au plus tôt. Pour ce faire, il est dit qu'il doit disposer de tests, qu'ils soient unitaire, d'intégration, fonctionnel ou d'acceptance.

 Pour adresser le problème des tests d'intégration, il est souvent utile de démarrer l'application cible de manière embedded. 

Cette article montrera comment il est possible de faire pour un contexte donné.

En outre, vu que ce blog me sert également d'aide mémoire, cela me donnera une excuse pour marquer noir sur blanc des informations que je peine toujours à retrouver... ;-)

Pour les habitués de ce blog (oui, je sais, cela fait un moment que je n'ai rien écrit... ;-) , le plan sera toujours le même : dans un premier temps, le contexte sera décrit puis des ébauches de solutions seront proposées jusqu'à tendre vers celle qui a été retenue.

<!-- more -->

#Contexte

Le contexte du projet est le suivant : l'application cible est composée de deux applications web qui suivent le même cycle de vie et qui sont dépendantes l'une de l'autre (au sens runtime en non compilation).

En effet, elle dispose d'une application web proposant un site web écrit en Java et utilisant un framework de type struts 2 (et qui sera nommée par la suite "application webapp") et une application web servant un ensemble de ressources en proposant une interface REST (nommée par la suite "application rest").

L'application webapp nécessitera, en plus de son rendu de page dynamique, des ressources offertes par l'application REST (appel ajax, ...).

ndlr : je sais que la mouvance actuelle dit que les framework web java sont le "mal" pour faire du web et qu'il est préférable de tendre vers une solution type framework javascript coté client. Cependant, dans notre cas, une forte contrainte SEO faisait qu'il était nécessaire de "conserver" un rendu dynamique des pages cotés serveur.

D'un point de vu technique, le projet s'appuiera sur Maven 3 (encore lui... ;-) ), et sur des technologies standards à base de Servlets (2.5+), de JSP, de JAX-RS, ... enfin, de techno standard capable de tourner sur un conteneur de Servlet classique type Tomcat 6 ou 7.

L'arborescence du projet est la suivante :

![center](http://4.bp.blogspot.com/-1N_8WSbysb8/UTZKH2xIekI/AAAAAAAAA2c/itx2Qxav2CA/s1600/tree01.png)

Avant de commencer à rentrer dans le vif du sujet, il est important de remarquer qu'il existe de nombreux plugins Maven permettant de démarrer de manière embedded un artifact maven de type war via les goals adéquates (mvn tomcat7:run, mvn jetty:run, ...). Cependant, pour rappel, notre objectif est de démarrer de manière conjointe nos deux applications web.

En outre, le but ultime (pour ceux qui ne l'auraient pas encore deviné ;-) ) étant d'exécuter de manière boite noire des tests d'integration/d'acceptance, le démarrage devra se faire dans un module Maven frère de nos deux modules webapp et restful.

L'arborescence attendue du projet est donc la suivante :

![center](http://2.bp.blogspot.com/-QYIOY3HI__s/UTZKLlWV1bI/AAAAAAAAA2k/HXtJ0QDW6BE/s1600/tree02.png)

Enfin, pour finir le tour de notre petit cahier des charges, le livrable généré ne devra pas dépendre de profils Maven particuliés (le but étant, bien sûr, de ne pas avoir un livrable dépendant d'une configuration donnée).

Cependant, afin d'avoir la main, lors des tests d'intégration/d'acceptance, sur le jeu de données qui sera injecté dans le système à tester, il devra être possible de modifier "à chaud" certaines configurations. Pour ce faire, une surcharge des fichiers de configuration des applications devra être faite.

ndlr : l'application ayant un certain existant, elle ne dispose pas de fonctionnalités comme les Profile Spring ou l'utilisation de variables systèmes : toutes les propriétés de configuration se trouvent donc dans des fichiers properties ou dans des fichiers de contexte Spring.

#Le plugin tomcat 7

![center](http://2.bp.blogspot.com/-vO0vp3GQG_4/UTbgJFKXJ5I/AAAAAAAAA3E/Z8EtqhZbPJU/s1600/tomcat.gif)

##Mise en oeuvre

Le plugin [Maven Tomcat 7](http://tomcat.apache.org/maven-plugin-2.0/index.html) est un plugin que j'apprécie pour sa simplicité d'utilisation et ses différentes features (merci [@olamy](https://twitter.com/olamy) pour me l'avoir fait découvrir/redécouvrir ;-) ) et c'est donc naturellement que c'est le premier qui a été testé. De plus, la cible de déploiement étant Tomcat, cela tombait bien ;-) .

Pour répondre à notre cas d'usage, le plugin Tomcat 7 propose le goal run-war-only qui permet, via la configuration `<warDirectory>`, de préciser un répertoire où se trouve l'application web.

En effet, le war étant généré dans un module maven frère de celui où doit être démarré le Tomcat embedded, il doit, préalablement, être récupéré (en évitant, bien évidemment, les chemins relatifs). En outre, pour rappel, la nécessité de surcharger la configuration de certains fichiers à fait tendre la solution vers les étapes suivantes (opération spécifique à une application web) :

* récupération du war dans le repository maven,
* dézippage du war dans le répertoire target/webapp,
* copie des fichiers de configuration permettant la surcharge des fichiers de configuration de l'application dans le répertoire target/webapp/WEB-INF/classes,
* un appel au goal run-war-only du plugin Tomcat en précisant l'emplacement du war éclaté à charger.

Cela a été fait via la configuration Maven suivante :

```xml
<plugin>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>unzip-webapp</id>
            <phase>compile</phase>
            <goals>
                <goal>unpack</goal>
            </goals>
            <configuration>
                <artifactItems>
                     <artifactItem>
                         <groupId>${project.groupId}</groupId>
                         <artifactId>webapp</artifactId>
                         <version>${project.version}</version>
                         <type>war</type>
                     </artifactItem>
                </artifactItems>
 
                <outputDirectory>${project.build.directory}/webapp</outputDirectory>
                <overWriteSnapshots>true</overWriteSnapshots>
            </configuration>
        </execution>
    </executions>
</plugin>
 
<!--use to copy test resources into webapp classpath-->
<plugin>
    <artifactId>maven-resources-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-webapp-resources</id>
            <phase>process-test-resources</phase>
            <goals>
                <goal>copy-resources</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.directory}/webapp/WEB-INF/classes</outputDirectory>
                <resources>
      <resource>
          <directory>src/test/resources/webapp</directory>
          <filtering>true</filtering>
      </resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
 
<plugin>
    <groupId>org.apache.tomcat.maven</groupId>
    <artifactId>tomcat7-maven-plugin</artifactId>
    <version>2.0</version>
 
    <executions>
        <execution>
            <id>tomcat-war-exec</id>
            <goals>
                <goal>run-war-only</goal>
            </goals>
            <phase>pre-integration-test</phase>
            <configuration>
                <warDirectory>${project.build.directory}/webapp/</warDirectory>
                <fork>true</fork>
                <ignorePackaging>true</ignorePackaging>
                <contextFile>src/test/resources/webapp/context.xml</contextFile>
            </configuration>
        </execution>
        <execution>
            <id>start-tomcat</id>
            <phase>pre-integration-test</phase>
            <goals>
                <goal>run</goal>
            </goals>
            <configuration>
            </configuration>
        </execution>
        <execution>
            <id>stop-tomcat</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>shutdown</goal>
            </goals>
        </execution>
    </executions>
 
    <configuration>
        <port>9090</port>
        <path>/</path>
    </configuration>
 
    <dependencies>
        <!-- les artéfacts des jars à mettre dans le classpath du serveur Tomcat tels que les drivers de connexion-->
    </dependencies>
</plugin>
```

Il est intéressant de noter plusieurs points :

* la précision du fichier context.xml qui contient, entre autre, le contextName de l'application web mais surtout, dans notre cas, la déclaration de notre base de données dans l'annuaire JNDI,
* le rajout éventuel de jar dans le classpath serveur,
* le fait de lancer le serveur en phase de pre-integration-test et de l'éteindre en phase de post-integration-test.

Cependant, pour ceux qui auraient suivi, je rappelle que l'application cible était composée de deux applications web : webapp et restful...

Malheureusement, sauf erreur de ma part, le plugin Maven Tomcat7 ne propose pas de déployer deux applications web simultanément lorsque ces dernières sont éclatées dans un répertoire.

##Conclusion

On a vu dans ce paragraphe comment il était possible de déployer simplement, via le plugin Maven Tomcat 7, une application web de manière embedded dans un processus Maven.

Malheureusement, le goal qui nous intéressait ne permettant pas démarrer deux applications web simultanément, il n'a pas pu répondre à notre besoin.

#Le plugin Jetty

![center](http://3.bp.blogspot.com/-9CX6HuFGgPA/UTbgPT2O6gI/AAAAAAAAA3M/JQ_lOt2NnsI/s1600/jetty_logo.png)

##Mise en oeuvre

Dans cette deuxième tentative, c'est le plugin Maven Jetty qui a été utilisé. 

Même s'il n'est pas la cible de déploiement, nos applications étant assez standards, il a été acté que le conteneur n'aurait que peu d'impacts sur les tests d'intégration/d'acceptance.

La philosophie mise en oeuvre est similaire à celle choisie avec le plugin Maven Tomcat 7, à savoir :

* récupération du war dans le repository maven,
* dézippage du war dans le répertoire target/webapp,
* copie des fichiers de configuration dans le répertoire target/webapp/WEB-INF/classes,
* appel du bon goal du plugin Jetty en précisant l'emplacement du war éclaté à charger.

Bien sûr, ici, le goal est propre au plugin Jetty, à savoir run-exploded :

```xml
<plugin>
    <groupId>org.mortbay.jetty</groupId>
    <artifactId>jetty-maven-plugin</artifactId>
    <version>7.6.9.v20130131</version>     
    <executions>
        <execution>
            <id>jetty-war-exec</id>
            <goals>
                <goal>run-exploded</goal>
            </goals>
            <phase>pre-integration-test</phase>
            <configuration>
                <daemon>true</daemon>             
                <jvmArgs><!-- eventuellement les options jvm--></jvmArgs>
                <scanIntervalSeconds>0</scanIntervalSeconds>
                <jettyConfig>${basedir}/src/test/resources/jetty.xml</jettyConfig>
                <webAppConfig>
                   <contextPath>/webapp</contextPath>
                   <jettyEnvXml>${basedir}/src/test/resources/webapp/jetty-webapp-context.xml</jettyEnvXml>
                </webAppConfig>
                <war>${project.build.directory}/webapp</war>
                <connectors>
                   <connector implementation="org.eclipse.jetty.server.nio.SelectChannelConnector">
                       <host>127.0.0.1</host>
                       <port>9090</port>
                   </connector>
                </connectors>
 
                <contextHandlers>
                   <contextHandler implementation="org.eclipse.jetty.webapp.WebAppContext">
                       <war>${project.build.directory}/restful</war>
                       <contextPath>/restful</contextPath>
                   </contextHandler>
                </contextHandlers>
 
            </configuration>
        </execution>
 
        <execution>
            <id>stop-jetty</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>stop</goal>
            </goals>
            <configuration>
                <stopPort>9999</stopPort>
                <stopKey>stopKey</stopKey>
            </configuration>
        </execution>
 
    </executions>
    <dependencies>
        <!-- eventuellement les jar à ajouter au classpath du conteneur de servlet-->
    </dependencies>
</plugin>
```

On peut remarquer que la configuration du plugin Jetty est beaucoup plus verbeuse que celle du plugin Tomcat avec, notamment, la nécessité de préciser :

* un fichier de configuration jetty permettant, dans notre cas, de préciser que Jetty doit démarrer son module pour charger l'annuaire JNDI (fichier jetty.xml),
* la déclaration d'un connecteur pour pouvoir préciser le port de lancement du conteneur de Servlet,
* le fichier jetty-webapp-context.xml pendant du context.xml de Tomcat,
* l'obligation de rajouter l'élément contextHandlers pour pouvoir déclarer la deuxième application web à démarrer.

A noter qu'il est également possible de déclarer les deux applications web via des contextHandler, permettant d'avoir une configuration plus symétrique :

```xml
<plugin>
    <groupId>org.mortbay.jetty</groupId>
    <artifactId>jetty-maven-plugin</artifactId>
    <version>7.6.9.v20130131</version>
    <executions>
        <execution>
            <id>jetty-war-exec</id>
            <goals>
                <goal>start</goal>
            </goals>
            <phase>pre-integration-test</phase>
            <configuration>
                <daemon>true</daemon>
                <jvmArgs></jvmArgs>
                <scanIntervalSeconds>0</scanIntervalSeconds>
                <jettyConfig>${basedir}/src/test/resources/jetty.xml</jettyConfig>
                <webAppConfig>
                     <jettyEnvXml>${basedir}/src/test/resources/webapp/jetty-webapp-context.xml</jettyEnvXml>
                </webAppConfig>
                <connectors>
                     <connector implementation="org.eclipse.jetty.server.nio.SelectChannelConnector">
                         <host>127.0.0.1</host>
                         <port>9090</port>
                     </connector>
                </connectors>
 
                <contextHandlers>
                     <contextHandler implementation="org.eclipse.jetty.webapp.WebAppContext">
                         <war>${project.build.directory}/webapp</war>
                         <contextPath>/webapp</contextPath>
                     </contextHandler>
 
                     <contextHandler implementation="org.eclipse.jetty.webapp.WebAppContext">
                         <war>${project.build.directory}/restful</war>
                         <contextPath>/restful</contextPath>
                     </contextHandler>
                </contextHandlers>
 
            </configuration>
        </execution>
        <execution>
            <id>stop-jetty</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>stop</goal>
            </goals>
            <configuration>
                <stopPort>9999</stopPort>
                <stopKey>stopKey</stopKey>
            </configuration>
        </execution>
 
 
    </executions>
    <dependencies>
         
    </dependencies>
</plugin>
```

A titre informatif, le fichier jetty.xml permettant de charger le module JNDI est le suivant :

```xml
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure.dtd">
 
 
<Configure id="Server" class="org.eclipse.jetty.server.Server">
    <Array id="plusConfig" type="java.lang.String">
        <Item>org.eclipse.jetty.webapp.WebInfConfiguration</Item>
        <Item>org.eclipse.jetty.webapp.WebXmlConfiguration</Item>
        <Item>org.eclipse.jetty.webapp.MetaInfConfiguration</Item>
        <Item>org.eclipse.jetty.webapp.FragmentConfiguration</Item>
        <Item>org.eclipse.jetty.plus.webapp.EnvConfiguration</Item>                  <!-- add for JNDI -->
        <Item>org.eclipse.jetty.plus.webapp.PlusConfiguration</Item>                 <!-- add for JNDI -->
        <Item>org.eclipse.jetty.annotations.AnnotationConfiguration</Item>
        <Item>org.eclipse.jetty.webapp.JettyWebXmlConfiguration</Item>
        <Item>org.eclipse.jetty.webapp.TagLibConfiguration</Item>
    </Array>
 
    <Call name="setAttribute">
        <Arg>org.eclipse.jetty.webapp.configuration</Arg>
        <Arg>
            <Ref id="plusConfig"/>
        </Arg>
    </Call>
</Configure>
```

et le fichier jetty-webapp-context.xml pendant du fichier context.xml de Tomcat est :

```xml
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure.dtd">
 
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
 
<New id="myoracle" class="org.eclipse.jetty.plus.jndi.Resource">
    <Arg>jdbc/myoracle</Arg>
    <Arg>
        <New class="org.apache.commons.dbcp.BasicDataSource">
            <Set name="driverClassName">driver.jdbc.class</Set>
            <Set name="url">jdbc:url_connection</Set>
            <Set name="username">login</Set>
            <Set name="password">password</Set>
        </New>
    </Arg>
</New>
 
</Configure>
```

##Conclusion

On a pu constater que l'utilisation du plugin Maven Jetty a parfaitement répondu à notre petit cahier des charges. C'est vrai que cela peut sembler un peu poussif mais cela est surtout dû au fonctionnement même de Jetty.

Enfin, il est à noter que la configuration présentée dans le paragraphe précédent ne fonctionne pas pour la version 6 du plugin (connecteurs et packages différents, ...) et qu'elle a été testé avec la version 7.6.9.v20130131. Normalement, cela devrait fonctionner avec la version 8 mais n'ayant pas testé, je ne pourrais pas le certifier...

#Conclusion

En conclusion, cet article avait pour objectif de présenter quelques-unes des façons de démarrer de manière embedded des applications web dans des conteneurs de Servlet légés au sein d'un processus Maven. 

Cela peut, notamment, être utile pour initialiser une ou plusieurs applications au sein du processus de tests pour, par exemple, exécuter de manière automatisée des tests d'intégration ou d'acceptance, chose qui est de plus en plus courante au sein de nos usine d'intégration continue.

Bien sûr, il existe de nombreuses autres solutions (par exemple, Arquillian) mais aussi d'autres approches (le choix retenu ici a été celui de la boite noire) qui ont toutes leurs avantages et leurs inconvénients par rapport à un besoin donné.
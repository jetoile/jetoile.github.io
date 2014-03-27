---
layout: post
title: "Petit focus sur 2 plugins Maven"
date: 2012-02-28 23:09:51 +0100
comments: true
categories: 
- intégration continue
- java
- maven
---
![left-small](http://1.bp.blogspot.com/-l-NZQRxE0vI/TevA5LfJQ_I/AAAAAAAAAXI/IP8zRk4wbeg/s1600/maven-logo-2.gif)


Bon, ça fait un moment que je n'ai rien écrit... je n'ai pas foncièrement d'excuses si ce n'est que j'ai été pas mal occupé sur un projet sur lequel j'essaierai de faire un petit retour dans un futur proche...

En fait, dans ce rapide post, je tenais à faire partager 2 "petits" plugins maven que j'ai eu l'opportunité de découvrir récemment par le biais de 2 Olivier :

* [Olivier Lamy](http://twitter.com/olamy) (http://olamy.blogspot.com/), qui est architecte open source et commiteur actif de moultes projets (je ne les citerai pas, il y en a trop... ;-) ) ~~et qui travaille chez Talend~~,
* et [Olivier Bazoud](http://twitter.com/obazoud) (http://blog.bazoud.com/) ~~qui est architecte chez Ekino et également~~ co-auteur du livre [Spring Batch In Action](http://www.manning.com/templier/) aux éditions Manning.

Les deux plugins sont les suivants :

* [Tomcat 7 Maven Plugin](http://tomcat.apache.org/maven-plugin-2.0-SNAPSHOT/tomcat7-maven-plugin/plugin-info.html)
* [Application Assembler Maven Plugin](http://mojo.codehaus.org/appassembler/appassembler-maven-plugin/)

En fait, je ne rentrerai pas en détail dans les 2 plugins Maven qui, pour un, est assez connu mais je me contenterai juste de décrire les fonctionnalités que j'ai appréciées.

N'étant pas expert sur ces derniers et n'ayant pas creusé dans tous les paramétrages, j'espère que la présentation qui suit ne dira pas trop de bêtises... ;-)

<!-- more -->

#Application Assembler Maven Plugin

##Description

[Cette description est une traduction libre de la page officielle du plugin]

Le plugin Application Assembler est un plugin maven qui permet de générer les scripts utilisés au démarrage d'une application Java. Pour ce faire, le plugin copie toutes les dépendances et les artifacts du projet dans un répertoire utilisé par le script pour la gestion du classpath de l'application à démarrer.

Les plateformes actuellement supportées sont :

* \*Nix,
* Windows NT
* Java Service Wrapper

##Cas d'utilisation

Ce plugin trouve parfaitement sa place pour générer les scripts d'exécution d'une application java standalone (ie. non exécuté au sein d'un conteneur de servlet ou d'un serveur d'application) ou utilisée dans un batch.
En fait, pour être plus précis, ce plugin dispose de 3 goals :

* `appassembler:assemble` qui permet de créer une arborescence de fichiers disposant des scripts de lancement de l'application ainsi que de toutes les dépendances nécessaires à son exécution.
* `appassembler:create-repository` qui permet de créer le répertoire disposant de toutes les librairies nécessaires au bon fonctionnement de l'application.
* `appassembler:generate-daemons` qui permet de créer une arborescence de fichiers disposant des scripts et des librairies à utiliser pour wrapper l'application dans un service via Java Service Wrapper. 

##Exemple

Pour montrer comment peut être utilisé ce plugin, je vais prendre un projet simple (accessible sur mon [github](http://github.com/jetoile/sample-assembler-plugin)) qui ne contient qu'une classe disposant d'un main et qui log un message via logback au travers de slf4j (histoire de vérifier le comportement du plugin sur les dépendances).

![center](http://1.bp.blogspot.com/-Dr-0c8AkLHs/T0tyqJW4UVI/AAAAAAAAAiY/A7dW1I3ve4Q/s1600/appassembler01.png)

où on a :
```java
package com.jetoile.maven.sample;
 
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
 
public class FooClass {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(FooClass.class);
 
    public static void main(String[] args) {
        LOGGER.info("this is FooClass sample");
    }
}
```

Cet exemple a pour objectif de montrer, dans un premier temps, ce que génère le goal assemble puis, dans un second temps, le goal `generate-daemons`.

Pour ce faire, le pom.xml suivant sera utilisé :
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>com.jetoile.maven.sample</groupId>
    <artifactId>sample-assembler-plugin</artifactId>
    <version>1.0-SNAPSHOT</version>
 
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <projectBaseUri>${project.baseUri}</projectBaseUri>
        <java.version>1.6</java.version>
 
        <slf4j.version>1.6.4</slf4j.version>
        <logback.version>1.0.0</logback.version>
    </properties>
 
    <dependencies>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback.version}</version>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
 
    <build>
        <plugins>
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
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>appassembler-maven-plugin</artifactId>
                <version>1.2</version>
                <configuration>
                    <daemons>
                        <daemon>
                            <id>mainService</id>
                            <mainClass>com.jetoile.maven.sample.FooClass</mainClass>
                            <commandLineArguments>
                                <commandLineArgument>start</commandLineArgument>
                            </commandLineArguments>
                            <platforms>
                                <platform>jsw</platform>
                            </platforms>
                            <generatorConfigurations>
                                <generatorConfiguration>
                                    <generator>jsw</generator>
                                    <includes>
                                        <include>linux-x86-32</include>
                                        <include>linux-x86-64</include>
                                    </includes>
                                </generatorConfiguration>
                            </generatorConfigurations>
                        </daemon>
                    </daemons>
                </configuration>
            </plugin>
 
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>appassembler-maven-plugin</artifactId>
                <version>1.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>assemble</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <programs>
                        <program>
                            <mainClass>com.jetoile.maven.sample.FooClass</mainClass>
                            <name>main</name>
                        </program>
                    </programs>
                    <binFileExtensions>
                        <unix>.sh</unix>
                    </binFileExtensions>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

Dans le pom, on constate que l'exécution du goal `assemble` n'est pas associé à une phase en particulier. Cependant, c'est la phase `package` qui est utilisée par défaut. Dans sa configuration (disponible [ici](http://mojo.codehaus.org/appassembler/appassembler-maven-plugin/assemble-mojo.html)), il y figure le nom de la classe qui dispose du `main` ainsi que, via l'élément `binFileExtensions`, l'extension qui doit être utilisée pour le script dans le monde *Nix.

Ainsi, à la commande suivante :
```bash
mvn package
```

on obtient :
![medium](http://4.bp.blogspot.com/-6TcmRrHkwm8/T0uMn3oeh5I/AAAAAAAAAio/VyNjedWdxm8/s1600/appassembler03.png)

où le `main.sh` est le suivant :
```bash
#!/bin/sh
# ----------------------------------------------------------------------------
#  Copyright 2001-2006 The Apache Software Foundation.
#
#  Licensed under the Apache License, Version 2.0 (the "License");
#  you may not use this file except in compliance with the License.
#  You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
#  limitations under the License.
# ----------------------------------------------------------------------------
#
#   Copyright (c) 2001-2006 The Apache Software Foundation.  All rights
#   reserved.
 
BASEDIR=`dirname $0`/..
BASEDIR=`(cd "$BASEDIR"; pwd)`
 
# OS specific support.  $var _must_ be set to either true or false.
cygwin=false;
darwin=false;
case "`uname`" in
  CYGWIN*) cygwin=true ;;
  Darwin*) darwin=true
           if [ -z "$JAVA_VERSION" ] ; then
             JAVA_VERSION="CurrentJDK"
           else
             echo "Using Java version: $JAVA_VERSION"
           fi
           if [ -z "$JAVA_HOME" ] ; then
             JAVA_HOME=/System/Library/Frameworks/JavaVM.framework/Versions/${JAVA_VERSION}/Home
           fi
           ;;
esac
 
if [ -z "$JAVA_HOME" ] ; then
  if [ -r /etc/gentoo-release ] ; then
    JAVA_HOME=`java-config --jre-home`
  fi
fi
 
# For Cygwin, ensure paths are in UNIX format before anything is touched
if $cygwin ; then
  [ -n "$JAVA_HOME" ] && JAVA_HOME=`cygpath --unix "$JAVA_HOME"`
  [ -n "$CLASSPATH" ] && CLASSPATH=`cygpath --path --unix "$CLASSPATH"`
fi
 
# If a specific java binary isn't specified search for the standard 'java' binary
if [ -z "$JAVACMD" ] ; then
  if [ -n "$JAVA_HOME"  ] ; then
    if [ -x "$JAVA_HOME/jre/sh/java" ] ; then
      # IBM's JDK on AIX uses strange locations for the executables
      JAVACMD="$JAVA_HOME/jre/sh/java"
    else
      JAVACMD="$JAVA_HOME/bin/java"
    fi
  else
    JAVACMD=`which java`
  fi
fi
 
if [ ! -x "$JAVACMD" ] ; then
  echo "Error: JAVA_HOME is not defined correctly."
  echo "  We cannot execute $JAVACMD"
  exit 1
fi
 
if [ -z "$REPO" ]
then
  REPO="$BASEDIR"/repo
fi
 
CLASSPATH=$CLASSPATH_PREFIX:"$BASEDIR"/etc:"$REPO"/org/slf4j/slf4j-api/1.6.4/slf4j-api-1.6.4.jar:"$REPO"/ch/qos/logback/logback-classic/1.0.0/logback-classic-1.0.0.jar:"$REPO"/ch/qos/logback/logback-core/1.0.0/logback-core-1.0.0.jar:"$REPO"/com/jetoile/maven/sample/sample-assembler-plugin/1.0-SNAPSHOT/sample-assembler-plugin-1.0-SNAPSHOT.jar
 
EXTRA_JVM_ARGUMENTS=""
 
# For Cygwin, switch paths to Windows format before running java
if $cygwin; then
  [ -n "$CLASSPATH" ] && CLASSPATH=`cygpath --path --windows "$CLASSPATH"
  [ -n "$JAVA_HOME" ] && JAVA_HOME=`cygpath --path --windows "$JAVA_HOME"
  [ -n "$HOME" ] && HOME=`cygpath --path --windows "$HOME"`
  [ -n "$BASEDIR" ] && BASEDIR=`cygpath --path --windows "$BASEDIR"`
  [ -n "$REPO" ] && REPO=`cygpath --path --windows "$REPO"`
fi
 
exec "$JAVACMD" $JAVA_OPTS \
  $EXTRA_JVM_ARGUMENTS \
  -classpath "$CLASSPATH" \
  -Dapp.name="main" \
  -Dapp.pid="$$" \
  -Dapp.repo="$REPO" \
  -Dbasedir="$BASEDIR" \
  com.jetoile.maven.sample.FooClass \
  "$@"
```
Après un rapide :
```bash
chmod +x target/appassembler/bin/main.sh
```
Le résultat escompté est obtenu.

Afin de tester la génération de la couche wrapper __Java Service Wrapper__, le `pom.xml` contient également la déclaration du plugin __appassembler-maven-plugin__ associé aux paramètres de configuration nécessaire à la génération du service. Pour cette partie, j'ai choisi de ne pas l'associer à une phase mais de l'exécuter manuellement :

```xml
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>appassembler-maven-plugin</artifactId>
                <version>1.2</version>
                <configuration>
                    <daemons>
                        <daemon>
                            <id>mainService</id>
                            <mainClass>com.jetoile.maven.sample.FooClass</mainClass>
                            <commandLineArguments>
                                <commandLineArgument>start</commandLineArgument>
                            </commandLineArguments>
                            <platforms>
                                <platform>jsw</platform>
                            </platforms>
                            <generatorConfigurations>
                                <generatorConfiguration>
                                    <generator>jsw</generator>
                                    <includes>
                                        <include>linux-x86-32</include>
                                        <include>linux-x86-64</include>
                                    </includes>
                                </generatorConfiguration>
                            </generatorConfigurations>
                        </daemon>
                    </daemons>
                </configuration>
            </plugin>
```
Ainsi, après l'exécution de la commande :

```bash
mvn appassembler:generate-daemons
```

on obtient :
![medium](http://2.bp.blogspot.com/-zDcT-L4IP74/T0uawGcSPCI/AAAAAAAAAiw/j8giu9U41wI/s1600/appassembler04.png)

Cependant, après un rapide chmod sur les fichiers `mainService` et `wrapper-linux-x86-64`, la première tentative d'exécution échoue...  :(

```bash
chmod +x target/generated-resources/appassembler/jsw/mainService/bin/mainService
chmod +x target/generated-resources/appassembler/jsw/mainService/bin/wrapper-linux-x86-64
./target/generated-resources/appassembler/jsw/mainService/bin/mainService start
```

![medium](http://2.bp.blogspot.com/-RJ0R6CrTvU8/T0uczKHGWqI/AAAAAAAAAi4/gd7ftUBeKh8/s1600/appassembler05.png)

Un rapide coup d'oeil au fichier `wrapper.log`... :'(

Qu'à cela ne tienne, s'il n'y manque le répertoire logs, il suffit de le créer...

![large](http://3.bp.blogspot.com/-VQFurPRDpUI/T0udmE8zNvI/AAAAAAAAAjA/LBpih32n8XI/s1600/appassembler06.png)

Arf... encore loupé...

A priori, il semble que le classpath soit on ne peut plus... foireux...

Après un rapide coup d'oeil au fichier `wrapper.conf`,  on peut remarquer les lignes suivantes :

```text
wrapper.java.mainclass=org.tanukisoftware.wrapper.WrapperSimpleApp
set.default.REPO_DIR=repo
set.default.APP_BASE=.
 
# Java Classpath (include wrapper.jar)  Add class path elements as
#  needed starting from 1
wrapper.java.classpath.1=lib/wrapper.jar
wrapper.java.classpath.2=%REPO_DIR%/com/jetoile/maven/sample/sample-assembler-plugin/1.0-SNAPSHOT/sample-assembler-plugin-1.0-SNAPSHOT.jar
wrapper.java.classpath.3=%REPO_DIR%/org/slf4j/slf4j-api/1.6.4/slf4j-api-1.6.4.jar
wrapper.java.classpath.4=%REPO_DIR%/ch/qos/logback/logback-classic/1.0.0/logback-classic-1.0.0.jar
wrapper.java.classpath.5=%REPO_DIR%/ch/qos/logback/logback-core/1.0.0/logback-core-1.0.0.jar
```
hum... l'explication est là... il semble qu'il manque le répertoire repo où les librairies sont recherchées...

Un petit coup de :

```bash
mvn appassembler:appassembler:create-repository
```
devrait permettre de créer le fichier repo dans le répertoire `target/appassembler`, et un petit cp devrait tout remettre dans l'ordre... en fait, non!

Il manque l'archetype de notre application dans le répertoire repo... (d'un coté, pour les personnes qui ont suivi, cela était visible dans un des screenshots précédents...)

Bon, je vous passe les quelques commandes à base de `mvn` et `cp` que j'ai effectué pour, au final, obtenir :

![medium](http://1.bp.blogspot.com/-5JoIX1E06E8/T0ugnyyJ4YI/AAAAAAAAAjI/b66wqmxMFtM/s1600/appassembler07.png)


##Conclusion

Ce qui m'a intéressé dans ce plugin est qu'il s'occupe de générer automatiquement les scripts nécessaires au lancement de l'application. En effet, il est toujours possible d'utiliser le plugin assembly (tel que je l'avais décrit [ici](/2010/02/de-l-du-livrable.html)), mais cette solution reste assez verbeuse et rébarbative.
Aussi, avoir la possibilité, en n'ayant qu'à appeler ou qu'à associer un goal à une phase du cycle de vie du projet pour générer les scripts est assez tentant.
En outre, le plugin permet également de wrapper l'application dans un service via Java Service Wrapper.

Par contre, il peut sembler dommage que le "livrable" (au sens naïf du terme) ainsi produit ne puisse pas être considéré comme un archetype (et être déployé automatiquement sur un repository manager afin qu'il puisse être directement utilisable par d'éventuels OPS). Cependant, concernant ce point, vu que l'on s'appuie alors sur un plugin maven (dont la version est, bien sûr, maîtrisée), l'utilisateur est garanti d'avoir un script fonctionnel et reproductible.

Concernant la partie __Java Service Wrapper__, il semble, cependant, qu'il faille "retoucher" un peu ce qui est généré puisqu'il manque le répertoire repo (utilisé par le fichier `wrapper.conf` (chargé par Java Service Wrapper)) nécessaire au chargement du classpath à l'exécution du service. 

Autre point un peu dommage est que les droits d'exécution ne soient pas directement positionnés sur les scripts .sh.

#Tomcat 7 Maven Plugin

##Description
Le plugin Tomcat7 Maven Plugin permet (pour ceux qui ne le savent pas encore... ;-) ) de déployer une application web dans le conteneur de servlets Tomcat 7 via  le goal `deploy`. 

Il permet, en outre, de démarrer directement un Tomcat de manière _embedded_ à Maven (un peu comme le plugin jetty) via le goal `run`.

Cependant, ce n'est pas pour ces fonctionnalités que je tenais à parler de ce plugin.

En effet, le plugin Tomcat7 dispose d'une fonctionnalité "amusante", à savoir la possibilité de générer un jar exécutable embarquant directement un Tomcat 7.

En outre, de manière "un peu" transverse à ce plugin, un [archetype](http://tomcat.apache.org/maven-plugin-2.0-beta-1/archetype.html) existe également. Il permet de _settuper_ un projet exposant un service REST via Apache CXF et qui dispose de tests d'intégration via Selenium.

##Cas d'utilisation
Tomcat7 Maven Plugin dispose des [goals](http://tomcat.apache.org/maven-plugin-2.0-SNAPSHOT/tomcat7-maven-plugin/plugin-info.html) suivants (que je décrirai pas...) :

* `tomcat7:deploy`
* `tomcat7:deploy-only`
* `tomcat7:exec-war`
* `tomcat7:exec-war-only`
* `tomcat7:run`
* `tomcat7:run-war`
* `tomcat7:run-war-only`
* `tomcat7:shutdown`

Concernant l'archetype, il est très bien décrit à la page suivante :
http://tomcat.apache.org/maven-plugin-2.0-beta-1/archetype.html

##Exemple

###Maven Tomcat 7 Plugin

Comme je l'ai dit précédemment, je ne reviendrai pas sur l'utilisation des goals `run` et `deploy` mais me focaliserai plutôt sur le goal `exec-war`.

Pour montrer comment peut être utilisé ce plugin, je vais prendre un projet simple (accessible sur mon [github](http://github.com/jetoile/sample-tomcat-plugin)) qui ne contient qu'un simple Servlet `FooServlet`.

![center](http://2.bp.blogspot.com/-ai5ZyP1YbgY/T0u6selcDwI/AAAAAAAAAjQ/Uq8UY04YtRw/s1600/tomcat01.png)

où on a :
```java
package com.jetoile.maven.tomcat.sample;
 
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.Writer;
 
public class FooServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Writer out = resp.getWriter();
        out.append("GET FooServlet successfully called...");
        out.flush();
        out.close();
    }
}
```
pour le `web.xml` :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">
 
    <servlet>
        <servlet-name>FooServlet</servlet-name>
        <servlet-class>com.jetoile.maven.tomcat.sample.FooServlet</servlet-class>
    </servlet>
    
    <servlet-mapping>
        <servlet-name>FooServlet</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>
</web-app>
```
Le `pom.xml` qui est utilisé est le suivant :


```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>com.jetoile.maven.tomcat.sample</groupId>
    <artifactId>sample-tomcat-plugin</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>
 
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <projectBaseUri>${project.baseUri}</projectBaseUri>
        <java.version>1.6</java.version>
 
        <servlet-api.version>3.0-alpha-1</servlet-api.version>
    </properties>
 
    <dependencies>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>servlet-api</artifactId>
            <version>${servlet-api.version}</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
 
    <build>
        <plugins>
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
                <groupId>org.apache.tomcat.maven</groupId>
                <artifactId>tomcat7-maven-plugin</artifactId>
                <version>2.0-SNAPSHOT</version>
 
                <executions>
                    <execution>
                        <id>tomcat-war-exec</id>
                        <goals>
                            <goal>exec-war</goal>
                        </goals>
                        <phase>package</phase>
                        <configuration>
                            <warRunDependencies>
                                <warRunDependency>
                                    <dependency>
                                        <groupId>com.jetoile.maven.tomcat.sample</groupId>
                                        <artifactId>sample-tomcat-plugin</artifactId>
                                        <version>${project.version}</version>
                                        <type>war</type>
                                    </dependency>
                                </warRunDependency>
                            </warRunDependencies>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

On constate que le goal `exec-war` est associé à la phase `package`.

Ainsi, en exécutant la commande suivante :

```bash
mvn package
```

on obtient :
![center](http://1.bp.blogspot.com/-2AcPO28rAXU/T0u7_kOTMYI/AAAAAAAAAjY/TTEYG1XHSUg/s1600/tomcat02.png)

Ainsi, avec la commande suivante :
```bash
cd target
java -jar sample-tomcat-plugin-1.0-SNAPSHOT-war-exec.jar
```

il devient alors possible via notre navigateur préféré (à l'url suivante : http://localhost:8080/sample-tomcat-plugin/) d'obtenir la page suivante :
![center](http://2.bp.blogspot.com/-iHTkHAqCDRQ/T0u82VVpcHI/AAAAAAAAAjg/B1DIln_qwbk/s1600/tomcat03.png)

###Maven Tomcat 7 Archetype

Ce petit paragraphe a juste pour objectif de montrer ce que génère l'archetype Tomcat7.
Aussi, à la commande :
```bash
mvn archetype:generate \
   -DarchetypeGroupId=org.apache.tomcat.maven \
   -DarchetypeArtifactId=tomcat-maven-archetype \
   -DarchetypeVersion=2.0-SNAPSHOT \
   -DarchetypeRepository=https://repository.apache.org/content/repositories/snapshots/
```

on obtient le résultat suivant :
![center](http://3.bp.blogspot.com/-fOfKG8OeGRk/T0yaZtjtEGI/AAAAAAAAAjo/CAVLKEsD7lw/s1600/tomcat06.png)

Un rapide coup d'oeil permet de se rendre compte que :

* le `module basic-api` contient les interfaces pour le webservice REST
* le `module basic-api-impl` contient l'implémentation des webservices REST
* le `module basic-webapp` contient l'application web 
* le `module basic-webapp-exec` permet de générer un jar exécutable embarquant l'application web ainsi que le nécessaire de tomcat pour le démarrer en standalone
* le `module basic-webapp-it` permet de démarrer des tests Selenium sur l'application web démarrer au sein d'un Tomcat7

Pour tester, rien de plus simple, il suffit de lancer la commande suivante :
```bash
mvn install -Pchrome
```


<div>
<iframe allowfullscreen="" frameborder="0" height="315" src="http://www.youtube.com/embed/M3b6OPj6mGo" width="560"></iframe></div>

##Conclusion

Ce qui m'a plu sur le plugin Apache Tomcat 7 (en plus, bien sûr de la possibilité de lancer l'application web de manière embedded à Maven et de pouvoir déployer la webapp sur un Tomcat existant) est la possibilité de créer un jar exécutable.

C'est vrai que je n'ai toujours pas trouvé d'utilité pour cela mais la performance méritait d'être applaudit.

Concernant l'archetype Apache Tomcat 7, cela fournit en 1 ligne de commande un excellent template de projet qui dispose de tout.

#Conclusion

Comme vous avez pu vous en rendre compte, il n'y a rien de révolutionnaire dans cet article mais je tenais à mettre en avant ces 2 plugins qui, soit on des fonctionnalités utiles dans le cadre d'un projet, soit ont quelques features intéressantes.

#Pour aller plus loin...

* Site officiel de l'archetype Maven Tomcat7 : http://tomcat.apache.org/maven-plugin-2.0-beta-1/archetype.html
* Site officiel du plugin Maven Tomcat7 : http://tomcat.apache.org/maven-plugin-2.0-SNAPSHOT/tomcat7-maven-plugin/
* Site officiel du plugin Maven AppAssembler : http://mojo.codehaus.org/appassembler/appassembler-maven-plugin/
* Blog d'Olivier Lamy : http://olamy.blogspot.com/
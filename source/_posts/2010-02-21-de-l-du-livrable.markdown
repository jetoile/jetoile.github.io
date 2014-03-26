---
layout: post
title: "De l'art du livrable"
date: 2010-02-22 21:08:47 +0100
comments: true
sharing: true
footer: true
categories: 
- java
- maven
---
![left](http://3.bp.blogspot.com/_XLL8sJPQ97g/S4Gsb3IQnEI/AAAAAAAAAIo/mXDdrzwnmL0/s320/zip.png)
Au cours de certaines missions, je suis arrivé à un constat qui était que le processus de génération d'un livrable était souvent délaissé au profit de l'effort de développement de l'application.

C'est vrai qu'il peut être concevable qu'il ne s'agit que de la dernière étape d'un processus de développement, cependant, avec l'introduction de cycles courts (agilité, ...), pouvoir fournir rapidement un livrable de qualité au client est primordial.

Dans un [post précédent](/2010/01/retour-sur-la-mise-en-uvre-d_04.html), j'avais parler de la façon dont j'avais en place maven dans un processus d'usine logicielle. J'y avais également abordé succinctement comment il était possible de générer un livrable.

Ce post présente donc plus précisément comment cela est possible avec maven (je ne reviendrai pas sur le pourquoi car cela me parait évident dans le sens où c'est ce que verra le client final et que, tout le monde le sait, la première impression est très souvent importante - un peu comme lorsque l'on offre un cadeau à quelqu'un, l'emballage à son importance même si, au final, le cadeau est pourri... ;-) - ).

<!-- more --> 
 #Cahier des charges
L'application de référence sera une application simple de type J2SE (nommé foo) qui devra pouvoir s'exécuter de manière simple en exécutant par exemple un .bat ou un .sh.

Le livrable devra contenir la javadoc de l'application ainsi que son code source.

Il pourra optionnellement venir avec un fichier README qui précisera certaines informations (version, howto, issues, ...).

Enfin, il sera supposé que le client ne dispose que d'un JRE sur la machine où sera amenée à être exécutée l'application.

Le livrable devra contenir :

* dans le répertoire lib, toutes les librairies nécessaires à l'exécution de l'application,
* dans le répertoire api, la javadoc de notre application,
* dans le répertoire source, les sources de l'application sous forme de jar,
* à la racine, le fichier README ainsi que les .bat et .sh nécessaires à l'exécution de l'application.

#Mise en oeuvre
##Methodologie
Comme vous l'aurez compris, maven 2 sera notre outil pour gérer notre compilation, l'exécution des tests unitaires (qui ne doivent pas être oubliés), et production du livrable.

Pour ce faire, notre application suivra les préconisations maven et le plugin assembly sera utilisé pour la phase de génération du livrable qui sera sous forme de zip. En outre, ce plugin sera associé à la phase maven d'installation de notre application.

##Arborescence du projet

```bash
foo
  |-- src
        |-- main
            |-- java
            |-- resources
                    |-- launcher.bat
                    |-- launcher.sh
                    |-- cp.bat
                    |-- README.txt
            |-- assembly
                    |-- src.xml
        |-- test 
            |-- java
  |-- pom.xml
```

##Fichier pom.xml
```xml
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0" xsi:schemalocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
 <modelVersion>4.0.0</modelVersion>
 <artifactId>foo</artifactId>
 <groupId>com.xxx</groupId>
 <version>1.0-SNAPSHOT</version>
 <packaging>jar</packaging>
 <dependencies>
  <dependency>
   <groupId>org.testng</groupId>
   <artifactId>testng</artifactId>
   <version>5.9</version>
   <classifier>jdk15</classifier>
   <scope>test</scope>
  </dependency>
  <dependency>
   <groupId>log4j</groupId>
   <artifactId>log4j</artifactId>
   <version>1.2.15</version>
  </dependency>
  ...
 </dependencies>
 <build>
  <finalName>foo</finalName>
  <plugins>
   <plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>2.2-beta-4</version>
    <configuration>
     <descriptors>
      <descriptor>src/main/assembly/src.xml</descriptor>
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
    <version>2.1.1</version>
    <executions>
     <execution>
      <phase>package</phase>
      <goals>
       <goal>jar</goal>
      </goals>
     </execution>
    </executions>
   </plugin>
   <plugin>
    <artifactId>maven-javadoc-plugin</artifactId>
    <version>2.6.1</version>
    <executions>
     <execution>
      <goals>
       <goal>javadoc</goal>
      </goals>
      <phase>package</phase>
     </execution>
    </executions>
   </plugin>
  </plugins>
 </build>
</project>
```

##Fichier src.xml
```xml
<assembly>
 <formats>
  <format>zip</format>
 </formats>
 <includeBaseDirectory>
  false
 </includeBaseDirectory>
 <dependencySets>
  <dependencySet>
   <scope>compile</scope>
   <useProjectArtifact>false</useProjectArtifact>
   <useTransitiveDependencies>true</useTransitiveDependencies>
   <outputDirectory>lib</outputDirectory>
  </dependencySet>
 </dependencySets>
 <fileSets>
  <fileSet>
   <directory>target/site/apidocs</directory>
   <outputDirectory>api</outputDirectory>
   <includes>
    <include>**</include>
   </includes>
  </fileSet>
 </fileSets>
 <files>
  <file>
   <source>src/main/resources/README.txt
   <destName>README.txt</destName>
  </file>
  <file>
   <source>target/sources.jar
   <destName>sources/sources.jar</destName>
  </file>
  <file>
   <source>src/main/resources/launcher.sh
   <destName>launcher.sh</destName>
   <fileMode>0755</fileMode>
  </file>
  <file>
   <source>src/main/resources/launcher.bat
   <destName>launcher.bat</destName>
  </file>
  <file>
   <source>src/main/resources/cp.bat
   <destName>cp.bat</destName>
  </file>
 </files>
</assembly>
```

##Fichier launcher.bat et cp.bat
fichier cp.bat :
```bash
set CP=%CP%;%1
```
fichier launcher.bat :
```bash
set TEST_HOME=.
set CP=libfor %%i in (%TEST_HOME%\lib\*.jar) do
call cp.bat %%i
java -classpath %CP% monpackage.ClasseMain
```

##Fichier launcher.sh
```
#!/bin/bash
export TEST_HOME=.
export TEST_CP=.
for i in ${TEST_HOME}/lib/*.jar;
    do TEST_CP=$i:${TEST_CP}
done
java -classpath ${TEST_CP}:lib monpackage.ClasseMain
```

#Conclusion

On a donc pu constater que la procédure de production d'un livrable est totalement intégrable avec maven 2 et qu'elle est extrêmement simple et peu consommatrice en temps une fois mise en place. 

En outre, cela permet de normaliser le livrable et d'être sûr de ne rien oublier d'une version à une autre... alors pourquoi s'en passer?

Il est à noter que la problématique est identique avec d'autres outils (ant, graddle, easy-ant, ...) et qu'il est indispensable de prendre en compte cette phase de développement.

#Pour aller plus loin...

* __Apache Maven__ de N. De Loof, A. Héritier chez Pearson
* Better Builds with Maven : http://repo.exist.com/dist/maestro/1.7.0/BetterBuildsWithMaven.pdf
* Maven. Definitive Guide : http://www.sonatype.com/products/maven/documentation/book-defguide
* Site de maven : http://maven.apache.org/
* Présentation de Maven 2 par le BreizhJUG : http://sites.google.com/a/breizhjug.org/home/2008/Maven
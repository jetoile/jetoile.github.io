---
title: "Petites astuces avec maven 2"
date: 2010-03-28 23:58:17 +0100
comments: true
sharing: true
footer: true
---

![left](http://maven.apache.org/images/maven-logo-2.gif)
Ce post aura pour principal objectif de répertorier un ensemble d'astuces et de pointeurs sur maven2 (cela m'évitant également de rechercher dans mes liens ;-) ). Ne voulant pas répéter ce qu'y a déjà été traité par d'autres sites ou blogs, je me contenterai seulement de faire des références.

<!-- more -->
#Descriptif des différentes phases

Page du site de maven2 décrivant le cycle de vie d'un projet :
http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html

Cette page récapitule le cycle des phases utilisés pour un packaging donné. Cela est très utile lors de l'utilisation de plugins comme maven-assembly-plugin, maven-surefire-plugin, maven-sql-plugin, maven-checkstyle-plugin ou encore cargo-maven2-plugin, en permettant de positionner son exécution sur une phase donnée.

exemple :
```xml
<plugin>
 ...
 <executions>
  <execution>
   <id>start</id>
   <phase>pre-integration-test</phase>
   <goals>
    <goal>start</goal>
   </goals>
  </execution>
 </executions>
 ...
</plugin>
```
Également un petit pointeur sur deux images le résumant :
http://www.sonatype.com/books/mvnex-book/reference/simple-project-sect-lifecycle.html
![center](http://www.sonatype.com/books/mvnex-book/reference/figs/web/simple-project_lifecyclebinding.png)

http://www.sonatype.com/books/mvnex-book/reference/figs/web/simple-project_lifecyclebinding.png

![center](http://dgouyette.developpez.com/tutoriels/java/exposer-service-crud-restful-avec-jboss-resteasy/images/Image%201.png)

#Liste de plugins maven 2 très utiles

Article du blog d'Octo Technologie sur une liste très utile de plugins maven2 :
http://blog.octo.com/maven-mes-plugins-preferes/

L'auteur de cette page nous donne la liste des plugins maven2 qu'il utilise fréquemment.

A ces derniers, j'ajouterai également :

* le plugin [maven-enforcer-plugin](http://maven.apache.org/plugins/maven-enforcer-plugin/http://maven.apache.org/plugins/maven-enforcer-plugin/) qui permet de vérifier, par exemple que chaque plugin utilisé déclare bien son numéro de version. Attention, cependant, si un IDE est utilisé : il sera nécessaire de déclarer également la version du plugin qu'il utilise (par exemple, c'est le cas pour le plugin maven-eclipse-plugin).

exemple d'utilsation :
```xml
<build>
 <plugins>
 ...
  <plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-enforcer-plugin</artifactId>
   <version>1.0-beta-1</version>
   <executions>
    <execution>
     <id>enforce-versions</id>
     <goals>
      <goal>enforce</goal>
     </goals>
     <configuration>
      <rules>
       <requirePluginVersions>
        <message>Definissez plugin.version</message>
       </requirePluginVersions>
      </rules>
     </configuration>
    </execution>
   </executions>
  </plugin>
  ...
 </plugins>
 ...
</build>
```

* le plugin [builnumber-maven-plugin](http://mojo.codehaus.org/buildnumber-maven-plugin/index.html) qui permet d'indiquer le numéro de build dans le fichier MANIFEST permettant ainsi de vérifier qu'un jar est à la bonne version (ndlr : afin d'être sur que la machine d'integration n'a pas été cassé par un petit malin sur ses tests de pirates... ;-) )

exemple d'utilsation :

```xml
<plugin>
 <groupId>org.codehaus.mojo</groupId>
 <artifactId>buildnumber-maven-plugin</artifactId>
 <version>1.0-beta-4</version>
 <executions>
  <execution>
   <phase>prepare-package</phase>
   <goals>
    <goal>create</goal>
   </goals>
  </execution>
 </executions>
 <configuration>
  <format>{0,date,yyyy-MM-dd_HH-mm-ss}</format>
  <items>
   <item>timestamp</item>
  </items>
  <doCheck>false</doCheck>
  <doUpdate>false</doUpdate>
 </configuration>
</plugin>
```

* le plugin [maven-version-plugin](http://mojo.codehaus.org/versions-maven-plugin/) dont Arnaud Héritier a parlé lors de l'excellent postcast numéro 17 des castcodeurs qui permet de modifier la version d'un projet (ie. du pom parent et des modules).

exemple de déclaration :

```xml
<plugin>
 <groupId>org.sonatype.plugins</groupId>
 <artifactId>maven-version-plugin</artifactId>
 <version>1.0</version>
</plugin>
```

exemple d'utilisation :

```bash
mvn versions:set -DnewVersion=1.1-SNAPSHOT
```

* le plugin [maven-release-plugin](http://maven.apache.org/plugins/maven-release-plugin/) qui en 1 ou 2 commandes permet de vérifier le contenu "checkouté" du SCM, de lancer la compilation du code (ainsi que d'exécuter les Tests Unitaires), d'incrémenter le numéro de version du projet, de tagger dans le SCM et de générer tout ce qui peut être utile à la livraison, etc, etc, etc... en gros de vous faire gagner du temps lors du processus de livraison.

exemple de déclaration :

```xml
<plugin>
 <groupId>org.apache.maven.plugins</groupId>
 <artifactId>maven-release-plugin</artifactId>
 <version>2.0</version>
 <configuration>
  <preparationGoals>clean verify -Plivraison</preparationGoals>
  <!-- clean et verify sont les goals de preparations par defaut -->
  <releaseProfiles>livraison</releaseProfiles>
  <!-- during release:perform, enable livraison profile -->
 </configuration>
</plugin>
```
exemple d'utilisation :
```bash
mvn release:prepare release:perform
```
#S'en sortir quand on ne comprend pas un plugin avec mvnDebug
Bon, quand on a besoin d'un plugin maven mais que la documentation vient à manquer ou que google n'est pas gentil (sic.), il reste toujours la solution de rentrer dans le code du plugin. Pour ce faire, maven, en plus des scripts mvn.bat ou mvn, vient également avec les scripts mvnDebug et mvnDebug.bat.

Ces derniers scripts permettent de rentrer en remote debug dans votre exécution maven, permettant ainsi, après récupération du code source du plugin de suivre son déroulement en utilisant votre IDE préféré et en mettant vos points d'arrêt où bon vous semble.

Par défaut, mvnDebug attend que votre debugger soit branché pour continuer l'exécution et émet ses informations sur le port 8000 :
```bash
MAVEN_DEBUG_OPTS="-Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=8000"
```
Avec ça, plus d'excuse pour dire que l'on ne sait pas ce que fait un plugin! ;-)
#Ne pas commiter les fichiers de travail de maven dans Subversion
Article du blog d'Arnaud Héritier sur une petite astuce pour créer les .ignore :
http://blog.aheritier.net/subversion-exclure-en-masse-les-fichiers-et-repertoires-generes-dun-projet-maven/

Cette page fournit une petite astuce pour ne plus avoir à gérer les svn ignore. Pas testé pour ma part mais j'essaierai!

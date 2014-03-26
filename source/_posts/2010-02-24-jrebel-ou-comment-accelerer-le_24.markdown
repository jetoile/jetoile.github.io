---
layout: post
title: "JRebel ou comment accélérer le développement"
date: 2010-02-24 23:19:33 +0100
comments: true
sharing: true
footer: true
categories: 
- java
- jrebel
- maven
---
![left-small](http://3.bp.blogspot.com/_XLL8sJPQ97g/S4VHVRtCwTI/AAAAAAAAAI4/TexL6XUuprc/s200/jrebel.png)

Dans des posts précédents ([ici](/2010/01/retour-sur-la-mise-en-uvre-d_04.html) et [là](http://jetoile.blogspot.com/2010/02/de-l-du-livrable.html)), j'avais parlé d'une façon d'utiliser maven 2 pour fournir, entre autre, une solution pour accélérer le déploiement d'applications web avec Cargo. Cependant, afin d'optimiser le temps de développement, il est préférable, plutôt que d'avoir à redéployer l'application web à chaque modification de son contenu (que ce soit sa vue, son contrôleur ou son modèle) ou de ses librairies tierces (ce qui est connu pour être un anti-pattern), de n'avoir pas à le faire mais d'avoir plutôt un mécanisme permettant de prendre les modifications à chaud afin de pouvoir tester le plus rapidement possible.

Il existe différentes approches telles que :

* l'utilisation du plugin WTP d'Eclipse en lançant le conteneur de Servlet ou le serveur d'application directement au sein d'Eclipse,
* l'utilisation du [plugin Sysdeo](http://www.eclipsetotale.com/tomcatPlugin.html) sous Eclipse pour le conteneur de Servlet Tomcat,
* ...

Cependant, aucune de ces solutions ne m'avait convaincu et je continuais à utiliser la bonne vieille ligne de commande.

Jusqu'au jour où j'ai entendu parlé de JRebel... Cette solution, bien que payante, remporte l'unanimité des suffrages dans la communauté open-source en raison de sa simplicité et de sa puissance.

Suite à l'obtention d'une licence gracieusement offerte par [ZeroTurnaround](http://www.zeroturnaround.com/) lors du 2ième anniversaire du [Paris JUG](http://www.parisjug.org/) (merci à eux et longue vie aux JUGs!!), je ne pouvais que tester à mon tour...

Cet article va donc donner mon retour d'expérience.
<!-- more -->
#JRebel, quesako?
JRebel de ZeroTurnaround, anciennement JavaRebel, est un outil de développement permettant de prendre en compte des modifications de code sans avoir à redéployer l'application ou à redémarrer le serveur. Pour ce faire, il utilise un agent java qui lui permet d'instrumenter le classloader de la JVM (opération rendu possible depuis java 5).

Ainsi, lorsque qu'une classe est chargée, JRebel essaie d'y associer un fichier .class qui est recherché dans le classpath ainsi que dans les répertoires cibles précisés par le fichier de configuration `rebel.xml`. Si un fichier .class est trouvé, JRebel instrumente la classe chargée et y associe le fichier trouvé. Le timestamp du fichier .class est alors monitoré afin de détecter les changements et les modifications sont propagées au travers du class loader à l'application.

En outre, JRebel permet de monitorer les fichiers .class présents dans les JARs s'ils possèdent un fichier rebel.xml.

Enfin, JRebel permet également la prise en compte d'autres types de ressources telles que les CSS, javascript, HTML, XML, JSP et properties.

Ce fichier `rebel.xml` contient, en fait, l'emplacement des ressources à surveiller, c'est à dire quelque chose de la forme :
```xml
<application xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://www.zeroturnaround.com" xsi:schemalocation="http://www.zeroturnaround.com/alderaan/rebel-2_0.xsd">
 <classpath>
  <dir name="/home/khanh/eclipse-workspace3/module-parent/module-webapp/target/classes">
  </dir>
 </classpath>
 <web>
  <link target="/">
   <dir name="/home/khanh/eclipse-workspace3/module-parent/module-webapp/src/main/webapp">
   </dir>
  </link>
 </web>
</application>
```

Cependant, il est important de préciser que lorsque des changements ont lieu dans une classe, JRebel préserve toutes les instances existantes de la classe. Cela permet à l'application de continuer à fonctionner mais cela signifie également que l'ajout d'une nouvelle variable d'instance ne sera pas initialisé puisque le constructeur ne sera pas réexécuté. Il est toutefois possible de forcer le rechargement d'une classe à l'aide de frameworks comme Tapestry 5.

En fait, JRebel peut être vu comme un HotSwap++ puisqu'il permet, en plus, de prendre en compte le changement de la structure d'une classe, l'ajout de méthodes, de variables, de constructeurs, d'annotations et même l'ajout de classes ou le changement de configuration (pour rappel, HotSwap ne permet que de prendre en compte le changement du corps d'une méthode). Enfin, JRebel n'est pas dépendant d'un IDE.

Il est cependant important de noter que :

* JRebel peut ralentir l'application au démarrage,
* que des warnings (pouvant être ignorés) peuvent apparaitre avec certains IDEs puisque JRebel remplace le HotSwapping,
* qu'en mode debug, "this" doit être remplacé par "that" en raison de limitations de certains debugger (comme Eclipse).

En conclusion, pour utiliser JRebel, il est nécessaire :

* de démarrer l'application ou le conteneur de Servlet ou Serveur d'application en précisant à la JVM d'utiliser l'agent JRebel,
* d'ajouter dans le répertoire WEB-INF/classes (pour un war) ou racine (pour un jar) le fichier rebel.xml contenant l'emplacement des fichiers susceptibles de changer (classes, ressources, ...) qui se trouve être généralement le répertoires build ou target/classes (pour maven).

#JRebel, mise en oeuvre
Pour expliquer une mise en œuvre de JRebel, je prendrai un exemple simple (qui, je pense, couvrira le besoin de la majorité des développeurs) : une application web embarquant un certain nombre de librairies métiers "maison" déployée dans un conteneur de Servlet. Cette application web pouvant être indifféremment un service web ou une application ayant pour objectif de fournir un rendu graphique dans un navigateur web (ie. pouvant contenir des jsp, css, javascript, ...). L'objectif étant, ici, de montrer comment l'application web est capable de se recharger à chaud afin de prendre en compte les modifications du développeur.

Cette application web, développé avec maven 2, se composera de deux modules :
```bash
module-parent
   |-- module-metier
            |-- src
                  |-- main
                          |-- java
            |-- pom.xml
   |-- module-webapp
            |-- src
                 |-- main
                         |-- java
                         |-- resources
                         |-- webapp
                                  |-- WEB-INF
                                            |-- jsp
                                            |-- css
                                            |-- js
            |-- pom.xml
   |-- pom.xml
```

Je ne rentrerai pas dans la procédure d'installation de JRebel puisque cela est très bien expliqué et qu'un wizard guide l'utilisateur en fonction de ses besoins (configuration de son IDE, mode de fonctionnement du serveur d'application ou du conteneur de Servlet - ie. embarqué par l'IDE ou en standalone - , plugin nécessaire à maven 2 pour générer les fichiers rebel.xml, syntaxe de ces fichiers rebel.xml, ...).

Dans notre cas, il sera donc nécessaire d'instrumenter le jar métier issu du module métier et embarqué par l'application web avec le fichier rebel.xml. Ce fichier devra se trouver à la racine du jar produit. De la même manière, l'application web devra embarqué son fichier rebel.xml dans son répertoire WEB-INF/classes. Ce sont ces fichiers rebel.xml qui sont la clé de tous...

Plutôt que de générer ces fichiers manuellement, le plugin maven ~~javarebel-maven-plugin~~ jrebel-maven-plugin sera utilisé :

```xml
<build>
 <plugins>
  <plugin>
   <groupId>org.zeroturnaround</groupId>
   <artifactId>jrebel-maven-plugin</artifactId>
  </plugin>
 </plugins>
</build>
```
Cependant, le fichier rebel.xml, comme vu dans le paragraphe précédent, contient l'emplacement où se trouvent les ressources (au sens large du terme). Afin d'éviter d'être dépendant d'un chemin en dur, il est possible d'utiliser une variable qui devra soit être déclarée comme variable système, soit déclarée dans le fichier de configuration de JRebel (fichier `jrebel.properties` se trouvant dans le répertoire conf où a été installé JRebel).

extrait du pom du module parent :

```xml
<project>
 <modelVersion>4.0.0</modelVersion>
 <artifactId>module-parent</artifactId>
 <groupId>com.xxx</groupId>
 <version>1.0-SNAPSHOT</version>
 <packaging>pom</packaging>
 <dependencies>
  ...
 </dependencies>
 <build>
  <pluginManagement>
   <plugins>
    <plugin>
     <groupId>org.zeroturnaround</groupId>
     <artifactId>jrebel-maven-plugin</artifactId>
     <version>1.0.8</version>
     <executions>
      <execution>
       <id>generate-rebel-xml</id>
       <phase>process-resources</phase>
       <goals>
        <goal>generate</goal>
       </goals>
      </execution>
     </executions>
    </plugin>
   </plugins>
  </pluginManagement>
 </build>
 <modules>
  <module>module-metier</module>
  <module>module-webapp</module>
 </modules>
</project>
```
On remarquera que la génération du fichier rebel.xml a été positionnée sur la phase `generate-resources` de maven afin de le regénérer à chaque fois qu'un packaging est effectué.

extrait du pom du module métier :
```xml
</parent>
<modelVersion>4.0.0</modelVersion>
<artifactId>module-metier</artifactId>
 
<dependencies>
 ...
</dependencies>
 
<build>
 <plugins>
  <plugin>
   <groupId>org.zeroturnaround</groupId>
   <artifactId>jrebel-maven-plugin</artifactId>
   <configuration>
    <relativePath>../</relativePath>
    <rootPath>${eclipse-workspace.root}</rootPath>
   </configuration>
  </plugin>
 </plugins>
</build>
```
extrait du pom du module webapp :
```xml
<parent>
 <groupId>com.xxx</groupId>
 <artifactId>module-parent</artifactId>
 <relativePath>../pom.xml</relativePath>
 <version>1.0-SNAPSHOT</version>
</parent>
<modelVersion>4.0.0</modelVersion>
<artifactId>module-webapp</artifactId>
<packaging>war</packaging>
 
<dependencies>
 <dependency>
  <artifactId>module-metier</artifactId>
  <groupId>com.xxx</groupId>
  <version>1.0-SNAPSHOT</version>
 </dependency>
 ...
</dependencies>
 
<build>
 <plugins>
  <plugin>
   <groupId>org.zeroturnaround</groupId>
   <artifactId>jrebel-maven-plugin</artifactId>
   <configuration>
    <relativePath>../</relativePath>
    <rootPath>${eclipse-workspace.root}</rootPath>
   </configuration>
  </plugin>
 </plugins>
</build>
```
Il ne reste alors plus qu'à packager notre application web instrumenté à l'aide du fichier `rebel.xml`, à la déployer et à lancer notre conteneur de servlet ou server d'application en positionnant l'agent JRebel sur la JVM.

exemple de fichier `startup-jrebel.sh` pour tomcat :
```xml
#!/bin/bash
export JAVA_OPTS="-noverify -javaagent:/opt/jrebel/jrebel.jar $JAVA_OPTS"
`dirname $0`/startup.sh $@
```

Les modifications sont alors pris à chaud par notre conteneur de Servlet ou serveur d'application!

Attention cependant à ne pas oublier de préciser la valeur de la variable que nous avons utilisé sous via un -D, soit via le fichier de configuration de JRebel.
#Conclusion

Suite à l'utilisation de JRebel, j'ai vraiment été agréablement surpris par sa facilité d'utilisation que j'ai essayé de retranscrire ici.

Cependant, JRebel ne doit pas être vu non plus comme un outil miracle mais plutôt juste comme un accélérateur de développement. De part son fonctionnement, il possède un certain nombre de limitations mais reste tout de même plus naturel, plus puissant et plus simple d'utilisation que le mode debugger qui permet via le mécanisme de HotSwap de prendre à chaud certaines modifications du code (pour ma part, je ne branche le débuggeur qu'au besoin).

Cependant, il est tout de même conseiller de se référer à la documentation officielle dans le cas d'usage plus complexe (avec la possibilité de tuner plus finement le fichier rebel.xml entre autre).
En outre, il est important de garder en mémoire certaines de ses limitations (il faut, par exemple, forcer le rechargement des fichiers de contexte spring pour que certaines modifications soient pris en compte).
#Pour aller plus loin...

* Site de ZeroTurnaround : http://www.zeroturnaround.com/
* Page de FAQ de JRebel : http://www.zeroturnaround.com/jrebel/faq/
* Article du blog de ZeroTurnaround : http://www.zeroturnaround.com/blog/reloading_java_classes_401_hotswap_jrebel/
* Page des plugins supportés par JRebel : http://www.zeroturnaround.com/jrebel/plugins/
* Blog parlant de JRebel : http://codemunchies.com/2010/02/reduce-development-time-with-jrebel/

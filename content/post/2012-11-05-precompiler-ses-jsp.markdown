---
title: "Précompiler ses jsp"
date: 2012-11-05 12:57:40 +0100
comments: true
tags: 
- maven
- java
- ant
- intégration continue
- jsp
- tomcat
---
![left-small](http://1.bp.blogspot.com/-9Hhc_8qdf1Y/UIhESh6Ls9I/AAAAAAAAAso/2fJIJUvqvIo/s1600/title.png)

Parce qu'il est parfois nécessaire de valider ses jsp, cet article fera un petit retour sur comment cela peut être mis en oeuvre.

Oui, je sais, les jsp c'est _has been_ me diront certains... Cependant, il ne faut pas oublier que dans le monde Java, cela reste une des technologies indispensables quelle que soit le framework utilisé (j'entends par là dans le cas où les pages sont rendus dynamiquement par une technologie java).

Ayant eu récemment à intégrer une phase de validation des jsp maintenus principalement par des équipes "front" n'ayant que peu de connaissance en Java, ce rapide retour d'expérience tentera d'exposer les quelques points qui ont pu me poser problème et en l'occurrence les quelques-unes des différentes solutions possibles pour traiter ce point.

Aussi, dans une première partie, je présenterai le contexte, pour ensuite aborder trois des solutions possibles (et relativement récentes car il en existe plusieurs dans différentes versions...) pour valider les jsp.

<!-- more -->

# Contexte

Cette partie présentera le contexte de l'application web où j'ai eu à mettre en oeuvre une validation des jsp. Le but de l'application web étant d'offrir aux utilisateurs un site web, elle disposait de nombreuses pages jsp.

Coté organisation, les équipes étaient divisées en 2 :

* l'équipe Java dont la charge était de fournir toute la partie controleur/modèle ainsi qu'un premier jet de la couche présentation au travers de jsp réduites à leurs plus simples appareils,
* l'équipe front dont la charge était de décorer les pages jsp avec toutes les parties CSS et Javascript.

Coté technologie, le site tournait sur un Tomcat 6 avec du Java 7. Pour la partie build/packaging, Maven 3 était de la partie et, pour la partie usine logicielle, un serveur Jenkins.

Le site étant assez vieux, il s'appuyait sur un framework de présentation maison ainsi que sur un framework de décoration : [Sitemesh 2](http://wiki.sitemesh.org/display/sitemesh/Home).

La partie compilation des jsp ayant pour objectif de prévenir au plus tôt l'équipe de développeurs front de la non conformité (au sens compilation du terme) d'une page jsp, le but était de créer un job Jenkins dont la seule tâche était de compiler les jsp.

Embarquer les jsp pré-compilés dans le livrable final (ie. le war) n'était donc pas la cible.

# Mise en oeuvre

## jspc-maven-plugin

### Mise en oeuvre

Dans ma première tentative de mise en oeuvre, j'ai utilisé le goal compile du plugin __jspc-maven-plugin__ pour tomcat6 :

* http://mojo.codehaus.org/jspc-maven-plugin/usage.html

* http://mojo.codehaus.org/jspc/jspc-maven-plugin/usage.html

Rien de bien compliqué si on regarde la documentation officielle puisqu'il n'y a cas déclarer dans le pom les informations suivantes :

```xml
<profiles>
    <profile>
        <id>jsp-compile-jspc-maven</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>jspc-maven-plugin</artifactId>
                    <version>2.0-alpha-3</version>
                    <executions>
                        <execution>
                            <id>pre-compile-jsp</id>
                            <goals>
                                <goal>compile</goal>
                            </goals>
                            <phase>test</phase>
                            <configuration>
                            </configuration>
                        </execution>
                    </executions>
                    <dependencies>
                        <dependency>
                            <groupId>org.codehaus.mojo.jspc</groupId>
                            <artifactId>jspc-compiler-tomcat6</artifactId>
                            <version>2.0-alpha-3</version>
                        </dependency>
 
                    </dependencies>
                </plugin>
 
            </plugins>
        </build>
 
    </profile>
    ...
</profiles>    
...
<build>
    <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.3.2</version>
        <configuration>
            <target>1.7</target>
            <source>1.7</source>
        </configuration>
    </plugin>
    </plugins>
</build>
 
<dependencies>
    <dependency>
        <groupId>opensymphony</groupId>
        <artifactId>sitemesh</artifactId>
        <version>2.4.2</version>
    </dependency>
 
    <dependency>
        <groupId>jstl</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>
</dependencies>
```

On constate que les dépendances à sitemesh ainsi qu'à la jstl ont été déclaré afin de fournir à la compilation des jsp le nécessaire. De même, il est important de noter que le plugin __jspc-maven-plugin__ doit obligatoirement fournir un __jspc-compiler__ qui est spécifique à la version de tomcat utilisé.

Malheureusement, après un petit coup de :
```bash
mvn test -Pjsp-compile-jspc-maven
```
on peut se rendre compte que, si les jsp contiennent des scritlets avec du code Java 5 telle qu'une petite enum (oui, je sais, les scriptlets c'est mal... mais, dans mon cas, c'était pour tester...), on se fait joliment insulté... :
![medium](http://2.bp.blogspot.com/-E1qsxbHRQfI/UIgT-ax6M_I/AAAAAAAAArY/p1RBmt2us0o/s1600/error01.png)

Après quelques recherches, je suis tombé sur ce lien :
http://stackoverflow.com/questions/7156953/maven-jspc-should-use-external-jsp-files

Cool, peut-on se dire, il suffit de mettre la configuration de compilation en 1.5... mais ma cible n'était-elle pas le jdk 1.7?

Heureusement, le lien suivant semble annoncé que cela a été corrigé...
http://jira.codehaus.org/browse/MJSPC-54

Malheureusement, la version actuelle ne semble pas bénéficier de ces modifications (la alpha-3 date de 2008 et le ticket a été mis à jour en 2012...). Ne reste donc plus qu'à récupérer le code source du svn et à le recompiler... (`svn checkout https://svn.codehaus.org/mojo/trunk/mojo/jspc`).

Donc, après avoir modifié dans mon pom les informations adéquates, je lance la précompilation et là... miracle...!!! Ca marche ;-)

Au final, on aura donc le code suivant :
```xml
<profiles>
    <profile>
        <id>jsp-compile-jspc-maven</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>jspc-maven-plugin</artifactId>
                    <version>2.0-alpha-4-SNAPSHOT</version>
                    <executions>
                        <execution>
                            <id>pre-compile-jsp</id>
                            <goals>
                                <goal>compile</goal>
                            </goals>
                            <phase>test</phase>
                            <configuration>
                                <source>1.7</source>
                                <target>1.7</target>
                            </configuration>
                        </execution>
                    </executions>
                    <dependencies>
                        <dependency>
                            <groupId>org.codehaus.mojo.jspc</groupId>
                            <artifactId>jspc-compiler-tomcat6</artifactId>
                            <version>2.0-alpha-4-SNAPSHOT</version>
                        </dependency>
 
                    </dependencies>
                </plugin>
 
            </plugins>
        </build>
 
    </profile>
    ...
</profiles>    
...
<build>
    <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.3.2</version>
        <configuration>
            <target>1.7</target>
            <source>1.7</source>
        </configuration>
    </plugin>
    </plugins>
</build>
 
<dependencies>
    <dependency>
        <groupId>opensymphony</groupId>
        <artifactId>sitemesh</artifactId>
        <version>2.4.2</version>
    </dependency>
 
    <dependency>
        <groupId>jstl</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>
</dependencies>
```

### Conclusion

Au final, ce plugin m'aura bien fait suer (et là, je n'ai fait qu'un rapide condenser de mes recherches et de mes galères - modification des pom en cascade, recherche internet, ...), mais bon... la solution est fonctionnelle même si elle demande une petite recompilation du plugin.

Cependant, le plugin __jspc-maven-plugin__ ne propose pas (si j'ai bien regardé le code source) d'exclure certaines pages de la compilation, chose qui m'était nécessaire... :'(

Du coup, cette solution n'a pas été retenue...

## jsp-compile-maven-jetty

### Mise en oeuvre

Dans cette deuxième tentative de mise en oeuvre, j'ai utilisé le goal `jspc` du plugin __jsp-compile-maven-plugin__. On peut se demander pourquoi choisir le plugin Jetty alors que la cible était Tomcat. La réponse est que, vu que ce qui m'intéressait était la partie compilation pure et non la partie packaging, cela m'était égal.

Pour mettre en place, ce plugin, il suffit de suivre la documentation officielle :
http://docs.codehaus.org/display/JETTY/Maven+Jetty+Jspc+Plugin

```xml
<profiles>
    <profile>
        <id>jsp-compile-maven-jetty</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.mortbay.jetty</groupId>
                    <artifactId>jetty-jspc-maven-plugin</artifactId>
                    <version>8.1.7.v20120910</version>
                    <executions>
                        <execution>
                            <id>pre-compile-jsp</id>
                            <goals>
                                <goal>jspc</goal>
                            </goals>
                            <phase>test</phase>
                            <configuration>
                                <includes/>
                                <excludes/>
                                <verbose>true</verbose>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
 
            </plugins>
        </build>
 
    </profile>
    ...
</profiles>    
...
<build>
    <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.3.2</version>
        <configuration>
            <target>1.7</target>
            <source>1.7</source>
        </configuration>
    </plugin>
    </plugins>
</build>
 
<dependencies>
    <dependency>
        <groupId>opensymphony</groupId>
        <artifactId>sitemesh</artifactId>
        <version>2.4.2</version>
    </dependency>
 
    <dependency>
        <groupId>jstl</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>
</dependencies>        
```
Malheureusement, même en épluchant la documentation et le code source, on constate qu'il n'est pas possible de spécifier la version du compilateur.

Aussi, avec mon code de test qui utilise de superbes "switch case" avec des String seulement disponible à partir de Java 7, j'obtiens le résultat suivant :
![medium](http://2.bp.blogspot.com/-MM9hVOZq5sg/UIgxyuqIjFI/AAAAAAAAAsA/Y_9ELLHlMM0/s1600/error02.png)

En effet, après analyses (si je ne me suis pas fourvoyé...), le code source ([git://git.codehaus.org/jetty-project.git](git://git.codehaus.org/jetty-project.git)) ne positionne pas le `compilerTarget` sur __JspC__ et donc, à part hacker le plugin, impossible de surmonter ce manque...

### Conclusion

On a vu que le plugin __jetty-jspc-maven-plugin__ était très simple à mettre en oeuvre mais qu'il manquait quelques fonctionnalités comme la possibilité de compiler avec une version supérieure au jdk 1.5.

Du coup, cette solution n'a pas été retenue...

## Ant

### Mise en oeuvre

Dans cette troisième tentative de mise en oeuvre, j'ai gentiment suivi les préconisations présentes sur le site de Tomcat pour précompiler les jsp (http://tomcat.apache.org/tomcat-6.0-doc/jasper-howto.html) :

```xml
<profiles>
    <profile>
        <id>jsp-compile-ant</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-antrun-plugin</artifactId>
                    <version>1.6</version>
                    <executions>
                        <execution>
                            <id>pre-compile-jsp</id>
                            <phase>test</phase>
                            <configuration>
                                <target name="pre-compile-jsp">
                                    <property name="maven.dependencies.classpath" refid="maven.test.classpath"/>
                                    <ant antfile="${basedir}/build.xml">
                                        <target name="all"/>
                                    </ant>
                                </target>
                            </configuration>
                            <goals>
                                <goal>run</goal>
                            </goals>
                        </execution>
                    </executions>
                    <dependencies>
                        <!-- http://docs.codehaus.org/display/MAVENUSER/Running+ant+tasks+that+use+the+JDK -->
                        <dependency>
                            <groupId>com.sun</groupId>
                            <artifactId>tools</artifactId>
                            <version>1.6.0</version>
                            <scope>system</scope>
                            <systemPath>${java.home}/../lib/tools.jar</systemPath>
                        </dependency>
 
                        <dependency>
                            <groupId>org.apache.tomcat</groupId>
                            <artifactId>jasper</artifactId>
                            <version>6.0.35</version>
                        </dependency>
 
                        <dependency>
                            <groupId>org.apache.tomcat</groupId>
                            <artifactId>jsp-api</artifactId>
                            <version>6.0.35</version>
                        </dependency>
 
                        <dependency>
                            <groupId>jstl</groupId>
                            <artifactId>jstl</artifactId>
                            <version>1.2</version>
                        </dependency>
 
                        <dependency>
                            <groupId>opensymphony</groupId>
                            <artifactId>sitemesh</artifactId>
                            <version>2.4.2</version>
                        </dependency>
                    </dependencies>
                </plugin>
 
            </plugins>
        </build>
 
        <dependencies>
            <dependency>
                <groupId>org.apache.tomcat</groupId>
                <artifactId>jasper</artifactId>
                <version>6.0.35</version>
                <scope>test</scope>
            </dependency>
 
            <dependency>
                <groupId>opensymphony</groupId>
                <artifactId>sitemesh</artifactId>
                <version>2.4.2</version>
                <scope>test</scope>
            </dependency>
 
        </dependencies>
    </profile>
    ...
</profiles>    
...
<build>
    <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.3.2</version>
        <configuration>
            <target>1.7</target>
            <source>1.7</source>
        </configuration>
    </plugin>
    </plugins>
</build>
 
<dependencies>
    <dependency>
        <groupId>opensymphony</groupId>
        <artifactId>sitemesh</artifactId>
        <version>2.4.2</version>
    </dependency>
 
    <dependency>
        <groupId>jstl</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>
</dependencies>        
```

avec le fichier `build.xml` suivant :

```xml
<?xml version="1.0" encoding="UTF-8"?>
 
<project name="Webapp Precompilation" default="all" basedir=".">
 
 
    <property name="webapp.path" value="${basedir}/src/main/webapp" />
    <property name="jsp.dest.compile" value="${basedir}/target/jsp-generated"/>
    <property name="jsp.dest" value="${basedir}/target/jsp"/>
    <!--<property name="tomcat.home" value="/opt/apache-tomcat-7.0.23"/>-->
    <property name="tomcat.home" value="/usr/local/tomcat60"/>
 
    <import file="${tomcat.home}/bin/catalina-tasks.xml"/>
 
    <target name="jspc">
        <jasper
                validateXml="false"
                uriroot="${webapp.path}"
                webXmlFragment="${basedir}/target/generated_web.xml"
                outputDir="${jsp.dest}"
                compiler="${java.version}"
                compilersourcevm="${java.version}"
                compilertargetvm="${java.version}"
                failonerror="true"
                verbose="9"/>
    </target>
 
    <target name="compile">
 
        <mkdir dir="${jsp.dest}"/>
        <mkdir dir="${jsp.dest.compile}"/>
 
        <javac destdir="${jsp.dest.compile}"
               optimize="off"
               fork="true"
               failonerror="true"
               srcdir="${jsp.dest}"
               memoryinitialsize="1024m"
               memoryMaximumSize="1024m">
            <classpath>
                <path path="${maven.dependencies.classpath}"/>
            </classpath>
            <include name="**" />
            <!--<exclude name="**/blabla*_jsp.java"/>-->
        </javac>
 
    </target>
 
    <target name="all" depends="jspc,compile">
    </target>
 
</project>
```

On peut constater quelques petites adaptations sur le script ant par rapport à la version proposée. Les raisons sont les suivantes :

* il ne me plaisait pas de dépendre de l'emplacement d'un conteneur de servlets pour effectuer une simple compilation : j'ai préféré déclarer les lib de Tomcat via Maven. Etant dans un profil Maven particulié, l'impact était négligeable pour le reste du projet,
* je ne voulais pas que mes ressources générées partent n'importe où : j'ai donc précisé ces éléments dans les taches jasper et compile,
* je n'ai pas pris la target cleanup car j'ai redirigé toutes mes générations dans le répertoire target géré par Maven.

Par contre, j'ai malheureusement été obligé de garder une dépendance à Tomcat pour le fichier `catalina-tasks.xml`. J'aurais pu l'embarqué en le customisant un peu et en récupérant, via Maven, les jars déclarés via la variable `CATALINA_HOME`, mais j'avoue avoir été paresseux sur ce point... par contre, je pense que c'est tout à fait possible de le faire.

On notera que la phase se fait en 2 passes :

* la première qui transforme les jsp en java (target `jspc`)
* la deuxième qui compile réellement les java en class (target `compile`)

Cependant, il reste encore un point qui mérite qu'on s'attarde un peu à cette solution... En effet, le __maven-ant-plugin__ utilisé ici pour invoquer la target Ant depuis Maven s'exécute avec la variable d'environnement `JAVA_HOME` positionnée sur le JRE : http://docs.codehaus.org/display/MAVENUSER/Running+ant+tasks+that+use+the+JDK.

Afin de palier à ce point, il convient de référencer en dépendance du plugin Ant la librairie `tools.jar`... (si ça, c'est pas laid...!!!) et, en plus, de l'utiliser avec un scope `system`... :'(

```xml
<dependency>
    <groupId>com.sun</groupId>
    <artifactId>tools</artifactId>
    <version>1.7.0</version>
    <scope>system</scope>
    <systemPath>${java.home}/../lib/tools.jar</systemPath>
</dependency>
```

### Conclusion

Au final, le plugin Ant fonctionne bien (même si je ne suis pas fan d'appeler du Ant via Maven...) et répond au besoin de mon cahier des charges.

Même si cela m'embête, c'est la solution qui a été retenue... :'(

# Conclusion

En conclusion, comme vous avez pu le constater, j'ai bien galéré avec une tache enfantine qui n'était qu'une simple validation des jsp par compilation.

Parmi les 3 solutions testées, seule une a réussi à remplir le cahier des charges qui, pourtant, était simple...

Aussi pas grand chose à conclure si ce n'est que je tenais à faire partager mes petites galères si cela pouvait servir à quelqu'un... ;-)

Le code est disponible [ici](https://github.com/jetoile/jsp-compile) avec les différents profils :

* __jsp-compile-ant__,
* __jsp-compile-jspc-maven__,
* __jspc-compile-maven-jetty__.

Autre point (transverse) concernant Sitemesh (mais là, j'y vais avec des pincettes car je connais très peu ce framework...)... :

La visibilité des variables (au sens scriptlet) n'est pas possible entre le décorateur et la page décorée puisque les jsp sont compilées avant le passage de Sitemesh (http://wiki.sitemesh.org/display/sitemesh/Flow+Diagram). Par contre, les el sont tout à fait possible d'un coté comme de l'autre.

Pour avoir un exemple là dessus, c'est accessible sur mon github (https://github.com/jetoile/jsp-compile) dans les jsp menu.jsp, include.jsp et basic-theme.jsp qui sont, elles-même issues d'un ensemble de copier/coller des tutoriels officiels.

En fait, c'est surtout ce dernier point que les différents tests présentés dans cet article voulait validé... ;-)

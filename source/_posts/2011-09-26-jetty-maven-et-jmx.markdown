---
layout: post
title: "Jetty, Maven et JMX"
date: 2011-09-26 22:19:12 +0100
comments: true
categories: 
- java
- jetty
- jmx
- maven
---

![left-small](http://2.bp.blogspot.com/-bGP4Fnvfp1s/TnEtqz-UNkI/AAAAAAAAAa4/KBXNM8QXHKc/s1600/logoArticle.png)

Vous avez peut être remarqué que ces derniers temps, j'étais très Maven et JMX. Cet article ne déroge pas à la règle puisque je vais parler de... Maven et de JMX.

Enfin pour être plus précis, je vais montrer comment il est facilement possible de déployer une application web dans le conteneur embarqué Jetty via Maven en activant la couche JMX afin de pouvoir tester de manière intégrée cette couche.

Pour ce faire, je présenterai dans un premier temps le contexte, puis comment cela peut être mise en œuvre. 
Bien sûr, cet article montre comment j'ai fait mais il ne représente pas la seule manière de faire... ;-). En outre, il ne présente rien de novateur mais je me suis dit que cela pouvait toujours être utile afin d'éviter de faire perdre du temps à d'autres personnes.

<!-- more -->

#Contexte

Ce paragraphe présente mes motivations pour avoir eu un besoin d'une utilisation d'un Jetty embedded à Maven en activant la couche JMX car, c'est vrai, ce n'est pas un besoin très courant et qu'il peut même sembler, au premier abord, paraitre un peu stupide (en effet, utiliser un Jetty embedded répond plutôt à un besoin de poste de développement ou au pire (et je dis bien au pire!!) à un pseudo TU qui n'est, du coup, pas un TU... (ndlr : enfin, encore une fois, je diverge... ;-) )).

En fait, pour les besoins d'une démonstration, je voulais juste présenter une petite application web qui remontait des métriques via JMX. N'ayant pas foncièrement envie d'avoir à décompresser un Jetty sur mon poste, je me suis dit qu'un Jetty embedded dans Maven suffisait largement à ma petite démo.

En outre, cela offrait également la possibilité à n'importe qui de rejouer la démo sans avoir à installer localement un conteneur de Servlets et sans à avoir à modifier le paramétrage de ce dernier pour activer la couche JMX de la JVM.

L'application web utilisée pour ce petit POC est assez basique puisqu'il ne s'agit que d'un simple Servlet redirigeant vers une simple jsp.

Ce Servlet incrémentera un compteur exposé en JMX afin de fournir cette métrique basique à la couche de supervision.

Spring 3 sera également utilisé pour lier le tout...

En fait, si vous me demandez pourquoi j'ai branché Spring pour une application ausi simple, c'est, en fait, que JMX n'était pas le seul composant que je voulais montrer pendant la petite démonstration...

Si vous avez l'esprit chagrin et que vous me re-demandez pourquoi je n'ai pas utilisé un aspect, c'est que l'aspect était justement l'autre aspect de ma démo ;-).

#Mise en oeuvre
Pour ce paragraphe de mise en œuvre, je présenterai successivement :

* Le code de l'application web utilisé.
* La configuration Maven utilisée permettant de démarrer l'application web dans un Jetty embedded.
* Les subtilités de la configuration à effectuer pour ajouter la couche JMX.

#Application web cible

Comme annoncé précédemment, notre application web cible se compose de :

* un Servlet,
* une page JSP,
* un POJO utilisé par Spring pour la partie MBean (Spring aura, à sa charge, sa proxification en MBean Standard et son enregistrement au sein du MBean Server),
* un fichier de configuration web.xml,
* et un fichier de contexte Spring.

Il est à noter qu'elle utilisera la spécification 2.5 des Servlets afin de permettre, dans les parties suivantes, de montrer la différence de configuration qu'il existe entre Jetty 6 et 7.

##Code du Servlet

```java
public class SimpleServlet extends HttpServlet {
    private static Logger LOGGER = LoggerFactory.getLogger(SimpleServlet.class);
 
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        ApplicationContext beanFactory = WebApplicationContextUtils.getRequiredWebApplicationContext(getServletContext());
        SimpleCounter jmxTestBean = beanFactory.getBean("simpleCounter", SimpleCounter.class);
        jmxTestBean.inc();
 
        req.setAttribute("nbInvocation", jmxTestBean.getNbGet());
 
        RequestDispatcher dispatcher = req.getRequestDispatcher("WEB-INF/jsp/simpleDisplay.jsp");
        if (dispatcher != null) {
            dispatcher.forward(req, resp);
        } else {
            LOGGER.warn("unable to get the request dispatcher");
        }
    }
}
```

On constate que le Servlet est basique. Il n'y a donc rien à dire dessus si ce n'est la récupération du POJO proxifié par Spring pour incrémenter un compteur qui sera exposé en JMX.

##Code du POJO utilisé comme MBean Standard

```java
public class SimpleCounter {
    private int nbGet = 0;
 
    public void inc() {
        this.nbGet++;
    }
 
    public void setNbGet(int nbGet) {
        this.nbGet = nbGet;
    }
 
    public int getNbGet() {
        return nbGet;
    }
}
```

Concernant le POJO, rien de spécial non plus à remarquer. Juste à noter que les setter et getter sont nécessaires afin de rendre l'attribut nbGet accessible en lecture/écriture par la couche cliente JMX.

##Code de la jsp
```xml
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <title>simple display</title>
    </head>
    <body>
        <h1>This is a simple display</h1>
        Has been invoked: ${nbInvocation} times
    </body>
</html>
```

Rien de particulier non plus à dire sur la jsp...

##Code du web.xml

```xml
<web-app version="2.5" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemalocation="http://java.sun.com/xml/ns/javaee 
 http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">
 <context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>classpath:applicationContext*.xml</param-value>
 </context-param>
 <listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
 </listener>
 
 <servlet>
  <servlet-name>SimpleServlet</servlet-name>
  <servlet-class>fr.jetoile.demo.servlet.SimpleServlet</servlet-class>
  <init-param>
   <param-name>sleep-time-in-seconds</param-name>
   <param-value>10</param-value>
  </init-param>
  <load-on-startup>1</load-on-startup>
 </servlet>
 
 <servlet-mapping>
  <servlet-name>SimpleServlet</servlet-name>
  <url-pattern>/sample</url-pattern>
 </servlet-mapping>
</web-app>
```

Le fichier descripteur web.xml est classique donc rien de nouveau sur les tropiques ;-). Comme dit plus haut, c'est ici la version 2.5 des servlets qui est utilisée afin de permettre un choix plus important de conteneurs de Servlets.

##Code du contexte Spring

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://www.springframework.org/schema/beans" xsi:schemalocation="http://www.springframework.org/schema/beans
 http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
 <bean class="org.springframework.jmx.support.MBeanServerFactoryBean" id="mbeanServer">
  <property name="locateExistingServerIfPossible" value="true"></property>
 </bean>
 
 <bean class="org.springframework.jmx.export.MBeanExporter" id="exporter">
  <property name="beans">
   <map>
    <entry key="bean:application=jmx-sample,name=simpleCounter" value-ref="simpleCounter"></entry>
   </map>
  </property>
   <property name="server" ref="mbeanServer"></property>
 </bean>
 
 <bean class="fr.jetoile.demo.jmx.SimpleCounter" id="simpleCounter"></bean>
</beans>
```

Concernant le fichier descripteur de Spring, on constate que la classe MBeanServerFactoryBean de Spring est utilisée comme moyen de récupération/création du MBeanServer et que c'est le MBeanExporter qui a la charge de l'enregistrement du MBean injecté.

#Application web exécutable via Jetty au sein de Maven

Afin de permettre une exécution de notre application web dans un Jetty embedded dans Maven, il faut, bien sûr, une structure Maven.

Elle est la suivante :
![center](http://3.bp.blogspot.com/-q2jJEYSygfg/TnErFlaNEDI/AAAAAAAAAag/2vgwO_zYc6A/s1600/jmx-sample-tree.png)

Concernant le fichier descripteur pom.xml, il contiendra, bien sûr, le plugin nécessaire au lancement de jetty.

Afin d'être plus exhaustif, dans notre POC, deux profils Maven seront utilisés afin de pouvoir spécifier la version de Jetty à utiliser (le choix sera laissé entre Jetty 6 et 7). En outre, le port d'écoute est forcé à 9090 :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
 <modelVersion>4.0.0</modelVersion>
 <groupId>fr.jetoile.demo</groupId>
 <artifactId>jmx-sample</artifactId>
 <packaging>war</packaging>
 <version>1.0-SNAPSHOT</version>
 
 <properties>
  <java.version>1.5</java.version>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
 
  <slf4j.version>1.6.1</slf4j.version>
  <logback.version>0.9.27</logback.version>
  <commons-lang.version>2.5</commons-lang.version>
  <spring.version>3.0.6.RELEASE</spring.version>
  <servlet.version>2.5</servlet.version>
 </properties>
 
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
   <groupId>org.springframework</groupId>
   <artifactId>spring-context</artifactId>
   <version>${spring.version}</version>
  </dependency>
  <dependency>
   <groupId>org.springframework</groupId>
   <artifactId>spring-web</artifactId>
   <version>${spring.version}</version>
  </dependency>
  <dependency>
   <groupId>javax.servlet</groupId>
   <artifactId>servlet-api</artifactId>
   <version>${servlet.version}</version>
   <scope>provided</scope>
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
    <artifactId>maven-compiler-plugin</artifactId>
    <version>2.3.2</version>
    <configuration>
     <source>${java.version}</source>
     <target>${java.version}</target>
     <encoding>UTF-8</encoding>
    </configuration>
   </plugin>
 
   <plugin>
    <artifactId>maven-resources-plugin</artifactId>
    <version>2.4.3</version>
    <configuration>
     <encoding>UTF-8</encoding>
    </configuration>
   </plugin>
  </plugins>
 </build>
  
  
 <profiles>
  <profile>
   <id>jetty6</id>
   <properties>
    <jetty.version>6.1.26</jetty.version>
   </properties>
   <build>
    <plugins>
     <plugin>
      <groupId>org.mortbay.jetty</groupId>
      <artifactId>maven-jetty-plugin</artifactId>
      <version>${jetty.version}</version>
      <configuration>
       <scanIntervalSeconds>10</scanIntervalSeconds>
       <webAppConfig>
        <contextPath>/jmx-sample</contextPath>
       </webAppConfig>
       <connectors>
         <connector implementation="org.mortbay.jetty.nio.SelectChannelConnector">
         <port>9090</port>
         <host>0.0.0.0</host>
         <maxIdleTime>60000</maxIdleTime>
        </connector>
       </connectors>  
      </configuration>
     </plugin>
    </plugins>
   </build>
  </profile>
  <profile>
   <id>jetty7</id>
   <properties>
    <jetty.version>7.4.5.v20110725</jetty.version>
   </properties>
   <build>
    <plugins>
     <plugin>
      <groupId>org.mortbay.jetty</groupId>
      <artifactId>jetty-maven-plugin</artifactId>
      <version>${jetty.version}</version>
      <configuration>
       <scanIntervalSeconds>10</scanIntervalSeconds>
       <webAppConfig>
        <contextPath>/jmx-sample</contextPath>
       </webAppConfig>
       <connectors>
           <connector implementation="org.eclipse.jetty.server.nio.SelectChannelConnector">
         <port>9090</port>
         <host>0.0.0.0</host>
         <maxIdleTime>60000</maxIdleTime>
        </connector>
       </connectors>  
      </configuration>
     </plugin>
    </plugins>
   </build>
  </profile>
 </profiles>
</project>
```
Il n'y a rien de particulier à remarquer si ce n'est une version de plugin maven pour Jetty qui diffère, ainsi qu'un changement de package pour le SelectChannelConnector utilisé pour spécifier le port d'écoute.

Pour démarrer notre Jetty, il ne reste plus qu'à exécuter, au choix, la commande :

```bash
mvn -Pjetty6 jetty:run
```

ou :

```bash
mvn -Pjetty7 jetty:run
```

##Activation de JMX

Après avoir montré l'application web ainsi que la configuration du plugin Maven de Jetty nécessaire à son exécution dans le conteneur embarqué, il est possible de démarrer l'application web dans le Jetty embedded.

![center](http://1.bp.blogspot.com/-MC5ASoy7hYY/TnEsBl97o9I/AAAAAAAAAao/wKFzpPyRaI0/s1600/visualVM.png)

![center](http://1.bp.blogspot.com/-h1-h-jZ47Ys/TnEr6OFgvnI/AAAAAAAAAak/hkrxFyewmwA/s1600/jconsole.png)
Cependant, même si nos JConsole ou VisualVM préférées proposent une connexion au MBean Server local (visible au travers notre processus Maven3 org.codehaus.plexus.classworlds.launcher.Launcher), il n'est pas possible d'effectuer une connexion en spécifiant le JMXServiceURL de type : service:jmx:rmi:///jndi/rmi://localhost:1099/jmxrmi .

En effet, par défaut, Jetty ne créé pas de MBeanServer et, à fortiori, n'enregistre aucun connecteur RMI nécessaire à une application JMX cliente.

A titre informatif, même en démarrant un serveur Jetty en mode standalone, il est nécessaire de préciser, à son démarrage, le fichier de configuration jetty-jmx.xml se trouvant, par défaut, dans son répertoire etc et qu'il faut éditer (pour préciser le port du serveur RMI à démarrer et le JMXServiceURL du connecteur JMX RMI).
Pour Jetty 7, cela peut être fait, soit en modifiant le fichier start.ini, soit en démarrant Jetty en ligne de commande :

```bash
java -jar start.jar etc/jetty-jmx.xml etc/jetty.xml
```
Pour Jetty 6, cela peut être fait, soit en modifiant le fichier bin/jetty-service.conf, soit en démarrant Jetty en ligne de commande :

```bash
java -DOPTIONS=jmx -jar start.jar etc/jetty-jmx.xml etc/jetty.xml
```

Aussi, pour activer la création d'un serveur RMI et créer un MBean Server dans le Jetty embarqué par Maven, il est nécessaire de fournir au plugin le fichier jetty-jmx.xml.

Pour ce faire, il est possible de spécifier une configuration à Jetty via l'élément : &lt;<jettyConfig&gt;
```xml
<profiles>
 <profile>
  <id>jetty6</id>
  <properties>
   <jetty.version>6.1.26</jetty.version>
  </properties>
  <build>
   <plugins>
    <plugin>
     <groupId>org.mortbay.jetty</groupId>
     <artifactId>maven-jetty-plugin</artifactId>
     <version>${jetty.version}</version>
     <configuration>
      <scanIntervalSeconds>10</scanIntervalSeconds>
      <jettyConfig>${basedir}/src/config/jetty-jmx-6.xml</jettyConfig>
      <webAppConfig>
       <contextPath>/jmx-sample</contextPath>
      </webAppConfig>
      <connectors>
        <connector implementation="org.mortbay.jetty.nio.SelectChannelConnector">
        <port>9090</port>
        <host>0.0.0.0</host>
        <maxIdleTime>60000</maxIdleTime>
       </connector>
      </connectors>
     </configuration>
    </plugin>
   </plugins>
  </build>
 </profile>
 <profile>
  <id>jetty7</id>
  <properties>
   <jetty.version>7.4.5.v20110725</jetty.version>
  </properties>
  <build>
   <plugins>
    <plugin>
     <groupId>org.mortbay.jetty</groupId>
     <artifactId>jetty-maven-plugin</artifactId>
     <version>${jetty.version}</version>
     <configuration>
      <scanIntervalSeconds>10</scanIntervalSeconds>
      <jettyConfig>${basedir}/src/config/jetty-jmx.xml</jettyConfig>
      <webAppConfig>
       <contextPath>/jmx-sample</contextPath>
      </webAppConfig>
      <connectors>
          <connector implementation="org.eclipse.jetty.server.nio.SelectChannelConnector">
        <port>9090</port>
        <host>0.0.0.0</host>
        <maxIdleTime>60000</maxIdleTime>
       </connector>
      </connectors>  
     </configuration>
    </plugin>
   </plugins>
  </build>
 </profile>
</profiles>
```

La nouvelle arborescence de notre projet devient donc :

![center](http://1.bp.blogspot.com/-isxZpdoIozo/TnHVHTqIwRI/AAAAAAAAAbA/urXO-bvkGsM/s1600/jmx-sample-tree2.png)

En outre, il est nécessaire de démarrer la JVM avec l'option -Dcom.sun.management.jmxremote :
```bash
mvn -Dcom.sun.management.jmxremote -Pjetty6 package jetty:run
```
<br/>
```bash
mvn -Dcom.sun.management.jmxremote -Pjetty7 package jetty:run
```

Suite au lancement du serveur Jetty via Maven, il est alors possible d'y accéder au travers d'un client JMX en mode distant :
![center](http://4.bp.blogspot.com/-neKAz6J3YXk/TnEsy6vJiZI/AAAAAAAAAa0/S8go6tW4weA/s1600/connection.png)

![medium](http://4.bp.blogspot.com/-FlX8ZwRcjTU/TnEsVSwh0yI/AAAAAAAAAas/VxjMj6QeEHU/s1600/visualVM2.png)

![medium](http://3.bp.blogspot.com/-WYs5-5s-Aik/TnEsZ24uHdI/AAAAAAAAAaw/3bTMjhjYLdg/s1600/visualVM3.png)

A titre informatif, ci-joint les fichiers de configuration de Jetty permettant la création d'un serveur RMI, du MBean Server et du connecteur RMI ainsi que de sa configuration.

Pour Jetty 6 :
```xml
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Mort Bay Consulting//DTD Configure//EN" "http://jetty.mortbay.org/configure.dtd">
 
<Configure id="Server" class="org.mortbay.jetty.Server">
    <Call id="MBeanServer" class="java.lang.management.ManagementFactory" name="getPlatformMBeanServer"/>
 
    <Get id="Container" name="container">
      <Call name="addEventListener">
        <Arg>
          <New class="org.mortbay.management.MBeanContainer">
            <Arg><Ref id="MBeanServer"/></Arg>
            <Call name="start" />
          </New>
        </Arg>
      </Call>
    </Get>
 
    <Call id="rmiRegistry" class="java.rmi.registry.LocateRegistry" name="createRegistry">
      <Arg type="int">1099</Arg>
    </Call>
     
    <Call id="jmxConnectorServer" class="javax.management.remote.JMXConnectorServerFactory" name="newJMXConnectorServer">
      <Arg>
        <New  class="javax.management.remote.JMXServiceURL">
          <Arg>service:jmx:rmi://localhost:1099/jndi/rmi://localhost:1099/jmxrmi</Arg>
        </New>
      </Arg>
      <Arg/>
      <Arg><Ref id="MBeanServer"/></Arg>
      <Call name="start"/>
    </Call>
</Configure>
```

Pour Jetty 7 :
```xml
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure.dtd">
<Configure id="Server" class="org.eclipse.jetty.server.Server">
 
  <Call id="MBeanServer" class="java.lang.management.ManagementFactory"
    name="getPlatformMBeanServer" />
 
  <New id="MBeanContainer" class="org.eclipse.jetty.jmx.MBeanContainer">
    <Arg>
      <Ref id="MBeanServer" />
    </Arg>
  </New>
 
  <Get id="Container" name="container">
    <Call name="addEventListener">
      <Arg>
        <Ref id="MBeanContainer" />
      </Arg>
    </Call>
  </Get>
 
  <Call name="addBean">
    <Arg>
      <Ref id="MBeanContainer" />
    </Arg>
  </Call>
 
  <Get id="Logger" class="org.eclipse.jetty.util.log.Log" name="log" />
  <Ref id="MBeanContainer">
    <Call name="addBean">
      <Arg>
        <Ref id="Logger" />
      </Arg>
    </Call>
  </Ref>
   
  <Call name="createRegistry" class="java.rmi.registry.LocateRegistry">
    <Arg type="java.lang.Integer">1099</Arg>
    <Call name="sleep" class="java.lang.Thread">
       <Arg type="java.lang.Integer">1000</Arg>
    </Call>
  </Call>
  
 <New id="ConnectorServer" class="org.eclipse.jetty.jmx.ConnectorServer">
    <Arg>
      <New class="javax.management.remote.JMXServiceURL">
        <Arg type="java.lang.String">rmi</Arg>
        <Arg type="java.lang.String" />
        <Arg type="java.lang.Integer">0</Arg>
        <Arg type="java.lang.String">/jndi/rmi://localhost:1099/jmxrmi</Arg>
      </New>
    </Arg>
    <Arg>org.eclipse.jetty:name=rmiconnectorserver</Arg>
    <Call name="start" />
  </New>
</Configure>
```

#Conclusion

Cet article avait pour objectif de montrer comment il pouvait être facile d'activer la couche JMX pour un serveur Jetty embedded dans Maven.

C'est vrai que démarrer un Jetty embarqué n'a pas pour objectif de faire des tests d'intégration mais cela peut toujours être utile dans le cas d'un POC ou d'une démonstration (qui était d'ailleurs l'objectif de ce petit cas d'usage. ;-) ).
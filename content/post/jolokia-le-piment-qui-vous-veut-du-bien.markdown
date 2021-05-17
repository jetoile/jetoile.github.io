---
title: "Jolokia : le piment qui vous veut du bien "
date: 2014-03-04 00:30:40 +0100
comments: true
tags: 
- java
- jmx
- jolokia
---
![left-small](http://3.bp.blogspot.com/-6Xm2grUWMtI/UxTNAazymaI/AAAAAAAABPE/dXJ7fti_FLA/s1600/jolokia.png)

 Dans des articles précédents, je m'étais déjà exprimé sur le fait que je trouvais qu'il était important de monitorer son application (qu'il s'agisse d'une application web, d'un batch ou d'une application standalone) (cf. [ici](/2010/05/jmx-ou-comment-administrer-et.html)). J'avais même creusé un peu la spécification JMX (cf. [là](/2010/10/jmx-pour-les-nuls-introduction.html)).

Pour faire suite à ce besoin, je vais, dans cet article, faire un focus sur un outils que j'ai découvert récemment (merci Romain ;-) ) mais qui existe depuis un moment (la version 1.0.0 est apparue en octobre 2011 sur le repo Maven central et le premier commit apparaissant sur Github date de Juillet 2010) : cet outils est __Jolokia__.

Comme à mon habitude, pour présenter cet outils, je m'appuierai sur la document officielle dans sa version courante, à savoir la 1.2.0.

Cependant, je ne ferai pas un plagiat exhaustif de la documentation qui est très complète (et surtout, je n'ai pas envie de me traduire les 92 pages de cette dernière... ;-) ) mais j'essaierai de faire un focus sur les points que je trouve les plus intéressants (à savoir les principes ainsi que le mode agent JVM (cf. plus tard) ).

<!-- more -->

# Principes et concepts

## Jolokia... pour quoi faire?

Dans le monde Java, il existe un standard pour faire de l'administration/supervision : il s'agit de JMX (Java Management eXtension). Il est inclus depuis le JDK 1.5. 

Cependant, JMX est malheureusement un des parents pauvres de Java : souvent méconnu ou mal utilisé, il est aussi très orienté vers le monde Java (et cela, même s'il existe la [JSR 160 - Java Management eXtension Remote API](https://jcp.org/en/jsr/detail?id=160)).

C'est la raison d'être de Jolokia qui offre une approche agent (qui cohabite avec la JSR 160) tout en offrant une interopérabilité via HTTP au moyen de JSON pour la partie _payload_. Cela lui permet d'exposer les couches d'administration/supervision JMX des applicatifs Java via une protocole interopérable de tous.

## Architecture

L'architecture de Jolokia diffère de celle de la JSR 160. En effet, la JSR 160 permet à un client d'invoquer de manière transparente un MBean qu'il soit dans un __MBeanServer__ local ou distant.

Cependant, même si cela est intéressant, c'est également une approche dangereuse puisque cela masque la partie transport qui peut entraîner un _overhead_ mais cela expose aussi le modèle des objets qui transitent. 

En effet, il existe une adhérence implicite au protocole RMI (qui est d'ailleurs le protocole par défaut des connecteurs JMX) pour la partie mécanisme de sérialisation des objets. C'est ce dernier point qui pose un problème d'interopérabilité avec tout programme extérieur à une JVM.

Ainsi, Jolokia, en offrant une approche différente via HTTP/JSon, permet de réconcilier ces différents mondes. Pour ce faire, il propose 2 modes :

* un mode agent,
* un mode proxy.

### Le mode agent

Dans ce mode, Jolokia se présente comme un agent qui expose un protocole au format JSON via HTTP et qui permet de servir de bridge vers les MBeans JMX locaux. Cela se passe donc en dehors du scope de la JSR 160.

Ainsi, il est possible d'exporter ce protocole de différentes manières dont la plus courante est via un conteneur de Servlet (qu'il soit légé ou pas).

Cependant, il existe d'autres possibilités comme des agents spécialisés qui peuvent utiliser un service HTTP OSGI ou qui peuvent embarquer un serveur Jetty.

Il est à noter que l'agent utilise le serveur HTTP embarqué dans toutes les JVM 6 d'Oracle et qu'il peut donc s'attacher dynamiquement à toutes les processus Java.

![medium](http://2.bp.blogspot.com/-TnigS1cXxaI/UxTNwI9jtzI/AAAAAAAABPM/q0rHjKSMmq4/s1600/jolokia-agent.png)

### Le mode proxy

Ce mode peut être utilisé lorsqu'il n'est pas possible de déployer un agent Jolokia sur la plateforme cible. Pour ce mode, le seul prérequis est l'accès au serveur cible via une connexion à travers de la JSR 160. Cela peut être le cas si l'application ne peut pas être modifiée ou si l'application expose déjà ses MBeans via la JSR 160.

Ce mode nécessite un conteneur de Servlet dans lequel sera déployée l'application web `jolokia.war` qui a, à sa charge, de "_proxyfier_" et qui, par défaut, supporte à la fois le mode agent et le mode proxy.

Ainsi, dans ce mode, un client enverra une requête Jolokia avec une section supplémentaire spécifiant la cible qui doit être atteinte. Toutes les information de routage est donc contenu dans la requête elle-même de manière à ce que le proxy puisse agir sans configuration spécifique. 

![medium](http://2.bp.blogspot.com/-wf_ReDc9AKk/UxTN2msI-BI/AAAAAAAABPU/neUmwiePEs0/s1600/jolokia-proxy.png)

## Les agents

Jolokia propose une approche orientée agent qui doit être, soit déployé sur la cible (__mode agent__), soit sur un serveur proxy (__mode proxy__).

Pour ces deux modes, il existe 4 types d'agent :

* __WAR agent__ : cet agent est packagé sous forme de WAR.
* __OSGI agent__ : cet agent au format OSGI (bundle) vient sous 2 formes : un agent minimal qui dispose d'une dépendance sur un OSGI _HTTService_ qui doit être démarré et un agent "tout en un" qui embarque une implémentation de _HTTPService_.
* __Mule agent__ : cet agent s'intègre à Mule et fourni une API d'adminitration/supervision dans lequel un agent jolokia dédié est intégré. Il inclut un serveur Jetty embarqué.
* __JVM agent__ : Depuis la version 1.6 du JDK d'Oracle, la JVM embarque un serveur HTTP légé. En utilisant ce dernier, Jolokia expose ses fonctionnalité. Cependant, cet agent peut être un peu lent en raison du fait que le serveur HTTP embarqué dans la JVM ne soit pas optimisé pour les performances.

Les deux types d'agent qui seront un peu plus détaillés dans cet article sont les types WAR agent et JVM agent.

## WAR agent

Le type WAR agent se présente comme une application web standard (au format WAR). La configuration se fait alors via l'élément `init-param` du web.xml.

Un autre moyen consiste à utiliser le context (dans le cas de Tomcat) qui permet de déporter la configuration en dehors du WAR. Ainsi, par exemple avec Tomcat, avec le context de l'application web se trouvant dans `$TOMCAT_HOME/conf/Catalina/localhost`, on peut avoir par exemple : 

```xml
<Context>
<Parameter name="maxDepth" value="1">
</Context>
```

Au niveau paramétrage, je vous laisse aller voir la document officielle ;-).

Du point de vue sécurité, il est possible de bénéficier de la sécurité du conteneur de Servlet via le `web.xml`.

Une autre manière de faire consite à intégrer Jolokia comme Servlet dans son application web. Pour ce faire, il suffit de tirer la bonne dépendance et de préciser le Servlet de Jolokia de manière classique via les éléments `servlet` et `servlet-mapping` du `web.xml` de l'application. 

```xml
<dependency>
  <groupId>org.jolokia</groupId>
  <artifactId>jolokia-core</artifactId>
  <version>${jolokia.version}</version>
</dependency>
```

Comme il est possible de déployer de multiples WAR agents Jolokia sur une même JVM, puisque des MBeans spécifiques Jolokia sont déployé dans le __PlatformMBeansServer__, il faut préciser la valeur de l'élément mbeanQualifier dans les paramètres d'init.

## JVM agent

Ce type d'agent Jolokia que j'ai testé IRL est très simple à utiliser et permet d'attacher un agent à une JVM afin qu'elle expose à la mode REST ses couches d'administration et de supervision.

Il est possible de :

* _bootstrapper_ l'agent JVM au démarrage de la JVM en lui fournissant le jar adéquate ainsi que les options qui vont bien :

```bash
java -javaagent:agent.jar=port=7777,host=localhost
```

* _bootstrapper_ l'agent JVM en lui fournissant directement une fichier de configuration :

```bash
java -javaagent:agent.jar=config=$FICHIER_CONFIG_JOLOKIA
```

où un exemple de fichier de configuration peut être trouvé dans jar de l'agent (au nom de `default-jolokia-agent.properties`).

où `agent.jar` peut être téléchargé de http://www.jolokia.org/download.html (artifact : __JVM-Agent__).

De la même manière, un agent Jolokia peut être attaché à une processus Java à la demande (un peu comme lorsque JConsole se connecte à un processus local). Pour ce faire, il suffit de lancer la commande suivante : 

```bash
java -jar agent.jar start <PID>
```

## Jolokia et la sécurité

Jolokia permet de configurer la sécurité assez finement. N'ayant pas tester, je ne m'attarderai pas trop sur le sujet. 

Cependant, il est intéressant de noter que Jolokia permet de filtrer l'accès des clients par IP mais offre également un accès plus fin pour l'accès aux MBeans (`read`/`write`/`exec`/`list`/`search`/`version`).

Point également important, Jolokia supporte la spécification W3C pour le Cross-Origin Resource Sharing (CORS) qui permet d'utiliser des outils comme [Hawt.io](http://hawt.io/) (mais nous y reviendront plus tard...).

## Le protocole Jolokia

Jolokia utilise un protocole JSON sur HTTP. La communication est basé sur le paradigme requête/réponse où chaque requête fournit une réponse.

Les requêtes peuvent être envoyées de deux manières :

* soit avec une requête HTTP GET (dans ce cas, les paramètres sont encodés dans l'URL),
* soit avec une requête HTTP POST où la requête est incluse dans le corps de la requête au format JSON.

Les réponses retournées par l'agent sont, quant à elles, toujours envoyées en JSON.

De plus, les requêtes au format HTTP GET peuvent prendre deux formes :

* utiliser un format REST (ex : http://localhost:8080/jolokia/read/java.lang:type=Memory/HeapMemoryUsage)
* utiliser un format où la requête est donnée par un paramètre __p=__ (ex : http://localhost:8080/jolokia?p=/read/jboss.jmx:alias=jmx%2Frmi%2FRMIAdaptor/State)

A noter que Jolokia utilise le caractère "!" comme caractère d'échappement.

Plutôt que de longs discours, quelques exemples issus de la documentation aideront à comprendre...

Pour lire la valeur du MBean dont l'objectName est __java.lang:type=Memory/HeapMemoryUsage__, les requêtes suivantes sont équivalentes : 

```bash
curl -XGET "http://127.0.0.1:8778/jolokia/read/java.lang:type=Memory/HeapMemoryUsage/used"
```
<br/>

```bash
curl -XGET "http://127.0.0.1:8778/jolokia/?p=/read/java.lang:type=Memory/HeapMemoryUsage/used"
```
<br/>

```bash
curl -XPOST -d '{"type" : "read", "mbean" : "java.lang:type=Memory", "attribute" : "HeapMemoryUsage", "path" : "used" }' http://127.0.0.1:8778/jolokia/
```
soit en format lisible : 

```javascript
{
    "type": "read",
    "mbean": "java.lang:type=Memory",
    "attribute": "HeapMemoryUsage",
    "path": "used"
}
```
La réponse obtenue est de la forme : 

```javascript
{
    "timestamp": 1393865787,
    "status": 200,
    "request": {
        "mbean": "java.lang:type=Memory",
        "path": "used",
        "attribute": "HeapMemoryUsage",
        "type": "read"
    },
    "value": 162037504
}
```
Pour montrer le coté structuré de la réponse, à une requête de type : 

```bash
http://127.0.0.1:8778/jolokia/read/java.lang:type=Memory
```
on obtient : 
```javascript
{
    "timestamp": 1393866039,
    "status": 200,
    "request": {
        "mbean": "java.lang:type=Memory",
        "type": "read"
    },
    "value": {
        "Verbose": false,
        "ObjectPendingFinalizationCount": 0,
        "NonHeapMemoryUsage": {
            "max": 329252864,
            "committed": 218951680,
            "init": 19136512,
            "used": 139774968
        },
        "HeapMemoryUsage": {
            "max": 518979584,
            "committed": 518979584,
            "init": 134217728,
            "used": 179096552
        },
        "ObjectName": {
            "objectName": "java.lang:type=Memory"
        }
    }
}
```

 Concernant les opérations possibles (dans l'exemple précédent, il s'agissait d'une lecture), il existe :

* __read__ avec le format GET suivant : 

`<base-url>/read/<mbean name>/<attribute name>/<inner path>`

(ex : http://localhost:8080/jolokia/read/java.lang:type=Memory/HeapMemoryUsage/used)

* __write__ avec le foramt GET suivant : 

`<base url>/write/<mbean name>/<attribute name>/<value>/<inner path>`

(ex : http://localhost:8080/jolokia/write/java.lang:type=Memory/Verbose/true)

* __exec__ avec le format GET suivant : 

`<base url>/exec/<mbean name>/<operation name>/<arg1>/<arg2>/....`

(ex : http://localhost:8080/jolokia/exec/java.lang:type=Memory/gc)

* __search__ avec le format GET suivant : 

`<base-url>/search/<pattern>` 

(ex : http://localhost:8080/jolokia/search/*:j2eeType=J2EEServer,*)

* __list__ avec le format GET suivant : 

`<base-url>/list/<inner path>` 

(ex : http://localhost:8080/jolokia/list/java.lang/type=Memory/attr)

* __version__ qui permet d'avoir la version du protocole utilisé ainsi qu'un ensemble de paramètre avec le format GET suivant : 

`<base-url>/version` 

(ex : http://localhost:8080/jolokia/version)


Enfin, si le mode proxy est utilisé, seul le mode POST peut être utilisé et doit, alors, avoir le format suivant : 

```javascript
{
    "type" : "read",
    "mbean" : "java.lang:type=Memory",
    "attribute" : "HeapMemoryUsage",
    "target" : { 
         "url" : "service:jmx:rmi:///jndi/rmi://targethost:9999/jmxrmi",
         "user" : "jolokia",
         "password" : "s!cr!t"
    } 
} 
```

Je ne rentrerai pas plus en détaille sur cette partie là qui est beaucoup plus exhaustive dans la documentation officielle (comment les objets sont sérialisés, le mapping complet des MXBeans, la découverte des agents ou les différentes versions du protocole Jolokia - actuellement la version 7.1 - ) et si cela est nécessaire, je conseille d'aller directement se référer à la documentation.

# Autres features

En plus des fonctionnalités présentées précédemment, Jolokia offre les fonctionnalités suivantes :

* exposition de son propre MBean,
* différents clients (javascript, plugin cubism, java, Jmx4Perl),
* une API de programmation pour exposer sont MBeanServer,
* un JSonMBean,
* une integration Spring,

Pour l'intégration avec Spring, cela se fait via l'import Maven suivant : 
```xml
<dependency>
  <groupId>org.jolokia</groupId>
  <artifactId>jolokia-spring</artifactId>
  <version>1.2.0</version>
</dependency>  
```

et le contexte Spring suivant : 

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:jolokia="http://www.jolokia.org/jolokia-spring/schema/config"
       xsi:schemaLocation="
       http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.jolokia.org/jolokia-spring/schema/config  http://www.jolokia.org/jolokia-spring/schema/config/jolokia-config.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
 
  <jolokia:agent lookupConfig="true" systemPropertiesMode="never">
        <jolokia:config
                autoStart="true"
                host="${jolokia.host}"
                port="${jolokia.port}"
                user="${jolokia.user}"
                password="${jolokia.password}"/>
    </jolokia:agent>
 
    <bean id="mbeanServer" class="org.springframework.jmx.support.MBeanServerFactoryBean">
        <property name="locateExistingServerIfPossible" value="true"/>
    </bean>
</beans>
```

# Conclusion
Dans cet article, une présentation succincte a été faite de Jolokia. J'espère qu'elle vous aura plu ;-)...

Il s'agit d'un outils simple, pluggable très facilement à n'importe quelle application (une intégration à Cassandra s'est fait en 5 minutes).

Cette présentation était surtout axé concepts et principes afin de bien comprendre ce que peut apporter cet outils.

Cependant, un des gros avantage de Jolokia est un point qui n'a pas été abordé dans cet article : il s'agit de son intégration à [Hawt.io](http://hawt.io/). Cet outils ayant déjà fait le sujet d'article sur le blog de Zenika, je vous invite à y jeter un oeil :

* [HawtIO, la console web polyvalente](http://blog.zenika.com/index.php?post/2014/01/07/HawtIO-la-console-web-polyvalente)
* [HawtIO, écrire un plugin](http://blog.zenika.com/index.php?post/2014/01/14/HawtIO-ecrire-un-plugin)


Ainsi, en production, disposer du combo Hawt.io + Jolokia offre, à mon sens, d'énormes avantages comme, par exemple, accèder aux informations de n'importe quelle application qui, généralement, n'est pas accessible pour des raisons de sécurité (cf. [ici](/2010/05/jmx-et-firewall.html)).

Bien sûr, il existe d'autres solutions comme l'utilisation de [CraSH](http://www.crashub.org/) mais exposer ses MBeans via JSON over HTTP est tellement simple et peut surtout être exploité simplement par les équipes de production ;-) .

# Pour aller plus loin...

* Site de Jolokia : http://www.jolokia.org/
* Site de documentation de Jolokia : http://www.jolokia.org/reference/html/index.html
* Site d'Hawt.io : http://hawt.io/

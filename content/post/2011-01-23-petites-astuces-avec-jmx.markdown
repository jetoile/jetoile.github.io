---
title: "Petites astuces avec JMX"
date: 2011-01-23 18:27:07 +0100
comments: true
tags: 
- java
- jmx
---

![left-small](http://1.bp.blogspot.com/_XLL8sJPQ97g/TTr8YAhjHqI/AAAAAAAAAUA/nx8ug8MDDNA/s320/jmx02.png)
Ce rapide article fournit un petit retour d'expérience de quelques galères que j'ai pu avoir avec JMX (_Java Management eXtension_), galères certes stupides mais qui pourront peut être servir à d'autres... ;-)

Seront donc abordés deux points :

* Erreur lors de l'arrêt de tomcat configuré avec JMX
* Impossibilité de se connecter à l'agent JMX

<!-- more -->

#Message d'erreur lors de l'arrêt d'Apache Tomcat?

##Cas d'usage
Les options permettant d'activer JMX sur la JVM ont été passées via la variable `JAVA_OPTS` ou autre (via le catalina.sh par exemple).

Le démarrage de Tomcat s'effectue sans problème mais lors de son arrêt, une erreur apparaît et tomcat refuse de s'arrêter c'est-à-dire que son processus tourne toujours en tâche de fond.

Exemple de variable JAVA_OPTS :
```bash
export JAVA_OPTS="$JAVA_OPTS \
-Dcom.sun.management.jmxremote \
-Dcom.sun.management.jmxremote.port=18080 \
-Dcom.sun.management.jmxremote.authenticate=false \
-Dcom.sun.management.jmxremote.ssl=false"
```
Message affiché sur la console lors du démarrage de Tomcat :
```bash
khanh@khanh-vaio:/opt/apache-tomcat-6.0.20/bin$ ./startup.sh
Using CATALINA_BASE:   /opt/apache-tomcat-6.0.20
Using CATALINA_TMPDIR:  /opt/apache-tomcat-6.0.20
Using CATALINA_HOME:  /opt/apache-tomcat-6.0.20/temp
Using JRE_HOME:     /usr/lib/jvm/java-6-sun/jre
```
Message affiché sur la console lors de l'arrêt de Tomcat :

```bash
khanh@khanh-vaio:/opt/apache-tomcat-6.0.20/bin$ ./shutdown.sh
Using CATALINA_BASE:   /opt/apache-tomcat-6.0.20
Using CATALINA_TMPDIR:  /opt/apache-tomcat-6.0.20
Using CATALINA_HOME:   /opt/apache-tomcat-6.0.20/temp
Using JRE_HOME:     /usr/lib/jvm/java-6-sun/jre
Erreur: Exception envoyée par l'agent : java.rmi.server.ExportException: Port already in use: 18080; nested exception is: 
        java.net.BindException: Address already in use
```

##Explication
En fait, cette erreur se produit lors de l'arrêt de Tomcat car le script `shutdown.sh` (ou `catalina.sh stop`) relance un processus java qui tente alors de démarrer un serveur RMI qui est utilisé par JMX. Bien sûr, ce port étant déjà utilisé par le processus java que l'on tente d'arrêter, cela échoue.

##Proposition pour résoudre le problème

* ~~Ne pas positionner les options pour activer JMX sur la variable système `JAVA_OPTS` au risque de ne pas pouvoir démarrer plusieurs processus Java.~~
* ~~Si c'est le script catalina.sh qui positionne la variable `JAVA_OPTS`, ne pas la positionner dans tous les cas mais seulement dans le cas où l'argument start est passé au script.~~
* ~~Positionner la variable `JAVA_OPTS` dans le script `startup.sh`.~~
* Utiliser la variable `CATALINA_OPTS` qui n'est utilisée que quand les options run et start sont passées"
>CATALINA_OPTS   (Optional) Java runtime options used when the "start",  or "run" command is executed.

#Problème de connexion?

##Cas d'usage

Une jconsole ou une jvisualvm qui n'arrive pas à se connecter à un programme ou une connexion au connecteur d'un agent JMX qui échoue avec une exception.

Chose d'autant plus étonnante qu'avec une connexion réseau, il n'y a pas de problèmes alors qu'en mode hors-ligne, impossible d'avoir accès à l'agent JMX.

Exemple d'exception lors de la tentative de connexion à l'agent JMX distant :
```bash
java.lang.NullPointerException
at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:281)
at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:227)
at com.blogspot.jetoile.jgroups.connector.ConnectorStubListener.update(ConnectorStubListener.java:47)
at com.blogspot.jetoile.jgroups.connector.ConnectorStubManager.put(ConnectorStubManager.java:46)
[...]
at com.blogspot.jetoile.jgroups.connector.TestJMXServerTest.main(TestJMXServerTest.java:34)

```

Fenêtre d'erreur de la jconsole :

![medium](http://3.bp.blogspot.com/_XLL8sJPQ97g/TTsrkOibDDI/AAAAAAAAAUE/qpX-1nw0OOA/s1600/error02.png)

Fenêtre d'erreur de jvisualvm : _"Data not available because JMX connection to the JMX agent could not be established."_ :

![medium](http://4.bp.blogspot.com/_XLL8sJPQ97g/TTsroeTxKoI/AAAAAAAAAUI/0aUghCdjv0w/s1600/error01.png)

##Analyse

Après de multiples tentatives pour comprendre, j'ai tenté un changement de jre pour voir si cela ne pouvait pas venir d'une anomalie (si j'avais un problème de connexion en mode hors-ligne, ce n'était pas pour rien puisque j'étais effectivement hors-ligne et, par conséquent, je devais me débrouiller par moi-même... ;-) ).

Ainsi, en passant d'une jre hotspot vers une openjdk, j'ai obtenu une stacktrace beaucoup plus complète puisqu'elle m'a clairement indiquée la source du problème :

```bash
ERROR c.b.j.j.c.ConnectorStubListener - ConnectorStubListener update: {}
java.rmi.ConnectIOException: Exception creating connection to: 192.168.xxx.xxx; nested exception is:
java.net.NoRouteToHostException: Network is unreachable
at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:632) ~[na:1.6.0_20]
at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:216) ~[na:1.6.0_20]
at sun.rmi.transport.tcp.TCPChannel.newConnection(TCPChannel.java:202) ~[na:1.6.0_20]
at sun.rmi.server.UnicastRef.invoke(UnicastRef.java:128) ~[na:1.6.0_20]
at javax.management.remote.rmi.RMIServerImpl_Stub.newClient(Unknown Source) ~[na:1.6.0_20]
at javax.management.remote.rmi.RMIConnector.getConnection(RMIConnector.java:2343) ~[na:1.6.0_20]
at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:296) ~[na:1.6.0_20]
at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:244) ~[na:1.6.0_20]
at com.blogspot.jetoile.jgroups.connector.ConnectorStubListener.update(ConnectorStubListener.java:47) ~[classes/:na]
at com.blogspot.jetoile.jgroups.connector.ConnectorStubManager.put(ConnectorStubManager.java:46) [classes/:na]
...
at com.blogspot.jetoile.jgroups.connector.JMXServer.start(JMXServer.java:152) [classes/:na]
at com.blogspot.jetoile.jgroups.connector.TestJMXServerTest2.main(TestJMXServerTest2.java:34) [test-classes/:na]
 
 
Caused by: java.net.NoRouteToHostException: Network is unreachable
at java.net.PlainSocketImpl.socketConnect(Native Method) ~[na:1.6.0_20]
at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:310) ~[na:1.6.0_20]
at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:176) ~[na:1.6.0_20]
at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:163) ~[na:1.6.0_20]
at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:384) ~[na:1.6.0_20]
at java.net.Socket.connect(Socket.java:546) ~[na:1.6.0_20]
at java.net.Socket.connect(Socket.java:495) ~[na:1.6.0_20]
at java.net.Socket.<init>(Socket.java:392) ~[na:1.6.0_20]</init>
at java.net.Socket.<init>(Socket.java:206) ~[na:1.6.0_20]</init>
at sun.rmi.transport.proxy.RMIDirectSocketFactory.createSocket(RMIDirectSocketFactory.java:40) ~[na:1.6.0_20]
at sun.rmi.transport.proxy.RMIMasterSocketFactory.createSocket(RMIMasterSocketFactory.java:146) ~[na:1.6.0_20]
at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:613) ~[na:1.6.0_20]
... 34 common frames omitted
```

##Explication

En fait, mon problème venait du fait que mon fichier hosts était incorrect (je vous avais prévenu que c'était une erreur stupide : n'importe qui aurait pensé à vérifier sa configuration réseau... ! ;-) ). 

En effet, la couche logicielle gérant mes connexions réseaux sur mon ordinateur a tendance à ajouter dans mon fichier /etc/hosts mon adresse réseau. Cependant, lorsque je coupe mon wifi, mon ordinateur passe en mode hors-ligne sans modifier mon fichier hosts, ce qui corrompt ma configuration réseau.

#Proposition pour résoudre le problème

* Corriger son fichier hosts et vérifier sa configuration réseau

A noter que dans mon cas, le fait d'avoir changer de jre m'a permis d'obtenir une stacktrace beaucoup plus complète et précise. Aussi, dans un moment de désespoir et en dernier recours, il peut être intéressant de changer de jre ;-).
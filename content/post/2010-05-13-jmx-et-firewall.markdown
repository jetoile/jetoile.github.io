---
title: "JMX et Firewall"
date: 2010-05-13 11:16:06 +0100
comments: true
tags: 
- java
- jmx
---
![left-small](http://1.bp.blogspot.com/_XLL8sJPQ97g/S-HXgq4HI7I/AAAAAAAAAJ4/LzJKs3wTNZM/s200/jmx4.png)
Cet article fait suite à mon [post précédent](/2010-05-05-jmx-ou-comment-administrer-et.html) qui, je le rappelle, pour ceux qui auraient eux le courage de le lire jusqu'à la fin ;-), avait pour objectif de rappeler en quoi JMX (_Java Management eXtension_) pouvait être une bonne réponse aux problématiques de supervision et d'administration dans une application (au sens large du terme). Cet article portera sur le sujet que je voulais initialement traiter, à savoir, comment accéder à un serveur JMX se trouvant derrière un Firewall. Cette problématique est indiquée sur le site de Sun/Oracle mais je vais, ici, faire un résumer de la méthode à suivre.

<!-- more -->

En effet, supposions que nous ayons l'architecture suivante :
![center](http://4.bp.blogspot.com/_XLL8sJPQ97g/S-vg1m-cF6I/AAAAAAAAAKI/glTWe7EakaA/s1600/Firewall2.png "licence : Creative Commons Attribution-Share Alike")

En fait, dans ce cas, le client sur lequel la jConsole, jVisualVM ou autre est lancée, ne verra pas le serveur JMX, et cela, même en précisant le port.
En effet, le connecteur JMX RMI ouvre deux ports :

* un pour le RMI Registry (et qui est configurable avec l'option `com.sun.management.jmxremote.port`)
* et un autre utilisé pour exporter les objets utilisés par la connexion JMX RMI.

Cependant, ce dernier port est, par défaut, alloué de manière dynamique et aléatoirement (puisqu'il n'est pas nécessaire de connaitre ce port pour se connecter à l'agent JMX).

C'est cela qui pose problème lorsque l'on tente de se connecter à un serveur JMX se trouvant derrière un firewall puisqu'il n'est, alors, pas possible de demander l'ouverture d'un port que l'on ne connait pas...

Aussi, la seule possibilité pour résoudre ce problème est de passer par la déclaration de son JMXServiceURL pour spécifier le port permettant d'exporter les objets utilisés par le connexion JMX RMI. Cependant, ce JMXServiceURL ne peut pas être modifié dans l'agent JMX utilisé par défaut...

Heureusement, il est possible de définir programmatiquement son propre agent JMX.

En fonction de l'usage, deux choix sont possibles :

* définir son agent JMX au sein de l'application
* utiliser un agent JVM pour définir son propre agent JMX

Il est à noter que l'implémentation présentée ici ne prend pas en compte la problématique de sécurité (et en particulier SSL) où deux ports différents doivent être utilisés. En outre, il sera supposé que le RMI Registry sera démarré dans l'application.

# 1ère méthode
Ainsi, si vous avez la main sur la méthode `main` sur l'application, il suffit de rajouter le code qui va bien, à savoir :
```java
package example;
 
import java.lang.management.ManagementFactory;
import java.rmi.registry.LocateRegistry;
import java.util.HashMap;
 
import javax.management.MBeanServer;
import javax.management.remote.JMXConnectorServer;
import javax.management.remote.JMXConnectorServerFactory;
import javax.management.remote.JMXServiceURL;
 
public class MyApp {
 
 public static void main(String[] args) throws Exception {
  final int port1 = Integer.parseInt(System.getProperty("example.rmi.agent.port", "3000"));
  System.out.println("Create RMI registry on port "+port1);
  LocateRegistry.createRegistry(port1);
 
  // Retrieve the PlatformMBeanServer.
  System.out.println("Get the platform's MBean server");
  MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
 
  // Environment map.
  System.out.println("Initialize the environment map");
  HashMap env = new HashMap();
 
  // Create an RMI connector server.
  //
  // As specified in the JMXServiceURL the RMIServer stub will
  // be registered in the RMI registry running in the local
  // host on port 3000 with the name "jmxrmi". This is the same
  // name the out-of-the-box management agent uses to register 
  // the RMIServer stub too.
  System.out.println("Create an RMI connector server");
  JMXServiceURL url = new JMXServiceURL("service:jmx:rmi://localhost:" + port1 + "/jndi/rmi://localhost:" + port1 + "/jmxrmi");
  JMXConnectorServer cs = JMXConnectorServerFactory.newJMXConnectorServer(url, env, mbs);
 
  // Start the RMI connector server.
  System.out.println("Start the RMI connector server");
  cs.start();
}
```
Pour tester ce code, exécuter le code suivant :

```bash
java -Dexample.rmi.agent.port=3010 -classpath . example.MyApp
```
puis lancer une jconsole et renseigner le champ `Remote Process` :

```text
service:jmx:rmi://localhost:3010/jndi/rmi://localhost:3010/jmxrmi
```

# 2ième méthode
Par contre, la solution précédente n'est parfois pas applicable car le code de l'application à exécuter peut ne pas être disponible ou car récupérer ses sources, les modifiés et recompiler la dite application (par exemple, si c'est un conteneur de Servlet que vous voulez administrer...) peut être un peu problématique (je ne suis pas sûr que l'équipe de production soit enchantée d'utiliser une version modifiée d'Apache Tomcat...).

Aussi, dans ce cas, il est possible de déclarer son propre agent à enregistrer dans la JVM (en définissant la méthode `premain` et en renseignant l'attribut `Premain-Class` du fichier `MANIFEST.MF`) qui se chargera de définir et de lancer l'agent JMX (une sorte de wrapper). Cependant, dans ce cas, l'agent JMX lancé par l'agent java ne sera pas arrêté par l'envoie du signal de fin de l'application. Il convient alors de lancer un Thread chargé de scrupter le processus de l'agentJMX.

Ci-dessous un exemple de l'agent JVM :

```java
/*
* CustomAgent.java
*
* Copyright 2007 Sun Microsystems, Inc.  All Rights Reserved.
*
* Redistribution and use in source and binary forms, with or
* without modification, are permitted provided that the
* following conditions are met:
*
* - Redistributions of source code must retain the above 
*   copyright notice, this list of conditions and the 
*   following disclaimer.
*
* - Redistributions in binary form must reproduce the above 
*   copyright notice, this list of conditions and the following
*   disclaimer in the documentation and/or other materials
*   provided with the distribution.
*
* - Neither the name of Sun Microsystems nor the names of its
*   contributors may be used to endorse or promote products 
*   derived from this software without specific prior written
*   permission.
*
* THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND 
* CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, 
* INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF 
* MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE 
* DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
* CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
* SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, 
* BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR 
* SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS 
* INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
* LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
* (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT
* OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE 
* POSSIBILITY OF SUCH DAMAGE.
*
* Created on Jul 25, 2007, 11:42:49 AM
*
*/
 
package example.rmi.agent;
 
import java.io.IOException;
import java.lang.management.ManagementFactory;
import java.net.InetAddress;
import java.rmi.registry.LocateRegistry;
import java.util.HashMap;
import javax.management.MBeanServer;
import javax.management.remote.*;
import javax.management.remote.rmi.*;
 
/**
* This CustomAgent will start an RMI COnnector Server using 
* only port "example.rmi.agent.port".
*
* @author dfuchs
*/
public class CustomAgent {
 public static class CleanThread extends Thread {
  private final JMXConnectorServer cs;
  public CleanThread(JMXConnectorServer cs) {
   super("JMX Agent Cleaner");
   this.cs = cs;
   setDaemon(true);
  }
  public void run() {
   boolean loop = true;
   try {
    while (loop) {
     final Thread[] all = new Thread[Thread.activeCount()+100];
     final int count = Thread.enumerate(all);
     loop = false;
     for (int i=0; i < count; i++) {
      final Thread t = all[i];
      // daemon: skip it.
      if (t.isDaemon()) continue;
      // RMI Reaper: skip it.
      if (t.getName().startsWith("RMI Reaper")) continue;
      if (t.getName().startsWith("DestroyJavaVM")) continue;
      // Non daemon, non RMI Reaper: join it, break the for
      // loop, continue in the while loop (loop=true)
      loop = true;
      try {
       System.out.println("Waiting on "+t.getName()+
         " [id="+t.getId()+"]");
       t.join();
      } catch (Exception ex) {
       ex.printStackTrace();
      }
      break;
     }
    }
    // We went through a whole for-loop without finding any 
    // thread to join. We can close cs.
   } catch (Exception ex) {
    ex.printStackTrace();
   } finally {
    try {
     // if we reach here it means the only non-daemon threads
     // that remain are reaper threads - or that we got an
     // unexpected exception/error.
     cs.stop();
    } catch (Exception ex) {
     ex.printStackTrace();
    }
   }
  }
 }
 
 
 private CustomAgent() { }
 
 public static void premain(String agentArgs) 
 throws IOException {
  // Start an RMI registry on port specified by 
  // example.rmi.agent.port (default 3000).
  final int port= Integer.parseInt(System.getProperty("example.rmi.agent.port","3000"));
  System.out.println("Create RMI registry on port "+port);
  LocateRegistry.createRegistry(port);
 
  // Retrieve the PlatformMBeanServer.
  System.out.println("Get the platform's MBean server");
  MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
 
  // Environment map.
  System.out.println("Initialize the environment map");
  HashMap env = new HashMap();
 
  // Create an RMI connector server.
  //
  // As specified in the JMXServiceURL the RMIServer stub will 
  // be registered in the RMI registry running in the local 
  // host on port 3000 with the name "jmxrmi". This is the 
  // same name the out-of-the-box management agent uses to 
  // register the RMIServer stub too.
  //
  // The port specified in "service:jmx:rmi://"+hostname+":"+port
  // is the second port, where RMI connection objects will be 
  // exported. Here we use the same port as that we choose   
  // for the RMI registry. The port for the RMI registry is  
  // specifiedin the second part of the URL, in 
  // "rmi://"+hostname+":"+port
  System.out.println("Create an RMI connector server");
  final String hostname = InetAddress.getLocalHost().getHostName();
  JMXServiceURL url =
   new JMXServiceURL("service:jmx:rmi://"+hostname+":"+(port)+"/jndi/rmi://"+hostname+":"+port+"/jmxrmi");
 
  // Now create the server from the JMXServiceURL
  JMXConnectorServer cs = JMXConnectorServerFactory.newJMXConnectorServer(url, env, mbs);
 
  // Start the RMI connector server.
  System.out.println("Start the RMI connector server on port "+port);
  cs.start();
  System.out.println("Server started at: "+cs.getAddress());
 
  // Start the CleanThread.
  final Thread clean = new CleanThread(cs);
  clean.start();
 }
}
```

Pour créer l'agent, il est possible d'utiliser le script _ant_ suivant qui renseigne de manière adéquate le fichier `MANIFEST` :
```xml
<project basedir="." default="build-agent-jar" name="customAgent">
 
  <property location="src" name="src"></property>
  <property location="build" name="build"></property>
  <property location="dist" name="dist"></property>
  <property name="dist.agent.jar" value="customAgent.jar"></property>
 
  <target name="init">
    <mkdir dir="${build}"></mkdir>
    <mkdir dir="${dist}"></mkdir>
  </target>
 
  <target description="clean" name="clean">
    <delete dir="${build}"></delete>
    <delete dir="${dist}"></delete>
  </target>
 
  <target depends="init" name="compile">
    <javac destdir="${build}" srcdir="${src}"></javac>
  </target>
       
  <target depends="clean,compile" name="build-agent-jar">
    <jar basedir="${build}" destfile="${dist}/${dist.agent.jar}">
      <manifest>
        <attribute name="Premain-Class" value="example.rmi.agent.CustomAgent"></attribute>
      </manifest>
    </jar>
  </target>
</project>
```
Pour tester ce code, ajouter le jar `customAgent.jar` au classpath de l'application et exécuter la commande suivante :

```bash
java -Dexample.rmi.agent.port=3010 -javaagent:customAgent.jar -classpath customAgent.jar example.MyApp2

```
Le dernier point à prendre en compte est le fait que pour rendre visible le serveur RMI, il peut être nécessaire de préciser le nom du serveur avec l'option `java.rmi.server.hostname`. Aussi, si le serveur RMI utilisé est embarqué dans votre conteneur de servlet ou autre, il peut s'avérer utile de le préciser lors du lancement de la JVM.

Par exemple, dans le cas d'Apache Tomcat, il est possible d'ajouter l'option Java suivante :

```bash
export JAVA_OPTS="$JAVA_OPTS -javaagent:${CATALINA_HOME}/lib/customAgent.jar -Djava.rmi.server.hostname=192.168.1.105"
```

Enfin, il est à noter que je ne suis pas expert JMX (je m'excuse donc pour les quelques petites approximations qu'il pourrait y avoir)...

Pour plus d'informations, je conseillerai, en vrac, les liens suivants : 

* Blogs de Daniel Fuchs : http://blogs.sun.com/jmxetc/entry/troubleshooting_connection_problems_in_jconsole
* Blogs de Daniel Fuchs : http://blogs.sun.com/jmxetc/entry/connecting_through_firewall_using_jmx
* Page d'Oracle de documentation sur JMX : http://java.sun.com/javase/6/docs/technotes/guides/management/agent.html
* Blog de Oleg Zhurakousky : http://olegz.wordpress.com/2009/03/23/jmx-connectivity-through-the-firewall/
* Documentation sur MX4J : http://mx4j.sourceforge.net/docs/ch03.html#N10324

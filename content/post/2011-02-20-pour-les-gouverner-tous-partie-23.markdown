---
title: "Pour les gouverner tous - partie 2/3"
date: 2011-02-20 18:41:34 +0100
comments: true
tags: 
- java
- jgroups
---

![left-small](http://3.bp.blogspot.com/_XLL8sJPQ97g/TUcoTratqiI/AAAAAAAAAUQ/Gmc1h0rvA2w/s1600/jmx_jgroups01.png)


Cet article fait suite à mon [article précédent](/2011/01/pour-les-gouverner-tous-partie-13.html) et a pour objectif de présenter un petit POC (_Proof Of Concept_) simplicime mettant en oeuvre JGroups en version 2.11.0.GA (la dernière version stable à ce jour). Le principe est de montrer comment il est possible d'utiliser JGroups pour permettre à plusieurs instances d'une même application de se partager les valeurs d'une donnée. Enfin, pour être plus précis, cet objet partagé ne le sera pas vraiment (ndlr : partagé) par toutes les instances mais il s'agira plutôt de permettre à chaque nouvelle instance de récupérer la valeur d'une donnée auprès des autres instance déjà présentes dans le système. En outre, les autres instances déjà présentes devront recevoir directement la valeur de la donnée de la nouvelle instance.

L'autre raison d'être de cet article permettra d'introduire la couche protocolaire utilisée par mon petit POC qui permet de rendre distribuable un agent JMX dans une architecture distribuée.

<!-- more -->

#Spécification des besoins/pré-requis

Ce projet s'appuiera sur les pré-requis suivant :

* [slf4j](http://www.slf4j.org/)/[logback](http://logback.qos.ch/) pour la partie log
* [maven 3](http://maven.apache.org/index.html) (ou 2 au choix) pour la partie build
* et... [JGroups](http://www.jgroups.org/) dans sa version 2.11.0.GA ;-)

Coté tests unitaires, je m'excuse préalablement auprès de vous, mais il n'y en aura pas... et cela pour deux raisons que je vous laisse choisir :

* je suis flemmard ;-) mais surtout, je n'ai pas envie de mocker la terre entière. 
* ce petit programme est plus une utilisation basique de JGroups et, le produit fonctionnant bien, je ne vois pas l'intérêt de le retester. En outre, ici, il est plus intéressant de tester de manière intégrée que de manière unitaire en raison de l'aspect distribué de l'application.

Comme je l'ai mentionné précédemment dans l'introduction, le but de cet article est simple : une application dispose de plusieurs instances qui se trouvent sur des JVM distinctes (il peut donc s'agir d'instances exécutées sur une ou plusieurs machines).

* Si une nouvelle instance est démarrée, elle doit pouvoir demander à toutes les autres instances de l'application la valeur d'une donnée X afin, par exemple, de connaître leur état. 
* De plus, toutes les instances déjà existantes dans le système doivent pouvoir être notifiées de l'arrivée d'une nouvelle instance et, par la même occasion, recevoir sa valeur courante de la donnée X. 
* Enfin, si une instance disparait du sytème (arrêt, ...), toutes les instances doivent automatiquement supprimer en leur sein la valeur de la donnée de l'instance incriminée.

Coté configuration de JGroups, cet article s'appuiera sur une configuration par défaut, c'est à dire une configuration en UDP.

#Architecture de l'application

Comme vous pouvez vous en douter, l'architecture de l'application sera simple puisque JGroups fournit nativement de nombreuses possibilités. Aussi, je ne présenterai pas de super conception.

Par contre, si les termes __ReceiverAdapter__, __MembershipListener__ ou __View__ ne vous parlent pas, je vous renverrai :

* soit, à mon [article](http://jetoile.blogspot.com/2010/12/jgroups-tour-d.html) sur JGroups ;-)
* soit (mieux), à la [documentation](http://www.jgroups.org/ug.html) officielle de ce dernier.

Notre application sera composée de trois parties :

* La partie donnée : la classe `Data` représentera la donnée à faire transiter. Il s'agira d'un simple POJO qui sera, bien sûr, sérialisable.
* La partie notification de changement de l'infrastructure (ie. arrivé ou arrêt d'une instance dans le système) : la classe `ChangeInfraListener` qui implémentera l'interface `MembershipListener` et donc la méthode `viewAccepted()` _call-backé_ par JGroups pour notifier d'un changement au niveau d'une des ses vues.
* La partie qui aura à sa charge l'exposition de la donnée représentée par la classe `Data` et qui aura initialisera l'application : la classe `JGroupsClient` qui étendra la classe `ReceiverAdapter` afin de permettre aux autres instances d'interargir avec.

Coté canaux, l'application en utilisera un seul canal : le canal _"channel"_ qui sera utilisé pour être être notifié de changement d'état sur la vue (il sera donc connecté à la classe implémentant l'interface `MembershipListener` (ie. `ChangeInfraListener`) et qui sera également utilisé pour la communication point à point entre les différentes instances : pour rappel, à un membre d'une vue est associée une adresse unique et une vue contient l'ensemble des membres de cette dernière.

A noter que pour la récupération de la donnée au démarrage d'une instance, il aurait également été possible d'implémenter les méthodes `getState()` et `setState()` (en combinaison de l'utilisation de la méthode `connect(<String> , <Address>, <String>, <long>)` sur l'instance de `Channel` utilisée). Cependant, il était, quand même nécessaire d'implémenter la méthode `viewAccepted()` afin d'être notifié du démarrage ou de l'arrêt d'une instances dans le système et cela aurait été redondant avec les notifications reçues (en effet, récupérer l'état des autres instances des membres de la vue ne dispense pas de recevoir l'état de la vue via la méthode `viewAccepted()`). Aussi, je n'ai pas utilisé cette fonctionnalité de JGroups.

Diagramme de séquence lors de la connexion d'une nouvelle instance de l'application au système, coté nouvelle instance mais aussi coté instances déjà présentes dans le système :

![medium](http://4.bp.blogspot.com/-DsKcW_KWIZM/TWFDF_0MbsI/AAAAAAAAAUw/EAXva9PJsm8/s1600/jgroups_seq_diagram.png)

#Mise en oeuvre

A noter que le code écrit ici ne contiendra pas les imports par souci de lisibilité.

##La classe JGroupsClient

Commençons donc par la mise en oeuvre de la classe `JGroupsClient` :

```java
package com.jetoile.jgroups.sample;
public class JGroupsClient extends ReceiverAdapter {
 
    final static private Logger LOGGER = LoggerFactory.getLogger(JGroupsClient.class);
 
    final private Data data = new Data();
    private RpcDispatcher rpcDispatcher;
    private Channel channel;
 
    public JGroupsClient(final String data) {
        this.data.setData(data);
    }
 
    public void stop() throws IOException {
        this.channel.close();
    }
 
    public void start() throws ChannelException {
        this.channel = new JChannel("default-udp.xml");
 
        final ChangeInfraListener changeSetListener = new ChangeInfraListener(channel);
        rpcDispatcher = new RpcDispatcher(this.channel, null, changeSetListener, this);
        changeSetListener.setRpcDispatcher(rpcDispatcher);
        this.channel.connect("privateChannel");
        this.data.setAddress(this.channel.getAddress());
    }
 
    public Data getData() {
        return this.data;
    }
}
```

Dans cette classe, on observe donc que l'on a :

* la méthode `start()` qui a à sa charge la partie connexion à JGroups,
* la méthode `getData()` qui correspond à la méthode exposée utilisée pour transmettre la valeur de la donnée aux autres instances.

##La classe ChangeInfraListener

Pour la classe `ChangeInfraListener`, nous aurons :
```java
package com.jetoile.jgroups.sample;
 
public class ChangeInfraListener implements MembershipListener {
 
    final static private Logger LOGGER = LoggerFactory.getLogger(ChangeInfraListener.class);
 
    final private Map<Address, String> dataCache = new HashMap<Address, String>();
 
    final private Channel privateChannel;
 
    private RpcDispatcher rpcDispatcher;
 
    public ChangeInfraListener(final Channel privateChannel) {
        this.privateChannel = privateChannel;
    }
 
    public void setRpcDispatcher(RpcDispatcher rpcDispatcher) {
        this.rpcDispatcher = rpcDispatcher;
    }
 
    @Override
    public void viewAccepted(View new_view) {
        // when a new member is up
        List<Address> newAddresses = getNewAddresses(new_view.getMembers());
 
        newAddresses.remove(privateChannel.getAddress());
 
        List<Address> ads = new ArrayList<Address>();
        for (Address ad : newAddresses) {
            if (!dataCache.containsKey(ad)) {
                ads.add(ad);
            }
        }
 
        if (!ads.isEmpty()) {
            MethodCall methodCall = new MethodCall("getData", new Object[] {}, new Class[] {});
            LOGGER.debug("invoke remote getData on: {}", ads);
 
            RspList resps = rpcDispatcher.callRemoteMethods(ads, methodCall, RequestOptions.SYNC);
            LOGGER.debug("after invoke getData - nb result {}", resps.numReceived());
 
            if (resps.numReceived() == 0) {
                LOGGER.debug("retry...");
                resps = rpcDispatcher.callRemoteMethods(ads, methodCall, RequestOptions.SYNC);
            }
 
            for (Object resp : resps.getResults()) {
                Data data = (Data) resp;
                LOGGER.debug("new data: {}", data);
                dataCache.put(data.getAddress(), data.getData());
            }
        }
 
        List<Address> olds = getObsoleteAddresses(new_view.getMembers());
        for (Address old : olds) {
            LOGGER.debug("remove data: {}", old);
            dataCache.remove(old);
        }
    }
 
    @Override
    public void suspect(Address suspected_mbr) {
        // NOTHING TO DO
    }
 
    @Override
    public void block() {
        // NOTHING TO DO
    }
 
    List<Address> getNewAddresses(Vector<Address> newMembers) {
        List<Address> result = new ArrayList<Address>();
        for (Address address : newMembers) {
            if (!this.dataCache.containsKey(address)) {
                result.add(address);
            }
        }
        return result;
    }
 
    List<Address> getObsoleteAddresses(Vector<Address> newMembers) {
        List<Address> result = new ArrayList<Address>();
        for (Address address : this.dataCache.keySet()) {
            if (!newMembers.contains(address)) {
                result.add(address);
            }
        }
        return result;
    }
}
```

Dans cette classe, on observe que la méthode `viewAccepted()` (qui est la méthode _call-backé_ par JGroups lors d'une modification de la vue (ie. lors de la connexion ou de la déconnexion d'un autre membre du groupe)), invoque, si un nouveau membre est apparu, l'appel de la méthode distante `getData()` sur la nouvelle instance en question. 

Il est intéressant de noter l'utilisation qui est faite de la classe `RequestOptions` mais également le fait que les méthode `block()` et `suspect()` n'ont pas été spécifiées dans notre cas d'utilisation. 

##La classe Data

Enfin, la classe `Data` qui sera utilisée est la suivante (par souci de lisibilité, les méthodes `equals()` et `hashCode()` ne sont pas détaillées ici) :

```java
package com.jetoile.jgroups.sample;
 
public class Data implements Serializable {
 private Address address;
 private String data;
 
 public Data() {
 }
 
 public Address getAddress() {
  return address;
 }
 
 public void setAddress(final Address address) {
  this.address = address;
 }
 
 public String getData() {
  return data;
 }
 
 public void setData(String data) {
  this.data = data;
 }
 
 @Override
 public int hashCode() {
  // cf. gitHub
  return 0;
 }
 
 @Override
 public boolean equals(Object obj) {
  // cf. gitHub
  return true;
 }
 
 @Override
 public String toString() {
  return "Data [address=" + address + ", data=" + data + "]";
 }
}
```

Cette classe, comme on peut le remarquer, n'a rien de particulier, si ce n'est qu'elle est sérialisable.

Ainsi, on peut voir que l'implémentation est très simple (je ne commenterai donc pas ce qui est fait ici...).

#Exécution et utilisation

L'exécution, quant à elle, pourra se faire avec une classe de type :
```java
package com.jetoile.jgroups.sample.sample;
 
public class JGroupsClientTest {
 
 public static void main(String[] args) throws ChannelException {
  JGroupsClient jgroupsClient = new JGroupsClient("toto");
  jgroupsClient.start();
 }
}
```
A noter que si les différentes instances venaient à ne pas se voir, cela peut provenir d'un souci avec la configuration réseau et qu'il est possible de palier à ce problème en forçant l'utilisation d'adresse IPV4 avec l'option JVM suivante :
```bash
-Djava.net.preferIPv4Stack=true
```

#Conclusion
On a vu ici que permettre la communication d'instances d'une application de manière distribuée était aisée avec JGroups. Bien sûr (et comme je l'ai fait remarqué précédemment), la notion de _tuning_ de JGroups (ie. la configuration de la couche protocolaire - cf. mon [article précédent](/2010/12/jgroups-tour-d.html#protocoles)) n'a pas été abordée, mais cela doit être fait en fonction des besoins de l'infrastructure (trafic réseau, firewall, sécurité, ...) et je laisse donc ce point à la convenance de chacun ;-).

Ici se clôture donc la partie JGroups de notre petit POC jmanager4all qui nous a permis de voir comment JGroups répondait à notre besoin mais également comment il allait être utilisé par la suite.

Le prochain article s'attaquera donc à la partie interopérabilité avec JMX.

A oui... j'oubliais... le code de se petit POC se trouve sur GitHub : https://github.com/jetoile/jgroups-sample
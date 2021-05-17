---
title: "Pour les gouverner tous - partie 3/3"
date: 2011-02-24 18:49:13 +0100
comments: true
tags: 
- java
- jmx
- jgroups
---

![left-small](http://3.bp.blogspot.com/_XLL8sJPQ97g/TUcoTratqiI/AAAAAAAAAUQ/Gmc1h0rvA2w/s1600/jmx_jgroups01.png)

Cet article fait suite à mes précédents posts ([ici](/2011/01/pour-les-gouverner-tous-partie-13.html) et [là](/2011/02/pour-les-gouverner-tous-partie-23.html)) et à pour objectif d'intégrer la partie JMX à mon petit POC JGroups afin d'offrir une solution permettant de rendre complètement scalable la partie supervision/administration par JMX d'une application distribuée (ie. d'aggréger tous les MBeans au sein de tous les serveurs JMX). Pour rappel, le post précédent introduisait JGroups dans une petite application qui permettait à chaque instance d'une application d'obtenir la valeur d'une donnée offerte par les autres instances.

<!-- more -->

# Expression du besoin et conception

Comme je l'ai expliqué [ici](/2011/01/pour-les-gouverner-tous-partie-13.html), le principe est simple : en démarrant son application qui aura à sa charge d'appeler un petit bout de code de notre toolkit, tous les MBeans se trouvant sur les autres serveurs JMX (modulo que l'application qui les ait démarrés ait démarré en instanciant notre toolkit) doivent être remonter dans notre serveur JMX courant. Réciproquement, tous les MBeanServer devront enregistrer les MBeans offerts par notre instance d'application.

Pour ce faire, notre toolkit gérera la partie JMX, c'est-à-dire qu'il créera un connecteur JMX serveur (connecteur s'appuyant sur le protocole RMI) permettant d'accéder au MBeanServer courant (ie. il utilisera le MBeanServer s'il en existe un ou, dans le cas échéant, il en créera un) et qu'il l'exposera aux autres instances en fournissant un stub de la partie connecteur JMX cliente. Ce stub permettra aux autres instances d'instancier la couche de communication nécessaire vers le MBeanServer cible. C'est ce stub qui sera transmis par JGroups (et qui remplacera donc la donnée partagée de notre _POC_ JGroups).

Ca va? Tout le monde suit? Bon, on va pouvoir accélérer... ;-)

Ainsi, lorsqu'une nouvelle instance sera démarrée dans le système, les instances déjà présentes seront notifiées de l'arrivé d'un nouveau membre dans la vue ([concepts JGroups](/2010/12/jgroups-tour-d.html) et cf. [post précédent](/2011/02/pour-les-gouverner-tous-partie-23.html)). Suite à cela, elles récupèreront le stub du JMXConnector du nouveau MBeanServer (toujours cf. [post précédent](/2011/02/pour-les-gouverner-tous-partie-23.html)) qui leur permettra de créer un proxy dynamique correspondant aux MBeans présents dans le MBeanServer cible. Ce proxy dynamique sera instancié et enregistré comme étant un nouveau MBean dans le MBeanServer de l'instance courante.

Cependant, ce proxy dynamique ne pourra pas être utilisé directement (sinon ça serait trop simple). En effet, la méthode statique `JMX.newMBeanProxy()` permet de créer un proxy dynamique du MBean standard d'un MBeanServer distant ou local. Cependant, il n'est pas possible de l'enregistrer comme MBean au sein d'un MBeanServer car il ne répond alors pas à la règle : "un MBean doit implémenter une interface de type MBean ou DynamicMBean". Aussi, la solution retenue a été de wrapper ce proxy dynamique dans un MBean dynamique. Ce wrapper fera donc passe-plat (en utilisant la réflexion) avec le proxy dynamique récupéré par la méthode `JMX.newMBeanProxy()` lors de l'invocation d'opérations dessus (pour ce faire, il s'appuiera sur les normes spécifiées par JMX pour l'accès en lecture/écriture aux variables d'instance).

En outre, pour le cas spécifique des MXBeans (un certain nombre de MXBean est spécifié par JMX : http://download.oracle.com/javase/6/docs/api/), un traitement particulier sera effectué. En effet, JMX offre la possibilité de créer directement un proxy de l'interface d'un MXBean donné via la méthode statique `ManagementFactory.newPlatformMXBeanProxy()`. Aussi, pour les MXBeans, notre wrapper ne sera pas utilisé puisqu'il est possible d'enregistrer directement au sein de notre MBeanServer courant le MXBean distant.

Bien sûr, encore une fois, cela aurait été trop simple s'il n'y avait pas un mais... ;-) En effet, la spécification JMX précise qu'il est possible d'avoir plusieurs MXBeans de type `MemoryPoolMXBean`, `MemoryManagerMXBean` et `GarbageCollector` (cf. http://download.oracle.com/javase/6/docs/api/). Dans ces cas précis, ce sont les propriétés de ces MXBeans qui permettent de les différencier : un traitement particulier sera donc effectué dans ces cas précis puisqu'une recherche pour décrouvrir les propritétés du MXBean sera faite.

En outre, un cas particulier lié à la JRE 6 de Sun (Hostpot) a, ici, été traité (du coup, cela rend le code proposé adhérent à notre jdk...) puisque le MXBean `HotpostDiagnostic` a fait l'objet d'une attention particulière...

# Mise en oeuvre

## Intégration de JMX

<u>Note</u> : à noter que le code présenté dans cet article comporte quelques raccourcis et n'a pour objectif que de montrer les points clé. Aussi, certaines portions (comme, entre autre, la gestion des threads, des exceptions ou les méthodes `equals()`, `hashCode()` ou `toString()`) ne sont pas présentes par soucis de clarté. Le code complet peut être trouvé sur GitHub : https://github.com/jetoile/jmanager4all.

Ici, la donnée, traduite par la classe `Data` dans mon article précédent, sera remplacée par le stub au `JMXConnector`. Ce stub sera créé par la classe utilisée pour instancier les objets nécessaires à la bonne initalisation du toolkit et il sera encapsulé dans la classe `JManagerConnector` (à noter, bien sûr que la classe `RMIConnector` (implémentation de l'interface `JMXConnector`) utilisée ici est sérialisable) lors de sa transmission entre les différentes instances de l'application.

```java
public class JManagerConnector implements Serializable {
 private String location;
 private JMXConnector connector;
 
 public JManagerConnector() {
 }
 
 public String getLocation() {
  return location;
 }
 
 public void setLocation(final String location) {
  this.location = location;
 }
 
 public JMXConnector getConnector() {
  return connector;
 }
 
 public void setConnector(final JMXConnector connector) {
  this.connector = connector;
 }
}
```
Du point de vue création des objets JMX exposés (ie. le `JMXConnector` et son stub) et de l'initialisation de ces derniers (ie. démarrage du connecteur), cela sera fait, comme dit précédemment, par notre classe qui nous servira de point d'entrée à notre toolkit `JManager4All` :

```java
public class JManager4All {
 
    final static private Logger LOGGER = LoggerFactory.getLogger(JManager4All.class);
 
    private static final String CONNECTOR_PROTOCOL = "rmi";
 
    private JMXConnectorServer jmxConnector;
    private JMXServiceURL jmxServiceUrl;
    final private JManagerConnector jmanagerConnector = new JManagerConnector();
    private final MBeanServer mBeanServer;
    final private JMXConnectorStubCache connectorsStub = new JMXConnectorStubCache();
 
    private JGroupsBindingComponent jmanagerBindingComponent;
    public JManager4All(final int port) {
        try {
            this.jmxServiceUrl = new JMXServiceURL(CONNECTOR_PROTOCOL, null, port);
        } catch (MalformedURLException e) {
            LOGGER.error("unable to create JMXServiceURL: {}", e);
        }
        this.mBeanServer = ManagementFactory.getPlatformMBeanServer();
        init();
    }
 
    private void init() {
        try {
            this.jmxConnector = JMXConnectorServerFactory.newJMXConnectorServer(this.jmxServiceUrl, null, mBeanServer);
            this.jmxConnector.start();
 
            final ObjectName objectName = new ObjectName(":type=csserver, name=csserver");
            mBeanServer.registerMBean(this.jmxConnector, objectName);
            this.jmanagerConnector.setConnector(this.jmxConnector.toJMXConnector(null));
 
            this.jmanagerBindingComponent = new JGroupsBindingComponent(this.jmanagerConnector);
            this.jmanagerBindingComponent.setConnectorsStub(this.connectorsStub);
        } catch (Exception e) {
            LOGGER.error("unable to init: {}", e);
        }
    }
 
    public void stop() throws IOException {
        this.jmxConnector.stop();
        this.jmanagerBindingComponent.stop();
    }
 
    public void start() {
        this.jmanagerBindingComponent.start();
    }
 
    public JManagerConnector getStubConnector() {
        return this.jmanagerConnector;
    }
}
```

On remarquera également que cette classe instancie et démarre une instance d'un `JGroupsBindingComponent` qui s'occupe de la couche communication pour récupérer nos stubs du système (ici, ça sera JGroups). Je ne reviendrai pas sur les détails d'implémentation de cette partie puisque cela a déjà été traité [ici](/2011/02/pour-les-gouverner-tous-partie-23.html).

A noter également que nos stubs seront conservés dans un cache `JMXConnectorStubCache` qui déclenchera, lorsqu'un élément (ie. un stub d'un `JMXConnector`) sera ajouté ou supprimé, la récupération de nos MBean distant et leur enregistrement au sein du MBeanServer courant (resp. leur suppression).

```java
public class JMXConnectorStubCache implements Serializable {
 
 volatile private Map<JManagerAddress, JMXConnector> connectorsStub = Collections.synchronizedMap(new HashMap<JManagerAddress, JMXConnector>());
 
 synchronized public Map<JManagerAddress, JMXConnector> getValue() {
  return this.connectorsStub;
 }
 
 synchronized public JMXConnector put(JManagerAddress key, JMXConnector value) {
  JMXConnector result = this.connectorsStub.put(key, value);
  MBeanHandler mBeanHandler = new MBeanHandler(key, value);
  mBeanHandler.handleAdd();
  return result;
 }
 
 synchronized public JMXConnector remove(JManagerAddress key) {
  JMXConnector result = this.connectorsStub.remove(key);
  MBeanHandler mBeanHandler = new MBeanHandler(key, result);
  mBeanHandler.handleRemove();
  return result;
 }
}
```
Ce dernier point nous amène au paragraphe suivant, à savoir la récupération des MBeans distants et leur enregistrement.

# Récupération des MBeans distants et enregistrement

La récupération des MBeans distants est globalement simple et suit le schéma suivant (cet algorithme ne s'applique pas aux MXBeans) :

* à partir du `JMXConnector` client, récupération du `MBeanServerConnection`,
* à partir du `MBeanServerConnection`, une recherche est lancée sur le `MBeanServer` distant pour récupérer l'ensemble des `ObjectInstance` présenst sur ce dernier,
* pour chaque `ObjectInstance` qui n'est pas dans le domaine JMX _"remote"_ (qui sera le domaine utilisé pour stocker les proxy des MBeans distants), création d'un proxy dynamique représentant l'objet exposé par le MBean et encapsulation de ce dernier dans la classe `MBeanWrapper` qui est un `DynamicMBean`,
* enfin enregistrement de chaque `DynamicMBean` créé dans le `MBeanServer` local via un `ObjectName` défini grâce à :
	* ses propriétés, 
	* son domaine JMX distant sous forme de propriété _"subdomain"_, 
	* mais également avec une propriété permettant de connaitre la provenance du MBean (propriété qui sera appelée _"instance"_).





```java
public class MBeanHandler {
 private static final String DOMAIN_REMOTE = "remote";
 
 private static final Logger LOGGER = LoggerFactory.getLogger(MBeanHandler.class);
 
 private final MBeanServer mbeanServer = ManagementFactory.getPlatformMBeanServer();
 private final JMXConnector connector;
 private final JManagerAddress address;
 
 public MBeanHandler(final JManagerAddress address, final JMXConnector connector) {
  this.connector = connector;
  this.address = address;
 }
 
 void handleAdd() throws Exception {
  if (connector != null) {
   connector.connect();
   ObjectName objectName = new ObjectName("*:*");
   final MBeanServerConnection mBeanServerConnection = connector.getMBeanServerConnection();
 
   final Set<ObjectInstance> instances = mBeanServerConnection.queryMBeans(objectName, null);
 
   for (ObjectInstance objectInstance : instances) {
    final ObjectName distantObjectName = objectInstance.getObjectName();
    if (DOMAIN_REMOTE.equals(distantObjectName.getDomain())) {
     continue;
    }
    final ObjectName newObjectName = new ObjectName(DOMAIN_REMOTE + ":instance=" + address + ", " + "subdomain=" + distantObjectName.getDomain()
      + ", " + distantObjectName.getKeyPropertyListString());
    final MBeanInfo mBeanInfo = mBeanServerConnection.getMBeanInfo(distantObjectName);
    registerRemoteMBean(mBeanServerConnection, distantObjectName, newObjectName, mBeanInfo);
   }
  }
 }
 
 private void registerRemoteMBean(final MBeanServerConnection mBeanServerConnection, final ObjectName distantObjectName, final ObjectName newObjectName,
   final MBeanInfo mBeanInfo) throws Exception {
  Object mBean = getRemoteMBean(mBeanServerConnection, mBeanInfo, distantObjectName);
  if (mBean != null) {
   mbeanServer.registerMBean(mBean, newObjectName);
  }
 }
 
 public Object getRemoteMBean(final MBeanServerConnection mBeanServerConnection, final MBeanInfo mBeanInfo, final ObjectName distantObjectName)
   throws Exception {
  if (!Boolean.parseBoolean((String) mBeanInfo.getDescriptor().getFieldValue("mxbean"))) {
   LOGGER.debug("current mxBeanInfo : {}", mBeanInfo);
   final Class<?> clazz = Class.forName(mBeanInfo.getClassName());
   final Class<?>[] interfazes = clazz.getInterfaces();
   if (interfazes.length == 1) {
    LOGGER.debug("current interface : {}", interfazes[0]);
    final Object proxy = JMX.newMBeanProxy(mBeanServerConnection, distantObjectName, interfazes[0]);
    final MBeanWrapper mBeanWrapper = new MBeanWrapper(mBeanInfo, proxy);
    return mBeanWrapper;
   }
  }
  return null;
 }
}
```
<br/>

```java
public class MBeanWrapper implements DynamicMBean {
 
 static final private Logger LOGGER = LoggerFactory.getLogger(MBeanWrapper.class);
 
 private final MBeanInfo mBeanInfo;
 private final Object proxy;
 
 public MBeanWrapper(final MBeanInfo mBeanInfo, final Object proxy) {
  this.mBeanInfo = mBeanInfo;
  this.proxy = proxy;
 }
 
 @Override
 public Object getAttribute(String attribute) throws AttributeNotFoundException, MBeanException, ReflectionException {
  MBeanAttributeInfo[] mBeanAttributeInfos = mBeanInfo.getAttributes();
  for (MBeanAttributeInfo mBeanAttributeInfo : mBeanAttributeInfos) {
   if (StringUtils.equals(attribute, mBeanAttributeInfo.getName())) {
    if (mBeanAttributeInfo.isReadable()) {
     return invokeGetter(attribute, mBeanAttributeInfo);
    } else {
     return null;
    }
   }
  }
  return null;
 }
 
 Object invokeGetter(String attribute, MBeanAttributeInfo mBeanAttributeInfo) {
  try {
   Method method = null;
   if (mBeanAttributeInfo.isIs()) {
    method = this.proxy.getClass().getMethod("is" + StringUtils.capitalize(attribute), new Class[0]);
    return method.invoke(this.proxy, new Object[0]);
   } else {
    method = this.proxy.getClass().getMethod("get" + StringUtils.capitalize(attribute), new Class[0]);
    return method.invoke(this.proxy, new Object[0]);
   }
  } catch (Exception e) {
   LOGGER.error("unable to get remote attribute info {}: {}", attribute, e);
  }
  return null;
 }
 
 @Override
 public void setAttribute(Attribute attribute) throws AttributeNotFoundException, InvalidAttributeValueException, MBeanException, ReflectionException {
  MBeanAttributeInfo[] mBeanAttributeInfos = mBeanInfo.getAttributes();
  for (MBeanAttributeInfo mBeanAttributeInfo : mBeanAttributeInfos) {
   if (StringUtils.equals(attribute.getName(), mBeanAttributeInfo.getName())) {
    if (mBeanAttributeInfo.isWritable()) {
     invokeSetter(attribute, mBeanAttributeInfo);
    } else {
     return;
    }
   }
  }
  return;
 }
 
 void invokeSetter(Attribute attribute, MBeanAttributeInfo mBeanAttributeInfo) {
  try {
   Method method = null;
   if (mBeanAttributeInfo.isIs()) {
    method = this.proxy.getClass().getMethod("set" + StringUtils.capitalize(attribute.getName()), boolean.class);
    method.invoke(this.proxy, attribute.getValue());
   } else {
    method = this.proxy.getClass().getMethod("set" + StringUtils.capitalize(attribute.getName()), attribute.getValue().getClass());
    method.invoke(this.proxy, attribute.getValue());
   }
  } catch (Exception e) {
   LOGGER.error("unable to set remote attribute info {}: {}", attribute, e);
  }
 }
 
 @Override
 public AttributeList getAttributes(String[] attributes) {
  AttributeList result = new AttributeList();
  for (String attribute : attributes) {
   Attribute currentAttribute;
   try {
    currentAttribute = new Attribute(attribute, getAttribute(attribute));
    result.add(currentAttribute);
   } catch (Exception e) {
    LOGGER.error("unable to get remote attribute info {}: {}", attribute, e);
   }
  }
  return result;
 }
 
 @Override
 public AttributeList setAttributes(AttributeList attributes) {
  AttributeList result = new AttributeList();
  for (int i = 0; i < attributes.size(); i++) {
   Attribute attribute = (Attribute) attributes.get(i);
   try {
    setAttribute(attribute);
    result.add(attribute);
   } catch (Exception e) {
    LOGGER.error("unable to set remote attribute {}: {}", attribute.getName(), e);
   }
  }
  return result;
 }
 
 @Override
 public Object invoke(String actionName, Object[] params, String[] signature) throws MBeanException, ReflectionException {
  Class[] paramTypes = null;
  if (signature != null) {
   paramTypes = new Class[signature.length];
   for (int i = 0; i < signature.length; ++i) {
    paramTypes[i] = signature[i].getClass();
   }
  }
 
  try {
   Method method = this.proxy.getClass().getMethod(actionName, paramTypes);
   return method.invoke(this.proxy, params);
  } catch (Exception e) {
   LOGGER.error("unable to invoke {}: {}", actionName, e);
  }
  return null;
 }
 
 @Override
 public MBeanInfo getMBeanInfo() {
  return this.mBeanInfo;
 }
}
```

On voit donc que le processus n'est pas vraiment compliqué. Pour le cas des MXBeans, cela est un peu plus sioux... mais globalement pas compliqué non plus ;-)

En effet, plusieurs cas se présentent :

* Pour les `MXBeans` de types (type au sens propriété JMX) __Compilation__, __ClassLoading__, __Memory__, __OperationSystem__, __Runtime__ et __Threading__, cela ne pose pas de problèmes puisque la spécification JMX nous dit qu'il n'y a qu'un seul MBean de ce type dans le `MBeanServer` pour domaine _java.lang_. Dans ce cas, il suffit juste d'instancier un proxy dynamique du MXBean lui-même et de l'enregistrer avec le "bon" `ObjectName`.
* par contre, pour les `MXBeans` de types __MemoryPool__, __GarbageCollector__ et __MemoryManager__, vu qu'il peut y en avoir plusieurs, une recherche est effectuée (en limitant la recherche aux `MBeans` voulus bien sûr...) sur le `MBeanServer` distant afin d'obtenir (mais également d'instancier) le bon type de proxy dynamique mais également les bonnes propriétés sous lequel il est enregistré.

```java
final private static Map<String, Class<?>> MXBEAN_MAP = new HashMap<String, Class<?>>();
 static {
  MXBEAN_MAP.put(ManagementFactory.COMPILATION_MXBEAN_NAME, CompilationMXBean.class);
  MXBEAN_MAP.put(ManagementFactory.CLASS_LOADING_MXBEAN_NAME, ClassLoadingMXBean.class);
  MXBEAN_MAP.put(ManagementFactory.MEMORY_MXBEAN_NAME, MemoryMXBean.class);
  MXBEAN_MAP.put(ManagementFactory.OPERATING_SYSTEM_MXBEAN_NAME, OperatingSystemMXBean.class);
  MXBEAN_MAP.put(ManagementFactory.RUNTIME_MXBEAN_NAME, RuntimeMXBean.class);
  MXBEAN_MAP.put(ManagementFactory.THREAD_MXBEAN_NAME, ThreadMXBean.class);
  MXBEAN_MAP.put("java.util.logging:type=Logging", LoggingMXBean.class);
 }
 
 public RemoteMXBeanHandler() {
 }
 
 public Object getRemoteMBean(final MBeanServerConnection mBeanServerConnection, final MBeanInfo mBeanInfo, final ObjectName distantObjectName)
   throws Exception {
  if (Boolean.parseBoolean((String) mBeanInfo.getDescriptor().getFieldValue("mxbean"))) {
   LOGGER.debug("current mxBeanInfo : {}", mBeanInfo);
   Object proxy = getMxBeanProxy(mBeanServerConnection, mBeanInfo, distantObjectName);
   return proxy;
  }
  return null;
 }
 
 private Object getMxBeanProxy(final MBeanServerConnection mBeanServerConnection, final MBeanInfo mBeanInfo, final ObjectName distantObjectName)
   throws IOException {
  String distantName = distantObjectName.toString();
  int index = distantName.indexOf(",");
  String substring = distantName.substring(0, (index != -1) ? index : distantName.length());
  Class<?> mBeanClass = MXBEAN_MAP.get(substring);
  if (mBeanClass != null) {
   return ManagementFactory.newPlatformMXBeanProxy(mBeanServerConnection, distantName, mBeanClass);
  } else if (!StringUtils.equals(ManagementFactory.MEMORY_POOL_MXBEAN_DOMAIN_TYPE, substring)
    && !StringUtils.equals(ManagementFactory.GARBAGE_COLLECTOR_MXBEAN_DOMAIN_TYPE, substring)
    && !StringUtils.equals(ManagementFactory.MEMORY_MANAGER_MXBEAN_DOMAIN_TYPE, substring)
    && !StringUtils.contains(substring, "com.sun.management")) {
   return ManagementFactory.newPlatformMXBeanProxy(mBeanServerConnection, ManagementFactory.RUNTIME_MXBEAN_NAME, RuntimeMXBean.class);
  } else {
   return null;
  }
 }
 
 private void handleSpecificMXBean(final MBeanServerConnection mBeanServerConnection) throws Exception {
  // cf. http://download.oracle.com/javase/6/docs/api/
 
  // traitement particulier pour les MXBeans de type MemoryPoolMXBean
  registerOtherMxBean(mBeanServerConnection, ManagementFactory.MEMORY_POOL_MXBEAN_DOMAIN_TYPE, MemoryPoolMXBean.class);
  // traitement particulier pour les MXBeans de type MemoryManagerMXBean
  registerOtherMxBean(mBeanServerConnection, ManagementFactory.MEMORY_MANAGER_MXBEAN_DOMAIN_TYPE, MemoryManagerMXBean.class);
  // traitement particulier pour les MXBeans de type GarbageCollector
  registerOtherMxBean(mBeanServerConnection, ManagementFactory.GARBAGE_COLLECTOR_MXBEAN_DOMAIN_TYPE, GarbageCollectorMXBean.class);
 
  // traitement particulier pour les MXBeans de type Hotspot
  registerOtherMxBean(mBeanServerConnection, "com.sun.management:type=HotSpotDiagnostic", HotSpotDiagnosticMXBean.class);
 }
 
 private void registerOtherMxBean(final MBeanServerConnection mBeanServerConnection, String type, Class<?> clazz) throws Exception {
  final ObjectName requestedObjectName = new ObjectName(type + ",*");
  final Set<ObjectName> objectNames = mBeanServerConnection.queryNames(requestedObjectName, null);
  for (ObjectName objectName : objectNames) {
   final Object proxy = ManagementFactory.newPlatformMXBeanProxy(mBeanServerConnection, objectName.getCanonicalName(), clazz);
   final ObjectName newObjectName = new ObjectName("remote:instance=" + address + ", " + "subdomain=" + objectName.getDomain() + ", "
     + objectName.getKeyPropertyListString());
   mbeanServer.registerMBean(proxy, newObjectName);
  }
 }
```
Pour le cas particulier du `MXBean HotspotDiagnostic`, je vous laisse le soin de jeter un oeil sur le code... ;-)

# Désenregistrement d'une instance (et donc de ses MBeans associés)

Bien entendu, si une instance venait à disparaitre, il est nécessaire de désenregistrer ses `MBeans` pour chaque instance de l'application encore présente.

Là encore, la principe est ultra-simple puisqu'il suffit de récupérer l'ensemble des MBeans qui se trouvent dans le domaine JMX _"remote"_ (qui pour rappel est le domaine utilisé pour enregistrer tous nos proxy) avec la "bonne" propriété (à savoir la propriété `instance` qui doit valoir la valeur de l'adresse de l'instance à supprimer) et de les désenregistrer du `MBeanServer` local.

```java
void handleRemove() throws Exception {
  final ObjectName queryObjectName = new ObjectName(DOMAIN_REMOTE + ":instance=" + address + ",*");
  final Set<ObjectName> objectNames = mbeanServer.queryNames(queryObjectName, null);
  for (ObjectName objectName : objectNames) {
   LOGGER.debug("remove from mBeanServer objectName : {}", objectName);
   mbeanServer.unregisterMBean(objectName);
  }
 }
```

# Conclusion
On a vu tout au long de cette article comment il était possible d'utiliser JMX coté client (on récupère des `MBeans` distants) mais également coté serveur (on créé des `MBeans`). Bien sûr, la couche JGroups n'est qu'un prétexte mais je trouvais intéressant de pouvoir s'appuyer sur ce dernier pour être notifié de changement dans le système. Si je suis motivé, j'intégrerais peut être une implémentation NoSQL pour stocker et partager les JMXConnector ou un truc du genre... ;-)

Par contre, je n'ai pas testé le comportement dans le cas où des notifications seraient émises... en outre, si un service de __Relation__ JMX était utilisé, je pense que cela pourrait poser quelques soucis...

A noter que je suis tombé tout récemment sur une [autre solution](http://blog.infin-it.fr/2010/08/05/aggregateur-jmx-2/) (que je n'ai pas testé) qui permet d'aggréger des informations JMX au sein d'un même serveur `MBeanServer`. La cible n'est pas tout à fait identique mais peut être suffisante pour la plupart des cas... par contre, il faut aimer Spring... ;-)

# Pour aller plus loin...

* Site officiel de JGroups : http://www.jgroups.org/
* Wiki de JGroups : http://community.jboss.org/wiki/JGroups
* Site de la JSR 3 - Java Management Extensions (JMX) Specification : http://www.jcp.org/en/jsr/detail?id=3
* Site de la JSR 160 - Java Management Extensions (JMX) Remote API : http://jcp.org/en/jsr/detail?id=160
* Site officiel d'Oracle sur JMX : http://www.oracle.com/technetwork/java/javase/tech/docs-jsp-135989.html

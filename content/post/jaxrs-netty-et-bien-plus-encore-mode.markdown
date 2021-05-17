---
title: "JAXRS, Netty et bien plus encore... mode d'emploi..."
date: 2014-03-10 08:03:49 +0100
comments: true
tags: 
- netty
- java
- jmx
- metrics
- rest
- performance
- jolokia
- maven
---

![left-small](http://1.bp.blogspot.com/-HYBTn5anFdE/Uxdv3cRgu_I/AAAAAAAABP4/gsCZjUIxDyQ/s1600/resteasy_jolokia_metrics.png)

L'informatique évolue constamment et c'est également le cas des architectures qui ont tendance à s'orienter de plus en plus vers l'utilisation de services REST. Ces services REST doivent, en outre, être de plus en plus véloces afin de pouvoir répondre à une charge de plus en plus forte (que ce soit d'un point de vue temps de réponse mais également d'un point de vue charge suportée). C'est dans ce contexte que des solutions comme [Restlet](http://restlet.org/) ou [RestX](http://restx.io/) (pour n'en citer que quelques-unes) ont vu le jour.

En effet, en plus d'offrir la possibilité de servir des services REST, elles s'appuient sur des framework dont la particularité est d'offrir des traitements non bloquant sur les entrées/sorties (NIO).

C'est dans ce contexte que cet article parlera principalement de Resteasy-Netty 3 (la version 3 a été utilisé en raison de contraintes techniques (connexion à [Apache Cassandra](http://cassandra.apache.org/) dont le [driver](https://github.com/datastax/java-driver) utilise Netty 3)).

Cependant, ce ne sera pas le seul protagoniste car, comme on verra par la suite, il est très simple à utiliser...

Le vrai sujet de cet article est, en fait, comment il a été possible d'ajouter d'autres framework comme Swagger ou Jolokia à Resteasy-Netty 3.

Cet article sera découpé en deux parties :

* Besoin et conception
* Mise en oeuvre

Le code se trouve sur Github [ici](https://github.com/jetoile/resteasy-netty-sample).

<!-- more -->

# Besoin et conception

Le besoin était d'offrir un ensemble de services REST qui devait être suffisamment véloce pour répondre au besoin de performance en terme de charge mais également en terme de temps de réponse.

Venant du monde Java et plus précisément de Java EE, il aurait été pertinent de partir sur une solution classique à base de [JAX-RS](https://jcp.org/en/jsr/detail?id=311) ([Jersey](https://jersey.java.net/) ou [RestEasy](http://www.jboss.org/resteasy)) hébergée par un [Tomcat](http://tomcat.apache.org/) ou un [Jetty](http://www.eclipse.org/jetty/).

Cependant, une crainte était que le mode de fonctionnement des Servlets soit limitant concernant les entrées/sorties. Bien sûr, il était possible d'utiliser le connecteur NIO de Tomcat mais ce n'est pas cette solution qui a été retenue... ;-)

Suite à la lecture de l'excellent [article](http://blog.xebia.fr/2011/11/09/java-nio-et-framework-web-haute-performance/) sur le retour d'expérience de [Séven](https://twitter.com/slemesle) et de [Julien](https://twitter.com/julienBuret) lors du challenge USI 2011, le choix a été fait de partir sur une solution basée sur [Netty](http://netty.io/).

Par contre, développer des services directement sur Netty était embêtant et risquait surtout de rebuter l'équipe de développement. De la même manière, introduire un nouveau framework disposant de ses propres API n'était pas préconisé (NDLR : les standards c'est bien ! ;-) ).

C'est pour cette raison qu'il était préférable de trouver une solution alliant à la fois les avantages de NIO (et si possible s'appuyant sur Netty) et de JAX-RS.

Ainsi, il a été décidé de partir sur Resteasy-Netty 3 qui semblait offrir le meilleur des deux mondes (je dis "semblais" car aucun comparatif en charge des différents protagonistes n'a été réalisé et les résultats obtenus ont été suffisamment satisfaisant pour n'avoir pas à pousser plus loin l'expérimentation).

L'un des autres avantages de n'avoir pas utiliser un conteneur de Servlet classique était qu'il permettait de rendre le livrable auto-porteur et légé (il aurait bien sûr été possible d'embarquer un Tomcat ou Jetty embedded ou de "s'embeddé" dans un Tomcat via le goal exec-war de Tomcat7-maven-plugin).

Bien sûr, l'application devait être administrable et supervisable.

Enfin, cerise sur le gateau, intégrer une solution comme [Swagger](https://helloreverb.com/developers/swagger) pour documenter les API REST était un _"nice to have"_.

Pour notre cas d'exemple, le seul service exposé sera le classique service qui répète ce qu'on lui demande...

Il répondra donc à une requête de type GET du type :
`http://localhost:8081/sample/say/<message>`

Du coté de la réponse, elle aura la forme suivante : 
```javascript
{
    "message": <message>,
    "time": "2014-03-05T10:55:39.835+01:00"
}
```

La date de la réponse sera ajoutée juste pour le "fun" ;-) 

# Mise en oeuvre

A titre informatif, les versions des différentes librairies qui sont utilisés dans les exemples de code ci-dessous sont les suivantes (au format gradle pour gagner de la place) :
```text
compile group: 'org.jboss.resteasy', name: 'jaxrs-api', version:'3.0.4.Final'
compile group: 'org.jolokia', name: 'jolokia-jvm', version:'1.1.2'
compile group: 'com.wordnik', name: 'swagger-jaxrs_2.10', version:'1.3.0'
compile group: 'com.wordnik', name: 'swagger-annotations_2.10', version:'1.3.0'
compile group: 'javax.servlet', name: 'servlet-api', version:'2.5'
compile group: 'com.codahale.metrics', name: 'metrics-core', version:'3.0.1'
compile group: 'org.jboss.resteasy', name: 'resteasy-netty', version:'3.0.6.Final'
compile group: 'org.jboss.resteasy', name: 'resteasy-jackson-provider', version:'3.0.6.Final'
compile group: 'org.codehaus.jackson', name: 'jackson-core-asl', version:'1.9.13'
compile group: 'org.codehaus.jackson', name: 'jackson-mapper-asl', version:'1.9.13'
compile group: 'commons-configuration', name: 'commons-configuration', version:'1.9'
compile group: 'commons-collections', name: 'commons-collections', version:'3.2.1'
compile group: 'commons-io', name: 'commons-io', version:'2.4'
compile group: 'joda-time', name: 'joda-time', version:'2.3'
compile(group: 'com.google.guava', name: 'guava', version:'15.0') { exclude(module: 'jsr305') }
```

## Implémentation du service REST

Le mise en place du service REST basé sur JAX-RS est on ne peut plus trivial... et la classe ci-dessous fait humblement l'affaire :

```java
@Path("/sample")
public class SimpleService {
    private final static Logger log = LoggerFactory.getLogger(SimpleService.class);

    @GET
    @Path("/say/{msg}")
    @Produces(MediaType.APPLICATION_JSON)
    public Response getPortDataSet(@PathParam("msg") String message) {
        DtoResponse response = new DtoResponse();
        try {
            response.setMessage(message);
            response.setTime(DateTime.now());
        } catch (Exception e) {
            log.error("internal error: {}", e);
            return Response.status(Response.Status.INTERNAL_SERVER_ERROR).build();
        }
        return Response.ok(response).build();
    }
}
```

Du coté de l'objet retourné par la réponse au format JSON, Jackson intégré à Resteasy a été utilisé pour la partie marshalling/unmarshalling.

Coté gestion des dates, ce sera JodaTime (l'application tourne avec Java 7).

Du coup, un objet DTO a été écrit et annoté à l'aide d'annotations JAXB :

```java
@XmlRootElement
public class DtoResponse {

    private String message;
    private DateTime time;

    public DtoResponse() {}

    public String getMessage() { return message;}

    public void setMessage(String message) { this.message = message; }

    public DateTime getTime() { return time; }

    public void setTime(DateTime time) { this.time = time; }
}
```

## Mise en oeuvre de Resteasy-Netty 3

Mettre en place Resteasy-Netty 3 est très simple, d'après la [documnentation](https://docs.jboss.org/resteasy/docs/3.0.6.Final/userguide/pdf/resteasy-reference-guide-en-US.pdf), il suffit de faire :

```java
public static void start(ResteasyDeployment deployment) throws Exception {
  netty = new NettyJaxrsServer();
  netty.setDeployment(deployment);
  netty.setPort(TestPortProvider.getPort());
  netty.setRootResourcePath("");
  netty.setSecurityDomain(null);
  netty.start();
 }
```
 
et c'est donc ce que l'on va faire... ;-)

[Apache commons-configuration](http://commons.apache.org/proper/commons-configuration/) a été utilisé afin de déporter la configuration dans un fichier _properties_.

```java
public class Client {
    private final static Logger LOGGER = LoggerFactory.getLogger(Client.class);
    private static final String CONF_PROPERTIES = "conf.properties";
    private static Configuration config;

    public static void main(String[] args) throws ConfigurationException, BootstrapException {
        try {
            config = new PropertiesConfiguration(CONF_PROPERTIES);
        } catch (ConfigurationException e) {
            throw new BootstrapException("bad config", e);
        }
        initServer();
    }

    private static void initServer() {
        SimpleService service = new SimpleService();
        ResteasyDeployment deployment = new ResteasyDeployment();

        int nettyPort = 8081;

        if (config != null) {
            deployment.setAsyncJobServiceEnabled(config.getBoolean("netty.asyncJobServiceEnabled", false));
            deployment.setAsyncJobServiceMaxJobResults(config.getInt("netty.asyncJobServiceMaxJobResults", 100));
            deployment.setAsyncJobServiceMaxWait(config.getLong("netty.asyncJobServiceMaxWait", 300000));
            deployment.setAsyncJobServiceThreadPoolSize(config.getInt("netty.asyncJobServiceThreadPoolSize", 100));

            nettyPort = config.getInt("netty.port", TestPortProvider.getPort());
        } else {
            LOGGER.warn("is going to use default netty config");
        }

        deployment.setResources(Arrays.<Object>asList(service));

        NettyJaxrsServer netty = new NettyJaxrsServer();
        netty.setDeployment(deployment);
        netty.setPort(nettyPort);
        netty.setRootResourcePath("");
        netty.setSecurityDomain(null);
        netty.start();
    }    
}
```

On y constate que pour ajouter un service, il suffit juste de déclarer la classe implémentant JAX-RS via la méthode `setResources()` sur l'instance de `ResteasyDeployment` fournit au serveur __NettyJaxrs__ :

```java
SimpleService service = new SimpleService();
ResteasyDeployment deployment = new ResteasyDeployment();
deployment.setResources(Arrays.<Object>asList(service));
NettyJaxrsServer netty = new NettyJaxrsServer();
netty.setDeployment(deployment);
...
netty.start();
```

Et voilà! On dispose désormais d'un programme exécutable qui démarre un serveur REST basé sur Netty.

Plutôt simple non? ;-)

## Configuration de Jackson

Avec le code précédent, si la commande suivante est exécutée :
 
```bash
 curl -XGET  http://localhost:8081/sample/say/hello 
``` 

Le résultat suivant est obtenu : 
```javascript
{
    "message": "hello",
    "time": 1402560438128
}
```

Hum... la date n'est pas formatté comme il faut... pas glop... :'(

En fait, il est possible de modifier la configuration de [Jackson](http://jackson.codehaus.org/) et on trouve, dans la littérature, un moyen très simple de le faire en configurant l'_ObjectMapper_ comme suit :
```java
objectMapper.configure(SerializationConfig.Feature.WRITE_DATES_AS_TIMESTAMPS, false);
```

Bien sûr, le but n'étant pas de faire cette transformation manuellement à chaque fois, on préfère laisser Resteasy le gérer lui-même.

Ainsi, il existe [deux autres](http://stackoverflow.com/questions/19229341/changing-default-json-time-format-with-resteasy-3-x) manières de faire :

* Le faire par annotation
* Le faire par configuration dans le `web.xml`

Cependant, dans notre cas, nous ne disposons pas d'un conteneur de Servlet classique et il n'est donc pas possible de s'appuyer sur une configuration par web.xml. Pour le faire par annotation, j'avoue ne pas avoir testé mais je suis sceptique...

Du coup, il reste une possibilité qui est de déclarer un `JacksonConfig` et de demander à Resteasy-Netty de nous l'enregistrer en tant que _provider_ (en gros de demander à Resteasy-Netty de faire manuellement ce qui est fait via le `web.xml`) :

```java
public class JacksonConfig implements ContextResolver<ObjectMapper> {
    private final ObjectMapper objectMapper;

    public JacksonConfig() {
        objectMapper = new ObjectMapper();
        objectMapper.configure(SerializationConfig.Feature.WRITE_DATES_AS_TIMESTAMPS, false);
        objectMapper.setSerializationInclusion(JsonSerialize.Inclusion.NON_NULL);
    }

    @Override
    public ObjectMapper getContext(Class<?> objectType) {
        return objectMapper;
    }
}
```

Pour l'enregistrement, c'est très simple puisqu'il suffit d'ajouter la ligne suivante :
`deployment.setProviderClasses(Lists.newArrayList("fr.jetoile.sample.JacksonConfig"));`

Et voilà! C'est tout!

Encore une fois, simple et efficace et le résultat obtenu est bien celui escompté :
```javascript
{
    "message": "hello",
    "time": "2014-06-12T10:06:54.553+02:00"
}
```

A noter que l'_ancienne_ version de Jackson est utilisée ici car c'est celle qui est utilisé par défaut par Resteasy. Il aurait été possible de l'utiliser dans sa version plus récente mais j'avoue ne pas avoir fait l'exercice... (cf. [ici](https://docs.jboss.org/resteasy/docs/3.0.6.Final/userguide/html/json.html#d4e1046))

## Intégration de Metrics

Afin de permettre une mesure des temps d'invocation de différentes opérations, la librairie [Metrics](http://metrics.codahale.com/) a été utilisée.

Pour plus d'information dessus, le sujet est très bien traité sur le blog de [Charles](https://twitter.com/clescot) :

* [Metrics : Les Bases](http://clescot.com/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-les-bases/)
* [Metrics : Intégration Avec JEE](http://clescot.com/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-jee/)
* [Metrics : Intégration Avec Spring Et Guice](http://clescot.com/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-spring-et-guice/)
* [Metrics : Intégration Avec JDBC, Logback Et Jersey](http://clescot.com/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-JDBC-logback-et-jersey/)

Dans notre cas, bien sûr, pas de _Spring_, de _Guice_ ou de _Servlet Listener_. Une simple variable de classe dans la classe portant la méthode `main()` suffit :
```java
public static MetricRegistry metricRegistry;
    
public static void main(String[] args) throws ConfigurationException, BootstrapException {
   ...

    metricRegistry = new MetricRegistry();
    final JmxReporter reporter = JmxReporter.forRegistry(metricRegistry).build();
    reporter.start();
}
```

Concernant l'utilisation à proprement parler, cela se fait de cette manière (dans notre cas, utilisation du __Timer__ qui représente un histogramme des durées et une mesure de la fréquence d’apparition) :
```java
@GET
@Path("/say/{msg}")
@Produces(MediaType.APPLICATION_JSON)
public Response getPortDataSet(@PathParam("msg") String message) {

    final Timer timer = Client.metricRegistry.timer(name(SimpleService.class, "say-service"));
    final Timer.Context context = timer.time();
    try {
  ...
        return Response.ok(response).build();
    } finally {
        if (context != null) context.stop();
    }
}
```

Une fois l'application démarrée et après 1 ou 2 appels, l'ObjectName apparait dans la console JMX et il est alors possible de voir les différents résultats.

![medium](http://1.bp.blogspot.com/-RMh6acQ8Qkc/UxdsRqymVkI/AAAAAAAABPk/r3rvWdgPNtg/s1600/metrics-hawtio.png)

On constate encore une fois que la mise en place de Metrics n'a demandé aucun effort particulier.

## Intégration de Jolokia

Une autre étape de notre périple consiste à activer Jolokia que j'ai déjà présenté dans un [article précédent](/2014/03/jolokia-le-piment-qui-vous-veut-du-bien.html).

Dans notre cas d'usage, cela sera fait de manière programmatique.

Pour ce faire, c'est encore une fois très simple et il suffit d'ajouter le code suivant dans notre classe principale :
```java
private static void initJolokiaServer() {
    try {
        JolokiaServerConfig config = new JolokiaServerConfig(new HashMap<String, String>());

        JolokiaServer jolokiaServer = new JolokiaServer(config, true);
        jolokiaServer.start();
    } catch (Exception e) {
        LOGGER.error("unable to start jolokia server", e);
    }
}
```

Concernant sa configuration, pour éviter d'avoir à aller chercher des properties et à repeupler une Map, le fichier par défaut (`default-jolokia-agent.properties`) a été copié (en renseignant certaines informations comme le user/password) dans le répertoire `src/main/resources` :
```text
# Configuration properties for the JVM jolokia-agent

# Host address to bind to.
# Default: localhost, determinated dynamically via InetAddress.getLocalHost()
host=0.0.0.0

# Port to listen to
port=7778

# Context path
agentContext=/jolokia

# Backlog of request to keep when queue
backlog=10

# Possible values:
#  * "fixed"  : Thread pool with at max nrThreads
#  * "single" : A single thread serves all requests (default)
#  * "cached" : A thread pool which reuses threads and creates threads on demand (unbounded)
# executor=fixed
# nrThreads=5

# User and password for basic authentication
user=jolokia
password=jolokia

# How many entroes to keep in the history
historyMaxEntries=10

# Switch on debugging
debug=false

# How many debug entries to keep on the server side which can be queried by JMX
debugMaxEntries=100

# Maximum traversal depth for serialization of complex objects.
maxDepth=15

# Maximum size of collections returned during serialization.
maxCollectionSize=1000

# Maximum number of objects returned by serialization
maxObjects=0
```

Un petit coup de (le user jolokia et le mot de passe jolokia ont été positionné dans le fichier _properties_) :
```bash
curl -XGET -u jolokia:jolokia http://localhost:7778/jolokia/version
```

nous permet bien d'obtenir la réponse attendue :
```javascript
{
    "timestamp": 1394036344,
    "status": 200,
    "request": {
        "type": "version"
    },
    "value": {
        "protocol": "7.0",
        "agent": "1.1.2",
        "info": {}
    }
}
```

A noter que les user/password ont été positionné car cela permet une connexion via [Hawt.io](http://hawt.io/). 

## Intégration de Swagger

[Swagger](https://helloreverb.com/developers/swagger) offre une manière très simple de documenter une API REST. En effet, en s'appuyant sur des annotations à mettre dans la classe de service, elle permet d'offrir une interface d'écrivant les API.

![medium](http://4.bp.blogspot.com/-92lPQm1F_DM/UxdsgUK3bDI/AAAAAAAABPs/yWh6QRHKkFc/s1600/swagger.png)

Pour le mettre en place, il suffit donc de rajouter les annotations adéquates à notre classe `SimpleService` :

```java
@Api(value = "/sample", description = "the sample api")
@Path("/sample")
public class SimpleService {
    private final static Logger log = LoggerFactory.getLogger(SimpleService.class);

    @GET
    @Path("/say/{msg}")
    @Produces(MediaType.APPLICATION_JSON)
    @ApiOperation(value = "repeat the word",
            notes = "response the word",
            response = DtoResponse.class)
    @ApiResponses(value = {@ApiResponse(code = 500, message = "Internal server error")})
    public Response getPortDataSet(@PathParam("msg") String message) {

        final Timer timer = Client.metricRegistry.timer(name(SimpleService.class, "say-service"));
        final Timer.Context context = timer.time();
        try {
            DtoResponse response = new DtoResponse();
            try {
                response.setMessage(message);
                response.setTime(DateTime.now());
            } catch (Exception e) {
                log.error("internal error: {}", e);
                return Response.status(Response.Status.INTERNAL_SERVER_ERROR).build();
            }
            return Response.ok(response).build();
        } finally {
            if (context != null) context.stop();
        }
    }
}
```

Reste maintenant à ajouter Swagger à notre `main()` que l'on doit faire programmatiquement faute d'être dans un conteneur de Servlet standard...

Pour ce faire, il est nécessaire d'instancier un objet `BeanConfig` qui contient la configuration de Swagger mais surtout l'adresse et le port du serveur sur lequel tourne le service ainsi que le package où se trouve ce dernier. Ces informations sont renseignées, dans notre cas, dans notre fichier de configuration et positionnées programmatiquement dans notre `BeanConfig`.

Enfin, il faut trouver le moyen de faire le pendant de ce qui est déclaré sur [cette page](https://github.com/wordnik/swagger-core/wiki/Servlet-Quickstart)... bien sûr, le tout sans Servlet... ouch... :'( En fouillant un peu, on tombe rapidement sur le [quickstart swagger/cxf](https://github.com/wordnik/swagger-core/wiki/Java-CXF-Quickstart) où les _providers_ sont positionnés : il suffit de faire pareil avec Resteasy-Netty ;-)

```java
private static void initSwagger(ResteasyDeployment deployment) {
    BeanConfig swaggerConfig = new BeanConfig();
    swaggerConfig.setVersion(config.getString("swagger.version", "1.0.0"));
    swaggerConfig.setBasePath("http://" + config.getString("swagger.host", "localhost") + ":" + config.getString("swagger.port", "8081"));
    swaggerConfig.setTitle(config.getString("swagger.title", "jetoile sample app"));
    swaggerConfig.setScan(true);
    swaggerConfig.setResourcePackage("fr.jetoile.sample.service");

    deployment.setProviderClasses(Lists.newArrayList("fr.jetoile.sample.JacksonConfig",
            "com.wordnik.swagger.jaxrs.listing.ResourceListingProvider",
            "com.wordnik.swagger.jaxrs.listing.ApiDeclarationProvider"));
    deployment.setResourceClasses(Lists.newArrayList("com.wordnik.swagger.jaxrs.listing.ApiListingResourceJSON"));
    deployment.setSecurityEnabled(false);
}
```

Et voilà, ça fonctionne!

En exécutant la commande :
```bash
curl -XGET http://localhost:8081/api-docs/sample
```

On obtient bien le JSON escompté :
```javascript
{
    "apiVersion": "1.0.0",
    "swaggerVersion": "1.2",
    "basePath": "http://localhost:8081",
    "resourcePath": "/sample",
    "apis": [
        {
            "path": "/sample/say/{msg}",
            "operations": [
                {
                    "method": "GET",
                    "summary": "repeat the word",
                    "notes": "response the word",
                    "type": "DtoResponse",
                    "nickname": "getPortDataSet",
                    "produces": [
                        "application/json"
                    ],
                    "parameters": [
                        {
                            "name": "msg",
                            "required": true,
                            "allowMultiple": false,
                            "type": "string",
                            "paramType": "path"
                        }
                    ],
                    "responseMessages": [
                        {
                            "code": 500,
                            "message": "Internal server error"
                        }
                    ]
                }
            ]
        }
    ],
    "models": {
        "DtoResponse": {
            "id": "DtoResponse",
            "properties": {
                "message": {
                    "type": "string"
                },
                "time": {
                    "$ref": "DateTime"
                }
            }
        }
    }
}
```

Mais (car il y a un mais...) en utilisant [Swagger-UI](https://github.com/wordnik/swagger-ui) (qu'il faut déployer sur un apache/nginx/tomcat ou autre), il peut arriver que cela ne fonctionne pas... )(ie. que Swagger-IU n'arrive pas à fetcher les ressources de notre service REST). Cela arrivera d'ailleurs sûrement si notre application est déployée sur une machine différente de celle où est déployée Swagger-UI (pour rappel, on ne dispose pas, ici, d'un conteneur de Servlet et exposer des pages statiques n'est pas l'objectif de notre petite application). Le problème vient de notre cher ami, le [CORS](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing)... Du coup, il devient nécessaire d'ajouter des _headers_ dans le requête de réponse.

Et c'est là que la tâche se gâte... En effet, pas de possibilité de positionner un filtre comme avec les Servlets. Pas non plus de possibilité de modifier la configuration de Resteasy-Netty 3 pour lui demander d'ajouter des headers (si cela existe, je n'ai pas trouvé)...

Du coup, la seule solution a été de patcher sauvagement notre ami Resteasy-Netty 3 en surchargeant une de ses classes pour y ajouter les bons headers... Pas très classe mais bon...

Pour ce faire, il suffit de créer dans notre application le package `org.jboss.resteasy.plugins.server.netty` et d'y copier la classe `RequestHandler` en y ajoutant les headers utiles :
```java
package org.jboss.resteasy.plugins.server.netty;

import org.jboss.netty.channel.*;
import org.jboss.netty.channel.ChannelHandler.Sharable;
import org.jboss.netty.handler.codec.frame.TooLongFrameException;
import org.jboss.netty.handler.codec.http.DefaultHttpResponse;
import org.jboss.netty.handler.codec.http.HttpResponse;
import org.jboss.netty.handler.codec.http.HttpResponseStatus;
import org.jboss.resteasy.logging.Logger;
import org.jboss.resteasy.spi.Failure;

import static org.jboss.netty.handler.codec.http.HttpResponseStatus.CONTINUE;
import static org.jboss.netty.handler.codec.http.HttpVersion.HTTP_1_1;

/**
 * TODO : hack to add CORS into header
 *
 * {@link org.jboss.netty.channel.SimpleChannelUpstreamHandler} which handles the requests and dispatch them.
 *
 * This class is {@link org.jboss.netty.channel.ChannelHandler.Sharable}.
 *
 * @author The Netty Project
 * @author Andy Taylor (andy.taylor@jboss.org)
 * @author Trustin Lee
 * @author Norman Maurer
 * @version $Rev: 2368 $, $Date: 2010-10-18 17:19:03 +0900 (Mon, 18 Oct 2010) $
 */
@Sharable
public class RequestHandler extends SimpleChannelUpstreamHandler {
    protected final RequestDispatcher dispatcher;
    private final static Logger logger = Logger.getLogger(org.jboss.resteasy.plugins.server.netty.RequestHandler.class);

    public RequestHandler(RequestDispatcher dispatcher) { this.dispatcher = dispatcher; }

    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        if (e.getMessage() instanceof NettyHttpRequest) {
            NettyHttpRequest request = (NettyHttpRequest) e.getMessage();

            //HACK ICI!!!
            request.getResponse().getOutputHeaders().add("Access-Control-Allow-Origin", "*");
            request.getResponse().getOutputHeaders().add("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS");
            request.getResponse().getOutputHeaders().add("Access-Control-Allow-Headers", "X-Requested-With, Content-Type, Content-Length");
            //FIN DU HACK

            if (request.is100ContinueExpected()) { send100Continue(e); }

            NettyHttpResponse response = request.getResponse();
            try {
                dispatcher.service(request, response, true);
            } catch (Failure e1) {
                response.reset();
                response.setStatus(e1.getErrorCode());
                return;
            } catch (Exception ex) {
                response.reset();
                response.setStatus(500);
                logger.error("Unexpected", ex);
                return;
            }

            // Write the response.
            ChannelFuture future = e.getChannel().write(response);

            // Close the non-keep-alive connection after the write operation is done.
            if (!request.isKeepAlive()) { future.addListener(ChannelFutureListener.CLOSE); }
        }
    }

    private void send100Continue(MessageEvent e) {
        HttpResponse response = new DefaultHttpResponse(HTTP_1_1, CONTINUE);
        e.getChannel().write(response);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e) throws Exception {
        // handle the case of to big requests.
        if (e.getCause() instanceof TooLongFrameException) {
            DefaultHttpResponse response = new DefaultHttpResponse(HTTP_1_1, HttpResponseStatus.REQUEST_ENTITY_TOO_LARGE);
            e.getChannel().write(response).addListener(ChannelFutureListener.CLOSE);
        } else {
            e.getCause().printStackTrace();
            e.getChannel().close();
        }
    }
}
```

Voilà, après ce petit tour de passe passe, notre swagger-UI fonctionne comme un charme ;-)

Au final, (presque?) simple non? ;-)

## Branchement des plugins Maven Appassembler et Assembly

Afin de générer une application utilisable _out of the box_, le plugin maven [appassembler](http://mojo.codehaus.org/appassembler/appassembler-maven-plugin/) a été utilisé. Pour ceux qui ne saurait pas ce que c'est, je les invite à regarder soit la documentation officielle soit un article que j'avais fait [précédemment](/2012/02/petit-focus-sur-2-plugins-maven.html) (#autopromo ;-) ).

Ainsi, ici, le goal `generate-daemons` du plugin a été utilisé : 
```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>appassembler-maven-plugin</artifactId>
    <executions>

        <execution>
            <id>spring-integ-reader</id>
            <phase>package</phase>
            <goals>
                <goal>generate-daemons</goal>
            </goals>
            <configuration>
                <target>${project.build.directory}/appassembler-jsw</target>

                <repositoryLayout>flat</repositoryLayout>

                <daemons>
                    <daemon>
                        <id>${project.name}</id>
                        <mainClass>fr.jetoile.sample.Client</mainClass>
                        <commandLineArguments>
                        </commandLineArguments>
                        <platforms>
                            <platform>jsw</platform>
                        </platforms>
                        <generatorConfigurations>
                            <generatorConfiguration>
                                <generator>jsw</generator>
                                <includes>
                                    <include>linux-x86-64</include>
                                    <include>linux-x86-32</include>
                                </includes>
                                <configuration>

                                    <property>
                                        <name>configuration.directory.in.classpath.first</name>
                                        <value>conf</value>
                                    </property>

                                </configuration>
                            </generatorConfiguration>
                        </generatorConfigurations>
                        <jvmSettings>
                            <initialMemorySize>256M</initialMemorySize>
                            <maxMemorySize>2048M</maxMemorySize>
                            <systemProperties>
                                <systemProperty>com.sun.management.jmxremote</systemProperty>
                                <systemProperty>com.sun.management.jmxremote.port=8199</systemProperty>
                                <systemProperty>com.sun.management.jmxremote.authenticate=false
                                </systemProperty>
                                <systemProperty>com.sun.management.jmxremote.ssl=false</systemProperty>
                                <systemProperty>com.sun.management.jmxremote.local.only=false
                                </systemProperty>
                            </systemProperties>
                            <extraArguments>
                                <extraArgument>-Xdebug</extraArgument>
                                <extraArgument>
                                    -Xrunjdwp:transport=dt_socket,address=9999,server=y,suspend=n
                                </extraArgument>
                                <extraArgument>-server</extraArgument>
                                <extraArgument>-XX:+UnlockCommercialFeatures</extraArgument>
                                <extraArgument>-XX:+FlightRecorder</extraArgument>
                                <extraArgument>-XX:+HeapDumpOnOutOfMemoryError</extraArgument>
                            </extraArguments>
                        </jvmSettings>
                    </daemon>
                </daemons>
            </configuration>
        </execution>
    </executions>
    <configuration>

    </configuration>
</plugin>
```

En outre, ce plugin ne créant pas le répertoire `logs` et ne positionnant pas les droits d'exécution sur les fichiers du répertoire bin, le plugin Maven [assembly](https://maven.apache.org/plugins/maven-assembly-plugin/) a été utilisé conjointement :
```xml
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <descriptors>
            <descriptor>src/main/assembly/descriptor.xml</descriptor>
        </descriptors>
        <appendAssemblyId>false</appendAssemblyId>

    </configuration>
    <executions>
        <execution>
            <id>assembly</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Avec le descripteur simple suivant : 
```xml
<?xml version="1.0"?>
<assembly xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2 http://maven.apache.org/xsd/assembly-1.1.2.xsd">
    <id>reader</id>
    <includeBaseDirectory>false</includeBaseDirectory>
    <formats><format>tar.gz</format></formats>

    <fileSets>
        <fileSet>
            <directory>${project.build.directory}/appassembler-jsw/jsw/${project.name}</directory>
            <outputDirectory>/</outputDirectory>
            <excludes>
                <exclude>bin/${project.name}</exclude>
                <exclude>bin/wrapper-linux-x86-32</exclude>
                <exclude>bin/wrapper-linux-x86-64</exclude>
            </excludes>
            <fileMode>640</fileMode>
            <directoryMode>750</directoryMode>
        </fileSet>
        <fileSet>
            <directory>src/main/assembly</directory>
            <outputDirectory>/logs</outputDirectory>
            <excludes><exclude>*</exclude></excludes>
        </fileSet>
    </fileSets>

    <files>
        <file>
            <source>${project.build.directory}/appassembler-jsw/jsw/${project.name}/bin/${project.name}</source>
            <outputDirectory>bin</outputDirectory>
            <fileMode>750</fileMode>
        </file>
        <file>
            <source>${project.build.directory}/appassembler-jsw/jsw/${project.name}/bin/wrapper-linux-x86-64</source>
            <outputDirectory>bin</outputDirectory>
            <fileMode>750</fileMode>
        </file>
    </files>
</assembly>
```

Ainsi, l'exécution de la commande suivante : 
```bash
mvn package
```

génère un livrable exploitable directement après sa décompression.

Lors d'un `mvn release`, il sera également automatiquement uploadé sur le _Repository Manager_.

# Conclusion

En conclusion, je n'ai pas grand chose à ajouter si ce n'est que j'ai trouvé Resteasy-Netty simple à utiliser et qu'il a été aisé d'y ajouter tout ce qui était nécessaire à notre besoin.

Et le tout de manière simple et efficace pour une solution véloce et légère!

Pour faire encore plus simple, [Lombok](http://projectlombok.org/) aurait pu être utilisé mais, de mémoire, en test de Java 8, une incompatibilité est apparue... à creuser donc pour cette partie... ;-)


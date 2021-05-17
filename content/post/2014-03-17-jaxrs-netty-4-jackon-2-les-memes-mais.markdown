---
title: "JAXRS, Netty 4, Jackon 2... les mêmes mais en mieux..."
date: 2014-03-17 08:43:12 +0100
comments: true
tags: 
- netty
- java
- jmx
- rest
- performance
- jolokia
---

![left-small](http://3.bp.blogspot.com/-Vzol1CndcjY/Uxib17Rlf2I/AAAAAAAABQI/Qi4u0DWe2s4/s1600/resteasy_jolokia_metrics.png)

Pour faire suite à mon [article précédent](/2014/03/jaxrs-netty-et-bien-plus-encore-mode.html) qui montrait comment il était possible de construire une _stack_ légère basée sur Resteasy-Netty3, Jackson, [Jolokia](http://www.jolokia.org/) et [Swagger](https://helloreverb.com/developers/swagger), cet article montrera comment il est possible de faire la même chose avec Resteasy-Netty4 et Jackson 2.

Même si les changements ne sont pas énormes, il y a quand même quelques variantes, et, histoire d'être exhaustif, cela permet de faire le tour complet... ;-)

En fait, les seuls points qui diffèrent, par rapport au code précédent, touchent :

* les dépendances,
* l'intégration de Resteasy-netty4,
* l'intégration du JacksonConfig (changement d'API coté Jackson),
* le support de JodaTime dans Jackson 2,
* et le support du CORS dans Resteasy-Netty4.

C'est donc ces différents points qui seront abordés dans cet article.

Le code se trouve sur github sur la branche [netty4](https://github.com/jetoile/resteasy-netty-sample/tree/netty4).

<!-- more -->

#Les Dépendances

Les dépendances utilisées sont les suivantes (au format gradle) :
```text
compile group: 'io.netty', name: 'netty-all', version:'4.0.17.Final'
compile group: 'org.jboss.resteasy', name: 'jaxrs-api', version:'3.0.4.Final'
compile group: 'org.jolokia', name: 'jolokia-jvm', version:'1.1.2'
compile group: 'com.wordnik', name: 'swagger-jaxrs_2.10', version:'1.3.0'
compile group: 'com.wordnik', name: 'swagger-annotations_2.10', version:'1.3.0'
compile group: 'javax.servlet', name: 'servlet-api', version:'2.5'
compile group: 'com.codahale.metrics', name: 'metrics-core', version:'3.0.1'
compile group: 'org.jboss.resteasy', name: 'resteasy-netty4', version:'3.0.6.Final'
compile group: 'org.jboss.resteasy', name: 'resteasy-jackson2-provider', version:'3.0.6.Final'
compile group: 'com.fasterxml.jackson.datatype', name: 'jackson-datatype-joda', version:'2.3.2'
compile group: 'commons-configuration', name: 'commons-configuration', version:'1.9'
compile group: 'commons-collections', name: 'commons-collections', version:'3.2.1'
compile group: 'commons-io', name: 'commons-io', version:'2.4'
compile group: 'joda-time', name: 'joda-time', version:'2.3'
compile(group: 'com.google.guava', name: 'guava', version:'15.0') {exclude(module: 'jsr305')}
```

On peut y constater depuis la version précédente que netty est passé en version __4.0.17.Final__ mais également c'est maintenant l'artefact __resteasy-netty4__ qui est utilisé plutôt que __resteasy-netty__. De la même manière, c'est maintenant l'artefact __resteasy-jackson2-provider__ plutôt que __resteasy-jackson-provider__.

En outre l'artefact __jackson-datatype-joda__ a été ajouté (nous y reviendrons ultérieurement).

#Intégration de Resteasy-netty4

Afin de remplacer Resteasy-netty 3 par Resteasy-Netty4, il suffit de modifier les dépendances et de supprimer le hack fait précédemment concernant le CORS (ie. la classe `RequestHandler`) qui est incompatible avec cette nouvelle version.

Une fois cela fait, le programme devrait être de nouveau fonctionnel sans avoir à modifier quoique ce soit (modulo Swagger-UI mais nous y reviendrons ultérieurement.)

#Intégration de Jackson 2

Comme il a été vu précédemment, c'est maintenant la version de Jackson 2 qui est utilisé plutôt que la 1.

Aussi, il est nécessaire de modifier les packages de Jackson importés : cela n'est à faire que dans la classe `JacksonConfig`.

Certaines des API ayant également évoluées, la classe `JacksonConfig` devient :
```java
public class JacksonConfig implements ContextResolver<ObjectMapper> {
    private final ObjectMapper objectMapper;

    public JacksonConfig() {
        objectMapper = new ObjectMapper();
        objectMapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
    }

    @Override
    public ObjectMapper getContext(Class<?> objectType) {
        return objectMapper;
    }
}
```

#Support de JodaTime dans Jackson 2

On a vu dans le paragraphe précédent comment il fallait modifier notre code pour utiliser Jackson 2 à la place de Jackson 1.

Cependant, lors d'un :
```bash
curl -XGET http://localhost:8081/sample/say/hello
```

On obtient : 
```javascript
{
    "message": "hello",
    "time": {
        "era": 1,
        "dayOfMonth": 6,
        "dayOfWeek": 4,
        "dayOfYear": 65,
        "weekyear": 2014,
        "weekOfWeekyear": 10,
        "monthOfYear": 3,
        "yearOfEra": 2014,
        "yearOfCentury": 14,
        "centuryOfEra": 20,
        "millisOfSecond": 173,
        "millisOfDay": 53370173,
        "secondOfMinute": 30,
        "secondOfDay": 53370,
        "minuteOfHour": 49,
        "minuteOfDay": 889,
        "hourOfDay": 14,
        "year": 2014,
        "zone": {
            "fixed": false,
            "uncachedZone": {
                "cachable": true,
                "fixed": false,
                "id": "Europe/Paris"
            },
            "id": "Europe/Paris"
        },
        "millis": 1394113770173,
        "chronology": {
            "zone": {
                "fixed": false,
                "uncachedZone": {
                    "cachable": true,
                    "fixed": false,
                    "id": "Europe/Paris"
                },
                "id": "Europe/Paris"
            }
        },
        "afterNow": false,
        "beforeNow": false,
        "equalNow": true
    }
}
```

Pour corriger cela, il suffit d'importer la dépendance __'com.fasterxml.jackson.datatype', name: 'jackson-datatype-joda'__ et d'ajouter à l'`objectMapper` le module `JodaModule` :
```java
objectMapper = new ObjectMapper();
objectMapper.registerModule(new JodaModule());
objectMapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);  
```

Ainsi, on obtient bien :
```javascript
{
    "message": "hello",
    "time": "2014-03-06T13:53:38.714Z"
}
```

#Support du CORS dans Resteasy-Netty4

Précédemment, avec Resteast-netty 3, nous avions remarqué un problème de CORS avec Swagger-UI.
Pour en venir à bout, un _hack_ avait été fait mais ce n'était pas très propre...

Malheureusement, Resteasy-netty4 n'offre pas, non plus, de manière simple pour surmonter ce problème. Heureusement, en fouillant un peu sur internet, un [article](http://stackoverflow.com/questions/18857546/implement-cross-origin-resource-sharing-cors-on-resteasy-netty-server) propose de rajouter un `ChannelInboundHandler` au _pipeline_ Netty.

Cependant, je n'ai pas trouvé de moyen simple de le faire mis à part la surcharge de la méthode...

Le code obtenu est donc le suivant :

La classe `ChannedInboundHandler` :
```java
public class CorsHeadersChannelHandler extends SimpleChannelInboundHandler<NettyHttpRequest> {
    protected void channelRead0(ChannelHandlerContext ctx, NettyHttpRequest request) throws Exception {
        request.getResponse().getOutputHeaders().add("Access-Control-Allow-Origin", "*");
        request.getResponse().getOutputHeaders().add("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS");
        request.getResponse().getOutputHeaders().add("Access-Control-Allow-Headers", "X-Requested-With, Content-Type, Content-Length");

        ctx.fireChannelRead(request);
    }
}
```

La surcharge de la méthode `start()` pour ajouter le _handler_ au pipeline Netty (désolé pour le nom...) :

```java
public class MyNettyJaxrsServer extends NettyJaxrsServer {
    private EventLoopGroup eventLoopGroup;
    private EventLoopGroup eventExecutor;
    private int ioWorkerCount = Runtime.getRuntime().availableProcessors() * 2;
    private int executorThreadCount = 16;
    private SSLContext sslContext;
    private int maxRequestSize = 1024 * 1024 * 10;
    private int backlog = 128;

    @Override
    public void setSSLContext(SSLContext sslContext) { this.sslContext = sslContext; }

    @Override
    public void setIoWorkerCount(int ioWorkerCount) { this.ioWorkerCount = ioWorkerCount; }

    @Override
    public void setExecutorThreadCount(int executorThreadCount) { this.executorThreadCount =  executorThreadCount; }

    @Override
    public void setMaxRequestSize(int maxRequestSize) { this.maxRequestSize  = maxRequestSize; }

    public void setBacklog(int backlog) { this.backlog = backlog; }

    @Override
    public void start() {
        eventLoopGroup = new NioEventLoopGroup(ioWorkerCount);
        eventExecutor = new NioEventLoopGroup(executorThreadCount);
        deployment.start();
        final RequestDispatcher dispatcher = new RequestDispatcher((SynchronousDispatcher)deployment.getDispatcher(), deployment.getProviderFactory(), domain);
        // Configure the server.
        if (sslContext == null) {
            bootstrap.group(eventLoopGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new HttpRequestDecoder());
                            ch.pipeline().addLast(new HttpObjectAggregator(maxRequestSize));
                            ch.pipeline().addLast(new HttpResponseEncoder());
                            ch.pipeline().addLast(new RestEasyHttpRequestDecoder(dispatcher.getDispatcher(), root, RestEasyHttpRequestDecoder.Protocol.HTTP));
                            ch.pipeline().addLast(new CorsHeadersChannelHandler());
                            ch.pipeline().addLast(new RestEasyHttpResponseEncoder(dispatcher));
                            ch.pipeline().addLast(eventExecutor, new RequestHandler(dispatcher));
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, backlog)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);
        } else {
            final SSLEngine engine = sslContext.createSSLEngine();
            engine.setUseClientMode(false);
            bootstrap.group(eventLoopGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addFirst(new SslHandler(engine));
                            ch.pipeline().addLast(new HttpRequestDecoder());
                            ch.pipeline().addLast(new HttpObjectAggregator(maxRequestSize));
                            ch.pipeline().addLast(new HttpResponseEncoder());
                            ch.pipeline().addLast(new RestEasyHttpRequestDecoder(dispatcher.getDispatcher(), root, RestEasyHttpRequestDecoder.Protocol.HTTPS));
                            ch.pipeline().addLast(new CorsHeadersChannelHandler());
                            ch.pipeline().addLast(new RestEasyHttpResponseEncoder(dispatcher));
                            ch.pipeline().addLast(eventExecutor, new RequestHandler(dispatcher));
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, backlog)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);
        }
        bootstrap.bind(port).syncUninterruptibly();
    }
}
```

On y observe le rajout du _handler_ : 
```java
ch.pipeline().addLast(new CorsHeadersChannelHandler());
```

Enfin, l'initialisation du serveur Resteasy-netty : 
```java
MyNettyJaxrsServer netty = new MyNettyJaxrsServer();
```

#Conclusion

On a vu, dans cet article, comment il était possible d'intégrer JAX-RS avec Netty 4 à l'aide de Resteasy tout en ayant une intégration de Jackson 2.

On a également montré qu'il était possible d'y intégrer très simplement Swagger et Jolokia.
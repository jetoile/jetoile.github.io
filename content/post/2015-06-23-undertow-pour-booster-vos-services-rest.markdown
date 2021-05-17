---
title: "Undertow pour booster vos services REST"
date: 2015-06-23 14:51:56 +0200
comments: true
tags: 
- rest
- java
- microservice
- jmx
- jolokia
- maven
- metrics
- performance
---
![left-small](/images/undertow.png)
Il y a quelques temps, j'avais fait une série d'articles sur [resteasy-netty](http://blog.jetoile.fr/2014/03/jaxrs-netty-et-bien-plus-encore-mode.html) et [resteasy-netty4](http://blog.jetoile.fr/2014/03/jaxrs-netty-4-jackon-2-les-memes-mais.html).

Cette article repart du même besoin, à savoir disposer d'une _stack_ légère pour réaliser un service REST, mais en utilisant [Undertow](http://undertow.io/) plutôt que Resteasy-Netty.

<!-- more -->

Au niveau des besoins, ils seront identiques ie. :

* utiliser JAX-RS,
* intégrer Swagger,
* intégrer Jolokia,
* générer un livrable autoporteur.

RestEasy-Netty, même s'il existe de nombreux points d'entrée, demande quelques phases de _hack_ (gestion du _crossover domain_ par exemple) et dispose d'un mécanisme un peu limité concernant la partie sécurité.

En outre, l'absence du mécanisme de Servlet reste un peu embêtant pour mettre en place certaines _features_ comme le MDC ( [_Mapped Diagnostic Context_](http://logback.qos.ch/manual/mdc.html) ) bien pratique lorsque l'on est dans une architecture type microservice.

Le code complet est disponible [ici](https://github.com/jetoile/undertow-sample).

#Rappel du cahier des charges
Comme je l'ai déjà indiqué dans les autres posts, l'objectif est seulement de montrer comme il peut être simple d'exposer un service REST à l'aide d'[Undertow](http://undertow.io/). Pour ce faire, un simple service sera exposé et il consistera à répèter ce qu’on lui demande…

Il répondra donc à une requête de type GET du type : http://localhost:8081/sample/say/<message>

Du coté de la réponse, elle aura la forme suivante :
```javascript
{
    "message": <message>,
    "time":"2015-06-23T15:18:50.748"
}
```

#Mise en oeuvre
A titre informatif, les versions des différentes librairies qui sont utilisés dans les exemples de code ci-dessous sont les suivantes (au format gradle pour gagner de la place) :
```text
    compile group: 'org.jboss.resteasy', name: 'jaxrs-api', version:'3.0.11.Final'
    compile group: 'org.jolokia', name: 'jolokia-jvm', version:'1.3.1'
    compile group: 'com.wordnik', name: 'swagger-jaxrs_2.10', version:'1.3.12'
    compile group: 'com.wordnik', name: 'swagger-annotations_2.10', version:'1.3.0'
    compile group: 'javax.servlet', name: 'javax.servlet-api', version:'3.1.0'
    compile group: 'io.dropwizard.metrics', name: 'metrics-core', version:'3.1.2'
    compile group: 'io.undertow', name: 'undertow-core', version:'1.2.8.Final'
    compile group: 'io.undertow', name: 'undertow-servlet', version:'1.2.8.Final'
    compile group: 'org.jboss.resteasy', name: 'resteasy-undertow', version:'3.0.11.Final'
    compile group: 'org.jboss.resteasy', name: 'resteasy-jackson2-provider', version:'3.0.11.Final'
    compile group: 'com.fasterxml.jackson.core', name: 'jackson-core', version:'2.5.4'
    compile group: 'com.fasterxml.jackson.core', name: 'jackson-annotations', version:'2.5.4'
    compile group: 'com.fasterxml.jackson.core', name: 'jackson-databind', version:'2.5.4'
    compile group: 'commons-configuration', name: 'commons-configuration', version:'1.10'
    compile group: 'commons-collections', name: 'commons-collections', version:'3.2.1'
    compile group: 'commons-io', name: 'commons-io', version:'2.4'
    compile group: 'org.slf4j', name: 'slf4j-api', version:'1.7.12'
    compile group: 'ch.qos.logback', name: 'logback-classic', version:'1.1.3'
```

Concernant la version des différentes dépendances, on constate que ce n'est pas swagger2 qui est utilisé en raison d'une incapacité de ma part à l'intégrer... :'(

#Implémentation du service REST
Le mise en place du service REST basé sur JAX-RS est on ne peut plus trivial… et la classe ci-dessous fait humblement l’affaire :
```java
@Api(value = "/sample",
        description = "the sample api")
@Path("/sample")
@RolesAllowed("admin")
public class SimpleService {
    private final static Logger log = LoggerFactory.getLogger(SimpleService.class);


    @GET
    @Path("/say/{msg}")
    @Produces(MediaType.APPLICATION_JSON)
    @ApiOperation(value = "repeat the word",
            notes = "response the word",
            response = DtoResponse.class)
    @ApiResponses(value = {@ApiResponse(code = 500, message = "Internal server error")})
    public Response sayHello(@PathParam("msg") String message) {

        log.info("sample log");

        final Timer timer = Main.metricRegistry.timer(name(SimpleService.class, "say-service"));
        final Timer.Context context = timer.time();
        try {

            DtoResponse response = new DtoResponse();
            try {
                response.setMessage(message);
                response.setTime(LocalDateTime.now());
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
Coté du DTO, il est le suivant :
```java
@XmlRootElement
public class DtoResponse {

    private String message;
    private LocalDateTime time;

    public DtoResponse() {
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public LocalDateTime getTime() {
        return time;
    }

    public void setTime(LocalDateTime time) {
        this.time = time;
    }
}
```

On remarquera l'utilisation de Java8 pour la gestion du temps plutôt que Joda-Time.

En outre, concernant les annotations Swagger et l'utilisation de metrics, nous y reviendrons plus tard.

Concernant le message de log, de même, nous y reviendrons plus tard avec l'intégration d'un MDC pour les logs.

#Mise en oeuvre avec Undertow

Mettre en place Resteasy avec Undertow est très simple, d’après la documnentation, il suffit de faire :
```java
SimpleService simpleService = new SimpleService();
ResteasyDeployment deployment = new ResteasyDeployment();

deployment.setResources(Arrays.<Object>asList(simpleService));

int port = config.getInt("undertow.port", TestPortProvider.getPort());
String host = config.getString("undertow.host", String.valueOf(TestPortProvider.getHost()));
System.setProperty("org.jboss.resteasy.port", String.valueOf(TestPortProvider.getPort());
System.setProperty("org.jboss.resteasy.host", String.valueOf(TestPortProvider.getHost());

UndertowJaxrsServer server = new UndertowJaxrsServer();

DeploymentInfo deploymentInfo = server.undertowDeployment(deployment);
deploymentInfo.setDeploymentName("");
deploymentInfo.setContextPath("/");
deploymentInfo.setClassLoader(Main.class.getClassLoader());

deployment.setProviderFactory(new ResteasyProviderFactory());
server.deploy(deploymentInfo);
server.start(Undertow.builder().addHttpListener(port, host));
```

On y constate que pour ajouter un service, il suffit juste de déclarer la classe implémentant JAX-RS via la méthode ```setResources()``` sur l’instance de _ResteasyDeployment_ fournit au serveur _UndertowJaxrsServer_ :

Et voilà! On dispose désormais d’un programme exécutable qui démarre un serveur REST basé sur Undertow.

Par contre, il semble que le service ne rende pas vraiment ce que l'on voulait :
```bash
curl 'http://localhost:8081/sample/say/<message>'
```

```json
{
    "message": "<message>",
    "time": {
        "hour": 15,
        "minute": 55,
        "second": 51,
        "nano": 225000000,
        "year": 2015,
        "month": "JUNE",
        "dayOfMonth": 23,
        "dayOfWeek": "TUESDAY",
        "dayOfYear": 174,
        "monthValue": 6,
        "chronology": {
            "calendarType": "iso8601",
            "id": "ISO"
        }
    }
}
```

Pas de souci, il suffit de préciser comment on souhaite que LocalDateTime soit sérialisé par Jackson :

Ainsi, notre DTO devient :
```java

@XmlRootElement
public class DtoResponse {

    private String message;
    @JsonSerialize(using = LocalDateTimeToStringSerializer.class)
    private LocalDateTime time;

    public DtoResponse() {
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public LocalDateTime getTime() {
        return time;
    }

    public void setTime(LocalDateTime time) {
        this.time = time;
    }
}
```

où :

```java
public class LocalDateTimeToStringSerializer extends JsonSerializer<LocalDateTime> {
    @Override
    public void serialize(LocalDateTime value, JsonGenerator jgen, SerializerProvider provider) throws IOException, JsonProcessingException {
        jgen.writeObject(value.format(DateTimeFormatter.ISO_DATE_TIME));
    }
}
```

Après ces modifications, on obtient bien :

```json
{"message":"<message>","time":"2015-06-23T16:04:01.419"}
```

#Intégration de Metrics

Concernant l'intégration de Metrics, pas grand chose de nouveau et donc pas grand chose à dire ;-)

Déclarer le registry :
```java
metricRegistry = new MetricRegistry();
final JmxReporter reporter = JmxReporter.forRegistry(metricRegistry).build();
reporter.start();
```

Et utiliser le dans vos classes :
```java
final Timer timer = Main.metricRegistry.timer(name(SimpleService.class, "say-service"));
final Timer.Context context = timer.time();
try {
    ...
} finally {
	if (context != null) context.stop();
}
```


#Intégration de la sécurité
Undertow permet une bien meilleur intégration de la sécurité que RestEasy-Netty. En effet, grâce au mécanisme de Servlet, il est possible de bénéficier de toute la puissance des conteneurs de Servlets.

Du coté du serveur Undertow, il suffit donc de définir un _ServletIdentityManager_ et de lui fournir un _LoginConfig_ :
```java
deployment.setSecurityEnabled(true);

ServletIdentityManager identityManager = new ServletIdentityManager();
identityManager.addUser("khanh", "khanh", "admin");

deploymentInfo = deploymentInfo.setIdentityManager(identityManager).setLoginConfig(new LoginConfig("BASIC", "Test Realm")); 
```

où :
```java
public class ServletIdentityManager implements IdentityManager {

    private static final Charset UTF_8 = Charset.forName("UTF-8");
    private final Map<String, UserAccount> users = new HashMap<>();

    public void addUser(final String name, final String password, final String... roles) {
        UserAccount user = new UserAccount();
        user.name = name;
        user.password = password.toCharArray();
        user.roles = new HashSet<>(Arrays.asList(roles));
        users.put(name, user);
    }

    @Override
    public Account verify(Account account) {
        // Just re-use the existing account.
        return account;
    }

    @Override
    public Account verify(String id, Credential credential) {
        Account account = users.get(id);
        if (account != null && verifyCredential(account, credential)) {
            return account;
        }

        return null;
    }

    @Override
    public Account verify(Credential credential) {
        return null;
    }

    private boolean verifyCredential(Account account, Credential credential) {
        // This approach should never be copied in a realm IdentityManager.
        if (account instanceof UserAccount) {
            if (credential instanceof PasswordCredential) {
                char[] expectedPassword = ((UserAccount) account).password;
                char[] suppliedPassword = ((PasswordCredential) credential).getPassword();

                return Arrays.equals(expectedPassword, suppliedPassword);
            } else if (credential instanceof DigestCredential) {
                DigestCredential digCred = (DigestCredential) credential;
                MessageDigest digest = null;
                try {
                    digest = digCred.getAlgorithm().getMessageDigest();

                    digest.update(account.getPrincipal().getName().getBytes(UTF_8));
                    digest.update((byte) ':');
                    digest.update(digCred.getRealm().getBytes(UTF_8));
                    digest.update((byte) ':');
                    char[] expectedPassword = ((UserAccount) account).password;
                    digest.update(new String(expectedPassword).getBytes(UTF_8));

                    return digCred.verifyHA1(HexConverter.convertToHexBytes(digest.digest()));
                } catch (NoSuchAlgorithmException e) {
                    throw new IllegalStateException("Unsupported Algorithm", e);
                } finally {
                    digest.reset();
                }
            }
        }
        return false;
    }

    private static class UserAccount implements Account {
        // In no way whatsoever should a class like this be considered a good idea for a real IdentityManager implementation,
        // this is for testing only.

        String name;
        char[] password;
        Set<String> roles;

        private final Principal principal = new Principal() {
            @Override
            public String getName() {
                return name;
            }
        };

        @Override
        public Principal getPrincipal() {
            return principal;
        }

        @Override
        public Set<String> getRoles() {
            return roles;
        }
    }
}
```

Il s'agit ici d'une Basic Authentification mais il est bien sûr possible d'en mettre en place d'autre.

Coté autorisation, il est alors possible de bénéficier de l'annotation ```@RolesAllowed``` de JAX-RS :
```java
@Path("/sample")
@RolesAllowed("admin")
public class SimpleService {
...
}
```

#Intégration d'un MDC
Concernant la mise en place d'un MDC (_Mapped Diagnostic Context_), le fait de bénéficier du mécanisme de _Filter_ des Servlets rend la chose beaucoup plus simple. 

En effet, une fois la couche sécurité branchée, il suffit de récupérer le ```UserPrincipal``` dans la requête et l'enregistrer dans le MDC.

La déclaration des Filters se fait de la manière suivante pour Undertow :
```java
FilterInfo mdcFilter = new FilterInfo("MDCFilter", MDCServletFilter.class);
deploymentInfo.addFilter(mdcFilter);
deploymentInfo.addFilterUrlMapping("MDCFilter", "*", DispatcherType.REQUEST);

FilterInfo mdcInsertingFilter = new FilterInfo("MDCInsertingServletFilter", MDCInsertingServletFilter.class);
deploymentInfo.addFilter(mdcInsertingFilter);
deploymentInfo.addFilterUrlMapping("MDCInsertingServletFilter", "*", DispatcherType.REQUEST);
```

Avec le filter ci-dessous :
```java
public class MDCServletFilter implements Filter {

    private final String USER_KEY = "username";

    public void destroy() {
    }

    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        boolean successfulRegistration = false;

        HttpServletRequest req = (HttpServletRequest) request;
        Principal principal = req.getUserPrincipal();
        // Please note that we could have also used a cookie to
        // retrieve the user name

        if (principal != null) {
            String username = principal.getName();
            successfulRegistration = registerUsername(username);
        }

        try {
            chain.doFilter(request, response);
        } finally {
            if (successfulRegistration) {
                MDC.remove(USER_KEY);
            }
        }
    }

    public void init(FilterConfig arg0) throws ServletException {
    }


    /**
     * Register the user in the MDC under USER_KEY.
     *
     * @param username
     * @return true id the user can be successfully registered
     */
    private boolean registerUsername(String username) {
        if (username != null && username.trim().length() > 0) {
            MDC.put(USER_KEY, username);
            return true;
        }
        return false;
    }
}
```
Ainsi, disposer d'un MDC permet d'ajouter automatiquement des informations dans les logs :
```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} %X{req.remoteHost} %X{req.requestURI} - C:%X{username} - %msg%n

            </pattern>
        </encoder>
    </appender>


    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>

</configuration>
```

On obtient alors bien les logs voulues :
```text
17:15:11.466 INFO  f.j.sample.service.SimpleService 127.0.0.1 /sample/say/<message> - C:khanh - sample log
```

#Intégration de Jolokia
Coté Jolokia, pas grand chose à ajouter par rapport à ma série d'article précédent...
```java
try {
            JolokiaServerConfig config = new JolokiaServerConfig(new HashMap<String, String>());

            JolokiaServer jolokiaServer = new JolokiaServer(config, true);
            jolokiaServer.start();
} catch (Exception e) {
            LOGGER.error("unable to start jolokia server", e);
}
```

#Intégration de Swagger
Concernant l'intégration de Swagger, le fait de disposer des _Filter_ de Servlet permet de n'avoir pas à faire de _hack_ immonde pour gérer le CORS (cf. [article précédent](http://blog.jetoile.fr/2014/03/jaxrs-netty-et-bien-plus-encore-mode.html)) : il suffit de déclarer un _Filter_ dans Undertow qui a, en outre, la chance d'exister :
```java
CorsFilter filter = new CorsFilter();
filter.setAllowedMethods("GET,POST,PUT,DELETE,OPTIONS");
filter.setAllowedHeaders("X-Requested-With, Content-Type, Content-Length, Authorization");
filter.getAllowedOrigins().add("*");

deployment.setProviderFactory(new ResteasyProviderFactory());
deployment.getProviderFactory().register(filter);
```

Concernant la déclaration dans Undertow, pas grand chose à ajouter :
```java

    private static void initSwagger(ResteasyDeployment deployment) {
        BeanConfig swaggerConfig = new BeanConfig();
        swaggerConfig.setVersion(config.getString("swagger.version", "1.0.0"));
        swaggerConfig.setBasePath("http://" + config.getString("swagger.host", "localhost") + ":" + config.getString("swagger.port", "8081"));
        swaggerConfig.setTitle(config.getString("swagger.title", "jetoile sample app"));
        swaggerConfig.setScan(true);
        swaggerConfig.setResourcePackage("fr.jetoile.sample.service");

        deployment.setProviderClasses(Lists.newArrayList(
                "com.wordnik.swagger.jaxrs.listing.ResourceListingProvider",
                "com.wordnik.swagger.jaxrs.listing.ApiDeclarationProvider"));
        deployment.setResourceClasses(Lists.newArrayList("com.wordnik.swagger.jaxrs.listing.ApiListingResourceJSON"));
        deployment.setSecurityEnabled(false);
    }
```

#Branchement des plugins Maven Appassembler et Assembly

Coté génération du livrable, encore une fois, pas grand chose à ajouter par rapport à mon précédent article : l'utilisation des plugins assembly et appassembler est identique.

#Conclusion

On avait vu dans les articles précédents que RestEasy-Netty était une solution intéressante pour la simplicité de sa mise en oeuvre ainsi que pour le faible overhead.

Cependant, certaines intégrations ressemblaient plus à du _hack_ qu'à une solution configurable.

Undertow (enfin pour être plus précis RestEasy-Undertow) pour sa part offre la même simplicité que RestEasy-Netty mais il permet en plus de s'intégrer avec beaucoup d'autres choses et le fait de retrouver le mécanisme de _Filter_ facilite énormément les choses (par exemple, je ne suis pas sûr que bénéficier du MDC avec RestEasy-Netty ait été aussi simple).

Coté performance, je reviendrai dessus dans un autre article mais je peux déjà dire que la solution RestEasy-Undertow n'a rien à envier à RestEasy-Netty.



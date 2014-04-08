---
layout: post
title: "Lecture et traitement de fichiers : comment faire simple?"
date: 2014-04-02 14:15:44 +0200
comments: true
published: false
categories: 
- java
- spring integration
- eip
---

De nombreuses applications ou systèmes d'information nécessitent le chargement de données issues de fichiers. 

Bien souvent, cet import est exécuté par _batch_, mais il peut aussi être intéressant de faire cet import au fils de l'eau.

En outre, bien souvent, les fichiers à importer sont, soient nombreux, soient volumineux. Du coup, il peut être _touchy_ d'écrire un code simple et fiable. Si, de plus, on veut ajouter des logs parlant (c'est à dire avec, au minimum, le temps de traitement d'un fichier et son nom), cela a tendance a rajouter du bruit au code. Sans oublier que lire un fichier est bien mais que, souvent, un traitement est effectué dessus...

Enfin, lors d'une forte volumétrie, une scalabilité horizontale est souvent intéressante surtout dans le contexte actuelle où la quantité d'information vient à augmenter de plus en plus.

Cet article parlera donc de la problématique d'import de fichiers dans une application en s'appuyant sur des framework comme [Spring Batch](http://projects.spring.io/spring-batch/) ou [Spring Integration](http://projects.spring.io/spring-integration/). Le mot d'ordre sera de le faire le plus simple possible en s'appuyant au maximum sur ces framework.



<!-- more -->
#Solution à base de _batch_

Ecrire un batch permettant de traiter des fichiers peut sembler simple mais lorsque le nombre de ces derniers vient à augmenter ou lorsque la taille des fichiers est volumineux, il arrive souvent que des bugs apparaissent. En outre, il convient alors de gérer manuellement les logs ainsi que la partie supervision. 

Pour répondre à ce besoin, il est peut être avantageux d'utiliser [Spring Batch](http://projects.spring.io/spring-batch/) (ou une autre implémentation de la [JSR 352](https://jcp.org/en/jsr/detail?id=352)). 

_ndlr_ : je ne présenterai pas le fonctionnement de Spring Batch à base de __Job__ et __Step__ puisque cela se trouve très facilement dans les documents officiels, livres ou articles de blog et je mettrai plutôt l'accent sur la faisabilité de tel ou tel chose.

![center](/images/batch/spring-batch-reference-model.png "crédit photo : http://docs.spring.io/spring-batch/trunk/reference/html-single/index.html")

Spring Batch offre nativement la possibilité de traiter les fichiers par _chunk_ via :

* `FlatFileItemReader` qui permet de lire un fichier plat ligne par ligne et où chaque ligne dispose de la même information (il est également possible de traiter des types de lignes différentes issu du même fichier avec `PatternMatchingCompositeLineMapper`).
* `StaxEventItemReader` pour lire fichiers xml composés de format de _fragments_ identiques :
![center](/images/batch/xmlinput.png "crédit photo : http://docs.spring.io/spring-batch/trunk/reference/html-single/index.html")
 
avec :

```xml
<bean id="itemReader" class="org.springframework.batch.item.xml.StaxEventItemReader">
    <property name="fragmentRootElementName" value="trade" />
    <property name="resource" value="data/iosample/input/input.xml" />
    <property name="unmarshaller" ref="tradeMarshaller" />
</bean>

<bean id="tradeMarshaller"
      class="org.springframework.oxm.xstream.XStreamMarshaller">
    <property name="aliases">
        <util:map id="aliases">
            <entry key="trade"
                   value="org.springframework.batch.sample.domain.Trade" />
            <entry key="price" value="java.math.BigDecimal" />
            <entry key="name" value="java.lang.String" />
        </util:map>
    </property>
</bean>
```

Généralement, il est nécessaire de préciser le nom du fichier à traiter mais il est également possible de traiter plusieurs fichiers de même type dans la même _Step_ via la classe `MultiResourceItemReader`.

```xml
<bean id="multiResourceReader" class="org.springframework.batch.item.file.MultiResourceItemReader">
    <property name="resources" value="classpath:data/input/file-*.txt" />
    <property name="delegate" ref="flatFileItemReader" />
</bean>
```

Spring Batch supporte également la scalabilité horizontale en permettant de préciser un `taskExecutor` au niveau de la _Step_.

```xml
<step id="loading">
    <tasklet task-executor="taskExecutor">...</tasklet>
</step>
```

Ainsi, on constate que Spring Batch offre nativement la possibilité de traiter des fichiers volumineux en les découpant par _chunk_. 

De même, il offre nativement la possibilité de passer sur du traitement parallèle.

Concernant la partie supervision, vu que l'on est dans un environnement Spring, on bénéficie, bien sûr, de toute la partie JMX.

Pour la partie gestion des erreurs, Spring Batch permet de les gérer de manière très simple.

Cependant, on perd un grand intérêt si, par fichier, il n'y a qu'une seule donnée. En effet, le mécanisme de _chunk_ devient alors inutile. Il reste cependant possible d'utiliser la scalabilité horizontale.

#Solution à base d'EIP

Dans le cas où la volonté serait de traiter les fichiers au fils de l'eau, Spring Batch n'est pas la solution la plus adaptée... 

Cependant, Spring Integration répond à ce besoin de manière très simple. 

En effet, en utilisant un simple __Service Activator__ (au sens EIP) de type `inbound-channel-adapter`, il devient alors possible de _poller_ un répertoire et d'envoyer le contenu du fichier vers un _filter_ (au sens EIP).

```xml
<file:inbound-channel-adapter id="fileAdapter" auto-startup="true" auto-create-directory="true"
                              filename-pattern="*.xml"
                              directory="file:/tmp"
                              scanner="recursiveScanner"
                              prevent-duplicates="true"
                              channel="inputChannel">
    <int:poller fixed-delay="30000" max-messages-per-poll="500"/>
</file:inbound-channel-adapter>

<file:file-to-string-transformer charset="UTF-8" delete-files="true" input-channel="inputChannel"
                                     output-channel="toLogger"/>

<int:logging-channel-adapter auto-startup="true" channel="toLogger" level="DEBUG" log-full-message="true"/>

<int:channel id="inputChannel"/>
<int:channel id="toLogger"/>
```

D'un point de vue scalabilité horizontale, il suffit de renseigner (tout comme pour Spring Batch) un `taskExecutor` au niveau du _Channel_ et... c'est tout!

```xml
<task:executor id="someExecutor"
                   pool-size="20"
                   keep-alive="2400"/>


<int:channel id="toLogger">
    <int:dispatcher task-executor="someExecutor"/>
</int:channel>
```

Enfin, disposer d'une supervision est on ne peut plus simple puisqu'il suffit de rajouter l'élément `message-history` :

```xml
<int:message-history/>
```

Ainsi, Spring Integration positionne automatiquement dans le _header_ du message le temps d'exécution de chaque _Filter_. Concernant le nom du fichier et son chemin, il se trouve renseigner automatiquement dans le header par l'_adapter_ `file:inbound-channel-adapter`.

Coté gestion des erreurs, Spring Integration permet de les gérer très simplement sur le principe du canal d'erreur qui peut récupérer tous les messages en erreur.

Cependant, avec Spring Integration, si le fichier est volumineux, il n'est plus possible de le traiter en _chunk_ et un risque de contention mémoire existe.

#Conclusion

On a vu dans ce très rapide article comment Spring Integration et Spring Batch pouvait chacun de leurs manières répondre 

#Pour aller plus loin...

* http://projects.spring.io/spring-batch

* http://www.technologies-ebusiness.com/langages/spring-batch-spring-integration-une-usine-de-batchs-a-moindre-cout
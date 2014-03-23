---
layout: post
title: "Devoxx 2011 - What's probably coming in Java Message Service 2.0"
date: 2011-11-20 22:49:24 +0100
comments: true
categories: 
- avis
- devoxx
- java
- jms
---

![left-small](http://4.bp.blogspot.com/-FB-AoRLfOOs/Tsk-J2XdFUI/AAAAAAAAAgE/CpRpQXHqH5M/s1600/devoxx1.jpg
)
La semaine dernière, j'ai eu la chance d'aller à Devoxx en Belgique à Anvers.

Pour ceux, qui ne connaissent pas, je vous invite à aller directement à la pêche aux informations sur le site : http://www.devoxx.com/ et même à vous y inscrire l'année prochaine et/ou même mieux... d'aller assister à [Devoxx France](http://devoxx.fr/)!!

<iframe allowfullscreen="" frameborder="0" height="315" src="http://www.youtube.com/embed/II6XiGGlJX0" width="420"></iframe>

Bon, sinon, pour revenir à nos moutons, cet article est un petit retour de la session de [Nigel Deakin](http://www.devoxx.com/display/DV11/Nigel+Deakin) présentée à [Devoxx 2011](http://www.devoxx.com/display/DV11/Home) et à laquelle j'ai assisté. 

Elle avait pour objectif de montrer l'avancée des travaux sur JMS 2.0 (_Java Message Service_) aussi connu sous le doux nom de [JSR 343](http://jcp.org/en/jsr/detail?id=343). A ce jour, en version early draft, elle devrait être intégrée à JEE7.

Cet article a donc pour vocation de tenter de retranscrire ce que nous a présenté Nigel.
<!-- more -->
Pour la repositionner dans son contexte, la JSR343 fait suite à JMS 1.1 ([JSR 914](http://www.jcp.org/en/jsr/detail?id=914)) et a pour but de :

* Simplifier l'API de JMS :
	* En utilisant CDI (Context Dependency Injection).
	* En clarifiant certaines ambiguïtés présentes dans les spécifications.
* Améliorer l'intégration de JMS avec les serveurs d'application :
	* En intégrant plus facilement les problématiques de PaaS (_Platform As A Service_).
	* En permettant le [multi-tenancy](http://en.wikipedia.org/wiki/Multitenancy).
	* En clarifiant ses relations avec les autres spécifications de JEE7 (ou même ultérieure) : cela est notamment vrai avec la partie MDB (_Message-Driven Beans_) de la spécifications des EJB.
* Ajouter des nouvelles fonctionnalités à l'API.

Je ne reviendrai pas sur ce qu'est JMS (Java Message Service) si ce n'est que sa version actuelle est la 1.1 et qu'elle date de 2003, mais le fait que la spécification JMS n'ait pas évoluée depuis 2003, montre bien qu'elle était solide et qu'elle répondait bien à ce pour quoi elle avait été écrite.

Cependant, autant il n'y a rien à redire quant à la réception d'un message, autant la partie émission était souvent verbeuse puisqu'il était nécessaire pour émettre un message, dans le cas des EJB, de :

* injecter les factory (_ConnectionFactory_ et _Destination_) à l'aide de l'annotation `@Resource`,
* créer la `Connection` à l'aide de la `ConnectionFactory`,
* créer la `Session` à l'aide de la `Connection`,
* créer le `MessageProducer` à l'aide de la `Session`.

C'est seulement suite à toutes ces actions que l'émission d'un message était possible via la méthode send().

En outre, à cela, il fallait, bien sûr, gérer les exceptions _checkées_ (`JMSException` pour ceux à qui cela parle ;-)) mais également la fermeture de la `Session` et de la `Connection`...

Enfin, lors de l'utilisation conjointe de JMS et des EJB, certains paramètres de méthodes (notamment lors de la création de la Session) étaient redondants car gérés par le conteneur EJB (`createSession(boolean transacted, int acknowledgeMode)`).

Un des prérequis principals de cette nouvelle version de la spécification est donc la simplification... simplification mais pas à n'importe quel prix puisqu'il était nécessaire de conserver la rétro compatibilité avec les versions ultérieures de la spécification.

Nigel nous a donc présenté certaines des pistes qu'ils avaient (pistes encore ouvertes à discussion).
Ce sont ces dernières que je vais essayer de retranscrire ci-dessous dans l'ordre dans lesquelles il les a énoncé.

#Simplification sur la création de Sessions
Afin de simplifier l'API de JMS pour créer une Session, Nigel nous a présenté deux pistes :

* Ajout d'une méthode `createSession(SessionMode sessionMode)` pour JavaSE où la classe SessionMode pourrait être une classe ayant comme variable d'instance le mode de transaction et le type d'acknowledge.
* Ajout d'une méthode `createSession()` qui serait utilisée et présente seulement pour JEE.

#Supprimer la lourdeur des close()

Afin de rendre moins verbeuse l'utilisation des `Connection` et des `Session`, une proposition plausible pourrait être de leur faire implémenter l'interface `java.lang.AutoCloseable`.

Ainsi, il serait alors possible d'avoir les résultats suivants (pour rappel, il ne s'agit que de propositions car cela est toujours à l'étude) :

* avec Java 7 dans un contexte JEE :
```java
@Resource(mappedName="...")
ContextFactory contextFactory;
 
@Resource(mappepdName="...")
Queue orderQueue;
 
public void sendMessage(String payload) {
  try (messagingContext mCtx = contextFactory.createContext() ;) {
    TextMessage textMessage = mCtx.createTextMessage(payload);
    mCtx.send(orderQueue, textMessage);
  }
}
```

* avec CDI dans un contexte JEE :

```java
@Resource(mappedName="...")
Queue orderQueue;
 
@Inject
@MessagingContext(lookup="...")
MessagingContext mCtx;
 
@Inject
TextMessage textMessage;
 
public void sendMessage(String payload) {
  textMesage.setText(payload);
  mCtx.send(orderQueue, textMessage);
}
```

* toujours avec Java 7 dans un contexte JEE :
 é
```java
@Inject
@JMSConnection(lookup="...")
@JMSDestination(lookup="...")
MessageProducer producer;
 
@Inject
TextMessage textMessage;
 
public void sendMessage(String payload) {
  try {
    textMessage.setText(payload);
    producer.send(textMessage);
  } catch (JMSException e) {
    //todo
  }
}
```

#Autres simplifications

Les autres simplifications d'API envisageables pourraient être :

* de ne pas à créer préalablement un objet de type Message (ce qui pourrait permettrait de faire directement : producer.send(String/Serializable); ). Cependant, ce type d'API ne permettrait pas de positionner des propriétés sur le message et ne serait pas adapter pour des messages de types BytesMessage ou StreamMessage. A ajouter à cela la question de son pendant pour la réception
* de simplifier la gestion des DurableSubscriber (http://download.oracle.com/javaee/1.4/api/javax/jms/Session.html#createDurableSubscriber(javax.jms.Topic, java.lang.String)), qui, au jour d'aujourd'hui, doivent obligatoirement posséder un identifiant client et un nom de souscription. Ces deux paramètres pourraient être rendus optionnels dans le cadre d'une utilisation conjointe avec les EJB 3.2

#Vers le futur...

Autre que la simplification des APIs, Nigel nous a présenté ce que pourrait apporter JMS 2.0 dans nos besoins de demain.

Ainsi, JMS 2.0 devra, pour répondre aux besoins des problématiques de type SaaS (_Software As A Service_), permettre de déclarer ses ressources créées dans un serveur d'applications et de les enregistrer dans un annuaire JNDI (comme c'est actuellement le cas pour les DataSource).
Pour ce faire, de nouvelles annotations pourraient voir le jour (par exemple : `@JMSConnectionFactoryDefinition` et `@JMSDestinationDefinition`) mais également un SPI (_Service Provider Interface_) pour le faire de manière programmatique.

Concernant une meilleure intégration avec les serveurs d'applications (ce qui permettrait d'utiliser n'importe quel JMS Provider dans n'importe quel serveur d'application JEE), Nigel propose la solution de JCA (_Java Connector Architecture_), un peu comme ce qui existe avec le drivers de bases de données.

#Nouvelles features

Concernant les nouvelles features de l'API, Nigel nous a ensuite présenté ce qui pourrait arriver, à savoir (en vrac) :

* l'émission de messages avec un acquittement asynchrone du serveur,
* la possibilité pour un client JMS d'utiliser des Future,
* la possibilité d'émettre un message et de ne pas à avoir à attendre la réception d'un acquittement pour savoir si le message a bien été émis.

Il se pourrait également que la propriété `JMSXDeliveryCount` ne soit plus optionnelle, ce qui permettrait aux serveurs d'applications de mieux gérer le _flood_. 

De même, la gestion des Topics hiérarchiques pourraient être rendue obligatoire.

Enfin, une meilleure gestion des batch pourrait être possible via de nouvelles API comme par exemple l'introduction de la méthode `receive(Message[])`.

#Conclusion

En conclusion, je suis assez mitigé par cette session.

En effet, autant, je trouve que certaines propositions permettraient de rendre moins verbeuses l'utilisation de JMS, autant, je ne suis pas convaincu que pouvoir faire la même chose de différentes manières soit une bonne chose (et, cela, même si ça simplifie le travail du développeur...).

En outre, je trouve que, pour la majorité des propositions (même si elles sont totalement viables), l'axe de JEE est trop important.

Enfin, intégrer fortement CDI avec JMS m'embête un peu car cela nécessite de disposer d'un conteneur CDI, chose qui n'est pas toujours le cas dans un contexte Java SE et, même si je trouve CDI sexy, je ne pense pas que la majorité des développeurs ou que nos clients soient prêts à franchir le pas en raison de la complexité intrinsèque à CDI (bien sûr, JMS 2.0 se devait de se tourner vers le futur et CDI se devait d'être pris en compte)... enfin, il ne s'agit que d'un avis personnel... l'avenir nous dira ce qu'il en est... ;-)

Dernier point (mais là encore, c'est totalement personnel), j'aurais aimé voir une intégration des notions d'EIP (Enterprise Integration Pattern) même si la notion de filter (au sens EIP du terme) sort un peu du scope de JMS mais bon... ;-)

~~Pour conclure, Nigel a bien présenté la direction que pourrait prendre JMS dans un futur proche mais, comme il nous l'a fréquemment fait remarquer, il ne s'agit que de pistes et, d'ailleurs, toutes les contributions sont les bienvenues.~~
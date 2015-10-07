---
layout: post
title: "Template de projets REST"
date: 2015-10-07 17:33:08 +0200
comments: true
footer: true
categories: 
- rest
- java
- microservice
---
![left-small](/images/template_rest.png)
Il y a déjà un long moment, j'avais posté une série d'article expliquant comment il était possible de faire des web service de type REST de manière simple via [RestEasy-Netty](http://blog.jetoile.fr/2014/03/jaxrs-netty-et-bien-plus-encore-mode.html) ou via [Undertow](http://blog.jetoile.fr/2015/06/undertow-pour-booster-vos-services-rest.html).

Dans la continuité de cette course au plus léger, je me suis dit que cela pouvait être intéressant de faire une petite étude un peu plus exhaustive des solutions légères qui existaient.

L'objectif étant extraire une sorte de bench un peu naïf et un peu _out of the box_. Parmi les solutions retenues, il y a :

* Resteasy-Netty
* Resteasy-Undertow
* Restlet
* SpringBoot
* Resteasy sur Tomcat en utilisant ses connecteurs NIO
* Resteasy sur Jetty

Cette article est là pour restituer mes résultats...
<!-- more -->

En fait, non... j'ai menti puisque je ne ferai aucun retour mais que je donnerai seulement le lien vers mon github où il est possible de trouver ces _bootstrap_ de projets...

En effet, faire un _bench_ est dangereux et complexe surtout quand toutes les implémentations ne sont pas maitrisées et qu'un _tuning_ de ces dernières peut grandement modifier le résultat.

En outre, avoir un service exhaustif (autre que un simple _helloword_) qui est représentatif d'une vrai application et qui ne fait pas que taper dans le cache de la JVM ou de l'OS est plus complexe qu'écrire un simple _sample_.

Enfin, par manque de moyen (2 ordinateurs reliés par un wifi capricieux et par flemme de me monter des environnements plus représentatifs), je n'ai pu obtenir de résultats fiables...

Aussi, ci-joint les _repos_ où il est possible de trouver le code (qui se veut ultra simple et qui a été fait sans chercher l'optimisation et sur un coin de table donc si des bourdes ont été faites, je m'en excuse...) :

* [Sample RestEasy-Netty](https://github.com/jetoile/resteasy-netty-sample)
* [Sample Dropwizard](https://github.com/jetoile/dropwizard-sample)
* [Sample Restlet](https://github.com/jetoile/restlet-sample)
* [Sample RestReasy-Undertow](https://github.com/jetoile/undertow-sample)
* [Sample Tomcat/Jetty](https://github.com/jetoile/tomcat-resteasy-sample) : simple webapp à déployer dans les conteneurs avec les bonnes options
* [Sample SpringBoot](https://github.com/jetoile/springboot-sample)
* [Sample de projet Gatling pour le tir de performance](https://github.com/jetoile/gatling-sample)



Ainsi, si le coeur vous en dit, vous pourrez vous faire vous même une idée de qui est le plus fort... et même comparer avec vos solutions maisons... ;)

Allez, et parce que je suis sympa, je mets quand même le rapport Gatling obtenu suite à 1 seul tir en local. Je laisse le lecteur se faire une idée... ou pas...

* [Résultat Dropwizard](/images/res_charge_gatling/dropwizard-1/index.html)
* [Résultat Jetty](/images/res_charge_gatling/jetty-1/index.html)
* [Résultat RestEasy-Netty](/images/res_charge_gatling/netty-1/index.html)
* [Résultat Restlet](/images/res_charge_gatling/restlet-1/index.html)
* [Résultat SpringBoot](/images/res_charge_gatling/springboot-1/index.html)
* [Résultat Tomcat NIO](/images/res_charge_gatling/tomcat-nio-1/index.html)
* [Résultat RestEasy-Undertow](/images/res_charge_gatling/undertow-1/index.html)

Voilà un article un peu facile et qui n'apporte pas grand chose mais je trouvais qu'il était toujours intéressant pour les lecteurs curieux d'avoir la possibilité de voir différentes implementations...
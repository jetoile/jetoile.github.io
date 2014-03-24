---
layout: post
title: "Petite découverte de CloudBees"
date: 2011-11-14 22:40:16 +0100
comments: true
categories: 
- avis
- cloudbees
- intégration continue
- java
---
![left-small](http://1.bp.blogspot.com/-F_mNTL2_BE4/TsACHNBk0fI/AAAAAAAAAdE/Ngcp8YNiDL0/s1600/cloudbees03.png)

La semaine dernière, j'ai eu la chance d'assister à un atelier sur [CloudBees](http://www.cloudbees.com/) chez nos amis de [Xebia](http://blog.xebia.fr/). 

Ce dernier a été organisé avec brio par (je le fais par ordre alphabétique pour éviter tout favoritisme ;-) ) :

* Eric Briand
* Simon Caplette
* [Nicolas De Loof](https://twitter.com/#%21/ndeloof)
* [Cyrille Leclerc](https://twitter.com/#%21/cyrilleleclerc)
* [Olivier Michallat](https://twitter.com/#%21/olim7t)
* [Jean-Louis Rigau](https://twitter.com/#%21/jlrigau)
* [Emmanuel Servent](https://twitter.com/#%21/eservent)

<!-- more -->

Comme d'habitude, organisation bien rodée et atelier préparé aux petits oignons mais ce n'est pas le but de mon article...

L'objectif de cet article est, pour ceux qui n'étaient pas présents, de faire un rapide tour d'horizon de ce qui m'a marqué dans la solution qu'apporte Cloudbees avec ses offres DEV@Cloud et RUN@Cloud. Cet article sera surtout focalisé sur la partie usine logicielle offerte par Cloudbees car c'est celle qui a été mise en avant pendant l'atelier et c'est aussi celle qui me semble la plus intéressante du point de vue de mes besoins actuels.

Je vais donc essayer de vous résumer ce que j'ai apprécié... Pour ce faire, je ne reviendrai pas sur le usecase qui a été utilisé car je pense qu'il vaut mieux assister à l'atelier et qu'il faut bien laisser un peu de suspens... ;-). De même, je ne parlerai ni de l'intérêt du cloud ni de comment mettre en place DEV@Cloud ou RUN@Cloud.

Cet article s'articulera donc en deux parties :

* la première consistant en un très rapide retour sur ce que propose CloudBees,
* et la deuxième sur ce qui me plait dans une approche telle que celle proposée par CloudBees.



PS : au passage, encore un grand merci aux organisateurs de l'atelier!

#\{DEV,RUN\}@Cloud?... Kesako?

CloudBees propose deux services DEV@Cloud et RUN@Cloud.

DEV@Cloud offre, pour sa part, un service de type PaaS (Platform As A Service) SaaS (Software As A Service) qui offre un environnement de développement, de construction et de test en s'appuyant sur le cloud.

DEV@Cloud permet :
de collaborer avec d'autres personnes en s'appuyant sur des SCM/DVCS comme SVN ou Git,
de construire, tester et déployer de manière continue avec Jenkins,
de tester l'application web cible en s'appuyant sur le service à la demande SauceLab,
de controler la qualité du code en s'appuyant sur Sonar,
de gérer les artéfacts en utilisant Artifactory.
RUN@Cloud fournit, quant à lui, un ensemble d'outils permettant de simplifier le processus de déploiement d'une application Java dans le cloud. Il correspond, pour sa part, un service de type PaaS (Platform As A Service).

![medium](http://1.bp.blogspot.com/-aGeioKNPLJc/TsACXTWxgJI/AAAAAAAAAdM/yG4C8B3PmOA/s1600/cloudbees04.png)

<u>PS</u> : vous aurez remarqué que, pour cette partie, je ne me suis pas trop foulé puisque je n'ai fait que reprendre les informations présentes sur le site... ;-)

#C'est bien gentil tout ça et moi, ça m'apporte quoi?

Comme vous avez pu le constater, DEV@Cloud et RUN@Cloud fournissent tout pleins de services.

Cependant, la première question qui me vient à l'esprit quand je vois tout ça, c'est : 
> "ben, je fais pareil dans mon usine logicielle... : pourquoi je m'embêterai à payer une solution et surtout, pourquoi serais-je prêt à perdre la main sur mon infra mais aussi qu'on m'enlève l'occasion de m'amuser avec de nouveaux jouets..." 

(pour le dernier argument, cela ne regarde que moi... ;-))

![medium](http://2.bp.blogspot.com/-dDnr_qH0vM0/TsACyBmfmsI/AAAAAAAAAdU/TI3TYA-Ujmc/s1600/Cloudbees01.png)

En fait, les réponses sont simples (et je ne parlerai pas des réponses évidentes de coûts -infra, supervision, maintenance, etc.-).

Déjà revenons sur les possibilités offertes par DEV@Cloud (je n'aborderai pas le fait que CloudBees propose également l'hébergement du SCM, du Repository Manager et de Sonar) :

En offrant la partie orchestrateur (via Jenkins), on délègue au cloud le soin de gérer les agents de build. Ainsi, il n'y a plus à administrer la ferme de serveurs succeptibles d'héberger les agents (tâche assez réberbative et ingrate).

De plus, en combinant la partie build/package à la partie RUN@Cloud permettant de déployer de manière continue l'application sur une instance dédiée, cela permet de tester fonctionnellement et de manière intégrer la solution. Encore une fois, en passant par un service de CloudBees, il n'y a plus à se soucier de devoir réserver un serveur ou de devoir déployer un conteneur de servlets puisque cela est fait pour nous. 

Concernant les tests à proprement parler, CloudBees offre une intégration avec [SauceLab](http://saucelabs.com/) qui se charge de l'exécution des tests Selenium (ie. qui prends à sa charge l'installation des agents Selenium ainsi que leurs configurations en mettant à disposition les browsers (butineurs ;-)) nécessaires à l'exécution des tests Selenium). En outre, SauceLab propose même la vidéo du test exécuté permettant, ainsi, a postériori d'analyser ce qui a échoué pendant le test. Bien sûr, DEV@Cloud n'offre pas encore la décorrélation entre le test fonctionnel exécuté et la partie vue mais, coté infrastructure, il n'y a plus à se soucier des agents et coté analyse, il n'est plus nécessaire d'analyser les rapports surefire pour connaitre le test en échec.

Il est également possible d'effectuer un test de charge en s'appuyant sur le plugin maven [JMeter](http://jmeter.apache.org/) en ciblant l'instance déployée dans RUN@Cloud. L'intégration avec le service [New Relic](http://newrelic.com/) offre la possibilité de superviser l'activité de l'instance afin d'évaluer son comportement. En outre, en configurant RUN@Cloud de manière scalable, il est aisé de tester la charge supportée par de multiples instances (pour ce faire, CloudBees passe par un load balancer qui, si je ne me trompe pas, fonctionne, à ce jour, en round-robin). L'intégration du service New Relic permet de s'abstraire de la mise en place d'une solution comme Hyperic sur un environnement de recette, chose qui manque régulièrement et qui peut être consommateur en temps. En outre, New Relic permet même de déterminer les points de contension en offrant la possibilité d'activer un profiling à chaud dans l'application cible. 

Concernant le comportement applicatif, l'intégration du service [Papertrail](https://papertrailapp.com/) offre la possibilité d'accéder aux logs de l'application mais permet également d'agréger les logs pour l'ensemble des instances et de les analyser. Ce dernier point est généralement manquant lors de la mise en place d'un environnement de test/validation/recette pour des questions d'infrastructure mais également de moyen. Aussi, disposer d'un tel service intégré avec l'offre RUN@Cloud dans un contexte de test (mais, bien sûr, pas uniquement) apporte un réel plus dans l'analyse des anomalies et du comportement de l'application.

#Conclusion

Vous l'aurez compris (enfin, je l'espère), les offres DEV@Cloud et RUN@Cloud de CloudBees m'ont vraiment fait bonne impression. Je n'ai fait qu'effleurer les possibilités offertes et je suis sûrement passé à coté de beaucoup de features mais je tenais tout de même à partager mon ressenti sur ces offres.

C'est vrai que mon article est très orienté usine logicielle (peut être parce que c'est justement le point de départ de l'atelier) mais, au moins pour ce seul point, il mérite qu'on s'intéresse aux services offerts par CloudBees (et services, il y en a...).

![small](http://4.bp.blogspot.com/-Nws_YjWNluM/TsAC_MmnpzI/AAAAAAAAAdc/W78rV1GiHns/s1600/cloudbees02.png)

Cependant, il reste un point qui me chagrine... en effet, CloudBees offre une excellente forge logicielle clé en main et qui semble parfaitement fonctionnelle mais, à mon sens, ce qui manque le plus est la bonne utilisation de ces outils et surtout, en amont, la bonne compréhension de ce qu'est une bonne usine logicielle et de son importance.

Bien sûr, si le client est réticent, il est toujours possible de faire une approche bottom/up en attaquant par la forge et en lui montrant le gain mais, je trouve cela dommage que l'on soit obligé d'en arriver là alors qu'il devrait en voir tout le bénéfice (un peu comme maitriser son livrable et son build ou le cycle de vie de son application...) surtout dans un contexte où l'agilité est un terme de plus en plus courant et qui prone des livraisons fréquentes, chose possible que si l'on dispose d'une bonne méthodologie, d'une bonne forge et que l'on a compris de l'intérêt des TUs, du refactoring et de la revue de code...

Enfin, je m'égare... ;-)
Pour revenir à nos moutons, ce que j'ai apprécié avec DEV@Cloud et RUN@Cloud est la grande intéraction et la modularisation des services avec Jenkins. Cependant, même si la solution offerte par CloudBees est de qualité et est simple à mettre en oeuvre en étant clé en main, il reste du ressort de l'équipe de mettre en place les bonnes pratiques pour construire l'usine logicielle sans lesquelles aussi bon que soient les outils et services à sa disposition, cela s'achèvera par un échec (enfin, là, il s'agit de mon humble avis... ;-)).

Pour finir, je tiens à préciser que je n'ai pas testé de manière plus approfondie les différents services offerts par CloudBees autre que ce qui nous a été présenté lors de l'atelier et j'espère ne pas avoir dénaturé le produit. Si des points étaient à ajouter, n'hésitez pas à m'en faire part afin de pouvoir compléter l'approche naïve que je pourrais en avoir ;-)

A noter aussi que concernant l'injection de charge, à ce jour, il n'existe pas de possibilité de scaler les injecteurs dans différents agents, ce qui limite l'intérêt de l'injection (je ne reviendrai pas sur le fait que si l'injecteur est saturé, cela fausse les résultats obtenus pour le SUT (System Under Test)) mais que cela devrait être résolu dans un futur proche.

<u>PS</u> : Je tenais aussi à préciser que je ne touche rien pour ce post mais que cela me faisait plaisir de parler d'un truc que j'ai apprécié (et puis je fais ce qui me chante avec mon blog... !! ;-) ) et puis aussi à remercier Stéphanie pour avoir consacré du temps à relire cet article et m'avoir apporté son esprit critique ;-)
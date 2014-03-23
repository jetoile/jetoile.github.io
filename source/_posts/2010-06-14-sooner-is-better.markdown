---
layout: post
title: "Sooner is better"
date: 2010-06-14 11:34:55 +0100
comments: true
categories: 
- réflexion
---
<img height="171" src="http://farm3.static.flickr.com/2089/2172462686_56d3373111.jpg" width="171" alt="left"/>
Bon, voilà, cela fait un moment que je n'ai pas bloggé... manque d'inspiration peut être... manque de temps ou changement des priorités pour consacrer plus de temps à la lecture de livres en retard...

Dans cet article, je voudrais revenir sur un point que j'avais légèrement abordé dans ce post portant sur les livrables mais en le développant pour se focaliser sur une petite réflexion portant sur l'importance de s'intégrer aux processus de compilation et de livraison au plus tôt... en fait, pour être plus précis, je n'avais pas vraiment abordé cette problématique mais cela vient du même constat... ;-)

<!-- more -->

En effet, j'ai parfois pu constater que certaines équipes ou personnes avaient tendance à reculer jusqu'au dernier moment (1 ou 2 jours -si ce n'est quelques heures- avant la livraison de leur partie dans le projet ou l'application cible) l'intégration de leur code dans le référentiel (terme que j'utiliserai, ici, au sens large et incluant le SCM, le build et le processus de livraison) et que, le plus souvent, ce projet n'avait pas été pensé pour être livré ou compatible avec la procédure de livraison mis en place (et utilisé) par les autres projets du référentiel.

Pour moi, il s'agit d'une erreur!

Je vais donc lister un ensemble d'idées reçues ou entendues à ce sujet et en dire ce que j'en pense... : 

* Le développeur peut se concentrer sur les fonctionnalités à développer : Si ses fonctionnalités ne peuvent être intégrées au livrable, cela met en péril la totalité de l'équipe et est, donc, source de retard qui seront, généralement, totalement incompris par vos manageurs.
* Il faut livrer au plus vite car la fonctionnalité était demandée pour hier : De même que pour le point précédent, si la fonctionnalité ne marche QUE sur le poste du développeur, elle ne pourra être exploitable en production. En outre, si l'équipe de production est incapable de déployer sur l'environnement cible simplement, si un fait technique (Rapport d'anomalie ou demande d'évolution) apparait (qui peut assurer que son code est bugs-free...? ;-) ), alors il faudra refaire le travail de déploiement et de configuration manuellement (et qui dit manuellement dit aussi facteur humain et donc soucis...). Et puis, entre nous, je trouverai ça assez drôle d'être obligé de lancer eclipse sur une machine de production... ;-)
* Ce n'est pas le travail du développeur : Cela fait aussi parti de son travail que de penser aux problématiques transverses : il se doit de prendre en compte les problématiques transverses et non d'être passif et d'attendre les spécifications pour coder sans se poser de questions. 
* Il y a une équipe (personne?) dédiée pour faire ce boulot et intégrer le code à l'existant : Une équipe est peut être dédiée à cette tache, mais qui est le plus au courant des subtilités de comment faire fonctionner le code (présence de fichier de configuration, de variables systèmes, de pré-requis, ...) si ce n'est le développeur lui-même? De toute façon, le développeur devra re-tester le code intégrer dans le référentiel, donc autant que ce soit lui qui le fasse.

De plus, à ces points, j'aimerais ajouter les suivants :

* Savoir son code intégré au référentiel dès le début permet de ne plus avoir cette épée au dessus de sa tête et donc de se concentrer pleinement jusqu'au bout à sa fonctionnalité : la variable du comment livrer ou intégrer est écartée et cela engendre moins de stress. Le code peut également être repris par quelqu'un d'autre rapidement et donc, un coup de main pourra être demandé en cas de besoin sans passer par la phase du "je te file le code sur une clé USB et je t'explique comment configurer ton environnement pour compiler et lancer le projet" ;-)
* Intégrer son code dans le référentiel permet généralement de se poser la question des dépendances nécessaires au projet et, cela peut mettre en avant des problèmes structurels (dépendances cycliques, besoin de factoriser un module, ...)
* L'intégration continue, si elle est mise en place sur le projet, ne peut prendre en compte un projet qui n'est pas dans le SCM. De même, elle ne pourra pas non plus construire (et tester) la fonctionnalité si elle n'est pas capable de la builder... cqfd...

Voilà! 

J'espère avoir réussi à expliquer pourquoi je pensais qu'il était important de s'intégrer au plus vite à un référentiel projet. C'est le travail de tous et cela ne doit surtout pas être pris par dessus la jambe. 
Pour conclure, j'ai envie de dire que reculer pour mieux sauter peut parfois faire beaucoup plus mal quand l'on se prend le saut de cheval dans la figure (cf. les cours de gym) ;-)
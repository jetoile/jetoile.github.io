---
title: "Resign Patterns : les nouveaux patterns"
date: 2011-12-02 23:04:54 +0100
comments: true
tags: 
- architecture
- avis
- divers
---

![left-small](http://2.bp.blogspot.com/-mP_z1e0Qo6M/TtiCRYS_PVI/AAAAAAAAAgQ/Dntolyd6oUg/s1600/Fail.png "Crédit photo : http://www.flickr.com/photos/esthervargasc/6010520395/")

Cet article est une traduction "libre" de l'excellent papier de [Michael Duell](mitework@yercompany.com) qui se nomme ["Resign Patterns" Ailments of Unsuitable Project-Disoriented Software](http://www.lsd.ic.unicamp.br/~oliva/fun/prog/resign-patterns).

En fait, _Resign Patterns_ reprend le principe des _Design Patterns_ tels que décrit par _the Gang Of Four_ mais en proposant un tout autres types de _Patterns_... Je vous laisse juger de leur véracité... Je pense qu'ils ont suffisamment fait leurs preuves pour ne pas avoir droit, eux aussi, à leur gloire... ;-)

Aussi, au même titre que les patterns du GoF, je vous invite à utiliser les dénominations décrites par les Resign Patterns pour vous faire comprendre de vos collègues quand vous parlez du design d'un programme. Ainsi, vous pourrez briller en société mais surtout vous faire comprendre par vos pairs ;-) 

<!-- more -->

Dans le même style, je vous conseille les excellents articles suivants :

* http://blog.xebia.fr/2011/11/29/revue-de-presse-xebia-239#Humourdedveloppeur
* http://www.dodgycoder.net/2011/11/yoda-conditions-pokemon-exception.html

#Cremational Patterns

Cette catégorie de Patterns en regroupe 5 : 
##Abject Poverty

Le __Abject Poverty Pattern__ est visible dans les logiciels qui sont si difficiles à tester et à maintenir que cela abouti généralement à un dépassement de budget pharaonique.

##Blinder

Le __Blinder Pattern__ est la solution opportune à un problème sans qu'aucune anticipation n'ait été faite sur de futurs modifications dans les exigences. Il est difficile de savoir s'il est nommé de la sorte en raison de la pauvre vision des personnes qui ont faites le design du logiciel pendant la phase de développement, ou du désir de faire souffrir ses yeux pendant la phase de maintenance.


##Fallacy Method

Le __Fallacy Method Pattern__ est visible dans le traitement de cas particuliers. La logique semble correct mais si quiconque tente de tester ces cas aux limites ou s'ils venaient à se produit, alors les erreurs de logiques apparaitraient au grand jour.

##ProtoTry

Le __ProtoTry Pattern__ offre une façon rapide et sale de développer un logiciel. Généralement, l'intention première est de vouloir réécrire le code utilisant le __ProtoTry__ mais en l'améliorant en mettant en pratique les leçons apprises pendant la phase de conception. Malheureusement, souvent, un planning inadapté ne le permet pas. Le __ProtoTry__ est aussi connu sous le nom de __code Legacy__.

##Simpleton

Le __Simpleton Pattern__ est un pattern extrèmement complexe utilisé pour faire une tache triviale. Le __Simpleton__ offre un indicateur fiable sur le niveau de compétence des concepteurs du code.

#Les Destructural Patterns

Cette catégorie de Patterns en regroupe 7 : 

##Adopter

L'__Adopter Pattern__ fournit un foyer à toutes les fonctions orphelines. Il en résulte une large famille de fonctions qui ne se ressemblent pas et qui n'ont de commun que le fait d'avoir été adopté.

##Brig

Le __Brig Pattern__ est un conteneur de classes pour le mauvais logiciel, aussi connu sous le nom de __Module__.

##Compromise

Le __Compromise Pattern__ est utilisé pour trouver un compromis entre la qualité et le planning. Il en résulte généralement un logiciel d'une piètre qualité qui, de plus, est en retard.

##Detonator

Le __Detonator Pattern__ est extrèmement commun mais souvent indétectable. Par exemple, une implémentation fréquente du __Detonator Pattern__ est l'utilisation des deux derniers chiffres d'une année lors de la manipulation et le calcul appliqué à des dates. La bombe est présente et n'attend qu'à faire son travail...

##Fromage

Le __Fromage Pattern__ est souvent remplis de trous. Il consiste en une multitude de petites astuces qui rendent impossible toute portabilité du code. Généralement, plus il est vieux et plus il sent fort!

##Flypaper

Le __FlyPaper Pattern__ est écrit par une personne et maintenu par une autre. Cette dernière se retrouve alors coincé et préfèrerait périr avant de se perdre.

##ePoxy

L'__ePoxy Pattern__ est visible dans les modules logiciels fortement couplés. Lorsque le couplage entre modules augmente, c'est qu'il y a souvent le pattern ePoxy qui lie ces modules.

#Misbehavioral Patterns

Cette catégorie de Patterns en regroupe 11 : 

##Chain of Possibilities

Le __Chain of Possibilities Pattern__ est visible dans les gros modules peu documenté. Personne n'est sûr de l'étendu de ses fonctionnalités, mais ses possibilités semblent infinis. Il est aussi connu sous le nom de __Non-Deterministic Pattern__.

##Commando

Le __Commando Pattern__ est utilisé pour intervenir rapidement et faire que le travail soit fait. Ce pattern est capable de briser toutes les encapsultation afin d'accomplir sa mission. Il ne fait pas de prisonniers.

##Intersperser

L'__Intersperser Pattern__ éparpille ses fonctionnalités dans tout le système, rendant impossible le test, la modification ou même la compréhension d'une fonction.

##Instigator

L'__Instigator Pattern__ semble bénin mais permet de faire des ravages dans d'autres parties du système.

##Momentum

Le __Momentum Pattern__ grossit de manière exponentielle, en taille, en besoin mémoire, en complexité et en temps d'exécution.

##Medicator

Le __Medicator Pattern__ est un tel goulot d'étranglement en terme de performance pour le système que toute autre partie du système semble dopé aux stéroïdes.

##Absolver

L'__Absolver Pattern__ est visible dans des problème résolus par des anciens employés. Tant de problèmes historiques ont été résolus par le logiciel que les personnes présentes peuvent absoudre le logiciel de blâme en déclarant que l'__Absolver__ est responsable de tous les problèmes présents. Aussi connu sous le nom de __"It's not in my code"__.

##Stake

Le __Stake Pattern__ est visible dans les problèmes dirigés par un logiciel écrit par une personne qui a, depuis lors, choisi la voie du management. Ainsi, même si de nombreux problèmes se produisent, l'auteur du pattern __Stake__ (qui est donc devenu manageur) empêchera quiconque de réécrire le logiciel sous prétexte qu'il est l'image de la réussite technique du manageur.

###Eulogy

L'__Eulogy Pattern__ est présent dans tous les projets qui emploie les 22 autres Resign Patterns. Il est aussi connu sous le nom de __Post Mortem__.

##Tempest Method

Le __Tempest Method Pattern__ et utilisé dans les derniers jours qui précède la livraison du logiciel. Il est caractérisé par un manque de commentaires tout en introduisant un grand nombre de __Detonator Pattern__.

##Visitor From Hell

Le __Visitor From Hell Pattern__ coïncide avec l'absence de contrôle sur le temps écoulé entre deux vérification d'un tableau. Ainsi, au moins une boucle de contrôle du système aura le pattern __Visitor From Hell__ qui surchagera les données critiques.
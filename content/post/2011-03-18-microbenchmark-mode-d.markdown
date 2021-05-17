---
title: "MicroBenchmark : mode d'emploi"
date: 2011-03-18 19:04:33 +0100
comments: true
tags: 
- java
- performance
---
![left-small](https://lh4.googleusercontent.com/-VYpiI5ySjEA/TYKVFQyw27I/AAAAAAAAAU0/XDPtplNTTKQ/s1600/perf03.png)

Cet article fait suite à une petite discussion que j'ai eue récemment avec un collègue sur qui était le plus fort entre le for et le foreach. Question qui peut paraitre, au premier abord, stupide mais qui m'a fait pas mal me creuser les neurones sur des points qui n'ont pas grand chose à voir avec le problème initial... (mais là, je m'avance un peu sur mon article... ;-) ).

Enfin, pour revenir au sujet initial (qui était, je le répète : "qui était le plus fort entre le for et foreach"), j'ai donc décidé de me faire un petit benchmark pour tester la performance des deux. Et là, ce fut le début des galères...

<!-- more -->

Je m'explique : suite à l'écriture rapide d'un petit benchmark pour tester mes deux méthodes d'itérations, j'ai obtenu des résultats qui pouvaient paraître bizarres (enfin bizarres pour moi). S'en est alors suivi de multiples discussions avec (par ordre alphabétique) Zouheir Cadi (@ZouheirCadi), [Olivier Croisier](http://thecodersbreakfast.net/) (@OlivierCroisier), [Arnault Jeanson](http://www.opensides.fr/) (@ArnaultJeanson), Séven Lemesle (@slemesle) et [François Ostyn](http://blog.ostyn.fr/) (@ostynf) que je tiens d'ailleurs à remercier pour leurs lumières et leur disponibilité.

Je vous passe les résultats (cela fera l'effet d'un article ultérieur) pour en revenir juste à la notion de Benchmark qui s'avère être beaucoup plus complexe que cela peut paraitre.

Aussi, suite à cette longue introduction (qui, mis à part raconter ma vie, n'apporte pas grand chose mais qui permet de positionner le contexte ;-) ), je vais donc tenter de montrer en quoi le fait de benchmarker un simple petit bout de code peut amener à des résultats erronés et peut être quelque chose de non trivial.
Pour ce faire, je ne vais pas me casser trop la tête mais juste vous faire un petit compte rendu de la présentation de Monsieur Joshua Bloch (himself) qu'il a donné en 2010 à Devoxx sur l'importance du micro-benchmark ou plus précisément sur les optimisations des performances (encore merci à François pour le lien! ;-) ).

#Un peu de vocabulaire...

Avant de commencer, nous allons d'abord définir ce qu'est un microBenchmark : un microBenchmark a pour objectif de mesurer les performances d'une petite portion de code. Ces tests s'exécutent généralement en un laps très court de temps (de l'ordre de la milliseconde) et ne font généralement pas d'opérations d'entrée/sortie (sauf, si bien sûr, c'est le but du microBenchmark). Le microBenchmark est très différent du profiling qui a pour cible l'application complète et qui fournit des résultats se voulant représentatifs de la réalité alors que le microBenchmark a vraiment pour objectif de se focaliser sur une portion de code très restreinte afin d'observer ses performances "brutes" sans se soucier d'un éventuel contexte d'exécution.

#Le premier point que l'on peut se demander est : faut-il optimiser son code?

La réponse ne diffère pas de celle fourni par le livre __"Effective Java"__ de Joshua Bloch : il ne faut pas tenter d'optimiser son code sauf dans certains cas comme lorsque l'on développe un API, un framework ou un langage (comme Groovy par exemple). En effet, écrire du code qui se veut optimisé peut conduire à complexifier le code le rendant ainsi illisible mais peut surtout avoir l'effet contraire en le rendant plus lent.

En effet, à l'époque, il était possible d'évaluer les performances d'un programme en comptant le nombre d'instructions qui était exécutées, c'est-à-dire en comptant le nombre de cycles des opérations. Cependant, cela n'est actuellement plus possible en raison de la trop grande différence qu'il existe entre le code écrit et le code qui est réellement exécuté par le processeur. Les coupables : le nombre de couches qui augmente avec notamment un langage de plus en plus compliqué qui s'appuie sur de plus en plus de librairies mais également sur une JVM de plus en plus complexe. A ces différentes couches logicielles vient également s'ajouter une couche matérielle qui se complexifie également et que peu de personnes peuvent prétendre maîtriser : les nouvelles architectures de processeurs, les unités de calculs fournies par les GPU, la gestion de la RAM. Et bien sûr, tout cela en y ajoutant le système d'exploitation.

Du coup, prédire les performances d'un bout de code devient quelque chose de non déterministe qu'il devient très difficile d'estimer.

Prenons un exemple simple : A votre avis, quel est le plus performant entre l'opérande `&&` (_et conditionnel_) et l'opérande `&` (_et logique_)?

A première vue, on pourrait se dire que le _et conditionnel_ est plus performant puisque la deuxième condition n'est pas obligatoirement évaluée. Cependant, cela n'est pas si simple. En effet, avec un bonne architecture (processeur, compilateur, ...), les deux conditions du _et logique_ peuvent être évaluées en parallèle, ce qui est généralement plus rapide que de laisser au processeur le soin de deviner si la deuxième condition doit être évaluée (dans le cas du _et conditionnel_). Bien sûr, cela dépend du compilateur et de la capacité de la machine d'exécution (capacité du Just-In-Time Compiler ou de la famille de processeur utilisée) à traiter l'opération (modulo bien sûr que les conditions ne soient pas trop complexes).

Du coup, prédire de manière absolue laquelle de ces deux opérandes est la plus performante est quasi-impossible et le seul moyen de le savoir est de le mesurer.

#Mesurer, oui!... mais comment?

![medium](https://lh4.googleusercontent.com/-Ay1I5wIy1g4/TYKYAd7UcQI/AAAAAAAAAU4/ZIDUoDTyD6Y/s1600/question.jpg "copyright photo : http://www.flickr.com/photos/susyna/3643831785/sizes/m/in/photostream/"")

Mesurer... cela semble simple, cependant, il est facile de se douter que les mesures relevées dépendent de l'environnement d'exécution. Plus grave encore, sur une même machine, l'exécution successive d'un même test (que l'on appellera tir par la suite) peut remonter des résultats très différents (différences qui peuvent atteindre 20%).

En outre, il est aisé de constater que si, lors d'un tir, le même test est effectué un grand nombre de fois, les métriques remontées ne sont pas linéaires. En effet, on constate trois phases (si on se concentre sur le temps de réponse d'une ou d'un ensemble d'opérations) : 

* la première phase où les temps observés sont élevés. Cette phase correspond au warm-up de la JVM,
* la seconde phase où les temps observés diminuent et stagnent. Cette phase correspond à la phase de stabilisation de la JVM,
* et la troisième phase où les temps diminuent. Cette dernière phase correspond à la phase de speed up de la JVM.

Ainsi, pour résumer, un benchmark doit effectuer un grand nombre de tirs qui doivent contenir plusieurs itérations du code à tester afin de permettre à la JVM de se stabiliser.

#Les raisons de tous ces soucis?

En fait, si prédire les performances d'une opération par rapport à une autre est difficile, cela est paradoxalement dû à l'augmentation de la complexité de nos systèmes, complexité croissante entrainée par la recherche de la performance. En effet, ces optimisations d'exécution ont entraînées l'inline de code, le cache ainsi que beaucoup d'heuristiques (Paradoxal tout ça... n'est-il pas?). 

Je m'explique (enfin pour être plus précis, Joshua Bloch explique ;-) ) : en fait, le thread _compilateur planning_ utilise de nombreuses heuristiques pour décider ce qu'il faut _inliner_. Ces décisions sont prises en fonction d'informations locales et peuvent avoir un impact global. En outre le _compilateur planning_ est exécuté par un thread tournant en tache de fond et dont le comportement peut varier pendant son exécution. Il est donc non déterministe. 

Du coup, il est quasiment impossible de prédire quelles seront les performances d'une application et plus embêtant de savoir si une opération sera plus performante qu'une autre dans un contexte donné.

#Et moi? Du coup, je fais quoi? Comment puis-je connaitre le gain de performances de mon code?

Et bien, la solution est statistique! Il faut mesurer les performances (en effectuant de nombreux tirs tels que cela a été décrit précédemment) du bout de code critique à surveiller régulièrement et en sortir des statistiques. Ce sont ces statistiques qui sont les seules métriques garantes des performance du code. Cependant, il est important de conserver les métriques de tous les tirs (qui pour rappel effectue l'opération de multiples fois) même si elles peuvent sembler aberrantes (bien sûr, si on suspecte que les résultats obtenus sont biaisés en raison d'un événement extérieur - comme une réindexation du disque dur ou le passage de l'anti-virus -, il ne faut pas les garder). Il convient alors de sortir de cet ensemble de tirs les métriques adéquates (moyenne, médiane, écart-type, ...).

Ainsi, ce sont ces résultats obtenus qui doivent être comparés dans le temps afin de déterminer l'impact d'une modification sur le bout de code à surveiller (en terme de performance).

#Conclusion

Ainsi, vous l'aurez compris (enfin j'espère ;-) ), Le microbenchmark est un exercice bien plus complexe qu'il en a l'air et il ne suffit pas d'ouvrir son eclipse et de jeter trois lignes de code dans un `main()`. Bien sûr, cela permet de se donner un ordre de grandeur mais qui est souvent non représentatif de la réalité et dont le comportement sera, presque à coup sûr, différent dans son contexte d'exécution de ce qui est observé dans son `main()`.

Cependant, il faut garder à l'esprit qu'il reste fortement déconseillé de tenter d'optimiser son code au risque de faire pire que mieux et surtout de rendre son code illisible et donc immaintenable. 

Du coup (et toujours d'après Joshua Bloch) :

* Si vous êtes développeur, utiliser les librairies high-level et déléguer les problématiques de performances aux couches inférieures (inférieures au sens géographique du terme bien sûr! ;-) ). Même si cela peut être dur à admettre, accepter de ne pas savoir et de ne pas pouvoir prédire les performances de votre code.
* Si vous êtes développeur de librairies, langages, frameworks ou de JVM, alors il faut apprendre à éviter les sources d'étonnement en s'appuyant sur des bonnes pratiques de développement (comme privilégier l'accès aux variables d'instances en jouant sur le niveau de visibilité plutôt que d'utiliser des getter/setter, ...) mais surtout mesurer constamment (tel que cela a été décrit) et faire des statistiques pour déterminer le comportement global au niveau performance.
* Si vous êtes un ayatolla (comme Doug Lea ou Cliff Dick), alors il faut prêcher la bonne parole autour de vous pour éviter les écueils dans lesquels il ne faut pas tomber.

Bien sûr (et cela conclura cet article), il faut garder à l'esprit qu'une optimisation ne s'applique potentiellement qu'à une famille de processeurs, VM ou architecture matérielle, et qu'au vu de la difficulté de prédire de quoi seront constitués nos ordinateurs de demain ou de prédire des optimisations qu'apporteront nos prochaines JVM, tout le travail effectué à l'instant t risque d'être totalement obsolète et erroné demain! 

#Pour aller plus loin...

* Présentation de Joshua Bloch sur Parleys : http://www.parleys.com/#sl=0&st=5&id=2103
* Site du framework Caliper : http://code.google.com/p/caliper/
* Présentation de Cliff Click : http://www.azulsystems.com/events/javaone_2009/session/2009_J1_Benchmark.pdf
* Page Wiki de Sun sur le microBenchmark : http://wikis.sun.com/display/HotSpotInternals/MicroBenchmarks
* Page de google Android sur la gestion des perfomances  d'Android : http://developer.android.com/guide/practices/design/performance.html
* Blog de Peter Ahé sur les compilateurs Java : http://blogs.sun.com/ahe/entry/hotspot_and_other_compilers

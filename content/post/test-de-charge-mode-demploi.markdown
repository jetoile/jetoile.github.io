---
title: "Test de charge : mode d'emploi"
date: 2012-10-17 12:29:39 +0100
comments: true
tags: 
- gatling
- jmeter
- clif
- java
---
![left-small](http://2.bp.blogspot.com/-8b4S0NQnPtk/UH2hoKTgkcI/AAAAAAAAAoc/iY1PXGjWPcA/s1600/_charge01.png)

Au cours de la vie d'un projet, il est courant d'avoir recours à des tests de charge.

Cependant, je me suis rendu compte que, trop souvent, le sujet est mal maîtrisé et donc malheureusement mal traité.

En effet, combien de fois n'ai-je pas vu un test de charge effectué avec la méthodologie "clic clic partout", l'utilisation de __Selenium__, la commande __curl__, ou, dans le meilleur des cas, avec quelques scénarii JMeter issus de l'utilisation du recorder de ce dernier.

Cela ayant la facheuse tendance à m'énerver (pour ceux qui me connaisse, cela se traduit par une certaine agitation de ma part consistant à me mettre à tourner sur la chaise en regardant le plafond ;-) ), je vais tenter dans cet article de redéfinir certains des concepts qui me semblent primordiaux pour effectuer un test de charge qui se veut un minimum bien conçu, c'est à dire se voulant représentatif de la charge réelle que peut supporter le SUT (_System Under Test_) mais surtout qui permettra d'obtenir des métriques exploitables et représentatives du comportement du SUT.

<!-- more -->

Je m'explique... sans vouloir être plus royaliste que le roi, il convient de re-préciser que le but intrinsèque d'un test de charge est, surtout, de mesurer et observer comment se comporte le SUT et pas seulement de lui en mettre plein la tête. En effet, ce sont ces mesures qui doivent être exploitées et il convient donc de mettre tout en oeuvre pour :

* qu'elles soient représentatives de ce qu'on veut tester en utilisant un bon jeu d'injection et en l'injectant comme il faut,
* que les métriques remontées soient exploitables c'est à dire quelles permettent, par exemple, de connaitre le débit ou le nombre d'utilisateurs simultanés qu'est capable de supporter le SUT mais également le temps de réponse moyen par rapport aux nombres d'utilisateurs simultanés.

Il est à noter que le test de charge n'a pas pour vocation intrinsèque la recherche d'un Hotspot dans l'application même si la méthode d'injection peut être identique. En effet, les outils nécessaires à la détection de point chaud peuvent génèrer un overhead qui fausse les résultats obtenus. Ce point est donc, même si souvent proche de la théorie des tests de charge, transverse avec, entre autre, la mise en place de sondes, l'activation de logs, l'utilisation d'un profiler ou la surveillance du traffic réseau et des serveurs (IO, CPU, RAM, collision des trames, ...).

# Test de charge, oui, mais lequel?...

Lorsque l'on parle de test de charge, il convient de se poser la question de ce que l'on veut tester. Il en existe 3 types :

* Test de tenue en charge.
* Test de surcharge.
* Test de robustesse.

# Test de tenue en charge

Ce type de test a pour objectif de valider la capacité de traitement du SUT qui peut alors être vu comme une boite noire.

Pour ce faire, on injecte des scénarii représentatifs de manière simultanés : on parle alors de charge injectée par des utilisateurs virtuels. L'objectif des tests de tenue en charge étant de déterminer la charge maximale admissible en continu, il est donc logique qu'il faille effectuer de nombreux tirs, tirs qui seront fonction du nombre d'utilisateurs virtuels.

Lorsque le SUT aura atteint sa capacité maximale, son fonctionnement se verra dégradé entraînant  entre autre, une hausse de son temps de réponse c'est à dire le débit délivré par le SUT. A ce moment, il pourra être constaté une inflexion dans les métriques collectées (temps de réponse, nombre de requêtes par seconde traitées, type de réponses récoltées).

Ce type de test doit être constitué de tirs se chiffrant en minute ou dizaine de minutes.

# Test de surcharge

Ce type de test a pour objectif de vérifier le comportement du SUT après avoir subi une charge supérieure à ce qu'il est capable d'absorber, charge qui aura été déterminé par le test de tenue en charge.

En effet, alors qu'il est connu que les performances du SUT seront dégradées lors de l'injection d'une charge avérée critique, il est important de savoir comment le SUT se comportera suite à cela : retourne-t-il dans un état stable ou faut-il le redémarrer...?

Le but de ce test n'est donc pas de collecter les métriques pendant le test de surcharge mais après.

# Test de robustesse

Ce type de test a pour objectif de valider le comportement du SUT après une longue activité. En effet, un système qui se veut un minimum bien conçu doit être capable d'avoir un _uptime_ se chiffrant en mois et non en heure. Cependant, il est courant de voir des contentions mémoires se produire après une longue activité du système qui peuvent être liés à un phénomène de résonance produite par un évènement particulier.

Pour ce faire, le test de robustesse doit être exécuté pendant plusieurs heures à une charge admissible maximale (qui aura été déterminé par le test de tenue en charge).

# Que mesurer et comment

Nous avons vu dans un premier paragraphe qu'il existait différents types de test charge. Cependant, l'accent a été également mis sur les métriques récoltées, et c'est ce que tentera de traiter cette partie, à savoir ce qu'on entend par métrique et comment elles doivent être récoltées.

Comme nous l'avons mentionné précédemment, le but premier d'une campagne de test de charge est de récolter des métriques afin de pouvoir qualifier le SUT. Par exemple, la détermination du débit maximal admissible en continu peut être déterminé grâce au schéma temps de réponse/charge.

![medium](http://2.bp.blogspot.com/-8b4S0NQnPtk/UH2hoKTgkcI/AAAAAAAAAoc/iY1PXGjWPcA/s1600/_charge01.png)

On constate sur cet exemple un point d'inflexion aux alentours de 150 utilisateurs virtuels qui est représentatif de la congestion du SUT.

# Extraction des métriques utiles

En fait, chaque point du tracé doit être la conclusion d'un test de charge pour un nombre d'utilisateur virtuel donné.

En effet, lors d'une campagne de test de charge, un ensemble de scénarii est déroulé. La majorité des outils d'injection de charge digne de ce nom agrège alors les résultats par requête et de manière globale.
Cependant, dans la majorité des cas, le résultat escompté est de savoir globalement comment se comporte le SUT par rapport à un nombre d'utilisateurs virtuels.

Aussi, il convient de prendre la médiane des résultats obtenus afin d'obtenir, pour un nombre d'utilisateurs virtuels, un seul et unique point alors utilisé pour déterminer le point d'inflexion.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
La médiane peut mieux convenir que la moyenne car elle est plus robuste que cette dernière en présence de valeurs extrêmes.
</ul></td></tr>
</tbody></table>

Cependant, il est courant qu'entre deux tirs d'un même scénario, pour un même nombre d'utilisateurs virtuels, des résultats différents soient obtenus. Aussi, pour lisser le résultat qui sera utilisé pour calculer le point d'inflexion, il convient d'effectuer plusieurs tirs, d'élaguer les résultats aberrants puis d'en effectuer la médiane.

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Il est important d'élaguer les mesures obtenues aberrantes car il peut arriver que les résultats d'un test soit faussés par des perturbations qui n'ont pas origine le SUT (garbage collector, contention lié à une ressource externe, ...). Cela peut biaisé le calcul statistique puisque, dans la majorité des cas, la distribution des temps de réponse ne suit pas une loi normale, ce qui a comme conséquence qu'une très longue queue de distribution biaise les calculs.</ul></td></tr>
</tbody></table>

Pour élaguer les résultats aberrants, il est possible de rejeter les valeurs qui se situent au delà des bornes définies par [m - k.s, m + k.s] où :

* m est la moyenne, 
* s est l'écart-type 
* et k le facteur de tri.

Il est à noter que, si le tri ramène la distribution des mesures à une loi normale, alors il existe une adéquation entre le nombre et l'étendue des valeurs :

![center](http://1.bp.blogspot.com/-vZtVUWA8gSI/UH2h77Qj1EI/AAAAAAAAAok/8MWSuwuiLFk/s1600/facteurTri.png)

<table border="1" cellpadding="0" cellspacing="0" style="text-align: justify;" width="100%"><tbody>
<tr><td style="border: 0px; text-align: center;"><a href="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" imageanchor="1" style="clear: left; margin-bottom: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/_XLL8sJPQ97g/TNCS7qzv19I/AAAAAAAAAL4/6qkEYy4pX6o/s1600/remarque.png" style="cursor: move;" /></a></td> <td style="border: 0px;">
Ce mécanisme d'élagage est aussi intégré dans des outils comme JMeter ou Gatling sous la notion de percentile.
</ul></td></tr>
</tbody></table>

![medium](http://2.bp.blogspot.com/-WUEu4x_dmEE/UH2iBj-XvTI/AAAAAAAAAos/0fkkNTOlRVA/s1600/gatling.png)

Une fois que les mesures significatives ont été obtenues pour chaque charge (ie. après que plusieurs tirs aient été effectués, élagués puis que la médiane ait été calculée), il est enfin temps de les agréger ensemble afin de déterminer le point d'inflexion. Pour ce faire, il convient d'établir les charges pour lesquelles le SUT se comporte de manière linéaire.

Cette détermination peut être réalisé par une régression linéaire entre le temps d'exécution et la charge. Ce calcul étant dépendant des données sélectionnées, il faut donc trouver les données pour lesquelles cette régression est optimisée.

![medium](http://4.bp.blogspot.com/-nqEeZTYd-Qo/UH2iIATj8zI/AAAAAAAAAo0/6Hm4hlG7qLA/s1600/regressionLineaire.png)

Une fois la régression linéaire estimée, il est alors possible de placer des bornes de confiance sur l'estimation afin de déterminer le point d'inflexion de manière exacte. Pour ce faire, il suffit de prendre comme l'équation de la régression linéaire additionné de k fois l'écart type (avec k choisie de manière à garder un niveau de confiance de 99%). 

![medium](http://1.bp.blogspot.com/-rHoZWEC8WrY/UH2iOcqnAOI/AAAAAAAAAo8/EggYczucZrI/s1600/determinationInflexion.png)

Bien sûr, la méthode d'élagage est également à appliquer aux résultats obtenus au sein d'un tir de test de charge et, de même, il convient d'utiliser la médiane afin d'avoir les meilleurs résultats possibles (ça aurait été trop simple sinon... ;-) ).

# Comment injecter

Nous avons vu dans les paragraphes précédents les différents types de test de charge ainsi que la théorie qu'il fallait appliquer lors de la récupération des métriques. 

Dans ce paragraphe, nous allons voir comment il convient de configurer son plan de test (ie. la configuration de l'exécution des scénarii).

## Injecter avec les bons injecteurs

Lors d'une injection de charge, il est important (si ce n'est indispensable) de s'assurer que les injecteurs ne sont pas surchargés.
En effet, dans la majorité des outils capable d'injecter de la charge, un thread est créé par utilisateur virtuel (chose qui n'est pas le cas avec Gatling).

Aussi, il convient de s'assurer de la bonne santé des injecteurs :

* soit en positionnant des sondes sur ces derniers et en agrégeant l'état des injecteurs avec les métriques récoltées,
* soit en surveillant "manuellement" leur comportement et en déterminant de manière empirique la charge que peut supporter un injecteur.

En effet, avoir un injecteur surchargé implique que la charge injectée sera biaisée et, donc, que les métriques récoltées seront fausses. Cela aura, bien sûr, des conséquences sur l'analyse du SUT.

![medium](http://4.bp.blogspot.com/-cAQI5ekhicE/UH5sWpSdg_I/AAAAAAAAAqo/75ahGaWInsU/s1600/multipleInjecteur01.png)

![medium](http://4.bp.blogspot.com/-EYVHyd0GCEU/UH5sbOKyUII/AAAAAAAAAqw/A0o-VjJgW7E/s1600/multipleInjecteur02.png)

Aussi, lors du choix d'un outils d'injection de charge, il est important de prendre ces éléments en considération.

De plus, si l'outils permet la gestion de plusieurs injecteurs, il est doit également être à sa charge l'agrégation de ces dernières, ie. qu'il doit être capable de recaler les dates des différents injecteurs et sondes afin de les faire correspondre.
Pour faire simple, c'est au système d'injection de gouverner l'ensemble des injecteurs mais également la phase de récupération et d'agrégation des métriques.

En outre, en plus des considération de surcharge potentielle des injecteurs, il peut être intéressant de distribuer les injecteurs afin de simuler des utilisateurs virtuels venant de différentes machines clientes (c'est aussi l'un des points qui doit être pris en compte lors de la variabilisation du scénario injecté).

# Injecter avec le bon profil de charge

Le profil de charge permet au système d'injection de gérer un ensemble d'utilisateurs virtuels qui exécutent en boucle le scénario.

Ainsi, un profil de charge comprend les notions suivantes :

* une rampe permettant d'atteindre progressivement le niveau spécifié pour les utilisateurs virtuels,
* un palier qui est atteint suite à la rampe où tous les utilisateurs virtuels injectent simultanément leurs scénarii,
* la phase terminale qui prend place lorsque l'injection a été effectuée.

![medium](http://3.bp.blogspot.com/-KrYwbUoMBdo/UH3JeeVR5RI/AAAAAAAAApk/YE4-lvz7h3k/s1600/profilCharge.png)

La rampe doit être choisie en fonction de la durée du test d'injection mais également en fonction du nombre d'utilisateurs virtuels qui sera utilisé. Elle permet d'éviter que tous les utilisateurs arrivent en même temps sur le SUT (chose qui ne se produit pour ainsi dire jamais) mais permet également d'effectuer un _warmup_ du SUT (JVM, chargement des caches, ...). En effet, il est courant que le SUT soit réinitialisé après chaque test de charge afin d'éviter les effets de bords qu'aurait pu produire un test de charge antérieure.

Le palier est, quant à lui, réellement représentatif de ce qui doit être validé par le SUT et c'est durant cette période que devra se faire l'analyse des métriques obtenues. De plus, l'enchainement des scénarii pour les utilisateurs virtuels doit se faire suivant une loi aléatoire afin que la charge puisse s'exprimer comme une valeur moyenne de débit. De plus, ce coté aléatoire permettra d'être au plus proche des actions des utilisateurs réels.

# Conclusion

Cet article n'avait pas pour objectif d'expliquer toute la théorie qui se cachait derrière les tests de charge (qui est oh combien compliqué... et qui s'appuie, entre autre, sur la théorie des files d'attente mais aussi sur de nombreuses notions statistiques) mais juste d'effleurer les notions primodiales qui se cachent derrières et qui, à mon humble avis, devraient être prises en considération (quitte à en élaguer sciemment certaines).

A noter enfin que cet article n'a pas abordé de nombreux points comme la variabilisation d'un scénario ou la mise en place d'_assets_ dans les injecteurs afin de tester si le SUT répond de manière cohérente ou pas.

Enfin, pour finir, je dirai qu'une campagne de test de charge devrait toujours être faite sérieusement (chose qui malheureusement est assez rare...) et qu'il est primordial de mesurer, observer encore et encore : injecter de la charge sans récolter de métrique est, à mon sens, totalement inutile (ça, c'est dit... ;-) ). Alors par pitié, qu'on ne me parle plus de clic clic partout ni de __Selenium__ pour les tests de charge... 

[__update__] ah oui, j'oubliais... il coule de sens qu'un test de charge doit pouvoir être reproduit à l'identique afin de pouvoir comparer les dégradations ou amélioration des performances entre deux moments d'où l'importance de maîtriser son scénario d'injection et de faire cela de manière un minimum industriel...

# Pour aller plus loin...

Parce que je suis tombé sur cette [ressource](http://home.nordnet.fr/~ericleleu/cours/nfe210/Tests_V10.00.pdf) en écrivant cet article, je le mets ;-)

Quelques outils d'injection de charge :
* [Gatling](http://gatling-tool.org/)
* [Apache JMeter](http://jmeter.apache.org/)
* [CLIF](http://clif.ow2.org/)

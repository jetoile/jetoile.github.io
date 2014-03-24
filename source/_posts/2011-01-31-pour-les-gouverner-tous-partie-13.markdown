---
layout: post
title: "Pour les gouverner tous - partie 1/3"
date: 2011-01-31 18:37:27 +0100
comments: true
categories: 
- java
- jgroups
- jmx
---

![left-small](http://3.bp.blogspot.com/_XLL8sJPQ97g/TUcoTratqiI/AAAAAAAAAUQ/Gmc1h0rvA2w/s1600/jmx_jgroups01.png)

Pour ceux qui l'auraient manqués (comment ça? Personne ne me suit... :( ), j'ai annoncé dans un post précédent que j'avais ouvert un compte sur GitHub pour hoster du code. Bien sûr, il y avait une petite idée derrière... ;-)

En fait, l'idée est partie d'un constat assez simple : sur différent projet, à chaque fois que j'ai voulu mettre en place JMX, ce qui m'a toujours profondément attristé était de ne pas pouvoir regrouper l'ensemble des informations des MBeans à un même endroit, c'est-à-dire, qu'il me fallait me demander où se trouvait tel ou tel MBean dans mon système alors que j'aurais aimé pouvoir me connecter sur n'importe quelle instance de mon système et y trouver tous les MBeans présents dans mon système. Enfin, pour être plus précis, j'aurais aimé pouvoir me connecter à n'importe quelle JVM qui hébergeait mon application et avoir accès à tous les serveurs JMX.
<!-- more -->

Bien sûr, me direz-vous, il existe des solutions comme Hyperic qui permettent d'agréger toutes les informations et d'offrir une solution à mon problème. Cependant, cela ne répondait pas à une des problématiques que j'ai pu avoir il y a quelques temps : dans le cas où il ne doit pas y avoir de serveurs dans mon système (même passif), une telle solution n'est pas envisageable. En effet, dans l'architecture de cette application, tous les noeuds de mon système étaient au même niveau hiérarchique afin d'éviter d'avoir un single point of failure. Aussi, introduire un manageur pour administrer mon système n'était pas envisageable. En outre, un autre des pré-requis de ce projet était qu'il n'y avait pas de connaissance préalable de où tournait mon application (au sens scalabilité du terme) : tout poste était susceptible de démarrer l'application sans qu'un serveur centralisé n'en ait connaissance.

Aussi, vous l'aurez compris, l'idée est, ici, de proposer une ébauche de solution pour répondre à la problématique précédemment citée : pour n'importe quelle application susceptible d'être exécutée de manière distributée, être capable d'avoir sur chacune la connaissance (connaissance au sens supervision et administration et non au sens applicatif du terme...) de l'état de mes autres instances.

En outre, cette petite mise en oeuvre était également un bon prétexte pour utiliser JGroups et JMX et montrer comment il était possible de les utiliser. 

La problématique étant posée, je vous propose donc une petite boite à outils permettant de faire cela de manière simple : en démarrant son application et en positionnant un agent sur la JVM, elle permet de remonter dans le serveur JMX courant tous les MBeans se trouvant sur les autres serveurs JMX (modulo qu'ils aient été démarrés par une JVM où l'agent aurait été mis). 

Pour ce faire, un premier choix a été de m'appuyer sur JGroups pour connaitre et être informé de l'état de mon système : je n'avais pas envie de gérer toutes les problématiques de Heartbeat & co. et j'ai donc décidé de déléguer à JGroups cette problématique. En outre, cela permettait une plus grande configuration de la couche de découverte et de surveillance de mon système puisque JGroups possédait déjà une grande facilité de configuration. 

La série d'articles qui suivra expliquera donc comment cela fonctionne et sur quoi je me suis appuyé pour le faire.

Elle sera découpée en deux parties :

* La première partie fournira une sorte de pseudo exemple d'utilisation de JGroups en proposant une solution permettant lors de l'utilisation d'une application distribuée de récupérer et de conserver localement l'état d'un objet distant dans une sorte de cache tout en supprimant cet objet si l'application distante est arrêtée.
* La deuxième partie ajoutera l'utilisation de JMX pour permettre la récupération des MBeans distants ainsi que leurs enregistrement dans le MBeanServer courant.
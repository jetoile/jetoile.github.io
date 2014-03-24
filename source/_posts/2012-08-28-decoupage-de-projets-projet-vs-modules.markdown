---
layout: post
title: "Découpage de projets : projet vs. modules"
date: 2012-08-28 10:47:11 +0100
comments: true
categories: 
- architecture
- avis
- java
- maven
- réflexion
- soa
---

![left-small](http://2.bp.blogspot.com/-wz814rCkbHY/UC1JmXEpcyI/AAAAAAAAAoA/6YjiYTg7fzw/s1600/couteaux.png)

Lorsqu'un projet débute, il est important (à mon avis) de se poser la question sur la façon dont celui-ci sera découpé. Pour être plus précis, il existe deux types d'approches :

* le découper fonctionnellement,
* le découper techniquement.

En outre, en plus de ce type de découpage, il est également important de s'interroger sur la façon dont il sera représenté dans le SCM : faut-il tout mettre dans le même projet (au sens SVN ou git du terme) en utilisant éventuellement des sous modules maven si c'est ce dernier qui est utilisé, ou faut-il en créer plusieurs?

C'est de ce dernier point dont il sera question dans ce court article qui présentera l'avis que j'ai pu me faire concernant le découpage technique du projet ie. s'il vaut mieux le découper en projets séparés ou en module (au sens maven du terme).

Il s'agit d'une opinion très personnelle qui peut ne pas être partagée par tous mais je trouvais intéressant de fournir mon humble avis et de le marquer noir sur blanc. Si je venais à dire des bêtises, au moins, cet article servira d'amorce à la discussion ;-)

<!-- more -->

#Module ou projet?

Comme mentionné en introduction, cet article traitera de comment architecturer son projet, à savoir, s'il vaut mieux découper son projet en différents projets ou en sous-modules. Bien sûr, ce point doit être pris en compte conjointement avec le découpage technique et/ou fonctionnel.

Dans la suite de l'article, les termes "projet" et "module" sont utilisés au sens maven du terme. Pour avoir les vraies définitions, il est quand même préférable de se référer à la documentation officielle, mais voilà comment je les décrirai succinctement :

* projet : il contient ses propres modules et produit un seul livrable (assembly, war, ear ou autre). Il peut éventuellement hériter d'un super pom.
* module : il s'agit d'un module maven ie. qu'il suit le même versionning que son père (pour rappel, les bonnes pratiques maven demandent à ce que les sous-modules d'un projet hérite de la version et du groupId du père). Le fait de builder le module parent build également les sous-modules. Concrètement, le père déclare ses modules fils avec l'élément module et le module fils déclare son père avec l'élément parent

En fait, pour moi, la seule chose qui va décider de découper un projet (au sens large du terme) en modules ou en différents projets se résume en 3 mots : cycle de vie :

* Si lors de la relivraison d'un sous composant tout le système doit être relivré car dépendant du premier, alors les composants sont dans le même projet, ie. qu'ils sont des modules. Ils héritent alors de la version du père et ne doivent pas être relivré indépendamment.
* S'il est possible de livrer indépendamment un composant du projet et que cela n'a pas foncièrement d'impact sur les autres, alors il doit être dans un projet indépendant. Les autres composants l'utilisant auront alors dans leur pom un numéro de version figée et le verrons comme une boite noire.

Bien sûr, le fait de gérer des projets indépendants complexifie le processus de livraison ainsi que l'usine logicielle (au nombre de jobs jenkins, sonar & co.) et c'est aussi pourquoi, souvent, les composants se retrouvent déclarer en modules d'un seul et unique projet.

Pourtant je pense que ce choix est une fausse bonne idée à terme et que la question du cycle de vie des différents composants est primordiale. En effet, découper en briques disposant de leur propre cycle de livraison force à découpler l'architecture du produit et oblige les équipes de développement à réfléchir sur le design.

De même, coté vision globale, cela force à réfléchir sur une cartographie du système (processus aussi appelé urbanisation) en se forçant à raisonner service (au sens large du terme) (ndlr : désolé mais pour ceux qui ne me connaissent pas, je viens du monde SOA... ).

Bien sûr, si le projet se limite à une simple application web disposant de son controleur, de sa couche présentation et de sa couche modèle, alors un "simple" projet disposant de ses sous-modules est largement suffisant, mais s'il s'agit de composants nécessaires à la gestion du SI au sens plus large, la question mérite à être posée.

Enfin, juste pour conclure cette partie, concernant le SCM, il doit être découpé en conséquence. Si c'est Git qui est utilisé, pas de souci. Par contre, si c'est SVN (ou autre), alors il est primodiale de respecter les préconisations de ce dernier, ie. de ne pas avoir tous les projets sous trunk mais que chaque projet dispose de ses sous répertoires trunk/branches/tags. En effet, un projet doit avoir sa propre hiérarchie de tags et de branches. De plus, techniquement parlant, il peut arriver que le fait d'utiliser le plugin release de maven produise des informations fausses au niveau de l'élément scm.

Exemple :

Le projet des décomposé comme suit :

![center](http://2.bp.blogspot.com/-tB997UpDKpA/UCvnxlx4G5I/AAAAAAAAAnc/9bD09UesmJ4/s1600/scm-projets2.png)

Alors les valeurs des éléments scm pour trunk seront les suivantes :

```text
projet1                          scm : trunk/projet1
      projet11                   scm : trunk/projet1/projet11
projet2                          scm : trunk/projet2
```

Par contre, si le plugin release est utilisée (par exemple le goal branches), on aura  la valeur de l'élément scm pour branches projet11 qui sera la suivante :

![center](http://1.bp.blogspot.com/-0Ha4zM3jOGU/UCvo1B6DknI/AAAAAAAAAnk/6CJBHQurR3c/s1600/scm-projets.png)

```text
projet11                  scm : branches/projet1/projet11
```

On constate que la transformation de la valeur de l'élément scm pour le projet11 est erronée. Cela n'influe en rien le livrable, cependant, il devient alors impossible de créer une branche ou de faire une release du projet11 avec le plugin maven release...

#Conclusion

Pour conclure cet article, on peut voir que le choix entre module et projet n'est pas si complexe que cela mais qu'il est préférable (et même, à mon sens, indispensable) de se poser les bonnes questions. Pour ma part, j'aurai tendance à dire que plus cela est tôt dans le projet, mieux c'est. Cependant, il faut se méfier de l'overdesign et trop de découpage peut tuer le découpage... Certaines personnes préfèrent, d'ailleurs, n'avoir qu'un seul projet décomposé en différent sous module puis, en fonction de l'avancée du projet, effectuer un découpage : c'est également une possibilité qui se veut aussi plus pragmatique.

Après chacun fait comme il l'entend et c'est une question de goût. Cependant, si on pousse ce raisonnement à l'extrème, pourquoi, dans ce cas, ne pas faire un découpage par package? : cela est plus simple, on ne s'embête pas à gérer différents poms et c'est encore plus pragmatique non? ;-)

Une fois encore, lecteur, je te laisse seul juge. A toi de te faire ta propre expérience et de juger le pour et le contre. Moi j'ai le mien ;-)
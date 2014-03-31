---
layout: post
title: "Changement de look et d'hébergeur"
date: 2014-03-28 10:00:03 +0100
comments: true
categories: 
- java
- divers
---

![left-small](http://blog.jetoile.fr/images/octopress/img_octopress_blogger.png)

Cet article n'a rien à voir avec Java et il est plus à titre informatif.

En effet, j'ai décidé de passer de __Blogger__ pour l'hébergement de mon blog à une solution basée sur [Octopress](http://octopress.org/) et un hébergement sur [github.io](http://pages.github.com/).

Pourquoi ce choix?

Et ben, en fait, pour plusieurs raisons :

* J'ai toujours aimé la philosophie de [LaTeX](http://www.latex-project.org/) qui permettait d'écrire au kilomètre sans avoir à se soucier de la mise en forme. 
* Mon blog était long à charger et la partie _tuning_ de trucs _front_ (et je ne le dis de manière non péjorative) n'étant pas mon fort, je n'ai pas réussi à l'alléger.
* La partie _syntaxhighlight_ était vraiment _chiante_ à gérer et alourdissait vraiment (mais alors _vraiment_!) le chargement des pages.
* J'avais envie d'une solution plus simple pour me permettre d'écrire mes articles (d'où mon [post sur livereload](http://blog.jetoile.fr/2014/03/livereload-et-linux-ou-comment-rediger.html)) et pas à avoir à faire de copier/coller + retouche dans [Blogger](http://blogger.com/) :
	* Ca correspond à la philo jrebel (_ndlr_ : que j'aime beaucoup) (cf. [ici](ihttp://blog.jetoile.fr/2010/02/jrebel-ou-comment-accelerer-le_24.html)) pour le code Java mais appliqué, ici, à la partie front.
	* C'est toujours plus marrant de pouvoir écrire un truc et de le voir se recharger automatiquement dans son navigateur - ça c'est pour la partie_geek ;-).
	* C'est _nouveau_ (enfin pour moi...).
* Blogger ne me convenait plus car il faisait trop de choses dont je ne me servais pas et, de plus, la façon dont il le faisait ne me convenait plus (son mécanisme de _templating_ est assez lourd).
* De nombreux blogs ont migrés et je trouve leur _look & feel_ plus proche de ce qu'on attend d'un blog actuellement sans le coté _bling bling_ de ce qui se faisait il y a quelques années.

<!-- more -->

Ce sont ces différentes raisons qui m'ont fait préférer :

* [Markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) pour la partie écriture pure (sachant que je m'étais déjà fait chier à faire marcher livereload avec...)
* [Octopress](http://octopress.org/) pour la génération : plus simple et sûrement (je n'ai pas fait de recherche précise) plus utilisé (donc documenté) que d'autres solutions comme [Jekyll](http://jekyllrb.com/) ou [Pelican](http://blog.getpelican.com/). En outre, Octopress permettait de manière très simple de générer le site en local pour voir le rendu.

Je ne l'avais pas fait à l'époque car je n'avais pas envie de m'embêter avec l'intégration de services pour, au final, avoir des fonctionnalités offertes en natif par Blogger. Cependant, le nombre de blogs migrant et l'adoption de solutions comme Jekyll et Octopress m'ont montré que, au final, c'était simple.

Je ne reviendrai pas sur la façon dont j'ai migré puisque de nombreux blogs en parle et que je n'ai fait que suivre leurs instructions.

Par contre, la partie _import_ ne m'ayant pas convaincue (pages générée crade), j'ai décigé de porter tous mes anciens articles avec Markdown. Il ne s'agit que de copier/coller + ré-application des styles, des liens et des images mais cela m'a demandé un temps considérable... (je suis sûr que si j'avais été malin, j'aurais faire le gros du boulot avec un bon `sed`... mais bon...). Du coup, j'espère qu'il n'y a pas de pertes...

~~Pour l'instant, je n'ai pas réussi à importer les commentaires via [Disqus](http://disqus.com/) (enfin l'import est fait mais les commentaires ne s'affichent pas... :'( ).~~
[_ndlr_ : après un coup de remapping des url des commentaires importés via __Disqus__, les commentaires sont revenus \o/]

~~La partie rss est un peu _buggé_ car mon flux _atom_ généré par Octopress est trop gros pour être utilisé par [feedburner](http://www.feedburner.com) (du coup, le flux n'est pas global mais seulement sur la catégorie `Java`).~~ [_ndlr_ : cf ce [post](/2014/03/changement-flux-rss.html) ]


Enfin, il faut encore que je change l'image de bannière ;-)

Pour info, l'ancien blog est toujours accessible à l'adresse http://jetoile.blogspot.com mais a été retiré de l'indexation google. Je ne porterai pas non plus les nouveaux articles sur ce dernier...

Voili voilou... j'espère que cette nouvelle disposition vous plait ;-)

Enjoy!

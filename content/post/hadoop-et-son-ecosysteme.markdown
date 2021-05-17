---
title: "Hadoop et son écosystème"
date: 2015-10-09 14:53:03 +0200
comments: true
tags: 
- hadoop
---
![left-small](/images/hadoop-all.png)
Avec l'arrivé de Apache Spark, Hadoop est souvent vu comme désuet et _legacy_. Il est vrai que le monde BigData est en perpétuelle évolution et qu'un produit peut être déprécié en quelques mois.

Cependant, restreindre le terme Hadoop aux seuls technologies MapReduce, HDFS et YARN est, pour moi, une erreur.

Déjà parce que ces technologies peuvent être décorrélées et ensuite car, souvent, la très grande majorité des nouvelles technologies issues du monde BigData s'appuient sur les couches existantes et s'intègrent avec ces dernières. 

Par exemple, plutôt que dire que Hadoop est mort et que Spark est son remplaçant, il serait plus juste de dire que l'écosystème Hadoop se voit rajouter le nouveau moteur d'exécution Spark (n'oublions pas que Spark s'intègre très bien avec HDFS en l'occurence pour la partie colocalisation des données/traitements ou même pour répondre aux besoins de _checkpointing_). 

Dans la présentation ci-dessous, j'ai tenté, de manière non exhaustive, de lister et regrouper par usage quelques unes des technologies que je considère faire partie de l'écosystème Hadoop et qui, de mon point de vue, constitue l'environnement Hadoop que certains nomment également _Data Platform_. 

<iframe src="//fr.slideshare.net/slideshow/embed_code/key/Bqq5ELZYyU3HSO" width="425" height="355" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//fr.slideshare.net/jetoile/hadoop-et-son-cosystme" title="Hadoop et son écosystème" target="_blank">Hadoop et son écosystème</a> </strong> from <strong><a href="//www.slideshare.net/jetoile" target="_blank">Khanh Maudoux</a></strong> </div>


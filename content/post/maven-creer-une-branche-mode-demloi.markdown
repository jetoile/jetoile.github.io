---
title: "Maven : créer une branche - Mode d'emploi"
date: 2012-08-13 10:40:31 +0100
comments: true
tags: 
- java
- maven
---

![left-small](http://1.bp.blogspot.com/-l-NZQRxE0vI/TevA5LfJQ_I/AAAAAAAAAXI/IP8zRk4wbeg/s1600/maven-logo-2.gif)

Il y a un moment déjà, j'avais fait un [article sur le plugin release](/2010/04/le-plugin-release-un-peu-demystifie.html) oh combien pratique dans notre utilisation de maven. J'y avais mentionné les 2 goals `prepare` et `perform` qui sont les plus connus et qui, pour rappel, permettent de :

* vérifier que lors du lancement du processus de génération du livrable, le code est bien conforme à ce qui se trouve sur le SCM,
* tagger le code source à partir duquel le livrable est produit et permettant ainsi sa reproductivité,
* incrémenter le numéro de version des poms,
* générer le livrable,
* et le déployer sur le _remote proxy repository_.

Dans cet article, je me concentrerai sur un goal du plugin release malheureusement assez méconnu mais qui est tout aussi pratique et important (à mon sens) que ses 2 comparses `prepare` et `perform`. Il s'agit du plugin __branch__.

<!-- more -->

Ce plugin permet de créer une branche en utilisant un sous-ensemble de ce que fait le goal prepare mais appliqué à une branche plutôt qu'à un tag :

* vérifie que lors du lancement du processus de création de branche, le code est bien conforme à ce qui se trouve sur le SCM (ie. qu'il n'y a pas de modification en local),
* modifie optionnellement la version du pom qui sera utilisé sur la branche créée (ie. cible),
* transforme les informations du SCM dans le pom (ie. le contenu des éléments fils de l'élément scm afin qu'ils correspondent à la branche),
* commite le pom modifié,
* tag le code dans le SCM comme une nouvelle branche,
* modifie optionnellement la version du pom si une nouvelle version doit être utilisée sur la branche/trunk/master source (enfin sur la code ayant servi à créer la branche),
* commite le pom modifié.

Pour les habitués de ce blog (ndlr : oui... j'avoue... un peu mort en ce moment... ), ils sauront que j'ai tendance à m'appuyer fortement sur la documentation officielle, chose que je ne changerai pas dans cet article.

Il est à noter que cet article n'est, en fait, qu'un prétexte afin de me servir d'aide mémoire sur la ligne de commande à utiliser pour effectuer cette opération... mais vu que je suis hyper sympa (... ou pas...), je développerai un peu plus que la simple commande à taper... ;-)

Enfin, je ne reviendrai pas sur le fait que créer une branche doit, bien évidemment, se faire dans les règles de l'art (ie. au moins pouvoir permettre une itération du numéro de version du pom dans le cas de branches correctives et le changement de la valeur des balises contenu dans l'élément scm).

#Les options

Les options les plus utiles (à mon sens) disponibles sont les suivantes :

* `branchName` : permet de définir le nom de la branche à créer.
* `autoVersionSubmodules` : permet d'utiliser automatiquement la même version que celle du parent. Si elle est mise à false (valeur par défaut), alors une version sera demandé pour chaque sous module.
* `branchBase` : permet de redéfinir le répertoire de base des branches dans SVN si les préconisations SVN n'ont pas été respectées.
* `developmentVersion` : permet de préciser la version à mettre pour la nouvelle copie local de travail.
* `dryRyun` : permet de faire un checkout local plutôt que de le faire sur le repository. Ainsi, par exemple, cette option peut être utile pour savoir si les modifications qui seront apportées aux pom qui seront poussés sur le SCM sont bien celles attendues.
* `localCheckout` : permet de faire un checkout local plutôt qu'un distant (utile seulement pour les SCM de type DVCS qui supportent le protocole file - git, jgit, hg - ).
* `pushChanges` : permet, seulement avec git, de pousser ou non ses changements sur le repository distant (par défaut, true).
* `updateBranchVersions` : permet de définir la version à mettre sur la branche (par défaut, false).
* `updateDependencies` : permet de mettre à jour la version du pom pour la prochaine phase de développement (par défaut, true).
* `updateWorkingCopyVersion` : permet de mettre à jour la version sur la copie de travail locale (par défault, true).

Enfin, il est possible de préciser la version à utiliser dans le ligne de commande. Cela peut être utile, par exemple, si la commande est exécutée en commande non interactive. Le goal branch peut utiliser les mêmes propriétés utilisées par le goal prepare pour définir les versions à utiliser.

Exemple :

```bash
mvn --batch-mode release:branch -DbranchName=my-branch-1.2 \
-Dproject.rel.org.myCompany:projectA=1.2 \
-Dproject.dev.org.myCompany:projectA=2.0-SNAPSHOT 
```

Dans cet exemple, le pom dans la nouvelle branche sera en version 1.2-SNAPSHOT alors que le pom local sera en version 2.0-SNAPSHOT.

#La commande! La commande!... enfin

Bon, voilà la commande tant attendue que j'utilise le plus souvent :

```bash
mvn --batch-mode release:branch \
-DbranchName=<nomBrancheACreer> \
-DupdateBranchVersions=true \
-DupdateWorkingCopyVersions=false \
-DautoVersionSubmodules=true \
-Dproject.rel.<groupId>:<artifactId>=<version> 
```
avec la déclaration suivante :

```xml
<plugin>
 <groupId>org.apache.maven.plugins</groupId>
 <artifactId>maven-release-plugin</artifactId>
 <version>2.3.2</version>
</plugin>
```

# Conclusion

En conclusion de ce court article, pas grand chose à ajouter sauf que le goal `branch` du plugin __release__ est vraiment très utile puisqu'il permet en une seule commande de faire plusieurs opérations qui se trouvent être assez rébarbatives.

# Pour aller plus loin...

http://maven.apache.org/plugins/maven-release-plugin/branch-mojo.html`

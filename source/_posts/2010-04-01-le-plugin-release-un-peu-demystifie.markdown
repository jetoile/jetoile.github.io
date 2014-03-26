---
layout: post
title: "Le plugin release (un peu) démystifié"
date: 2010-04-01 10:39:26 +0100
comments: true
categories: 
- java
- maven
---
![left](http://maven.apache.org/images/maven-logo-2.gif)
J'ai déjà longuement parlé de maven 2 dans des posts précédents ([Petites astuces avec maven 2](/2010/03/petites-astuces-avec-maven-2.html), [De l'art du livrable](/2010/02/de-l-du-livrable.html) et  [Retour sur la mise en œuvre d'un environnement de développement](/2010/01/retour-sur-la-mise-en-uvre-d_04.html)). Ici, je vais revenir sur le plugin [release](http://maven.apache.org/plugins/maven-release-plugin/) en version 2.0 en explicitant ce que font ses goals `prepare` et `perform` plus concrètement. 

En effet, il peut être utile de vouloir savoir quels sont les actions appelées et comment il est possible d'y ajouter son petit grain de sel... ;-)

Il est à noter que je ne reviendrai pas sur l'intérêt d'utiliser un tel plugin (ce n'est pas le sujet qui m'intéresse ici et d'autres blogs ou livres le feront mieux que moi) même si je ferai un petite piqure de rappel sur ce qu'il permet.

<!-- more -->

#Plugin release? Quoiqu'il fait?
Le principal intérêt du plugin release est de fournir un processus de livraison reproductible, maitrisé et auto-sufisant en permettant, entre autre, de :

* vérifier que lors du lancement du processus de génération du livrable, le code est bien conforme à ce qui se trouve sur le SCM,
* tagger le code source à partir duquel le livrable est produit et permettant ainsi sa reproductivité,
* incrémenter le numéro de version des poms,
* générer le livrable (ben oui... ;-) ), 
* et le déployer sur le remote proxy repositorie.

Pour se faire, il se décompose en 2 phases :

* la préparation du livrable (`mvn release:prepare`) qui :
	* vérifie que le code local ne diffère pas de ce qui se trouve sur le SCM
	* vérifie qu'il n'existe pas de dépendances en version SNAPSHOT (entrainant la non reproductivité du livrable)
	* demande à l'utilisateur d'indiquer le numéro de version du livrable à fournir, le nom du tag qui sera posé et la version du numéro de projet à utiliser à la suite de la génération du livrable
	* vérifie la conformité du code récupéré du SCM en lançant le goal maven `verify`
	* modifie les informations des pom du projet avec le numéro de version du livrable
	* commite le projet dans le SCM au tag indiqué
	* modifie les informations des pom du projet avec le numéro de version à utiliser à la suite de la génération du livrable
	* commite le projet dans le SCM
	* prépare la phase suivante en générant le fichier `release.properties`
* la génération du livrable (`mvn release:perform`) qui :
	* checkout le code source du SCM
	* génère le livrable, la javadoc et la génération des source puis les déploie dans le remote proxy repository maven

Il est également à noter qu'il est possible de l'utiliser pour créer des branches à partir d'un tag donné mais que ce point ne sera pas traité dans cet article.

#Et plus concrètement, comment qu'il marche?
En fait, le goal `prepare` du plugin release dispose d'une liste d'actions qui sont effectuées séquentiellement et où chaque action est associée à une classe se trouvant dans le package __org.apache.maven.shared.release.phase__ :

* __check-poms__ qui appelle la classe `CheckPomPhase` qui vérifie que le pom déclare bien la balise &lt;developerConnection&gt;, que le repository est bien déclaré et qu'il existe bien au moins un module du projet qui est en version SNAPSHOT, 
* scm-check-modifications qui appelle la classe `ScmCheckModificationsPhase` qui vérifie qu'il n'y a pas de modifications locales par rapport à ce qui se trouve sur le SCM,
* __check-dependency-snapshots__ qui appelle la classe `CheckDependencySnapshotsPhase` qui vérifie qu'il n'y a pas de dépendances en SNAPSHOT,
* __create-backup-poms__ qui appelle la classe `CreateBackupPomsPhase` qui créer une sauvegarde des fichiers pom (pom.xml.releaseBackup) après avoir éventuellement supprimé ceux existant,
map-release-versions qui appelle la classe `MapVersionsPhase` qui demande à l'utilisateur le numéro de version du projet à tagger,
* __input-variables__ qui appelle la classe `InputVariablesPhase` qui calcule, à partir de l'information entrée par l'utilisateur dans la phase précédente, le label à poser et qui la propose à l'utilisateur en lui demandant si cela lui convient. Dans le cas contraire, il peut saisir le tag à poser,
map-development-versions qui appelle la classe `MapVersionsPhase` qui demande à l'utilisateur le numéro de version à mettre dans les pom pour le trunk ou la branche courante suite à la phase prepare,
rewrite-poms-for-release qui appelle la classe `RewritePomsForReleasePhase` qui réécrit les poms à la version qui sera taggée,
* __generate-release-poms__ qui appelle la classe `GenerateReleasePomsPhase`,
* __run-preparation-goals__ qui appelle la classe `RunPrepareGoalsPhase` qui appelle les goals maven par défaut (`clean` et `verify`) pour vérifier le projet,
* __scm-commit-release__ qui appelle la classe `ScmCommitPhase` qui commite le projet dans le SCM,
* __scm-tag__ qui appelle la classe ScmTagPhase qui pose le tag indiqué,
* __rewrite-poms-for-development__ qui appelle la classe `RewritePomsForDevelopmentPhase` qui réécrit les poms avec la version suivante de développement,
* __remove-release-poms__ qui appelle la classe `RemoveReleasePomsPhase`,
* __scm-commit-development__ qui appelle la classe `ScmCommitPhase` qui commite le projet dans le SCM,
* __end-release__ qui appelle la classe `EndReleasePhase` qui finalise le processus en indiquant dans le fichier `release.properties` que la dernière phase du cycle s'est bien déroulée.

Il est à noter également que chaque action peuple le fichier `release.properties` qui indique, entre autre, la dernière action qui s'est déroulée sans erreur. 

Le goal `perform` dispose, quant à lui, de la liste d'actions suivantes : 

* __verify-completed-prepare-phases__ qui appelle la classe `CheckCompletedPreparePhasesPhase` qui vérifie dans le fichier `release.properties` que la phase `prepare` s'est bien déroulée (action effectuée par l'action end-release du goal `prepare`),
* __checkout-project-from-scm__ qui appelle la classe `CheckoutProjectFromScm` qui checkout le projet du SCM dans le répertoire checkout de target,
* __run-perform-goals__ qui appelle la classe `RunPerformGoalsPhase` qui exécute le goal maven `deploy` pour déployer le projet dans le repository maven.

#et sinon, comment qu'on l'utilise?
On a donc vu ce que faisait concrètement le plugin release. Cependant, il peut parfois être utile de redéfinir certaines de ses actions, soit en précisant le profile à utiliser lors de la phase de déploiement (si le plugin [assembly](http://maven.apache.org/plugins/maven-assembly-plugin/) est utilisé par exemple), soit si l'environnement où est effectué le livrable dispose d'une configuration folklorique (oui, oui, c'est possible... :( ).

En fait, il est possible de préciser les goals à exécuter lors de l'action run-preparation-goal de la goal `prepare` (qui, pour rappel, exécute par défaut les goals `clean` et `verify`) en configurant le plugin release via la balise `<preparationGoals>` :

exemple :
```xml
<plugin>
 <groupId>org.apache.maven.plugins</groupId>
 <artifactId>maven-release-plugin</artifactId>
 <version>2.0</version>
  <configuration>
   <preparationGoals>clean verify -Plivraison</preparationGoals>
   <!-- clean et verify sont les goals de preparations par defaut -->
  </configuration>
</plugin>
```
De même, il est également possible de préciser le profile à utiliser lors de l'action run-perform-goals de la phase `perform` via la balise.

exemple : 

```xml
<plugin>
 <groupId>org.apache.maven.plugins</groupId>
 <artifactId>maven-release-plugin</artifactId>
 <version>2.0</version>
  <configuration>
    <releaseProfiles>livraison</releaseProfiles>
    <!-- during release:perform, enable livraison profile -->
  </configuration>
</plugin>
```
En outre, si les sources du projet ainsi que la javadoc ne sont pas désirés dans la phase de déploiement (par exemple à cause de ça), il est possible de positionner `useReleaseProfile` à `false` :
```bash
mvn release:perform -DuseReleaseProfile=fals
```
ou en le configurant dans le pom :
```xml
<plugin>
 <groupId>org.apache.maven.plugins</groupId>
 <artifactId>maven-release-plugin</artifactId>
 <version>2.0</version>
  <configuration>
    <useReleaseProfile>false</useReleaseProfile>
  </configuration>
</plugin>
```

#Conclusion
Vous l'aurez donc compris, le maven-release-plugin est vraiment un outil puissant qui effectue un certain nombre d'opérations prédéfinis et qui sont généralement faites manuellement par l'utilisateur... Ainsi, il permet de gagner beaucoup de temps.

Attention toutefois à ne pas oublier ce qu'il fait. Cela évitera les petites surprises (enfin c'est le cas de tous les outils...) et permettra de le configurer aux petits oignons (par exemple, en utilisant les options disponibles ou un fichier de configuration pour éviter d'avoir un prompt de sa part)... ;-)

A noter que certains points n'ont pas été traités dans cet article puisque le plugin release permet également d'effectuer, entre autre, :

* un rollback en cas, soit d'échec de génération, soit d'anomalies détectées dans le livrable,
* la création de branches permettant ainsi de produire des correctifs sur une version déjà livrée.

#Pour aller plus loin...
* __Apache Maven__ de N. De Loof, A. Héritier chez Pearson
* Better Builds with Maven : http://repo.exist.com/dist/maestro/1.7.0/BetterBuildsWithMaven.pdf
* Maven. Definitive Guide : http://www.sonatype.com/products/maven/documentation/book-defguide
* Site de maven : http://maven.apache.org/
* Site du plugin release : http://maven.apache.org/plugins/maven-release-plugin/

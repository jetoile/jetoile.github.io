---
layout: post
title: "Les repositories Manager pour les nuls... focus sur Nexus"
date: 2011-05-20 19:50:24 +0100
comments: true
categories: 
- maven
- nexus
---
![left-small](http://1.bp.blogspot.com/-E8NziC8CXkk/TdY_sPIbLmI/AAAAAAAAAWY/kSnZW5IE2_g/s1600/nexus01.png)

Cela fait un moment que je n'ai pas bloggé faute de temps mais également d'inspiration... (ça c'est dit et ça, on s'en fout d'un coté ;-) ) Cet article parlera des repository manager. Rien de nouveau à l'horizon puisque la problématique et les solutions existent depuis un moment mais ayant dû en mettre un en place récemment et au vu de nombreuses questions que l'on m'a posées, je vais ici coucher sur "papier" ce qu'est un repository manager, c'est à dire quelques uns de ses concepts clés. En outre, cela me servira également d'aide mémoire et je me dis qu'une piqûre de rappel ne fait jamais de mal... ;-)

Pour ce faire, je vais m'appuyer sur l'excellent repository manager [Nexus de Sonatype](http://www.sonatype.com/products.html) et plus particulièrement (comme à mon habitude... ;-) ) sur [sa documentation officielle](http://www.sonatype.com/repository-management-with-nexus-book.html).

Par contre, je ne reviendrai pas sur les concepts de base de maven comme la définition de ce que sont les repositories local et distant.

Enfin, ici, j'utiliserai la même dénomination que le guide Nexus pour le terme organisation qui devra être pris au sens large du terme (entreprise, projet open source, poste local, ...).

<!-- more -->

#Un repository Manager pour quoi faire?

Ce paragraphe fournit une courte introduction sur les repository Manager, qui pour ceux qui en douteraient encore, permet de répondre à deux problématiques :

* il agit comme un proxy configurable entre votre organisation et les repositories publics maven,
* il permet d'organiser l'endroit où se feront les déploiements des projets maven spécifiques à l'organisation.

En fait, ils sont le pendant de ce que sont les outils SCM (_Source Code Management_) au code source mais appliqué aux dépendances et aux artéfacts maven générés par le build.

Aussi, un repository Manager est un élément clé d'une usine logicielle en permettant, entre autre, une meilleure collaboration entre les développeurs et la mise à disposition de leurs binaires.

En effet, fournir un proxy (faisant également office de cache) à un repository maven public distant permet d'accélérer le processus de build en évitant les téléchargements inutiles sur internet : si une dépendance particulière est nécessaire (ainsi que ses dépendances transitives), elle ne sera téléchargée qu'une et une seule fois du repository public distant et sera mise à disposition de tous les développeurs. En outre, si un projet dépend de dépendances snapshot, maven, par défaut, tente d'accéder, à chaque build, aux repositories distants afin de vérifier s'il n'existe pas une version plus récente que celle présente dans le repository local. Le fait d'avoir un repository manager permet de scheduler ces mises à jour en fonction du besoin afin d'éviter un traffic trop important et parfois inutile.

De plus, un repository Manager permet à l'organisation de garder le contrôle de ce qui est téléchargé par maven empêchant ainsi à nos chers développeurs (dont je fais parti) de tirer n'importe quelle dépendance (ainsi que n'importe quelle version) dans son projet mais également de contrôler les licences des dépendances utilisés (c'est vrai que cela serait dommage de voir son projet tomber dans l'open source juste à cause d'une licence un peu trop contaminatrice...).

Enfin, un repository manager permet de fournir un repository partagé par tout ou parti de l'organisation offrant ainsi un repository privé où peuvent être entreposés les binaires de cette dernière, les rendant ainsi accessibles à tout ou parti des équipes de développement. Ces binaires pouvant être dans un état stable (release) ou en cours de développement (snapshot) permettent ainsi aux développeurs de ne pas à avoir à récupérer et à builder les sources lorsqu'une dépendance à une librairie interne est nécessaire.

Dernier point qui peut avoir son importance, mais si l'organisation se retrouve coupé d'internet ou que les repositories publics maven sont indisponibles, cela n'impacte pas les développeurs (modulo le fait qu'ils n'aient pas besoin d'une dépendance ne se trouvant pas sur le repository manager) qui peuvent alors continuer à travailler et même à releaser leurs projets, et cela même s'ils disposent d'un repository local vierge.

#Retour sur les types de repositories possibles

Cet article prenant ses racines dans la documentation officielle de Nexus, il est possible que certains types de repositories présentés ci-dessous ne soient pas offerts par d'autres implémentations. Cependant, les concepts sont similaires même si certaines subtilités peuvent être présentes dans d'autres solutions.

Nexus offre 3 types de repositories :
![center](http://1.bp.blogspot.com/-Z4HaCDB8Eho/TdZAASMnH8I/AAAAAAAAAWc/96XhX5UdsQQ/s1600/nexus05.png)

* __Proxy Repository__ qui, comme son nom l'indique peut être vu comme le proxy d'un repository distant. Par défaut, Nexus vient configuré avec les proxies suivants : 
	* apache-snapshots (http://people.apache.org/repo/m2-snapshot-repository), 
	* codehaus-snapshot (http://snapshots.repository.codehaus.org/),  
	* et maven-central-repo (http://repo1.maven.org/maven2/)
* __Hosted Repository__ qui sont des repositories hébergés par Nexus. Par défaut, Nexus vient configuré avec les hosted repositories suivants : 
	* 3rd Party devant être utilisé pour des librairies non présentes dans les repositories maven publics
	* Releases devant être utilisé pour les librairies stable de l'organisation 
	* Snapshots devant être utilisé pour les librairies en cours de développement de l'organisation
* __Virtual Repository__ qui peut être vu comme un adaptateur de repositories au format différents que ceux attendus. Par défaut, Nexus vient configuré avec 2 virtual repository permettant la conversion de repository au format maven 1 et 2 vers (respectivement) des repositories au format maven 2 et maven 1.

![meidum](http://3.bp.blogspot.com/-2j9iQR-sn4Q/TdZAGVFRyiI/AAAAAAAAAWg/4D9WVWTH-38/s1600/nexus02.png)

En outre, Nexus propose la notion de __groupe__ qui permet de combiner de multiples repositories sous la même url d'accès. Ainsi, cela permet d'alléger la configuration du fichier settings.xml en offrant, sous la houlette d'un groupe, l'aggrégation de plusieurs repositories pouvant être de différentes natures.

Par exemple, Nexus vient configuré avec les deux groupes suivants :

* _public_ qui regroupe les repositories 3rd Party, releases et snapshots du repository maven-central-repo,

![center](http://1.bp.blogspot.com/-DIrq1MhM_oQ/TdZAOhG1WtI/AAAAAAAAAWk/hKnBf0lI2mo/s1600/nexus03.png)

* et _public-snapshots_ qui regroupe les repositories apache snapshots et codehaus snapshot.

![center](http://1.bp.blogspot.com/-AMfb7cXEBPg/TdZAUedthTI/AAAAAAAAAWo/8GTVLZCGYys/s1600/nexus04.png)

Il est à noter que l'ordre de déclaration des repositories dans un groupe a son importance puisque la recherche est ordonnée et que dès que la librairies est trouvée, la recherche s'arrête et l'artéfact rendu (ainsi, il convient de mettre le repository succeptible de posséder les librairies les plus recherchés en premier).

En outre, il est possible d'associer à un groupe des routes qui se comportent comme des filtres permettant ainsi d'inclure ou d'exclure des repositories. Cette fonctionnalité peut, par exemple, être utilisée afin d'être sûr que les artéfacts récupérés dans un groupe donné ne sont que ceux de l'organisation (en s'appuyant sur le groupId maven).

![center](http://2.bp.blogspot.com/-AxzBLRMUcQA/TdZAcBJX_oI/AAAAAAAAAWs/btqGC8lg5r8/s1600/nexus06.png)

#Gestion des repositories dans le cas où plusieurs équipes accèdent au même repository Manager

Ces recommandations s'appuient sur le chapitre 17 du guide sur Nexus.
Les façons les plus communes pour supporter différentes équipes est :

* d'associer spécifiquement à chaque équipe/projet deux repositories (un repository release et un repository snapshot) en leurs associant les bons droits.
* d'avoir un ou des ensembles de deux repositories (release et snapshot) pour l'ensemble de l'organisation et de contrôler l'accès en utilisant des repository target dans la gestion des privilèges d'accès par un rôle aux repositories. Ce sont ces rôles qui peuvent ensuite être associés aux utilisateurs pour gérer la sécurité.

![medium](http://2.bp.blogspot.com/-bK8Hh_gtn-c/TdZAqwhpudI/AAAAAAAAAWw/cCWKhZDbaTI/s1600/nexus07.png)

<br/>
![medium](http://1.bp.blogspot.com/-z7V2gLhRfqM/TdZA0XKmyjI/AAAAAAAAAW0/tqsv5yFtymc/s1600/nexus08.png)

<br/>
![medium](http://3.bp.blogspot.com/-mQZmpHgoHkw/TdZA7Xrs5oI/AAAAAAAAAW4/2LplkYKr-1Y/s1600/nexus09.png)

<br/>
![medium](http://4.bp.blogspot.com/-pOaBDAUQ5v0/TdZBAv3nkrI/AAAAAAAAAW8/kpJampXGKe0/s1600/nexus10.png)

#Trucs et astuces en vrac...

Enfin, ce paragraphe liste en vrac un certain nombre de points qu'il peut être nécessaire de prendre en compte lors de la mise en oeuvre d'un repository Manager.

Ayant la flemme de commenter les différents points, je ne jetterai que les idées sans ordre de préférence en laissant le soin à chacun de les prendre en compte... ou pas... ;-)

* Utilisation et mise en oeuvre de repository de Staging.
* Attention au distributionManagement se trouvant dans le pom parent (ce qui est une bonne pratique malgré tout) : il peut être nécessaire de positionner cette information dans le fichier settings.xml via des propriétés afin de permettre une éventuelle migration du repository Manager sur un autre serveur (et s'il n'est pas possible d'avoir un DNS) et cela sans avoir à maintenir à tout jamais une redirection apache (cf. bonnes pratiques maven) (http://maven.40175.n5.nabble.com/Can-t-specify-distributionManagement-in-settings-xml-td3181781.html).
* Le repository Manager étant un élément clé de l'usine logicielle au même titre qu'un annuaire d'entreprise pour une entreprise, il peut s'apparenter à un Single Point Of Failure (même s'il est possible de le mirrorer). Aussi, il convient de superviser le serveur avec soin (CPU, RAM, IO, espace disque, ...) et si possible de posséder une machine dédiée à cette tache. En outre, il convient de faire attention, si une VM est utilisée, de lui affecter suffisamment de ressources qu'elles soient réseau ou autre. 
* Mettre en place un Repository Manager, même s'il s'agit d'une tache aisée, peut demander du temps mais également quelques connaissances de base. Aussi, il ne faut pas prendre cette tache à la légère, en particulier concernant les choix d'organisation des différents repositories ainsi que des choix de sécurité. En effet, migrer d'une solution à une autre, même si cela est généralement possible, peut avoir des conséquences sur l'ensemble du parc informatique de développement.
* Un repository Manager (ou plutôt ses hosted repositories) contenant les binaires et éventuellement les livrables des projets, il est important de le sauvegarder avec soin.
* Il convient de réfléchir aux besoins de mettre en place une authentification des différents repositories et, dans ce cas, du besoin de la coupler à un annuaire. 
* Il convient de faire attention à l'ordre de résolution des différents repositories au risque de diminuer les performances de recherche et de téléchargement des différentes librairies.
* Il convient de bien configurer les différentes taches d'administration et de maintenance du repository Manager (reindéxation, suppression des snaphots obsolètes, ...).

![medium](http://1.bp.blogspot.com/-q-ibFGeayTU/TdZBL9AfMxI/AAAAAAAAAXA/4AZv5qunm_A/s1600/nexus11.png)

Enfin un point transverse, mais si la sécurité est mise en place sur les repositories et qu'elle utilise une authentification pour chaque utilisateur, il est possible de crypter les mots de passe plutôt que de les avoir en clair dans le fichier settings.xml (http://sonatype.com/books/maven-book/reference/appendix-settings-sect-encrypting-passwords.html et http://maven.apache.org/guides/mini/guide-encryption.html).

#Conclusion

Vous aurez compris (si vous avez lu juste qu'au bout cet article) que cet article correspond plus à un aide-mémoire qu'à une vrai explication de ce qu'est un repository Manager... ;-)

Concernant les trucs et astuces, si vous en avez d'autre, je suis preneur et je me ferai un plaisir les intégrer dans la rubrique trucs et astuces ;-)

#Pour aller plus loin...

* Pdf de Sonatype sur Nexus : http://www.sonatype.com/repository-management-with-nexus-book.html
* Apache Maven de N. De Loof, A. Héritier chez Pearson
* Better Builds with Maven : http://repo.exist.com/dist/maestro/1.7.0/BetterBuildsWithMaven.pdf
* Maven. Definitive Guide : http://www.sonatype.com/products/maven/documentation/book-defguide
* Site de maven : http://maven.apache.org/
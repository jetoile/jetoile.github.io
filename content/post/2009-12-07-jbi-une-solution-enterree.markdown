---
title: "JBI : Qu'est ce que c'est?"
date: 2009-12-07 09:58:58 +0100
comments: true
sharing: true
footer: true
tags: 
- java
- jbi
- soa
---

![left](http://4.bp.blogspot.com/_XLL8sJPQ97g/SyioXBC9u0I/AAAAAAAAAII/DM5NReY1mJ4/s200/jbi.png)
Voilà mon premier article. Il a pour objectif de présenter JBI (_Java Business Service_) aussi connue sous le doux nom de JSR 208 (_Java Specification Release_).

# Pourquoi un article sur JBI?
Vous pouvez vous demander pourquoi un article sur JBI? Les réponses sont simples : j'aime bien cette spécification et je trouve qu'elle a du potentiel et qu'elle est sous exploitée.

Bien sûr, me direz-vous, en tant qu'utilisateur d'un PEtALS ou ServiceMix, pourquoi devrais-je comprendre la technologie sous-jacente? Eh bien, c'est une affaire de goût : si vous préférez utiliser un produit comme une boite noire, passer votre chemin. Par contre, si vous voulez comprendre ce qui se passe à l'intérieur, j'espère que cet article pourra répondre à vos attentes.

Autre interrogation : pourquoi m'embêter à comprendre une technologie qui ne décolle pas (du moins de ma fenêtre) et sur laquelle les éditeurs ne communiquent pas énormément (c'est vrai qu'ils préfèrent communiquer sur ce qu'ils offrent et non comment ils le font) ? Là encore, question de goût et puis qui croyait en l'émergence du téléphone portable ou même de la télé réalité ? (personnellement, je ne croyais ni en l'un ni en l'autre...) cqfd... ;-)

Ce premier article a donc pour objectif de présenter JBI. Seront abordés ses concepts généraux avec une approche telle que celle décrite dans ses spécifications ainsi qu'un succinct retour sur ce que je pense de JBI..

Cependant, ne seront pas abordés, ici, les mécanismes utilisés en son sein ainsi que des notions de SOA (de très bon article abordent déjà SOA) même si, qui dit JBI dit SOA (la réciproque n'est pas foncièrement vrai...).

Il est à noter que dans cet article j'utiliserai souvent le terme de composant (peut être à mauvais escient) au lieu du terme service. Il s'agit d'un abus de langage lié au fait que j'ai tenté d'être le plus clair possible et que en évitant d'aborder trop de notions afin d'éviter de vous perdre ;-).
<!-- more -->
# JBI : Qu'est ce que c'est?

JBI (_Java Business Integration_) est définie dans sa première version par la JSR (_Java Specification Release_) 208. Elle a pour objectif de proposer une solution d'intégration de composants métiers et définie une norme pour l'assemblage des moyens d'intégration de composants.

JBI peut être vue comme le conteneur de service des ESB (_Enterprise Service Bus_)
>JBI defines what is sometimes termed a “service container” in Enterprise Service Bus (ESB) systems(...). - JBI_Spec

et aborde l'intégration de composants métiers d'un point de vue infrastructure.

Il s'agit d'une spécification et doit donc être implémentée par ce que je nommerai dans le reste de cet article un JBI Provider.

Elle peut être vue comme une solution concurrente de SCA (_Service Component Architecture_), cependant, il n'en est rien : alors que SCA aborde l'intégration de composants d'un point de vue applicatif, JBI l'arborde d'un point de vue infrastructure. Cela en fait donc un très bon candidat pour héberger une solution basée sur SCA. Ces deux solutions doivent donc être vue comme complémentaires et non comme concurrentes.

JBI s'appuie sur des technologies web comme les WSDL (_Web Service Description Language_) pour la description des composants (nous verrons plus tard dans que cette notion de composant se trouve une notion de service) ou le XML (_eXtensible Markup Language_) pour l'échange des messages.

Si je devais définir JBI en une phrase, je dirais qu'il s'agit d'une spécification visant à définir un conteneur de conteneurs pour offrir une solution normalisée pour l'intégration de composants métiers. Les conteneurs étant des composants qui hébergent les services que nous verront plus tard.

# Concepts

Cette partie tentera d'expliquer ses concepts qui, je pense, sont nécessaires pour comprendre son fonctionnement et utiliser une solution basée sur JBI. 

## Fonctionnement général

Un environnement JBI expose des services qui interagisent entre eux via un échange asynchrone de message au format XML. Cela permet un couplage faible entre les services. De plus, afin de découpler encore plus l'échange de messages, c'est l'environnement qui est chargé de les router. En fait, les services sont hébergés au sein de composants, mais c'est l'objet des paragraphes suivants... ;-)

## Notion de composants JBI

JBI définit deux types de composants :

* Les Service Engine (SE) qui sont des composants qui fournissent la logique métier et des services de transformation aux autres composants. Ils peuvent également être consommateur (nous reviendront sur la notion de consommateurs et de producteurs) d’autres SEs.
* Les Binding Component (BC) qui sont des composants qui assurent la connectivité pour les services extérieurs à l’environnement JBI. Cela peut impliquer les protocoles de communication ou des services fournis par le SI de l’entreprise.

Il est à noter que dans la première version de JBI, la différence entre les SE et les BC est purement conceptuelle puisqu'il n'existe pas de différence de traitement entre l'un ou l'autre 
>Service engines and binding components can function as service providers, consumers, or both. (The distinction between SEs and BCs is purely pragmatic, but is based on sound architectural principles. The separation of business (and processing) logical from communications logic reduces implementation complexity, and increases flexibility.) - JBI_Spec.

Afin de clarifier la notion de composant (SE ou BC), voici une liste de composants (cette liste n'est pas exhaustive et est spécifique au JBI Provider : rien n'est dit à ce propos dans la spécification) :

* exemple de SE :
	* SE XSLT (_eXtensible Stylesheet Language Transformations_) qui permet d’appliquer une transformation d’un fichier/message XML à partir d’une feuille de style XSL,
	* SE Validator qui permet de valider un message grâce à son XSD,
	* SE CSV (_Comma-Separated Value_) qui permet de transformer un document au format CSV en document au format XML,
	* SE EIP (_Enterprise Integration Pattern_) qui permet d’implémenter certains patrons d’intégration d’entreprise,
	* SE JSR181 (_Web Services Metadata for the java Platform_) qui permet d’exposer un POJO (Plain Old Java Object) annoté comme un service JBI,
	* SE POJO (_Plain Old Java Object_) qui permet de déployer des classes Java comme des services,
	* SE BPEL (_Business Process Execution Language_) qui permet d'offrir un moteur d'exécution BPEL,
	* SE SCA (_Service Component Architecture_) qui permet d'offrir une implémentation de SCA,
	* ...

* exemple de BC :
	* BC FileTransfert qui permet d’envoyer et de recevoir des fichiers,
	* BC FTP (_File Transfert Protocol_) qui permet le transfert et le listage de fichiers sur un serveur ftp,
	* BC JMS (_Java Message Service_) qui permet de s’interconnecter avec des Destinations (Queue/Topic) extérieures,
	* BC SMTP qui permet d’émettre et de recevoir des fichiers d’un service de mail extérieur,
	* BC SOAP (_Simple Object Access Protocol_) qui permet de s’interconnecter avec un service Web extérieur afin d’exposer des services JBI comme des services Web,
	* BC XMPP (_eXtensible Messaging and Presence Protocol_) qui permet de s’interconnecter avec le protocole XMPP qui est un protocole standard de messagerie instantané (Google Talk, Jabber, …),
	* ...

Un composant JBI se présente sous forme de fichier zippé et doit être installé dans l'environnement JBI (en fait, dans ce fichier zip, des informations contenues dans le fichier jbi.xml se trouvant dans le répertoire META-INF spécifie le composant -cycle de vie & co. -).

## Notion de Service Unit et Service Endpoint
Les composants JBI (SE ou BC) offrent un comportement général et leur utilisation est spécifique au besoin : par exemple, la réception de Messages JMS ne se fait pas toujours sur la même Destination et cela doit être configurable. De même, un fichier XML est considéré comme valide via son schéma XSD. Aussi il est nécessaire de configurer ces composants. C'est pour cette raison qu'ils peuvent être vus comme des conteneurs. Cette étape où à un composant générique (comme un BC File System) est associé sa spécificité (dire au BC File System qu'il doit scruter un répertoire particulier) est appelée configuration du composant. C'est suite à sa configuration qu'un composant expose un service en tant que tel.

De plus, il ne faut pas oublier que le but premier de JBI est d'offrir des possibilités d'assemblage : l'environnement JBI offre un certain nombre de "meta-services" au reste du monde qui est un assemblage d'autres services qui sont hébergés en son sein ou non. Pour ce faire, une notion de services fournisseurs (provider) et de services consommateurs (consumer) a été introduite :
* un service provider permet d'intégrer un service dans l’environnement JBI
* un service consumer permetd'exposer un service dans l’environnement JBI
Cette spécificité du service est également faite lors de la phase de configuration du composant.

En fait, la notion de fournisseur et de consommateur doit être vu du point de vue de l'environnement JBI. Par exemple, afin d'offrir un service disponible qui écoute sur la Queue JMS "myQueue", il faut donc configurer le composant BC JMS (dans le cas du JBI Provider PEtALS) comme consommateur en lui fournissant le nom de la Queue qu'il écoute. De même, si un service doit être utilisé par un autre service de l'environnement JBI, il doit être défini comme fournisseur (en fait, plus techniquement, un service fournisseur est enregistré dans l'annuaire interne de l'environnement JBI et peut ainsi être utilisable par d'autres services alors qu'un service consommateur n'est pas accessible à d'autres services de l'environnement JBI).

Cette phase de configuration est faite via un _Service Unit_ (SU). Ainsi, pour résumer, un composant JBI dans l'environnement JBI ne fait rien et doit être configuré via un ou des SU pour exposer un ou des services. C'est pour cette raison que j'ai préalablement définie JBI comme un conteneur de conteneurs : un composant JBI peut être vu comme un conteneur de service (un peu comme un conteneur de servlet qui, en soit, ne fait rien mais dans lequel des servlets doivent être déployées afin qu'il offre une réelle utilitée).

Cependant, une autre question peut se poser : comment sont accessibles les composants JBI configurés comme provider... ? En fait, le SU permet de définir le endpoint (appelé ServiceEndpoint) par lequel le service exposé par le composant sera accessible. Inversement, un composant JBI configuré comme consumer doit préciser avec quel(s) service(s) il interagit (n'oublions pas que les services ont besoin de communiquer entre eux). Cela est fait également via les endpoints.

Plus concrètement, un SU se présente sous forme d'un fichier zippé et doit être déployé sur un composant JBI (en fait, dans ce fichier zip doit se trouver un fichier jbi.xml dans le répertoire META-INF qui précise s'il est fournisseur -et dans ce cas, son endoint- ou s'il est consommateur -et dans ce cas, le ou les endpoint avec le(s)quel il interagit-). Les autres informations nécessaires à la configuration du service doivent également être présentent dans le SU : un SU permet donc de configurer un composant (SE ou BC) afin de lui fournir une logique de fournisseur ou de consommateur ainsi que son ServiceEndpoint associé. Il permet également de fournir d’autres informations au composant comme le répertoire d’écoute (dans le cas d’un BC consommateur) ou d’écriture (dans le cas d’un BC fournisseur) pour le BC transfert de fichiers (BC permettant de scruter un répertoire pour lire les fichiers qui y sont déposés ou permettant d’écrire dans ce répertoire).

# Notion de Service Assembly
Dans les paragraphes précédents, ont été introduit les notions de componsants JBI, de services (au sens JBI) et de ServiceEndpoint : il a été vu qu'un composant doit être configuré par un SU qui lui offre la notion de service via la déclaration (entre autres) de ses endpoint.

En fait, pour déployer un SU dans l’environnement JBI, il faut utiliser un SA (_Service Assembly_) dans lequel est indiqué quel(s) SU(s) configure(nt) un composant donné. Ce SA contient également les SU (un SU ne peut être déployé directement et doit l'être au travers d'un SA).

# Notion de Normalized Message Router
En introduction, il a été dit que c'était l'environnement JBI qui avait à sa charge l'émission des messages transitant entre services. En fait, c'est le routeur de messages normalisés (_Normalized Message Router_) qui reçoit les messages échangés et qui les redirige vers leurs destinataires respectifs. C'est lui qui permet de découpler les fournisseurs de service et les consommateurs de service.

En outre, il est essentiel de noter que dans les spécifications il est dit que quand un message échangé est routé par le fournisseur de service, il doit toujours être renvoyé au composant initiateur de l’échange : 
>"When a message exchange is routed from the service provider, it MUST always be sent to the component that originated the exchange. Subsequent sending of the exchange from the consumer to provider MUST always send the exchange to the provider endpoint selected during the initial send of the exchange" - JBI_Spec

Cela implique qu'il n'est pas possible de chainer directement les appels entre les services (en fait, techniquement, cela est possible mais les composants deviendraient alors fortement couplé ce qui est contraire à la philosophie JBI et SOA) : un composant (qui est alors un SE) doit donc être garant de la médiation des messages.

Ce composant JBI chargé d'offrir la notion de médiation entre les messages (cela sort du périmètre des spécifications JBI) est généralement fourni par les JBI Provider et est implémenté de manière spécifique :
* le JBI Provider PEtALS de EBM Websourcing propose son propre composant SE EIP,
* et les JBI Provider OpenESB de Sun et ServiceMix d'apache utilise un SE basé sur apache Camel.

En fait, les SE EIP ou le SE apache Camel s'appuient sur les EIP (_Enterprise Integration Patterns_) qui sont des designs pattern pour tous ce qui est l'échange de message asynchrone (un article sera fait ultérieurement sur ce point).

De plus, en fonction du type de messages, le Normalized Message Router (NMR) offre une gestion de qualité de service variable :
* stratégie meilleur effort : les messages peuvent être perdus ou délivrés plus d’une fois,
* stratégie au moins un : les messages peuvent être perdus mais ne peuvent pas être délivrés plus d’une fois,
* stratégie un et un seul : les messages sont assurés d’être délivrés une et une seule fois.

Ainsi, pour résumer, on peut dire que le NMR joue le rôle de médiateur dans l'environnement JBI (en fait, dans l’environnement JBI, les messages sont normalisés et encapsulés par les composants dans un MessageExchange (ME) contenant des propriétés (metadata) afin d’être envoyés d’un composant à un autre. Les BCs et SEs interagissent avec le routeur de messages normalisés via un DeliveryChannel qui est bidirectionnel. En outre, il est possible d’associer des pièces jointes aux MEs).

## Message Exchange Patterns (MEP)
Un JBI Provider permet d'émettre et de recevoir des messages (ME) entre les divers services qu'il héberge au travers de composants. En fait, ces transmissions de messages suivent certains patterns qui sont définis au nombre de 4 (en fait, ce nombre n'est pas imposé et chaque JBI Provider est libre d'un offrir plus) :

* In-Only

![center](http://3.bp.blogspot.com/_XLL8sJPQ97g/SxzF-Wzfz_I/AAAAAAAAAEc/2_KfWG0qdqY/s400/Image1.png)

* In-Out

![center](http://4.bp.blogspot.com/_XLL8sJPQ97g/SxzGH7vqLXI/AAAAAAAAAEs/6Hv5OsZS3pI/s400/Image3.png)

* In-Optional Out

![center](http://3.bp.blogspot.com/_XLL8sJPQ97g/SxzGO2sFd7I/AAAAAAAAAE0/xjnLA3PIHjY/s400/Image4.png)

* Robust In-Only

![center](http://3.bp.blogspot.com/_XLL8sJPQ97g/SxzGDeqLnjI/AAAAAAAAAEk/wRFwV4IanRM/s400/Image2.png)

En fait, ce pattern d'échange (Message Exchange Pattern - MEP- ) est positionné par le composant émetteur (ou plutôt, pour être plus précis, par sa configuration et donc le SU), et c'est au composant recevant le message de préciser les patterns qu'il supporte et comment il les supporte (par exemple, un BC JMS en mode fournisseur ne fournit pas de réponse s'il ne fait qu'émettre. Aussi, les MEPs qu'il supporte ne peuvent être que In-Only ou Robust In-Only).

# Cycle de vie des composants, SU et SA
Jusque là, les chapitres précédents ont traité les notions et concepts apportés par les spécifications JBI. Parmi ces notions, nous avons vu qu'il existait réellement trois "objets" manipulables : les composants JBI, les SUs et les SAs.

En fait, ces trois "objets" disposent de leur propre cycle de vie :

* Cycle de vie du composant JBI

![center](http://1.bp.blogspot.com/_XLL8sJPQ97g/SxzHqws3v5I/AAAAAAAAAE8/msIKJNQ0FDs/s200/Image6.png)

* Cycle de vie du SU

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/SxzHy5e8r5I/AAAAAAAAAFE/9TDIzCLqN-U/s200/Image5.png)

* Cycle de vie du SA

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/SxzHy5e8r5I/AAAAAAAAAFE/9TDIzCLqN-U/s200/Image5.png)

Un composant JBI doit être démarré pour être fonctionnel.

Il en est de même pour SU. Cependant, étant contenu dans un SA, il suit, en fait, le même cycle de vie que ce dernier.

La procédure consistant à installer un composant JBI dans l'environnement s'appelle installation, alors que la procédure consistant à déployer un SU dans un composant (au travers d'un SA) s'appelle déploiement.

## Features
Dans les précédents paragraphes, nous avons survolé les notions utilisées par JBI. Cependant, les spécifications JBI apportent également des précisions sur les points suivants :
* JMX (_Java Management eXtension_) : un JBI Provider doit fournir une interface (au sens Java du terme) JMX d'administration
* Ant : un JBI Provider doit offrir des targets Ant pour gérer le cycle de vie des composants (installation/déploiement, démarrage, arrêt, désinstallation/undeployment)
* Shared Library : ce composant non abordé précédemment est une sorte de bundle contenant des librairies accessible de tout l'environnement JBI. Il dispose de son propre cycle de vie (installation, désinstallation) et permet, pour simplifier, un mécanisme de chargement/déchargement dynamique de librairie au sein de l'environnement.

# Avis sur JBI
Ce paragraphe fait un retour de mon humble avis sur JBI. Je n'aborderai pas ici ses implémentations comme PEtALS, ServiceMix ou OpenESB puisque de très bon articles existent déjà et qu'ils couvrent bien mieux que je ne saurai le faire leurs prises en main et la description de leurs spécificités.

## Points faibles
### EIP Provider
Dans les spécifications, une phrase doit vraiment marquer le lecteur :
>"When a message exchange is routed from the service provider, it MUST always be sent to the component that originated the exchange. Subsequent sending of the exchange from the consumer to provider MUST always send the exchange to the provider endpoint selected during the initial send of the exchange" - JBI_Spec

Cette simple citation a de grosses conséquences quant à l'utilisation de JBI : en effet, elle impose qu'il est nécessaire de déléguer à un composant tierce la gestion du chaînage d'appels, du routage, du filtrage, ...

En effet, l'une des forces de JBI est d'offrir un environnement où les composants sont faiblement couplés et dialoguent de manière asynchrone grâce au NMR (entre autre) et il est important de ne pas oublier que JBI permet d'intégrer des composants métiers. Ces services métiers sont cependant hétérogènes et peuvent ne pas parler exactement le même dialecte. Aussi, ils ont besoin de médiation (routage, filtrage, transformation, ...) et donc de mécanismes tels que ceux décrits par les EIP (Enterprise Integration Pattern) qui décrivent des patterns pour la gestion des échanges asynchrones. Cependant, ce mécanisme n'est pas directement utilisable (puisque la réponse doit toujours être réémise à l'émetteur du message) et un composant spécifique doit être utilisé.

Il peut s'agir, dans le cas de ServiceMix ou OpenESB, du composant SE Camel ou, dans le cas de PEtALS, du SE EIP. Cela apporte un surcouche supplémentaire et un point de contension au système. En outre, cela peut sembler peu naturel.

En outre, l'implémentation du SE EIP diffère en fonction du JBI Provider et on peut ne pas l'aimer (pour ma part, je n'aime pas du tout Apache Camel... mais cela fera l'objet d'un article ultérieur...).

Du coup, j'aurai envie de dire que sur ce point, JBI a oublié son but premier en ne couvrant pas ce point essentiel.

En outre, les solutions offertes par les JBI Provider peuvent manquer de maturité (question de point de vue) et surtout, pour l'instant, d'une norme pour les utiliser de manière plus _user-friendly_.

Bien sûr, un moteur SCA, BPEL ou BPMN pourrait répondre à ce besoin mais ils ne couvrent pas le même besoin (qui sont plus applicatifs que techniques) et cela sort du scope de JBI et donc de cet article.

### Qui fait quoi?
Les composants JBI sont tous spécifiques et doivent être compris par l'utilisateur. Cela peut, du coup, être un peu repoussant et demander une recherche de la documentation.

### Portage des composants JBI d'un JBI Provider à l'autre
Sur le papier les composants JBI doivent pouvoir être installés sur n'importe quel JBI Provider. En fait, cela n'est pas tout à fait exacte et pour en porter un, il faut parfois s'arracher les cheveux. En outre, il ne faut pas oublier qu'un composant JBI doit être configuré (via un SU) pour être utilisable, et généralement, le déploiement d'un SU sur un composant qu'on a réussi à installer (c'est déjà pas mal...) sur un JBI Provider tierce s'avère être également peu aisé...

## Points forts
Le point fort de JBI est, de mon point de vue, qu'il offre un conteneur de conteneurs.

En outre, il existe un grand nombre de ces conteneurs (au sens composants JBI du terme) offerts par les différents JBI Provider et généralement, ils peuvent suffire à répondre au besoin.

Si tel est le cas, une utilisation d'un JBI Provider s'avère simple (il suffit de copier/coller le composant JBI, de le configurer via l'écriture d'un SU et de le déployer en copiant/collant le SA qui contient le SU).

En fait, dans ce cas, le plus gros du travail est la configuration des composants JBI et donc l'écriture d'un fichier XML.

# Conclusion
Suite à cet article, la question qui peut se poser est : Ai-je vraiment besoin de JBI ? En fait, j'aurai envie de dire que cela dépend du besoin. Si le besoin est la mise en place d'une grosse infrastructure qui doit être déployée sur plusieurs sites/noeuds, une implémentation JBI peut être un bon choix. Par contre, si les besoins sont plus des problématiques de médiation au sein d'un controlleur, alors un simple EIP Provider (tel que Apache Camel, Spring Integration ou Mule iBeans) suffit.

En conclusion, je pense que pour pouvoir utiliser pleinement un ESB basé sur JBI, il est important d'avoir un minimum de connaissance de JBI et cela, même si les JBI Provider préfèrent communiquer plus sur ce que peut apporter leur produit que sur le comment (mais cela est normal au final... ).

# Pour aller plus loin...
* __SOA : le guide de l’architecte__ de X. Fourner-Morel, P. Grojean, G. Plouin et C. Rognon chez Dunod
* __Enterprise Integration Patterns__ de G. Hohpe et B. Woolf chez Addisson Wesley 
* Spécification de JBI : http://jcp.org/aboutJava/communityprocess/edr/jsr208/index.html 
* Site de PEtALS : http://petals.objectweb.org/
* Site de ServiceMix : http://servicemix.apache.org
* Article du site InfoQ sur la problématique de médiation/orchestration avec JBI : http://www.infoq.com/articles/louis-dutoo-esb-routing 
* Article du site JDJ sur JBI et ServiceMix : http://java.sys-con.com/node/117740
* Article du site JavaWorld sur ServiceMix : http://www.javaworld.com/javaworld/jw-12-2005/jw-1212-esb.html 
* Article du blog de Chip Overclock sur JBI : http://coverclock.blogspot.com/2007/01/fun-facts-to-known-and-tell-java.html
* Article du site de Architect zone sur ServiceMix : http://architects.dzone.com/articles/pattern-based-development-with-0
* Article du site Java World sur une étude de cas avec ServiceMix : http://www.javaworld.com/javaworld/jw-09-2006/jw-0904-jbi.html
 http://opensourceesb.blogspot.com/2007/10/using-petals-jbi-components-in.html
* Site de OSOA (Open Service Oriented Architecture) : http://www.osoa.org/display/Main/Service+Component+Architecture+Home
* Blog de The Server Labs sur JBI et SCA : http://www.theserverlabs.com/blog/2009/07/07/developing-apps-with-sca-and-jbi/  
http://blogs.sun.com/tombarrett/entry/sca_raining_on_the_jbi
* Blog de Bruce Snyder comparant JBI et SCA : http://www.jroller.com/bsnyder/entry/jbi_and_sca_are_complimentary
* Blog de Frank Wolgang comparant JBI et SCA : http://apps.itemis.de/roller/wfrank/entry/jbi_2_0_jbi_vs
* Article du site Le Monde Informatique sur SCA : http://blog1.lemondeinformatique.fr/ingenierie_logicielle/2006/01/sca_raction_de_.html
* Blog de Xebia sur les ESB légers : http://blog.xebia.fr/2007/12/17/spring-integration-lavenement-des-lightweight-esb/ 


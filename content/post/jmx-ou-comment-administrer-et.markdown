---
title: "JMX ou comment administrer et superviser son système..."
date: 2010-05-05 11:07:07 +0100
comments: true
tags: 
- java
- jmx
- réflexion
---
![left-small](http://1.bp.blogspot.com/_XLL8sJPQ97g/S-HXgq4HI7I/AAAAAAAAAJ4/LzJKs3wTNZM/s200/jmx4.png)
Je n'ai encore fait aucun article sur JMX (_Java Management eXtension_). Pourtant, je pense qu'il s'agit d'une technologie indispensable pour toute application qui se veut un minimum sérieuse et industrialisable. Dans ce post, je ne parlerai pas de "comment ça marche" ou je ne fournirai pas de tutoriaux. Il s'agit juste d'un petit coup de gueule parce que j'en ai marre d'entendre toujours les mêmes inepties sur JMX...

Bon, pourquoi JMX me direz-vous? Encore une technologie? Encore un nouveau truc dans mon architecture à mettre en œuvre sur lequel il va falloir faire monter des gens en compétence et qu'il va falloir maintenir?
<!-- more -->

Ben, en fait, non... quoique, en fait, si... il est vrai qu'il s'agit d'une "nouvelle technologie" (au sens encore une nouvelle technologie dans le projet) à mettre en place sur le projet, qu'il va falloir faire monter des gens dessus et qu'il va falloir maintenir.

Cependant, elle correspond parfaitement aux besoins de supervision et d'administration de l'application (en plus d'être un standard et une spécification)... je m'explique : si votre application est en production et qu'une anomalie a lieu (ben oui, une application a toujours des anomalies ;-) ), le développeur, qui va devoir traiter le problème, a besoin de connaitre le contexte du problème, l'état de la mémoire et du CPU Java, l'état de l'application, etc, etc, etc...

Bon, vous allez me répondre qu'il y a les logs! Ah, les logs... parlons-en! Des méga et des méga de logs qui, avec un peu de chance, sont un minimum classés dans des fichiers. Par contre, quid de la recherche de l'information dans toute ce bruit, ou quid des architectures distribués où la synchronisation horloge est dans les choux?

En fait, JMX permet la supervision de votre application en permettant, par exemple, de positionner des indicateurs (nombre de hits d'un web service, nombre de requêtes, ...) dans l'application et de les interroger. Il permet également de voir leur évolution et même d'émettre des notifications afin de remonter en temps réel ces valeurs.

En outre, JMX permet d'administrer le système. Comment? En mettant à disposition un ensemble d'opérations (défini - via les différents types de MBeans - par l'application ou le framework) avec lesquels il est possible d'interagir. Cela permet, par exemple de changer le niveau de logs de l'application à chaud ou de récupérer l'état du cache applicatif.

A la question : encore une nouvelle technologie à mettre en place? En fait, si vous utilisez déjà un serveur d'application, un conteneur de servlet, un JMS Provider, un framework de logs ou une pile SOAP tel qu'Axis ou CXF, il est fort à parier que JMX est déjà présent dans votre application puisqu'il est déjà offert par ces derniers afin de permettre leur administration et supervision...

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/S-HWn0wnVAI/AAAAAAAAAJo/rllVP2RIP_s/s200/jmx.png)
![center](http://4.bp.blogspot.com/_XLL8sJPQ97g/S-HW5MyQV5I/AAAAAAAAAJw/xosMzejwZHs/s200/jmx2.png)

Par contre, j'avoue que niveau mise en œuvre, JMX souffre de quelques lacunes... En effet, du point de vue de l'équipe de production ou de l'exploitation, qui utilise souvent SNMP pour administrer et superviser l'infrastructure matérielle, il sera sûrement nécessaire d'utiliser des solutions Java pour agréger les différentes métriques et centraliser l'administration (surtout si les applications tournent sur des JVM différentes). Des ponts existent vers SNMP (en l'occurrence en générant des MIB) mais cela s'avère généralement ardue (et j'avoue avoir lamentablement échoué... :( ). Aussi, je préconiserai plutôt d'utiliser des outils autres que ceux dédier aux infrastructures matérielles pour superviser et administrer les couches applicatives (de plus, il s'agira généralement d'équipes différentes puisque la problématique est quand même différente! ... ouf, je m'en sors bien... ;-) ). Des outils comme [Hyperic](http://www.hyperic.com/) peuvent être de très bon candidat pour faire cela. Sinon, il reste toujours la bonne vieille jConsole nativement présente dans le JDK qui permet toujours de s'en sortir ou alors, il est toujours possible de développer son propre outils (application web ou application lourde) pour faire le boulot...

![center](http://2.bp.blogspot.com/_XLL8sJPQ97g/S-HWRCOOSBI/AAAAAAAAAJg/DUEhO7O-hkI/s200/logo-hyperic.gif)

Enfin, le dernier point que je voulais aborder est que même si JMX n'est pas très compliqué (du moins pour des utilisations simples), il est, à mon sens, indispensable d'y penser en début du projet (d'un autre coté, une application un minimum industrielle est censée couvrir ce type de problématique dès le début... non?). Sa mise en œuvre peut être faite, par exemple, via le développement d'une petite API simple qui peut aller introspecter les interfaces de l'application à administrer et qui peut aller enregistrer des Dynamics MBeans dans l'agent JMX adéquate.

En espérant vous avoir convaincu sur l'intérêt de JMX qui permet de rendre beaucoup de services pour pas trop d'effort!

Voilà, fin de ce petit post... il est à noter que d'origine, je voulais juste parler de comment s'en sortir avec JMX et les Firewall... ;-)
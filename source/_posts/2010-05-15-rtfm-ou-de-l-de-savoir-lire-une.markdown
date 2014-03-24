---
layout: post
title: "RTFM ou de l'importance de savoir lire une documentation"
date: 2010-05-15 11:24:47 +0100
comments: true
categories: 
- java
- réflexion
---
<img height="200" src="http://farm1.static.flickr.com/100/291194341_5d141633c8.jpg" width="200" alt="left"/>
Ce petit article est plus une tranche de vie ayant pour objectif de montrer l'importance de savoir lire une documentation qu'un "vrai" post intéressant...

Il est vrai que savoir trouver son information s'applique énormément dans le monde Open Source où il est crucial de savoir fouiller (documentation, forum, ...) lorsqu'un fait technique (anomalie? framework incomplet? cas d'usage délirant? ...) a lieu mais cela s'applique aussi dans notre vie de programmeur de tous les jours.

 
Bien sur, le monde Open Source n'est pas le seul où doit s'appliquer ce principe.

Je vais donc raconter mon anecdote (stupide, j'en conviens donc soyez indulgent avec moi)...
<!-- more -->
#Contexte
Alors que j'étais tranquillement en train de farfouiller dans le code source du projet sur lequel je travaille, je tombe sur le code suivant (j'ai élagué le contrôle de la taille de mon tableau pour éviter d'alourdir les exemples...) :
```java
public static String[] troncateString(String[] input) {
        int inputSize = input.length;
        String[] output = new String[inputSize - 1];
        for (int i = 1; i < inputSize; i++) {
            output[i-1] = input[i];
        }
        return output;
}
```
Là, je me dis : mais qu'est ce que c'est que ça!!!... je commence alors à m'agiter sur ma chaise et je me dis, tiens, n'y aurait-t-il pas un petit gain en passant par une Collection. Essayons! J'écris alors un petit programme de test :
```java
public static String[] troncateString2(String[] input) {
        List<String> output = Arrays.asList(input);
        output.remove(0);
        return output.toArray(new String[0]);
}
```
Je le lance confiant, et là, c'est la douche froide : un horrible message s'affiche :

```text
Exception in thread "main" java.lang.UnsupportedOperationException
    at java.util.AbstractList.remove(AbstractList.java:144)
    at Test.troncateString2(Test.java:21)
```
Je me dis alors : Mince, qu'est ce que j'ai encore fait comme bêtise... (je vous le fais en version censurée là... ;-)) Je prends donc mon ami la javadoc et je vérifie rapidement la méthode `asList`, je ne vois toujours rien... J'ajoute alors un `getClass()` qui me sort un class `java.util.Arrays$ArrayList`... bizarre... En désespoir de cause, je me dis : allez, faisons un petit F3 (je suis sous eclipse) pour aller voir les entrailles de la bête. J'obtiens donc :
```java
public static <T> List<T> asList(T... a) {
        return new ArrayList<T>(a);
}
```
puis ça :
```java
private static class ArrayList<E> extends AbstractList<E>
    implements RandomAccess, java.io.Serializable
    {
    private static final long serialVersionUID = -2764017481108945198L;
    private final E[] a;
 
    ArrayList(E[] array) {
        if (array==null)
             throw new NullPointerException();
        a = array;
    }
    ...
}
```
Je cherche alors ma méthode `remove()` mais ne la trouve pas... tous s'explique : mon horrible `UnsupportedOperationException` vient de `AbstractList`... Mais pourquoi Sun a-t-il fait un truc aussi bizarre que de redéfinir son `ArrayList` en inner class de `Arrays` et en plus en n'implémentant pas certaines méthodes susceptibles de créer des erreurs au Runtime... ? Je râle, je vais en parler à un collègue et là, il me dit : ben oui, dans la javadoc, il est indiqué la chose suivante :

>Returns a fixed-size list backed by the specified array.

En plus, première ligne... je n'avais pas lu le fixed-size... :( Voilà, conclusion, 15 minutes de perdu... tout ça parce que, trop confiant, je n'ai pas lu consciencieusement la documentation. Bien sûr, cela s'applique à tous types de cas et cette anecdote n'en est qu'un exemple... (il n'empêche que je trouve quand même cela vraiment trompeur cette méthode asList... :( ).

[update : un collègue m'a fait remarqué que je n'avais pas été très précis quant au pourquoi du comment j'avais eu cette erreur... : le fait que la classe interne `ArrayList` de `Arrays` n'implémente pas la méthode `remove()` implique que c'est l'implémentation de la méthode `remove()` de `AbstractList` qui est utilisée. Cette méthode lançant un `UnsupportedOperationException`, c'est donc ce qui remonte. Ce choix ayant été fait en raison de la volonté de renvoyer un ArrayList de taille fixe et donc de ne pas implémenter cette méthode.]

#Conclusion

Lisez la documentation lorsqu'un problème se présente. Si vous ne le faites pas, et si vous "osez" poser la question sur un forum sans recherche préalable, ne vous vexez pas si comme toute réponse vous obtenez un jolie RTFM... En outre, le risque pour vous est de vous faire "griller" sur le forum où plus personne ne daignera vous répondre. De même, quoi de plus énervant quand un collègue vous pose une question alors qu'un simple coup de google aurait pu le renseigner et, en plus, si cela vous a fait perdre votre idée alors que vous veniez de trouver la solution du siècle! ;-)
#Epilogue

Ci-joint le code que j'aurai dû utiliser :

```java
public static String[] troncateString(String[] input) {
        List<String> output = new ArrayList<String>(Arrays.asList(input));
        output.remove(0);
        return output.toArray(new String[0]);
}
```
A noter que j'aurais également pu faire un simple :
```java
public static String[] troncateString(String[] input) {
        return Arrays.copyOfRange(input, 1, input.length);
}
```
Ce qui me donne au final les résultats suivants pour un tableau de String de 100000 éléments (Jdk 1.6) :
```text
time : 11 == version avec boucle for et décalage
time : 9 == version avec utilisation d'une liste intermédiaire
time : 2 == version avec le copyOfRange
```
---
title: "Ubuntu 11.04"
date: 2011-06-11 21:59:50 +0100
comments: true
tags: 
- divers
---

![left-small](http://2.bp.blogspot.com/-PYGo1WbVX3E/TfOOwCue0II/AAAAAAAAAYE/bP0fMBPp99I/s1600/ubuntuLogo.png)

Ubuntu 11.04 est sorti depuis le 28 avril 2011. Je ne reviendrai pas ni sur son ergonomie ni sur ses ajouts puisque cela a déjà été débattu sur de nombreux sites/blogs. Cependant, depuis la mise à jour, un point avait tendance à m'agacer.

En effet, dans un shell bash, sur la complétion de la commande ls (entre autre) avec des répertoires contenant des espaces, il ne m'échappait plus ces derniers. Du coup, obligation d'aller les échapper manuellement mais chose encore plus embêtante était qu'il ne me proposait plus le contenu de mon répertoire.

Pire, en échappant les espaces et en utilisant la complétion, il me supprimait mon échappement...

Cet article fournira donc une rapide solution pour corriger ce problème.

<!-- more -->

# Symptôme

Pour l'arborescence ci-dessous, j'avais le comportement suivant :
![center](http://1.bp.blogspot.com/-s9j2IBv_w7M/TfODhMgXhYI/AAAAAAAAAXk/imMqhaAW9LY/s1600/ubuntu02.png)

Utilisation basique de la complétion :
![center](http://4.bp.blogspot.com/-YSypzYsV_Zk/TfOBzlrIsVI/AAAAAAAAAXg/-T6-n0Hu5pM/s1600/ubuntu01.png)

Avec échappement de l'espace...

Avant :
![center](http://3.bp.blogspot.com/-YPHW04jHH0s/TfOEEp0gqDI/AAAAAAAAAXo/t8FKxJedORo/s1600/ubuntu03.png)

Avec échappement de l'espace...

Avant :
![center](http://3.bp.blogspot.com/-YPHW04jHH0s/TfOEEp0gqDI/AAAAAAAAAXo/t8FKxJedORo/s1600/ubuntu03.png)

Après exécution de la commande Tab :
![center](http://2.bp.blogspot.com/-MSYG8ZCqVAE/TfOEVNKnNfI/AAAAAAAAAXs/k6heNnlMZhg/s1600/ubuntu04.png)

Entraînant, bien sûr, une erreur lors de son exécution :
![center](http://1.bp.blogspot.com/-qdZH3XNVpns/TfOFJ5O3xYI/AAAAAAAAAXw/2ueSn_2rZ9s/s1600/ubuntu05.png)

# Correctif

Du coup, après une ou deux recherche sur mon ami google, je suis tombé sur le [bug suivant](https://bugs.launchpad.net/ubuntu/+source/bash-completion/+bug/769866).

En fait, pour résumer, il semble que cela vienne du script de complétion utilisé par bash.

Aussi pour corriger ce problème, il ne vous reste plus qu'à éditer le fichier `/etc/bash_completion` avec les bons droits et à modifier sa ligne 1587 pour y remplacer `default` par `filenames`.

```bash
# makeinfo and texi2dvi are defined elsewhere.
for i in a2ps awk bash bc bison cat colordiff cp csplit \
    curl cut date df diff dir du enscript env expand fmt fold gperf gprof \
    grep grub head indent irb ld ldd less ln ls m4 md5sum mkdir mkfifo mknod \
    mv netstat nl nm objcopy objdump od paste patch pr ptx readelf rm rmdir \
    sed seq sha{,1,224,256,384,512}sum shar sort split strip tac tail tee \
    texindex touch tr uname unexpand uniq units vdir wc wget who; do
    have $i && complete -F _longopt -o filenames $i
done
unset i
```
Après un petit coup de rechargement du `bashrc` (qui charge le `bash_completion`), vous aurez alors résolu votre problème ;-)

Rechargement du ~/.bashrc :
![center](http://1.bp.blogspot.com/-_e-p2C3ZNPs/TfOJc6I-LoI/AAAAAAAAAX8/Um_QhF9V_18/s1600/ubuntu09.png)

Utilisation basique de la complétion et exécution :
![center](http://3.bp.blogspot.com/-2wMokee8xhI/TfOJhXsTgiI/AAAAAAAAAYA/FK6AiN7vDfo/s1600/ubuntu08.png)

Utilisation de la complétion après complétion du répertoire :
![center](http://3.bp.blogspot.com/-fSXGkhwkeuE/TfOH-TUEJeI/AAAAAAAAAX4/6K1qhgEIt6U/s1600/ubuntu07.png)

Voilà pour cet article qui, pour une fois, ne parle pas Java mais qui, je l'espère, pourra être utile... ;-)

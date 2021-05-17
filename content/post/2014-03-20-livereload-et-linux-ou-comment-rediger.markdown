---
title: "Livereload et Linux ou comment rédiger du html simplement"
date: 2014-03-20 11:24:11 +0100
comments: true
tags: 
- divers
- html
---
![left-small](http://3.bp.blogspot.com/-iICRkMH0vhY/UxnsLscNtII/AAAAAAAABRA/uXMjhVZjOBg/s1600/ubuntu_livereload.png)

Cet article n'a rien à voir avec le monde Java.

Il s'agit plus d'un petit tutoriel surtout destiné... à moi-même... ;-)

En effet, mon blog étant hébergé sur blogger, lorsque je rédige un article, je le fais le plus souvent directement en HTML. Afin d'avoir une idée de l'aperçu, je le charge directement dans mon navigateur mais passer son temps à faire du Alt-Tab/F5 a rapidement commencé à m'énerver.

Aussi, ayant entendu parlé depuis un moment de livereload, je me suis dis que cela pouvait être pas mal de l'utiliser.

Cependant, j'avoue avoir subi un échec cuisant (je suis sous un environnement linux et je tiens à préciser que je ne suis pas des plus à l'aise avec le monde nodejs ou ruby - ceci explique aussi cela... ;-) ) et cela malgré avoir mes multiples tentatives (à coup de gem install) et recherche sur mon ami Google (enfin, quand on n'est pas doué, on n'est pas doué... :'( )

Du coup, j'ai pris l'option :
>"J'appelle un ami"

Dans mon cas, je remercie [Harold](https://twitter.com/hcapitaine) pour son aide.

Il m'a conseillé de me jeter un oeil sur [Grunt](http://gruntjs.com/) qui gérait nativement livereload (bon, pour ma défense, je n'avais pas prévu de partir sur Grunt... ;-) ).

Cet article présentera donc comment j'ai réussi à faire fonctionner le bouzin avec, en prime, la possibilité d'utiliser Markdown.

Concernant l'environnement, il s'agit d'Ubuntu 13.10 (sur Mac, je ne doute pas que cela aurait été plus simple mais bon...)

[NDLR : encore une fois, je précise que je ne suis pas du tout expert dans ce domaine, il est donc inutile de me poser des questions. Par contre, toutes propositions d'amélioration est la bienvenue ;-)]

A titre informatifs, les fichiers se trouvent sur [Github](https://github.com/jetoile/livereload-grunt).

<!-- more -->

#Installation de nodejs, de npm et de grunt

Dans un premier temps, il convient d'installer tout le strict minimum afin de permettre à Grunt de fonctionner.

Installation de NodeJS :
```bash
sudo apt-get install nodejs
```

Installation de Npm :
```bash
sudo apt-get install npm
```

Installation de Grunt (de manière global au système) :
```bash
sudo npm install -g grunt-cli
```

Installer le [plugin](http://feedback.livereload.com/knowledgebase/articles/86242-how-do-i-install-and-use-the-browser-extensions-) livereload sur Chrome ou Firefox.

Dans le cas de Chrome, pensez à modifier sa configuration pour qu'il autorise l'accès aux URLs de fichiers.
![medium](http://1.bp.blogspot.com/-jg_Pm58wTpc/Uxno8Z2A74I/AAAAAAAABQo/zUsVZpiPSz4/s1600/chrome-config.png)

#Configuration de Grunt

Par la suite, toutes les opérations seront effectué dans le répertoire `$HOME/tmp`.

Une fois tous les logiciels installés, il ne reste qu'à créer les fichiers nécessaires au bon fonctionnement de Grunt :

* `package.json` qui permet de télécharger les modules et de les installer localement dans répertoire courant (un peu comme la rubrique dependencies de Maven),
* `gruntfile.js` qui permet de configurer les différents modules précédemment installés par la commande npm install (un peu comme la rubrique build/plugins de Maven).

`$HOME/tmp/package.json`

```javascript
{
  "name": "yo",
  "version": "0.0.0",
  "dependencies": {},
  "devDependencies": {
    "grunt": "~0.4.1",
    "matchdep": "~0.1.2",
    "grunt-express": "~1.0.0-beta2",
    "grunt-contrib-watch": "~0.5.1",
    "grunt-open": "~0.2.1"
  },
  "engines": {
    "node": ">=0.8.0"
  }
}
```

Il convient maintenant de lancer le téléchargement et l'installation des modules en exécutant la commande suivante :
```bash
npm install
```

Ne reste plus qu'à configurer les modules via le fichier Gruntfile.js.

`$HOME/tmp/Gruntfile.js`

```javascript
module.exports = function(grunt) {

  // Load Grunt tasks declared in the package.json file
  require('matchdep').filterDev('grunt-*').forEach(grunt.loadNpmTasks);

  grunt.initConfig({

    // grunt-express will serve the files from the folders listed in `bases` on specified `port` and `hostname`
    express: {
      all: {
        options: {
          port: 9000,
          hostname: "0.0.0.0",
          bases: ['html'], // Replace with the directory you want the files served from
                              // Make sure you don't use `.` or `..` in the path as Express
                              // is likely to return 403 Forbidden responses if you do
                              // http://stackoverflow.com/questions/14594121/express-res-sendfile-throwing-forbidden-error
          livereload: true
        }
      }
    },

    // grunt-watch will monitor the projects files
    watch: {
      all: {
        files: '*.html',
        options: {
          livereload: true
        }
      }
    },

    // grunt-open will open your browser at the project's URL
    open: {
      all: {
        path: 'http://localhost:<%= express.all.options.port%>'
      }
    }
  });

  // Creates the `server` task
  grunt.registerTask('default', [
    'express',
    'open',
    'watch'
  ]);
```

Cette configuration est fonctionnelle avec l'arborescence suivante :

![center](http://4.bp.blogspot.com/-yzXs78VTiFw/Uxnor5jGo_I/AAAAAAAABQY/01_s7DCQHT4/s1600/arbo01.png)

Dans cette dernière et avec cette configuration, tous les fichiers du répertoire `html` avec l'extension .html seront suivis par livereload. Aussi, il suffit de créer un fichier html dans le répertoire `html` (par exemple html/sample.html) et de lancer la commande :

```bash
grunt
```
Le navigateur par défaut s'ouvrira alors avec l'url : http://localhost:9000/. Il suffit d'y ajouter la page voulu (dans notre exemple http://localhost:9000/sample.html) et d'activer le plugin `livereload`.

![center](http://1.bp.blogspot.com/-edBGc5pdGLg/UxnpKGFtjsI/AAAAAAAABQw/E6maHZkgGcs/s1600/icon_livereload.png)

Il ne reste plus qu'à modifier le fichier html et lors de sa sauvegarde, la page sera automatiquement mis à jour dans le navigateur!

Si l'erreur suivante apparait :

```bash
/usr/bin/env: node: Aucun fichier ou dossier de ce type
```

Il faut installer `nodejs-legacy` (cf. [ici](https://github.com/volojs/volo/issues/154))

```bash
sudo apt-get install nodejs-legacy
```


#Configuration de Grunt pour l'intégration de Markdown

L'intégration de Markdown se fait de la même manière que précédemment.

Les fichiers `package.json` et `Gruntfile.js` se voient juste ajouter le module `grunt-markdown`.

`package.json`
```javascript
{
  "name": "yo",
  "version": "0.0.0",
  "dependencies": {},
  "devDependencies": {
    "grunt": "~0.4.1",
    "matchdep": "~0.1.2",
    "grunt-express": "~1.0.0-beta2",
    "grunt-contrib-watch": "~0.5.1",
    "grunt-open": "~0.2.1",
    "grunt-markdown": "~0.5.0"
  },
  "engines": {
    "node": ">=0.8.0"
  }
}
```

`Gruntfile.js`

```javascript
odule.exports = function(grunt) {
  require('matchdep').filterDev('grunt-*').forEach(grunt.loadNpmTasks);

  grunt.initConfig({

     markdown: {
      all: {
        files: [
          {
            expand: true,
            src: '*.md',
            dest: 'html/',
            ext: '.html'
          }
        ]
      }
    },

    express: {
      all: {
        options: {
          port: 9000,
          hostname: "0.0.0.0",
          bases: ['html'], 
          livereload: true
        }
      }
    },

    watch: {
      all: {
        files: ['*.md','*.html'],
        tasks : ['markdown'],
        options: {
          livereload: true
        }
      }
    },

    open: {
      all: {
        path: 'http://localhost:<%= express.all.options.port%>'
      }
    }
  });

  grunt.registerTask('default', [
    'markdown',
    'express',
    'open',
    'watch'
  ]);
};
```

Suite à la modification de ces fichiers (attention à penser de faire un `npm install` après modification du fichier `package.json`), il est alors possible d'écrire son document en Markdown.

L'arborescence à respecter par rapport à la configuration est la suivante :
![center](http://2.bp.blogspot.com/-kgC6a8jRwG8/Uxno29Sr9EI/AAAAAAAABQg/lf3W_rnDAvU/s1600/arbo02.png)

Lors de la modification du fichier .md, son pendant html sera automatiquement créé (ou modifié) et pris en compte par un rechargement du navigateur.

#Conclusion

Comme on a pu le voir, au final, configurer livereload pour permettre un rechargement automatique lors de la modification d'un fichier HTML ou Markdown n'est pas si compliquée une fois que l'on a tous les fichiers et les bons modules configurés.

D'ailleurs, cet article est le premier écrit en Markdown (quand je disais que ce n'était pas ma tasse de thé...).

En espérant qu'il pourra être utile à d'autres lecteurs et en m'excusant par avance si des informations sont approximatives...

#Pour aller plus loin...

* http://rhumaric.com/2013/07/renewing-the-grunt-livereload-magic/
* http://justinklemm.com/grunt-watch-livereload-javascript-less-sass-compilation/
* http://feedback.livereload.com/knowledgebase/articles/86242-how-do-i-install-and-use-the-browser-extensions-
* http://blog.roddet.com/2013/12/nantesjs-gruntjs/
* https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet#wiki-lists
eZ (Publish / Platform) et Angular.js
=====================================

Ce tutoriel va vous aider à mettre en place un site basé sur [Angular.js](https://angularjs.org/) et eZ Publish/Platform.

Pour Angular.js nous utiliserons la version **1.4.x** étant donné que la version 2.0 est encore en développement au moment de l'écriture de ce tutoriel (rc1).
Pour eZ, nous vous recommandons une version >= à 5.4 (2014.11) afin de profiter pleinement des nouveautés et optimisations de l'API REST.

Mise en place
-------------

L'installation d'eZ Publish/Platform est facilement réalisable en suivant la [documentation en ligne](https://doc.ez.no/display/DEVELOPER/Get+Started+with+eZ+Platform).
Par la suite nous allons utiliser **bower** afin de gérer les dépendances de notre application, entre autres. Si vous n'avez pas déjà installer ```node.js```, rendez vous sur [la page de téléchargement](https://nodejs.org/en/).
Une fois l'installation terminée, il est fortement suggéré de mettre à jour ```npm``` via:
```
$ sudo npm install -g npm
```
On peut alors installer bower:
```
$ sudo npm install -g bower
```

Nous allons créer un nouveau dossier contenant notre installation ```Angular.js``` (je choisis volontairement d'être en dehors du dossier d'eZ).

```
$ mkdir angular
$ cd angular
$ bower init
```

Bower va vous demander deux trois petites choses pour initialiser le fichier ```bower.json```. Vous devriez avoir un fichier de ce genre:

```json
{
  name: 'angular',
  authors: [
    'dspe <vincent.royol@gmail.com>'
  ],
  description: 'eZ & Angular example',
  main: 'app.js',
  keywords: [
    'ez',
    'angular',
    'js',
    'publish',
    'platform'
  ],
  license: 'MIT',
  homepage: 'dspe.github.io',
  ignore: [
    '**/.*',
    'node_modules',
    'bower_components',
    'test',
    'tests'
  ]
}
```

Et nous allons créer un fichier ```.bowerrc``` afin de configurer le répertoire dans lequel nous aurons nos dépendances:

```json
{
  "directory" : "public/components"
}
```

Maintenant nous pouvons donc installer tout notre petit monde pour le projet
```
$ bower install --save capi=ezsystems/ez-js-rest-client
$ bower install --save angular#1.4.10 angular-resource angular-route
```

Le code
-------

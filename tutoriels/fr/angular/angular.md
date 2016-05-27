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

```
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

Je ne vais pas vous détailler tous les rouages d'Angular.js et de l'API Rest d'eZ, mais vous donner les premières billes pour continuer votre site.
Dans un premier temps nous pouvons se baser sur l'architecture des dossiers suivants
![Architecture des fichiers](./assets/tree.png)

On va commencer par notre fichier principal qui sera ```app/app.js``` avec le contenu suivant:
```javascript
var module = angular.module("NewsApplication", ["ngRoute", "ngResource"]);

// Route configuration
module.config( [ "$routeProvider", "$locationProvider", function($routeProvider, $locationProvider) {

	$routeProvider
	.when('/', {
		templateUrl: "./app/views/root.html",
		controller: 'MainController'
	})
	.otherwise({
		redirectTo: '/'
	});

    // use the HTML5 history API
    //$locationProvider.html5Mode(true);
}]);

// Global configuration
module.run(['$http', '$rootScope', 'ezpublish', function($http, $rootScope, ezpublish) {

    $rootScope.ezpublish = ezpublish;
    $rootScope.capi = new eZ.CAPI(
        ezpublish.url,
        new eZ.SessionAuthAgent({login: "admin", password: "publish"})
    );

    $rootScope.capi.logIn(function (error, response) {
        if ( error ) {
            console.log('Error!');
            return;
        }
    });

    //$http.defaults.headers.common.Authorization = 'Basic YWRtaW46cHVibGlzaA==';
    //$http.defaults.headers.post['Access-Control-Allow-Credentials'] = 'true';
}]);
```

La première partie est du pur Angular.js permettant la gestion des routes et de charger un ```template``` et ```controller``` associés. Nous reviendrons sur le controller un peu plus tard.
Afin d'utiliser la libraire ```CAPI```, nous devons nous connecter à l'API d'eZ. Dans notre exemple, nous allons faire une connection très peu sécurisé en utilisant un identifiant et mot de passe administrateur. **Ne jamais faire cela en production***.

Dans l'exemple ci dessus, nous utilisons l'authentification par session, demandant un login et password. Il existe deux autres possibilités tel que: ```Session authentification``` (demande une session déjà existante) et ```Basic authentication```. Vous trouverez tous les informations sur la [documentation](https://doc.ez.no/display/DEVELOPER/Using+the+JavaScript+REST+API+Client#UsingtheJavaScriptRESTAPIClient-Instantiationandauthentication). On sauvegarde le tout à travers une variable du rootScope afin de pouvoir l'utiliser ailleurs.

Comme vous avez pu le contaster, nous avons une variable ```ezpublish```. Elle  vient d'un fichier de constantes que nous avons crées: ```app/constants.js```

```
module.constant("ezpublish", {
        "url": 'http://ezpublish.dev',
        "path": '/api/ezp/v2',
        "rootNode": '2',
});
```

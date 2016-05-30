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
Afin d'utiliser la libraire ```CAPI```, nous devons nous connecter à l'API d'eZ. Dans notre exemple, nous allons faire une connection très peu sécurisé en utilisant un identifiant et mot de passe administrateur. **Ne jamais faire cela en production**.

Dans l'exemple ci dessus, nous utilisons l'authentification par session, demandant un login et password. Il existe deux autres possibilités tel que: ```Session authentification``` (demande une session déjà existante) et ```Basic authentication```. Vous trouverez tous les informations sur la [documentation](https://doc.ez.no/display/DEVELOPER/Using+the+JavaScript+REST+API+Client#UsingtheJavaScriptRESTAPIClient-Instantiationandauthentication). On sauvegarde le tout à travers une variable du rootScope afin de pouvoir l'utiliser ailleurs.

Comme vous avez pu le contaster, nous avons une variable ```ezpublish```. Elle  vient d'un fichier de constantes que nous avons crées: ```app/constants.js```

```
module.constant("ezpublish", {
        "url": 'http://ezpublish.dev',
        "path": '/api/ezp/v2',
        "rootNode": '2',
});
```

Je vais vous montrer les deux façons de faire pour récupérer votre contenu: par la libraire CAPI ou en utilisant Angular Resource.
Il est cependant à noter qu'Angular, dans sa conception, adore communiquer avec des API en End Point: nous récupérons toutes les informations nécessaires pour l'affichage. Hors l'API d'eZ a été conçu en RESTful: vous devez faire plusieurs requêtes HTTP pour récupérer toutes les informations nécessaires.

# 1ère méthode:

Nous allons d'abord créer une factory afin de pouvoir réutiliser le code au besoin dans notre application. Pour celà nous placerons dans le fichier ```app/services/restservice.js``` le code suivant:

```javascript
module.factory( "contentRestService", ["ezpublish", "$resource", function(ezpublish, $resource) {
    var ezPath = ezpublish.url + ezpublish.path;

    return $resource(ezPath + '/content/objects/:contentId', {contentId:'@contentId'}, {
        get: {
            method: 'GET',
            withCredentials: false,
            cache: false,
            //isArray: false,
            headers: {
                'Accept': 'application/vnd.ez.api.Content+json'
            }
        }
    });
}]);
```
Ici, nous attendons un contentId et utilisons la ressource ```/content/objects/<id>```. Vous trouverez toutes les informations nécessaires [ici](https://github.com/ezsystems/ezpublish-kernel/blob/master/doc/specifications/rest/REST-API-V2.rst#load-content)

Attention: n'oubliez pas de décommenter les deux lignes ```$http.defaults.header``` dans le fichier ```app/app.js``` afin de pouvoir dialoguer avec l'API REST.

# 2ème méthode:

Cette fois ci nous allons utiliser la librairie ```CAPI``` afin de récupérer les informations. Dans l'exemple suivant, nous allons créer une ```view``` afin de rechercher tous les articles dans la section *news*.

```javascript
module.factory( "viewsRestService", ["ezpublish", "$rootScope", function(ezpublish, $rootScope) {

    var ezPath = ezpublish.url + ezpublish.path;

    return {
        getView : function() {
            jsPromiseCAPI = new eZ.PromiseCAPI($rootScope.capi);
            promiseCapiContentService = jsPromiseCAPI.getContentService();
            query = $rootScope.capi.getContentService().newViewCreateStruct('articleSearch', 'ContentQuery');
            query.body.ViewInput.ContentQuery.Criteria = {
                ContentTypeIdentifierCriterion: 'article',
                SectionIdentifierCriterion: 'news'
            };
            promise = promiseCapiContentService.createView(query);
            return promise;
        }
    };
}]);
```
Ici, nous avons un cas un peu particulier, c'est pourquoi nous utilisons une ```Promise```. Pour une utilisation classique, vous pouvez vous référencer à la documentation.
Afin de générer cette ```view```, nous devons lui donner un nom suivi du type de donnée que nous souhaitons récupérer: des contenus ou des emplacements, via la méthode ```newViewCreateStruct```.
L'étape suivante est le passage de critères. Pour notre cas, cela sera le type de contenu souhaité, ainsi qu'une section.
Pour voir les informations de retour de l'API REST vous trouverez le tout détailler sur la [documentation](https://github.com/ezsystems/ezpublish-kernel/blob/master/doc/specifications/rest/REST-API-V2.rst#views).

# Quelque petites notes

Attention cependant, tout n'est pas forcément trivial:
* Dans les versions 5.x d'eZ, l'API REST vous renvoie les champs ```ezxmltext``` en format XML.
* Les champs de type ```ezlandingpage``` et ```ezpage``` vous renvoie, pour le premier cas, une information en JSON contenant les différents éléments à afficher.
* Le nombre de requêtes HTTP avec l'API REST et CAPI sont assez importantes mais il ne faut pas (trop) s'inquiéter: en effet des mécanismes de caches sont prévus à cet effet.

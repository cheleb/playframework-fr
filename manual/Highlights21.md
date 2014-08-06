<!--- Copyright (C) 2009-2013 Typesafe Inc. <http://www.typesafe.com> -->
# Quoi de neuf dans Play 2.1?

## Migration vers Scala 2.10 

La totalité de l'API de Play a été migrée sur Scala 2.10 afin que vos applications bénéficient des nouvelles fonctionnalités du language.

Dans un même temps la dépendence qui existait entre la version de Scala utilisée par le _système de build_ (sbt), et la version de Scala utilisé par le runtime a été coupée. Ainsi, il est plus aisé de construire et de tester Play sur des version expérimentale ou instable de Scala.

## Migration vers `scala.concurrent.Future`

Une des grandes fonctionnalité apportée par Scala 2.10 est la nouvelle librairie standard `scala.concurrent.Future` pour gérer le code asynchrone en Scala. Play s'appuie maintenant sur cette API, ces fonctions HTTP asynchrones HTTP et de streaming sont désormais compatible avec les autres librairies qui uitilent cette API. 

Cela rend l'utilisation de Play avec Akka encore plus simple, comme avec n'importe quel futur pilote de dépot de donnée asynchrone à venir qui utiliseront cette API.

Dans un même temps, nous avons simplifié le modèle du contexte d'excécution, et en fournissant un moyen simple de choisir pour chaque partie de votre application, l' `ExecutionContext` utilisé pour exécuter votre code. 

## Modularisation

Le projet Play a été découpé en plusieurs sous-projets, vous permettant de selection un ensemble minimal de dépendences pour votre projet.

Vous devez choisir l'ensemble précies de dependences optionnelles dont vous avez besoin, parmis la liste:

- `jdbc` : Le pool de connexion **JDBC** et l'API `play.api.db`. 
- `anorm` : Le composant **Anorm**.
- `javaCore` : L'API core **Java**.
- `javaJdbc` : L'API base de donnée Java.
- `javaEbean` : Le plugin Ebean pour Java.
- `javaJpa` : Le plugin JPA pour Java.
- `filters` : Un ensemble de filtre "build-in" pour Play (tel que le filtre CSRF)

Le projet core `play` tire maintenant un très petit d'ensemble de dépendences externe et peut être utilisé comme un serveur HTTP asynchrone et haute performance sans les autres composants.

## Permettre plus de modularité pour vos projets

Afin de vous permettre de composer et de réutiliser vos composant dans vos prochains projets, Play 2.1 supporte la composition de sous-routers.

Par exemple, un sous projet peut definir son propre composant router et utilisant son espace de nom, tel que:

Dans `conf/my.subproject.routes`

```
GET   /                   my.subproject.controllers.Application.index
```

And then, you can integrate this component into your main application, by wiring the Router, such as:

```
# The home page
GET   /                   controllers.Application.index

# Include a sub-project
->    /my-subproject      my.subproject.Routes

# The static assets
GET   /public/*file       controllers.Assets.at(file)
```

In the configuration, at runtime a call to the `/my-subproject` URL will eventually invoke the `my.subproject.controllers.Application.index` Action.

> Note: in order to avoid name collision issues with the main application, always make sure that you define a subpackage within your controller classes that belong to a sub project (i.e. `my.subproject` in this particular example). You'll also need to make sure that the subproject's Assets controller is defined in the same name space.

More information about this feature can be found at [[Working with sub-projects|SBTSubProjects]].


## `Http.Context` propagation in the Java API

In Play 2.0, the HTTP context was lost during the asynchronous callback, since these code fragment are run on different thread than the original one handling the HTTP request.

Consider:

```
public static Result index() {
  return async(
    aServiceSomewhere.getData().map(new Function<String,Result>(data) {
      // Ouch! You try to access the request data in an asynchronous callback
      String user = session().get("user"); 
      return ok("Here is the result " + user + ": " + data);
    })
  );
}
```

This code wasn't working this way. For really good reason, if you think about the underlying asynchronous architecture, but yet it was really surprising for Java developers.

We eventually found a way to solve this problem and to propagate the `Http.Context` over a stack spanning several threads, so this code is now working this way.

## Better threading model for the Java API

While running asynchronous code over mutable data structures, chances are right that you hit some race conditions if you don't synchronize properly your code. Because Play promotes highly asynchronous and non-blocking code, and because Java data structure are mostly mutable and not thread-safe, it is the responsibility of your code to deal with the synchronization issues.

Consider:

```
public static Result index() {

  final HashMap<String,Integer> result = new HashMap<String,Integer>();

  aService.doSomethingAsync().map(new Function<String,String>(key) {
    Integer i = result.get(key);
    if(i != null) {
      result.set(key, i++);
    }
    return key;
  });

  aService.doSomethingElse().map(new Function<String,String>(key) {
    result.remove(key);
    return null;
  });

  ...
}
```

In this code, chances are really high to hit a race condition if the 2 callbacks are run in the same time on 2 different threads, while accessing each to the share `result` HashMap. And the consequence will be probably some pseudo-deadlock in your application because of the implementation of the underlying Java HashMap.

To avoid any of these problems, we decided to manage the callback execution synchronization at the framework level. Play 2.1 will never run concurrently 2 callbacks for the same `Http.Context`. In this context, all callback execution will be run sequentially (while there is no guarantees that they will run on the same thread).

## Managed Controller classes instantiation

By default Play binds URLs to controller methods statically, that is, no Controller instances are created by the framework and the appropriate static method is invoked depending on the given URL. In certain situations, however, you may want to manage controller creation and that’s when the new routing syntax comes handy.

Route definitions starting with @ will be managed by the `Global::getControllerInstance` method, so given the following route definition:

```
GET     /                  @controllers.Application.index()
```

Play will invoke the `getControllerInstance` method which in return will provide an instance of `controllers.Application` (by default this is happening via the default constructor). Therefore, if you want to manage controller class instantiation either via a dependency injection framework or manually you can do so by overriding getControllerInstance in your application’s Global class.

As this example [demonstrates it](https://github.com/guillaumebort/play20-spring-demo), it allows to wire any dependency injection framework such as __Spring__ into your Play application.

## New Scala JSON API

The new Scala JSON API provide great features such as transformation and validation of JSON tree. Check the new documentation at the [[Scala Json combinators document|ScalaJsonCombinators]].

## New Filter API and CSRF protection

Play 2.1 provides a new and really powerful filter API allowing to intercept each part of the HTTP request or response, in a fully non-blocking way.

For that, we introduced a new new simpler type replacing the old `Action[A]` type, called `EssentialAction` which is defined as:

```
RequestHeader => Iteratee[Array[Byte], Result]
```

As a result, a filter is simply defined as: 

```
EssentialAction => EssentialAction
```

> __Note__: The old `Action[A]` type is still available for compatibility.

The `filters` project that is part of the standard Play distribution contain a set of standard filter, such as the `CSRF` providing automatic token management against the CSRF security issue. 

## RequireJS

In play 2.0 the default behavior for Javascript was to use google closure's commonJS module support. In 2.1 this was changed to use [requireJS](http://requirejs.org/) instead.

What this means in practice is that by default Play will only minify and combine files in stage, dist, start modes only. In dev mode Play will resolve dependencies client side.

If you wish to use this feature, you will need to add your modules to the settings block of your build file:

```
requireJs := "main.js"
```

More information about this feature can be found on the [[RequireJS documentation page|RequireJS-support]].

## Content negotiation

The support of content negotiation has been improved, for example now controllers automatically choose the most appropriate lang according to the quality values set in the `Accept-Language` request header value.

It is also easier to write Web Services supporting several representations of a same resource and automatically choosing the best according to the `Accept` request header value:

```
val list = Action { implicit request =>
  val items = Item.findAll
  render {
    case Accepts.Html() => Ok(views.html.list(items))
    case Accepts.Json() => Ok(Json.toJson(items))
  }
}
```

More information can be found on the content negotiation documentation pages for [[Scala|ScalaContentNegotiation]] and [[Java|JavaContentNegotiation]].

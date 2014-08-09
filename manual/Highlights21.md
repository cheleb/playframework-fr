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

Ensuite, vous pouvez integrer ce composant dans l'application principal, en cablant le routeur:

```
# La page d'accueil
GET   /                   controllers.Application.index

# Inclure un sous-projet
->    /my-subproject      my.subproject.Routes

# Les resources statiques
GET   /public/*file       controllers.Assets.at(file)
```

Dans la configuration, à l'exécution un appel à l'URL `/my-subproject` URL invoquera éventuellemnt l'action `my.subproject.controllers.Application.index`.

> Remarque: afin d'éviter les collisions avec l'application principale, assurez vous de définir un sous package au seins de vos classes de controleur qui appartienent à un sous projet(ici `my.subproject`).
> 
> Vous devrez également vous assurer que le controleur de resources du sous projet soit défini dans le même espace de nom.

Plus de détail à ce sujet peut être trouvé ici [[Travailler avec des sous projets|SBTSubProjects]].


## Propagation de `Http.Context` dans l'API Java

Dans Play 2.0, le context HTTP était perdu lors de l'appel asynchrone, du fait ce ce fragment de code s'execute dans une thread différent de celui qui a traité la requête HTTP.

Considérons:

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

Ce code ne fonctionnait pas. Pour de très bonne raison, si vous pensez à l'architecture asynchrone sous-jacente, mais cela restait vraiment surprenant pour les développeurs Java.

Nous avons finalement trouvé un moyen de résoudre ce problème et à propager le `Http.Context` sur ​​une pile sur plusieurs threads, ainsi maintenant ce code fonctionne.


## Un meilleur modèle de thread pour l'API Java

Lors de l'exécution du code asynchrone sur des structures de données mutables, les chances sont grandes que vous recontriez des situations de compétition (ndt: "race conditions") si vous ne synchronisez pas correctement votre code. Du fait que Play favorise fortement le code asynchrone et non-bloquant, et parce que les structures de données Java sont pour la plupart mutable et non thread-safe, il est de la responsabilité de votre code de faire face aux problèmes de synchronisation.

Considérons:

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

Dans ce code, les chances sont très élevés de recontrer une condition de concurrence lors de l'accès à la HashMap `result`, si les deux fonctions de rappels sont exécutés en même temps sur deux threads différents. Et la conséquence sera probablement un pseudo-interblocage dans votre application en raison de l'implémentation de la HashMap en Java. 

Pour éviter un de ces problèmes, nous avons décidé de gérer la synchronisation de l'exécution de rappel au niveau du framework. Play 2.1 n'executera jamais simultanément deux fonctions de rappels pour le même `Http.Context`. Dans ce contexte, toutes les exécutions de fonction rappel sera exécutée séquentiellement (bien qu'il n'y ai aucune garantie qu'ils s'execute sur le même thread).

## Instantiation des classes de Controlleur

Par défaut Play lie les URLs aux méthodes statiques de contrôleur, auncune classes de contoleur ne sont créés par le framework, seule la méthode statique appropriée est appelée en fonction de l'URL donnée. Dans certains cas, cependant, vous voudrez peut-être gérer la création du contrôleur et c'est alors que la nouvelle syntaxe de routage est très utile. 

Les définitions de Route commençant par @ seront gérés par la méthode `Global::getControllerInstance`, compte tenu de la définition de la route suivante:

```
GET     /                  @controllers.Application.index()
```

Play invoquera la méthode `getControllerInstance`, qui en retour fournira une instance `controllers.Application` (en appellant le constucteur par default). Par conséquent, si vous souhaitez gérer l'instanciation de classe de contrôleur soit par un framework de l'injection de dépendance soit manuellement, vous pouvez le faire en remplaçant GetControllerInstance dans classe Global de votre application.

Comme dans cet exemple [demonstrates it](https://github.com/guillaumebort/play20-spring-demo), qui permet d'utiliser n'importe quel frameword d'injection de dépendence comme __Spring__ dans votre application Play.

## Nouvelle API JSON pour Scala

La nouvelle API JSON pour Scala fournie des super fonctionnalités comme des transformations et des validations de l'arbre JSON. Reportez vous à la nouvelle documentation [[Scala Json combinators document|ScalaJsonCombinators]].

## Nouvelle API de filtre et protection CSRF

Play 2.1 fournie une nouvelle et vraiment puissante API de filtre qui permet d'intercepter chaque partie d'une requête ou réponse HTTP, de manière non bloquante.

A cette fin, nous avons introduit un nouveau type plus simple pour remplacer l'ancient `Action[A]`, nommé `EssentialAction` qui est défini ainsi:

```
RequestHeader => Iteratee[Array[Byte], Result]
```

Un filtre ce définit simplement part: 

```
EssentialAction => EssentialAction
```

> __Note__: L'ancien type `Action[A]` est toujours disponible pour la compatibilité.



Le projet `filters` qui fait partie de la distribution Play standard, contient un ensemble de  filtre standard, tel que le `CSRF` qui fournit une gestion automatique de token pour lutter contre les problèmes de sécurité liés au CSRF. 

## RequireJS

Dans Play 2.0 le comportement par défault pour Javascript était d'utiliser le module commonJS de Google closure. En 2.1 cela a changé en faveur de [requireJS](http://requirejs.org/).

En practique, cela signifie que par défault, Play ne minifiera et combinera les fichiers seulement dans les modes stage, dist et start. Dans le mode dev, Play resoudre les dépendances coté client.

Si vous voulez utiliser cette fonction, vous devez juste ajouter vos modules à la configuration:

```
requireJs := "main.js"
```
Plus de détails sur cette fonctionnalité peuvent être trouvés sur [[RequireJS documentation page|RequireJS-support]].

## Négotiation du contentu

Le support pour la négociation du contenu a été amélioré, par exemple désormais les controleurs choisissent automatiquement la langue en fonction des paramètres du header `Accept-Language` de la requête.

Il est également très simple d'écrire des Services Web qui supportent plusieurs représentation pour une même ressource et qui choisissent au mieux en fonction de la valeur de l'entête  `Accept` de la requête:

```
val list = Action { implicit request =>
  val items = Item.findAll
  render {
    case Accepts.Html() => Ok(views.html.list(items))
    case Accepts.Json() => Ok(Json.toJson(items))
  }
}
```
Plus d'information peut être trouvé sur les pages de documentation sur la négociation de contenue pour  [[Scala|ScalaContentNegotiation]] and [[Java|JavaContentNegotiation]].

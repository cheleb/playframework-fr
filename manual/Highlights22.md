<!--- Copyright (C) 2009-2013 Typesafe Inc. <http://www.typesafe.com> -->
# Quoi de neuf dans Play 2.2

## Nouvelle structure de resultats pour Java et Scala

Auparavant, les résultats pouvaient être soit simple ou asynchrone, fractionné ou simple. Avoir à traiter avec tous ces différents types rendait la composition d'action et les filtres difficiles à mettre en œuvre, car, souvent, il y avait une fonctionnalité qui devait être appliquée à tous les types de résultats, mais le code devait déballer récursivement les résultats asynchrones et d'appliquer la même logique à fragmenté et des résultats simples. 

Il a également créé une distinction artificielle entre actions asynchrone et synchrone dans Play, qui ont provoqué la confusion, conduisant les gens à penser que Play pourrait fonctionner dans un mode synchrone et asynchrone. En fait, Play est 100% asynchrone, la seule chose qui différencie si un résultat est retourné de façon asynchrone ou non est de savoir si d'autres actions asynchrones, tels que IO, doivent être faites pendant le traitement de l'action.

Ainsi nous avons simplifié la stucture de résultat pour Java et Scala. Il y a maintenant qu'un seul type de résultat, `SimpleResult`.  La super classes `Result` est toujours utilisée mais elle est marqué comme dépréciée.

Dans les applications Java, cela signifie que les actions doivent juste retourner `Promise<SimpleResult>` si elle veulent que leur traitement soit asynchrone, en Scala il faut utiliser le constructeur d'action `async`:

```scala
def index = Action.async {
  val foo: Future[Foo] = getFoo()
  foo.map(f => Ok(f))
}
```

## Un meilleur controle du buffering et du keep alive

Comment et quand Plays bufferise les resulats est maintenant mieux exprimé dans l'API Scala, [`SimpleResult`](api/scala/index.html#play.api.mvc.SimpleResult) possède une nouvelle propriété nommée `connection`, qui est du type [`HttpConnection`](api/scala/index.html#play.api.mvc.HttpConnection$).

Si elle est positionée à `Close`, la réponse sera fermée des que le contenu sera envoyé, et aucun buffering ne sera effectif. Si elle est positionné à `KeepAlive`, Play fera de son mieux pour conserver la connexion, en accord avec la spécification HTTP, bufferisant la réponse seulement si l'encodage de transfert et la taille du contenue ne sont pas précisés.

## Nouvelle composition d'action et methodes de constructeur d'action

Nous fournissons désormais un trait [`ActionBuilder`](api/scala/index.html#play.api.mvc.ActionBuilder) pour les applications Scala qui permet des constructions plus puissantes de piles d'actions. Par exemple:

```scala
object MyAction extends ActionBuilder[AuthenticatedRequest] {
  def invokeBlock[A](request: Request[A], block: (AuthenticatedRequest[A]) => Future[SimpleResult]) = {
    // Authenticate the action and wrap the request in an authenticated request
    getUserFromRequest(request).map { user =>
      block(new AuthenticatedRequest(user, request))
    } getOrElse Future.successful(Forbidden)
  }

  // Compose une action avec une logging action, une action qui gère le CSRF checking, et une action qui authorise seulement HTTPS
  def composeAction[A](action: Action[A]) =
    LoggingAction(CheckCSRF(OnlyHttpsAction(action)))
}
```

Le constucteur d'action résultant peut être utilisé comme l'objet `Action`, avec un analyseur optionnel et les paramètres de requête, et une variantes asynchrone. Le type du paramètre de la requête passé à l'action sera le type spécifié dans le constructeur, dans ce cas, `AuthenticatedRequest`:

```scala
def save(id: String) MyAction(parse.formUrlEncoded) = { request =>
  Ok("User " + request.user + " saved " + request.body)
}
```

## Une API de Promise Java améliorée

La classe Java Promise a été améliorée afin d'approcher ses fonctionnalité des Futures et des Promises de Scala. En particulier, le contexte d'exécution peut maintenant être passé aux méthodes de Promise.

## Contexte d'exécution et librairies Iteratee

Les contextes d'exécution sont maintenant nécessaires lors des appels aux méthodes d'Iteratee, Enumeratee et Enumerator. Exposer les contextes d'exécution pour les libraries d'Iteratee fournie un contrôle plus fin lors de l'exécution et peut conduire dans certains cas à un gain de performance.

Les contextes d'exécution peuvent être fournis implicitement, ce qui signifie qu'il y a peu d'impact sur le code existant qui utilise des Iteratees.

## Support de sbt 0.13

Il y a eu diverses améliorations d'utilisabilité et de performance.

Une amélioration d'utilisabilité est nous supportons désormais les fichiers `build.sbt` pour construire les projets Play. Par exemple `samples/java/helloworld/build.sbt`:

```scala
import play.Project._

name := "helloworld"

version := "1.0"

playJavaSettings
```

`playJavaSettings` déclare tout ce qui est nécessaire pour un projet Java. De même `playScalaSettings` existe pour les projets Play Scala. Consultez les exemples de projets pour des exemples de cette nouvelle configuration. Notez que la méthode précédente qui utilise build.scala avec play.Project` `est toujours supportée.

Pour plus d'information sur ce qui a changé pour sbt 0.13 se référer aux [release notes](http://www.scala-sbt.org/0.13.0/docs/Community/ChangeSummary_0.13.0.html)

## Les tâches stage et dist

Les tâches _stage_ et _dist_ tasks ont été complètement remanié afin d'exploiter le [Native Packager Plugin](https://github.com/sbt/sbt-native-packager).

L'interêt d'utiliser le packager natif réside dans le fait que plusieurs type d'archive sont maintenant supportés en plus des fichier zip,  tar.gz, RPM, OS X disk images, Microsoft Installers (MSI) et plus. De plus, un script batch pour Windows est maintenant fournie pour Play ainsi que pour Unix.

Plus d'information peut être trouver dans [[Creating a standalone version of your application|ProductionDist]].

## Support pour gzip en standard

Play supporte maintenant la compression gzip pour toutes les requêtes. Pour savoir comment l'activer, voir [[Configuring gzip encoding|GzipEncoding]].

## JAR de documentation 

La distribution de Play range désormais sa documentation dans un JAR plutôt que dans un répertoire. Les fichers JAR sont plus pratiques.

Comme dans Play 2.1, vous pouver consulter la documentation lorsque vous [[exécutez votre appication Play en mode dévelopment |PlayConsole]] en allant sur l'adresse speciale [`/@documentation`](http://localhost:9000/@documentation).

Si vous voulez accéder à la documentation sous forme de fichers, ils sont dans le JAR `play-docs` contenu dans la distribution.

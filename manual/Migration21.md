<!--- Copyright (C) 2009-2013 Typesafe Inc. <http://www.typesafe.com> -->
# Play 2.1 guide de migration
  
Ce guide concerne la migration vers Play 2.1 de Play 2.0.

Pour migrer une application **Play 2.0.x** vers une **Play 2.1.0** commencez par mettre à jour `sbt-plugin` de Play dans le fichier `project/plugins.sbt`:

```
addSbtPlugin("play" % "sbt-plugin" % "2.1.0")
```

Maintenant mettez à jour le fichier `project/Build.scala` afin d'utiliser la nouvelle classe `play.Project` à la place de `PlayProject`:

Commencer par les imports:

```
import play.Project._
```

Ensuite la création du `main` project:

```
val main = play.Project(appName, appVersion, appDependencies).settings(
```

Enfin, mettez à jour le fichier `project/build.properties`:

```
sbt.version=0.12.2
```

Ensuite clean et re-compiler votre projet en utilisant la commande `play` de la distribution **Play 2.1.0**:

```
play clean
play ~run
```
Si vous rencontrez des erreurs de compilation, ce document vous aidera à comprendre ce qui a été déprécié ou les incompatibilités qui ont provoquer les erreurs.

## Changement dans le fichier de build

Du fait que Play 2.1 apporte plus de modularité, vous devez maintenant explicitement définir les dépendences nécessaire à votre application. Par defaut tout `play.Project` contiendra seulement une dépendence vers le cœur de la librairie Play.  Vous devez choisir l'ensemble de dépendences optionnelle dont votre projet à besoin. Voici les nouvelles dépendences modularisées dans **Play 2.1**:

- `jdbc` : Le pool de connexion **JDBC** et l'API `play.api.db`.
- `anorm` : Le composant **Anorm**.
- `javaCore` : Le cœur de l'API **Java**.
- `javaJdbc` : L'API base de donnée Java.
- `javaEbean` : Le plugin Ebean pour Java.
- `javaJpa` : Le plugin JPA pour Java.
- `filters` : Un ensemble de filtre intégrés à Play (tels que le filtre CSRF).

Voici un fichier `Build.scala` typique pour **Play 2.1**:

```
import sbt._
import Keys._
import play.Project._

object ApplicationBuild extends Build {

    val appName         = "app-name"
    val appVersion      = "1.0"

    val appDependencies = Seq(
       javaCore, javaJdbc, javaEbean
    )

    val main = play.Project(appName, appVersion, appDependencies).settings(
      // Add your own project settings here      
    )

}
```

Le paramètre `mainLang` pour le projet n'est plus nécessaire. Le language est déterminé à partir des dépendences du projet. Si les dépendences contiennent `javaCore` alors le language est `JAVA` sinon c'est `SCALA`. Remarquez la dépendences modulaire dans la section `appDependencies`. 

## play.mvc.Controller.form() renommé en play.data.Form.form()

En relation avec la modularisation, la package `play.data` et ses dépendences ont été déplacé du cœur de Play vers l'artéfact `javaCore`. En conséquence de cela, `play.mvc.Controller#form` a été renommé en `play.data.Form#form`

## play.db.ebean.Model.Finder.join() renommé en fetch()

Dans le cadre du nettoyage de l'API Finder, les méthodes "join" ont été replacées par des méthodes "fetch". Leur comportement est inchangé.

## Le Promise de Play sont remplacées par les Future de Scala
Avec l'introduction des `scala.concurrent.Future` de Scala 2.10 l'écosystem scala a fait un grand pas vers l'unification des différetes librairies de Futures et de Promises.

Du fait que Play utilise `scala.concurrent.Future` on peut maintenant combiner les futures/promises qui viennent des APIs internes ou externes.

> Les utilisateurs Java users continueront à utiliser le wrapper de Play autour des scala.concurrent.Future pour l'instant. 

Considerons le bout de code suivant:


```
import play.api.libs.iteratee._
import play.api.libs.concurrent._
import akka.util.duration._

def stream = Action {
    AsyncResult {
      implicit val timeout = Timeout(5.seconds)
      val akkaFuture =  (ChatRoomActor.ref ? (Join()) ).mapTo[Enumerator[String]]
      //convert to play promise before sending the response
      akkaFuture.asPromise.map { chunks =>
        Ok.stream(chunks &> Comet( callback = "parent.message"))
      }
    }
  }
  
```

En utilisant la nouvelle `scala.concurrent.Future` cela devient:

```
import play.api.libs.iteratee._
import play.api.libs.concurrent._
import play.api.libs.concurrent.Execution.Implicits._

import scala.concurrent.duration._

  def stream = Action {
    AsyncResult {
      implicit val timeout = Timeout(5.seconds)
      val scalaFuture = (ChatRoomActor.ref ? (Join()) ).mapTo[Enumerator[String]]
      scalaFuture.map { chunks =>
        Ok.stream(chunks &> Comet( callback = "parent.message"))
      }
    }
  }
```

Notez les nouveaux imports pour:

- Nouvel import pour le contexte d'execution `play.api.libs.concurrent.Execution.Implicits`
- Le changement pour les duration `scala.concurrent.duration` (au lieu de l'utilisation de l'API Akka).
- La méthode `asPromise` a été enlevée.

D'une manière générale, si vous voyez le message d'erreur: "error: could not find implicit value for parameter executor", vous devez surement ajouter:

```
import play.api.libs.concurrent.Execution.Implicits._
```

_(Voir la [documentation Scala  à propos du context d'Execution](http://docs.scala-lang.org/overviews/core/futures.html) pour plus d'information)_

Et souvenez vous que:

- Une `Promise` Play est maintenant une Future Scala
- Un `Redeemable` Play est maintent une `Promise` Scala

## Changements de l'API JSON de Scala

**Play 2.1** vient avec une nouveau validateur de JSON Scala et un "path navigator". Ces nouvelles APIs case la compatibilité avec les parsers JSON actuels.

La signature de `play.api.libs.json.Reads` a changé. Par exemple:

```
trait play.api.libs.json.Reads[A] {
  self =>

  def reads(jsValue: JsValue): A

}
```

En 2.1 cela devient:

```
trait play.api.libs.json.Reads[A] {
  self =>

  def reads(jsValue: JsValue): JsResult[A]

}
```

Ainso, dans **Play 2.0** une implementation d'un sérialiseur JSON serializer pour le type `User` était:

```
implicit object UserFormat extends Format[User] {

  def writes(o: User): JsValue = JsObject(
    List("id" -> JsNumber(o.id),
      "name" -> JsString(o.name),
      "favThings" -> JsArray(o.favThings.map(JsString(_)))
    )
  )

  def reads(json: JsValue): User = User(
    (json \ "id").as[Long],
    (json \ "name").as[String],
    (json \ "favThings").as[List[String]]
  )

}
```

Dans **Play 2.1** vous devrez changer le code comme il suit: 

```
implicit object UserFormat extends Format[User] {

  def writes(o: User): JsValue = JsObject(
    List("id" -> JsNumber(o.id),
      "name" -> JsString(o.name),
      "favThings" -> JsArray(o.favThings.map(JsString(_)))
    )   
  )   

  def reads(json: JsValue): JsResult[User] = JsSuccess(User(
    (json \ "id").as[Long],
    (json \ "name").as[String],
    (json \ "favThings").as[List[String]]
  ))  

}
```

L'API pour générer du JSON a également évolué:

```
val jsonObject = Json.toJson(
  Map(
    "users" -> Seq(
      toJson(
        Map(
          "name" -> toJson("Bob"),
          "age" -> toJson(31),
          "email" -> toJson("bob@gmail.com")
        )
      ),
      toJson(
        Map(
          "name" -> toJson("Kiki"),
          "age" -> toJson(25),
          "email" -> JsNull
        )
      )
    )
  )
)
```

Avec **Play 2.1** cela devient:

```
val jsonObject = Json.obj(
  "users" -> Json.arr(
    Json.obj(
      "name" -> "Bob",
      "age" -> 31,
      "email" -> "bob@gmail.com"
    ),
    Json.obj(
      "name" -> "Kiki",
      "age" -> 25,
      "email" -> JsNull
    )
  )
)
```
Plus d'information au sujet de ces fonctionnalités peut être trouvé [[à la documentation JSON|ScalaJson]].

## Changement de la gestion des Cookies

Suite à une changement dans _JBoss Netty_, les cookies sont marqués transient en affectant leur `maxAge` à `null` ou `None` (fonction de l'API) au lieu de mettre  `maxAge` à -1.  Toute valeur inférieure ou égale à 0 de `maxAge` fera expirer le cookie immediatement.

Les méthodes `discardingCookies(String\*)` (Scala) et `discardCookies(String...)` (Java) sur `SimpleResult` sont devenues obsolète, since these methods are unable to handle cookies set on a particular path, domain or set to be secure.  Please use the `discardingCookies(DiscardingCookie*)` (Scala) and `discardCookie` (Java) methods instead.

## RequireJS

In **Play 2.0** the default behavior for Javascript was to use Google's Closure CommonJS module support. In **Play 2.1** this was changed to use RequireJS instead.

What this means in practice is that by default Play will only minify and combine files in stage, dist, start modes only. In dev mode Play will resolve dependencies client side.

If you wish to use this feature, you will need to add your modules to the settings block of your `project/Build.scala` file:

```
requireJs := "main.js"
```

More information about this feature can be found on the [[RequireJS documentation page|RequireJS-support]].

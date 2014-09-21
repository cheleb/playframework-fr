<!--- Copyright (C) 2009-2013 Typesafe Inc. <http://www.typesafe.com> -->
# Play 2.2 Guide de Migration 

Ce guide concerne la migration vers Play 2.2 de Play 2.1.  Pour migrer vers Play 2.1, commencez par le  [[Guide de Migration Play 2.1|Migration21]].

## Build tasks

### Mise à jour de l'organisation de Playet de la version

Play est désormais publié sous un id d'organisation différent.  Ainsi nous pourrons éventuellement déployer Play sur le Maven Central.  L'ancien id d'organisation était `play`, le nouveau est `com.typesafe.play`.

La version doit également être changée à 2.2.0.

Dans `project/plugins.sbt`, changez le plugin Play plugin afin d'utiliser le nouveau id d'organisation:

```scala
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.2.0")
```

De plus, si vous dépendez d'autres artefacts de Play, sans utiliser de Helper, vous devrez probablement corriger ces id d'organisation et ces versions.

### Mise à jour de la version de SBT

`project/build.properties` doit être changé afin d'utiliser sbt 0.13.0.

### Mise à jour du projet root

Si vous utilisez un build multi-project, et qu'aucun de ces projets ne possède de répertoire root sur le répertoire courant, maintenant le projet root est détérminé en surchargeant rootProject au lieu de l'ordre alphabétique:

```scala
override def rootProject = Some(myProject) 
```

### Mise à jour de la version de Scala

Si vous avez positionné scalaVersion (e.g. parce que vous avez un build de multi-projet qui utilise Project en plus de play.Project), vous devriez le corriger en 2.10.2.

### Module de cache de Play

Le cache de Play est maintenant séparé dans son propre module. Si vous utilisez le cache de Play, vous devrez cette l'ajouter comme dépendence. Par exemple, dans `Build.scala`:

```scala
val addDependencies = Seq(
  jdbc,
  cache,
  ...
)
```

Notez que si vous dépendez de plugins qui dependent d'une version de Play antérieure a 2.2, vous obtiendrez des conflits car plusieurs cache seront chargés. Mettez à jour la version de ces plugins ou assurez vous que les vieilles de Play sont exclues si vous rencontrait ce problème.

### L'espace de nom de sbt n'est plus étendu

L'espace de nom de `sbt` était étendu par Play e.g. `sbt.PlayCommands.intellijCommandSettings`. Ceci est considéré comme une mauvaise pratique et en conséquence Play utilise desormais son propre espace de nom pour les éléments en relation avec sbt:  `play.PlayProject.intellijCommandSettings`.

## Nouvelles structure de resultats en Scala

Afin de simplifier la composition et le filtrage des actions, les structures de resultats ont été simplifiées. Il n'y a maintenant plus qu'un type de resultat, `SimpleResult`, là où avant il y avait  `SimpleResult`, `ChunkedResult` et `AsyncResult`, plus les interfaces `Result` and `PlainResult`.  Tous à l'exeption de `SimpleResult` ont été marquées obsolètes.  `Status`, une sous classes de `SimpleResult`, existe toujours comme une classes de confort pour construire les resultats.  Dans la plupart des cas, les actions peuvent toujours utiliser les types obsolètes, mais cela génèrerera les deprecation warnings.  Les actions qui composent ou filtre devront utiliser `SimpleResult`.

### Action Async

Avant, là où vous aviez le code suivant:

```scala
def asyncAction = Action {
  Async {
    Future(someExpensiveComputation)
  }
}
```

Vous pouvez maintenant utiliser le constructeur [`Action.async`](api/scala/index.html#play.api.mvc.ActionBuilder):

```scala
def asyncAction = Action.async {
  Future(someExpensiveComputation)
}
```

### Travailler avec des resultats fragmentés

Avant, la méthode `stream` de `Status` était utilisé afin de produire des résultats fragmentés.  Ceci est devenu obsolète, remplacé par la méthode  [`chunked`](api/scala/index.html#play.api.mvc.Results$Status), qui est plus explicite sur le fait que le résultats sera fragmenté.  Par exemple:

```scala
def cometAction = Action {
  Ok.chunked(Enumerator("a", "b", "c") &> Comet(callback = "parent.cometMessage"))
}
```

Les utilisations avancées qui créait ou utilisait `ChunkedResult` directement doivent être changées avec un code qui définit/vérifie manuellement les entêtes `TransferEncoding: chunked`, et utilise le nouveau `Results.chunk` ainsi que les enumeratees `Results.dechunk`.

### Composition d'action 

Nous recommandons que la composition d'action soit faite en utilisant l'implémention [`ActionBuilder`](api/scala/index.html#play.api.mvc.ActionBuilder) pour créer les actions.

Des détails sur comment faire cela peuvent être trouvés [[là|ScalaActionsComposition]].

### Filtres

Les iteratees produits par `EssentialAction` produisent désormais des  `SimpleResult` au lieu de `Result`.  Ce qui signifie que les filtres qui doivent fonctionner le resultat non plus à déballer `AsyncResult` en un `PlainResult`, ce qui sans aucun doute rend les filtres beaucoup plus simple et plus facile à écrire. Le code qui auparavant déballait peut en général être remplacé par un unique appel `map` de iteratee.

### Application play.api.http.Writeable 

Auparavant le constucteur de `SimpleResult` prenait un `Writeable` pour le type de l'`Enumerator` passé en paramètre.  Maintenant l'enumerator doit être un `Array[Byte]`, et `Writeable` est seulement utilisé par les méthodes pratiques de `Status`.

### Tests

Avant `Helpers.route()` et les méthodes similaire returnaient un `Result`, qui était toujours un `AsyncResult`, les autres méthodes sur `Helpers` telles que `status`, `header` et `contentAsString` prenaient `Result` comme paramètre.  Maintenant un `Future[SimpleResult]` est retourné par `Helpers.route()`, et accepté par les méthodes d'extraction.  En général, lorque l'inference de type est utilisé,  aucun changement doit être nécessaire pour le code de test.

## Nouvelle structure de résultats en Java

Afin de simplifier la composition d'action, la structute Java des résultats a été changée. `AsyncResult` est devenue obsolète, et `SimpleResult` a été introduit, afin de distinguer les résultats normaux des `AsyncResult`.

### Actions Async

Avant, les futures dans les action asycn étaient enveloppées dans un appel `async`. Maintenant, les actions peuvent retourner soit un `Result` soit une `Promise<Result>`.  Par exemple:

```java
public static Promise<Result> myAsyncAction() {
  Promise<Integer> promiseOfInt = Promise.promise(
    new Function0<Integer>() {
      public Integer apply() {
        return intensiveComputation();
      }
    }
  );
  return promiseOfInt.map(
    new Function<Integer, Result>() {
      public Result apply(Integer i) {
        return ok("Got result: " + i);
      }
    }
  );
}
```

### Composition d'action

La signature de la méthode `call` dans `play.mvc.Action` a changé afin de  retourner une `Promise<SimpleResult>`.  Si rien n'est fait avec le résutat, alors il siffira juste de changer la signature de la méthode.

## Contexte d'exécution des Iteratees

Iteratees, enumeratees and enumerators qui exécute le code fourni par l'application on maintenant besoin d'un contexte d'exécution implicit. Par exemple:

```scala
import play.api.libs.concurrent.Execution.Implicits._

Iteratee.foreach[String] { msg =>
  println(msg)
}
```

## Exécution concurrente de F.Promise

Le moyen par lequel la classe [`F.Promise`](api/java/play/libs/F.Promise.html)  exécute le code fournit par l'utilisateur a changé en Play 2.2.

Dans Play 2.1, la classe `F.Promise` restreignait la manière dont le code était exécuté. Les opérations promises pour une requête HTTP s'exécutaientdans l'ordre ou, en séquence.

Avec Play 2.2, cette restriction sur l'ordre a disparue de telle manière que  les opéarations promises peuvent s'exécuter en concurrence. Le travail exécuté par la classe `F.Promise` utilise [[le pool de thread par default de Play|ThreadPools]] sans apporter plus de restrictions sur l'exécution.

Cependant, pour ceux qui le désire toujours, le comportement historique de Play 2.1's est conservé dans la classe `OrderedExecutionContext`. Ce comportement historique de Play 2.1 peut facilement être restauré en fournissant un `OrderedExecutionContext` comme argument à n'importe quelles des méthodes de `F.Promise`'.

Le code suivant illustre comment restaurer le comportement de Play 2.1 dans Play 2.2. Remarquez que cet exemple utilise les mêmes paramètres que Play 2.1: un pool de 64 acteurs s'exécutant au sein de l'`ActorSystem` par default  de Play.

````java
import play.core.j.OrderedExecutionContext;
import play.libs.Akka;
import play.libs.F.*;
import scala.concurrent.ExecutionContext;

ExecutionContext orderedExecutionContext = new OrderedExecutionContext(Akka.system(), 64);
Promise<Double> pi = Promise.promise(new Function0<Double>() {
  Double apply() {
    return Math.PI;
  }
}, orderedExecutionContext);
Promise<Double> mappedPi = pi.map(new Function<Double, Double>() {
  Double apply(x Double) {
    return 2 * x;
  }
}, orderedExecutionContext);
````

## Jackson Json
Nous avons mis à jour Jackson à la version 2 ceux qui signifie que l'espace de est maintenant `com.fasterxml.jackson.core` au lieu de `org.codehaus.jackson`.

## Préparer une distribution

Les tâches _stage_ et _dist_ ont été complètement ré-écrite en Play 2.2 de telle manière qu'elles utilise le [Native Packager Plugin](https://github.com/sbt/sbt-native-packager). 

Les distributions de Play distributions ne sont plus créées dans le répertoire `dist` du projet. A la place, ils sont créés dans le répertoire `target` du projet. 

Une autre chose qui a changé, c'est la place du script Unix qui démarre un application Play. Antérieurement à la 2.2 le script Unix était nommé `start` et résidait dans le répertoire racine de la distribution. Avec la 2.2, le script `start` est renommé avec le nom du projet et réside dans le répertoire  `bin` de la distribution. De plus, il y a un script `.bat` disponible pour démarrer l'application Play sous Windows.

> Remarquez que le format des arguments passés au script `start` script on changés. Donnez un `-h` au script `start` pour voir les argument attendus.

Consultez la documentation [["démarrer votre application en mode production"|Production]] pour plus d'information sur les nouvelles tâches `stage` et `dist`.

## Mise à jour de Akka 2.1 vers 2.2

Le guide de migration pour la mise à jour de Akka 2.1 vers 2.2 se trouve [là](http://doc.akka.io/docs/akka/2.2.0/project/migration-guide-2.1.x-2.2.x.html).

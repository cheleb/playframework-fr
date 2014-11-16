<!--- Copyright (C) 2009-2013 Typesafe Inc. <http://www.typesafe.com> -->
# Play 2.3 Guide de Migration

Ce guide concerne la migration vers Play 2.3 de Play 2.2. Pour migrer vers Play 2.2, commencez par suivre le [[Guide de Migration Play 2.2|Migration22]].

## Activator

Dans Play 2.3 la commande `play` est remplacée par `activator`. Play a évolué afin d'utiliser [Activator](https://typesafe.com/activator).

### La commande Activator

Toutes les fonctionnalités qui étaient disponible avec la commande `play` le sont également avec la commande `activator`.

* `activator new` pour créer un nouveau projet. Voir [[Créer une nouvelle application|NewApplication]].
* `activator` pour lancer la console. Voir [[Utiliser la console de Play|PlayConsole]].
* `activator ui` est une nouvelle commande qui lance un interface web..

> La nouvelle commande `activator` et l'ancienne commande `play`sont toutes deux des enveloppes autour de [sbt](http://www.scala-sbt.org/). Si vous préférez, vous pouvez utilisez la commande `sbt` directement. Cependant, en utilisant sbt directement vous passerez à coté de quelques fonctionnalités apportées par Activator, telles que les templates (`activator new`) et l'interface utilisateur web (`activator ui`). Tant sbt que Activator supportent tout les commandes classiques de console telles que `test` et `run`.

### Activator distribution

Play est distribué comme une distribution d'Activator qui contient toutes les dépendences de Play. Vous pouvez télécharger cette distribution à partir de la page [Play download](http://www.playframework.com/download).

Si vous préférez, vous pouvez également télécharger une version minimale (1MB) d'Activator à partir du [site Activator](https://typesafe.com/activator). Cherchez la "mini" distribution sur la page de téléchargement. La version minimale d'Activator téléchargera seulement les dépendences lorsqu'elle seront nécessaires.

Comme Activator est un wrapper autour de sbt, vous pouvez également télécharger et utiliser [sbt](http://www.scala-sbt.org/) directly, si vous préférez.

## Changement du Build

### sbt

Play utilise sbt 0.13.5. Si vous mettez à jour un projet existant, changez votre fichier `project/build.properties` en:

```
sbt.version=0.13.5
```

### Changement du Plugin

Changez la version du plugin de Play dans `project/plugins.sbt`:

```scala
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.3.XXX")
```

Où `2.3.XXX` est la version de Play que vous voulez utiliser.

Vous devrez également ajouter quelques plugins de sbt-web, voir la section *sbt-web* en dessous.

### Auto Plugins et plugin settings

sbt 0.13.5 apporte une nouvelle fonctionnalité appelée "auto plugins".

Auto plugins permet aux plugins sbt d'être déclarés dans le répertoire  `project`  (typiquement le `plugins.sbt`) comme avant. Ce qui a changé c'est que les plugins peuvent maintenant déclarer leurs dépendences envers d'autre plugins et ce qui déclenche leur activation pour un build donné. Avant auto plugins, les plugins ajoutés au build étaient toujours activés; maintenant les plugins sont activés sélectivement pour chaque modules.

Ce qui signifie que déclarer `addSbtPlugin` peut ne pas suffire pour des plugins qui utililisent "auto plugin". C'est une bonne chose. Vous pouvez maintenant choisir quels modules utilisent quels plugins, par ex:

```scala
lazy val root = (project in file(".")).enablePlugins(SbtWeb)
```
L'exemple ci dessus montre `SbtWeb` être ajouté au projet racine d'un build. Dans le cas de `SbtWeb`il y d'autres plugins qui vont s'activer, par exemple, si vous avez également ajouté `sbt-less-plugin` via `addSbtPlugin`, à ce moment il sera activé du fait que `SbtWeb` ait été  activé. `SbtWeb` est donc un plugin "racine" pour cette catégorie de plugin.

Play lui même utilise désormais le mécanisme d'auto plugins. Le mécanisme de Play 2.2 où `playJavaSettings` et `playScalaSettings` était utilisés a été enlevé. Maintenant vous devez utilisé à la place: 

```java
lazy val root = (project in file(".")).enablePlugins(PlayJava)
```

ou

```scala
lazy val root = (project in file(".")).enablePlugins(PlayScala)
```

Si auparavant vous utilisiez play.Project, par exemple un projet Scala:

```scala
object ApplicationBuild extends Build {

  val appName = "myproject"
  val appVersion = "1.0-SNAPSHOT"

  val appDependencies = Seq()

  val main = play.Project(appName, appVersion, appDependencies).settings(
  )

}
```

...alors vous pouvez continuer d'utiliser une approche similaire via sbt native:

```scala
object ApplicationBuild extends Build {

  val appName = "myproject"
  val appVersion = "1.0-SNAPSHOT"

  val appDependencies = Seq()

  val main = Project(appName, file(".")).enablePlugins(play.PlayScala).settings(
    version := appVersion,
    libraryDependencies ++= appDependencies
  )

}
```
En procédant de la sorte les settings sont maintenant automatiquement importés lorsqu'un plugin est activé.

Les clefs (keys) fournies par Play doivent maintenant être référencées au seins de l'objet `PlayKeys`. Par exemple pour to référencer `playVersion` vous devez soit importer les clefs:

```scala
import PlayKeys._
```

soit les qualifier complètement avec `PlayKeys.playVersion`.

En dehors des fichier `.sbt`, si vous utilisez Scala pour décrire vos build alors vous devrez écrire ces lignes afin d'avoir les `PlayKeys` disponibles:

```scala
import play.Play.autoImport._
import PlayKeys._
```

### Explicit scalaVersion

Play 2.3 supporte à la fois Scala 2.11 et Scala 2.10. Avant le plugin Play positionnait le setting sbt `scalaVersion` pour vous. Désormais vous devez indiquer quelle version de Scala vous voulez utiliser.

Mettez à jour votre `build.sbt` ou `Build.scala` en incluant la version de Scala:

Pourr Scala 2.11:

```scala
scalaVersion := "2.11.1"
```

Pour Scala 2.10:

```scala
scalaVersion := "2.10.4"
```

### sbt-web

La grande nouveauté de Play 2.3 est l'introduction de [sbt-web](https://github.com/sbt/sbt-web#sbt-web). En résumé sbt-web permet aux fonctionnalités  Html, CSS and JavaScript d'être extraite du cœur de Play en une ensemble de plugins sbt purs. Cela a deux avantages majeur pour vous:

* Play est moins dogmatique sur le HTML, CSS et JavaScript; et
* sbt-web peut avoir sa propre communauté and prospérer parallèlement à Play.

Il y a d'autres avantages, notamment le fait que les plugins SBT-Web sont en mesure de fonctionner au sein de la JVM via [Trireme](https://github.com/apigee/trireme#trireme), ou nativement en utilisant [Node.js](http://nodejs.org/). Notez que sbt-web est un environement de développement et ne participe pas à l'exécution d'une application Play. Trireme est utilisé par défault, mais si vous avez Node.js installé et voulez et voulez avoir des performance fulgurante pour votre build alors vous pouvez fournir une propriété système via la variable d'environnement sbt SBT_OPTS. Par exemple:

```bash
export SBT_OPTS="$SBT_OPTS -Dsbt.jse.engineType=Node"
```

Une caractéristique de sbt-web est qu'il s'intéresse pas au fait que vous utilisiez "javascripts" ou "stylesheets" comme nom de dossier. Tout fichier avec l'extension approprié sera filtré dans le dossier `app/assets`.

Une nuance avec SBT-web est que *toutes* les ressources sont servies à partir du dossier `public`. Par conséquent, si vous aviez précédemment des ressources à l'extérieur du dossier `public` c'est-à-dire que vous avez utilisé le paramètre` playAssetsDirectories` comme dans l'exemple suivant:

```scala
playAssetsDirectories <+= baseDirectory / "foo"
```

...alors vous devez maintenant faire la chose suivante:

```scala
unmanagedResourceDirectories in Assets += baseDirectory.value / "foo"
```
...cependant notez que les fichiers y seront regroupées dans le dossier public. Cela signifie qu'un fichier  "public / a.js" sera remplacé par le fichier "foo / a.js". Vous pouvez également utiliser des sous-dossiers dans votre dossier public comme espace de noms.

La suite liste tout les composants relatifs à sbt-web ainsi que leurs versions au moment de la la release Play 2.3.

#### Librairies
```scala
"com.typesafe" %% "webdriver" % "1.0.0"
"com.typesafe" %% "jse" % "1.0.0"
"com.typesafe" %% "npm" % "1.0.0"
```

#### sbt plugins
```scala
"com.typesafe.sbt" % "sbt-web" % "1.0.0"
"com.typesafe.sbt" % "sbt-webdriver" % "1.0.0"
"com.typesafe.sbt" % "sbt-js-engine" % "1.0.0"

"com.typesafe.sbt" % "sbt-coffeescript" % "1.0.0"
"com.typesafe.sbt" % "sbt-digest" % "1.0.0"
"com.typesafe.sbt" % "sbt-gzip" % "1.0.0"
"com.typesafe.sbt" % "sbt-less" % "1.0.0"
"com.typesafe.sbt" % "sbt-jshint" % "1.0.0"
"com.typesafe.sbt" % "sbt-mocha" % "1.0.0"
"com.typesafe.sbt" % "sbt-rjs" % "1.0.1"
```

#### WebJars

[WebJars](http://www.webjars.org/) joue maintenant un rôle important dans la fourniture des ressources d'une application Play. Par example vous pouvez déclarer que vous utilisez le populaire [Bootstrap library](http://getbootstrap.com/) simplement en ajoutant la dépendence suivant dans votre fichier de build:

```scala
libraryDependencies += "org.webjars" % "bootstrap" % "3.0.0"
```

Les WebJars sont automatiquement extraits dans un dossier `lib` relatif à vos ressources publique pour plus de commodité. Par exemple si vous déclarez une dépendence sur [RequireJs](http://requirejs.org/) alors vous pouvez la référencer d'une vue en un ligne telle que:

```html
<script data-main="@routes.Assets.at("javascripts/main.js")" type="text/javascript" src="@routes.Assets.at("lib/requirejs/require.js")"></script>
```

Notez le chemin `lib/requirejs/require.js`. Le dossier `lib` dénotes l'extraction de la ressource WebJar, le dossier `requirejs` correspond à l'artifactId de WebJar, et le `require.js` fait références à la ressources requise à la racine du WebJar.

#### npm

[npm](https://www.npmjs.org/) peut être utilisé comme WebJars en déclarant un fichier `package.json` à la racine de votre projet. Les ressources des paquetages npm sont extraites dans le même dossier `lib` que WebJars, ainsi au point de vue du code, il n'y a pas à se préoccuper si la sources d'une ressources est un WebJar ou un paquetage npm.

***

De votre point de vue, nous visons à offrir la parité des fonctionnalité avec les versions précédentes de Play. Alors que les choses ont beaucoup changé sous le capot, la transition pour vous devrait être facile. Le reste de cette section examine chaque partie de Play qui a été remplacé par SBT-web et décrit ce qui doit être changé.

#### CoffeeScript

Vous devez maintenant déclarer le plugin, typiquement dans votre fichier plugins.sbt:

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-coffeescript" % "1.0.0")
```

Les options de Coffeescript ont changé. Les nouvelles options sont:

* `sourceMaps` Lorsque elle est activée, elle génère les fichiers sourceMap. Défault à `true`.

  `CoffeeScriptKeys.sourceMaps := true`

* `bare` lorsque elle est activée, génère le JavaScript sans le [top-level function safety wrapper](http://coffeescript.org/#lexical-scope). Défault à `false`.

  `CoffeeScriptKeys.bare := false`

Pour plus plus d'information consulter [la documentation du plugin](https://github.com/sbt/sbt-coffeescript#sbt-coffeescript).

#### LESS

Vous devez maintenant déclarer le plugin, typiquement dans votre fichier plugins.sbt:

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-less" % "1.0.0")
```

Les points d'entrée sont maintenant déclarés en utilisant un filtre. Par exemple, pour déclarer que `foo.less` et `bar.less` sont requis:

```scala
includeFilter in (Assets, LessKeys.less) := "foo.less" | "bar.less"
```

Si vous utilisiez précédemment le fait que les fichiers commençant par un underscore étaient ignorés mais que tout les autres étaient compilés alors utilisez maintenant les filtres suivant:

```scala
includeFilter in (Assets, LessKeys.less) := "*.less"

excludeFilter in (Assets, LessKeys.less) := "_*.less"
```

Différemment de Play 2.2, the plugin sbt-less laisse n'importe quel utilisateur télécharger le fichier source original less et les sources maps générés. 
Unlike Play 2.2, the sbt-less plugin allows any user to download the original LESS source file and generated source maps. It allows easier debugging in modern web browsers. This feature is enabled even in production mode.

The plugin's options are:

Option              | Description
--------------------|------------
cleancss            | Compress output using clean-css.
cleancssOptions     | Pass an option to clean css, using CLI arguments from https://github.com/GoalSmashers/clean-css .
color               | Whether LESS output should be colorised
compress            | Compress output by removing some whitespaces.
ieCompat            | Do IE compatibility checks.
insecure            | Allow imports from insecure https hosts.
maxLineLen          | Maximum line length.
optimization        | Set the parser's optimization level.
relativeUrls        | Re-write relative urls to the base less file.
rootpath            | Set rootpath for url rewriting in relative imports and urls.
silent              | Suppress output of error messages.
sourceMap           | Outputs a v3 sourcemap.
sourceMapFileInline | Whether the source map should be embedded in the output file
sourceMapLessInline | Whether to embed the less code in the source map
sourceMapRootpath   | Adds this path onto the sourcemap filename and less file paths.
strictImports       | Whether imports should be strict.
strictMath          | Requires brackets. This option may default to true and be removed in future.
strictUnits         | Whether all unit should be strict, or if mixed units are allowed.
verbose             | Be verbose.

For more information please consult [the plugin's documentation](https://github.com/sbt/sbt-less#sbt-less).

#### Closure Compiler

The Closure Compiler has been replaced. Its two important functions of validating JavaScript and minifying it have been factored out into [JSHint](http://www.jshint.com/) and [UglifyJS 2](https://github.com/mishoo/UglifyJS2#uglifyjs-2) respectively.

To use JSHint you must declare it, typically in your plugins.sbt file:

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-jshint" % "1.0.0")
```

Options can be specified in accordance with the [JSHint website](http://www.jshint.com/docs) and they share the same set of defaults. To set an option you can provide a `.jshintrc` file within your project's base directory. If there is no such file then a `.jshintrc` file will be searched for in your home directory. This behaviour can be overridden by using a `JshintKeys.config` setting for the plugin.
`JshintKeys.config` is used to specify the location of a configuration file.

For more information please consult [the plugin's documentation](https://github.com/sbt/sbt-jshint#sbt-jshint).

UglifyJS 2 is presently provided via the RequireJS plugin (described next). The intent in future is to provide a standalone UglifyJS 2 plugin also for situations where RequireJS is not used.

#### RequireJS

The RequireJS Optimizer (rjs) has been entirely replaced with one that should be a great deal easier to use. The new rjs is part of sbt-web's asset pipeline functionality. Unlike its predecessor which was invoked on every build, the new one is invoked only when producing a distribution via Play's `stage` or `dist` tasks.

To use rjs you must declare it, typically in your plugins.sbt file:

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-rjs" % "1.0.1")
```

To add the plugin to the asset pipeline you can declare it as follows:

```scala
pipelineStages := Seq(rjs)
```

We also recommend that sbt-web's sbt-digest and sbt-gzip plugins are included in the pipeline. sbt-digest will provide Play's asset controller with the ability to fingerprint asset names for far-future caching. sbt-gzip produces a gzip of your assets that the asset controller will favor when requested. Your plugins.sbt file for this configuration will then look like:

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-digest" % "1.0.0")

addSbtPlugin("com.typesafe.sbt" % "sbt-gzip" % "1.0.0")
```

and your pipeline configuration now becomes:

```scala
pipelineStages := Seq(rjs, digest, gzip)
```

The order of stages is significant. You first want to optimize the files, produce digests of them and then produce gzip versions of all resultant assets.

The options for RequireJs optimization have changed entirely. The plugin's options are:

Option                  | Description
------------------------|------------
appBuildProfile         | The project build profile contents.
appDir                  | The top level directory that contains your app js files. In effect, this is the source folder that rjs reads from.
baseUrl                 | The dir relative to the assets or public folder where js files are housed. Will default to "js", "javascripts" or "." with the latter if the other two cannot be found.
buildProfile            | Build profile key -> value settings in addition to the defaults supplied by appBuildProfile. Any settings in here will also replace any defaults.
dir                     | By default, all modules are located relative to this path. In effect this is the target directory for rjs.
generateSourceMaps      | By default, source maps are generated.
mainConfig              | By default, 'main' is used as the module for configuration.
mainConfigFile          | The full path to the above.
mainModule              | By default, 'main' is used as the module.
modules                 | The json array of modules.
optimize                | The name of the optimizer, defaults to uglify2.
paths                   | RequireJS path mappings of module ids to a tuple of the build path and production path. By default all WebJar libraries are made available from a CDN and their mappings can be found here (unless the cdn is set to None).
preserveLicenseComments | Whether to preserve comments or not. Defaults to false given source maps (see http://requirejs.org/docs/errors.html#sourcemapcomments).
webJarCdns              | CDNs to be used for locating WebJars. By default "org.webjars" is mapped to "jsdelivr".
webJarModuleIds         | A sequence of webjar module ids to be used.

For more information please consult [the plugin's documentation](https://github.com/sbt/sbt-rjs#sbt-rjs).

### Default ivy local repository and cache

Due to Play now using Activator as a launcher, it now uses the default ivy cache and local repository.  This means anything previously published to your Play ivy cache that you depend on will need to be published to the local ivy repository in the `.ivy2` folder in your home directory.

## Results restructure

In Play 2.2, a number of result types were deprecated, and to facilitate migration to the new results structure, some new types introduced.  Play 2.3 finishes this restructuring.

### Scala results

The following deprecated types and helpers from Play 2.1 have been removed:

* `play.api.mvc.PlainResult`
* `play.api.mvc.ChunkedResult`
* `play.api.mvc.AsyncResult`
* `play.api.mvc.Async`

If you have code that is still using these, please see the [[Play 2.2 Migration Guide|Migration22]] to learn how to migrate to the new results structure.

As planned back in 2.2, 2.3 has renamed `play.api.mvc.SimpleResult` to `play.api.mvc.Result` (replacing the existing `Result` trait).  A type alias has been introduced to facilitate migration, so your Play 2.2 code should be source compatible with Play 2.3, however we will eventually remove this type alias so we have deprecated it, and recommend switching to `Result`.

### Java results

The following deprecated types and helpers from Play 2.1 have been removed:

* `play.mvc.Results.async`
* `play.mvc.Results.AsyncResult`

If you have code that is still using these, please see the [[Play 2.2 Migration Guide|Migration22]] to learn how to migrate to the new results structure.

As planned back in 2.2, 2.3 has renamed `play.mvc.SimpleResult` to `play.mvc.Result`.  This should be transparent to most Java code.  The most prominent places where this will impact is in the `Global.java` error callbacks, and in custom actions.

## Templates

The template engine is now a separate project, [Twirl](https://github.com/playframework/twirl).

### Content types

The template content types have moved to the twirl package. If the `play.mvc.Content` type is referenced then it needs to be updated to `play.twirl.api.Content`. For example, the following code in a Play Java project:

```java
import play.mvc.Content;

Content html = views.html.index.render("42");
```

will produce the error:

```
[error] ...: incompatible types
[error] found   : play.twirl.api.Html
[error] required: play.mvc.Content
```

and requires `play.twirl.api.Content` to be imported instead.

### sbt settings

The sbt settings for templates are now provided by the sbt-twirl plugin.

Adding additional imports to templates was previously:

```scala
templatesImport += "com.abc.backend._"
```

and is now:

```scala
TwirlKeys.templateImports += "org.abc.backend._"
```

Specifying custom template formats was previously:

```scala
templatesTypes += ("html" -> "my.HtmlFormat.instance")
```

and is now:

```scala
TwirlKeys.templateFormats += ("html" -> "my.HtmlFormat.instance")
```

For sbt builds that use the full scala syntax, `TwirlKeys` can be imported with:

```scala
import play.twirl.sbt.Import._
```

## Play WS

The WS client is now an optional library. If you are using WS in your project then you will need to add the library dependency. For Java projects you will also need to update to a new package.

#### Java projects

Add library dependency to `build.sbt`:

```scala
libraryDependencies += javaWs
```

Update to the new library package in source files:

```java
import play.libs.ws.*;
```

#### Scala projects

Add library dependency to `build.sbt`:

```scala
libraryDependencies += ws
```

In addition, usage of the WS client now requires a Play application in scope. Typically this is achieved by adding:

```scala
import play.api.Play.current
```

The WS API has changed slightly, and `WS.client` now returns an instance of `WSClient` rather than the underlying `AsyncHttpClient` object.  You can get to the `AsyncHttpClient` by calling `WS.client.underlying`.

## Anorm

There are various changes included for Anorm in this new release.

For improved type safety, type of query parameter must be visible, so that it [can be properly converted](https://github.com/playframework/playframework/blob/master/documentation/manual/scalaGuide/main/sql/ScalaAnorm.md#edge-cases). Now using `Any` as parameter value, explicitly or due to erasure, leads to compilation error `No implicit view available from Any => anorm.ParameterValue`.

```scala
// Wrong
val p: Any = "strAsAny"
SQL("SELECT * FROM test WHERE id={id}").
  on('id -> p) // Erroneous - No conversion Any => ParameterValue

// Right
val p = "strAsString"
SQL("SELECT * FROM test WHERE id={id}").on('id -> p)

// Wrong
val ps = Seq("a", "b", 3) // inferred as Seq[Any]
SQL("SELECT * FROM test WHERE (a={a} AND b={b}) OR c={c}").
  on('a -> ps(0), // ps(0) - No conversion Any => ParameterValue
    'b -> ps(1), 
    'c -> ps(2))

// Right
val ps = Seq[anorm.ParameterValue]("a", "b", 3) // Seq[ParameterValue]
SQL("SELECT * FROM test WHERE (a={a} AND b={b}) OR c={c}").
  on('a -> ps(0), 'b -> ps(1), 'c -> ps(2))
```

If passing value without safe conversion is required, `anorm.Object(anyVal)` can be used to set an opaque parameter.

Moreover, erasure issues about parameter value is fixed: type is no longer `ParameterValue[_]` but simply `ParameterValue`.

Types for parameter names are also unified (when using `.on(...)`). Only `String` and `Symbol` are now supported as name.

Type *`Pk[A]`* has been deprecated. You can still use it as column mapping, but need to explicitly pass either `Id[A]` or `NotAssigned` as query parameter (as consequence of type safety improvements):

```scala
// Column mapping, deprecated but Ok
val pk: Pk[Long] = SQL("SELECT id FROM test WHERE name={n}").
  on('n -> "mine").as(get[Pk[Long]].single)

// Wrong parameter
val pkParam: Pk[Long] = Id(1l)
val name1 = "Name #1"
SQL"INSERT INTO test(id, name) VALUES($pkParam, $name1)".execute()
// ... pkParam is passed as Pk in query parameter, 
// which is now wrong as a parameter type (won't compile)

// Right parameter Id
val idParam: Id[Long] = Id(2l) // same as pkParam but keep explicit Id type
val name2 = "Name #2"
SQL"INSERT INTO test(id, name) VALUES($idParam, $name2)".execute()

// Right parameter NotAssigned
val name2 = "Name #3"
SQL"INSERT INTO test(id, name) VALUES($NotAssigned, $name2)".execute()
```

As deprecated `Pk[A]` is similar to `Option[A]`, which is supported by Anorm in column mapping and as query parameter, it's preferred to replace `Id[A]` by `Some[A]` and `NotAssigned` by `None`:

```scala
// Column mapping, deprecated but Ok
val pk: Option[Long] = SQL("SELECT id FROM test WHERE name={n}").
  on('n -> "mine").as(get[Option[Long]].single)

// Assigned primary key as parameter
val idParam: Option[Long] = Some(2l)
val name1 = "Id"
SQL"INSERT INTO test(id, name) VALUES($idParam, $name1)".execute()

// Right parameter NotAssigned
val name2 = "NotAssigned"
SQL"INSERT INTO test(id, name) VALUES($None, $name2)".execute()
```

## Twitter Bootstrap

The in-built Twitter Bootstrap field constructor has been deprecated, and will be removed in a future version of Play.

There are a few reasons for this, one is that we have found that Bootstrap changes too drastically between versions and too frequently, such that any in-built support provided by Play quickly becomes stale and incompatible with the current Bootstrap version.

Another reason is that the current Bootstrap requirements for CSS classes can't be implemented with Play's field constructor alone, a custom input template is also required.

Our view going forward is that if this is a feature that is valuable to the community, a third party module can be created which provides a separate set of Bootstrap form helper templates, specific to given Bootstrap versions, allowing a much better user experience than can currently be provided.

## Session timeouts

The session timeout configuration item, `session.maxAge`, used to be an integer, defined to be in seconds.  Now it's a duration, so can be specified with values like `1h` or `30m`.  Unfortunately, the default unit if specified with no time unit is milliseconds, which means a config value of `3600` was previously treated as one hour, but is now treated as 3.6 seconds.  You will need to update your configuration to add a time unit.

## Java JUnit superclasses

The Java `WithApplication`, `WithServer` and `WithBrowser` JUnit test superclasses have been modified to define an `@Before` annotated method.  This means, previously where you had to explicitly start a fake application by defining:

```java
@Before
public void setUp() {
    start();
}
```

Now you don't need to. If you need to provide a custom fake application, you can do so by overriding the `provideFakeApplication` method:

```java
@Override
protected FakeApplication provideFakeApplication() {
    return Helpers.fakeApplication(Helpers.inMemoryDatabase());
}
```

## Session and Flash implicits

The Scala Controller provides implicit `Session`, `Flash` and `Lang` parameters, that take an implicit `RequestHeader`.  These exist for convenience, so a template for example can take an implicit argument and they will be automatically provided in the controller.  The name of these was changed to avoid conflicts where these parameter names might be shadowed by application local variables with the same name. `session` became `request2Session`, `flash` became `flash2Session`, `lang` became `lang2Session`.  Any code that invoked these explicitly consequently will break.

It is not recommended that you invoke these implicit methods explicitly, the `session`, `flash` and `lang` parameters are all available on the `RequestHeader`, and using the `RequestHeader` properties makes it much clearer where they come from when reading the code.  It is recommended that if you have code that uses the old methods, that you modify it to access the corresponding properties on `RequestHeader` directly.

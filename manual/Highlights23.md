<!--- Copyright (C) 2009-2014 Typesafe Inc. <http://www.typesafe.com> -->
# Quoi de neuf dans Play 2.3

Cette page met en évidence les nouvelles fonctionnalités de Play 2.3. Si vous voulez en apprendre davantage sur les changements que vous devez faire pour migrer vers PLay 2.3, consultez le [[Guide de migation Play 2.3|Migration23]].

## Activator

La première chose qui vous remarquerez avec Play 2.3, c'est que la commande `play` a changé en `activator`. Play a été modifié afin d'utiliser [Activator](https://typesafe.com/activator) ainsi nous pouvons:

* Étendre la gamme des templates que nous proposons pour commencer avec des projets Play. Activator supporte une bien plus [riche collection](https://typesafe.com/activator/templates) de template de project. Ces Templates peuvent également inclure des tutoriaux et d'autres resources pour démarrer. La communauté Play peut [contributeer aux templates](https://typesafe.com/activator/template/contribute) également.

* Fournir une jolie interface web pour commencer avec Play, en particulier pour les nouveaux qui ne sont pas familier avec la ligne de commande. Les utilisateurs peuvent écrire du code et jouer les tests à travers l'interface graphique web. Pour les utilisateurs aguerris, la ligne de commande est toujours disponible.
* Apporter la haute productivité du développement Play aux autres projets. Activator n'est pas seulement pour Play. Les autres projets peuvent également utiliser Activator.

A l'avenir Activator contiendra encore plus de fonctionnalités, et ces dernières bénéficieront automatiquement aux projets Play et autres qui utilisent Activator. [Activator est open source] (https://github.com/typesafehub/activator), de sorte que la communauté peut contribuer à son évolution.

### Commandes Activator

Toutes les fonctionnalités qui étaient disponibles avec la commande `play` sont toujours disponibles avec la commande `activator`.

* `activator new` pour créer un nouveau projet. Voir [[Créer une nouvelle application|NewApplication]].
* `activator` pour lancer la console. Voir [[Utiliser la console Play|PlayConsole]].
* `activator ui` est une nouvelle commande qui lance une interface web.

> La nouvelle command `activator`, comme l'ancienne commande `play` sont toute deux des wrappers autour de [sbt](http://www.scala-sbt.org/). Si vous préférez, vous pouvez utiliser la command `sbt` directement. Toutefois, si vous utilisez sbt seul vous passerez à coté de plusieurs fonctionnalités Activator, tels que les templates (`activator new`) ou l'interface web (`activator ui`). sbt and Activator supportent toutes les commandes habituelles comme `test` et `run`.

### La distribution Activator
Play est diffusé comme une distribution Activator qui contient toutes les dépendences pour Play. Vous pouvez télécharger cette distribution à partir de   la page [Play download](http://www.playframework.com/download).

Si vous préférez, vous pouvez également télécharger une version minimale (1MB) d'Activator à partir de [Activator site](https://typesafe.com/activator). Cherchez la "mini" distribution sur la page de téléchargement. La version minimale d'Activatot téléchargera les dépendences qui lorsqu'elle seront nécessaires.

Comme Activator n'est qu'un wrapper autour de sbt, vous pouvez également télécharger et utiliser [sbt](http://www.scala-sbt.org/) directement, si vous préférez.

## Amélioration de l'environnement de Build

### sbt-web

La plus grosses fonctionnalité de Play 2.3 est l'introduction de [sbt-web](https://github.com/sbt/sbt-web#sbt-web). En substance, sbt-web, déporte la prise en charge du HTML, des CSS et du JavaScript du cœeur de Play vers des plugins sbt. Il y a deux avantages majeur à cela:

* Play est moins dogmatique à propos du HTML, CSS et du Javascript; et
* sbt-web peut avoir sa propre communauté et se développer parallèlement à Play.

### Auto Plugins

Play utilise désormais sbt 0.13.5. Cette version apporte une nouvelle fonctionnalité appelée "auto plugins" qui, en substance assure une nette réduction de la quantité de code lié au settings dans les fichiers de build.

### Asset Pipeline et Fingerprinting

sbt-web apporte la notion de pipeline de resources hautement configurable à Play, par exemple:

```scala
pipelineStages := Seq(rjs, digest, gzip)
```
Ce qui précède ordonne l'optimiseur RequireJs (sbt-rjs), le digesteur (sbt-digest), puis compression (sbt-gzip). Contrairement à de nombreuses tâches de SBT, ces tâches vont s'exécuter ces tâches déclarées dans l'ordre, l'une après l'autre.

Une des nouvelles fonctionnalité de Play 2.3 est le support des empreintes sur les resources, conformément au principe des [Rails asset fingerprinting](http://guides.rubyonrails.org/asset_pipeline.html#what-is-fingerprinting-and-why-should-i-care-questionmark). La conséquence de cette gestion des empreintes est que l'on peut utiliser les fonctions de cache des navigateurs. Avec comme résultat des gains de performance lors du téléchargement de ces resources car elle seront fortement cachées.

### Le cache par défault ivy et le repository local

Play utilise désormais le cache et le repository de ivy, dans le répertoire `.ivy2` du répertoire de l'utilisateur.

Cela signifie que Play s'intègre mieux avec les autres builds sbt, et évite que les artifacts soit cachés plusieurs fois, de même il est plus simple de partager des artifacts localement.

## Amélioration Java

### Java 8
Play 2.3 a été testé avec Java 8. Votre projet fonctionnera bien avec Java 8, rien de particulier autre que de s'assurer que l'environnement Java soit configuré. Il existe un nouvel exemple Activator pour Java 8:

http://typesafe.com/activator/template/reactive-stocks-java8

Notre documentation a été amélioré avec des exemples de manière générale et, lorsque application, des exemples Java 8. Voir les [[exemples de programmation asynchrone avec Java 8|JavaAsync]].

Pour une vue d'ensemble complète sur devenier "Reactive" avec Java 8 et Play consultez ce blog: http://typesafe.com/blog/go-reactive-with-java-8

### Performance Java 

Nous avons travaillé sur la performance Java. Par rapport à Play 2.2, les vitesses de traitements des actions Java simples a augmenté de 40-90%. Voici les principales optimisations: 

* Réduire les commutateurs de thread pour les actions Java et des parsers du corps des requêtes. 
* La mise en cache de plus amples informations des routes et utilisation d'un cache par route plutôt qu'un cache commun. 
* Réduction de coût lié à l'analyse du corps de requête GET. 
* L'utilisation d'un enummerateur unicast pour les réponse fragmentées. 

Certains de ces changements ont également une amélioration des performances Scala, mais Java eu les plus grands gains de performance et a été le principal objectif de notre travail. 

Merci à [YourKit] (http://yourkit.com) de fournir des liences à l'équipe de Play pour rendre ce travail possible.

## Scala 2.11

Play 2.3 est la première version de Play construite pour plusieurs versions de Scala, 2.10 et 2.11.

Vous pouvez choisir la version de Scala que vous souhaitez utiliser en positionnant le paramètre `scalaVersion` de votre fichier `build.sbt` ou `Build.scala`.

Pout Scala 2.11:

```scala
scalaVersion := "2.11.1"
```

Pour Scala 2.10:

```scala
scalaVersion := "2.10.4"
```

## Play WS

###  Bibliothèque séparée

La bibliothèque client WS a été isolée et peut être utilisée en dehors de Play. Vous pouvez maintenant avoir plusieurs objet `WSClient`, plutôt que d'utiliser le singleton `WS`.

[[Java|JavaWS]]

```java
WSClient client = new NingWSClient(config);
Promise<WSResponse> response = client.url("http://example.com").get();
```

[[Scala|ScalaWS]]

```scala
val client: WSClient = new NingWSClient(config)
val response = client.url("http://example.com").get()
```
Chaque client WS peut être configuré avec ses propres options. Cela permet de supporter différentes options de timeout, redirect ou de sécurité.

L'objet `AsyncHttpClient` sous jacent peut maintenant être accédé, ce qui signifie que les form multi-part et les uploads en streaming sont supportés.

###  Sécurité WS

Les clients WS  ont des [[paramètres|WsSSL]] une configuration complète de SSL/TLS. La configuration par défaut du client WS est maintenant plus sécurisé.

## Actor WebSockets

Une méthode pour gérer les interactions avec les websockets avec des acteurs a été ajouté en Java et en Scala:

[[Java|JavaWebSockets]]

```java
public static WebSocket<String> socket() {
    return WebSocket.withActor(MyWebSocketActor::props);
}
```

[[Scala|ScalaWebSockets]]

```scala
def webSocket = WebSocket.acceptWithActor[JsValue, JsValue] { req => out =>
  MyWebSocketActor.props(out)
```

## Finalisation de la restructuration des Results
Dans Play 2.2, de nouveaux type de Result ont été introduit et les anciens dépréciés. Play 2.3 finalise cette restructuration. Ce référer à *Results restructure* dans le [[Guide de Migration|Migration23]] pour plus d'information.

## Anorm
Il y a divers fix inclus dans Anorm pour Play 2.3 (type safety, option parsing, gestion des erreurs, ...) et des fonctionnalités intéressantes.

- L'interpolation des chaînes de charactère est possible lors de l'écriture des requêtes SQL, plus simple, moins verbeux et plus performant ( x7 plus rapide qu'en passant les paramètres). Exemple: `SQL"SELECT * FROM table WHERE id = $id"`
- Multi-value (sequence/list) peuvent être passées en paramètre. Exemple: `SQL"""SELECT * FROM Test WHERE cat IN (${Seq("a", "b", "c")})"""`
- Il est possible maintenant d'analyser les colonnes par position. Exemple:  `val parser = long(1) ~ str(2) map { case l ~ s => ??? }`
- Les résultats des requêtes incluent également de contexte d'exécution (avec les warnings SQL) et plus seulement les données.
- Plus de types sont supportés comme paramètre et comme colonne: `java.util.UUID`, types numerique  (Java/Scala big decimal et integer, plus de conversion entre numeriques), type temporaux (`java.sql.Timestamp`), types charactères.

## SSLEngine personnalisé pour HTTPS

Le serveur Play peut maintenant [[utiliser un `SSLEngine` personnalisé|ConfiguringHttps]]. Cela est fort utilie lorsque cette personnalisation en requise, tel que lors d'une authentification client.
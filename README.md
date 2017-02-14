play-elastic4s [![Build Status](https://travis-ci.org/evojam/play-elastic4s.svg?branch=master)](https://travis-ci.org/evojam/play-elastic4s)
===========================

We've been using [elastic4s](https://github.com/sksamuel/elastic4s) with [Play framework](https://www.playframework.com/) for a while.
Without convenient way of configuration and injecting ElasticClient instance there was a lot boilerplate work to do.
This module enhances elastic4s with two main features:

1. Loading driver configuration from `application.conf`.
1. Automatic JSON conversions for Elasticsearch based on [Play JSON formatters](https://www.playframework.com/documentation/2.4.x/ScalaJson).

As a bonus, the Elasticsearch driver will automatically disconnect on Play application shutdown.


Quick Start
-----------

There are no stable releases yet. We're testing the current version
with Elasticsearch 2.4.0 and it will not work with Elasticsearch other
than 2.2.X. Moreover, we depend on Play 2.5.9 which in turn requires Java 8.

### 1. Installation

Add the dependency to your `build.sbt`:

	libraryDependencies += "com.evojam" %% "play-elastic4s" % "0.3.2"

	resolvers += Resolver.sonatypeRepo("snapshots")


#### Play 2.5.x compatibility note

Due to the difference in underlying Netty version (4.1.x in ES client and 4.0.x in Play)
that are binary incompatible you have to use Akka transport. Please ensure you have
appropriate plugins' setup in your `build.sbt`:

```
enablePlugins(PlayScala, PlayAkkaHttpServer)

disablePlugins(PlayNettyServer)
```

### 2. Setup

Extend your application.conf to load the module and get all goodies injectable:

```hocon
elastic4s {
    clusters {
        myCluster {
           type: "transport"               // <-- either "transport" or "node"
           cluster.name: "mycluster"       // <-- set your cluster name here
           uri: elasticsearch://host:port  // <-- if using transport client, pass uri to the cluster
        }
    }
    indexAndTypes {                        // <-- this section is not required, but
        book {                             //     this index/type pair will be injectable later
            index: "library"
            type: "book"
        }
    }
}

play.modules.enabled += "com.evojam.play.elastic4s.Elastic4sModule"
```

You may define any number of clusters. Transport clients require an `uri` field with elasticsearch
uri. This field determines whether an embedded client or transport client will be created. If the `uri` field
is present Transport client will be used, embedded client otherwise. Also, each cluster needs to
have `cluster.name` set. Any other fields will be passed to  Elasticsearch Java driver settings
builder. Please refer to [Java Client docs], paying attention to [Node Client docs] and [Transport Client docs].

  [Java Client docs]: https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/client.html
  [Node Client docs]: https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/node-client.html
  [Transport Client docs]: https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/transport-client.html

If you have `elasticsearch.yml` file in your classpath, it will also be loaded.
In case of conflicts, `application.conf` wins. It is discouraged to have
configuration in two places, though, so we'd rather you dropped the
`elasticsearch.yml`.

**Warning: Node clients will hurt you unless properly configured.** By default,
an embedded client will join the ES cluster and will try and store some data
(meaning that some shards will be allocated to it). Then on application shutdown
these shards will be lost. Moreover, the default settings allow the embedded
client to become the master node in the cluster. This is probably not what you
want.

As a final note on connection setup, please bear in mind that Elasticsearch
nodes have autodiscovery system and will automatically form clusters. Therefore
running multiple node clients in a single application may lead to strange
behavior.

You may use the `indexAndTypes` node to configure names of your indexes in ES.
As a good practice, you should probably put aliases here, not hard names
(refer to [ES docs on aliases](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html)). Every key in that node will be transformed to
an injectable [`IndexAndType`](https://github.com/sksamuel/elastic4s/blob/master/elastic4s-core/src/main/scala/com/sksamuel/elastic4s/IndexAndTypes.scala) instance.

### 3. Enjoy injectable configuration

Instead of instantiating `ElasticClient` manually, have the configuration
and our factory [injected](https://www.playframework.com/documentation/2.4.x/ScalaDependencyInjection):

```scala

class BookDao @Inject()(
	cs: ClusterSetup,
	elasticFactory: PlayElasticFactory,
	@Named("book") indexAndType: IndexAndType) extends
		ElasticDsl {

	private[this] lazy val client = elasticFactory(cs)

	def searchByAnything(q: String): Future[RichSearchResponse] = client execute {
		search in indexAndType query q
	}
}
```

Apart from the injection, nothing changes. The `client` generated by the factory is a regular `ElasticClient` from `elastic4s` library,
so all the original API stays. Calling the factory twice with the same `ClusterSetup`
will reuse the same client, so feel free to use the factory whenever you need
a client instance. Moreover, the created client instances are hooked to Play
application lifecycle and will gracefully disconnect on application shutdown.

### 4. Enjoy seamless JSON conversions

The original elastic4s API already provides automatic JSON conversions, but it requires you to provide
specific typeclasses ([Indexable](https://github.com/sksamuel/elastic4s#indexing-from-classes)
and [HitAs](https://github.com/sksamuel/elastic4s#search-conversion)). Play-elastic4s will derive them automatically
based on your Play JSON formatters - just mix in the `PlayElasticJsonSupport` trait:

```scala
case class Book(title: String, author: String, publishDate: DateTime)
object Book {
	implicit val format: Format[Book] = Json.format[Book]
}

class BookDao @Inject()(
	cs: ClusterSetup,
	elasticFactory: PlayElasticFactory,
	@Named("book") indexAndType: IndexAndType) extends
		ElasticDsl with
		PlayElasticJsonSupport {

	private[this] lazy val client = elasticFactory(cs)

	def searchByAnything(q: String): Future[Array[Book]] = client execute {
		search in indexAndType query q
	} map (_.as[Book])   // here the conversion happens. Will throw if documents are malformed.

	def add(bookId: String, book: Book) = client execute {
		index into indexAndType source book id bookId
	}

	// as an extra, you also get an extension method when getting by ID.
	// It returns None if there is no document with the given ID
	// and throws an exception if the document cannot be parsed:
	def getById(bookId: String): Future[Option[Book]] = client.execute {
		get id bookId from indexAndType
	} map (_.as[Book])
}
```

Note that this does not provide interoperability with the magic ES fields
(`_type`, `_id` and similar). Even worse, if your custom type produces
an `_id` field during serialization to JSON, the driver will produce
a request with `_id` field in the source document. Such requests are invalid
and will be rejected.

Advanced usage
--------------------------

### Detailed ES connection configuration
Simply add more options to "elastic4s.clusters.<your-cluster>" node in the config file.
They will be passed to the ES java driver.

### Using mutliple ES clusters
Specify them all in the config file:

```hocon
elastic4s {
		clusters {
				firstCluster {
					 uri: elasticsearch://main.es.example.com:9300
					 cluster.name: "first-es-cluster"
				}
				loggingCluster {
					uri: elasticsearch://log.es.example.com:9300
					cluster.name: "log-es-cluster"
				}

		}
		indexAndTypes {
				book {
						index: "library"
						type: "book"
				}
		}
}

play.modules.enabled += "com.evojam.play.elastic4s.Elastic4sModule"
```

The unnamed binding for `ClusterSetup` is available only if there is exactly one cluster configured.
With multiple clusters, use `@Named` annotation:

```scala
class BookDao @Inject()(
	@Named("firstCluster") firstCs: ClusterSetup,
	@Named("loggingCluster") loggingCs: ClusterSetup,
	elasticFactory: PlayElasticFactory,
	@Named("book") indexAndType: IndexAndType) extends
		ElasticDsl {

	private[this] lazy val client = elasticFactory(firstCs)
	private[this] lazy val logClient = elasticFactory(loggingCs)
	
	...
}
```


# For developers

Please use
[git-flow](http://jeffkreeftmeijer.com/2010/why-arent-you-using-git-flow/), with
one exception regarding finishing features: instead of doing `git flow feature
finish X`, submit a pull request and merge it in GitHub.

**Releasing:** all commits to master branch automatically trigger publishing to
[Sonatype OSS] from Travis CI. For snapshot versions, this is enough. For
release versions, one needs to login to the repository manager and close the
staging repository, as described at [Sonatype releasing guide]. Also make sure
to tag released versions, preferably using GitHub release feature. Finally,
after having released a `X.Y.Z` version, make another commit that changes
version number in `build.sbt` to `X.Y.(Z+1)-SNAPSHOT` in order to avoid a
duplicate release with the same number.

  [Sonatype OSS]: https://oss.sonatype.org
  [Sonatype releasing guide]: http://central.sonatype.org/pages/releasing-the-deployment.html

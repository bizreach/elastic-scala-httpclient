elastic-scala-httpclient   [![Build Status](https://secure.travis-ci.org/bizreach/elastic-scala-httpclient.png?branch=master)](http://travis-ci.org/bizreach/elastic-scala-httpclient)
===============

Elasticsearch HTTP client for Scala with code generator.

|Client version |Elasticsearch |Scala version |
|---------------|--------------|--------------|
|4.0.0 -        |6.7.x -       |2.11 / 2.12   |
|3.0.0 - 3.2.4  |5.2.x -       |2.11 / 2.12   |
|2.0.4 - 2.0.6  |2.3.5         |2.12          |
|2.0.0 - 2.0.3  |2.3.5         |2.11          |
|1.0.6          |1.7.3         |2.11          |
|1.0.0 - 1.0.5  |1.1.0         |2.11          |

## How to use

Add a following dependency into your `build.sbt` at first.

```scala
libraryDependencies += "jp.co.bizreach" %% "elastic-scala-httpclient" % "4.0.0"
```

You can access Elasticsearch via HTTP Rest API as following:

```scala
case class Tweet(name: String, message: String)
case class TweetMessage(message: String)

import jp.co.bizreach.elasticsearch4s._

ESClient.using("http://localhost:9200"){ client =>
  val config = ESConfig("twitter")

  // Insert
  client.insert(config, Tweet("takezoe", "Hello World!!"))
  client.insertJson(config, """{name: "takezoe", message: "Hello World!!"}""")

  // Update
  client.update(config, "1", Tweet("takezoe", "Hello Scala!!"))
  client.updateJson(config, "1", """{name: "takezoe", message: "Hello World!!"}""")

  // Update partially
  client.updatePartially(config, "1", TweetMessage("Hello Japan!!"))
  client.updatePartiallyJson(config, "1", """{name: "takechan" }""")

  // Delete
  client.delete(config, "1")

  // Find a document
  val tweet: Option[(String, Tweet)] = client.find[Tweet](config){ builder =>
    builder.query(termQuery("_id", "1"))
  }

  // Search documents
  val list: List[ESSearchResult] = client.list[Tweet](config){ builder =>
    builder.query(termQuery("name", "takezoe"))
  }
}
```

If you have to recycle `ESClient` instance, you can manage lyfecycle of `ESClient` manually.

```scala
// Call this method once before using ESClient
ESClient.init()

val client = ESClient("http://localhost:9200")
val config = ESConfig("twitter")

client.insert(config, Tweet("takezoe", "Hello World!!"))

// Call this method before shutting down application
ESClient.shutdown()
```

[AsyncESClient](https://github.com/bizreach/elastic-scala-httpclient/blob/master/elastic-scala-httpclient/src/main/scala/jp/co/bizreach/elasticsearch4s/AsyncESClient.scala) that is an asynchrnous version of ESClient is also available. All methods of `AsyncESClient` returns `Future`.

## Addtional requirements

Some methods of `ESClient` and `AsyncESClient` need Elasticsearch plug-ins. You have to install following plug-ins into Elasticsearch to use these methods:

|Method               |Elasticsearch plug-in                                                                                          |
|--------------------|----------------------------------------------------------------------------------------------------------------|
|searchByTemplate    |[elasticsearch-sstmpl plug-in](https://github.com/codelibs/elasticsearch-sstmpl)                                |
|listByTemplate      |[elasticsearch-sstmpl plug-in](https://github.com/codelibs/elasticsearch-sstmpl)                                |
|countByTemplate     |[elasticsearch-sstmpl plug-in](https://github.com/codelibs/elasticsearch-sstmpl)                                |
|countByTemplateAsInt|[elasticsearch-sstmpl plug-in](https://github.com/codelibs/elasticsearch-sstmpl)                                |

Furthermore you have to indicate to enable these methods as follows (In default, these methods throws `IllegalStateException`):

```scala
// Enable xxxxByTemplate methods
ESClient.using("http://localhost:9200",
  scriptTemplateIsAvailable = true){ client =>
  ...
}
```

## Retry

`ESClient` and `AsyncESClient` support automatic retry. Give `RetryConfig` to enable it.

```scala
import jp.co.bizreach.elasticsearch4s._
import jp.co.bizreach.elasticsearch4s.retry._

ESClient.using("http://localhost:9200",
  retryConfig = RetryConfig(
    maxAttempts = 2,
    duration = 1.second,
    backOff = FixedBackOff
  )){ client =>
  ...
}
```

or

```scala
val client = ESClient("http://localhost:9200",
  retryConfig = RetryConfig(
    maxAttempts = 2,
    duration = 1.second,
    backOff = FixedBackOff
  )  
)
```

`RetryConfig` has following parameters:

- `maxAttempts`: Max attenpts to retry. 0 means no retry.
- `duration`: Duration until next retry. Actual duration is calculated by this parameter and the back off strategy.
- `backOff`: `FixedBackOff`, `LinerBackOff` and `ExponentialBackOff` are available.

## Code Generator

elastic-scala-codegen can generate source code from Elasticsearch schema json file.

At first, add following setting into `project/plugins.sbt`:

```scala
addSbtPlugin("jp.co.bizreach" % "elastic-scala-codegen" % "3.1.0")
```

Then put Elasticsearch schema json file as `PROJECT_ROOT/schema.json` and execute `sbt es-codegen`. Source code will be generated into `src/main/scala/models`.

You can configure generation settings in `PROJECT_ROOT/es-codegen.json`. Here is a configuration example:

```json
{
  "outputDir": "sec/main/scala",
  "dateType": "java8",
  "mappings": [
    {
	  "path": "schemas/book.json",
	  "packageName": "jp.co.bizreach",
	  "className": "Book",
	  "arrayProperties": [
	    "author"
	  ],
	  "ignoreProperties": [
	    "internalCode"
	  ]
	}
  ],
  "typeMappings": {
    "minhash": "String"
  }
}
```

See [ESCodegenConfig.scala](https://github.com/bizreach/elastic-scala-httpclient/blob/master/elastic-scala-codegen/src/main/scala/jp/co/bizreach/elasticsearch4s/generator/ESCodegenConfig.scala) to know configuration details.

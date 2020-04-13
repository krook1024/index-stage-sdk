# Index Stage SDK
Java SDK framework for developing custom Fusion Index Stage plugins.

# Basic concepts

## Plugin
Plugin is a zip file that contains one or more stages implementation. It consists of one or more jar files 
(the plugin jar with Stage definition and additional dependencies jar files) and a manifest file that contains metadata
used by Fusion to run the plugin.

In [provided examples](../examples/README.md) plugin zip file is generated by `assemblePlugin` gradle task. 
Alternatively it can be assembled by any other build tool as long as the file and directory structure is correct.

Example of custom META-INF/MANIFEST.MF file:
```
Manifest-Version: 1.0
Plugin-Id: sample-plugin
Plugin-SDK-Version: 1.0.0
Plugin-Base-Package: com.lucidworks.sample
Plugin-Version: 0.0.1

```

## [Index Stage](src/main/java/com/lucidworks/indexing/api/IndexStage.java)
Index Stage class contains implementation of custom processing logic that is applied to each document passing through
the index stage. 

Plugin stage class must implement [com.lucidworks.indexing.api.IndexStage](src/main/java/com/lucidworks/indexing/api/IndexStage.java) 
interface and be annotated with [com.lucidworks.indexing.api.Stage](src/main/java/com/lucidworks/indexing/api/Stage.java) 
annotation. For additional convenience stage implementation can extend 
[com.lucidworks.indexing.api.IndexStageBase](src/main/java/com/lucidworks/indexing/api/IndexStageBase.java) class, which
already contains initialization logic and some helpful methods.

Example of a custom index stage implementation:
```java
@Stage(type = "simple", configClass = SimpleStageConfig.class)
public class SimpleStage extends IndexStageBase<SimpleStageConfig> {

  private static final Logger logger = LoggerFactory.getLogger(SimpleStage.class);

  @Override
  public Document process(Document document, Context context) {
    String text = StringUtils.trim(config.text());

    document.field(config.newField(), String.class).set(text).type(Types.STRING);

    return document;
  }
}
```

### Lifecycle
First Fusion will create `IndexStage` instance and initialize it. Once the stage created and initialized, Fusion will 
start calling `process` method for each document being processed through the index pipeline. One stage instance can 
be used by Fusion for processing multiple documents and `process` method may be called from multiple concurrently 
running threads, additionally Fusion can initialize and maintain multiple stage instances with the same configuration 
in separate indexing service nodes. Therefore it is important to make sure that plugin stage implementation is 
thread-safe and processing logic is stateless.

Stage may throw an exception while processing a document. This will **not** cause any other side effect except the fact, 
that current document will not be processed anymore. Whole indexing pipeline will be still in use and fusion will call  
`process` method for other documents. Information about thrown exception will be visible in logs.

#### Initialization
After creation each stage object will be initialized using the `init(T config, Fusion fusion)` method. This allows 
to create some internal stage structures and validate configuration. 

Note that initialization occurs immediately after stage configuration is saved in Fusion. After this the stage instance
can be maintained and re-used by Fusion for extensive periods of time even if no documents are being processed through
the stage. It is strongly advised to be mindful of this fact when making decisions on resource allocation.

#### Document processing
After the initialization is finished, Fusion will start sending documents for processing to the stage by invoking 
`process(Document document, Context context, Consumer<Document> output)` for every incoming document. Since in the 
majority of cases index stages process and emit single document it is often easier to implement the alternative 
`process(Document document, Context context)` method.

* If your stage is transforming single document (or filtering), you need to implement 
  `process(Document document, Context context)` method only.
* If your stage is intending to emit multiple documents for a single input document, you need to implement 
  `process(Document document, Context context, Consumer<Document> output)` method and send each output document
  by calling `output.accept(doc)`.

Note that multiple threads can call `process` method concurrently, therefore its implementation must be thread-safe.

#### Logging
Index Stage SDK is using slf4j logging API. Following example demonstrates logging inside a plugin stage.
```java
public class MyStage extends IndexStageBase<MyStageConfig> {
  
  private static final Logger logger = LoggerFactory.getLogger(MyStage.class);

  @Override
  public Document process(Document document, Context context) {
    // ......
    logger.info("Processing document with id '{}'", document.getId());
    // ......
  }
}
```

#### Metrics
Fusion is using [Prometheus and Grafana](https://doc.lucidworks.com/fusion-server/latest/concepts/system/prometheus-grafana/index.html) 
for enhanced metrics collection and querying. Index Stage plugins can add their own custom metrics to the list of 
default metrics already generated by Fusion indexing service.

To be able to publish custom metrics prometheus client library must be added to the plugin dependencies
```
provided 'io.prometheus:simpleclient_dropwizard:0.7.0'
```
After that, prometheus client can be used to record custom metrics. More information on prometheus Java client API 
can be found at https://github.com/prometheus/client_java#instrumenting

Following example code demonstrates capturing external request time metrics using prometheus client API.
```java
public class SampleStage {

  private static final Histogram EXTERNAL_REQUEST_TIME = Histogram.build()
      .help("Time to execute external query request.")
      .name("external_query_request_time")
      .labelNames("request_url")
      .register();
  
    @Override
    public Document process(Document document, Context context) {
      // ......
      Histogram.Timer externalQueryTimer = EXTERNAL_REQUEST_TIME
          .labels(requestUrl)
          .startTimer();
      try {
        // perform external request...
      } finally {
        externalQueryTimer.observeDuration();
      }
      // ......
    }
}
```

## [Index Stage Config](src/main/java/com/lucidworks/indexing/config/IndexStageConfig.java)
Index Stage config defines configuration options specific to particular index stage instance. These options will be 
available to the end user via Fusion UI and API. The plugin config class must extend 
[com.lucidworks.indexing.config.IndexStageConfig](src/main/java/com/lucidworks/indexing/config/IndexStageConfig.java) 
and annotated with `@RootSchema`. 

By adding `@Property` and type annotations to your stage configuration interface methods, you can define metadata and
type constraints for your plugin configuration fields. This is very similar to Fusion's connector configuration schema
More detailed information on the configuration and schema capabilities can be found 
[here](https://doc.lucidworks.com/fusion-server/5.0/search-development/getting-data-in/connectors/sdk/java-sdk/java-connector-dev.html#configuration).

Here is an example of simple stage configuration schema definition:

```java
@RootSchema(
    title = "Simple",
    description = "Simple Index Stage"
)
public interface SimpleStageConfig extends IndexStageConfig {

  @Property(
      title = "Field",
      description = "Name of field to add to document.",
      required = true
  )
  @StringSchema(minLength = 1)
  String newField();

  @Property(
      title = "Text",
      description = "Sample text to put into the field."
  )
  @StringSchema
  String text();
}

```

## Exposed Fusion APIs
SDK based plugins are capable of communication with other Fusion's components via 
[Fusion](src/main/java/com/lucidworks/indexing/api/fusion/Fusion.java) object. This object is passed to stage on 
initialization phase. 

### [RestCall]((src/main/java/com/lucidworks/indexing/api/fusion/RestCall.java))
This API provides access to [Fusion REST API](https://doc.lucidworks.com/fusion-server/5.0/reference-guides/api/index.html). 
You can find an example of its usage in 
[ExternalQueryStage](../examples/sample-plugin-stage/src/main/java/com/lucidworks/sample/query/ExternalQueryStage.java)
class.  

### [Blobs]((src/main/java/com/lucidworks/indexing/api/fusion/Blobs.java))
This is specialized API for interactions with Fusion [Blob Store API](https://doc.lucidworks.com/fusion-server/5.0/reference-guides/api/blob-store-api.html)

### [Documents](src/main/java/com/lucidworks/indexing/api/fusion/Documents.java)
This API provides methods for creation of new [Document](src/main/java/com/lucidworks/indexing/api/Document.java) 
instances. It is useful for those stages that need to create new documents instead of or in addition to modifying the 
original input document passed to the stage. 

# Data structures

## [Document](src/main/java/com/lucidworks/indexing/api/Document.java)
This is a representation of a Fusion document - base data structure stage is operating with. On a high level 
this can be thought of as a list of named fields containing one or multiple values.

New documents can be created using the [Documents](src/main/java/com/lucidworks/indexing/api/fusion/Documents.java) API.

## Field
Field is a part of document with its own unique name and one or more values. 
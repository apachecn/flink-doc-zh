---
title: "Table Sources & Sinks"
nav-parent_id: tableapi
nav-pos: 40
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

`TableSource`提供对存储在外部系统（database, key-value store, message queue）或者文件的数据进行访问。当[TableSource在TableEnvironment下注册](common.html#register-a-tablesource)后，就可以通过 [Table API](tableApi.html)或[SQL](sql.html)查询进行访问。

TableSource可以将[表发射](common.html#emit-a-table)到外部存储系统，例如database, key-value store, message queue或者文件系统（不同编码，例如，CSV，Parquet，或ORC）。

关于如何[注册TableSource](common.html#register-a-tablesource)及如何[通过TableSink发射表](common.html#emit-a-table)，参阅[common concepts and API](common.html)。

* This will be replaced by the TOC
{:toc}

Provided TableSources
---------------------

目前，Flink提供`CsvTableSource`读取CSV文件及部分table sources读取来自Kafka的JSON或Avro数据。关于通过使用`BatchTableSource`或`StreamTableSource`接口自定义TableSource的详情，参阅 [defining a custom TableSource](#define-a-tablesource)。

| **Class name** | **Maven dependency** | **Batch?** | **Streaming?** | **Description**
| `Kafka011AvroTableSource` | `flink-connector-kafka-0.11` | N | Y | A `TableSource` for Avro-encoded Kafka 0.11 topics.
| `Kafka011JsonTableSource` | `flink-connector-kafka-0.11` | N | Y | A `TableSource` for flat Json-encoded Kafka 0.11 topics.
| `Kafka010AvroTableSource` | `flink-connector-kafka-0.10` | N | Y | A `TableSource` for Avro-encoded Kafka 0.10 topics.
| `Kafka010JsonTableSource` | `flink-connector-kafka-0.10` | N | Y | A `TableSource` for flat Json-encoded Kafka 0.10 topics.
| `Kafka09AvroTableSource` | `flink-connector-kafka-0.9` | N | Y | A `TableSource` for Avro-encoded Kafka 0.9 topics.
| `Kafka09JsonTableSource` | `flink-connector-kafka-0.9` | N | Y | A `TableSource` for flat Json-encoded Kafka 0.9 topics.
| `Kafka08AvroTableSource` | `flink-connector-kafka-0.8` | N | Y | A `TableSource` for Avro-encoded Kafka 0.8 topics.
| `Kafka08JsonTableSource` | `flink-connector-kafka-0.8` | N | Y | A `TableSource` for flat Json-encoded Kafka 0.8 topics.
| `CsvTableSource` | `flink-table` | Y | Y | A simple `TableSource` for CSV files.
| `OrcTableSource` | `flink-orc` | Y | N | A `TableSource` for ORC files.

所有标注为`flink-table`依赖的sources都可以直接用于Table API或SQL程序。对于其他的table sources，除了`flink-table`依赖，你需要添加相应的依赖。

{% top %}

### KafkaJsonTableSource

`KafkaJsonTableSource`从Kafka主题中提取JSON-encoded信息。目前，仅支持平面式(非嵌入式)模式的JSON记录。

`KafkaJsonTableSource`使用builder创建和进行配置，下面示例展示如何使用基础属性创建`KafkaJsonTableSource`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// create builder
KafkaTableSource source = Kafka010JsonTableSource.builder()
  // set Kafka topic
  .forTopic("sensors")
  // set Kafka consumer properties
  .withKafkaProperties(kafkaProps)
  // set Table schema
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG())  
    .field("temp", Types.DOUBLE())
    .field("time", Types.SQL_TIMESTAMP()).build())
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// create builder
val source: KafkaTableSource = Kafka010JsonTableSource.builder()
  // set Kafka topic
  .forTopic("sensors")
  // set Kafka consumer properties
  .withKafkaProperties(kafkaProps)
  // set Table schema
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG)
    .field("temp", Types.DOUBLE)
    .field("time", Types.SQL_TIMESTAMP).build())
  .build()
{% endhighlight %}
</div>
</div>

#### 可选配置

* **时间属性：**详情参阅[configuring a rowtime attribute](#configure-a-rowtime-attribute) 及 [configuring a processing time attribute](#configure-a-processing-time-attribute)。
* **Explicit JSON parse schema：**默认情况下，JSON记录使用表模式进行分析，你可以配置explicit JSON parse，并提供表模式字段到 JSON字段的映射，如下图所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Map<String, String> mapping = new HashMap<>();
mapping.put("sensorId", "id");
mapping.put("temperature", "temp");

KafkaTableSource source = Kafka010JsonTableSource.builder()
  // ...
  // set Table schema
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG())
    .field("temperature", Types.DOUBLE()).build())
  // set JSON parsing schema
  .forJsonSchema(TableSchema.builder()
    .field("id", Types.LONG())
    .field("temp", Types.DOUBLE()).build())
  // set mapping from table fields to JSON fields
  .withTableToJsonMapping(mapping)
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val source: KafkaTableSource = Kafka010JsonTableSource.builder()
  // ...
  // set Table schema
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG)
    .field("temperature", Types.DOUBLE).build())
  // set JSON parsing schema
  .forJsonSchema(TableSchema.builder()
    .field("id", Types.LONG)
    .field("temp", Types.DOUBLE).build())
  // set mapping from table fields to JSON fields
  .withTableToJsonMapping(Map(
    "sensorId" -> "id", 
    "temperature" -> "temp").asJava)
  .build()
{% endhighlight %}
</div>
</div>

* **处理缺失字段：**默认情况下，JSON的缺失字段设为`null`，你也可以启用strict JSON parsing，如果存在缺失字段，它会取消source(查询)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
KafkaTableSource source = Kafka010JsonTableSource.builder()
  // ...
  // configure missing field behavior
  .failOnMissingField(true)
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val source: KafkaTableSource = Kafka010JsonTableSource.builder()
  // ...
  // configure missing field behavior
  .failOnMissingField(true)
  .build()
{% endhighlight %}
</div>
</div>

* **指定起始阅读位置：**默认情况下，table source从Zookeeper或 Kafka brokers已提交的组偏移量中读取数据。你也可以通过builder方法指定开始位置，对应于[Kafka Consumers Start Position Configuration](../connectors/kafka.html#kafka-consumers-start-position-configuration)中的配置。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
KafkaTableSource source = Kafka010JsonTableSource.builder()
  // ...
  // start reading from the earliest offset
  .fromEarliest()
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val source: KafkaTableSource = Kafka010JsonTableSource.builder()
  // ...
  // start reading from the earliest offset
  .fromEarliest()
  .build()
{% endhighlight %}
</div>
</div>

{% top %}

### KafkaAvroTableSource

`KafkaAvroTableSource`从Kafka主题提取Avro-encoded记录。

`KafkaAvroTableSource`是使用builder创建并进行配置，以下示例展示如何使用基础属性创建`KafkaAvroTableSource`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// create builder
KafkaTableSource source = Kafka010AvroTableSource.builder()
  // set Kafka topic
  .forTopic("sensors")
  // set Kafka consumer properties
  .withKafkaProperties(kafkaProps)
  // set Table schema
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG())
    .field("temp", Types.DOUBLE())
    .field("time", Types.SQL_TIMESTAMP()).build())
  // set class of Avro record
  .forAvroRecordClass(SensorReading.class)
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// create builder
val source: KafkaTableSource = Kafka010JsonTableSource.builder()
  // set Kafka topic
  .forTopic("sensors")
  // set Kafka consumer properties
  .withKafkaProperties(kafkaProps)
  // set Table schema
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG)
    .field("temp", Types.DOUBLE)
    .field("time", Types.SQL_TIMESTAMP).build())
  // set class of Avro record
  .forAvroRecordClass(classOf[SensorReading])
  .build()
{% endhighlight %}
</div>
</div>

**注意：**指定的Avro记录类必须提供相应类型的表模式的所有字段。

#### 可选配置

* **时间属性：**参阅[configuring a rowtime attribute](#configure-a-rowtime-attribute)和[configuring a processing time attribute](#configure-a-processing-time-attribute)。
* **Explicit Schema Field to Avro Mapping：**默认情况下，表模式所有字段按名字映射到Avro记录的字段。如果Avro记录的字段具有不同名字，则可以指定从表模式到Avro的字段映射。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Map<String, String> mapping = new HashMap<>();
mapping.put("sensorId", "id");
mapping.put("temperature", "temp");

KafkaTableSource source = Kafka010AvroTableSource.builder()
  // ...
  // set Table schema
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG())
    .field("temperature", Types.DOUBLE()).build())
  // set class of Avro record with fields [id, temp]
  .forAvroRecordClass(SensorReading.class)
  // set mapping from table fields to Avro fields
  .withTableToAvroMapping(mapping)
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val source: KafkaTableSource = Kafka010AvroTableSource.builder()
  // ...
  // set Table schema
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG)
    .field("temperature", Types.DOUBLE).build())
  // set class of Avro record with fields [id, temp]
  .forAvroRecordClass(classOf[SensorReading])
  // set mapping from table fields to Avro fields
  .withTableToAvroMapping(Map(
    "sensorId" -> "id", 
    "temperature" -> "temp").asJava)
  .build()
{% endhighlight %}
</div>
</div>

* **指定起始阅读位置：**默认情况下，table source从Zookeeper或 Kafka brokers已提交的组偏移量中读取数据。你也可以通过builder方法指定起始位置，相关配置参阅[Kafka Consumers Start Position Configuration](../connectors/kafka.html#kafka-consumers-start-position-configuration)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
KafkaTableSource source = Kafka010AvroTableSource.builder()
  // ...
  // start reading from the earliest offset
  .fromEarliest()
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val source: KafkaTableSource = Kafka010AvroTableSource.builder()
  // ...
  // start reading from the earliest offset
  .fromEarliest()
  .build()
{% endhighlight %}
</div>
</div>

{% top %}

### 配置处理时间属性

[Processing time attributes](streaming.html#processing-time)常用于流式查询。处理时间属性返回其操作员的当前wall-clock time。

批查询也支持处理时间属性。但是处理时间属性使用表扫描运算符的wall-clock time进行初始化，并在整个查询评估中保持其值。

`SQL-TIMESTAMP`的表模式字段可被声明为处理时间属性，操作如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
KafkaTableSource source = Kafka010JsonTableSource.builder()
  // ... 
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG())  
    .field("temp", Types.DOUBLE())
    // field "ptime" is of type SQL_TIMESTAMP
    .field("ptime", Types.SQL_TIMESTAMP()).build())
  // declare "ptime" as processing time attribute
  .withProctimeAttribute("ptime")
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val source: KafkaTableSource = Kafka010JsonTableSource.builder()
  // ...
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG)
    .field("temp", Types.DOUBLE)
    // field "ptime" is of type SQL_TIMESTAMP
    .field("ptime", Types.SQL_TIMESTAMP).build())
  // declare "ptime" as processing time attribute
  .withProctimeAttribute("ptime")
  .build()
{% endhighlight %}
</div>
</div>

{% top %}

### 配置Rowtime属性

[Rowtime attributes](streaming.html#event-time) 是`TIMESTAMP`类型的属性，在流式和批查询中有统一处理方式。

`SQL_TIMESTAMP`类型的表模式字段通过指定，可声明为`rowtime`属性。

- 字段名
- `TimesStampExtractor`：计算属性的实际值（通常来自一个或多个其他属性）
- `WatermarkStrategy`：指定如何为rowtime属性生成水印

以下示例展示如何配置rowtime属性：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
KafkaTableSource source = Kafka010JsonTableSource.builder()
  // ...
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG())
    .field("temp", Types.DOUBLE())
    // field "rtime" is of type SQL_TIMESTAMP
    .field("rtime", Types.SQL_TIMESTAMP()).build())
  .withRowtimeAttribute(
    // "rtime" is rowtime attribute
    "rtime",
    // value of "rtime" is extracted from existing field with same name
    new ExistingField("rtime"),
    // values of "rtime" are at most out-of-order by 30 seconds
    new BoundedOutOfOrderWatermarks(30000L))
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val source: KafkaTableSource = Kafka010JsonTableSource.builder()
  // ...
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG)
    .field("temp", Types.DOUBLE)
    // field "rtime" is of type SQL_TIMESTAMP
    .field("rtime", Types.SQL_TIMESTAMP).build())
  .withRowtimeAttribute(
    // "rtime" is rowtime attribute
    "rtime",
    // value of "rtime" is extracted from existing field with same name
    new ExistingField("rtime"),
    // values of "rtime" are at most out-of-order by 30 seconds
    new BoundedOutOfOrderTimestamps(30000L))
  .build()
{% endhighlight %}
</div>
</div>

#### 提取Kafka 0.10+ 时间戳到Rowtime属性

从Kafka0.10开始，Kafka消息用时间戳作为元数据，指定记录被写入到Kafka主题的时间。`KafkaTableSources`可分配Kafka的消息时间戳为rowtime属性，如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
KafkaTableSource source = Kafka010JsonTableSource.builder()
  // ...
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG())
    .field("temp", Types.DOUBLE())
    // field "rtime" is of type SQL_TIMESTAMP
    .field("rtime", Types.SQL_TIMESTAMP()).build())
  // use Kafka timestamp as rowtime attribute
  .withKafkaTimestampAsRowtimeAttribute()(
    // "rtime" is rowtime attribute
    "rtime",
    // values of "rtime" are ascending
    new AscendingTimestamps())
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val source: KafkaTableSource = Kafka010JsonTableSource.builder()
  // ...
  .withSchema(TableSchema.builder()
    .field("sensorId", Types.LONG)
    .field("temp", Types.DOUBLE)
    // field "rtime" is of type SQL_TIMESTAMP
    .field("rtime", Types.SQL_TIMESTAMP).build())
  // use Kafka timestamp as rowtime attribute
  .withKafkaTimestampAsRowtimeAttribute()(
    // "rtime" is rowtime attribute
    "rtime",
    // values of "rtime" are ascending
    new AscendingTimestamps())
  .build()
{% endhighlight %}
</div>
</div>

#### 提供的TimestampExtractors

Flink为常见用例提供`TimestampExtractors`实现。以下展示了目前可用的`TimestampExtractors`实现：

- `ExistingField(fieldName)`：从现存的`LONG`或`SQL_TIMESTAMP`字段提取rowtime属性值
- `StreamRecordTimestamp()`：从`DataStream` `StreamRecord`的时间戳提取rowtime属性值。注意，这里的TimestampExtractors不能用于批处理的table sources

通过使用相应的接口，自定义`TimestampExtractors`。

#### 提供的WatermarkStrategies

Flink为常见用例提供`WatermarkStrategies`实现。以下为目前可用的`WatermarkStrategies`实现：

- `AscendingTimestamps`：`ascending timestamps`的水印策略。带有无序时间戳的记录被认为是滞后的
- `BoundedOutOfOrderTimestamps(delay)`：在指定的延迟中最无序的时间戳的水印策略

使用相应的接口自定义`WatermarkStrategies`。

{% top %}

### CsvTableSource

`CsvTableSource`已被包含在`flink-table`中，无需额外依赖。

最简单创建`CsvTableSource`的方法，是使用随附的构建器`CsvTableSource.builder()`，通过以下方法来配置构建器的属性：

- `path(String path)`：设置要求的CSV文件路径
- `field(String fieldName, TypeInformation<?> fieldType)`：添加包含字段名和字段类型信息的字段，且可被多次调用。此方法的调用顺序也定义了行的字段顺序
- `fieldDelimiter(String delim)`：默认设置字段分隔符为`“，”`
- `lineDelimiter(String delim)`：默认设置行分隔符为“\n”
- `quoteCharacter(Character quote)`：可设置quote character为字符串值，默认为`null`
- `commentPrefix(String prefix)`：设置一个前缀来表示注释，默认为null
- `ignoreFirstLine()`：忽视第一行。默认为禁用。
- `ignoreParseErrors()`：跳过有解析错误的记录，而不报错，默认为抛出异常

你可以按如下步骤创建source：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
CsvTableSource csvTableSource = CsvTableSource
    .builder()
    .path("/path/to/your/file.csv")
    .field("name", Types.STRING())
    .field("id", Types.INT())
    .field("score", Types.DOUBLE())
    .field("comments", Types.STRING())
    .fieldDelimiter("#")
    .lineDelimiter("$")
    .ignoreFirstLine()
    .ignoreParseErrors()
    .commentPrefix("%")
    .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val csvTableSource = CsvTableSource
    .builder
    .path("/path/to/your/file.csv")
    .field("name", Types.STRING)
    .field("id", Types.INT)
    .field("score", Types.DOUBLE)
    .field("comments", Types.STRING)
    .fieldDelimiter("#")
    .lineDelimiter("$")
    .ignoreFirstLine
    .ignoreParseErrors
    .commentPrefix("%")
    .build
{% endhighlight %}
</div>
</div>

{% top %}

### OrcTableSource

通过`OrcTableSource`读取[ORC files](https://orc.apache.org)。ORC是结构化数据的文件格式，并以一个压缩的列式表示存储数据。ORC具有很高的存储效率，并支持projection和filter push-down。

`OrcTableSource`可以按如下方式创建：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

// create Hadoop Configuration
Configuration config = new Configuration();

OrcTableSource orcTableSource = OrcTableSource.builder()
  // path to ORC file(s). NOTE: By default, directories are recursively scanned.
  .path("file:///path/to/data")
  // schema of ORC files
  .forOrcSchema("struct<name:string,addresses:array<struct<street:string,zip:smallint>>>")
  // Hadoop configuration
  .withConfiguration(config)
  // build OrcTableSource
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

// create Hadoop Configuration
val config = new Configuration()

val orcTableSource = OrcTableSource.builder()
  // path to ORC file(s). NOTE: By default, directories are recursively scanned.
  .path("file:///path/to/data")
  // schema of ORC files
  .forOrcSchema("struct<name:string,addresses:array<struct<street:string,zip:smallint>>>")
  // Hadoop configuration
  .withConfiguration(config)
  // build OrcTableSource
  .build()
{% endhighlight %}
</div>
</div>

**注意：**`OrcTableSource`目前不支持ORC的`Union`类型。

{% top %}

提供的TableSinks
-------------------

下表列举Flink中提供的`TableSinks`：

| **Class name** | **Maven dependency** | **Batch?** | **Streaming?** | **Description**
| `CsvTableSink` | `flink-table` | Y | Append | A simple sink for CSV files.
| `JDBCAppendTableSink` | `flink-jdbc` | Y | Append | Writes a Table to a JDBC table.
| `CassandraAppendTableSink` | `flink-connector-cassandra` | N | Append | Writes a Table to a Cassandra table. 
| `Kafka08JsonTableSink` | `flink-connector-kafka-0.8` | N | Append | A Kafka 0.8 sink with JSON encoding.
| `Kafka09JsonTableSink` | `flink-connector-kafka-0.9` | N | Append | A Kafka 0.9 sink with JSON encoding.
| `Kafka010JsonTableSink` | `flink-connector-kafka-0.10` | N | Append | A Kafka 0.10 sink with JSON encoding.

所有标注`flink-table`依赖的sinks可以直接用在Table程序中。对于其他的table sinks，除了`flink-table`依赖外，你需要添加相应的依赖。

利用`BatchTableSink`，`RetractStreamTableSink`，或者`UpsertStreamTableSink`接口可以自定义`TableSink`，参阅 [defining a custom TableSink](#define-a-tablesink)。

{% top %}

### KafkaJsonTableSink

`KafkaJsonTableSink`发射一个[streaming append `Table`](./streaming.html#table-to-stream-conversion)到Apache Kafka主题。表的行被编码为JSON记录。目前，仅支持平面模式的表，即非嵌入式字段。

如果查询在[启用检查点]({{ site.baseurl }}/dev/stream/state/checkpointing.html#enabling-and-configuring-checkpointing)的情况下执行，`KafkaJsonTableSink`会有多重保证生成一个Kafka主题。

默认情况下，`KafkaJsonTableSink`写入至多与其自身并行性相同的分区(每个sink的并行实例只写入一个分区)。为了将写入分配到更多分区或为了控制行到分区的路由，可以利用自定义`FlinkKafkaJsonTableSink`。

下面示例展示如何在Kafka 0.10下创建`KafkaJsonTableSink`。Kafka 0.8及和0.9的sinks也可类似创建。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

Table table = ...

Properties props = new Properties();
props.setProperty("bootstrap.servers", "localhost:9092");

table.writeToSink(
  new Kafka010JsonTableSink(
    "myTopic",                // Kafka topic to write to
    props));                  // Properties to configure the producer

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

val table: Table = ???

val props = new Properties()
props.setProperty("bootstrap.servers", "localhost:9092")

table.writeToSink(
  new Kafka010JsonTableSink(
    "myTopic",                // Kafka topic to write to
    props))                   // Properties to configure the producer

{% endhighlight %}
</div>
</div>

### CsvTableSink

`CsvTableSink`发射`Table`到一个或多个CSV文件。

sink仅支持append-only的流式表，不能发射持续更新的`Table`。具体可参阅[documentation on Table to Stream conversions](./streaming.html#table-to-stream-conversion)。发射流式表时，行至少被写入一次（如果启用检查点），同时`CsvTableSink`不会将输出文件拆分为存储区文件，但会不断向同一个文件写入。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

Table table = ...

table.writeToSink(
  new CsvTableSink(
    path,                  // output path 
    "|",                   // optional: delimit files by '|'
    1,                     // optional: write to a single file
    WriteMode.OVERWRITE)); // optional: override existing files

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

val table: Table = ???

table.writeToSink(
  new CsvTableSink(
    path,                             // output path 
    fieldDelim = "|",                 // optional: delimit files by '|'
    numFiles = 1,                     // optional: write to a single file
    writeMode = WriteMode.OVERWRITE)) // optional: override existing files

{% endhighlight %}
</div>
</div>

### JDBCAppendTableSink

`JDBCAppendTableSink`发射`Table`到 JDBC连接。sink仅支持append-only的流式表，不能用于发射持续更新的`Table`，具体参阅 [documentation on Table to Stream conversions](./streaming.html#table-to-stream-conversion)。

`JDBCAppendTableSink`至少在数据库表中插入一行（如果启用检查点）。但是，你可以使用<code>REPLACE</code>或<code>INSERT OVERWRITE</code>向数据库执行upsert writes，指定插入查询。

要使用JDBC接收器，你需要添加JDBC连接器依赖(<code>flink-jdbc</code>)到你的项目中，然后你可以用<code>JDBCAppendSinkBuilder</code>来创建接收器：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

JDBCAppendTableSink sink = JDBCAppendTableSink.builder()
  .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
  .setDBUrl("jdbc:derby:memory:ebookshop")
  .setQuery("INSERT INTO books (id) VALUES (?)")
  .setParameterTypes(INT_TYPE_INFO)
  .build();

Table table = ...
table.writeToSink(sink);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val sink: JDBCAppendTableSink = JDBCAppendTableSink.builder()
  .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
  .setDBUrl("jdbc:derby:memory:ebookshop")
  .setQuery("INSERT INTO books (id) VALUES (?)")
  .setParameterTypes(INT_TYPE_INFO)
  .build()

val table: Table = ???
table.writeToSink(sink)
{% endhighlight %}
</div>
</div>

同使用<code>JDBCOutputFormat</code>，你需要明确指定JDBC驱动的名称，the JDBC URL，要执行的查询，和JDBC表的字段类型。

{% top %}

### CassandraAppendTableSink

`CassandraAppendTableSink`将表发射到Cassandra table。sink仅支持append-only的流式表，不能用于发射持续更新的表，具体参阅[documentation on Table to Stream conversions](./streaming.html#table-to-stream-conversion)。

如果启用检查点，`CassandraAppendTableSink`将所有行至少一次插入到Cassandra table，但你也可以指定为upsert查询。

为了使用`CassandraAppendTableSink`，你需要添加Cassandra连接器的依赖(<code>flink-connector-cassandra</code>)到你的项目里，下面的示例展示如何使用`CassandraAppendTableSink`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

ClusterBuilder builder = ... // configure Cassandra cluster connection

CassandraAppendTableSink sink = new CassandraAppendTableSink(
  builder, 
  // the query must match the schema of the table
  INSERT INTO flink.myTable (id, name, value) VALUES (?, ?, ?));

Table table = ...
table.writeToSink(sink);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val builder: ClusterBuilder = ... // configure Cassandra cluster connection

val sink: CassandraAppendTableSink = new CassandraAppendTableSink(
  builder, 
  // the query must match the schema of the table
  INSERT INTO flink.myTable (id, name, value) VALUES (?, ?, ?))

val table: Table = ???
table.writeToSink(sink)
{% endhighlight %}
</div>
</div>

{% top %}

定义TableSource
--------------------

`TableSource`是一个通用接口，它是Table API和SQL查询可以访问外部存储系统中的数据。它提供表模式及映射到表模式的记录。根据`TableSource`用于流查询还是批查询，记录相应的生成为`DataSet`或`DataStream`形式。

如果`TableSource`被用于流式查询，就需要引用`StreamTableSource`接口；如果被用于批查询，就需要引入`BatchTableSource`接口。由此，`TableSource`可以用于两个接口，并用于流式查询和批查询。

`StreamTableSource`和`BatchTableSource`扩展了`TableSource`的基础接口，定义如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
TableSource<T> {

  public TableSchema getTableSchema();

  public TypeInformation<T> getReturnType();

  public String explainSource();
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
TableSource[T] {

  def getTableSchema: TableSchema

  def getReturnType: TypeInformation[T]

  def explainSource: String

}
{% endhighlight %}
</div>
</div>

- `getTableSchema()`：返回表模式，即表字段的名称和类型。字段类型通过Flink的TypeInformation定义，参阅[Table API types](tableApi.html#data-types)和 [SQL types](sql.html#data-types)。
- `getReturnType()`：返回`DataStream(StreamTableSource)`或`DataSet (BatchTableSource)` 的物理类型及由`TableSource`生成的记录
- `explainSource()`：返回描述`TableSource`的字符串。这个方法是可选的，仅用于显示。

`TableSource`接口将逻辑表模式与返回`DataStream`或`DataSet`的物理类型分隔开。因此，所有的表模式字段(`getTableSchema()`)必须被映射到具有相应类型的物理返回类型(`getReturnType()`)。默认情况下，是基于字段名进行映射。例如，定义有两个字段`[name: String, size: Integer]`表模式的`TableSource`需要一个`TypeInformation`，其至少有两个称为`name`和`size`的字段名，且类型对应为`String`和`Integer`。这种情况可以采用`PojoTypeInfo`或`RowTypeInfo`，它们都有两个名为`name`和`size`的字段，且类型相匹配。

但是，一些类型，例如`Tuple`或`CaseClass`类型，不支持自定义字段名。如果`TableSource`返回具有固定字段名的`DataStream`或`DataSet`类型，则可用`DefinedFieldMapping`接口将字段名从表模式映射到物理返回类型的字段名。

### 定义BatchTableSource

`BatchTableSource`接口扩展了`TableSource`接口，并定义了一个方法：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
BatchTableSource<T> extends TableSource<T> {

  public DataSet<T> getDataSet(ExecutionEnvironment execEnv);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
BatchTableSource[T] extends TableSource[T] {

  def getDataSet(execEnv: ExecutionEnvironment): DataSet[T]
}
{% endhighlight %}
</div>
</div>

* `getDataSet(execEnv)`：用表数据返回`DataSet`。`DataSet`类型必须与`TableSource.getReturnType()`方法定义的返回类型相同。`DataSet`可以利用`DataSet API`常规数据源创建。通常，`BatchTableSource`可通过包装`InputFormat`或[batch connector]({{ site.baseurl }}/dev/batch/connectors.html)来实现。

{% top %}

### 定义StreamTableSource

`StreamTableSource`接口扩展了`TableSource`接口，并定义了一个方法：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamTableSource<T> extends TableSource<T> {

  public DataStream<T> getDataStream(StreamExecutionEnvironment execEnv);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
StreamTableSource[T] extends TableSource[T] {

  def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[T]
}
{% endhighlight %}
</div>
</div>

* `getDataStream(execEnv)`：用表数据返回`DataStream`。`DataStream`类型需要与`TableSource.getReturnType()`方法定义的返回类型一致。`DataStream`可利用`DataStream API`的常规[data source]({{ site.baseurl }}/dev/datastream_api.html#data-sources)创建。通常，`StreamTableSource`通过包装一个`SourceFunction`或[stream connector]({{ site.baseurl }}/dev/connectors/)实现。

### 用时间属性定义TableSource

基于时间操作的流式[Table API](tableApi.html#group-windows)和[SQL](sql.html#group-windows)查询，例如窗口式的聚合或连接，需要明确指定[time attributes]({{ site.baseurl }}/dev/table/streaming.html#time-attributes)。

`TableSource`在其表模式下，定义时间属性为`Types.SQL_TIMESTAMP`类型的字段。不同于模式下的所有常规字段，时间属性不能与表源返回类型的物理字段相匹配。相反，`TableSource`通过应用某些接口定义时间属性

#### 定义处理时间属性

`TableSource`通过`DefinedProctimeAttribute`接口定义[processing time attribute](streaming.html#processing-time)。接口如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DefinedProctimeAttribute {

  public String getProctimeAttribute();
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
DefinedProctimeAttribute {

  def getProctimeAttribute: String
}
{% endhighlight %}
</div>
</div>

- `getProctimeAttribute()`：返回处理时间属性的名称。在表模式下，指定的属性必须定义`Types.SQL_TIMESTAMP`类型，并可利用基于时间的操作。一个`DefinedProctimeAttribute`表源可以不定义处理时间属性，并返回`null`。

**注意：**`StreamTableSource`和`BatchTableSource`都可利用`DefinedProctimeAttribute`，定义处理数据属性。在`BatchTableSource`下，处理时间字段在表扫描期间基于当前时间戳被初始化。

#### 定义Rowtime属性

`TableSource`通过`DefinedRowtimeAttributes`接口定义[rowtime attribute](streaming.html#event-time)，接口如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DefinedRowtimeAttribute {

  public List<RowtimeAttributeDescriptor> getRowtimeAttributeDescriptors();
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
DefinedRowtimeAttributes {

  def getRowtimeAttributeDescriptors: util.List[RowtimeAttributeDescriptor]
}
{% endhighlight %}
</div>
</div>

* `getRowtimeAttributeDescriptors()`：返回一个列表`RowtimeAttributeDescriptor`。`RowtimeAttributeDescriptor`利用以下属性描述rowtime属性：
  - `attributeName`：表模式下的rowtime属性名，字段名需要用`Types.SQL_TIMESTAMP`类型定义
  - `timestampExtractor`：时间戳提取器从返回类型的记录中提取时间戳，例如，它可以将`Long`字段转为一个时间戳，或解析一个字符串编码的时间戳。Flink为常见用例提供一组内置`TimestampExtractor`实现，也可以提供自定义实现
  - `watermarkStrategy`：水印策略定义了如何为rowtime属性定义水印。Flink为常见用例提供一组内置`WatermarkStrategy`实现，也可以提供自定义实现
* **注意：**尽管`getRowtimeAttributeDescriptors()`方法返回描述列表，按目前仅支持一个rowtime属性。我们计划在将来消除此限制，使其支持不止一个rowtime属性

**注意：**`StreamTableSource`和`BatchTableSource`可以使用`DefinedRowtimeAttributes`，并定义rowtime属性。在任何情况下，rowtime字段利用`TimestampExtractor`进行提取。因此，应用`StreamTableSource`和`BatchTableSource`，并定义rowtime属性的`TableSource`，为流式和批查询提供相同的数据。

{% top %}

### 使用Projection Push-Down定义TableSource

`TableSource`利用`ProjectableTableSource`接口支持Projection Push-Down，如下接口定义了一个方法：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ProjectableTableSource<T> {

  public TableSource<T> projectFields(int[] fields);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
ProjectableTableSource[T] {

  def projectFields(fields: Array[Int]): TableSource[T]
}
{% endhighlight %}
</div>
</div>

- `projectFields(fields)`：返回一个已调整物理返回类型的`TableSource`副本，字段参数提供必需由`TableSource`提供字段的索引。字段索引与物理返回类型的`TypeInformation`相关，而不是逻辑表模式。`TableSource`副本需要调整其返回类型为`DataStream`或`DataSet`。`TableSource`副本的`TableSchema`不能变，即字段映射必须调整为新的返回类型

`ProjectableTableSource`支持平面式项目字段。如果`TableSource`定义一个嵌入模式的表，它可以使用`NestedFieldsProjectableTableSource`扩展投影到嵌入字段。`NestedFieldsProjectableTableSource`的定义如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
NestedFieldsProjectableTableSource<T> {

  public TableSource<T> projectNestedFields(int[] fields, String[][] nestedFields);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
NestedFieldsProjectableTableSource[T] {

  def projectNestedFields(fields: Array[Int], nestedFields: Array[Array[String]]): TableSource[T]
}
{% endhighlight %}
</div>
</div>

* `projectNestedField(fields, nestedFields)`：返回一个已调整物理返回类型的`TableSource`副本。物理返回类型的字段可能被移除或重新排序，但他们的类型不会变。该方法的`contract`与`ProjectableTableSource.projectFields()`方法的`contract`基本相同。此外，`nestedFields`参数包含字段列表中的每个字段索引，及查询访问的所有嵌入式字段的路径列表。其他嵌入式字段不需要被读取，解析，或设置到由`TableSource`生成的记录中。

**注意：**映射字段的类型不能更改，但未使用的字段可能设置为空或为默认值。

{% top %}

### 用Filter Push-Down定义TableSource

`FilterableTableSource`接口为`TableSource`增加了对filter push-down的支持。一个`TableSource`扩展了这个接口，使其能过滤记录，从而返回的`DataStream`或`DataSet`返回较少记录。

接口如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
FilterableTableSource<T> {

  public TableSource<T> applyPredicate(List<Expression> predicates);

  public boolean isFilterPushedDown();
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
FilterableTableSource[T] {

  def applyPredicate(predicates: java.util.List[Expression]): TableSource[T]

  def isFilterPushedDown: Boolean
}
{% endhighlight %}
</div>
</div>

* `applyPredicate(predicates)`：返回添加了谓词的`TableSource`，该谓词参数是“提供”给TableSource的连接谓词的一个可变列表。`TableSource`通过从列表中移除谓词来对其进行评估。列表中剩余的谓词将会被后续的filter operator评估
* `isFilterPushedDown()`：如果`applyPredicate()`方法在之前被调用过，则返回true。因此，`isFilterPushedDown()`必须对从`applyPredicate()`调用返回的`TableSource`实例返回`true`

{% top %}

定义TableSink
------------------

`TableSink`指定了如何向外部存储系统或地址发射表。这个接口是通用的，所以支持不同的存储位置及格式。批处理表和流处理表有不同的`table sinks`。

通用接口如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
TableSink<T> {

  public TypeInformation<T> getOutputType();

  public String[] getFieldNames();

  public TypeInformation[] getFieldTypes();

  public TableSink<T> configure(String[] fieldNames, TypeInformation[] fieldTypes);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
TableSink[T] {

  def getOutputType: TypeInformation<T>

  def getFieldNames: Array[String]

  def getFieldTypes: Array[TypeInformation]

  def configure(fieldNames: Array[String], fieldTypes: Array[TypeInformation]): TableSink[T]
}
{% endhighlight %}
</div>
</div>

`TableSink#configure`方法被调用来传递表（字段名和类型）的模式发送到`ableSink`。该方法必须返回一个配置为发射已提供的表模式的`TableSink`的新实例。

### BatchTableSink

定义一个外部`TableSink`发射一个批处理表。

接口如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
BatchTableSink<T> extends TableSink<T> {

  public void emitDataSet(DataSet<T> dataSet);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
BatchTableSink[T] extends TableSink[T] {

  def emitDataSet(dataSet: DataSet[T]): Unit
}
{% endhighlight %}
</div>
</div>

{% top %}

### AppendStreamTableSink

定义一个只通过更改插件来发射流式表的外部`TableSink`。

接口如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
AppendStreamTableSink<T> extends TableSink<T> {

  public void emitDataStream(DataStream<T> dataStream);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
AppendStreamTableSink[T] extends TableSink[T] {

  def emitDataStream(dataStream: DataStream<T>): Unit
}
{% endhighlight %}
</div>
</div>

如果表被更新或删除改变，会抛出一个`TableException`。

{% top %}

### RetractStreamTableSink

定义一个通过插入，更新及删除更改来发射流式表的外部`TableSink`。

接口如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
RetractStreamTableSink<T> extends TableSink<Tuple2<Boolean, T>> {

  public TypeInformation<T> getRecordType();

  public void emitDataStream(DataStream<Tuple2<Boolean, T>> dataStream);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
RetractStreamTableSink[T] extends TableSink[Tuple2[Boolean, T]] {

  def getRecordType: TypeInformation[T]

  def emitDataStream(dataStream: DataStream[Tuple2[Boolean, T]]): Unit
}
{% endhighlight %}
</div>
</div>

表被转换为一个流式的编码为Java `Tuple2`的累积和撤回消息。第一个字段是用来知识消息类型的布尔型标志（`true`指示插入，`false`指示删除）。第二个字段包含请求类型 `T`的记录。

{% top %}

### UpsertStreamTableSink

定义一个通过插入，更新和删除变化来发射流式表的外部`TableSink`。

接口如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
UpsertStreamTableSink<T> extends TableSink<Tuple2<Boolean, T>> {

  public void setKeyFields(String[] keys);

  public void setIsAppendOnly(boolean isAppendOnly);

  public TypeInformation<T> getRecordType();

  public void emitDataStream(DataStream<Tuple2<Boolean, T>> dataStream);
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
UpsertStreamTableSink[T] extends TableSink[Tuple2[Boolean, T]] {

  def setKeyFields(keys: Array[String]): Unit

  def setIsAppendOnly(isAppendOnly: Boolean): Unit

  def getRecordType: TypeInformation[T]

  def emitDataStream(dataStream: DataStream[Tuple2[Boolean, T]]): Unit
}
{% endhighlight %}
</div>
</div>

表必须具有唯一的键字段（`atomic`或`composite`）或为append-only，否则会抛出一个`TableException`。表的唯一键通过`UpsertStreamTableSink#setKeyFields()`方法配置。

表会被转为编码为Java `Tuple2`的流式的`upsert`和`delete`消息。第一个字段是知识消息类型的布尔标志，第二个包含要求类型T的记录。

带有true布尔字段的消息是配置键的`upsert`消息。带有false标志的消息配置键的`delete`消息。如果表为append-only，所有消息都带有`true`标志，且必须解释为插入。

{% top %}


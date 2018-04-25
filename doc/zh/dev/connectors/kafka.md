---
title: "Apache Kafka Connector"
nav-title: Kafka
nav-parent_id: connectors
nav-pos: 1
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

* This will be replaced by the TOC
{:toc}

这个连接器提供通过[Apache Kafka](https://kafka.apache.org/)进入事件流的服务。

Flink为kafka专门提供了一个连接器用来向kafka的topics发送和写入数据。Flink kafka的消费者整合Flink的检查点机制以提供恰好一次（exactly-once）的处理语义。为了达到这个目标，Flink并不完全依赖卡夫卡的消费群体偏移量跟踪，而是也会在内部跟踪和检查这些偏移量。

请针对您的用例和环境的选择一个包（maven artifact id）和类名。对大多数用户来说`FlinkKafkaConsumer08`（`flink-connector-kafka`的一部分）时比较合适的。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">Maven依赖</th>
      <th class="text-left">支持版本</th>
      <th class="text-left">消费者和<br>
      生产者类名</th>
      <th class="text-left">kafka版本</th>
      <th class="text-left">备注</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer08<br>
        FlinkKafkaProducer08</td>
        <td>0.8.x</td>
        <td>使用kafka内部的 <a href="https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example">SimpleConsumer</a> API。 Offsets 通过Flink提交到ZK（zookeeper）</td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.9{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer09<br>
        FlinkKafkaProducer09</td>
        <td>0.9.x</td>
        <td>使用新的kafkd <a href="http://kafka.apache.org/documentation.html#newconsumerapi">Consumer API</a> 。</td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.10{{ site.scala_version_suffix }}</td>
        <td>1.2.0</td>
        <td>FlinkKafkaConsumer010<br>
        FlinkKafkaProducer010</td>
        <td>0.10.x</td>
        <td>这个连接器支持 <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message">有时间戳的kafka message</a>，无论是producing还是consuming。</td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.11_2.11</td>
        <td>1.4.0</td>
        <td>FlinkKafkaConsumer011<br>
        FlinkKafkaProducer011</td>
        <td>0.11.x</td>
        <td>0.11.x版本的Kafka不支持scala 2.10.这个连接器支持 <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging">传统的Kafka messaging</a>用以对producer提供“恰好一次”的语义（semantic）。</td>
    </tr>
  </tbody>
</table>

然后，在你的maven中导入connector:

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

请注意，目前流式连接器（streaming connectors）不在二进制发行版本中。在[这里]({{ site.baseurl}}/dev/linking.html)查看如何将这两者关联起来。

## 安装Apache kafka

* 按照[Kafka's quickstart](https://kafka.apache.org/documentation.html#quickstart)中的说明去下载代码并启动服务器（在开始程序时，每次都要启动Zookeeper和kafka服务）。
* 如果kafka和Zookeeper服务运行子啊远程的机器上，那么在`config/server.properties`文件中需设置`advertised.host.name`来指定机器的IP地址。

## kafka消费者

Flink的kafka消费者为`FlinkKafkaConsumer08`（或者`09`对应kafka 0.9.0.x的版本等等）。它支持访问一个或多个的topics。

构造函数接收以下参数：

1. 指定的topic名或列表
2. 一个将kafka来的数据进行反序列化的DeserializationSchema或KeyedDeserializationSchema
3. kafka消费者的属性
  以下时必须要有的属性：
  - "bootstrap.servers" (以逗号分割的kafka broker列表)
  - "zookeeper.connect" (以逗号为分割的Zookeeper服务) (**只有kafka 0.8 需要**)
  - "group.id" 指定消费者的所在的组

例如：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
DataStream<String> stream = env
	.addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181")
properties.setProperty("group.id", "test")
stream = env
    .addSource(new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties))
    .print()
{% endhighlight %}
</div>
</div>

### The `DeserializationSchema`

Flink kafka消费者需要知道如何去将kafka中的二进制数据转换为Java/Scala的对象。而`DeserializationSchema`能让用户来指定这样的模型（schema）。其中`T deserialize(byte[] message)`方法会被每个kafka的消息（message）调用，并传递从kafka来的值。

从`AbstractDeserializationSchema`开始通常会很有帮助，它将负责描述为Flink的类型系统生成了Java / Scala类型。实现了`DeserializationSchema`的用户需要自己实现实现其中的`getProducedType(...)`方法。

为了访问kafka的键和值，`KeyedDeserializationSchema`有下列的反序列化方法：
` T deserialize(byte[] messageKey, byte[] message, String topic, int partition, long offset)`。

为了方便起见，Flink提供了以下模式：

1.`TypeInformationSerializationSchema`（和`TypeInformationKeyValueSerializationSchema`），它创建基于Flink的`TypeInformation`的模式（schema）。如果数据是同时写入和读取的，这会很有用。
    这是Flink用来替代其他通用的序列化方法而使用的高性能的模式（schema）。

2.`JsonDeserializationSchema`（和`JSONKeyValueDeserializationSchema`），它通过使用objectNode.get("field").as(Int/String/...)()，将连续的json数据转换成ObjectNode对象。
	键值objectNode包含一个具有所有字段的“键”和“值”的字段，以及一个可选的“元数据（metadata）”字段用来公开message的offset/partition/topic。
    
遇到因任何原因无法反序列化的损坏消息时，有两个选项 - 从`deserialize（...）`方法抛出异常这会导致作业失败并重新启动，或者返回`null`来允许FlinkKafka消费者默默地跳过损坏的消息。 注意由于消费者的容错性（更多细节请参见下面的部分），损坏消息上的失败作业将会让消费者尝试再次反序列化消息。 因此，如果反序列化仍然失败，那么消费者将陷入不间断的重启和不断得对这个损坏消息的失败操作。

### Kafka消费者 起始定位配置

Flink kafka消费者允许配置kafka分区决定的起始位置。

比如：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

FlinkKafkaConsumer08<String> myConsumer = new FlinkKafkaConsumer08<>(...);
myConsumer.setStartFromEarliest();     // start from the earliest record possible
myConsumer.setStartFromLatest();       // start from the latest record
myConsumer.setStartFromGroupOffsets(); // the default behaviour

DataStream<String> stream = env.addSource(myConsumer);
...
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val myConsumer = new FlinkKafkaConsumer08[String](...)
myConsumer.setStartFromEarliest()      // start from the earliest record possible
myConsumer.setStartFromLatest()        // start from the latest record
myConsumer.setStartFromGroupOffsets()  // the default behaviour

val stream = env.addSource(myConsumer)
...
{% endhighlight %}
</div>
</div>

所有版本的Flink kafka消费者，对于起始位置都有以上明确的配置方法。

 * `setStartFromGroupOffsets`（默认行为）：开始从消费者组（`group.id`在消费者属性中设置）在kafka broker（或者是kafka0.8的Zookeeper）提交offset信息的分区进行读操作。如果分区中没有发现offset信息，那么属性中的`auto.offset.reset`配置将会被使用。
 * `setStartFromEarliest()` / `setStartFromLatest()`:从更早的/更晚的记录开始。在这个模式下，在kafka提交offset将被忽略并且不会被用做起始位置。
  
  
你也能够指定每一个分区应该开始的那个精确的消费者偏移：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Map<KafkaTopicPartition, Long> specificStartOffsets = new HashMap<>();
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L);

myConsumer.setStartFromSpecificOffsets(specificStartOffsets);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val specificStartOffsets = new java.util.HashMap[KafkaTopicPartition, java.lang.Long]()
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L)

myConsumer.setStartFromSpecificOffsets(specificStartOffsets)
{% endhighlight %}
</div>
</div>

上面的例子配置消费者在`myTopic`这个tipic中的分区0，1和2，它们从指定的偏移开始。偏移值应该是每一个消费者会在每一个分区读的下一个记录。注意如果消费者需要读的分区没有在提供的offset映射（map）中找到指定的偏移（offset），那么这个特殊的分区将会回滚去使用默认的消费者组offset行为（比如`setStartFromGroupOffsets()`）。

注意当任务在失败后自动重启或者手动使用savepoint重启时，这些起始定位配置方法将无法对起始定位生效。在恢复过程中，每一个kafka分区的起始定位取决于存储在savepoint或者checkpoint的offset（请查看下一节关于启动consumer容错的checkpoint的信息）。

### kafka消费者和容错

当Flink启动了checkpoint，那么Flink kafka消费者将从topic消费记录（record）并定期checkpoint它所有的kafka offset，同时也会checkpoint其他操作的状况（state），这是以一致的方式进行的。在任务失败的情况下，Flink将以最后一次checkpoint的状态恢复流式的程序（streaming program），并且从kafka重新消费记录，这是从checkpoint中保存的offset开始的。

因此，在生成每个checkpoin的间隔之间定义了在失败的时候，最多可以返回多少程序。

为了使用可容错的kafka消费者，checkpoint拓扑需要在excution环境开启：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(5000); // checkpoint every 5000 msecs
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.enableCheckpointing(5000) // checkpoint every 5000 msecs
{% endhighlight %}
</div>
</div>

同时也要注意，如果有足够多的运行中slot可以用来重启拓扑，Flink只能重启拓扑。
所以如果拓扑因为TaskManager而失败，那么之后也必须要有足够的slot。

如果checkpoint没有开启，kafka消费者将定期提交offset呆Zookeeper。

### kafka消费者topic和分区检测

#### 分区检测

Flink kafka消费者提供动态地检测创建kafka分区（partitions），并且这些消费者有着恰好一次的语义。所有被元数据分区恢复初始化后被检索到的分区（比如当job开始运行）将会从最早的偏移（offset）开始消费。

默认是没有开启分区检测的。要开启它，可以在提供的数据配置中设置`flink.partition-discovery.interval-millis`不为负值，这代表以毫秒为单位的检测间隔。

<span class="label label-danger">局限性</span>，在Flink的1.3.x更早的版本中，当消费者从savepoint中恢复过来时，分区检测并不会启动。如果要启动，那么恢复过程将失败并抛出异常。在这种情况，为了使用分区检测，请首先将savepoint升级到1.3.x并再起重启。

#### topic检测

在高层抽象（higher-level）中，Flink kafka消费者也是有检测topic的能力的，这是基于正则表达式来对topic名的模式匹配。查看以下例子：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer011<String> myConsumer = new FlinkKafkaConsumer011<>(
    java.util.regex.Pattern.compile("test-topic-[0-9]"),
    new SimpleStringSchema(),
    properties);

DataStream<String> stream = env.addSource(myConsumer);
...
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
properties.setProperty("group.id", "test")

val myConsumer = new FlinkKafkaConsumer08[String](
  java.util.regex.Pattern.compile("test-topic-[0-9]"),
  new SimpleStringSchema,
  properties)

val stream = env.addSource(myConsumer)
...
{% endhighlight %}
</div>
</div>

在上述例子中，在job开始运行的时候，所有的匹配指定正则表达式的topic名的topic将被消费者订阅。

为了使消费者能在job启动后动态地检测创建的topic，可以设置`flink.partition-discovery.interval-millis`为非空值。这可以允许消费者检查匹配指定正则表达式的新topic的分区。

### kafka消费者offset提交行为配置

Flink kafka消费者允许配置offset如何提交回到kafka的broker（或是0.8的zookeeper）的行为。注意Flink kafka消费者没有依赖于提交offset来保证容错性。提交offset仅仅是为了监控的目的而暴漏消费者的处理过程。

配置offset提交行为的在某些情况下是不相同的，这取决于job是否开启checkpoint。

 - *不开启Checkpointing：*如果没有开启checkpoint机制，Flink kafka消费者将依靠内部kafka客户端的定期提交offset的能力。因此，要开启或关闭offset提交，通过设`enable.auto.commit`（或在kafka 0.8为`auto.commit.enable`）/`auto.commit.interval.ms`键来使值适配于提供的`Properties`配置。
 
 - *开启Checkpointing：*如果开启checkpoint，Flink kafka消费者将在checkpoint完成时提交offset并存储在checkpoint状态（state）中。这确保kafka broker中已提交的offset和checkpoint状态中的是一致的。用户能够通过消费者调用（默认，这些行为是`true`）`setCommitOffsetsOnCheckpoints(boolean)`方法选择开启或关闭offset提交。
注意在这种情况下，在`Properties`中自动周期性地提交offset的配置将会被完全忽略。

### kafka消费者时间戳提取/水印（指时间戳的水印）发送

在许多场景，记录（record）的时间戳是嵌入到（显示或隐式）记录自身当中的。
另外，用户可能想要定期发送时间戳水印，或者是以不定期的方式。比如基于包含当前时间水印的kafka stream的记录。对于这种情况，Flink kafka消费者允许指定一个`AssignerWithPeriodicWatermarks`或一个`AssignerWithPunctuatedWatermarks`。

你能指定你习惯的时间戳提取/水印发送，在[这]({{ site.baseurl }}/apis/streaming/event_timestamps_watermarks.html)查看，或者使用[预定义的]({{ site.baseurl }}/apis/streaming/event_timestamp_extractors.html)一个。这样做后，您可以通过以下方式将其传递给消费者：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer08<String> myConsumer =
    new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties);
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter());

DataStream<String> stream = env
	.addSource(myConsumer)
	.print();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181")
properties.setProperty("group.id", "test")

val myConsumer = new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties)
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter())
stream = env
    .addSource(myConsumer)
    .print()
{% endhighlight %}
</div>
</div>

在内部，每个kafka分区都会执行一个分配器实例。
当指定一个分配器，对于从kafka读取的每一条记录，`extractTimestamp(T element, long previousElementTimestamp)`会被调用来为记录分配一个时间戳（定期）或`Watermark checkAndGetNextWatermark(T lastElement, long extractedTimestamp)`（为了打断）会被调用来决定是否发送一个新的时间戳水印和应该发送哪一个水印。


## kafka生产者

Flinks的kafka生产者被叫做`FlinkKafkaProducer011`（在kafka 0.10.0.x被叫做`010`）。它允许写入一个记录流到一个或多个kafka topic。

比如：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<String> stream = ...;

FlinkKafkaProducer011<String> myProducer = new FlinkKafkaProducer011<String>(
        "localhost:9092",            // broker list
        "my-topic",                  // target topic
        new SimpleStringSchema());   // serialization schema

// versions 0.10+ allow attaching the records' event timestamp when writing them to Kafka;
// this method is not available for earlier Kafka versions
myProducer.setWriteTimestampToKafka(true);

stream.addSink(myProducer);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[String] = ...

val myProducer = new FlinkKafkaProducer011[String](
        "localhost:9092",         // broker list
        "my-topic",               // target topic
        new SimpleStringSchema)   // serialization schema

// versions 0.10+ allow attaching the records' event timestamp when writing them to Kafka;
// this method is not available for earlier Kafka versions
myProducer.setWriteTimestampToKafka(true)

stream.addSink(myProducer)
{% endhighlight %}
</div>
</div>

上述例子演示了基础的创建一个Flink kafka生产者去写一个流（stream）到单个目标topic。对于更高级的用法，还有其他的构造函数变体可以提供以下内容：

 * *提供自定义属性*：
 生产者可以为内部`KafkaProducer`提供一个自定义属性配置。请查看[Apache Kafka 文档](https://kafka.apache.org/documentation.html)以获取更多如何配置kafka生产者的细节。
 * *自定义分区*:为了分配记录到指定分区，你可以为构造函数提供`FlinkKafkaPartitioner`的实现。将为流中  的每条记录调用此分区器，以确定记录应发送到的目标主题的哪个确切分区。在[Kafka Producer Partitioning Scheme](#kafka-producer-partitioning-scheme)查看更多详细信息。
 * *高级序列化模型*：与消费者类似，生产者同业可以通过调用`KeyedSerializationSchema`使用高级序列化模型，这可以分别序列化键和值。它也允许覆盖目标topic，所以这一生产者实例可以发送数据到多个topic。
 
### kafka生产者分区模型
 
默认地，如果没有为Flink kafka生产者指定自定义分区器，那么生产者将使用一个`FlinkFixedPartitioner`来让每一个Flink kafka生产者的并行的子任务到单个kafka分区（比如，接收器子任务接收到的所有记录都将在相同的Kafka分区中结束）

自定义的分区器能够通过继承`FlinkKafkaPartitioner`来实现。所有的版本的kafka在实例化生产者时，都在构造器提中供自定义分区器。注意实现分区器比如时可序列化的，因为它们将通过Flink节点进行传输。
另外，请记住，自分区程序以来，分区程序中的任何状态在作业失败时都会丢失。

It is also possible to completely avoid using and kind of partitioner, and simply let Kafka partition
the written records by their attached key (as determined for each record using the provided serialization schema).
To do this, provide a `null` custom partitioner when instantiating the producer. It is important
to provide `null` as the custom partitioner; as explained above, if a custom partitioner is not specified
the `FlinkFixedPartitioner` is used instead.

### kafka生产者和容错

#### Kafka 0.8

在kafka0.9之前，kafka为提供任何最少一次语义或仅有一次语义保证的机制。

#### kafka 0.9和0.10

如果启动了checkpoint，那么`FlinkKafkaProducer09`和`FlinkKafkaProducer010`可以提供最少一次投递的保证。

除了开启Flink的checkpoint，你同时应该适当地配置`setLogFailuresOnly(boolean)`和`setFlushOnCheckpoint(boolean)`。

 * `setLogFailuresOnly(boolean)`：默认地，这被设置为`false`。
 开启这个将会让生产者仅记录失败日志而不是捕获或是重新抛出它们。这实质上是记录已成功的，即使它从未写入目标kafka主题。这种情况必须禁用最少一次（at-least-once）。
 * `setFlushOnCheckpoint(boolean)`：默认得，这个设置为`false`。
 如果开启这个配置，Flink的checkpoint将在成功地checkpoint之前将一直等待kafka确认检查点时的任何即时记录。这确保在checkpoint之前的记录会写入到kafka。这必须启动最少一次的配置（at-least-once）。
 
结论是，为了配置kafka0.9和0.10版本的生产者有最少一次的保证，`setLogFailureOnly`必须设置为`false`且`setFlushOnCheckpoint`必须设置为`true`。

**注意**：默认地，重试的次数设置为0.这意味着当`setLogFailuresOnly`被设置为`false`时，生产者的失败立即error，包括leader发生改变。这个值被默认设置为0以避免因为多次重试导致目标topic中有重复的消息。对于大多数broker会频繁改变的环境来说，我们推荐你将重试次数设置为更高的值。

**注意**：当前对kafka的生产者并非事务性的，所以Flink无法保证仅有一次投递到kafka topic。

<div class="alert alert-warning">
  <strong>请注意:</strong> 根据Kafka的配置，即使在Kafka确认写入后，数据仍可能会丢失。请特别注意以下Kafka设置：
  <ul>
    <li><tt>acks</tt></li>
    <li><tt>log.flush.interval.messages</tt></li>
    <li><tt>log.flush.interval.ms</tt></li>
    <li><tt>log.flush.*</tt></li>
  </ul>
  默认地，上面选项的选择很容易导致数据丢失。请参考kafka文档以查看更多说明。

</div>

#### Kafka 0.11

若开启Flink checkponit，`FlinkKafkaProducer011`将提供仅有一次的投递保证。

除了开启Flink checkpoint，你也可以选择三种不同的操作模式，通过选择`FlinkKafkaProducer011`的属性参数`semantic`来决定：

 * `Semantic.NONE`：Flink将不做任何保证。生产的记录可能丢失或可能重复。
 * `Semantic.AT_LEAST_ONCE` (默认设置)：与`FlinkKafkaProducer010`中的`setFlushOnCheckpoint(true)`类似。它能保证没有记录会丢失（虽然它们可能重复）。
 * `Semantic.EXACTLY_ONCE`：使用kafka事务以提供仅有一次的语义。当你写入到kafka并用到事务的时候，别忘了对每一个消费记录的程序设置期望的`隔离级别`（`read_committed`或`read_uncommitted` -后一个是默认值）。

<div class="alert alert-warning">
  <strong>注意:</strong> 根据Kafka的配置，即使在Kafka确认写入后，数据仍可能会丢失。请特别注意以下Kafka设置：
  in Kafka config:
  <ul>
    <li><tt>acks</tt></li>
    <li><tt>log.flush.interval.messages</tt></li>
    <li><tt>log.flush.interval.ms</tt></li>
    <li><tt>log.flush.*</tt></li>
  </ul>
  默认地，上面选项的选择很容易导致数据丢失。请参考kafka文档以查看更多说明。
</div>


##### Caveats

`Semantic.EXACTLY_ONCE`模式依赖于在从所述检查点恢复之后提交在获取检查点之前启动事务的能力。如果Flink应用程序崩溃和完成重新启动之间的时间较长，则Kafka的事务超时将导致数据丢失（Kafka会自动放弃超时的事务）。考虑到这一点，请将你的事务超时配置为你预期的停机时间。

kafka broker默认地将`transaction.max.timeout.ms`设置为15分治。不可以将生产者的事务超时时间设置为大于它的值。`FlinkKafkaProducer011`默认得在生产者配置中设置`transaction.timeout.ms`为1小时，并且当使用`Semantic.EXACTLY_ONCE`模式时，应该将 thus `transaction.max.timeout.ms`设置得更大些。


在`KafkaConsumer`的`read_committed`模式下，任务没有完成的事务将会阻塞所有从给定kafka tipic的读未完成的事务。换句话说，在遵循以下顺序的事件之后：

1. 用户开启 `transaction1` 并使用它来写入一些记录
2. 用户开启 `transaction2` 并进一步使用它写入一些记录
3. 用户提交`transaction2`

即使`transaction2`中的记录已经提交，但它对消费者来说时不可见的，指导`transaction1`提交完成或丢弃。这里有两个含义：

 * 首先，在Flink应用程序的正常工作期间，用户可以预计在kafka topic中产生的记录的可见性延迟，等于完成的检查点之间的平均时间。
 * 其次在Flink应用程序失败的情况下,应用程序写入的主题将被读取器阻塞，直到应用程序重新启动或配置的事务超时时间过去。此说法仅适用于有多个代理/应用程序写入同一个Kafka主题的情况

**注意**:`Semantic.EXACTLY_ONCE` 模式下在每一个`FlinkKafkaProducer011`实例化时都使用一个固定大小的kafka生产者池。每个检查点使用每个生产者中的一个。如果并发检查点的数量超过池大小，`FlinkKafkaProducer011`将抛出一个异常并使整个程序失败。请根据此配置最大池容量以及最大当前检查点数。

**注意**：`Semantic.EXACTLY_ONCE` 会采取一切可能的措施，不留下任何将阻塞消费者从kafka topic中阅读的滞留事务，这是必要的。但是，如果Flink应用程序在第一个检查点之前发生故障，那么在重新启动此类应用程序之后，系统中将没有关于先前池大小的信息。因此，在第一个检查点完成之前缩小Flink应用程序是不安全的，因为它比`FlinkKafkaProducer011.SAFE_SCALE_DOWN_FACTOR`大。

## 在kafka0.10使用kafka时间戳和Flink事件时间

从Apache kafka 0.10+以来，kafka的消息能够携带[时间戳](https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message)，指明已经发生事件的时间(看[Apache Flink中的"事件事件"](../event_time.html))或者消息被写入kafka broker的时间。

如果Flink的时间特性被设置为`TimeCharacteristic.EventTime` (`StreamExecutionEnvironment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)`)的话，那么`FlinkKafkaConsumer010`将会发送附带着时间戳的记录（records）。

kafka消费者发出时不附加水印时间戳。要发出附带水印的话，这种机制和前面描述的"kafka消费者和时间戳 抽取/打水印 发出"是一样的，使用`assignTimestampsAndWatermarks` 方法是适用的。

当在kafka使用时间戳时，不需要定义定义时间戳抽取器。在`extractTimestamp()`方法中的`previousElementTimestamp`参数包含了携带的时间戳信息。

一个kafka消费者的时间戳抽取器可能像这样：
{% highlight java %}
public long extractTimestamp(Long element, long previousElementTimestamp) {
    return previousElementTimestamp;
}
{% endhighlight %}



如果设置了`setWriteTimestampToKafka(true)`的话，那么`FlinkKafkaProducer010`只发出时间戳记录。

{% highlight java %}
FlinkKafkaProducer010.FlinkKafkaProducer010Configuration config = FlinkKafkaProducer010.writeToKafkaWithTimestamps(streamWithTimestamps, topic, new SimpleStringSchema(), standardProps);
config.setWriteTimestampToKafka(true);
{% endhighlight %}



## Kafka连接器metrics

Flink的kafka连接器通过Flink的[metrics系统]({{ site.baseurl }}/monitoring/metrics.html)提供一些metrics，用来分析连接器的行为。
生产者通过Flink的metric系统导出Kafka内部的metrics到所有支持的版本。
从kafka0.9版本开始，消费者导出所有metrics开始的信息。Kafka的[文档](http://kafka.apache.org/documentation/#selector_monitoring)中列出所有导出metrics的信息。

除了这些metrics，所有的消费者会对每一个topic的分区（partition）暴露其`current-offsets`和`committed-offsets`。
而`current-offsets`引用着当前分区的offset。它引用着我们最后检索和发送成功的元素的offset。`committed-offsets`则是最后committed的offset。

Flink的kafka消费者将ofset提交回Zookeeper(Kafka 0.8)或是提交到brokers(Kafka 0.9+)。如果不允许检查点（checkpointing）么offset回定期提交。如果使用检查点，那么一旦流式拓扑中的所有算子都确认他们已经创建了状态的检查点，就会提交。
这给用户提供了提交offset到Zookeeper或是broker的最少一次（at-least-once）的语义。
对Flink的checkpointed的偏移记录，系统提供了仅有一次（exactly once）的保证。

offset提交到ZK或是broker也可以用来跟踪kafka消费者的消费进度。对每一个分区，提交（committed）offset和最近的（the most recent）offset的不同被称之为*消费者落后（consumer lag）*。如果Flink拓扑从topic消费数据的速度比新数据到达的速度慢，那么这个落后会加大并且消费者会赶不上。
对于大型生产部署，我们推荐监控metric以避免增加落后量。

## 允许Kerberos的身份认证（只对0.9+的版本有效）

Flink通过kafka连接器进行身份验证以提供最好的对kafka安装Kerberos的配置的支持。简单得在`flink-conf.yaml`进行配置以启用对kafka的Kerberos身份验证如下：

 1. 通过配置Kerberos证书如下 -
 - `security.kerberos.login.use-ticket-cache`:默认得，这个值为`true`并且Flink会通过`kinit`在ticket缓存中尝试使用Kerberos证书。
 注意当使用部署在YARN的kafka连接器到Flink job时，Kerberos授权将使用的ticket缓存将失效。当部署在Mesos也一样，ticket缓存同意不支持Mesos不是的方式。
 - `security.kerberos.login.keytab` 和 `security.kerberos.login.principal`:为了改为使用Kerberos密钥表，需要为这两个属性设置值。
 
2. 追加`KafkaClient`到`security.kerberos.login.contexts`：这个配置会告诉Flink去提供Kerberos证书配置到kafka登陆上下文中，从而被用来kafka身份验证。

一旦启用基于Kerberos的Flink安全机制，您可以通过Flink Kafka Consumer或Producer向Kafka进行身份验证，只需在提供的属性配置中包含以下两个设置，并将其传递给内部Kafka客户端：

- 将`security.protocol`设置为`SASL_PLAINTEXT`（默认为`NONE`）：这一协议用于与kafkabrokers交互。
当使用standalone部署Flink时，你也能够使用`SASL_SSL`;请在[这](https://kafka.apache.org/documentation/#security_configclients)查看如何配置Kafka客户端以获取SSL。
- 将`sasl.kerberos.service.name`设置为`kafka`（默认为`kafka`）：这个值需要与kafka broker的配置`sasl.kerberos.service.name`的值匹配。服务端和客户端配置之间的服务名称不匹配将会导致验证失败。

更多Flink的Kerberos安全机制的配置，请看[这里]({{ site.baseurl}}/ops/config.html)。
同时您能够通过[这里]({{ site.baseurl}}/ops/security-kerberos.html)进一步查看更多Flink内部基于Kerberos的安全机制的细节。

{% top %}

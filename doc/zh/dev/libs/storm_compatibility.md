---
title: "Storm 兼容性"
is_beta: true
nav-parent_id: libs
nav-pos: 2
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

[Flink streaming]({{ site.baseurl }}/dev/datastream_api.html) 与 Apache Storm 接口是兼容的，因此可以重用 Storm 实现的代码.

您可以 :

- 在 Flink 中执行一个完整的 Storm `Topology`.
- 在 Flink 流式程序中使用 Storm `Spout`/`Bolt` 作为 source/operator.

该文档展示了如何搭配 Flink 使用现有的 Storm 代码。

* This will be replaced by the TOC
{:toc}

# Project Configuration（项目配置）

Storm 的支持包含在 Maven module 的 `flink-storm` 之中。
这些代码放在 `org.apache.flink.storm` 包中。

如果您想要在 Flink 中执行 Storm 代码的话，可以添加以下的依赖到你的 `pom.xml` 文件中去。

~~~xml
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-storm{{ site.scala_version_suffix }}</artifactId>
	<version>{{site.version}}</version>
</dependency>
~~~

**Please note**: 请不要将 `storm-core` 作依赖而添加. 它已经包含在 `flink-storm` 之中了。

**Please note**: `flink-storm` 不是 Flink 二进制发行版之中的一部分。
因此，您需要在您的程序 jar （也可以称作 uber-jar 或 fat-jar）中去包含 `flink-storm` 类（以及它们的依赖），用于提交到 Flink 的 JobManager 中。
请在 `flink-storm-examples/pom.xml` 之中参阅 *WordCount Storm* 示例，以了解如何正确的打包一个 jar。

如果您想要去避免 uber-jar 太大的话，您可以手动的复制 `storm-core-0.9.4.jar`, `json-simple-1.1.jar` 和 `flink-storm-{{site.version}}.jar` 到 Flink 集群的每个节点的 `lib/` 文件夹中（在集群启动 *之前* 就处理好） 。
在这种情况下，只需要将您自己的 Spout 和 Blot 类（以及它们的依赖）打包到程序 jar 中即可。

# Execute Storm Topologies（执行 Storm 拓扑）

Flink 提供了一个 Storm 兼容下的 API（`org.apache.flink.storm.api`） ，它为以下类提供了替换操作 : 

- `StormSubmitter` 被 `FlinkSubmitter` 所替换
- `NimbusClient` 和 `Client` 被 `FlinkClient` 所替换
- `LocalCluster` 被 `FlinkLocalCluster` 所替换

为了提交一个 Storm topology 到 Flink 中，在 Storm 中 *打包拓扑的客户端代码* 中用它们的 Flink 替换替换已使用的 Storm 类就足够了。
实际上运行的代码，即，Spouts 和 Bolts，可以 *不用修改* 。
如果 topology 在一个远程的集群中执行，参数 `nimbus.host` 和 `nimbus.thrift.port` 分别用于参数 `jobmanger.rpc.address` 和 `jobmanger.rpc.port`。
如果不指定参数，则从 `flink-conf.yaml` 配置文件中提取 。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
~~~java
TopologyBuilder builder = new TopologyBuilder(); // the Storm topology builder

// actual topology assembling code and used Spouts/Bolts can be used as-is
builder.setSpout("source", new FileSpout(inputFilePath));
builder.setBolt("tokenizer", new BoltTokenizer()).shuffleGrouping("source");
builder.setBolt("counter", new BoltCounter()).fieldsGrouping("tokenizer", new Fields("word"));
builder.setBolt("sink", new BoltFileSink(outputFilePath)).shuffleGrouping("counter");

Config conf = new Config();
if(runLocal) { // submit to test cluster
	// replaces: LocalCluster cluster = new LocalCluster();
	FlinkLocalCluster cluster = new FlinkLocalCluster();
	cluster.submitTopology("WordCount", conf, FlinkTopology.createTopology(builder));
} else { // submit to remote cluster
	// optional
	// conf.put(Config.NIMBUS_HOST, "remoteHost");
	// conf.put(Config.NIMBUS_THRIFT_PORT, 6123);
	// replaces: StormSubmitter.submitTopology(topologyId, conf, builder.createTopology());
	FlinkSubmitter.submitTopology("WordCount", conf, FlinkTopology.createTopology(builder));
}
~~~
</div>
</div>

# Embed Storm Operators in Flink Streaming Programs（在 Flink 流式程序中嵌入 Storm 操作）

一种选择就是，Spouts 和 Bolts 可以嵌入到常规的流式程序中。
Storm 兼容层为它们提供了一个包装类，叫做 `SpoutWrapper` 和 `BoltWrapper` （`org.apache.flink.storm.wrappers`） 。

默认情况下，两个包装器都将 Storm 输出的 tuples 转换成 Flink 的 [Tuple]({{site.baseurl}}/dev/api_concepts.html#tuples-and-case-classes) 类型（即，`Tuple0` 到 `Tuple25` 是根据 Storm tuples 字段数据类确定的）
针对单个字段的输出 tuples，也可以转换为字段的数据类型（例如，`String` 而不是 `Tuple1<String>`）

由于 Flink 不能够推测出 Storm 操作输出的字段类型，所以需要手动的去指定输出类型。
为了可以正确的获取对象的 `TypeInformation` ，Flink 的 `TypeExtractor` 是可用的。

## Embed Spouts（嵌入 Spouts）

为了使用 Spout 作为 Flink 的 source，可以使用 `StreamExecutionEnvironment.addSource(SourceFunction, TypeInformation)` 。
该 Spout 对象传递给 `SpoutWrapper<OUT>` 的构造函数，它作为 `addSource(...)` 方法的第一个参数。
泛型类型声明 `OUT` 指定了源输出流的类型。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
~~~java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// stream has `raw` type (single field output streams only)
DataStream<String> rawInput = env.addSource(
	new SpoutWrapper<String>(new FileSpout(localFilePath), new String[] { Utils.DEFAULT_STREAM_ID }), // emit default output stream as raw type
	TypeExtractor.getForClass(String.class)); // output type

// process data stream
[...]
~~~
</div>
</div>

如果 Spout 发生一个有限数量的 tuples， `Spout Wrapper` 可以配置为通过在其构造函数中设置 `numberOfInvocations` 参数来自动终止。
这个可以让 Flink 程序在所有数据处理完成后自动的停止。
默认情况下，程序将会一直运行，直到它手动的被 [取消]({{site.baseurl}}/ops/cli.html) 。

## Embed Bolts（嵌入 Bolts）

为了使用 Bolt 作为 Flink 的操作器，可以使用 `DataStream.transform(String, TypeInformation, OneInputStreamOperator)` 。
该 Blot 对象传递给 `BoltWrapper<IN,OUT>` 的构造函数，它作为 `transform(...)` 方法的最后一个参数。
泛型类型声明 `IN` 和 `OUT` 分别指定了输入输出流中操作器的类型。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
~~~java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStream<String> text = env.readTextFile(localFilePath);

DataStream<Tuple2<String, Integer>> counts = text.transform(
	"tokenizer", // operator name
	TypeExtractor.getForObject(new Tuple2<String, Integer>("", 0)), // output type
	new BoltWrapper<String, Tuple2<String, Integer>>(new BoltTokenizer())); // Bolt operator

// do further processing
[...]
~~~
</div>
</div>

### Named Attribute Access for Embedded Bolts（嵌入式 Blots 的名称属性访问）

Bolts 可以通过 name（也可以通过索引）来访问输入的 tuple 字段。
要使用嵌入 bolts 的这个特性，您需要有一个 
To use this feature with embedded Bolts, you need to have either a

 1. [POJO]({{site.baseurl}}/dev/api_concepts.html#pojos) 类型的输入流或
 2. [Tuple]({{site.baseurl}}/dev/api_concepts.html#tuples-and-case-classes) 类型的输入流，并且指定了输入的 schema (即. name 到 index 的映射)

针对 POJO 输入类型，Flink 可以通过反射来访问其字段。
在这种情况下，Flink 假设它有相对应的 public member variable 或 public getter method。
例如，如果一个 Bolt 通过 name `sentence` 访问了一个字段（例如， `String s = input.getStringByField("sentence");`，该输入的 POLO 类必须有一个成员变量 `public String sentence;` 或方法 `public String getSentence() { ... };` (pay attention to camel-case naming)

针对 `Tuple` 的输入类型，需要使用 Storm 的 `Fields` 类来指定输入的 schema。
在这种情况下，`BoltWrapper` 的构造方法会有一个额外的参数: `new BoltWrapper<Tuple1<String>, ...>(..., new Fields("sentence"))`.
该输入类型是 `Tuple1<String>` 和 `Fields("sentence")` 指定了 `input.getStringByField("sentence")` ，等价于 `input.getString(0)`.

这些示例请参阅 [BoltTokenizerWordCountPojo](https://github.com/apache/flink/tree/master/flink-contrib/flink-storm-examples/src/main/java/org/apache/flink/storm/wordcount/BoltTokenizerWordCountPojo.java) 和 [BoltTokenizerWordCountWithNames](https://github.com/apache/flink/tree/master/flink-contrib/flink-storm-examples/src/main/java/org/apache/flink/storm/wordcount/BoltTokenizerWordCountWithNames.java).

## Configuring Spouts and Bolts（配置 Spouts 和 Bolts）

在 Storm 中，Spouts 和 Bolts 可以使用一个全局的分布式的 `Map` 对象来进行配置，它被赋予 `LocalCluster` 或 `StormSubmitter` 的 `submitTopology(...)` 方法。
该 `Map` 是由用户来提供的，并且作为参数传递到 `Spout.open(...)` 和 `Bolt.prepare(...)` 方法调用中。
如果在 Flink 中整个 topology 是使用 `FlinkTopologyBuilder` 来执行的话，没有特别的注意需求，一样可以工作的很好。

针对嵌入情况的使用，Flink 的配置机制必须是可用的。
一个全局的配置可以在一个 `StreamExecutionEnvironment` 中通过  `.getConfig().setGlobalJobParameters(...)` 方法来设置。
Flink 的常规配置 `Configuration` 可以用配置  Spouts and Bolts.
然而，`Configuration` 不支持任意的 key data types 作为 Storm 来工作（只有 `String` 是可用的）。
因此，Flink 还提供了 `StormConfig` 类，它可以像原始的 `Map` 一样使用，以提供与 Storm 的完全兼容。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
~~~java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

StormConfig config = new StormConfig();
// set config values
[...]

// set global Storm configuration
env.getConfig().setGlobalJobParameters(config);

// assemble program with embedded Spouts and/or Bolts
[...]
~~~
</div>
</div>

## Multiple Output Streams（多输出流）

Flink 还可以处理 Spout 和 Bolt 的多个输出流的声明。
如果在 Flink 中一个完整的 topology 是使用 `FlinkTopologyBuilder` 来执行的话，没有特别的注意需求，一样可以工作的很好。

针对嵌入情况的使用，输出流必须是数据类型 `SplitStreamType<T>` ，并且必须使用 `DataStream.split(...)` 和 `SplitStream.select(...)` 来拆分。
Flink 也为 `.split(...)` 提供了预定义的输出选择器 `StormStreamSelector<T>` 。
此外，可以使用  `SplitStreamMapper<T>` 来移除包装类型 `SplitStreamTuple<T>`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
~~~java
[...]

// get DataStream from Spout or Bolt which declares two output streams s1 and s2 with output type SomeType
DataStream<SplitStreamType<SomeType>> multiStream = ...

SplitStream<SplitStreamType<SomeType>> splitStream = multiStream.split(new StormStreamSelector<SomeType>());

// remove SplitStreamType using SplitStreamMapper to get data stream of type SomeType
DataStream<SomeType> s1 = splitStream.select("s1").map(new SplitStreamMapper<SomeType>()).returns(SomeType.class);
DataStream<SomeType> s2 = splitStream.select("s2").map(new SplitStreamMapper<SomeType>()).returns(SomeType.class);

// do further processing on s1 and s2
[...]
~~~
</div>
</div>

See [SpoutSplitExample.java](https://github.com/apache/flink/tree/master/flink-contrib/flink-storm-examples/src/main/java/org/apache/flink/storm/split/SpoutSplitExample.java) for a full example.

# Flink Extensions

## Finite Spouts

In Flink, streaming sources can be finite, ie, emit a finite number of records and stop after emitting the last record. However, Spouts usually emit infinite streams.
The bridge between the two approaches is the `FiniteSpout` interface which, in addition to `IRichSpout`, contains a `reachedEnd()` method, where the user can specify a stopping-condition.
The user can create a finite Spout by implementing this interface instead of (or additionally to) `IRichSpout`, and implementing the `reachedEnd()` method in addition.
In contrast to a `SpoutWrapper` that is configured to emit a finite number of tuples, `FiniteSpout` interface allows to implement more complex termination criteria.

Although finite Spouts are not necessary to embed Spouts into a Flink streaming program or to submit a whole Storm topology to Flink, there are cases where they may come in handy:

 * to achieve that a native Spout behaves the same way as a finite Flink source with minimal modifications
 * the user wants to process a stream only for some time; after that, the Spout can stop automatically
 * reading a file into a stream
 * for testing purposes

An example of a finite Spout that emits records for 10 seconds only:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
~~~java
public class TimedFiniteSpout extends BaseRichSpout implements FiniteSpout {
	[...] // implement open(), nextTuple(), ...

	private long starttime = System.currentTimeMillis();

	public boolean reachedEnd() {
		return System.currentTimeMillis() - starttime > 10000l;
	}
}
~~~
</div>
</div>

# Storm Compatibility Examples（Storm 兼容性示例）

您可以在 Maven module `flink-storm-examples` 中找到更多的示例。
对于不同版本的 WordCount，请参阅 [README.md](https://github.com/apache/flink/tree/master/flink-contrib/flink-storm-examples/README.md).
要运行这些示例，您需要打包一个正确的 jar 文件，
`flink-storm-examples-{{ site.version }}.jar` 对于 job 执行来说是无效的 jar 文件（它只是一个标准的 maven 

针对嵌入的 Spout 和 Bolt 有一些示例 jar，名为 `WordCount-SpoutSource.jar` 和 `WordCount-BoltTokenizer.jar` 。
请比较 `pom.xml` 以了解它们是如何来构建的。
此外，还有一个整个 Storm topology 的示例（`WordCount-StormTopology.jar`）。

您可以通过 `bin/flink run <jarname>.jar` 来运行这些示例。正确的入口点包含在每一个 jar 文件的 manifest 文件中。

{% top %}

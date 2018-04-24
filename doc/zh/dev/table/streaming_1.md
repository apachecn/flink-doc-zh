---
title: "Streaming Concepts"
nav-parent_id: tableapi
nav-pos: 10
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

Flink中[Table API](tableApi.html)和 [SQL support](sql.html)是批处理和流处理的统一API。这意味着无论其是有界的批量输入还是无界的流输入，Table API和SQL查询都是相同的语义。由于关系代数和SQL最初是为批处理设计的，所以关于无界流输入的关系查询并不像有界批量输入的关系查询那么容易理解。

这节，我们会解释Flink的关系型API在流数据上的概念，实际限制和特定的流配置参数。

* This will be replaced by the TOC
{:toc}

数据流上的关系查询
----------------------------------

SQL和关系代数设计时并没有考虑流数据。因此，关于关系代数（和SQL）与流处理之间的概念差距很少。

<table class="table table-bordered">
	<tr>
		<th>关系代数/SQL</th>
		<th>流处理</th>
	</tr>
	<tr>
		<td>关系（或表）是有限（多）组元组</td>
		<td>流数据是元组的无限序列</td>
	</tr>
	<tr>
		<td>在批处理数据上（例如关系数据库中的表）执行的查询可以访问完整的输入数据</td>
		<td>流查询在启动时不能访问所有数据，必须要“等待”将流入的数</td>
	</tr>
	<tr>
		<td>批查询在生成一个固定大小的结果后终止</td>
		<td>流查询基于所接收的记录一直不断更新其结果</td>
	</tr>
</table>

尽管存在这些差异，但用关系查询和SQL处理流数据并非不可能。高级关系数据库系统提供称为*Materialized Views*（物化视图）的功能。物化视图被定义为SQL查询，就像常规虚拟视图。与虚拟视图相反，物化视图会缓存查询结果，以便在访问视图时不需要评估查询。缓存的一个常见挑战是防止提供过期结果缓存。当定义查询的基表被修改后，物化视图就会过时。 *Eager View Maintenance*是一种在物化视图基表更新后即时更新物化视图的技术。

从以下方面考虑时，会发现eager view maintenance和SQL查询之间的联系更为明显：

- 数据库表是`INSERT`，`UPDATE`和`DELETE` DML语句流的结果，通常被称为*更新日志流*
- 物化视图被定义为SQL查询，为了更新视图，查询不断地处理视图基本关系的更新日志流
- 物化视图是流式SQL查询的结果

基于以上要点，我们在下节介绍Flink中*Dynamic Tables*的相关概念。

动态表 &amp; 连续查询
---------------------------------------

Dynamic Tables是Flink中Table API和SQL支持流数据的核心概念。不同于表示批数据的静态表，动态表随时间而变化。它们可以像静态批处理表一样被查询，且会生成*Continuous Query*。连续查询不会终止，且会生成一个动态表作为结果。查询会不断更新结果动态表，以反映其输入动态表上的更改。实质上，动态表上的连续查询与物化视图的定义查询非常相似。

要注意，连续查询的结果在语义上与在输入动态表的快照上以批处理模式执行相同查询的结果一致：

下图可视化了streams, dynamic tables, 和 continuous queries的关系: 

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/stream-query-stream.png" width="80%">
</center>

1. stream转化为动态表
1. 在动态表上评估连续查询，会产生新的动态表
1. 动态表的结果转换为stream

**注意：**动态表首先是一个逻辑概念，其在查询执行期间，不需要被具体化。

接下来，我们利用有以下模式流数据的click events来解释动态表和连续查询的概念：

```
[ 
  user:  VARCHAR,   // the name of the user
  cTime: TIMESTAMP, // the time when the URL was accessed
  url:   VARCHAR    // the URL that was accessed by the user
]
```

### 在流上定义表

为了用关系查询来处理流数据，需要将其转换为`Table`。概念上来说，流数据的每个记录都会在结果表反应为一个`INSERT`修改。实质上，我们正在从一个`INSERT`-only变更日志流创建表。

下图展示如何将流的click event转换为表（右）。且随着点击流记录的输入，结果表不断增长。

<center>
<img alt="Append mode" src="{{ site.baseurl }}/fig/table-streaming/append-mode.png" width="60%">
</center>

**注意：**定义为流的表在内部是不会被物化。

### 连续查询

连续查询评估动态表，会生成新的动态表。相较于批量查询，连续查询不会终止，并会随其输入表上的更新，更新其结果表。连续查询的结果在语义上始终等与在输入表的快照上以批处理模式执行相同查询的结果一致。

下面的示例，我们会展示在点击事件流上定义的`clicks` table上的两个查询示例。

第一个查询是一个简单的`GROUP-BY COUNT`聚合查询。它将字段中的`clicks` table分组到`user`，并计算访问的URLs的数量。下图显示了当`clicks` table在有附加行更新时，查询如何被评估。

<center>
<img alt="Continuous Non-Windowed Query" src="{{ site.baseurl }}/fig/table-streaming/query-groupBy-cnt.png" width="90%">
</center>

查询开始时，`clicks` table（左）是空的。当第一行插入到`clicks` table时，查询开始计算结果表。在第一行`[Mary, . /home]`插入时，结果表生成一行`[Mary,1]`。当第二行`[Bob, . /cart]`插入时，查询更新结果表并插入`[Bob,1]`新行。第三行`[Mary, . /prod?id=1]`更新已经有的计算结果，将`[Marry,1]`更新为`[Mary,2]`。最后，第四行附加到`clicks` table时，插入`[Liz,1]`到结果表第三行。

第二个查询与第一个查询相似，但除了`user`属性，还可在计算`URLs`数量（基于时间的计算，例如基于特殊[时间属性](#time-attributes)的窗口，之后会介绍）之前，基于[小时滚动窗口](./sql.html#group-windows)对`clicks` table进行分组。下图展示了不同时间点的输入和输出，以便可视化动态表的变化性质。

<center>
<img alt="Continuous Group-Window Query" src="{{ site.baseurl }}/fig/table-streaming/query-groupBy-window-cnt.png" width="100%">
</center>

同样，左侧为输入表的`clicks`。查询每小时计算一次结果并更新结果表。该`clicks` table包含4行，时间（`cTime`）在`12:00:00`到`12:59:59`之间。查询计算出2个结果行并附加到结果表。在下一个在`13:00:00`到`13:59:59`之间的时间窗口，`clicks` table有3行，输出2行到结果表。随着时间推移，有更多行附加到`clicks`时，结果表会被更新。

#### 更新和附加查询

尽管以上两个查询示例很相似（都为计算分组计数聚合），但它们在一个重要方面有所不同：

- 第一个查询会以前发布的结果，即定义结果表的更改日志流包含`INSERT`和`UPDATE`更改。
- 第二个查询仅是附加到结果表，即结果表的更改日志流只包含`INSERT`更改。

查询是会生成append-only table还是会生成更新表，有一些要求：

- 产生更新更改的查询，通常必须有更多状态（请参阅以下部分）。
- 将append-only table转换为流与更新表的转换不同（请参阅[表格到流转换](#table-to-stream-conversion)部分）。

#### 查询限制

很多（但不是全部）语义上有效的查询可以被评估为流上的连续查询。有些查询计算起来太昂贵，要么是由于需要保持状态的大小，要么是因为计算更新太昂贵。

- **状态大小：**无界流上评估连续查询，通常会运行数周或数月。因此，连续查询处理的数据总量可能非常大。需要更新已发出结果的查询需要包含所有发出的行，才能够更新它们。例如，第一个示例查询需要存储每个用户的URL计数，从而能增加计数，并在输入表接收到新行时生成新结果。如果仅注册用户被跟踪，所需包含的计数数量可能不会太高。但是，如果未注册的用户都获得分配的唯一用户名，所需包含的计数数量将随着时间增长而增加，并最终可能导致查询失败。

{% highlight sql %}
SELECT user, COUNT(url)
FROM clicks
GROUP BY user;
{% endhighlight %}

- **计算更新：**即使只添加或更新了一个简单的输入记录，一些查询也需要重新计算和更新大部分已发出的结果行。显然，这样的查询不适合作为连续查询来执行。下面例子中提到的查询，是基于最后一次的click时间为每个用户计算`RANK`。一旦`clicks` table收到新行，用户的`lastAction`会被更新，并计算新rank。由于两行不能具有相同的等级，因此所有较低等级的行也需要被更新。

{% highlight sql %}
SELECT user, RANK() OVER (ORDER BY lastLogin) 
FROM (
  SELECT user, MAX(cTime) AS lastAction FROM clicks GROUP BY user
);
{% endhighlight %}

关于控制连续查询执行的参数，查阅[QueryConfig](#query-configuration)。一些参数可以改变包含状态的大小以获得结果正确性。

### Table to Stream Conversion

类似一个常规的数据库表，动态表通过`INSERT`，`UPDATE`和`DELETE`的变化不断被修改。它可能是一个不断更新的单行表，一个没有`UPDATE`和`DELETE`变化只有`INSERT`的表，或者其中的任何情况。

当将一个动态表转为流或者将其写入外部系统，需要对这些变化进行编码。Flink的Table API和SQL支持3种编码方式：

- **Append-only stream**：仅有INSERT变化修改的动态表，可以通过发出插入的行转为流
- **Retract stream（撤回流）**：撤回流时包含 *add messages* 和*retract messages*两种类型消息的流。动态表通过编码`INSERT`变化为添加消息，编码`DELETE`变化为撤回消息，编码`UPDATE`变化为更新先前行的撤回消息和更新行的添加消息。下图可视化展示动态图到撤回流的转变。

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/undo-redo-mode.png" width="85%">
</center>
<br><br>

* **Upsert stream**：Upsert stream是包含*upsert messages* and *delete message*两种类型消息的流。转换为upsert stream的动态表需要一个唯一键（可能是复合的）。具有唯一密钥的动态表通过编码`INSERT`和`UPDATE`变化为`upsert`消息及编码DELETE变化为删除消息，进行转变。为了正确应用消息，消费者流需要知道唯一密钥的属性。与撤回流主要不同点，是`UPDATE`变化利用单个消息编码，因此效率高。下图可视化展示动态表转为upsert stream的过程。

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/redo-mode.png" width="85%">
</center>
<br><br>

关于转化动态表为`DataStream`的API，参阅[Common Concepts](./common.html#convert-a-table-into-a-datastream)。注意当转化动态表为`DataStream`时，支持append-only和撤回流操作。关于通过`TableSink`接口发出动态表到外部系统，参阅[TableSources and TableSinks](./sourceSinks.html#define-a-tablesink)。

{% top %}

时间属性
---------------

Flink可以基于不同的时间概念处理流数据。

- *Processing time*(处理时间)：是执行相应操作的机器的系统时间（也称作“wall-clock time”）
- *Event time*(事件时间)：基于附加到每行的时间戳对流数据进行处理，时间戳是在事件发生时进行编码
- *Ingestion time*(摄取时间)：是事件进入Flink的时间，在内部处理方式同event time

更多关于Flink处理时间的问题，参阅[Event Time and Watermarks]({{ site.baseurl }}/dev/event_time.html)。

表程序要求为流处理环境已指定了相应的时间特征：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime); // default

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime) // default

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
{% endhighlight %}
</div>
</div>

类似[Table API]({{ site.baseurl }}/dev/table/tableApi.html#group-windows)和[SQL]({{ site.baseurl }}/dev/table/sql.html#group-windows)中时间窗口的这种基于时间的操作，需要相关的时间概念信息及其来源信息。因此，表会提供指示时间和访问相关时间戳的*logical time attributes*逻辑时间属性。

时间属性可以归为每个表模式的一部分。当从`DataStream`中创建表或者通过`TableSource`进行预定义时，时间属性会被定义，且一旦被定义，它可以被作为一个字段进行引用，也可以在基于时间操作中使用。

只要时间属性没有被修改且只是简单地从查询的一部分转发到另一部分，它就仍然是有效的。时间属性如同常规时间戳，可以进行计算访问。如果时间属性被用于计算，它会被物化成为常规时间戳。但常规时间戳不与Flink的时间和水印相配合，所以也不能用于基于时间的操作。

### 处理时间

处理时间允许表程序基于本地时间生成结果，是最简单的时间概念，但不提供决策，且不需要提取时间戳或生成水印。

有两种定义处理时间属性的方法。

#### 在DataStream转为Table的过程

处理时间属性是在定义模式期间用 `.proctime`属性定义的。时间属性只能通过附加的逻辑字段来扩展物理模式，因此，它只能在模式定义的最后被定义。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, String>> stream = ...;

// declare an additional logical field as a processing time attribute
Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.proctime");

WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[(String, String)] = ...

// declare an additional logical field as a processing time attribute
val table = tEnv.fromDataStream(stream, 'UserActionTimestamp, 'Username, 'Data, 'UserActionTime.proctime)

val windowedTable = table.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

#### 使用TableSource

处理时间属性由使用`DefinedProctimeAttribute`接口的`TableSource`进行定义。逻辑时间属性被附加到通过`TableSource`的返回类型定义的物理模式上。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// define a table source with a processing attribute
public class UserActionSource implements StreamTableSource<Row>, DefinedProctimeAttribute {

	@Override
	public TypeInformation<Row> getReturnType() {
		String[] names = new String[] {"Username" , "Data"};
		TypeInformation[] types = new TypeInformation[] {Types.STRING(), Types.STRING()};
		return Types.ROW(names, types);
	}
	
	@Override
	public DataStream<Row> getDataStream(StreamExecutionEnvironment execEnv) {
		// create stream 
		DataStream<Row> stream = ...;
		return stream;
	}
	
	@Override
	public String getProctimeAttribute() {
		// field with this name will be appended as a third field 
		return "UserActionTime";
	}
}

// register table source
tEnv.registerTableSource("UserActions", new UserActionSource());

WindowedTable windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// define a table source with a processing attribute
class UserActionSource extends StreamTableSource[Row] with DefinedProctimeAttribute {

	override def getReturnType = {
		val names = Array[String]("Username" , "Data")
		val types = Array[TypeInformation[_]](Types.STRING, Types.STRING)
		Types.ROW(names, types)
	}
	
	override def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[Row] = {
		// create stream
		val stream = ...
		stream
	}
	
	override def getProctimeAttribute = {
		// field with this name will be appended as a third field 
		"UserActionTime"
	}
}

// register table source
tEnv.registerTableSource("UserActions", new UserActionSource)

val windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

### 事件时间

事件时间允许表程序基于每条记录中包含的时间生成结果。即使在无序事件或晚期事件下，也可以获得一致结果。它也可确保在从持久存储中读取记录时，能重复使用表程序的结果。

此外，事件时间允许在批处理和流处理环境中表程序有统一的语法，即在流处理环境中的时间属性也可是批处理环境中记录的常规字段。

为了解决无序事件和在流中区分实时和晚期事件，Flink需要提取事件的时间戳并做一定处理（即为[watermarks]({{ site.baseurl }}/dev/event_time.html)）。

事件时间属性可以通过在DataStream转为表的过程或使用TableSource进行定义。

#### 在DataStream转为Table的过程

事件时间属性可在模式定义期间通过 `.rowtiem`属性定义。[时间戳和水印]({{ site.baseurl }}/dev/event_time.html)必须已经分配到已转换的`DataStream`中。

在`DataStream`转为`Table`的过程中，有两种定义表属性的方式，这取决于指定的 `.rowtime`字段名是否存在于`DataStream`模式，而时间戳字段也是如此。

- 作为新字段添加到模式
- 替换现有模式

无论哪种情况，事件时间的时间戳字段都会保存`DataStream`事件时间的时间戳的值。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

// Option 1:

// extract timestamp and assign watermarks based on knowledge of the stream
DataStream<Tuple2<String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);

// declare an additional logical field as an event time attribute
Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.rowtime");


// Option 2:

// extract timestamp from first field, and assign watermarks based on knowledge of the stream
DataStream<Tuple3<Long, String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);

// the first field has been used for timestamp extraction, and is no longer necessary
// replace first field with a logical event time attribute
Table table = tEnv.fromDataStream(stream, "UserActionTime.rowtime, Username, Data");

// Usage:

WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

// Option 1:

// extract timestamp and assign watermarks based on knowledge of the stream
val stream: DataStream[(String, String)] = inputStream.assignTimestampsAndWatermarks(...)

// declare an additional logical field as an event time attribute
val table = tEnv.fromDataStream(stream, 'Username, 'Data, 'UserActionTime.rowtime)


// Option 2:

// extract timestamp from first field, and assign watermarks based on knowledge of the stream
val stream: DataStream[(Long, String, String)] = inputStream.assignTimestampsAndWatermarks(...)

// the first field has been used for timestamp extraction, and is no longer necessary
// replace first field with a logical event time attribute
val table = tEnv.fromDataStream(stream, 'UserActionTime.rowtime, 'Username, 'Data)

// Usage:

val windowedTable = table.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

#### 使用TableSource

事件时间通过使用`DefineRowtimeAttribute`接口的`TableSource`定义。`getRowtimeAttribute()`方法返回一个携带了表的事件时间属性的已存字段的名称，且是`LONG`或`TIMESTAMP`类型。

此外，通过`getDataStream()`返回的`DataStream`必须有与定义时间属性相匹配的水印。注意`DataStream`的时间戳（被`TimestampAssigner`分配的）被忽略。只有`TableSource`属性的值是相关的。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// define a table source with a rowtime attribute
public class UserActionSource implements StreamTableSource<Row>, DefinedRowtimeAttribute {

	@Override
	public TypeInformation<Row> getReturnType() {
		String[] names = new String[] {"Username", "Data", "UserActionTime"};
		TypeInformation[] types = 
		    new TypeInformation[] {Types.STRING(), Types.STRING(), Types.LONG()};
		return Types.ROW(names, types);
	}
	
	@Override
	public DataStream<Row> getDataStream(StreamExecutionEnvironment execEnv) {
		// create stream 
		// ...
		// assign watermarks based on the "UserActionTime" attribute
		DataStream<Row> stream = inputStream.assignTimestampsAndWatermarks(...);
		return stream;
	}
	
	@Override
	public String getRowtimeAttribute() {
		// Mark the "UserActionTime" attribute as event-time attribute.
		return "UserActionTime";
	}
}

// register the table source
tEnv.registerTableSource("UserActions", new UserActionSource());

WindowedTable windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// define a table source with a rowtime attribute
class UserActionSource extends StreamTableSource[Row] with DefinedRowtimeAttribute {

	override def getReturnType = {
		val names = Array[String]("Username" , "Data", "UserActionTime")
		val types = Array[TypeInformation[_]](Types.STRING, Types.STRING, Types.LONG)
		Types.ROW(names, types)
	}
	
	override def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[Row] = {
		// create stream 
		// ...
		// assign watermarks based on the "UserActionTime" attribute
		val stream = inputStream.assignTimestampsAndWatermarks(...)
		stream
	}
	
	override def getRowtimeAttribute = {
		// Mark the "UserActionTime" attribute as event-time attribute.
		"UserActionTime"
	}
}

// register the table source
tEnv.registerTableSource("UserActions", new UserActionSource)

val windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

{% top %}

查询配置
-------------------

无论是有界批输入还是无界流输入，Table API和SQL查询有相同的语义。在很多情况，对流输入的连续查询可以计算出与离线计算结果相同的准确结果。但是，这在一般情况下是不可能的，因为连续查询必须要限制他们包含状态的大小，以避免用尽存储并能长时间处理流数据。因此，根据输入数据和查询本身的特征，连续查询仅能提供近似结果。

Flink的Table API和SQL 接口提供用来调整连续查询的准确性和资源消耗的参数，它们通过一个`QueryConfig`对象指定。`QueryConfig`可以从`TableEnvironment`中获取，且当`Table`被转化时，被返回，即当它 [transformed into a DataStream](common.html#convert-a-table-into-a-datastream-or-dataset)或[emitted via a TableSink](common.html#emit-a-table)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// obtain query configuration from TableEnvironment
StreamQueryConfig qConfig = tableEnv.queryConfig();
// set query parameters
qConfig.withIdleStateRetentionTime(Time.hours(12));

// define query
Table result = ...

// create TableSink
TableSink<Row> sink = ...

// emit result Table via a TableSink
result.writeToSink(sink, qConfig);

// convert result Table into a DataStream<Row>
DataStream<Row> stream = tableEnv.toAppendStream(result, Row.class, qConfig);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// obtain query configuration from TableEnvironment
val qConfig: StreamQueryConfig = tableEnv.queryConfig
// set query parameters
qConfig.withIdleStateRetentionTime(Time.hours(12))

// define query
val result: Table = ???

// create TableSink
val sink: TableSink[Row] = ???

// emit result Table via a TableSink
result.writeToSink(sink, qConfig)

// convert result Table into a DataStream[Row]
val stream: DataStream[Row] = result.toAppendStream[Row](qConfig)

{% endhighlight %}
</div>
</div>

下面，我们会介绍`QueryConfig`的参数，及说明它们是如何影响查询的准确性和资源损耗

### 空闲状态保留时间

很多查询通过一个或多个关键属性进行聚合或连接记录。当在流上执行查询时，连续查询需要手机记录或保存每个key的部分结果。如果输入流的关键域在演变，即活动键的值在随时间变化，则连续查询累积越来越多的状态，且越来越多的不同的键被观察到。然而，一段时间后，键会转为非活动状态，且它们的相应状态会变得陈旧无用。

例如，下面的查询计算出每个会话的点击次数：

```
SELECT sessionId, COUNT(*) FROM clicks GROUP BY sessionId;
```

其中，`sessionID`属性是分组键，且连续查询包含所有它所观察的`sessionID`的计数。`sessionID`属性随时间变化，且直到`session`结束，`sessionID`值失效，即在有限时间内，其属性变化。然而，连续查询无法直到`sessionID`属性并预计每个`sessionID`值在任意时间都可发生。它保存每个观测的`sessionID`值。因此，随着越来越多的`sessionID`值被观察，查询的状态大小不断增长。

*Idle State Retention Time*（空闲状态保留时间）参数定义了在键被移除之前未被更新时，键的状态被保留的时长。在先前的查询示例中，一旦`sessionID`在配置时间段内未被更新，它的计数会被移除。

在移除键的状态后，连续查询就完全忘记处理过该键。如果处理包含已移除键的记录，该记录会被认为是相应键的第一条记录。这就意味着`sessionID`的计数会从`0`开始。

有两个配置空闲状态保留时间的参数：

- minimum idle state retention（最小空闲状态保留时间）：非活动键的状态移除前保留的最短时长
- maximum idle state retention（最大空闲状态保留时间）：非活动键的状态移除前保留的最长时长

参数指定如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

StreamQueryConfig qConfig = ...

// set idle state retention time: min = 12 hour, max = 16 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(16));
// set idle state retention time. min = max = 12 hours
qConfig.withIdleStateRetentionTime(Time.hours(12);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val qConfig: StreamQueryConfig = ???

// set idle state retention time: min = 12 hour, max = 16 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(16))
// set idle state retention time. min = max = 12 hours
qConfig.withIdleStateRetentionTime(Time.hours(12)

{% endhighlight %}
</div>
</div>

配置不同的最小和最大空闲状态保留时间效率更高，因为它减少了从内部簿记，查询何时删除状态。

{% top %}



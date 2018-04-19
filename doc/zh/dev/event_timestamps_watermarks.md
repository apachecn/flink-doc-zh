---
title: "Generating Timestamps / Watermarks"
nav-parent_id: event_time
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

* toc
{:toc}


This section is relevant for programs running on **event time**. For an introduction to *event time*,
*processing time*, and *ingestion time*, please refer to the [introduction to event time]({{ site.baseurl }}/dev/event_time.html).

To work with *event time*, streaming programs need to set the *time characteristic* accordingly.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
{% endhighlight %}
</div>
</div>

## 分配时间戳

为了处理 *事件时间*，Flink需要知道事件的 *时间戳*，意味着流需要有事件时间戳的 *分配* 。 
这通常通过访问/解压来自元素中某个字段的时间戳。

时间戳分配与水印生成结合在一起，告诉系统，事件时间的进度。

有两种方法分配时间戳和生成水印：

  1. 直接在数据流的源中
  2. 通过时间戳分配器/水印生成器：在Flink时间戳分配器中还定义要水印的发射

<span class="label label-danger">Attention</span> Both timestamps and watermarks are specified as
milliseconds since the Java epoch of 1970-01-01T00:00:00Z.

### 具有时间戳和水印的源函数

数据流源也可以直接为其生成的元素分配时间戳，并且发射水印。
当其完成后，不需要时间戳分配器。
请注意，如果使用时间戳分配器，则数据源提供的任何时间戳和水印将被覆盖。

要将时间戳直接分配给源中的元素，则该源必须使用 `collectWithTimestamp(...)`方法在 `SourceContext`上。
要生成水印，源代码必须调用 `emitWatermark(Watermark)` 函数。

下面是一个*(non-checkpointed)* 源分配时间戳并生成水印的简单示例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
override def run(ctx: SourceContext[MyType]): Unit = {
	while (/* condition */) {
		val next: MyType = getNext()
		ctx.collectWithTimestamp(next, next.eventTimestamp)

		if (next.hasWatermarkTime) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime))
		}
	}
}
{% endhighlight %}
</div>
</div>


### 时间戳分配器/水印生成器

时间戳分配器接收流并生成具有时间戳元素和水印的新流。 
如果原始流已经有时间戳和水印，时间戳分配器将覆盖它们。

时间戳分配器通常在数据源之后立即指定，但并不严格要求这样做。
例如，常见模式是在时间戳分配器之前解析(*MapFunction*)和过滤 (*FilterFunction*) 。
在任何情况下，时间戳分配器都需要在事件时间的第一个算子（例如第一个窗口操作）之前指定。
特殊情况下，当使用Kafka作为流任务的数据源时，Flink允许在源（或消费者）本身指定时间戳分配器/水印发射器。
有关如何执行此操作的更多信息，请参阅[Kafka Connector documentation]({{ site.baseurl }}/dev/connectors/kafka.html).


**注意：**本节的其余部分介绍程序必须实现的主要接口，以创建自己的时间戳提取器/水印发射器。
要查看Flink已经实现的提取器，请参阅
[Pre-defined Timestamp Extractors / Watermark Emitters]({{ site.baseurl }}/dev/event_timestamp_extractors.html) 。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

DataStream<MyEvent> withTimestampsAndWatermarks = stream
        .filter( event -> event.severity() == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks());

withTimestampsAndWatermarks
        .keyBy( (event) -> event.getGroup() )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) -> a.add(b) )
        .addSink(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream: DataStream[MyEvent] = env.readFile(
         myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
         FilePathFilter.createDefaultFilter())

val withTimestampsAndWatermarks: DataStream[MyEvent] = stream
        .filter( _.severity == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks())

withTimestampsAndWatermarks
        .keyBy( _.getGroup )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) => a.add(b) )
        .addSink(...)
{% endhighlight %}
</div>
</div>


#### **定期水印**

`AssignerWithPeriodicWatermarks` 分配时间戳并定期生成水印（可能取决于流元素，或纯粹基于处理时间）。

通过`ExecutionConfig.setAutoWatermarkInterval(...)`定义生成水印间隔(every *n* milliseconds)。
每次分配器会调用`getCurrentWatermark()`方法，如果返回的水印非空且大于先前的水印，则将发射新的水印。

下面是两个带有周期性水印生成的时间戳分配器的简单例子。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
/**
 * This generator generates watermarks assuming that elements arrive out of order,
 * but only to a certain degree. The latest elements for a certain timestamp t will arrive
 * at most n milliseconds after the earliest elements for timestamp t.
 */
public class BoundedOutOfOrdernessGenerator extends AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

/**
 * This generator generates watermarks that are lagging behind processing time by a fixed amount.
 * It assumes that elements arrive in Flink after a bounded delay.
 */
public class TimeLagWatermarkGenerator extends AssignerWithPeriodicWatermarks<MyEvent> {

	private final long maxTimeLag = 5000; // 5 seconds

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark getCurrentWatermark() {
		// return the watermark as current time minus the maximum time lag
		return new Watermark(System.currentTimeMillis() - maxTimeLag);
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
/**
 * This generator generates watermarks assuming that elements arrive out of order,
 * but only to a certain degree. The latest elements for a certain timestamp t will arrive
 * at most n milliseconds after the earliest elements for timestamp t.
 */
class BoundedOutOfOrdernessGenerator extends AssignerWithPeriodicWatermarks[MyEvent] {

    val maxOutOfOrderness = 3500L // 3.5 seconds

    var currentMaxTimestamp: Long

    override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
        val timestamp = element.getCreationTime()
        currentMaxTimestamp = max(timestamp, currentMaxTimestamp)
        timestamp
    }

    override def getCurrentWatermark(): Watermark = {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        new Watermark(currentMaxTimestamp - maxOutOfOrderness)
    }
}

/**
 * This generator generates watermarks that are lagging behind processing time by a fixed amount.
 * It assumes that elements arrive in Flink after a bounded delay.
 */
class TimeLagWatermarkGenerator extends AssignerWithPeriodicWatermarks[MyEvent] {

    val maxTimeLag = 5000L // 5 seconds

    override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
        element.getCreationTime
    }

    override def getCurrentWatermark(): Watermark = {
        // return the watermark as current time minus the maximum time lag
        new Watermark(System.currentTimeMillis() - maxTimeLag)
    }
}
{% endhighlight %}
</div>
</div>

#### **中断水印**

每当某个事件指示生成水印时，中断它并可能生成新水印，请使用`AssignerWithPunctuatedWatermarks`。
对于这个类，Flink将首先调用`extractTimestamp(...)` 方法为元素指定一个时间戳，
然后立即调用该元素上的`checkAndGetNextWatermark(...)` 方法。

`checkAndGetNextWatermark(...)` 方法传递在`extractTimestamp(...)`方法中分配的时间戳，并可以决定是否要生成水印。
每当`checkAndGetNextWatermark(...)`方法返回一个非null水印，并且该水印大于最新的先前水印时，该新水印将被发射。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks[MyEvent] {

	override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
		element.getCreationTime
	}

	override def checkAndGetNextWatermark(lastElement: MyEvent, extractedTimestamp: Long): Watermark = {
		if (lastElement.hasWatermarkMarker()) new Watermark(extractedTimestamp) else null
	}
}
{% endhighlight %}
</div>
</div>

*注意：*可以在每个事件上生成水印。但是，由于每个水印都会导致一些下游进行计算，因此过多的水印会降低性能。


## 卡夫卡分区时间戳

当使用 [Apache Kafka](connectors/kafka.html) 作为数据源时，每个Kafka分区可能有一个简单的事件时间模式（时间戳升序或无序）。
但是，当消费Kafka的数据流时，多个分区通常会并行使用，交错多个分区之间的事件并摧毁每个分区的模式（这与Kafka消费者客户端的工作方式有关）。

在这种情况下，您可以使用Flink的Kafka-partition-aware水印生成器。
使用该特性，Kafka消费者可以在每个Kafka分区内生成水印，
每个分区的水印与在流洗牌(shuffled)上合并水印相同的方式进行合并。

例如，如果事件时间戳严格按照Kafka分区升序排列，
使用 [ascending timestamps watermark generator](event_timestamp_extractors.html#assigners-with-ascending-timestamps) （升序时间戳水印生成器）生成每个分区的水印，整体水印将会很完美。

下面的插图显示了每个Kafka分区如何使用水印生成器以及在这种情况下水印如何通过数据流传播。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
FlinkKafkaConsumer09<MyType> kafkaSource = new FlinkKafkaConsumer09<>("myTopic", schema, props);
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyType>() {

    @Override
    public long extractAscendingTimestamp(MyType element) {
        return element.eventTimestamp();
    }
});

DataStream<MyType> stream = env.addSource(kafkaSource);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val kafkaSource = new FlinkKafkaConsumer09[MyType]("myTopic", schema, props)
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor[MyType] {
    def extractAscendingTimestamp(element: MyType): Long = element.eventTimestamp
})

val stream: DataStream[MyType] = env.addSource(kafkaSource)
{% endhighlight %}
</div>
</div>

<img src="{{ site.baseurl }}/fig/parallel_kafka_watermarks.svg" alt="Generating Watermarks with awareness for Kafka-partitions" class="center" width="80%" />

{% top %}

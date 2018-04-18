---
title: "Event Time"
nav-id: event_time
nav-show_overview: true
nav-parent_id: streaming
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

* toc
{:toc}

# 事件时间 / 处理时间 / 摄取时间

Flink在流程序中支持不同的*时间*概念。

- **处理时间:** 处理时间是指执行算子时的机器系统时间。

    当流程序以处理时间运行时, 所有基于时间的算子 (比如 time windows) 将使用运行该算子的机器系统时钟。 
    例如，一个小时的处理时间窗口将包含在系统时钟显示整整一小时中到达指定算子的所有记录

    处理时间是最简单的时间概念，不需要数据流和机器之间的协调。 它提供了最佳的性能和最低的延迟。
    然而，在分布式和异步环境中，处理时间并不能提供确定性，
    因为它容易受到记录到达系统的速度（例如来自消息队列），以及记录在系统内部算子之间的流动速度影响。

- **事件时间:** 事件时间是每个事件在其生产设备上发生的时间。
    这段时间通常在进入Flink之前会被嵌入到记录中,可以从这个记录中提取*时间戳*。
    一个小时的事件时间窗口将包含落入这个小时的所有记录，无论记录是何时到达以及以何种顺序到达。

    事件时间即使在事件无序，事件迟到或重放来自备份,持久性日志时也能提供正确的结果。
    在事件时间内，时间进度取决于数据，而不在任何时钟上。
    事件时间程序必须指定如何生成*事件时间水印*，这是事件时间的进度信号机制。 
    机制如下面所描述。

    由于事件延迟会等待一段时间和无序事件的存在，事件时间处理通常会产生一定的延迟。因此，
    事件时间程序通常与*处理时间*操作相结合。

- **摄取时间:** 摄取时间是事件进入Flink的时间。在源算子中每个记录将该源的当前时间作为时间戳，基于时间的操作 (比如 time windows)则引用该时间戳。

     *摄取时间* 从概念上介于*事件时间* 和 *处理时间*之间。
    比较*处理时间*，它的代价会略高，但是会提供更多预期的结果。
    由于*摄取时间*使用稳定的时间戳（在数据源处分配），对记录的不同窗口操作将引用相同的时间戳，
    而在处理时间内，每个窗口算子可以将该记录分配给不同的窗口（基于本地系统时间和任何传输延时）。

    与*事件时间*相比，基于*摄入时间*的程序无法处理任何乱序事件或延迟数据，但是程序无需指定如何生成水印。

    从内部看来，摄取时间与事件时间非常相似，但是它具有自动时间戳分配和自动生成水印的功能。

<img src="{{ site.baseurl }}/fig/times_clocks.svg" class="center" width="80%" />


### 设置时间特征

Flink DataStream程序首先会设置*时间特征*。
这个设定定义了数据源的行为方式(例如, 是否分配时间戳), 以及时间应该被窗口算子使用的概念就像 `KeyedStream.timeWindow(Time.seconds(30))`.

以下例子展示了一个Flink程序，用于聚合每小时时间窗口中的事件。窗口的行为适应了时间特征。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream: DataStream[MyEvent] = env.addSource(new FlinkKafkaConsumer09[MyEvent](topic, schema, props))

stream
    .keyBy( _.getUser )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) => a.add(b) )
    .addSink(...)
{% endhighlight %}
</div>
</div>


注意，为了在*事件时间*中运行此示例，该程序需要使用直接定义事件时间，并自己发出水印的数据源，
或者程序必须在数据源接入后注入*时间戳分配 & 水印生成*。
这些功能描述了如何访问事件时间戳，以及事件流显示的乱序程度。


以下部分描述*时间戳*和*水印*背后的一般机制。
关于如何在Flink DataStream API中使用时间戳分配和水印生成，请参阅
[Generating Timestamps / Watermarks]({{ site.baseurl }}/dev/event_timestamps_watermarks.html).


# 事件时间和水印

*注意：Flink从数据流模型中实现了许多技术。 要详细了解事件时间和水印，请查看下面的文章。*

  - [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) by Tyler Akidau
  - The [Dataflow Model paper](https://research.google.com/pubs/archive/43864.pdf)


为了支持*事件时间*的流处理器需要一种方法来衡量事件时间的进度。
例如，当事件时间超过一个小时的结束时，需要通知建立每小时窗口的窗口算子。以便算子可以在程序中关闭该窗口。

*事件时间*可以独立于*处理时间* (通过挂钟测量)进行。
例如，在一个程序中，算子当前的*事件时间*可能稍微落后于*处理时间*（考虑到事件接收的延迟），
而两者都以相同的速度进行。
另一方面，在另一个流程序通过快速转发已经缓存在Kafka主题（或者别的消息队列）中的历史数据，
仅需几秒就可处理可能持续几周的事件时间。

------

Flink中测量事件时间进度的机制是**水印(watermarks)**。
水印作为数据流的一部分流动并带有时间戳*t*。
*Watermark(t)* 声明了事件时间到达流中的时间 *t*，这意味着流中不应该带有时间戳 *t' <= t* 的元素
（即事件的时间戳早于或等于水印的时间戳）。

下图显示了具有（逻辑）时间戳和内嵌水印的事件流。
在这个例子中，事件是经过排序的（按照它们的时间戳），
这意味着水印只是流中简单的周期性标记。


<img src="{{ site.baseurl }}/fig/stream_watermark_in_order.svg" alt="A data stream with events (in order) and watermarks" class="center" width="65%" />

水印对*无序*流是至关重要的，如下所示，其事件不按时间戳排序。
一般来说，水印是流中某个点上的一种声明，所有达到这个点上时间戳的事件都应该到达。
一旦水印抵达算子，算子可以将其内部*事件时间时钟*提前到水印的值。

<img src="{{ site.baseurl }}/fig/stream_watermark_out_of_order.svg" alt="A data stream with events (out of order) and watermarks" class="center" width="65%" />


## 并行流中的水印

Watermarks are generated at, or directly after, source functions. Each parallel subtask of a source function usually
generates its watermarks independently. These watermarks define the event time at that particular parallel source.

As the watermarks flow through the streaming program, they advance the event time at the operators where they arrive. Whenever an
operator advances its event time, it generates a new watermark downstream for its successor operators.

Some operators consume multiple input streams; a union, for example, or operators following a *keyBy(...)* or *partition(...)* function.
Such an operator's current event time is the minimum of its input streams' event times. As its input streams
update their event times, so does the operator.

The figure below shows an example of events and watermarks flowing through parallel streams, and operators tracking event time.

<img src="{{ site.baseurl }}/fig/parallel_streams_watermarks.svg" alt="Parallel data streams and operators with events and watermarks" class="center" width="80%" />


## Late Elements

It is possible that certain elements will violate the watermark condition, meaning that even after the *Watermark(t)* has occurred,
more elements with timestamp *t' <= t* will occur. In fact, in many real world setups, certain elements can be arbitrarily
delayed, making it impossible to specify a time by which all elements of a certain event timestamp will have occurred.
Furthermore, even if the lateness can be bounded, delaying the watermarks by too much is often not desirable, because it
causes too much delay in the evaluation of the event time windows.

For this reason, streaming programs may explicitly expect some *late* elements. Late elements are elements that
arrive after the system's event time clock (as signaled by the watermarks) has already passed the time of the late element's
timestamp. See [Allowed Lateness]({{ site.baseurl }}/dev/stream/operators/windows.html#allowed-lateness) for more information on how to work
with late elements in event time windows.


## Debugging Watermarks

Please refer to the [Debugging Windows & Event Time]({{ site.baseurl }}/monitoring/debugging_event_time.html) section for debugging
watermarks at runtime.

{% top %}

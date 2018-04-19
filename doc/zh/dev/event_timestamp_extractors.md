---
title: "Pre-defined Timestamp Extractors / Watermark Emitters"
nav-parent_id: event_time
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

正如 [timestamps and watermark handling]({{ site.baseurl }}/dev/event_timestamps_watermarks.html),
Flink提供的抽象方法允许程序员分配自己的时间戳并发出自己的水印。
更具体地说，可以通过实现 `AssignerWithPeriodicWatermarks`或者 `AssignerWithPunctuatedWatermarks`接口来实现，取决于具体案例。
简而言之，第一个将定期发出水印，而第二个则基于传入记录的某些属性，例如， 每当在流中遇到特殊元素时。

为了进一步简化这些任务的编程工作，Flink提供了一些预先实现的时间戳分配器。 
本节提供了它们的列表。 
除了它们的开箱即用功能之外，它们的实现可以作为自定义实现的示例。

### **Assigners with ascending timestamps**

对于 *周期性* 水印生成最简单的特殊情况是，给定的源任务看到的时间戳是按升序发生。
在这种情况下，当前时间戳总是可以充当水印，因为不会有更早的时间戳到达。

请注意，每个并行数据源任务的时间戳升序都是必要的。
例如，如果在特定设置中一个Kafka分区被一个并行数据源实例读取，那么只需要每个Kafka分区内的时间戳都是上升的。 
每当并行流被洗牌（shuffled），联合（unioned），连接（connected）或合并（merged）时，
Flink的水印合并机制将生成正确的水印。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyEvent>() {

        @Override
        public long extractAscendingTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignAscendingTimestamps( _.getCreationTime )
{% endhighlight %}
</div>
</div>

### **Assigners allowing a fixed amount of lateness**

周期性水印生成的另一个例子是水印滞后于在流中出现的最大（事件时间）时间戳的一段固定时间。
这种情况涵盖预先知道流中可能会遇到最大延迟的场景，
例如，创建包含时间戳的元素的自定义源时，在固定的一段时间内进行测试。
对于这些情况，Flink提供了`BoundedOutOfOrdernessTimestampExtractor`抽象类以`maxOutOfOrderness`作为参数，
即在计算给定窗口的最终结果时元素被忽略之前允许延迟的最大时间量。
延迟(lateness)对应与 `t - t_w` 的结果，其中 `t` 是元素的（事件时间）时间戳，`t_w` 是前一个水印。
如果 `lateness > 0` ，那么该元素被认为是延迟的，并且在计算其相应窗口的任务结果时默认被忽略。
查看关于延迟元素的文档 [allowed lateness]({{ site.baseurl }}/dev/stream/operators/windows.html#allowed-lateness)

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<MyEvent>(Time.seconds(10)) {

        @Override
        public long extractTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[MyEvent](Time.seconds(10))( _.getCreationTime ))
{% endhighlight %}
</div>
</div>

{% top %}

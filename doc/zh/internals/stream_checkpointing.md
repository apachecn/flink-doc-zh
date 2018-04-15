---

title:  "Data Streaming 容错"
nav-title: Fault Tolerance for Data Streaming
nav-parent_id: internals
nav-pos: 3

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

该文档描述了Flink的流式数据流的容错机制。

* This will be replaced by the TOC
{:toc}


## 介绍

Apache Flink提供了一个容错机制来统一恢复流失应用的状态。
这种机制保证了当故障发生时，data stream的每一行记录只会作用于状态一次**exactly once**。注意，flink提供了一个开关可以将作用一次*降级*成*至少一次*


这种容错机制为流失数据流连续不断的生成快照。对于状态很小（占用空间，时间等）的流式应用，这些快照需要非常的轻量,并且可以频繁生成，不会对程序有太大的性能影响。这些流式应用的状态被存储在某个可配置的地方（比如主节点或者hdfs)。

在程序失败的情况下(由于机器、网络、软件等原因),flink停止分布式流式数据。而后系统会重启所有operator，并将状态重置到最后一次成功的检查点。输入流任务被重置到该快照对应的检查点状态。在重启过程中任何并行处理的流数据任务中的数据，都保证不是上一个检查点数据的一部分。

*Note:* 默认情况下，检查点是不开启的。查看 [Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html) 来详细了解如何使用检查点配置。

*Note:* 为了完全保证这种机制实现，源数据流（例如消息队列）需要实现倒退到不久前声明的点。 [Apache Kafka](http://kafka.apache.org) 有这个功能，Flink的kafka connector 使用了这个功能。看[Fault Tolerance Guarantees of Data Sources and Sinks]({{ site.baseurl }}/dev/connectors/guarantees.html) 来了解更多信息，关于Flink's connectors 是如何提供保证的。

* Note:* 因为flink的检查点是通过分布式快照实现的，我们会交换使用*snapshot(快照)*和*checkpoint(检查点)* 。


## Checkpointing

Flink容错机制的核心部分是绘制持续的快照和分布式流数据的操作状态。这些快照充当统一的检查点，当发生故障的时候可以根据检查点回退。Flink的快照机制论文在这里"[Lightweight Asynchronous Snapshots for Distributed Dataflows](http://arxiv.org/abs/1506.08603)".它的灵感来自于分布式快照标准[Chandy-Lamport algorithm](http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf),checkpointing是为flink的执行模式特别订制的。



### Barriers

flink分布式快照的一个核心元素是*stream barriers*.这些barriers被注入到流式数据的每一行任务处理中。Barriers绝对不会赶超数据，数据流严格有序。一个barrier会拆分数据,一部分数据被放到当前正则处理的快照中，另一部分在下一个快照中。每一个快照都有对应的快照id，以标记在这个Barrier之前的数据要存放到哪个快照中。Barriers不会干扰流失数据任务，Barriers非常轻量。同一时间内，可以有多个来自不同快照的Barriers，这意味着可能有多个快照并发生成。

<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/stream_barriers.svg" alt="Checkpoint barriers in data streams" style="width:60%; padding-top:10px; padding-bottom:10px;" />
</div>

流式Barriers在数据源端被注入到平行的流任务中。当快照的barriers *n*被注入到流数据的任务中(我们称呼为<i>S<sub>n</sub></i>)。举例，在Apache kafka，这个位置可以是在对应partition中最近一次消费记录的offset(位置偏移量)。这个位置<i>S<sub>n</sub></i>被报告给了*checkpoint coordinator* (Flink's JobManager).

Barriers会继续流向下游。当一个中间的operator接受到所有来自它上流的快照*n*的barrier时，它会发出一个表示快照 *n* 的barriers给它的下流。一旦一个sink operator(流任务DAG图最后一位同学)接受到所有来自它上游的barrier *n* ，它会向heckpoint coordinator告知快照 *n* 已完成。当所有sink都告知了一个快照的时候，这个快照被标记为完成。

一旦快照*n*完成后，这个任务就再也不会去访问<i>S<sub>n</sub></i>之前的数据源，自这个点之后的记录(他们的后代记录) 都将会通过整个数据流拓扑。

<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/stream_aligning.svg" alt="Aligning data streams at operators with multiple inputs" style="width:100%; padding-top:10px; padding-bottom:10px;" />
</div>

当Operators接收超过一个输入流的时候,需要基于输入流快照的barriers对齐*align*.如上图表所示:

 - 当operator接收到一个来自输入流的快照barrier *n* ,它就不能继续处理这个输入流后续的数据，直到operator接收到其他数据流的barrier *n*.这可以避免快照*n*的数据和快照*n+1*的数据搞混。
 - 报告了Barrier *n* 的数据流暂时被搁置到一遍。这些数据流里接受的数据没有被处理，而是放到了一个输入buffer中。
 - 当从最后一个流中拿到barrier *n*是，这个operator发送出所有计划给下游处理的数据，然后发送snapshot *n* barriers自己,完成barriers的对齐
 - 在这之后,它会恢复所有输入流中处理的数据，输入缓存中的数据会在数据流其他数据前处理。

### 状态

当operators包含任意形式的*state*（状态）,这状态必需是包含在snapshots中，作为snapshots的一部分。operator 有以下形式的状态:

  - *User-defined state*: 这种状态是直接由转换函数(transformation functions，例如`map()`或者`filter()`等)创建或修改的。详情查看[Streaming Applications中的状态]({{ site.baseurl }}/dev/stream/state/index.html)

  - *System state*: 这种状态适用于operator计算缓存中的那部分数据。这种状态的一个典型例子窗口缓存(*window buffers*),系统为窗口聚合和收集数剧直到窗口计算完成和释放。


当Operators在收到了所有输入流的barriers这个点的时候，并且在他们Operatirs发送barriers给输出流之前，这个快照状态的时候。在此时所有记录的状态被更新完成，没有更新需要依赖于这个barrier之后被应用的数据。由于快照的状态可能非常大,所以快照可以配置存储*[state backend]({{ site.baseurl }}/ops/state/state_backends.html)* 默认情况下，快村存储在JobManager的内存中，但是生产环节下需要配置高可靠的分布式存储(比如hdfs).当状态被存储之后，当状态被存储之后，operatior会确认checkpoint，发送快照barrier给后续处理的输出流。

生成的快照现在包含了:
  - 对于每个并行处理的数据流，当快照开始时的位置偏移量
  - 对于每个operator, 一个指向状态的指针，作为快照的一部分存储起来。

<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/checkpointing.svg" alt="Illustration of the Checkpointing Mechanism" style="width:100%; padding-top:10px; padding-bottom:10px;" />
</div>


### 恰好一次(Exactly Once) VS 至少一次(At Least Once)

多个输入流的对齐操作可能会导致流程序延迟。通常情况下，这种额外的延迟大约在几毫秒内完成，但是也能看到一些额外的特例明显超过了这个时间。对于所有数据需要都是超低延迟（几毫秒）的应用，flink提供了一个开关来跳过checkpoint期间的流对齐操作。关闭对齐操作后，当一个operator收到每个检查点输入barrier的时候，就立即生成检查点快照。


当对齐操作被跳过，一个operator依然会处理所有的输入，尽管部分检查点的barriers 在对应的checkpoint *n* 之后，在这种情况下，operator 会继续处理属于checkpoint *n+1* 的数据,这个时候checkpoint *n* 可能还没有生成完成快照状态。当状态恢复的时候，就会出现一部分数据因为重复而冲突，因为这些数据存储在checkpoint *n* 中，这些数据会被当成checkpoint *n*的数据重复执行(此时checkpoint *n*的状态尚未被确认为全部完成，只能重复消费)。

*NOTE*: 对齐操作只会发生在有多个上游operator(joins) 和多个下流operator 输出的时候(流计算的repartitioning/shuffle).因此，并行流计算的一些操作 (`map()`, `flatMap()`, `filter()`, ...) 事实上总是会被赋予 *eactly once*(执行一次)，哪怕给他们配置了 *at least once * 模式。



### 异步快照状态

上面描述的机制，意味着operators将会停止处理数据，当他们正在存储一个快照状态的时候。 这种同步状态快照会在每次生成快照的时候带来延迟。

当创建快照的时候让operator继续处理数据是可行的，也就是让快照状态在后台异步生成。为了做到这个，operator必需能够生产一个状态对象，这个对象需要被某种方法存储起来，并且不被后续的操作影响。举个例子，*copy-on-write*数据结构,比如使用RocksDB，可以达到这个功能。

在operator收到所有输入的checkpoint barriers后，这个operator开始异步拷贝快照的状态。operator 立刻发出 barrier给它的下游输出，并继续进行流处理。一旦后台完成了这个拷贝操作，它会向 jobManager的checkpoint coordinator确认这个checkpoint.这个checkpoint 需要在后面动作之后完成:所有的sinks 收到了barriers和所有的状态在后台被告知完成(这个动作可能比sink接收到barriers晚).

查看[State Backends]({{ site.baseurl }}/ops/state/state_backends.html) 了解快照状态详细。


## 恢复

这种机制下的错误恢复很简单: 在遇到一个故障的时候,flink 选择最近完成的checkpont *k* 。系统会重新部署整个分部署流处理，并把每个operator的状态设置到checkpoint *k* 对应的状态。这个数据源的起始读位置被设置为<i>S<sub>k</sub></i>。 例如在apache kafka中，这意味着消费者开始的偏移位置为<i>S<sub>k</sub></i>.

如果这个快照是增量的，这时候operators开始的状态是，从最后一个快照开始进行全量恢复，然后对这个状态进行一系列的增量快照更新。

查看[Restart Strategies]({{ site.baseurl }}/dev/restart_strategies.html)  获得更多信息


## Operator 快照实现

当Operator 快照创建时，这有两部分: **同布**和**异步**.

Operators和state backends 使用 Java `FutureTask`生成他们的快照。这个任务包含了快照的同步部分 和异步部分的计划。检查点的异步部分在这之后被一个后台线程执行。

Operators checkpoint同步的操作完成时候会返回一个`FutureTask`。如果一个异步操作需要被执行，他会在`FutrueTask`的`run()`方法中运行。

为了流和其他的资源开销可以被释放，这些任务是可以被取消的，。


{% top %}

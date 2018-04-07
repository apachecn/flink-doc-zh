---
title: Dataflow Programming Model
nav-id: programming-model
nav-pos: 1
nav-title: Programming Model
nav-parent_id: concepts
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

## Levels of Abstraction
## 抽象层次

Flink offers different levels of abstraction to develop streaming/batch applications.
Flink对开发流式（Streaming）和批处理（batch）程序提供了不同的抽象层次。

<img src="../fig/levels_of_abstraction.svg" alt="Programming levels of abstraction" class="offset" width="80%" />

  - 最低层的抽象只是提供 **状态流(stateful streaming)**。它被整合到[DataStream API](../dev/datastream_api.html)中。
    通过[处理函数(Process Function)](../dev/stream/operators/process_function.html)。它允许用户自由得处理一个或多个流事件，
    同时能使用一致的容错*状态*。此外,用户能通过注册事件时间(event time)和处理时间(processing time)来回调，
    允许程序来实现复杂的计算。

  - 在实践中，大部分程序不会用到上述的低层次抽象，但相对的会去使用
  
  - **core APIs** 比如[DataStream API](../dev/datastream_api.html)(有界/无界流)和[DataSet API](../dev/batch/index.html)(有界数据集)。在数据处理过程中，这些流畅的API提供通用的建立基础模块的功能，比如用户指定的各种表单转换(transformations)操作，联合(joins)操作，aggregations, windows, state等等。这些API对数据类型进行处理的过程中，这些数据类型被表示成各自程序语言中的类。

    低层次的*Process Function*整合了*DataStream API*，但只有再某些核心操作中才有可能进行较低级别的抽象。*DataSet API*为有界数据集提供了额外的语法，如loop/iterations。

    **Table API**是一个围绕着*tables*为中心的的声明式的DSL，并且式是可动态改变的tables（当表示数据流的时候）。
    [Table API](../dev/table_api.html) 遵循（或者说扩展）着关系模型：Tables有一个附属的schema（如关系数据库中的tables）。
    并且API提供了可比较（comparable）的操作，比如select，project，josin，group-by，aggregate，等等。
    Table API编程声明定义了*应该做什么样的逻辑操作*而不是准确得指定*这种操作应该的代码应该是什么样子*。虽然Table API是可以用户定义的各种方法来拓展的，但一般用户定义的方法的表现力综述差于*核心（core）APIs*，不过用起来会更加简洁（只需写更少的代码）。
    另外，Table API编程也会再运行前通过优化器进行一定的优化。

    你可以再tables和*DataStream*/*DataSet*之间无缝转换，允许在编程中将*Table API*和*DataStream*与*DataSet* APIs混在一起使用。

  - Flink提供的最高层次的抽象是**SQL**。这一抽象与*Table API*在语义和表现上比较相似，但程序的表现是通过SQL语句展示的。
    [SQL](../dev/table_api.html#sql) 紧密得与Table API交互，同时SQL语句能够被*Table API*中定义的表格（tables）上执行。


## 程序和数据流

对Flink来说，基本building blocks是**streams**和**transformations**（注意在Flink中，虽然DataSets使用的是DataSet API，但其内部依然是通过streams实现的，更多方面稍后会介绍）。从概念上讲，*stream*是一个data records的流（可能持续不断），以及*transformation*是一个将一个或多个streams当作输入，产生一个或多个输出streams结果的操作。

在执行的时候，Flink程序被映射成**streaming dataflows**，由**streams**和转换**操作**（transformation **operators**）；
每一个dataflow都是以一个或多个的**sources**开始，并且以一个或多个的**sinks**结束（译者：这点和kafka connect相似）。dataflows类似于的**有向无环图（directed acyclic graphs）** *(DAGs)*。虽然这种特殊的循环能够通过*iteration*初始化，但大多数情况下，我们会简单说明这点。

<img src="../fig/program_dataflow.svg" alt="A DataStream program, and its dataflow." class="offset" width="80%" />

通常程序中的转换（transformations）和数据流中的操作（operators）是一一对应的。然而有时候一个转换（transformation）可能有多个其他转换（transformation）操作组成。

Sources和sinks的具体记录在[streaming connectors](../dev/connectors/index.html)和[batch connectors](../dev/batch/connectors.html)的文档中。
转换（Transformations）记录在[DataStream operators]({{ site.baseurl }}/dev/stream/operators/index.html)和[DataSet transformations](../dev/batch/dataset_transformations.html)的文档中中。

{% top %}

## 并行数据流

程序在Flink中的本质是并行和分布式的。执行期间，一个*stream*有一个或多个的**流分区（stream partitions）**，并且每一个*运算（operator）*都有一个或多个的**运算子任务（operator subtasks）**。每个运算子任务都独立其他任务，而且都是在不同的线程中运行，也有可能是在不同的机器或是容器中。

数量上，运算子任务与那些特殊的运算是**并行的（parallelism）**。这些并行运算的stream总是归属它们本身的生产运算。同一程序中不同的运算可能有不同层次的并行。

<img src="../fig/parallel_dataflow.svg" alt="A parallel dataflow" class="offset" width="80%" />

在一个一对一的模式中，或者是在*重新分配（redistributing）*模式中，streams能够让数据在两个运算之间运输：

  - **一对一** streams（比如上面提到的从*Source*到*map()*之间的操作）保留分区并且对元素进行排序。这意味着*map()*运算的子任务会如被*Source*运算的子任务一样，在相同的排序中看到相同的元素。

  - **重新分配** streams（如早先提到的从*map()*到*keyBy/window*，以及从*keyBy/window*到*Sink*）会改变streams的分区。每一个*运算子任务*会根据选择的转换操作（transformation），发送数据到不同的目标子任务。例如*keyBy()*（通过重新分区来对键进行再哈希（hashing）），*broadcast()*或者是*rebalance()*（随机重新分区）。在*重新分配（redistributing）*中，交换元素之间的顺序仅出现在每一对发送和接收的子任务中（比如子任务一是*map()*并且子任务2是*keyBy/window*）。所以在这些例子中，每个键的顺序是可保存的，但并行性确实会导致顺序的非确定性，其中汇总的结果会根据不同的键送到不同的sink。

配置和控制并行性的细节可以在文档[parallel execution](../dev/parallel.html)找到。

{% top %}

## 窗口

聚合事件（比如counts,suns）的任务在流（streams）处理中是不同于批（batch）处理的。
比如，它不可能去count流（stream）中的所有元素，因为流一般是无限的（无界）。另一方面，streams的聚合操作（counts,sums等等），它们的作用范围是在一个**窗口（windows）**，像*“count前一个五分钟的所有元素”*，或者*"sum前100个元素"*。

窗口能够以*时间为驱动（time driven）*（例如：每30秒钟）或者以*数据为驱动*（例如每100个元素）。
不同类型的窗口的有一些典型区别，比如*翻转窗口（tumbling windows）*（没有重叠），*滑动窗口（sliding windows）*（有重叠），和*会话窗口（session windows）*（不活跃将被中断）。

<img src="../fig/windows.svg" alt="Time- and Count Windows" class="offset" width="80%" />

更多的的窗口样例能在[blog post](https://flink.apache.org/news/2015/12/04/Introducing-windows.html)中发现。
更多详细介绍在[window docs](../dev/stream/operators/windows.html)。

{% top %}

## 时间

当在streaming编程过程中指定了时间（比如定义窗口），那么将可以参考不同的时间概念如下：

  - **事件时间（Event Time）**是创建一个事件的时间。这通常在事件用用来描述其时间戳，比如作为生产者传感器的附属，或者是其他生产者服务。Flink可以通过[timestamp assigners]({{ site.baseurl }}/dev/event_timestamps_watermarks.html)访问事件时间戳。

  - **提取时间（Ingestion time）**是在source操作过程中，事件进入Flink数据流中的时间。

  - **处理时间（Processing Time）**是每一个基于时间的运算这个过程的本地时间。

<img src="../fig/event_ingestion_processing_time.svg" alt="Event Time, Ingestion Time, and Processing Time" class="offset" width="80%" />

更多如何处理时间的详细资料在[event time docs]({{ site.baseurl }}/dev/event_time.html)。

{% top %}

## 状态操作

许多数据流（dataflow）的运算只需要管理好自己*此次事件*（比如事件解析器），一些运算会记得多个事件的信息（比如窗口操作）。这些运算被称之为**stateful**。

状态操作的状态保存在可以将它视为键值对保存的地方存储着。
这些状态被严格得与通过与由状态操作读取的stream一起被分区和分割。于是，要访问key/value的状态只可能在调用*keyBy()*方法之后，通过*keyed streams*实现，同时它被限制于它的值于当前事件的key关联着。对齐流和状态的key可确保所有状态更新都是本地操作，可以保证一致性而无需事务开销。此对齐还允许Flink重新分配状态并透明地调整流分区。

<img src="../fig/state_partitioning.svg" alt="State and Partitioning" class="offset" width="50%" />

查看更多信息，查看该文档[state](../dev/stream/state/index.html)。

{% top %}

## Checkpoints和容错

Flink通过**stream replay**和**checkpointing**实现了容错机制。一个checkpoint都关联着一个特殊节点，这个特殊的点记录着每一个输入流对每一个运算对应的状态。一个数据流能够从checkpoint被回复并且保持一致性，*(正好一次处理语义)*这其实是通过重新执行checkpoint记录的point中的运算状态来实现的。

checkpoint间隔是在执行恢复时间期间交换容错开销的手段（需要恢复的事件个数）。

在[fault tolerance internals]({{ site.baseurl }}/internals/stream_checkpointing.html)中有提供更多详细关于Flink如何管理checkpoints的详细信息以及其关联的话题。
关于启用和配置checkpointing的细节在[checkpointing API docs](../dev/stream/state/checkpointing.html)中。

{% top %}

## 批处理流

Flink将运行[batch programs](../dev/batch/index.html)当作一个特殊的流程序（streaming programs），因为这个流是有界的（有限的元素）。
一个* DataSet *在内部被视为一个数据流上述概念因此适用于批处理程序，同样也适用于流式处理程序，除少数例外：

  - [批处理程序的容错](../dev/batch/fault_tolerance.html)不能使用checkpointing。
    数据恢复是通过完全重放数据流来实现的。
    这是可能的，因为输入时有界的。
    这会将恢复成本推向更高的水平，但会使常规处理更便宜，因为它避免了检查点。

  - DataSet API中的状态操作使用简单的内存中/外核（in-memory/out-of-core）的数据结构，而不是key/value索引。

  - DataSet API引入了特殊的同步（superstep-based）迭代，这些迭代只能在有界流上实现。更多详细信息，可在[iteration docs]({{ site.baseurl }}/dev/batch/iterations.html)查看。

{% top %}

## 下一步

继续学习Flink的基础概念[Distributed Runtime](runtime.html)。

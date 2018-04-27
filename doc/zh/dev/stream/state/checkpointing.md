---
title: "Checkpointing"
nav-parent_id: streaming_state
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

* ToC
{:toc}
在Flink中，每一个function和operator都可以是**状态化**的(参见 [working with state](state.html)查看细节)。
状态化functions与operators通过单个元素/事件来存储数据，这使得状态对于任何更加缜密类型的操作来说都是一个重要的组件。
为了使状态是可容错的，Flink需要状态的**检查点**，检查点允许Flink恢复流中的状态和位置，使得应用拥有和无故障执行相同的语义。

 [关于流容错的文档]({{ site.baseurl }}/internals/stream_checkpointing.html)
The [documentation on streaming fault tolerance]({{ site.baseurl }}/internals/stream_checkpointing.html) 详细描述了Flink的流容错机制背后的技术。


## 先决条件

Flink的检查点机制与流和状态的持久化存储交互，通常来说，该机制需要：

  - *持久化*的数据源，它可以在某个时间内重放记录。例如持久化的消息队列(比如Apache Kafka，RabbitMQ， Google PubSub)或是文件系统(例如HDFS，S3，GFS，NFS，Ceph等)。
  - 持久化的状态存储器，具有代表性的是分布式文件系统(例如HDFS，S3，GFS，NFS，Ceph等)。
  - A persistent storage for state, typically a distributed filesystem (e.g., HDFS, S3, GFS, NFS, Ceph, ...)


## 启用和配置检查点

检查点是默认关闭的，想要启动检查点，需在`StreamExecutionEnvironment`上调用`enableCheckpointing(n)`方法，其中的*n*参数表示以毫秒为单位的检查点间隔时间。

检查点的其它参数包括：

  - *exactly-once vs. at-least-once*：你可以选择这两种模式之一传递给`enableCheckpointing(n)`方法。
    Exactly-once对于大多出应用都是更适用的。At-least-once对于某些超低延迟(固定几毫秒)的应用可能更有意义。

  - *检查点超时*：如果检查点没有在改时间内完成，则会终止正在进行中的检查点。

  - 检查点间最短时间：为了保证流应用能在两个检查点之间能有一定程度的进展，你可以设定检查点间的最短时间，例如将其设置为*5000*，那么下一个检查点将在上一个检查点完后5秒内开始，且无视检查点持续时间与间隔时间。需要注意的是这意味着检查点间隔时间参数应该永远不小于此参数。
    通过定义“检查点间最短时间”而不是“检查点间隔时间”会使得应用更容易配置，因为“检查点间最短时间”不容易受到检查点有时会耗时超出平均值这一事实的应用(例如目标存储系统暂时缓慢)。
	注意，该值也意味着并发检查点的数量的*1*。

  - *检查点并发数*：默认情况下，系统不会在一个检查点进行的同时触发另一个检查点。这保证了拓扑不会再检查点上耗费过多时间以至于导致流处理的进度停滞不前。
	允许多个重叠的检查点也是有可能的，例如在有固定处理延时(比如因为函数调用外部服务而需要一些响应时间)却任需要频繁建立检查点(几百毫秒)以减轻失败重试代价的pipelines中。
	当定义了检查点最短时间参数是此选项不可用。

  - *外部化检查点*：你可以配置周期性的检查点在外部持久执行。外部化的检查点将他们的元数据写入到外部持久化的存储器中，并在任务失败时也*不会*被自动清理。通过这种方式，你将拥有一个在任务失败时可以从之恢复的检查点。关于外部化检查点的更多细节请参考[deployment notes on externalized checkpoints]({{ site.baseurl }}/ops/state/checkpoints.html#externalized-checkpoints).

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 每1000毫秒开启一个检查点
env.enableCheckpointing(1000);

// 高级选项：

// 将模式设置为exactly-once (默认)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// 保证检查点间有500ms的时间推进进度
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// 检查点必须在一分钟内完成，否则会被取消
env.getCheckpointConfig().setCheckpointTimeout(60000);

// 只允许同时进行一个检查点操作
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// 开启外部化检查点，它在作业被取消后保存
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

// 每1000毫秒开启一个检查点
env.enableCheckpointing(1000)

// 高级选项：

// 将模式设置为exactly-once (默认)
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)

// 保证检查点间有500ms的时间推进进度
env.getCheckpointConfig.setMinPauseBetweenCheckpoints(500)

// 检查点必须在一分钟内完成，否则会被取消
env.getCheckpointConfig.setCheckpointTimeout(60000)

// 只允许同时进行一个检查点操作
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
{% endhighlight %}
</div>
</div>

### 相关配置选项

一些更多的参数和/或预设值可以通过`conf/flink-conf.yaml` (参加 [configuration]({{ site.baseurl }}/ops/config.html))设置:

- `state.backend`: 用于存储当检查点开启时算子状态检查点的后端，支持的后端有：
   -  `jobmanager`: 内存后端，备份到JobManager/ZooKeeper的内存中。仅当在最小状态(Kafka offsets)或测试和本地调试时使用。
   -  `filesystem`: 状态在TaskManagers上存储在内存中，状态快照存储在文件系统中。支持所有Flink支持的文件系统，例如HDFS,S3等。

- `state.backend.fs.checkpointdir`: 在一个Flink支持的文件系统中用于存储检查点的目录。注意：状态后端必须可以从JobManager访问，仅在本地安装时使用`file://`

- `state.backend.rocksdb.checkpointdir`:  存放RocksDB文件的本地目录，或是通过系统目录定义符分割(例如Linux/Unix中的‘:’ (colon))的目录表(默认值为`taskmanager.tmp.dirs`)

- `state.checkpoints.dir`:  [外部化检查点]({{ site.baseurl }}/ops/state/checkpoints.html#externalized-checkpoints)存储元数据的目标目录。

- `state.checkpoints.num-retained`: 需要保持的已完成的检查点实例的数量，若最新检查点出错，设置为1以上将允许恢复回滚到先前的检查点

{% top %}


## 选择一个状态后端

Flink的 [检查点机制]({{ site.baseurl }}/internals/stream_checkpointing.html)存储计时器和状态化operators所有状态一致的快照，包括连接器，窗口和任何[用户自定义状态](state.html)。检查点的存储位置(例如JobManager内存，文件系统，数据库)取决于配置的**状态后端**。
默认情况下，状态存储在TaskManager的内存中，检查点存储在JobManager的内存中。为了对大型状态进行合适的持久化存储，Flink支持各种在其它状态后端中存储状态和检查点的方式，可以通过`StreamExecutionEnvironment.setStateBackend(…)`方法来配置选择的状态后端。
参见 [状态后端]({{ site.baseurl }}/ops/state/state_backends.html) 寻找更多关于job-wide和cluster-wide配置的可用状态后端和相关选项的细节。


## 迭代作业中的状态检查点
Flink目前职位非迭代式的作业提供处理保证。在迭代式作业中启用检查点会导致异常。如需在迭代作业中强制启用检查点，用户需要在启动检查点时舌一一个特殊标识：
`env.enableCheckpointing(interval, force = true)`.

请注意在出现故障期间，循环迭代中的记录(以及相关的状态改变)将会丢失。

{% top %}


## 重启策略

Flink支持不同的重启策略，它们将控制作业在失败时如何重启。更多详情请参阅 [重启策略]({{ site.baseurl }}/dev/restart_strategies.html)。

{% top %}


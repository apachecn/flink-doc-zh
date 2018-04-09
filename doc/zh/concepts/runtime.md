---
title: Distributed Runtime Environment
nav-pos: 2
nav-title: Distributed Runtime
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

## 任务和运算链（Operator Chains）

在分布式的运行下，Flink的将运算子任务*链（chains）*成一个*任务（tasks）*。每一个任务都被一个线程执行。
将运算链在一起成为一个任务是一种有效的优化方式：它减少线程切换到另一个线程之间的开销和缓存，并且提高总体吞吐量同时降低延迟。
链这一动作能够被配置，可以查看[chaining docs](../dev/datastream_api.html#task-chaining-and-resource-groups)中的详细信息。

下图中的示例数据流由五个子任务执行，因此具有五个并行线程。

<img src="../fig/tasks_chains.svg" alt="Operator chaining into Tasks" class="offset" width="80%" />

{% top %}

## Job管理器，Task管理器，客户端

Flink运行时由两种类型的进程组成:

  - **JobManagers**（也可以称之为*masters*）用以协调分布式环境的运行。它规划任务执行，协调检查点，同时也协调失败节点的恢复等等。

    在Flink中总是至少有一个作业管理器。甚至一个高可用性的系统会设置多个JobManagers，其中一个是*leader*，而其他的是*standby*。

  - 而**TaskManagers**（也被称之为*workers*）的作用是执行数据流的*tasks*（或者是其他特殊的任务，比如子任务），同时还有缓存和交换数据之间*streams*的功能。

    必须至少有一个TaskManager。

JobManagers和TaskManagers能够以各种方式开始：直接上机器配置[standalone cluster](../ops/deployment/cluster_setup.html)，或者在容器中，或者是用资源管理框架比如[YARN](../ops/deployment/yarn_setup.html)或[Mesos](../ops/deployment/mesos.html)。
TaskManagers连接到JobManager，通知说自己可用，并分配工作。

**client**其实并不是程序执行中和运行时的部分，但它被用来准备和发送dataflow到JobManager中。
在这以后，client的链接会关闭，或者保持链接状态以接收进度报告。client或者为java/scala的程序触发执行，或者用在命令行程序`./bin/flink run ...`。

<img src="../fig/processes.svg" alt="The processes involved in executing a Flink dataflow" class="offset" width="80%" />

{% top %}

## 任务间隙和资源 and Resources

每一个worker（任务管理器）都是一个*JVM程序*，可能在不同线程中运行着一个或多个子任务。
那么控制一个worker接收多少个任务的是什么呢，一个worker将它称之为**task slots**（至少一个）。

每一个*任务slot*表示一个任务管理器资源的子集。比如说，一个任务管理器有三个slots，它会将三一之一的管理内存分别送给每个slot。置入（sloting）资源意味着子任务不会与其他作业的子任务竞争被管理的内存，而是拥有一定的可管理内存。注意这里不会发生CPU隔离。当前slot只是隔离任务的管理内存。

通过调整任务slot的数量，用户能够定义子任务如何与其他子任务隔离。每个TaskManager都有一个slot意味着每一个任务组运行在不同的JVM（比如，能够运行在相互分离的容器中）。有多个slot意味着多个子任务共享一个相同的JVM的资源。所有任务在一个JVM中共享TCP链接（通过复用）和心跳消息。可能还会共享数据集和数据结构，从而减少每个任务的开销。

<img src="../fig/tasks_slots.svg" alt="A TaskManager with Task Slots and Tasks" class="offset" width="80%" />

默认得，Flink允许子任务共享slots，即使这些子任务是属于不同的任务（task），只要他们是属于同一作业（job）。这样的结果是一个slot可能持有一个作业的实体管道。允许*共享slot*主要有两个好处：

  - 一个Flink集群需要一个作业中使用到的的最高并行度的相同数量的任务slot。不需要计算一个程序总共包含多少任务（具有不同的并行性）。

  - 要获得更好的资源利用也比较容易。不适用共享slot，那么非密集的*source/map()*子任务将阻塞与资源密集型窗口子任务一样多的资源。
    通过共享slot，在我们的示例中将基本并行性从两个增加到六个，可以充分利用slot的资源，同时确保繁重的子任务在TaskManager之间公平分配。

<img src="../fig/slot_sharing.svg" alt="TaskManagers with shared Task Slots" class="offset" width="80%" />

这些APIs也包含了*[resource group](../dev/datastream_api.html#task-chaining-and-resource-groups)*机制用来阻止不合理的共享slot。

作为rule-of-thumb，一个好的默认的任务slot数将回是CPU核的数量。随着超线程，每一个slot会用两个或更多的线程上下文所用到的硬件。

{% top %}

## 状态


存储键/值索引的确切数据结构取决于所选的[状态后端（state backend）](../ops/state/state_backends.html)。一种状态后端将数据存储在内存的哈希映射（hash map）中，而其他的状态后端将[RocksDB](http://rocksdb.org)作为key/value的存储地点。除了定义保存状态的数据结构之外，状态后端还实现了产生一个key/value状态的具体时间点的快照和将这个快照当作checkpoint保存。

<img src="../fig/checkpoints.svg" alt="checkpoints and snapshots" class="offset" width="60%" />

{% top %}

## 保存点

用Data Stream API编写的程序能够从**保存点（savepoint）**恢复。保存点支持更新你的程序和Flink汲取二者都不会丢失状态信息。


[保存点（Savepoints）](../ops/state/savepoints.html)是**手动触发的检查点（manually triggered checkpoint）**,会保存一个程序快照（snapshot ）并写入到状态后端中。它其实是依赖于常规的（checkpointing ）机制来实现的。在执行任务期间定期得在worker node上产生快照并产生检查点。要恢复的话其实只是需要最后一个完整的检查点，故而一旦一个写您的检查点完成，那么旧的就会被安全得删除。

Savepoints are similar to these periodic checkpoints except that they are **triggered by the user** and **don't automatically expire** when newer checkpoints are completed. Savepoints can be created from the [command line](../ops/cli.html#savepoints) or when cancelling a job via the [REST API](../monitoring/rest_api.html#cancel-job-with-savepoint).

保存点其实类似于定期的检查点，只是它**由用户触发**并且当一个新的检查点完成时它**不会自动到期（expire）**。保存点能够从[命令行](../ops/cli.html#savepoints)创建或者当一个作业（job）从[rest api](../monitoring/rest_api.html#cancel-job-with-savepoint)中被取消。

{% top %}

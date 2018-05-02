---
title:  "作业和调度"
nav-parent_id: internals
nav-pos: 4
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

该文档简单描述了Flink是如何调度作业的，以及如何在jonManager上体现并跟踪作业状态的。

This document briefly describes how Flink schedules jobs and
how it represents and tracks job status on the JobManager.

* This will be replaced by the TOC
{:toc}


## 调度

Flink中的执行资源被定义为 _Task Slots_。每一个TaskManager都有一个或多个 task slots,每个task slots可以用来运行并行任务中的一个任务。一个并行任务由多个连续的任务组成，比如 *n-th* 个并行的Map程序和 *n-th* 个并行的Reduce程序共同组成。注意，flink经常会并发执行多个连续任务:对于流任务，总是这种情况，对于批量任务，也经常是这种情况。

如下图表所示。一个程序有一个数据源，一个 *MapFunction(map程序)*,一个*ReduceFunction(reduce程序)*。数据源和map程序执行的并行度为4,Reduce程序的并发执行度被设置为3.一个数据管道由 Source-Map-Reduce的顺序组成。假设一个有2个TaskManagers,每个TaskManager有3个slots，这个程序的执行方式会如下所示。 

<div style="text-align: center;">
<img src="{{ site.baseurl }}/fig/slots.svg" alt="Assigning Pipelines of Tasks to Slots" height="250px" style="text-align: center;"/>
</div>

flink 内部通过定义 {% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobmanager/scheduler/SlotSharingGroup.java "SlotSharingGroup" %}
和 {% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobmanager/scheduler/CoLocationGroup.java "CoLocationGroup" %} 来决定哪些任务可能共用一个slot,也可以分别严格的规定哪些任务放置在同一个slot中。


## JobManager 数据结构

在job执行的时候，jobManager保持对分布式任务的跟踪，来决定什么时候执行下一个任务(或者下一组任务),并对结束的或执行失败的任务作出响应。

Jobmanager接收到{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/ "JobGraph" %}, jobGraph是流计算operators的表现形式，由若干operator ({% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/JobVertex.java "JobVertex" %})
和中间结果集 ({% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/IntermediateDataSet.java "IntermediateDataSet" %}) 组成.
每个Operator有一些属性，比如parallelism（并行度）和需要执行的代码。另外，JobGraph有一些执行依赖包,是operators执行代码时候需要用到的。

JobManager把JobGraph转换成一个 {% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/ "ExecutionGraph" %}.ExecutionGrap是JobGraph的平行版本：对于每个JobVertex，它的每个并行子任务都包含一个{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/ExecutionVertex.java "ExecutionVertex" %} 。假设一个Operator并行度为100，那这个Operatior将会有1个JobVertex和1000个ExecurtionVertex.ExecutionVertex会跟踪子任务的执行状态。来自同一个JobVertex的所有的ExecutionVertices会被存储到一个{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/ExecutionJobVertex.java "ExecutionJobVertex" %}中，ExecutionJobVertex跟踪整个operator的执行状态。
除了这些节点(各种Vertex)之外,ExecutionGraph也包含{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/IntermediateResult.java "IntermediateResult" %}(中间结果) 和{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/IntermediateResultPartition.java "IntermediateResultPartition" %}（中间结果分区）.前者跟踪 *IntermediateDateSet*的状态，后者跟踪*IntermediateDateSet* 分区的状态。

<div style="text-align: center;">
<img src="{{ site.baseurl }}/fig/job_and_execution_graph.svg" alt="JobGraph and ExecutionGraph" height="400px" style="text-align: center;"/>
</div>

每个ExecturionGraph有一个job status和它联系在一起。这个jon status表示作业执行时候的当前状态。

一个Flink作业最早的状态是 *created*,然后会切换到 *running* ，当所有的任务都完成后会转换成*finished*。
对于失败的情况，job的第一个状态为 *failing*，即它取消所有运行任务的状态。
如果作业所有的点都到达了一个最终状态并且job不能重启，这个时候作业会转换为*failed*。如果这个作业可以被重启，它会进入*restarting*状态。

在用户取消的job中,作业会进入*cancelling*状态。这个状态会导致所有正在运行的任务取消。一旦所有正在运行的作业达到了取消这个最终状态，这个作业会被转换为状态*cancelled*。

不像状态*finished*,*canceled*和*failed* 意味着一个全局的最终状态，并且触发清理作业的动作，*suspended* 状态只是本地终端的。本地终端意味着当一个执行的作业在相应JobManger被中断的时候，可以被另外一个Flink集群的JobManager可以作为HA存储并重启这个任务。因此，当一个任务到达*suspended*状态的时候，不会被完全清除。

<div style="text-align: center;">
<img src="{{ site.baseurl }}/fig/job_status.svg" alt="States and Transitions of Flink job" height="500px" style="text-align: center;"/>
</div>

在ExecutionGraph执行的过程中,每一个并行任务经过多个阶段，从*created*到*finished* or *failed*。如下图表是各个状态之间可能出现的转换。一个任务可能被多次执行（比如出现故障恢复）。由于这个原因，一个ExecutionVertex的执行被一个{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/Execution.java "Execution" %}跟踪。每个ExecutionVertex有一个当前Execution和前置Executions。

<div style="text-align: center;">
<img src="{{ site.baseurl }}/fig/state_machine.svg" alt="States and Transitions of Task Executions" height="300px" style="text-align: center;"/>
</div>

{% top %}

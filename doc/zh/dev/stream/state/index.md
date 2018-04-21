---
title: "State & Fault Tolerance"
nav-id: streaming_state
nav-title: "State & Fault Tolerance"
nav-parent_id: streaming
nav-pos: 3
nav-show_overview: true
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

状态化函数与算子通过个体元素/事件来存储数据，这使得状态对于任何更加缜密的操作类型来说都是一个重要的组件。

Stateful functions and operators store data across the processing of individual elements/events, making state a critical building block for
any type of more elaborate operation.

例如：
  - 当一个程序搜索确定的事件模式，状态将会存储当前为止发生的时间序列。
  - 当以分钟/小时/天为单位聚合事件时，状态会持有等待执行的聚合。
  - 当通过数据点流训练一个机器学习模型，状态将持有当前模型参数的版本。
  - 当历史性数据需要被管理，状态允许对过去发生的时间进行高效访问。


For example:

  - When an application searches for certain event patterns, the state will store the sequence of events encountered so far.
  - When aggregating events per minute/hour/day, the state holds the pending aggregates.
  - When training a machine learning model over a stream of data points, the state holds the current version of the model parameters.
  - When historic data needs to be managed, the state allows efficient access to events that occurred in the past.

Flink需要感知状态，以便使用检查点来使得状态是容错的，同时允许了流式应用使用保存点
Flink needs to be aware of the state in order to make state fault tolerant using [checkpoints](checkpointing.html) and to allow [savepoints]({{ site.baseurl }}/ops/state/savepoints.html) of streaming applications.

关于状态的知识也允许对Flink应用进行再次扩展，这意味着Flink关心通过并行实例重新分配的状态。
Knowledge about the state also allows for rescaling Flink applications, meaning that Flink takes care of redistributing state across parallel instances.

Flink的可查询状态特性允许你在运行时从Flink外部获取状态
The [queryable state](queryable_state.html) feature of Flink allows you to access state from outside of Flink during runtime.

当使用状态时，阅读Flink的状态后端可能会有所帮助.Flink提供不同的状态后端，这些后端具体到状态是如何被保存以及在哪被保存的。状态可能存也可能不存在于Java heap中，这取决于你的状态后端，Flink也能为程序管理状态，这意味着Flink处理内存管理(如果必要也有可能分配到磁盘中)来允许应用持有大型的状态。状态后端无需改变你的程序逻辑便可以进行配置。
When working with state, it might also be useful to read about [Flink's state backends]({{ site.baseurl }}/ops/state/state_backends.html). Flink provides different state backends that specify how and where state is stored. State can be located on Java's heap or off-heap. Depending on your state backend, Flink can also *manage* the state for the application, meaning Flink deals with the memory management (possibly spilling to disk if necessary) to allow applications to hold very large state. State backends can be configured without changing your application logic.

{% top %}

Where to go next?
-----------------

* [Working with State](state.html): Shows how to use state in a Flink application and explains the different kinds of state.
* [Checkpointing](checkpointing.html): Describes how to enable and configure checkpointing for fault tolerance.
* [Queryable State](queryable_state.html): Explains how to access state from outside of Flink during runtime.
* [Custom Serialization for Managed State](custom_serialization.html): Discusses custom serialization logic for state and its upgrades.

{% top %}

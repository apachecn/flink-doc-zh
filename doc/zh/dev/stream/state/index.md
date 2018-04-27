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

状态化函数与算子通过单个元素/事件来存储数据，这使得状态对于任何更加缜密类型的操作来说都是一个重要的组件。

例如：
  - 当一个程序搜索确定的事件模式，状态将会存储当前为止发生的时间序列。
  - 当以分钟/小时/天为单位聚合事件时，状态会持有等待执行的聚合。
  - 当通过数据点流训练一个机器学习模型，状态将持有当前模型参数的版本。
  - 当历史性数据需要被管理，状态允许对过去发生的时间进行高效访问。

Flink需要感知状态，以便使用 [检查点](checkpointing.html) 来使得状态是可容错的，同时允许了流式应用使用 [保存点]({{ site.baseurl }}/ops/state/savepoints.html)

关于状态的知识也允许对Flink应用进行再次扩展，这意味着Flink关心通过并行实例重新分配的状态。

Flink的可查询状态特性允许你在运行时从Flink外部获取状态
The [可查询状态](queryable_state.html) 

当使用状态时，阅读Flink的状态后端可能会有所帮助.Flink提供不同的状态后端，这些后端具体到状态是如何被保存以及在哪被保存的。状态可能存也可能不存在于Java heap中，这取决于你的状态后端，Flink也能为程序管理状态，这意味着Flink处理内存管理(如果必要也有可能分配到磁盘中)来允许应用持有大型的状态。状态后端无需改变你的程序逻辑便可以进行配置。

{% top %}

下一步去哪?
-----------------

* [使用状态](state.html): 展示如何在Flink程序中使用状态，并解释了不同的状态类型。
* [检查点](checkpointing.html):描述如何开启和配置检查点来使程序高容错。 
* [可查询状态](queryable_state.html):解释如何在运行时从Flink外部读取状态 
* [为托管状态自定义序列化操作](custom_serialization.html): 讨论状态和及升级的自定义序列化操作逻辑。

{% top %}

---
title:  组件堆栈
nav-parent_id: internals
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

作为一个软件堆栈，flink是一个分层的系统。不同的堆栈层建立在其他层之上，并在这些层上面抽象出不同的可供编程的抽象接口：


- **runtime** 层接收一个*JobGraph*形式的程序。一个JobGraph是通用数据流，可以是任意消费者和生产者的数据流任务。

- **DataStream API**和**DataSet API**都是通过单独的编译流程生成JobGraphs。DataSet Api使用optimizer(优化器)来确定程序的最佳计划，而DataStream API使用stream builder.

- JobGraph根据flink提供的的多种部署选项来执行（例如本地、远程、yarn等)

- 与flink捆绑在一起的Libraries和apis生成DataSet或DataStream api程序。这些包括用于逻辑表的表查询,用于机器学习的FlinkML,用于图像处理的Gelly。


点击图表中的组件了解更多。

You can click on the components in the figure to learn more.

<center>
  <img src="{{ site.baseurl }}/fig/stack.png" width="700px" alt="Apache Flink: Stack" usemap="#overview-stack">
</center>

<map name="overview-stack">
<area id="lib-datastream-cep" title="CEP: Complex Event Processing" href="{{ site.baseurl }}/dev/libs/cep.html" shape="rect" coords="63,0,143,177" />
<area id="lib-datastream-table" title="Table: Relational DataStreams" href="{{ site.baseurl }}/dev/table_api.html" shape="rect" coords="143,0,223,177" />
<area id="lib-dataset-ml" title="FlinkML: Machine Learning" href="{{ site.baseurl }}/dev/libs/ml/index.html" shape="rect" coords="382,2,462,176" />
<area id="lib-dataset-gelly" title="Gelly: Graph Processing" href="{{ site.baseurl }}/dev/libs/gelly/index.html" shape="rect" coords="461,0,541,177" />
<area id="lib-dataset-table" title="Table API and SQL" href="{{ site.baseurl }}/dev/table_api.html" shape="rect" coords="544,0,624,177" />
<area id="datastream" title="DataStream API" href="{{ site.baseurl }}/dev/datastream_api.html" shape="rect" coords="64,177,379,255" />
<area id="dataset" title="DataSet API" href="{{ site.baseurl }}/dev/batch/index.html" shape="rect" coords="382,177,697,255" />
<area id="runtime" title="Runtime" href="{{ site.baseurl }}/concepts/runtime.html" shape="rect" coords="63,257,700,335" />
<area id="local" title="Local" href="{{ site.baseurl }}/quickstart/setup_quickstart.html" shape="rect" coords="62,337,275,414" />
<area id="cluster" title="Cluster" href="{{ site.baseurl }}/ops/deployment/cluster_setup.html" shape="rect" coords="273,336,486,413" />
<area id="cloud" title="Cloud" href="{{ site.baseurl }}/ops/deployment/gce_setup.html" shape="rect" coords="485,336,700,414" />
</map>

{% top %}

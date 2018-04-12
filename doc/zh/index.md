---
title: "Apache Flink文档"
nav-pos: 0
nav-title: '<i class="fa fa-home title" aria-hidden="true"></i> Home'
nav-parent_id: root
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



本文档适用于Apache Flink 1.4.2，这些页面建立于{% build_time %}。

Apache Flink是一个关于分布式流式（stream）和批量（batch）数据处理的开源平台。Flink的核心在于通过数据流提供分布式环境下的数据分配，通讯，以及容错机制的数据流引擎（dataflow engine）。Flink的批处理（batch）功能亦是建立在流数据引擎的基础之上，并且覆盖其远程的迭代支持，管理内存，以及程序优化。

## 第一步

- **概念**: 从Flink的基础概念开始[数据流编程模型](concepts/programming-model.html)以及[分布式运行环境](concepts/runtime.html)。这两部分能帮帮助你理解文档的其他部分，包括安装和编程指南。我们推荐你先看这一部分。

- **快速开始**:[运行样例程序](quickstart/setup_quickstart.html)在你的本地机器上运行样例程序或者[学习一些其他样例](examples/index.html)。

- **编程指南**:你可以查看我们一些关于Flink的指导[基本的api概念](dev/api_concepts.html)和[DataStream API](dev/datastream_api.html)或者是[DataSet API](dev/batch/index.html)，来学习如何写你的第一个flink程序。

## 部署

在将你的Flink job放置到生产环境之前，可以查看[Production Readiness Checklist](ops/production_ready.html)。

## 迁移指南

对于那些使用Flink早先版本的用户，我们推荐你查看[API migration guide](dev/migration.html)。
然而所有被我们标记为public和stable的API任然是被支持的（标记为public的API是向下兼容的），我们建议迁移应用到可应用的最新版本。

对那些需要在生产环境将Flink系统升级的用户，我们推荐你查看[升级Apache Flink](ops/upgrading.html)的指南。

## 外部资源

- **先进的Flink（Flink Forward）**:过去会议的讨论记录可以在[Flink Forward](http://flink-forward.org/)网站以及在[YouTube](https://www.youtube.com/channel/UCY8_lgiZLZErZPF47a2hXMA)上得到。
[Robust Stream Processing with Apache Flink](http://2016.flink-forward.org/kb_sessions/robust-stream-processing-with-apache-flink/)是一个开始的好地方。

- **练习**:在[training materials](http://training.data-artisans.com/)上有从数据工匠（data Artisans）而来的放灯片，练习和一些简单的解决方案。

- **博客**:[Apache Flink](https://flink.apache.org/blog/)和[data Artisans](https://data-artisans.com/blog/)的博客上密切得发布一些深入于Flink技术的文章。

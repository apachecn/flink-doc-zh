---
title: "Streaming Connectors"
nav-id: connectors
nav-title: Connectors
nav-parent_id: streaming
nav-pos: 30
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

* toc
{:toc}

## 预定义Sources和Sioks

Flink内置了一些基本的数据Sources和Sink，并且始终可用。在[predefined data sources]({{ site.baseurl }}/dev/datastream_api.html#data-sources)中包含了如何从文件，目录，套接字，以及从集合和迭代器中提取数据的内容。在[predefined data sinks]({{ site.baseurl }}/dev/datastream_api.html#data-sinks)中有写（writing）文件，写标准输出，写标准错误和写套接字的内容。

## 捆绑（Bundled）连接器

连接器提供了与各种第三方系统接口的代码。当前支持的系统如下：

 * [Apache Kafka](kafka.html) (source/sink)
 * [Apache Cassandra](cassandra.html) (sink)
 * [Amazon Kinesis Streams](kinesis.html) (source/sink)
 * [Elasticsearch](elasticsearch.html) (sink)
 * [Hadoop FileSystem](filesystem_sink.html) (sink)
 * [RabbitMQ](rabbitmq.html) (source/sink)
 * [Apache NiFi](nifi.html) (source/sink)
 * [Twitter Streaming API](twitter.html) (source)

记住，要在应用程序中使用其中一个连接器，通常需要额外的第三方组件。比如，存储服务以及消息队列。还注意的是，虽然本节中列出的流连接器是Flink项目中的一部分并包含在源代码版本里，但它们不包含在二进制发行版中。

## Apache Bahir中的连接器

在Flink中，额外的流连接器是通过[Apache Bahir](https://bahir.apache.org/)发布的。包括：

 * [Apache ActiveMQ](https://bahir.apache.org/docs/flink/current/flink-streaming-activemq/) (source/sink)
 * [Apache Flume](https://bahir.apache.org/docs/flink/current/flink-streaming-flume/) (sink)
 * [Redis](https://bahir.apache.org/docs/flink/current/flink-streaming-redis/) (sink)
 * [Akka](https://bahir.apache.org/docs/flink/current/flink-streaming-akka/) (sink)
 * [Netty](https://bahir.apache.org/docs/flink/current/flink-streaming-netty/) (source)

## 其他链接到Flink的方式

### 通过异步实现数据浓缩（Data Enrichment）

使用连接器不是使数据进出Flink的唯一方式。一个通用的模式是在一个`Map`或`FlatMap`中查询一个外部数据库或者是web服务从而实现数据浓缩。
Flink为[Asynchronous I/O]({{ site.baseurl }}/dev/stream/operators/asyncio.html)提供API来使这些数据浓缩（enrichment）的操作更加方便和更加稳健。

### 可查询的状态

当一个Flink程序将大量数据推送到其他外部的存储地方时，这时I/O可能会成为瓶颈。
如果涉及的数据读取次数少于写入次数，则可采用更好的途径使得外部应用程序能从Flink提取所需的数据。
在[Queryable State]({{ site.baseurl }}/dev/stream/state/queryable_state.html)接口通过允许由Flink管理的状态按需查询来实现这一点。

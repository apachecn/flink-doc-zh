---
title: "对数据的Sources和Sinks的容错保证"
nav-title: 容错保证
nav-parent_id: 连接器
nav-pos: 0
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

Flink的容错机制能够使程序从错误中恢复并继续执行。这里的错误包括机器硬件错误，网络错误和短暂的程序错误等等。

只有当source参与到快照机制时，Flink才能保证恰好一次就将状态更新到瀛湖定义的状态。下表列出了Flink与捆绑连接器（bundled connectors）的状态更新保证。

请阅读每个连接器的文档以了解容错保证的详细信息。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Source</th>
      <th class="text-left" style="width: 25%">保证（Guarantees）</th>
      <th class="text-left">备注</th>
    </tr>
   </thead>
   <tbody>
        <tr>
            <td>Apache Kafka</td>
            <td>恰好一次</td>
            <td>为您的版本使用适当的Kafka连接器</td>
        </tr>
        <tr>
            <td>AWS Kinesis Streams</td>
            <td>恰好一次</td>
            <td></td>
        </tr>
        <tr>
            <td>RabbitMQ</td>
            <td>at 最多一次 (v 0.10) / 恰好一次 (v 1.0) </td>
            <td></td>
        </tr>
        <tr>
            <td>Twitter Streaming API</td>
            <td>最多一次</td>
            <td></td>
        </tr>
        <tr>
            <td>Collections</td>
            <td>恰好一次</td>
            <td></td>
        </tr>
        <tr>
            <td>Files</td>
            <td>恰好一次</td>
            <td></td>
        </tr>
        <tr>
            <td>Sockets</td>
            <td>最多一次</td>
            <td></td>
        </tr>
  </tbody>
</table>

为了保证端到端恰好一次的记录交付，数据sink需要参与到检查点（checkpointing）机制中。下表列出了Flink与捆绑式接收器的交付保证（假设只有一次状态更新）：	

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Sink</th>
      <th class="text-left" style="width: 25%">保证</th>
      <th class="text-left">备注</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>HDFS rolling sink</td>
        <td>恰好一次</td>
        <td>取决于Hadoop版本实现</td>
    </tr>
    <tr>
        <td>Elasticsearch</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Kafka producer</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Cassandra sink</td>
        <td>至少一次 / 恰好一次</td>
        <td>只有当idempotent updates才时恰好一次</td>
    </tr>
    <tr>
        <td>AWS Kinesis Streams</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>File sinks</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Socket sinks</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Standard output</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Redis sink</td>
        <td>至少一次</td>
        <td></td>
    </tr>
  </tbody>
</table>

{% top %}

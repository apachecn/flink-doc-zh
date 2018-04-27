---
title: "State Backends"
nav-parent_id: streaming_state
nav-pos: 5
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

Flink提供不同的状态后端来确定状态保存的位置和方式。

状态可能保存在Java的堆中或堆外，这取决于你的状态后端，Flink也可以为应用管理状态，这意味着Flink于内存管理(必要时也有可能分配到磁盘中)打交道，以允许应用持有大型状态。默认情况下，配置文件 *flink-conf.yaml*决定了所有Flink作业的状态后端。

当然，默认的状态后端也可以以单个作业为单位被重写，就像如下展示的一样。

寻找更多有关可用状态后端，以及他们的优势，限制和配置参数，请参阅[部署与操作]({{ site.baseurl }}/ops/state/state_backends.html)中相应的部分。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.setStateBackend(...)
{% endhighlight %}
</div>
</div>

{% top %}

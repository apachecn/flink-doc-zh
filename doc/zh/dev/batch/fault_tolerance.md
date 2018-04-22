---
title: "容错"
nav-parent_id: batch
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

Flink的容错机制可以在出现故障时恢复程序并继续执行它们。这种故障包括机器硬件故障，网络故障，瞬态程序故障等。

* This will be replaced by the TOC
{:toc}

批处理容错（DataSet API）
----------------------------------------------

*DataSet API*中程序的容错能力通过重试失败的执行来实现。在作业声明为失败之前，Flink重试执行的次数可通过执行重试参数进行配置。0意味着容错被禁用。

要激活容错机制，请将执行重试次数设置为大于零的值。 一个常见的选择是3。

此示例显示如何配置Flink DataSet程序的执行重试次数。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setNumberOfExecutionRetries(3);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setNumberOfExecutionRetries(3)
{% endhighlight %}
</div>
</div>


您还可以在flink-conf.yaml中配置执行重试次数和重试延迟的默认值：

~~~
execution-retries.default: 3
~~~


Retry Delays
------------

执行重试可以配置为延迟执行。延迟重试意味着在执行失败后，重新执行不会立即开始，而只会在某个延迟之后开始。

当程序与外部系统进行交互时，延迟重试会很有帮助，例如，在重新执行尝试之前，连接或待处理事务应达到超时。

您可以按如下方式为每个程序设置重试延迟（示例显示了DataStream API - DataSet API的工作原理类似）：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setExecutionRetryDelay(5000); // 5000 milliseconds delay
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.getConfig.setExecutionRetryDelay(5000) // 5000 milliseconds delay
{% endhighlight %}
</div>
</div>

您也可以在 `flink-conf.yaml`中配置重试延迟的默认值:

~~~
execution-retries.delay: 10 s
~~~

{% top %}

---
title: "Table API & SQL"
nav-id: tableapi
nav-parent_id: dev
is_beta: false
nav-show_overview: true
nav-pos: 35
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

Apache Flink有两个关系型API-Table API和SQL，作为统一流处理和批处理的API。Table API是兼容Java和Scala的语言集成查询API，同时，可以非常直观地通过例如selection，filter及join等的关系操作符进行查询组合。对于Flink的SQL操作，例如，SQL校验、SQL解析及SQL优化，都交由支持标准SQL语言的[Apache Calcite](https://calcite.apache.org) 处理。无论输入形式是DataSet还是DataStream，在两个接口中指定的查询都具有相同的语义并且指定相同的结果。

类似于DataStream API和DataSet API，Table API和SQL接口关系密切。开发人员可以很容易地在所有API和库之间切换。例如，你可以利用DataStream中的[CEP库]({{ site.baseurl }}/dev/libs/cep.html)提取模式，之后转为用Table API对模式进行分析，或者你还可以使用SQL查询，去scan，filter和aggregate一个批处理表，之后再对预处理数据运行[Gelly图算法]({{ site.baseurl }}/dev/libs/gelly)。

**注意：Table API和SQL在不断地积极开发中，还没有完成，所以并非所有的操作都支持。**

Setup
-----

由于`flink-table`的Maven插件中并不包括Table API和SQL，所以必须将一下的依赖添加到你的项目中，才可以使用Table API和SQL：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

除此之外，你还需要为Flink的Scala批处理或者流处理API添加一个依赖。对于批处理查询，你需要添加：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

对于流处理查询，你需要添加：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

**注意：**由于Apache Calcite会防止用户类加载器被当垃圾收集，我们不建议使用包含`flink-table`依赖项的jar包，而推荐在系统类加载器中配置flink，以包含依赖。这个可以通过从`./opt`文件夹中拷贝flink-table的.jar文件，并复制到`./lib`文件夹中。进一步详情，请参见[说明]({{ site.baseurl }}/dev/linking.html)。

{% top %}

Where to go next?
-----------------

* [Concepts & Common API]({{ site.baseurl }}/dev/table/common.html): 共享Table API和SQL的相关概念和APIs。
* [Streaming Table API & SQL]({{ site.baseurl }}/dev/table/streaming.html): 在Table API & SQL上的特定流处理文档，例如，配置时间属性和处理更新结果。
* [Table API]({{ site.baseurl }}/dev/table/tableApi.html): Table API支持的操作和API。
* [SQL]({{ site.baseurl }}/dev/table/sql.html): SQL支持的操作和语法。
* [Table Sources & Sinks]({{ site.baseurl }}/dev/table/sourceSinks.html): 从外部存储系统读取表，及对表进行存储。
* [User-Defined Functions]({{ site.baseurl }}/dev/table/udfs.html): 用户自定义函数的定义和使用。

{% top %}
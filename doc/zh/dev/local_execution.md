---
title:  "本地执行"
nav-parent_id: batch
nav-pos: 8
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

Flink可以在单台机器上运行，甚至在单个Java虚拟机中。这允许用户在本地测试和调试Flink程序。本节概述了本地执行机制。

本地环境和执行程序允许您在本地Java虚拟机中运行Flink程序，或者在任何JVM中运行Flink程序作为现有程序的一部分。只需点击IDE的“运行”按钮即可在本地启动大多数示例。

Flink支持两种不同的本地执行。LocalExecutionEnvironment启动完整的Flink运行时，包括一个JobManager和一个TaskManager。这些包括内存管理和在集群模式下执行的所有内部算法。

`CollectionEnvironment`在Java集合上执行Flink程序。 这种模式不会启动完整的Flink运行时，所以执行起来消耗非常低而且很轻。例如`DataSet.map()`- 转换将通过将`map()` 函数应用于Java List中的所有元素来执行。

* TOC
{:toc}


## Debugging

如果您在本地运行Flink程序，则还可以像调试其他Java程序一样调试程序。你可以使用 `System.out.println()`打印出一些内部变量，或者你可以使用调试器。可以在 `map()`, `reduce()` 和所有其他方法中设置断点。
请参阅Java API文档中的 [debugging section]({{ site.baseurl }}/dev/batch/index.html#debugging)以获取Java API中测试和本地调试程序的指南。

## Maven Dependency

如果您开发的程序是 Maven 项目, 您必须添加 `flink-clients` 模块依赖:

~~~xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version}}</version>
</dependency>
~~~

## Local Environment

 `LocalEnvironment` 是Flink程序本地执行的句柄。用它在本地JVM中运行程序 - 独立运行或嵌入其他程序中。

本地环境通过`ExecutionEnvironment.createLocalEnvironment()`方法实例化。默认情况下，它将使用尽可能多的本地线程数(与您的机器具有CPU核心数一致)执行。您也可以指定所需的并行度。本地环境可以通过`enableLogging()`/`disableLogging()` 打印日志到控制台。

在大多数情况下，调用`ExecutionEnvironment.getExecutionEnvironment()` 是更好的方法。当程序在本地启动时（命令行界面外），该方法返回一个`LocalEnvironment`，并在程序被 [command line interface]({{ site.baseurl }}/ops/cli.html)调用时返回一个预配置的群集执行环境。

~~~java
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

    DataSet<String> data = env.readTextFile("file:///path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("file:///path/to/result");

    JobExecutionResult res = env.execute();
}
~~~

`JobExecutionResult` 对象在执行完成后返回，它包含程序运行时和累加器结果。

`LocalEnvironment`还允许将自定义配置值传递给Flink。

~~~java
Configuration conf = new Configuration();
conf.setFloat(ConfigConstants.TASK_MANAGER_MEMORY_FRACTION_KEY, 0.5f);
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment(conf);
~~~

*Note:* 本地执行环境不启动任何Web前端来监视执行。

## Collection Environment

使用`CollectionEnvironment`在ava集合上执行是执行Flink程序的低开销方法。这种模式的典型用例是自动化测试，调试和代码重用。

用户也可以使用为批处理实施的算法，以便更具交互性的案例。可以在Java应用服务器中使用稍微变化的Flink程序来处理传入的请求。

**基于集合执行的骨架**

~~~java
public static void main(String[] args) throws Exception {
    // initialize a new Collection-based execution environment
    final ExecutionEnvironment env = new CollectionEnvironment();

    DataSet<User> users = env.fromCollection( /* get elements from a Java Collection */);

    /* Data Set transformations ... */

    // retrieve the resulting Tuple2 elements into a ArrayList.
    Collection<...> result = new ArrayList<...>();
    resultDataSet.output(new LocalCollectionOutputFormat<...>(result));

    // kick off execution.
    env.execute();

    // Do some work with the resulting ArrayList (=Collection).
    for(... t : result) {
        System.err.println("Result = "+t);
    }
}
~~~

`flink-examples-batch` 模块包含叫`CollectionExecutionExample`的完整示例 .

请注意，基于集合的Flink程序的执行仅适用于适合JVM堆的小数据。集合上的执行不是多线程的，只使用一个线程。

{% top %}

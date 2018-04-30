---
title: "Scala REPL"
nav-parent_id: start
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

Flink带有一个集成的交互式Scala Shell。它可以用于本地设置以及集群设置。

要使用带有集成Flink集群的shell，只需在Flink根目录下执行以下命令：

~~~bash
bin/start-scala-shell.sh local
~~~

要在群集上运行Shell，请参阅下面的“设置”部分。

## 用法

该shell支持批处理和流式处理。启动后，两个不同的运行环境会自动预先绑定。使用 “benv” 和 “senv” 分别访问批处理和流式处理环境。

### DataSet API

以下示例将执行在Scala shell中执行wordcount程序：

~~~scala
Scala-Flink> val text = benv.fromElements(
  "To be, or not to be,--that is the question:--",
  "Whether 'tis nobler in the mind to suffer",
  "The slings and arrows of outrageous fortune",
  "Or to take arms against a sea of troubles,")
Scala-Flink> val counts = text
    .flatMap { _.toLowerCase.split("\\W+") }
    .map { (_, 1) }.groupBy(0).sum(1)
Scala-Flink> counts.print()
~~~

print（）命令会自动将指定的任务发送给JobManager执行，并在终端中显示计算结果。

可以将结果写入文件。但是，在这种情况下，您需要在运行程序时调用execute：

~~~scala
Scala-Flink> benv.execute("MyProgram")
~~~

### DataStream API

与上面的批处理程序类似，我们可以通过DataStream API执行流式处理程序：

~~~scala
Scala-Flink> val textStreaming = senv.fromElements(
  "To be, or not to be,--that is the question:--",
  "Whether 'tis nobler in the mind to suffer",
  "The slings and arrows of outrageous fortune",
  "Or to take arms against a sea of troubles,")
Scala-Flink> val countsStreaming = textStreaming
    .flatMap { _.toLowerCase.split("\\W+") }
    .map { (_, 1) }.keyBy(0).sum(1)
Scala-Flink> countsStreaming.print()
Scala-Flink> senv.execute("Streaming Wordcount")
~~~

请注意，在运行流处理程序时，打印操作不被直接触发。

Flink Shell带有命令历史记录和自动完成功能。

## 添加外部依赖包

可以将外部类路径添加到Scala-shell。这些将在调用execute时自动与您的shell程序一起发送到Jobmanager。

使用参数 `-a <path/to/jar.jar>` 或者 `--addclasspath <path/to/jar.jar>` 来加载其他类。

~~~bash
bin/start-scala-shell.sh [local | remote <host> <port> | yarn] --addclasspath <path/to/jar.jar>
~~~


## 开始

要了解Scala Shell提供的参数，请使用

~~~bash
bin/start-scala-shell.sh --help
~~~

### 本地模式

要使用 Flink 集成的 Scala shell ，只需执行：

~~~bash
bin/start-scala-shell.sh local
~~~

### Remote

要在运行的集群中使用它，请使用关键字 `remote` 启动scala shell，并提供JobManager的主机名和端口：

~~~bash
bin/start-scala-shell.sh remote <hostname> <portnumber>
~~~

### Yarn Scala Shell cluster

Flink 内置的 shell 可以将 Flink 部署到 YARN 集群，并且只可以通过 shell 来部署。YARN容器的数量可以通过参数`-n <arg>`来控制。

Flink 内置的 shell 在 YARN 上可以部署新的 Flink 集群并连接到该集群。您还可以为YARN群集指定参数，例如JobManager的内存，YARN应用程序的名称等。

例如，要通过Scala Shell启动一个使用两个 TaskManagers 的 YARN 集群，请使用以下命令：

~~~bash
 bin/start-scala-shell.sh yarn -n 2
~~~

对于所有其他选项，请参阅底部的完整参考。

### Yarn Session

如果以前使用Flink Yarn Session部署了Flink群集，则可以使用以下命令将Scala Shell与其连接：

~~~bash
 bin/start-scala-shell.sh yarn
~~~


## 完整参考

~~~bash
Flink Scala Shell
Usage: start-scala-shell.sh [local|remote|yarn] [options] <args>...

Command: local [options]
Starts Flink scala shell with a local Flink cluster
  -a <path/to/jar> | --addclasspath <path/to/jar>
        Specifies additional jars to be used in Flink
Command: remote [options] <host> <port>
Starts Flink scala shell connecting to a remote cluster
  <host>
        Remote host name as string
  <port>
        Remote port as integer

  -a <path/to/jar> | --addclasspath <path/to/jar>
        Specifies additional jars to be used in Flink
Command: yarn [options]
Starts Flink scala shell connecting to a yarn cluster
  -n arg | --container arg
        Number of YARN container to allocate (= Number of TaskManagers)
  -jm arg | --jobManagerMemory arg
        Memory for JobManager container [in MB]
  -nm <value> | --name <value>
        Set a custom name for the application on YARN
  -qu <arg> | --queue <arg>
        Specifies YARN queue
  -s <arg> | --slots <arg>
        Number of slots per TaskManager
  -tm <arg> | --taskManagerMemory <arg>
        Memory per TaskManager container [in MB]
  -a <path/to/jar> | --addclasspath <path/to/jar>
        Specifies additional jars to be used in Flink
  --configDir <value>
        The configuration directory.
  -h | --help
        Prints this usage text
~~~

{% top %}

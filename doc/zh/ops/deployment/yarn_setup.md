---
title:  "YARN Setup"
nav-title: YARN
nav-parent_id: deployment
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

* This will be replaced by the TOC
{:toc}

## 快速开始

### 在YARN上启动一个长时间运行的Flink群集

启动一个拥有4个Task Manager的yarn session（每个Task Manager都有4GB 堆内存）:

~~~bash
# get the hadoop2 package from the Flink download page at
# {{ site.download_url }}
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/yarn-session.sh -n 4 -jm 1024 -tm 4096
~~~

特别指出，`-s`参数表示每个Task Manager上可用的处理槽（processing slot）数量。我们建议将插槽数量设置为每台机器的处理器数量。

一旦会话被启动，您可以使用`./bin/flink`工具向群集提交作业。

### 在YARN上运行一个Flink任务

~~~bash
# get the hadoop2 package from the Flink download page at
# {{ site.download_url }}
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/flink run -m yarn-cluster -yn 4 -yjm 1024 -ytm 4096 ./examples/batch/WordCount.jar
~~~

## Flink YARN会话

Apache [Hadoop YARN](http://hadoop.apache.org/) 是一个集群资源管理框架。它允许在群集上运行多种分布式应用程序。Flink可以和其他应用程序一起在YARN上运行。如果已经启动了YARN，用户就不需再启动或安装任何东西。

**要求**

- Apache Hadoop 版本至少2.2
- •	HDFS（Hadoop分布式文件系统）（或其他由Hadoop支持的分布式文件系统）

如果您使用Flink YARN客户端时遇到问题，请查看 [FAQ section](http://flink.apache.org/faq.html#yarn-deployment).

### 启动Flink会话

跟随以下介绍学习怎样在你的yran集群中启动一个Flink会话。

一个会话将启动所有必需的Flink服务（JobManager和TaskManagers），这样您可以将程序提交到群集。请注意，一个会话可以运行多个程序。

#### 下载Flink

从 [download page]({{ site.download_url }})下载Hadoop版本大于2的Flink软件包。它包含了所需的文件。

使用以下命令解压软件包：

~~~bash
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{site.version }}/
~~~

#### 启动一个会话

使用以下命令启动一个会话

~~~bash
./bin/yarn-session.sh
~~~

该命令概述如下：

~~~bash
Usage:
  必需参数:
     -n,--container <arg>   YARN容器数目(=Task Manager的个数)
   Optional
     -D <arg>                        动态属性
     -d,--detached                   启动分离（提交job的机器与yarn集群分离）
     -jm,--jobManagerMemory <arg>    JobManager Container内存大小[in MB]
     -nm,--name                      为应用程序在Flink上设置一个自定义名称
     -q,--query                      展示yarn的可用资源，内存和核数 (memory, cores)
     -qu,--queue <arg>               指定YARN队列
     -s,--slots <arg>                每个TaskManager的处理槽数
     -tm,--taskManagerMemory <arg>   每个TaskManager Container的内存大小 [in MB]
     -z,--zookeeperNamespace <arg>   在高可用模式下，命名空间为zookeeper创建子路径
~~~

请注意，客户端要求将`YARN_CONF_DIR`或`HADOOP_CONF_DIR` 环境变量设置为读取YARN和HDFS配置。

**示例:** 以下命令以分配10个任务管理器，每个任务管理器具有8 GB的内存和32个处理插槽：

~~~bash
./bin/yarn-session.sh -n 10 -tm 8192 -s 32
~~~

系统将使用`conf/flink-conf.yaml`中的配置。如果你想更改一些配置，请参考我们的[configuration guide]({{ site.baseurl }}/ops/config.html) 。

YARN上的Flink将重写以下配置参数`jobmanager.rpc.address`（因为JobManager总是分配在不同的机器上），`taskmanager.tmp.dirs` （我们使用YARN给出的tmp目录）以及 `parallelism.default` 如果指定了插槽数量。

如果您不想更改配置文件来设置配置参数，则可以选择通过`-D`标志传递动态属性。你可以这样传递参数：`-Dfs.overwrite-files=true -Dtaskmanager.network.memory.min=536346624`.

示例请求启动了11个容器（尽管只请求了10个容器），因为ApplicationMaster和Job Manager需要一个额外的容器。

一旦将Flink部署到YARN群集中，它就会显示Job Manager间连接的详细信息。

通过停止unix进程（使用CTRL + C）或在客户端输入“stop”来停止YARN会话。

如果群集上有足够的资源可用，YARN上的Flink将仅启动所有请求的容器。大多数YARN调度程序都会记录容器的请求内存，某些调度程序也会记录CPU核心数量。默认情况下，CUP核心数量等于处理槽 (`-s`) 参数。 `yarn.containers.vcores` 允许用自定义CUP核心数量。

#### 隔离YARN会话

如果您不想让Flink YARN客户端始终运行，那么也可以启动隔离YARN会话来达到目的。该参数被称为 `-d`或`--detached`。

在这种情况下，Flink YARN客户端只会将Flink提交给群集，然后关闭与集群的连接。请注意，在这种情况下，无法使用Flink来停止YARN会话。

使用YARN公用程序 (`yarn application -kill <appId>`) 停止YARN会话。

#### 关联现有会话

使用以下命令启动一个会话

~~~bash
./bin/yarn-session.sh
~~~

该命令将向您显示以下概述：

~~~bash
Usage:
   必须参数
     -id,--applicationId <yarnAppId> YARN application Id
~~~

如前所述，`YARN_CONF_DIR`或`HADOOP_CONF_DIR` 必须将环境变量设置为读取YARN和HDFS。

**示例:** 通过以下命令关联正在运行的Flink YARN会话 `application_1463870264508_0029`:

~~~bash
./bin/yarn-session.sh -id application_1463870264508_0029
~~~

连接到一个正在运行的会话，使用YARN ResourceManager来决定Job Manager RPC端口。

通过停止unix进程（使用CTRL + C）或在客户端输入“stop”来停止YARN会话。

### 提交Job到Flink

使用以下命令将Flink程序提交给YARN群集：

~~~bash
./bin/flink
~~~

请参阅[command-line client]({{ site.baseurl }}/ops/cli.html).

该命令将显示如下帮助菜单：

~~~bash
[...]
操作"run"编译和运行一个程序。

  语法: run [OPTIONS] <jar-file> <arguments>
  "run" 操作参数:
     -c,--class <classname>           程序入口类 ("main"方法或"getPlan()" 方法. 只有在jar文件没有指定主类时才需要指定该参数。
     -m,--jobmanager <host:port>      连接JobManager (master) 的地址. 使用此配置指定连接参数，好于在配置文件中指定。
     -p,--parallelism <parallelism>   程序运行的并行度. 此参数将覆盖配置文件中指定的参数。
~~~

使用*run* 将Job提交给YARN。客户端可以确定JobManager的地址。在极少数情况下，您也可以使用`-m` 参数传递JobManager地址。JobManager地址在YARN控制台可见。

**示例**

~~~bash
wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt
hadoop fs -copyFromLocal LICENSE-2.0.txt hdfs:/// ...
./bin/flink run ./examples/batch/WordCount.jar \
        hdfs:///..../LICENSE-2.0.txt hdfs:///.../wordcount-result.txt
~~~

如果出现以下错误，请确保所有TaskManagers已启动：

~~~bash
Exception in thread "main" org.apache.flink.compiler.CompilerException:
    Available instances could not be determined from job manager: Connection timed out.
~~~

您可以在JobManager Web接口中查看TaskManager的数量。该接口的地址将打印在YARN会话控制台中。

如果TaskManager在一分钟内不显示，那么你应该在日志文件中检查错误。


## 在YARN上运行一个Flink任务

以上文档介绍了如何在Hadoop YARN环境中启动Flink集群。也可以在YARN上启动Flink来执行单个job。

请注意，客户端需要通过`-yn`设置TaskManagers的数量。

***示例:***

~~~bash
./bin/flink run -m yarn-cluster -yn 2 ./examples/batch/WordCount.jar
~~~

YARN会话的命令行参数也可用于`./bin/flink`工具。它们的前缀是一个 `y`或`yarn` (对于长参数选项）。

注意：您可以通过设置`FLINK_CONF_DIR`. 环境变量来为每个job使用不同的配置目录。要使用此功能， `conf` 从Flink分配和修改，例如，每个作业的日志设置。

注意：在YARN集群中，结合 `-m yarn-cluster` 和隔离YARN会话 (`-yd`)命令“可以将一个Flink作业焚毁和忘记”。在这种情况下，您的应用程序不会从ExecutionEnvironment.execute()的调用中获得任何累加器结果或异常！

### 使用jars&Classpath

默认情况下，Flink将在运行单个作业时将用户jar添加到系统类路径中。这个行为可以用`yarn.per-job-cluster.include-user-jar` 参数来控制。

当设置此参数为`DISABLED` 时，Flink会将该jar包含在用户类路径中。

可以通过将参数设置为以下之一来控制类路径中的用户jar位置：

- `ORDER`: （默认）根据词典顺序将jar添加到系统类路径。
- `FIRST`: 将jar添加到系统类路径的开头。
- `LAST`: 将jar添加到系统类路径的末尾。

## Flink在YARN上的恢复行为

Flink的YARN客户端具有以下配置参数来控制容器故障时的行为。这些参数可以从`conf/flink-conf.yaml` 或使用 `-D` 参数启动YARN会话时进行设置。

- `yarn.reallocate-failed`: 该参数控制Flink是否应该重新分配失败的TaskManager容器。默认值：true
- `yarn.maximum-failed-containers`: ApplicationMaster接受的失败容器的最大数量，直到YARN会话失败。默认值：最初请求的TaskManagers (`-n`)的数量。
- `yarn.application-attempts`: ApplicationMaster（+它的TaskManager容器）尝试的次数。如果此值设置为1（默认值），则当应用程序主控失败时，整个YARN会话将失败。在YARN中指定更大的值以便重新启动ApplicationMaster。

## 调试失败的YARN会话

Flink YARN会话部署失败的原因有很多。配置错误的Hadoop设置（HDFS权限，YARN配置），版本不兼容（Flink运行在vanilla Hadoop上，却依赖Cloudera Hadoop）或其他错误。

### 日志文件

在部署期间Flink YARN会话失败的情况下，用户必须依赖Hadoop YARN的日志功能。 [YARN log aggregation](http://hortonworks.com/blog/simplifying-user-logs-management-and-access-in-yarn/)（YARN日志聚合）是最有用的功能。
要启用它，用户必须在`yarn-site.xml`文件将`yarn.log-aggregation-enable`属性设置`true`。一旦启用，用户可以使用以下命令来检索YARN会话的所有日志文件（包括失败的会话）。

~~~
yarn logs -applicationId <application ID>
~~~

请注意，从会话结束后到日志显示出来需要几秒钟时间。

### YARN客户端控制台和Web界面

如果运行期间发生错误（例如，如果TaskManager在一段时间后停止工作），Flink YARN客户端还会在终端中输出错误消息。

除此之外，还有YARN资源管理器Web界面（默认情况下在端口8088上），资源管理器Web界面的端口由`yarn.resourcemanager.webapp.address` 参数值决定。

它允许访问运行YARN应用程序的日志文件并显示失败应用程序的诊断信息。

## 为特定的Hadoop版本构建YARN客户端

使用Hortonworks，Cloudera或MapR等公司的Hadoop发行版的用户可能必须针对其特定的Hadoop（HDFS）和YARN版本构建Flink。请阅读 [build instructions]({{ site.baseurl }}/start/building.html) 以获取更多详细信息。

## 在防火墙内的YARN上运行Flink

一些YARN群集使用防火墙来控制群集与网络其余部分之间的网络传输。在这些设置下，Flink作业只能通过集群网络（防火墙后面）提交Job到YARN会话。如果这在生产环境下不可行，Flink允许为所有相关服务配置一个端口范围。在这些端口范围下，用户可以跨越防火墙提交job到Flink。

目前，需要两项服务才能提交工作：

 * JobManager（YARN中的ApplicationMaster)
 * 在JobManager中运行的BlobServer.

当提交一个job到Flink，BlobServer将会分发用户代码中的jars给所有工作节点（Task Manager）， Job Manager接收job本身并触发执行。

用于指定端口的两个配置参数如下：

 * `yarn.application-master.port`
 * `blob.server.port`

这两个配置选项接受单个端口（例如：“50010”），范围（“50000-50025”）或两者的组合（“50010,50011,50020-50025,50050-50075”）。

(Hadoop使用类似的机制，配置参数是`yarn.app.mapreduce.am.job.client.port-range`.)

## 背景/内部

本节简要介绍Flink和YARN如何交互。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.4/fig/FlinkOnYarn.svg" class="img-responsive">

YARN客户端需要访问Hadoop配置才能连接到YARN资源管理器和HDFS。它使用以下策略确定Hadoop配置：

* 测试 `YARN_CONF_DIR`, `HADOOP_CONF_DIR`或`HADOOP_CONF_PATH`是否已设置（按顺序）。如果设置了其中一个变量，将被用于读取配置。
* 如果上述策略失败（在正确的YARN设置中不应该这样），客户端使用`HADOOP_HOME`环境变量。如果已设置该环境变量，客户端将尝试访问`$HADOOP_HOME/etc/hadoop` (Hadoop 2) 和`$HADOOP_HOME/conf` (Hadoop 1).

当启动新的Flink YARN会话时，客户端首先检查请求的资源（容器和内存）是否可用。之后，它将Flink配置文件和jar文件上传到HDFS（步骤1）。

下一步客户端请求一个YARN容器（步骤2）来启动*ApplicationMaster*（步骤3）。由于客户端将配置和jar文件注册为容器的资源，因此运行在该特定机器上的YARN的NodeManager将负责准备容器（例如，下载文件）。一旦完成上述内容，*ApplicationMaster*（AM）就会启动。

*JobManager*和AM在同一容器中运行。一旦成功启动，AM知道JobManager（它自己的主机）的地址。Job Manager为TaskManagers生成一个新的Flink配置文件（以便task可以连接到JobManager）。该文件也被上传到HDFS。此外，AM容器还提供Flink的Web界面。YARN代码分配的所有端口都是*临时端口*。这允许用户并行执行多个Flink YARN会话。

之后，AM开始为Flink的TaskManagers分配容器，它将从HDFS下载jar文件和修改后的配置。完成这些步骤后，Flink就会设置并准备接收job。

{% top %}

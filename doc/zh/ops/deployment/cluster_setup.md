---
title: "Standalone Cluster"
nav-parent_id: deployment
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

本章介绍了如何在静态（但可能是异构）集群上以*完全分布式*的方式运行Flink。
* This will be replaced by the TOC
[toc]

## 需求

### 软件需求

Flink可以在所有*类UNIX环境*中运行，例如：**Linux**,**Mac OS X**和**Cygwin**(适用于Windows)，并且集群由**一个master节点**和**一个或多个worker节点**组成。在开始设置系统前，请确保一下软件已经安装到了**每一个节点**上:
- **Java 1.8.x** 或更高版本，
- **ssh** (为了确保Flink远程管理组件脚本可以使用，必须先运行ssh命令)

如果您的集群不满足这些软件需求，请先安装或升级对应软件。

在所有集群节点上拥有 __无密码SSH__ 和
__相同的目录结构__ 将允许您使用我们的脚本控制所有内容。

{% top %}

### `JAVA_HOME` 配置

Flink需要在master节点和所有worker节点上设置`JAVA_HOME`环境变量，并指向Java安装目录。

你可以通过设置`conf/flink-conf.yaml`配置文件中的`env.java.home`变量指定该值。

{% top %}

## Flink安装

转到[下载页面](http://flink.apache.org/downloads.html) 并获取准备运行的软件包。确保选择与您的**Hadoop版本**匹配的Flink软件包。如果您不打算使用任何hadoop版本，可以选择任意版本的Flink软件包。.

下载最新版本压缩包后，将其复制到您的主节点并解压：

~~~bash
tar xzf flink-*.tgz
cd flink-*
~~~

### 配置Flink

在提取系统文件后，您需要通过编辑*conf/flink-conf.yaml*来为集群配置Flink。

通过设置`jobmanager.rpc.address`来指定master节点。您还应该通过设置`jobmanager.heap.mb`和`taskmanager.heap.mb`来定义每个节点上JVM允许分配的主内存的最大值。

这些值的单位为MB。如果某些worker节点有更多的要分配给Flink系统的主内存，您可以通过该节点的`FLINK_TM_HEAP`环境变量来覆盖默认值。

最后，您必须提供群集中作为worker节点的节点列表。类似于HDFS配置，编辑*conf/slaves*文件并输入每个worker节点的IP/主机名。每个工作节点稍后将运行一个TaskManager。

下面以三个节点的集群为例说明（IP地址从 _10.0.0.1_ 到 _10.0.0.3_ 和主机名 _master_ ， _worker1_ ， _worker2_ ）配置文件的内容（配置文件在所有机器上的相同路径下）：

<div class="row">
  <div class="col-md-6 text-center">
    <img src="http://doc.flink-china.org/1.2.0/page/img/quickstart_cluster.png" style="width: 60%">
  </div>
<div class="col-md-6">
  <div class="row">
    <p class="lead text-center">
      /path/to/<strong>flink/conf/<br>flink-conf.yaml</strong>
    <pre>jobmanager.rpc.address: 10.0.0.1</pre>
    </p>
  </div>
<div class="row" style="margin-top: 1em;">
  <p class="lead text-center">
    /path/to/<strong>flink/<br>conf/slaves</strong>
  <pre>
10.0.0.2
10.0.0.3</pre>
  </p>
</div>
</div>
</div>

在每一个worker节点的相同路径下的Flink目录必须都是可用的。您可以使用共享NFS目录，或者将Flink目录复制到每一个worker节点。

有关详细信息和其他配置选项，请参阅[configuration page](../config.html) 。

特别是,

 * 每个JobManager (`jobmanager.heap.mb`)的可用内存量，
 * 每个TaskManager (`taskmanager.heap.mb`)的可用内存量，
 * 每台机器(`taskmanager.numberOfTaskSlots`)的可用CPU数据，
 * 集群中CPU的总数 (`parallelism.default`) 
 * 临时目录(`taskmanager.tmp.dirs`)

是非常重要的配置。

{% top %}

### 启动Flink

以下脚本在本地节点上启动JobManager并通过SSH连接到*slaves*文件中列出的所有worker节点，以便在每个worker节点上启动TaskManager。现在你的Flink系统已经启动并正在运行。本地节点上运行的JobManager将通过配置的RPC端口接收任务。

假设您位于主节点上并位于Flink目录中：

~~~bash
bin/start-cluster.sh
~~~

可以通过 `stop-cluster.sh`脚本关闭Flink。

{% top %}

### 为集群添加JobManager/TaskManager实例

You can add both JobManager and TaskManager instances to your running cluster with the `bin/jobmanager.sh` and `bin/taskmanager.sh` scripts.
您可以使用`bin/jobmanager.sh`和`bin/taskmanager.sh`脚本将JobManager和TaskManager实例添加到正在运行的群集中。

#### 添加JobManager

~~~bash
bin/jobmanager.sh ((start|start-foreground) cluster)|stop|stop-all
~~~

#### 添加TaskManager

~~~bash
bin/taskmanager.sh start|start-foreground|stop|stop-all
~~~

确保在要启动/停止相应实例的主机上调用这些脚本。

{% top %}

---
title:  "Mesos Setup"
nav-title: Mesos
nav-parent_id: deployment
nav-pos: 3
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

## 背景

Mesos实现由两个组件组成：应用程序Master和Worker。 这些worker是简单的TaskManagers，它们由应用程序master设置的环境参数化。Mesos实现中最复杂的组件是应用程序master。 应用程序master当前拥有以下组件：

### Mesos调度程序 

调度程序负责使用Mesos注册框架，请求资源并启动worker节点。 调度器不断需要向Mesos报告，以确保框架处于健康状态。 为了验证群集的健康状况，调度程序监控worker节点并将其标记为失败，在必要时重新启动它们。

Flink的Mesos调度程序目前本身不具备高可用性。 但是，它在Zookeeper中保存了关于其状态的所有必要信息（例如配置，worker列表）。 在出现故障时，它依靠外部系统启动新的调度程序。 调度程序将再次向Mesos注册并进入协调(reconciliation)阶段。 在协调阶段，调度程序会收到正在运行的worker节点列表。 它将这些与Zookeeper恢复的信息进行匹配，并确保群集恢复到发生故障之前的正常状态。

### 工件服务器

工件服务器负责为worker节点提供资源。 资源可以是从Flink二进制文件到共享秘密或配置文件的任何内容。 例如，在非集装箱环境中，工件服务器将提供Flink二进制文件。 将提供哪些文件取决于所使用的配置覆盖。

### Flink的JobManager和Web界面

Mesos调度程序当前驻留在JobManager中，但将在未来版本中独立于JobManager启动（参见
[FLIP-6](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65147077))。提议的修改还将增加一个Dipsatcher组件，该组件将成为作业提交和监控的核心。

### 启动脚本和配置覆盖

启动脚本提供了一种配置和启动应用程序master的方法。 所有进一步的配置由worker节点继承。 这是通过使用配置覆盖来实现的。 配置叠加提供了一种方法来从运行的worker节点的环境变量和配置文件推断配置的能力。


## DC/OS

本部分涉及 [DC/OS](https://dcos.io) ，它是具有复杂应用程序管理层的Mesos发行版。 它预先安装了Marathon，这是一种监控应用程序并在发生故障时维持其状态的服务。

如果您没有正在运行的DC / OS群集，请按照
[instructions on how to install DC/OS on the official website](https://dcos.io/install/)。

一旦拥有DC/OS集群，您可以通过DC/OS Universe安装Flink。 在搜索提示中，只需搜索Flink。 或者，您可以使用DC/OS CLI：

    dcos package install flink

更多信息可以在
[DC/OS examples documentation](https://github.com/dcos/examples/tree/master/1.8/flink)。


## 未包含DC/OS的Mesos 

您也可以在没有DC / OS的情况下运行Mesos。

### 安装Mesos

请按照说明如何在官方网站上设置Mesos [instructions on how to setup Mesos on the official website](http://mesos.apache.org/documentation/latest/getting-started/).

安装后，您必须通过创建文件`MESOS_HOME/etc/mesos/masters`和`MESOS_HOME/etc/mesos/slaves`.
来配置主节点和代理节点集。 这些文件在每行中包含一个单独的主机名，各自的组件将在其上启动（假定SSH能够访问这些节点）。

接下来，您必须创建`MESOS_HOME/etc/mesos/mesos-master-env.sh` 或使用在同一目录中找到的模板。 在这个文件中，你必须定义

    export MESOS_work_dir=WORK_DIRECTORY

建议取消注释

    export MESOS_log_dir=LOGGING_DIRECTORY


为了配置Mesos代理，您必须创建`MESOS_HOME/etc/mesos/mesos-agent-env.sh`或使用在同一目录中找到的模板。 
你必须配置

    export MESOS_master=MASTER_HOSTNAME:MASTER_PORT

并取消注释

    export MESOS_log_dir=LOGGING_DIRECTORY
    export MESOS_work_dir=WORK_DIRECTORY

#### Mesos Library

为了使用Mesos运行Java应用程序，您必须在Linux上配置系统环境变量：`MESOS_NATIVE_JAVA_LIBRARY=MESOS_HOME/lib/libmesos.so`。
在Mac OS X中，您必须配置系统环境变量：`MESOS_NATIVE_JAVA_LIBRARY=MESOS_HOME/lib/libmesos.dylib`.

#### 部署Mesos

为了启动您的mesos群集，请使用部署脚本`MESOS_HOME/sbin/mesos-start-cluster.sh`.
为了停止您的mesos群集，请使用部署脚本`MESOS_HOME/sbin/mesos-stop-cluster.sh`.
有关部署脚本的更多信息可以在 [here](http://mesos.apache.org/documentation/latest/deploy-scripts/)找到。

### 安装Marathon

或者，您也可以[install Marathon](https://mesosphere.github.io/marathon/docs/) ，这将在高可用性（HA）模式下运行Flink。

### 预安装Flink vs Docker / Mesos容器

您可以在您的所有Mesos Master和Agent节点上安装Flink。 您也可以在部署过程中从Flink网站获取二进制文件，并在启动应用程序主控之前应用您的自定义配置。 更方便和更易于维护的方法是使用Docker容器来管理Flink二进制文件和配置。

这通过以下配置条目进行控制：

    mesos.resourcemanager.tasks.container.type: mesos _or_ docker

如果设置为“docker”，请指定镜像名称：

    mesos.resourcemanager.tasks.container.image.name: image_name


### 单机

在Flink发行版的`/bin`目录中，您可以找到两个启动脚本来管理Mesos群集中的Flink进程

1. `mesos-appmaster.sh`
   这将启动注册Mesos调度程序的Mesos应用程序master。 它也负责启动worker节点。

2. `mesos-taskmanager.sh`
   工作进程的入口点。 您不需要明确执行此脚本。 它由Mesos worker点自动启动，以便启动一个新的TaskManager。

为了运行`mesos-appmaster.sh`脚本，您必须在`flink-conf.yaml`定义`mesos.master`或将其通过`-Dmesos.master=...`传递给Java进程。 此外，您应该通过`mesos.initial-tasks`定义由Mesos启动的任务管理器的数量。 该值也可以在`flink-conf.yaml`定义或作为Java属性传递。

执行`mesos-appmaster.sh`，它将在执行脚本的机器上创建一个job manager。 与此相反，task managers将作为Mesos集群中的Mesos任务运行。

#### 通用配置

通过传递给Mesos应用程序主机的Java属性，可以完全参数化Mesos应用程序。 这也允许指定通用的Flink配置参数。 
例如：

    bin/mesos-appmaster.sh \
        -Dmesos.master=master.foobar.org:5050 \
        -Djobmanager.heap.mb=1024 \
        -Djobmanager.rpc.port=6123 \
        -Djobmanager.web.port=8081 \
        -Dmesos.initial-tasks=10 \
        -Dmesos.resourcemanager.tasks.mem=4096 \
        -Dtaskmanager.heap.mb=3500 \
        -Dtaskmanager.numberOfTaskSlots=2 \
        -Dparallelism.default=10


### 高可用性

您需要运行Marathon或Apache Aurora等服务，以便在发生节点或进程故障时重新启动Flink主进程。 此外，Zookeeper需要按照 [High Availability section of the Flink docs]({{ site.baseurl }}/ops/jobmanager_high_availability.html)所述进行配置。

为了使任务协调正常工作，还请将`high-availability.zookeeper.path.mesos-workers`设置为有效的Zookeeper路径。

#### Marathon

需要设置Marathon 来启动`bin/mesos-appmaster.sh`脚本。 特别是，它还应该调整Flink群集的任何配置参数。

以下是Marathon的示例配置：

    {
        "id": "flink",
        "cmd": "$FLINK_HOME/bin/mesos-appmaster.sh -Djobmanager.heap.mb=1024 -Djobmanager.rpc.port=6123 -Djobmanager.web.port=8081 -Dmesos.initial-tasks=1 -Dmesos.resourcemanager.tasks.mem=1024 -Dtaskmanager.heap.mb=1024 -Dtaskmanager.numberOfTaskSlots=2 -Dparallelism.default=2 -Dmesos.resourcemanager.tasks.cpus=1",
        "cpus": 1.0,
        "mem": 1024
    }

当通过Marathon运行Flink时，包括作业管理器的整个Flink集群将作为Mesos集群中的Mesos任务运行。

### 配置参数

`mesos.initial-tasks`: 启动主服务器时启动的初始worker（ **DEFAULT**： 集群启动时指定的worker数量）。

`mesos.constraints.hard.hostattribute`: 基于代理属性的mesos任务放置的约束(**DEFAULT**：None)。
以逗号分隔的键列表:值对对应于目标mesos代理所暴露的属性。例子： `az:eu-west-1a,series:t2`

`mesos.maximum-failed-tasks`: 集群失败前的最大失败worker数(**DEFAULT**: 初始worker数).
可以设置为-1来禁用此功能。

`mesos.master`: Mesos   master的URL。该值应以下列形式之一:：

* `host:port`
* `zk://host1:port1,host2:port2,.../path`
* `zk://username:password@host1:port1,host2:port2,.../path`
* `file:///path/to/file`

`mesos.failover-timeout`: Mesos调度器的故障转移超时，之后运行任务自动关闭(**DEFAULT:** 600).

`mesos.resourcemanager.artifactserver.port`:定义Mesos工件服务器端口的配置参数。将端口设置为0将让操作系统选择一个可用的端口。

`mesos.resourcemanager.framework.name`: Mesos框架名称(**DEFAULT:** Flink)

`mesos.resourcemanager.framework.role`: Mesos框架角色定义 (**DEFAULT:** *)

`high-availability.zookeeper.path.mesos-workers`: 保存Mesos worker信息的ZooKeeper根路径。

`mesos.resourcemanager.framework.principal`: Mesos框架主体(**NO DEFAULT**)

`mesos.resourcemanager.framework.secret`: Mesos框架加密(**NO DEFAULT**)

`mesos.resourcemanager.framework.user`: Mesos框架用户 (**DEFAULT:**"")

`mesos.resourcemanager.artifactserver.ssl.enabled`:为Flink工件服务器启用SSL(**DEFAULT**: true)。注意，`security.ssl.enabled`也需要设置为`true`来启用加密。

`mesos.resourcemanager.tasks.mem`: 分配给Mesos worker的 内存，单位MB(**DEFAULT**: 1024)

`mesos.resourcemanager.tasks.cpus`: 分配给Mesos worker的CPU(**DEFAULT**: 0.0)

`mesos.resourcemanager.tasks.container.type`: 使用的集装箱化类型:“mesos”或“docker”(默认:mesos);

`mesos.resourcemanager.tasks.container.image.name`:用于容器的镜像名称(**NO DEFAULT**)

`mesos.resourcemanager.tasks.container.volumes`: 一个逗号分隔的列表`[host_path:]`container_path`[:RO|RW]`. 这允许在您的容器中安装额外的卷。 (**NO DEFAULT**)

`mesos.resourcemanager.tasks.hostname`: 可选值用来定义TaskManager的主机名。模式`_TASK_`被Mesos任务的实际id替换。这可以用来配置TaskManager 来使用Mesos DNS(例如： `_TASK_.flink-service.mesos`) 进行名称查找。 (**NO DEFAULT**)

`mesos.resourcemanager.tasks.bootstrap-cmd`: 在TaskManager 启动之前执行的命令 (**NO DEFAULT**).

{% top %}

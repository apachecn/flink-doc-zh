---
title:  "Docker Setup"
nav-title: Docker
nav-parent_id: deployment
nav-pos: 4
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

[Docker](https://www.docker.com) 是一个流行的容器运行环境。Docker Hub上提供了Apache Flink的正式Docker镜像，可直接使用或扩展以更好地集成到生产环境中。

* This will be replaced by the TOC
{:toc}

## 官方Docker镜像

该 [official Docker repository](https://hub.docker.com/_/flink/)(官方Docker仓库)托管在Docker Hub上，并且提供的Flink镜像为1.2.1或更高版本。

支持Hadoop和Scala的每种组合的镜像都是可用的，并且为了方便起见还提供了标记别名功能。

For example, the following aliases can be used: *(`1.2.y` indicates the latest
release of Flink 1.2)*
例如，下面的别名可以使用：*（1.2.y表示Flink 1.2的最新版本）*

* `flink:latest` →
`flink:<latest-flink>-hadoop<latest-hadoop>-scala_<latest-scala>`
* `flink:1.2` → `flink:1.2.y-hadoop27-scala_2.11`
* `flink:1.2.1-scala_2.10` → `flink:1.2.1-hadoop27-scala_2.10`
* `flink:1.2-hadoop26` → `flink:1.2.y-hadoop26-scala_2.11`

<!-- NOTE: uncomment when docker-flink/docker-flink/issues/14 is resolved. -->
<!--
Additionally, images based on Alpine Linux are available. Reference them by
appending `-alpine` to the tag. For the Alpine version of `flink:latest`, use
`flink:alpine`.

For example:

* `flink:alpine`
* `flink:1.2.1-alpine`
* `flink:1.2-scala_2.10-alpine`
-->

**注意:** Docker镜像是由个人在力所能及的基础上提供的社区项目。它们不是由Apache Flink PMC正式发布的。

## Flink使用Docker Compose

[Docker Compose](https://docs.docker.com/compose/)是一种在本地运行一组Docker容器的简便方法。

GitHub上提供了一个 [example config file](https://github.com/docker-flink/examples/blob/master/docker-compose.yml)
（示例配置文件）。

### 用法

* 在前台启动群集

        docker-compose up

* 在后台启动群集

        docker-compose up -d

* 将集群向上或向下扩展到*N*个TaskManagers

        docker-compose scale taskmanager=<N>

群集运行时，您可以访问[http://localhost:8081](http://localhost:8081) Web UI 并提交job。

要通过命令行提交job，您必须将JAR复制到Jobmanager容器并从那里提交job。

例如：

{% raw %}
    $ JOBMANAGER_CONTAINER=$(docker ps --filter name=jobmanager --format={{.ID}})
    $ docker cp path/to/jar "$JOBMANAGER_CONTAINER":/job.jar
    $ docker exec -t -i "$JOBMANAGER_CONTAINER" flink run /job.jar
{% endraw %}

{% top %}

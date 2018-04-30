---
title: 从源代码构建 Flink
nav-parent_id: start
nav-pos: 20
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

本页介绍如何从源代码构建Flink {{ site.version }} 。

* This will be replaced by the TOC
{:toc}

## 构建 Flink

您需要通过源代码来构建 Flink ，可以通过 [下载release版本的源码]({{ site.download_url }}) 或者 [从 Git 仓库克隆]({{ site.github_url }})来获取源代码。

另外您需要 **Maven 3** 和 **JDK** (Java Development Kit). Flink 需要 **至少 Java 8版本** 来构建。

*注意:Maven 3.3.x可以构建Flink，但不会正确屏蔽某些依赖库，而 Maven 3.0.3可以。要构建单元测试，请使用Java 8u51或更高版本，以防止使用PowerMock的单元测试出现故障。*

从 git 克隆源码，请输入:

~~~bash
git clone {{ site.github_url }}
~~~

构建Flink最简单的方法是运行:

~~~bash
mvn clean install -DskipTests
~~~

这将使[Maven](http://maven.apache.org) (`mvn`) 首先删除所有现有的构建 (`clean`) 然后创建一个新的Flink二进制文件 (`install`)。

为了加快构建，您可以跳过 tests, checkstyle, 和 JavaDocs: `mvn clean install -DskipTests -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true`.

默认构建为Hadoop 2添加了Flink特定的JAR，以允许将Flink与HDFS和YARN一起使用。

## Shade 处理依赖

Flink 为了避免版本与使用这些库的不同版本的用户程序发生冲突，Flink 会将其使用的一些库用 [shade 插件](https://maven.apache.org/plugins/maven-shade-plugin/) 隐藏起来。其中包括 *Google Guava*, *Asm*, *Apache Curator*, *Apache HTTP Components*, *Netty*, and others 。

Shade 依赖处理机制最近在Maven中进行了更改，并且要求用户构建Flink的方式略有不同，具体取决于他们的 Maven 版本：

**Maven 3.0.x, 3.1.x, 和 3.2.x**
只需要在 Flink 根目录调用 `mvn clean install -DskipTests` 。

**Maven 3.3.x**
构建必须分两步完成：首先在根目录中，然后在分发项目中：

~~~bash
mvn clean install -DskipTests
cd flink-dist
mvn clean install
~~~

*注意:* 使用 `mvn --version` 来检查 maven 版本。

{% top %}

## Hadoop 版本

{% info %} 大多数用户不需要进行手动操作。 这个[下载叶片]({{ site.download_url }}) 包含了大多数 Hadoop 二进制软件包。

Flink依赖于[Apache Hadoop](http://hadoop.apache.org)的 HDFS 和 YARN。 有许多不同版本的Hadoop（来自上游项目和不同的Hadoop发行版）。如果您使用错误的版本组合，则可能会发生异常。

Flink 只支持2.4.0以上版本的 Hadoop，您也可以指定一个特定的Hadoop版本来构建:

~~~bash
mvn clean install -DskipTests -Dhadoop.version=2.6.1
~~~

### 特定提供商版本的 Hadoop

要针对特定​​供应商的Hadoop版本构建Flink，请使用命令：

~~~bash
mvn clean install -DskipTests -Pvendor-repos -Dhadoop.version=2.6.1-cdh5.0.0
~~~

`-Pvendor-repos` 提供一个[构建文件](http://maven.apache.org/guides/introduction/introduction-to-profiles.html),其中包含了主流的Hadoop厂商如Cloudera，Hortonworks或MAPR的Maven存储库。

{% top %}

## Scala 版本

{% info %} 只使用Java API和库的用户可以*忽略*此部分。

Flink 有使用 [Scala](http://scala-lang.org) 编写的API，库和运行时模块。Scala API和库的用户必须将 Flink 的 Scala 版本与其项目的 Scala 版本相匹配（因为 Scala 不严格向下兼容）。

目前Flink 1.4只使用Scala版本2.11构建。

我们正在致力于支持Scala 2.12，但Scala 2.12中的某些彻底的变化使得这是一个更为复杂的工作。详情请查看[JIRA 问题](https://issues.apache.org/jira/browse/FLINK-7811)的更新。

{% top %}

## 加密的文件系统

如果您的主目录已加密，则可能会遇到 `java.io.IOException: File name too long` 异常。一些加密的文件系统，如Ubuntu使用的encfs，不允许长文件名，这是导致此错误的原因。

解决方法是增加一个在引起错误的模块的 `pom.xml` 文件的编译器配置：

~~~xml
<args>
    <arg>-Xmax-classfile-name</arg>
    <arg>128</arg>
</args>
~~~

例如，如果错误出现在 `flink-yarn` 模块中，上面的代码应该添加在 `scala-maven-plugin` 下的 `<configuration>` 标签 。请参阅[这个问题](https://issues.apache.org/jira/browse/FLINK-2003)以获取更多信息。

{% top %}

---
title:  "集群执行"
nav-parent_id: batch
nav-pos: 12
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

Flink程序可以在许多机器的集群上分布运行。有两种方法可将程序发送到群集以供执行：

## 命令行Interface

您可以通过命令行 interface 提交打好包的程序(JARs)到一个cluster
(或者单机).

有关详情请参考 [Command Line Interface]({{ site.baseurl }}/ops/cli.html) 文档。

## 远程环境

远程环境允许您直接在群集上执行Flink Java程序。
远程环境指向要在其上执行程序的群集。

### Maven Dependency

If you are developing your program as a Maven project, you have to add the
`flink-clients` module using this dependency:
如果您的程序是Maven项目，则必须添加`flink-clients`模块依赖：

~~~xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>{{ site.version }}</version>
</dependency>
~~~

### Example

以下说明使用'RemoteEnvironment'：

~~~java
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment
        .createRemoteEnvironment("flink-master", 6123, "/home/user/udfs.jar");

    DataSet<String> data = env.readTextFile("hdfs://path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("hdfs://path/to/result");

    env.execute();
}
~~~

Note that the program contains custom user code and hence requires a JAR file with
the classes of the code attached. The constructor of the remote environment
takes the path(s) to the JAR file(s).
请注意，该程序包含自定义用户代码，因此需要代码依赖的JAR文件。远程环境的构造函数需指定JAR文件的路径。

{% top %}

---
title:  "Connectors"
nav-parent_id: batch
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

* TOC
{:toc}

## 从文件系统里读取

Flink 内置支持下面的文件系统：

| Filesystem                            | Scheme       | Notes  |
| ------------------------------------- |--------------| ------ |
| Hadoop Distributed File System (HDFS) &nbsp; | `hdfs://`    | 只是所有HDFS版本 |
| Amazon S3                             | `s3://`      | Support 通过 Hadoop 文件系统实现 (见下文) |
| MapR file system                      | `maprfs://`  | 用户需要手动放置相关jar到`lib/` 目录 |
| Alluxio                               | `alluxio://` &nbsp; | Support 通过 Hadoop 文件系统实现 (见下文) | |



###  使用Hadoop 文件系统实现

Apache Flink允许用户使用任何实现了`org.apache.hadoop.fs.FileSystem`接口的文件系统。
实现了Hadoop `FileSystem`的有:

- [S3](https://aws.amazon.com/s3/) (已测试)
- [Google Cloud Storage Connector for Hadoop](https://cloud.google.com/hadoop/google-cloud-storage-connector) (已测试)
- [Alluxio](http://alluxio.org/) (已测试)
- [XtreemFS](http://www.xtreemfs.org/) (已测试)
- FTP via [Hftp](http://hadoop.apache.org/docs/r1.2.1/hftp.html) (未测试)
- 还有很多

为了在Flink中使用Hadoop文件系统，请确保

- `flink-conf.yaml`已经将`fs.hdfs.hadoopconf`属性设置为Hadoop配置目录。对于自动化测试或从IDE运行，可通过定义 `fs.hdfs.hadoopconf`环境变量来设置目录。
- Hadoop配置（在该目录中）在文件`core-site.xml`中具有所需文件系统的条目。S3和Alluxio的示例将在下面介绍。
- 使用文件系统所需的类可在Flink安装的`lib/`文件夹中找到（在运行Flink的所有机器上）。如果不能将文件放到目录中，Flink会使用`HADOOP_CLASSPATH`环境变量将Hadoop jar文件添加到classpath路径中。

#### Amazon S3

有关可用的S3文件系统实现，其配置和所需的库，请参阅[Deployment & Operations - Deployment - AWS - S3: Simple Storage Service]({{ site.baseurl }}/ops/deployment/aws.html) 。

#### Alluxio

对于Alluxio支持，将以下条目添加到`core-site.xml`文件中：

~~~xml
<property>
  <name>fs.alluxio.impl</name>
  <value>alluxio.hadoop.FileSystem</value>
</property>
~~~


## 使用Hadoop的Input / OutputFormat包装器连接到其他系统

Apache Flink允许用户访问许多不同的系统作为数据源或sink。
该系统的设计非常易于扩展。与Apache Hadoop类似，Flink具有所谓的InputFormats和OutputFormats的概念。

这些InputFormats的一个实现是HadoopInputFormat。这是一个允许用户在Flink中使用所有现有Hadoop输入格式的包装器。

本节介绍将Flink连接到其他系统的一些示例。
[了解更多关于Flink Hadoop兼容性的信息]({{ site.baseurl }}/dev/batch/hadoop_compatibility.html)。

## Flink对Avro的支持

Flink对ApacheAvro有广泛的内置支持。这允许使用Flink轻松读取Avro文件。
此外，Flink的序列化框架能够处理从Avro模式生成的类。确保将FlinkAvro依赖项包含到项目的pom.xml中。

~~~xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro</artifactId>
  <version>{{site.version }}</version>
</dependency>
~~~

为了从Avro文件读取数据，你必须指定一个`AvroInputFormat`。

**示例**:

~~~java
AvroInputFormat<User> users = new AvroInputFormat<User>(in, User.class);
DataSet<User> usersDS = env.createInput(users);
~~~

请注意`User`是Avro生成的POJO。Flink还允许执行这些POJO的基于字符串类型的key selection 。例如：

~~~java
usersDS.groupBy("name")
~~~


请注意，通过Flink可以使用`GenericData.Record`类型，但不推荐。由于该记录包含完整的模式，其数据密集型，因此使用起来可能很慢。

Flink的POJO字段selection 也适用于Avro生成的POJO。但是，仅当字段类型正确写入生成的类时才可能使用该用法。如果某个字段的类型为“Object”，则不能将该字段用作join或grouping 键。
在Avro中指定一个字段，就像这个`{"name": "type_double_test", "type": "double"},`是有效的，但是将它指定为只有一个字段的UNION-type（`{“name”：“ type_double_test“，”type“：[”double“]}，`）将生成一个类型为”Object“的字段。请注意，指定可以为空的类型 (`{"name": "type_double_test", "type": ["null", "double"]},`) 是可能的！



### 访问 Microsoft Azure Table Storage

_Note:这个例子从 Flink 0.6-incubating_ 开始

此示例使用HadoopInputFormat包装器来使用现有的Hadoop输入格式实现来访问[Azure's Table Storage](https://azure.microsoft.com/en-us/documentation/articles/storage-introduction/)。

1. 下载并编译`azure-tables-hadoop`项目。该项目开发的输入格式在Maven Central尚未提供，因此我们必须自行构建该项目。
执行以下命令:

   ~~~bash
   git clone https://github.com/mooso/azure-tables-hadoop.git
   cd azure-tables-hadoop
   mvn clean install
   ~~~

2. 使用quickstarts设置新的Flink项目：

   ~~~bash
   curl https://flink.apache.org/q/quickstart.sh | bash
   ~~~

3. 将以下依赖关系（在 `<dependencies>` 部分中）添加到`pom.xml`文件中：

   ~~~xml
   <dependency>
       <groupId>org.apache.flink</groupId>
       <artifactId>flink-hadoop-compatibility{{ site.scala_version_suffix }}</artifactId>
       <version>{{site.version}}</version>
   </dependency>
   <dependency>
     <groupId>com.microsoft.hadoop</groupId>
     <artifactId>microsoft-hadoop-azure</artifactId>
     <version>0.0.4</version>
   </dependency>
   ~~~

   `flink-hadoop-compatibility`是一个提供Hadoop输入格式包装器的Flink包。
   `microsoft-hadoop-azure`将我们之前构建的项目添加到我们的项目中。

该项目现在准备开始编码。我们建议将该项目导入IDE，例如Eclipse或IntelliJ。（作为Maven项目导入！）。
浏览到`Job.java`文件的代码。它是一个Flink作业的空框架。

将以下代码粘贴到它中：:

~~~java
import java.util.Map;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.hadoopcompatibility.mapreduce.HadoopInputFormat;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import com.microsoft.hadoop.azure.AzureTableConfiguration;
import com.microsoft.hadoop.azure.AzureTableInputFormat;
import com.microsoft.hadoop.azure.WritableEntity;
import com.microsoft.windowsazure.storage.table.EntityProperty;

public class AzureTableExample {

  public static void main(String[] args) throws Exception {
    // set up the execution environment
    final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

    // create a  AzureTableInputFormat, using a Hadoop input format wrapper
    HadoopInputFormat<Text, WritableEntity> hdIf = new HadoopInputFormat<Text, WritableEntity>(new AzureTableInputFormat(), Text.class, WritableEntity.class, new Job());

    // set the Account URI, something like: https://apacheflink.table.core.windows.net
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.ACCOUNT_URI.getKey(), "TODO");
    // set the secret storage key here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.STORAGE_KEY.getKey(), "TODO");
    // set the table name here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.TABLE_NAME.getKey(), "TODO");

    DataSet<Tuple2<Text, WritableEntity>> input = env.createInput(hdIf);
    // a little example how to use the data in a mapper.
    DataSet<String> fin = input.map(new MapFunction<Tuple2<Text,WritableEntity>, String>() {
      @Override
      public String map(Tuple2<Text, WritableEntity> arg0) throws Exception {
        System.err.println("--------------------------------\nKey = "+arg0.f0);
        WritableEntity we = arg0.f1;

        for(Map.Entry<String, EntityProperty> prop : we.getProperties().entrySet()) {
          System.err.println("key="+prop.getKey() + " ; value (asString)="+prop.getValue().getValueAsString());
        }

        return arg0.f0.toString();
      }
    });

    // emit result (this works only locally)
    fin.print();

    // execute program
    env.execute("Azure Example");
  }
}
~~~

该示例显示了如何访问Azure表并将数据转换为Flink的`DataSet`（更具体地说，该集的类型是`DataSet<Tuple2<Text, WritableEntity>>`）。使用`DataSet`，您可以将所有已知的转换应用于DataSet。

## 访问 MongoDB

This [GitHub repository documents how to use MongoDB with Apache Flink (starting from 0.7-incubating)](https://github.com/okkam-it/flink-mongodb-test).

{% top %}

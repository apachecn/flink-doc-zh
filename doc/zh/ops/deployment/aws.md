---
title: "Amazon Web Services (AWS)"
nav-title: AWS
nav-parent_id: deployment
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

Amazon Web Services提供可以运行Flink的云计算服务。

* ToC
{:toc}

## EMR：弹性MapReduce

[Amazon Elastic MapReduce](https://aws.amazon.com/elasticmapreduce/) (Amazon EMR)是一项Web服务，可以轻松快速设置Hadoop群集。这是在AWS上运行Flink 的 **推荐方式**，因为它负责设置所有内容。

### 标准EMR安装

link是Amazon EMR上受支持的应用程序。[Amazon's documentation](http://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-flink.html)介绍了配置Flink，创建和监控集群以及处理job。

### 自定义EMR安装

Amazon EMR服务定期更新为新版本，但可以手动将未提供的Flink版本安装在stock EMR群集中。

**创建EMR群集**

EMR文档包含[examples showing how to start an EMR cluster](http://docs.aws.amazon.com/ElasticMapReduce/latest/ManagementGuide/emr-gs-launch-sample-cluster.html)(显示如何启动EMR群集的示例)。您可以按照该指南安装任何EMR版本。您不需要安装EMR版本的*所有应用程序*部分，但必须安装*Core Hadoop*。

{% 注意 %}
在创建EMR集群时，访问S3存储桶需要
[配置IAM roles](http://docs.aws.amazon.com/ElasticMapReduce/latest/ManagementGuide/emr-iam-roles.html)。

**在EMR集群上安装Flink**

创建群集后，可以[connect to the master node](http://docs.aws.amazon.com/ElasticMapReduce/latest/ManagementGuide/emr-connect-master-node.html) 并安装Flink：

1. 转到[Downloads Page]({{ site.download_url }}) **并下载与您的EMR集群的Hadoop版本匹配的Flink版本**，例如Hadoop 2.7 for EMR版本4.3.0,4.0.0或4.5.0。
2. 解压Flink，在**设置了Hadoop config目录**后，[准备通过YARN部署Flink job](yarn_setup.html)：

```bash
HADOOP_CONF_DIR=/etc/hadoop/conf ./bin/flink run -m yarn-cluster -yn 1 examples/streaming/WordCount.jar
```

{% top %}

## S3: 简单存储服务

[Amazon Simple Storage Service](http://aws.amazon.com/s3/) （Amazon S3）为各种用例提供云对象存储。您可以使用带有Flink的S3来**读取**和**写入**数据，可以作为 [streaming **state backends**]({{ site.baseurl}}/ops/state/state_backends.html)（流式**状态后端**），甚至作为YARN对象存储。

通过使用以下格式指定路径，可以使用常规文件等S3对象：
您可以像使用常规文件一样使用S3对象，但需要以以如下格式指定路径:

```
s3://<your-bucket>/<endpoint>
```

端点可以是单个文件或目录，例如：

```java
// Read from S3 bucket
env.readTextFile("s3://<bucket>/<endpoint>");

// Write to S3 bucket
stream.writeAsText("s3://<bucket>/<endpoint>");

// Use S3 as FsStatebackend
env.setStateBackend(new FsStateBackend("s3://<your-bucket>/<endpoint>"));
```

注意，这些示例*不是*详尽的，您也可以在其他地方使用S3，包括[high availability setup](../jobmanager_high_availability.html) (高可用性设置)或 [RocksDBStateBackend]({{ site.baseurl }}/ops/state/state_backends.html#the-rocksdbstatebackend);（RocksDBStateBackend）； Flink期望每个地方都有一个文件系统URI。

对于大多数用例，您可能会使用我们的一种带有阴影的、由`flink-s3-fs-hadoop`和`flink-s3-fs-presto`文件系统包装，这是相当容易设置的。但是，对于某些情况，例如将S3用作YARN的资源存储目录，可能需要设置特定的Hadoop S3 文件系统实现。下面介绍两种方法。

### Shaded Hadoop / Presto S3文件系统（推荐）

{%  **注意:** 如果在 [Flink on EMR](#emr-elastic-mapreduce).（EMR上运行Flink），您不需要手动配置它。 %}

要使用`flink-s3-fs-hadoop`或者`flink-s3-fs-presto`，可以将各自的JAR文件从opt目录复制到Flink所在的lib目录，然后再开始Flink，例如：

```
cp ./opt/flink-s3-fs-presto-{{ site.version }}.jar ./lib/
```

#### 配置访问凭证

在设置S3 文件系统包装器之后，您需要确保允许Flink访问您的S3存储桶。

##### 身份和访问管理（IAM）（推荐）

在AWS上建立凭证的推荐方式是通过 [Identity and Access Management (IAM)](http://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)（身份和访问管理（IAM））。 您可以使用IAM功能安全地为Flink实例提供他们访问S3存储桶所需的凭据。有关如何执行此操作的详细信息超出了本文档的范围。请参阅AWS用户指南。 [IAM Roles](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html).

如果您正确设置了此项，则可以在AWS中管理对S3的访问权限，并且不需要将任何访问密钥分发给Flink。

##### 访问密钥（不推荐）

可以通过**访问和密钥对**获得对S3的访问。请注意，从 [introduction of IAM roles](https://blogs.aws.amazon.com/security/post/Tx1XG3FX6VMU6O5/A-safer-way-to-distribute-AWS-credentials-to-EC2)开始，这是不推荐使用的方式。

You need to configure both `s3.access-key` and `s3.secret-key`  in Flink's  `flink-conf.yaml`:
你需要 在Flink的配置文件`flink-conf.yaml`中配置 `s3.access-key`和`s3.secret-key`：

```
s3.access-key: your-access-key
s3.secret-key: your-secret-key
```

{% top %}

### Hadoop提供的S3文件系统 - 手动设置

{%  **注意:** 如果在 [Flink on EMR](#emr-elastic-mapreduce).（EMR上运行Flink），您不需要手动配置它。 %}

这个设置有点复杂，我们建议使用我们的配置的Hadoop/Presto文件系统(参见上文)，除非有其他要求，例如，通过配置Hadoop的`core-site.xml`文件中的`fs.defaultFS`属性将S3用作YARN的资源存储目录。

#### 设置S3文件系统

通过[Hadoop's S3 文件系统客户端](https://wiki.apache.org/hadoop/AmazonS3)之一与S3进行交互：

1. `S3AFileSystem` (**推荐** 用于Hadoop 2.7及更高版本）：用于在内部使用Amazon的SDK读取和写入常规文件的文件系统。没有最大文件大小并且与 IAM roles一起使用。
2. `NativeS3FileSystem`（用于Hadoop 2.6及更早版本）：用于读写常规文件的文件系统。最大对象大小为5GB，不适用于 IAM roles。

##### `S3AFileSystem` (推荐)

这是推荐的S3文件系统实现。它在内部使用Amazon的SDK并与IAM roles配合使用（请参阅 [Configure Access Credentials](#configure-access-credentials-1)).

您需要将Flink指向一个有效的Hadoop配置，该配置包含在`core-site.xml`配置文件中：

```xml
<configuration>

<property>
  <name>fs.s3.impl</name>
  <value>org.apache.hadoop.fs.s3a.S3AFileSystem</value>
</property>

<!-- Comma separated list of local directories used to buffer
     large results prior to transmitting them to S3. -->
<property>
  <name>fs.s3a.buffer.dir</name>
  <value>/tmp</value>
</property>

</configuration>
```

这将注册 `S3AFileSystem`为具有`s3a://`方案的URI的默认文件系统。

##### `NativeS3FileSystem`

该文件系统仅限于最大5GB的文件，并且不适用于 IAM roles（请参阅[Configure Access Credentials](#configure-access-credentials-1)），这意味着您必须在Hadoop配置文件中手动配置AWS凭证。

您需要将Flink指向一个有效的Hadoop配置，该配置包含在`core-site.xml`配置文件中：

```xml
<property>
  <name>fs.s3.impl</name>
  <value>org.apache.hadoop.fs.s3native.NativeS3FileSystem</value>
</property>
```

这将注册`NativeS3FileSystem`为具有`s3://`方案的URI的默认文件系统。

{% top %}

#### Hadoop配置

例如，您可以通过各种方式指定[Hadoop configuration](../config.html#hdfs) 将Flink指向Hadoop配置目录，例如：
- 通过设置环境变量`HADOOP_CONF_DIR`, 或者
- 通过在`flink-conf.yaml`中配置`fs.hdfs.hadoopconf`:
```
fs.hdfs.hadoopconf: /path/to/etc/hadoop
```

`/path/to/etc/hadoop` 用Flink 注册为Hadoop的配置目录。Flink会查找指定目录中的文件`core-site.xml`和`hdfs-site.xml`文件。

{% top %}

#### 配置访问凭证

{%  **注意:** 如果在 [Flink on EMR](#emr-elastic-mapreduce).（EMR上运行Flink），您不需要手动配置它。 %}

设置S3 FileSystem之后，您需要确保Flink可以访问您的S3存储桶。

##### Identity and Access Management (IAM) (推荐)

在使用时`S3AFileSystem`，建议在AWS上设置凭据的方式参考： [Identity and Access Management (IAM)](http://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)。您可以使用IAM功能安全地为Flink实例提供他们访问S3存储桶所需的凭据。有关如何执行此操作的详细信息超出了本文档的范围。请参阅AWS用户指南。 [IAM Roles](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html).

如果您正确设置了此项，则可以在AWS中管理对S3的访问权限，并且不需要将任何访问密钥分发给Flink。

请注意，这只适用于`S3AFileSystem` 而不适用于 `NativeS3FileSystem`。

{% top %}

##### Access Keys with `S3AFileSystem` (不推荐)

可以通过**访问和密钥对**获得对S3的访问。请注意，从[introduction of IAM roles](https://blogs.aws.amazon.com/security/post/Tx1XG3FX6VMU6O5/A-safer-way-to-distribute-AWS-credentials-to-EC2)开始，这是不建议是使用的方式。

对于`S3AFileSystem`，您需要在Hadoop的`core-site.xml`中配置 `fs.s3a.access.key`和 `fs.s3a.secret.key`：

```xml
<property>
  <name>fs.s3a.access.key</name>
  <value></value>
</property>

<property>
  <name>fs.s3a.secret.key</name>
  <value></value>
</property>
```

{% top %}

##### Access Keys with `NativeS3FileSystem` (不推荐)

可以通过**访问和密钥对**获得对S3的访问。请注意，从[introduction of IAM roles](https://blogs.aws.amazon.com/security/post/Tx1XG3FX6VMU6O5/A-safer-way-to-distribute-AWS-credentials-to-EC2)开始，这是不建议是使用的方式。

对于`NativeS3FileSystem`你需要 在Hadoop的`core-site.xml`中配置`fs.s3.awsAccessKeyId`和`fs.s3.awsSecretAccessKey`：

```xml
<property>
  <name>fs.s3.awsAccessKeyId</name>
  <value></value>
</property>

<property>
  <name>fs.s3.awsSecretAccessKey</name>
  <value></value>
</property>
```

{% top %}

#### 提供S3 文件系统依赖关系

{%  **注意:** 如果在[Flink on EMR](#emr-elastic-mapreduce)（EMR上运行Flink），您不需要手动配置它[Flink on EMR](#emr-elastic-mapreduce)。 %}

Hadoop的S3 文件系统客户端打包在`hadoop-aws`工件中（Hadoop版本2.6和更高版本）。此JAR及其所有依赖项需要添加到Flink的类路径中，例如：Job和TaskManagers的类路径。根据您所使用的文件系统实现和Flink和Hadoop版本的不同，您需要提供不同的依赖关系(见下文)。

将JAR添加到Flink的类路径有多种方式，最简单的方法就是将JAR放入Flink的`lib`文件夹中。您需要复制`hadoop-aws`JAR及其所有依赖项。您还可以将包含这些JAR的目录作为`HADOOP_CLASSPATH`环境变量的一部分输出到所有机器上。

##### Flink for Hadoop 2.7

根据您使用的文件系统，请添加以下依赖项。您可以在以下位置找到这些作为二进制文件的一部分：`hadoop-2.7/share/hadoop/tools/lib`:

- `S3AFileSystem`:
  - `hadoop-aws-2.7.3.jar`
  - `aws-java-sdk-s3-1.11.183.jar` and its dependencies:
    - `aws-java-sdk-core-1.11.183.jar`
    - `aws-java-sdk-kms-1.11.183.jar`
    - `jackson-annotations-2.6.7.jar`
    - `jackson-core-2.6.7.jar`
    - `jackson-databind-2.6.7.jar`
    - `joda-time-2.8.1.jar`
    - `httpcore-4.4.4.jar`
    - `httpclient-4.5.3.jar`

- `NativeS3FileSystem`:
  - `hadoop-aws-2.7.3.jar`
  - `guava-11.0.2.jar`

注意，`hadoop-common` 可以作为Flink的一部分，但是Guava被Flink遮蔽。

##### Flink for Hadoop 2.6

根据您使用的文件系统，请添加以下依赖项。您可以在以下位置找到这些作为二进制文件的一部分：`hadoop-2.6/share/hadoop/tools/lib`:

- `S3AFileSystem`:
  - `hadoop-aws-2.6.4.jar`
  - `aws-java-sdk-1.7.4.jar` and its dependencies:
    - `jackson-annotations-2.1.1.jar`
    - `jackson-core-2.1.1.jar`
    - `jackson-databind-2.1.1.jar`
    - `joda-time-2.2.jar`
    - `httpcore-4.2.5.jar`
    - `httpclient-4.2.5.jar`

- `NativeS3FileSystem`:
  - `hadoop-aws-2.6.4.jar`
  - `guava-11.0.2.jar`

注意，`hadoop-common` 可以作为Flink的一部分，但是Guava被Flink遮蔽。

##### Flink for Hadoop 2.4 及更早版本

这些Hadoop版本只支持`NativeS3FileSystem`。这是作为Hadoop 2的一部分与Flink一起预先打包到`hadoop-common`中。您不需要向类路径添加任何内容。

{% top %}

## 常见问题

以下部分列出了在AWS上使用Flink时的常见问题。

### Missing S3 FileSystem Configuration

如果您的job提交失败并显示异常消息，注意`No file system found with scheme s3`这意味着没有为S3配置文件系统。请查看我们 [shaded Hadoop/Presto](#shaded-hadooppresto-s3-file-systems-recommended)或[generic Hadoop](#set-s3-filesystem) 文件系统的配置部分，以了解如何正确配置它的详细信息。

```
org.apache.flink.client.program.ProgramInvocationException: The program execution failed:
  Failed to submit job cd927567a81b62d7da4c18eaa91c3c39 (WordCount Example) [...]
Caused by: org.apache.flink.runtime.JobException: Creating the input splits caused an error:
  No file system found with scheme s3, referenced in file URI 's3://<bucket>/<endpoint>'. [...]
Caused by: java.io.IOException: No file system found with scheme s3,
  referenced in file URI 's3://<bucket>/<endpoint>'.
    at o.a.f.core.fs.FileSystem.get(FileSystem.java:296)
    at o.a.f.core.fs.Path.getFileSystem(Path.java:311)
    at o.a.f.api.common.io.FileInputFormat.createInputSplits(FileInputFormat.java:450)
    at o.a.f.api.common.io.FileInputFormat.createInputSplits(FileInputFormat.java:57)
    at o.a.f.runtime.executiongraph.ExecutionJobVertex.<init>(ExecutionJobVertex.java:156)
```

{% top %}

### AWS Access Key ID and Secret Access Key Not Specified

如果您发现自己的job失败并注意到`AWS Access Key ID and Secret Access Key must be specified as the username or password`，您的访问凭证没有正确设置，请注意。有关如何配置此功能的详细信息，请参阅我们[shaded Hadoop/Presto](#configure-access-credentials)或[generic Hadoop](#configure-access-credentials-1) 文件系统的访问凭证部分。

```
org.apache.flink.client.program.ProgramInvocationException: The program execution failed:
  Failed to submit job cd927567a81b62d7da4c18eaa91c3c39 (WordCount Example) [...]
Caused by: java.io.IOException: The given file URI (s3://<bucket>/<endpoint>) points to the
  HDFS NameNode at <bucket>, but the File System could not be initialized with that address:
  AWS Access Key ID and Secret Access Key must be specified as the username or password
  (respectively) of a s3n URL, or by setting the fs.s3n.awsAccessKeyId
  or fs.s3n.awsSecretAccessKey properties (respectively) [...]
Caused by: java.lang.IllegalArgumentException: AWS Access Key ID and Secret Access Key must
  be specified as the username or password (respectively) of a s3 URL, or by setting
  the fs.s3n.awsAccessKeyId or fs.s3n.awsSecretAccessKey properties (respectively) [...]
    at o.a.h.fs.s3.S3Credentials.initialize(S3Credentials.java:70)
    at o.a.h.fs.s3native.Jets3tNativeFileSystemStore.initialize(Jets3tNativeFileSystemStore.java:80)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:606)
    at o.a.h.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:187)
    at o.a.h.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:102)
    at o.a.h.fs.s3native.$Proxy6.initialize(Unknown Source)
    at o.a.h.fs.s3native.NativeS3FileSystem.initialize(NativeS3FileSystem.java:330)
    at o.a.f.runtime.fs.hdfs.HadoopFileSystem.initialize(HadoopFileSystem.java:321)
```

{% top %}

### ClassNotFoundException: NativeS3FileSystem/S3AFileSystem Not Found

如果看到此异常，则S3 文件系统不是Flink类路径的一部分。有关如何正确配置的详细信息，请参阅[S3 FileSystem dependency section](#provide-s3-filesystem-dependency) 。

```
Caused by: java.lang.RuntimeException: java.lang.RuntimeException: java.lang.ClassNotFoundException: Class org.apache.hadoop.fs.s3native.NativeS3FileSystem not found
  at org.apache.hadoop.conf.Configuration.getClass(Configuration.java:2186)
  at org.apache.flink.runtime.fs.hdfs.HadoopFileSystem.getHadoopWrapperClassNameForFileSystem(HadoopFileSystem.java:460)
  at org.apache.flink.core.fs.FileSystem.getHadoopWrapperClassNameForFileSystem(FileSystem.java:352)
  at org.apache.flink.core.fs.FileSystem.get(FileSystem.java:280)
  at org.apache.flink.core.fs.Path.getFileSystem(Path.java:311)
  at org.apache.flink.api.common.io.FileInputFormat.createInputSplits(FileInputFormat.java:450)
  at org.apache.flink.api.common.io.FileInputFormat.createInputSplits(FileInputFormat.java:57)
  at org.apache.flink.runtime.executiongraph.ExecutionJobVertex.<init>(ExecutionJobVertex.java:156)
  ... 25 more
Caused by: java.lang.RuntimeException: java.lang.ClassNotFoundException: Class org.apache.hadoop.fs.s3native.NativeS3FileSystem not found
  at org.apache.hadoop.conf.Configuration.getClass(Configuration.java:2154)
  at org.apache.hadoop.conf.Configuration.getClass(Configuration.java:2178)
  ... 32 more
Caused by: java.lang.ClassNotFoundException: Class org.apache.hadoop.fs.s3native.NativeS3FileSystem not found
  at org.apache.hadoop.conf.Configuration.getClassByName(Configuration.java:2060)
  at org.apache.hadoop.conf.Configuration.getClass(Configuration.java:2152)
  ... 33 more
```

{% top %}

### IOException: `400: Bad Request`

如果您已正确配置所有内容，但得到`Bad Request`异常**并且**您的S3存储桶位于区域`eu-central-1`，则您可能正在运行S3客户端，该客户端不支持 [Amazon's signature version 4](http://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-authenticating-requests.html)。

```
[...]
Caused by: java.io.IOException: s3://<bucket-in-eu-central-1>/<endpoint> : 400 : Bad Request [...]
Caused by: org.jets3t.service.impl.rest.HttpException [...]
```
或
```
com.amazonaws.services.s3.model.AmazonS3Exception: Status Code: 400, AWS Service: Amazon S3, AWS Request ID: [...], AWS Error Code: null, AWS Error Message: Bad Request, S3 Extended Request ID: [...]

```

This should not apply to our shaded Hadoop/Presto S3 file systems but can occur for Hadoop-provided
S3 file systems. In particular, all Hadoop versions up to 2.7.2 running `NativeS3FileSystem` (which
depend on `JetS3t 0.9.0` instead of a version [>= 0.9.4](http://www.jets3t.org/RELEASE_NOTES.html))
are affected but users also reported this happening with the `S3AFileSystem`.
这不适用于我们定制的Hadoop/Presto S3文件系统，但可以在Hadoop提供的S3文件系统中使用。特别是，所有Hadoop版本更新到2.7.2以上，运行 `NativeS3FileSystem` （取决于`JetS3t 0.9.0` 版本[>= 0.9.4](http://www.jets3t.org/RELEASE_NOTES.html)）受到影响，但用户使用 `S3AFileSystem`也报告了发生这种情况。

Except for changing the bucket region, you may also be able to solve this by
[requesting signature version 4 for request authentication](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingAWSSDK.html#specify-signature-version),
e.g. by adding this to Flink's JVM options in `flink-conf.yaml` (see
[configuration](../config.html#common-options)):
除了更改存储区域之外，您还可以通过[requesting signature version 4 for request authentication](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingAWSSDK.html#specify-signature-version)（请求签名版本4来请求身份验证）来解决此问题 ，例如通过`flink-conf.yaml`文件，将其添加到Flink的JVM配置（请参阅
[configuration](../config.html#common-options)）：
```
env.java.opts: -Dcom.amazonaws.services.s3.enableV4
```

{% top %}

### NullPointerException at org.apache.hadoop.fs.LocalDirAllocator

This Exception is usually caused by skipping the local buffer directory configuration `fs.s3a.buffer.dir` for the `S3AFileSystem`. Please refer to the [S3AFileSystem configuration](#s3afilesystem-recommended) section to see how to configure the `S3AFileSystem` properly.
此异常通常由跳过本地缓冲目录配置`fs.s3a.buffer.dir`引起。请参阅[S3AFileSystem configuration](#s3afilesystem-recommended) 配置部分，了解如何正确配置`S3AFileSystem`。

```
[...]
Caused by: java.lang.NullPointerException at
o.a.h.fs.LocalDirAllocator$AllocatorPerContext.confChanged(LocalDirAllocator.java:268) at
o.a.h.fs.LocalDirAllocator$AllocatorPerContext.getLocalPathForWrite(LocalDirAllocator.java:344) at
o.a.h.fs.LocalDirAllocator$AllocatorPerContext.createTmpFileForWrite(LocalDirAllocator.java:416) at
o.a.h.fs.LocalDirAllocator.createTmpFileForWrite(LocalDirAllocator.java:198) at
o.a.h.fs.s3a.S3AOutputStream.<init>(S3AOutputStream.java:87) at
o.a.h.fs.s3a.S3AFileSystem.create(S3AFileSystem.java:410) at
o.a.h.fs.FileSystem.create(FileSystem.java:907) at
o.a.h.fs.FileSystem.create(FileSystem.java:888) at
o.a.h.fs.FileSystem.create(FileSystem.java:785) at
o.a.f.runtime.fs.hdfs.HadoopFileSystem.create(HadoopFileSystem.java:404) at
o.a.f.runtime.fs.hdfs.HadoopFileSystem.create(HadoopFileSystem.java:48) at
... 25 more
```

{% top %}

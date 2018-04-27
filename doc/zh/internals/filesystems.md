---
title: "File Systems"
nav-parent_id: internals
nav-pos: 10
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

* Replaced by the TOC
{:toc}

flink通过类`org.apache.flink.core.fs.FileSystem`来抽象自己的文件系统。这个抽象提供了各种文件系统的通用操作和实现的最低保证。

为了支持尽量多的文件系统，这个`FileSystem`的可用操作非常有限。比如，追加或者变更已有文件是不支持的。

文件系统由*文件系统格式*来区分，比如 `file://`,`hdfs://`等。

# 实现

Flink直接实现了的文件系统,有下面这些文件系统格式:
  - `file`,表示机器本地文件系统.
 
其他文件系统类型通过桥接到[Apache Hadoop](https://hadoop.apache.org/)来支持文件系统的适配。以下是个不完整的例子列表:
  - `hdfs`: hadoop 分布式文件系统
  - `s3`,`s3n`,`s3a`: 亚马逊的s3文件系统
  - `gcs`: 谷歌云存储服务
  - `maprfs`: MapR分布式文件系统
  - ...

如果Flink从class path中找到有效的hadoop文件系统类和有效的hadoop配置文件，flink会装载hadoop文件系统。默认情况下，flink会在classpath下寻找hadoop配置。或者也可以通过配置`fs.hdfs.hadoopconf`来自定义hadoop的配置文件路径.

# 持久化保证

这些`FileSystem`和它的`FsDataOutputStream`实例用来持久化存储数据，计算结果和程序都是高可用可恢复的。因此这些流的持久化语义被良好定义是至关重要的。

## 持久化保证的定义

如果数据的输出流满足这两个要求，则可以认为是持久化的：

  1. **可见性要求：** 当给出一个绝对路径时，必需保证所有的其他进程、机器、虚拟机、容器等都可以访问文件并一致的查看数据。这个要求类似于 POSIX(可移植操作系统接口标准)定义的*close-to-open*语义，但受限于文件本身（由于它自己的绝对路径）。
  2. **持久性要求：** 文件系统必需满足特定的持久性需求。这里指一些特定的文件系统。比如{@link LocalFileSystem} 当出现硬件或操作系统故障的时候不提供持久性保证，当具有备份的分布式文件系统(比如HDFS)保证典型的持久化，当副本因子设置为*n*的时候，持久化保证可以容忍*n*个节点同时出现故障。

对文件父目录的更新(比如列出目录内容显示的文件)不需要等待文件流的完成，这里没有强一致性要求。文件系统的目录内容是最终一致性的，这个一致性要求的放宽对文件系统来说很重要。

 `FSDataOutputStream` 的`FSDataOutputStream.close()`方法一旦调用并返回，就必需保证写入的数据被持久化存储。

## 示例
  - 对于**高可靠分布式文件系统**，数据是强一致性的，一旦数据被文件系统接受和确认后，通常会被复制到合法的机器上(*持久性要求*），另外其他所有可能访问该文件的机器都可以通过绝对路径来访问文件(*可见性要求*)
  
  数据在存储节点上是否命中非易失（高可用）依赖于文件系统的特性。
  
  文件的父目录元数据修改不需要状态一致性。在查看父目录文件列表时候，允许有些机器查看到而有些机器看不到，只要所有的机器都能通过绝对路径访问到指定文件。
    
  - **本地文件系统 ** 必需支持 POSIX(可移植操作系统) 的*close-to-open* 语义。因为本地文件系统没有高可用保证，也就不存在进一步的要求。
  
    上面说的情况意味着当数据被本地文件系统存储时，数据可能依然在操作系统缓存中。如果本地机器出现故障，操作系统可能丢失缓存中的数据，因此本地文件系统会不能满足Flink的文件系统保证.
    
    这意味着计算结果、checkpoints和savepoints如果只是写在本地文件系统，不能保证本地机器故障出现后总是可恢复的，因此本地文件系统不适合用于生产环境。
     

# 更新文件内容

许多文件系统根本不支持覆盖重写已存在的文件,或者这种情况下不支持文件内容更新的可见性一致。出于这样的原因，Flink的文件系统不支持追加已存在的文件，也不支持从输出流中搜索内容并进行修改，以便先前写入的数据可以在文件中修改。

# 覆盖文件

覆盖文件是可行的。一个文件的覆盖通过删除并新建文件达到。然而，有的文件系统不能保证这种更改对所有访问这个文件的访问方的可见性是一致的。比如[Amazon S3](https://aws.amazon.com/documentation/s3/) 在出现文件替换的时候的可见性只保证 *最终一致性* :有的机器访问了老的文件，而有的机器访问新的文件。

为了避免这种一致性问题，flink在故障/恢复机制的实现上，严格避免同一个文件被多次写入。

# 线程安全
`FileSystem`的实现必需是线程安全的：Flink的`FileSystem`实例经常需要在多个线程中共享，并且需要能并发创建输入/输出流和列出文件元数据。

`FSDataOutputStream`和`FSDataOutputStream`的实现是严格**非线程安全**的。流的实例不应该出现跨线程操作，因为不能保证跨线程操作的可见性(许多操作不会创建memory fences(操作系统的内存栅栏))

{% top %}

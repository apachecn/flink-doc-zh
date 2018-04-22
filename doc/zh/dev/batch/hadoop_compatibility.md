---
title: "Hadoop 兼容性"
is_beta: true
nav-parent_id: batch
nav-pos: 7
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

Flink与Apache Hadoop MapReduce接口兼容，因此允许重复使用为Hadoop MapReduce实现的代码。

您可以:

- 在Flink程序中使用 Hadoop 的`Writable` [数据类型](index.html#data-types)。
- 使用任何 Hadoop `InputFormat` 作为[数据源](index.html#data-sources).
- 使用任何 Hadoop `OutputFormat` 作为[DataSink](index.html#data-sinks).
- 使用一个 Hadoop `Mapper` 作为 [FlatMapFunction](dataset_transformations.html#flatmap).
- 使用一个 Hadoop `Reducer` 作为 [GroupReduceFunction](dataset_transformations.html#groupreduce-on-grouped-dataset).

本文档展示了如何在Flink中使用现有的Hadoop MapReduce代码。 请参阅[Connecting to other systems]({{ site.baseurl }}/dev/batch/connectors.html) 指南以阅读Hadoop支持的文件系统。

* This will be replaced by the TOC
{:toc}

### Project 配置

支持Hadoop输入/输出格式是编写Flink作业时始终需要的`flink-java`和`flink-scala` Maven模块的一部分。
代码位于`org.apache.flink.api.java.hadoop` 和`org.apache.flink.api.scala.hadoop`中，位于`mapred`和 `mapreduce` API的附加子包中。

Hadoop Mappers和Reducers的支持包含在`flink-hadoop-compatibility` Maven模块中。
这段代码在`org.apache.flink.hadoopcompatibility`包中。

如果要重用Mappers和Reducers，请将以下依赖项添加到“pom.xml”中。

~~~xml
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-hadoop-compatibility{{ site.scala_version_suffix }}</artifactId>
	<version>{{site.version}}</version>
</dependency>
~~~

###  使用 Hadoop 数据类型

Flink支持所有可立即使用的Hadoop `Writable` 和 `WritableComparable`数据类型。
如果您只想使用Hadoop数据类型，则不需要包含Hadoop兼容性依赖项。
请参阅[编程指南](index.html#data-types)了解更多详情。

### 使用 Hadoop InputFormats

通过使用`ExecutionEnvironment`的`readHadoopFile` 或 `createHadoopInput `方法之一，Hadoop输入格式可用于创建数据源。
前者用于从 `FileInputFormat` 派生的输入格式，而后者用于通用输入格式。

The resulting `DataSet` contains 2-tuples where the first field
is the key and the second field is the value retrieved from the Hadoop
InputFormat.

The following example shows how to use Hadoop's `TextInputFormat`.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Tuple2<LongWritable, Text>> input =
    env.readHadoopFile(new TextInputFormat(), LongWritable.class, Text.class, textPath);

// Do something with the data.
[...]
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val env = ExecutionEnvironment.getExecutionEnvironment

val input: DataSet[(LongWritable, Text)] =
  env.readHadoopFile(new TextInputFormat, classOf[LongWritable], classOf[Text], textPath)

// Do something with the data.
[...]
~~~

</div>

</div>

### 使用 Hadoop OutputFormats

Flink为Hadoop `OutputFormats`提供了一个兼容包装器。
支持任何实现了`org.apache.hadoop.mapred.OutputFormat`或扩展`org.apache.hadoop.mapreduce.OutputFormat`的类。
`OutputFormat`包装器期望其输入数据是包含键和值的2元组的DataSet。这些将由Hadoop OutputFormat进行处理。

下列演示了如何使用Hadoop 的`TextOutputFormat`

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// Obtain the result we want to emit
DataSet<Tuple2<Text, IntWritable>> hadoopResult = [...]

// Set up the Hadoop TextOutputFormat.
HadoopOutputFormat<Text, IntWritable> hadoopOF =
  // create the Flink wrapper.
  new HadoopOutputFormat<Text, IntWritable>(
    // set the Hadoop OutputFormat and specify the job.
    new TextOutputFormat<Text, IntWritable>(), job
  );
hadoopOF.getConfiguration().set("mapreduce.output.textoutputformat.separator", " ");
TextOutputFormat.setOutputPath(job, new Path(outputPath));

// Emit data using the Hadoop TextOutputFormat.
hadoopResult.output(hadoopOF);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// Obtain your result to emit.
val hadoopResult: DataSet[(Text, IntWritable)] = [...]

val hadoopOF = new HadoopOutputFormat[Text,IntWritable](
  new TextOutputFormat[Text, IntWritable],
  new JobConf)

hadoopOF.getJobConf.set("mapred.textoutputformat.separator", " ")
FileOutputFormat.setOutputPath(hadoopOF.getJobConf, new Path(resultPath))

hadoopResult.output(hadoopOF)


~~~

</div>

</div>

### 使用 Hadoop Mappers 和 Reducers

Hadoop Mappers 在语义上等价于Flink的[FlatMapFunctions](dataset_transformations.html#flatmap) ，而Hadoop Reducers与Flink的[GroupReduceFunctions](dataset_transformations.html#groupreduce-on-grouped-dataset)等效。Flink为Hadoop MapReduce的`Mapper`和`Reducer`接口的实现提供了包装器，也就是说，您可以在常规Flink程序中重用您的Hadoop Mappers and Reducers。目前，只支持Hadoop 的mapred  API（`org.apache.hadoop.mapred`）的Mapper和Reduce接口。

这些包装器将一个 `DataSet<Tuple2<KEYIN,VALUEIN>>` 作为输入，并产生一个`DataSet<Tuple2<KEYOUT,VALUEOUT>>`作为输出，其中`KEYIN`和`KEYOUT`是键和`VALUEIN`， VALUEOUT`是由Hadoop函数处理的Hadoop键值对的值。对于Reducers，Flink提供了带有（`HadoopReduceCombineFunction`）并且没有组合器（`HadoopReduceFunction`）的GroupReduceFunction封装。这些包装接受一个可选的`JobConf`对象来配置Hadoop Mapper或Reducer。

Flink的函数包装器是

- `org.apache.flink.hadoopcompatibility.mapred.HadoopMapFunction`,
- `org.apache.flink.hadoopcompatibility.mapred.HadoopReduceFunction`, and
- `org.apache.flink.hadoopcompatibility.mapred.HadoopReduceCombineFunction`.

并可以用作常规的 Flink [FlatMapFunctions](dataset_transformations.html#flatmap) 或 [GroupReduceFunctions](dataset_transformations.html#groupreduce-on-grouped-dataset).

下列演示了如何使用  Hadoop `Mapper` h和 `Reducer` functions.

~~~java
// Obtain data to process somehow.
DataSet<Tuple2<Text, LongWritable>> text = [...]

DataSet<Tuple2<Text, LongWritable>> result = text
  // use Hadoop Mapper (Tokenizer) as MapFunction
  .flatMap(new HadoopMapFunction<LongWritable, Text, Text, LongWritable>(
    new Tokenizer()
  ))
  .groupBy(0)
  // use Hadoop Reducer (Counter) as Reduce- and CombineFunction
  .reduceGroup(new HadoopReduceCombineFunction<Text, LongWritable, Text, LongWritable>(
    new Counter(), new Counter()
  ));
~~~

**Please note:** Reducer包装器按照Flink的[[groupBy()](dataset_transformations.html#transformations-on-grouped-dataset) 算子定义的组工作。它不考虑可能在“JobConf”中设置的自定义分区，排序或者分组。

### 完整的Hadoop WordCount示例

以下示例显示了使用Hadoop数据类型，Input-和OutputFormats以及Mapper和Reducer的完整WordCount实现。

~~~java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Set up the Hadoop TextInputFormat.
Job job = Job.getInstance();
HadoopInputFormat<LongWritable, Text> hadoopIF =
  new HadoopInputFormat<LongWritable, Text>(
    new TextInputFormat(), LongWritable.class, Text.class, job
  );
TextInputFormat.addInputPath(job, new Path(inputPath));

// Read data using the Hadoop TextInputFormat.
DataSet<Tuple2<LongWritable, Text>> text = env.createInput(hadoopIF);

DataSet<Tuple2<Text, LongWritable>> result = text
  // use Hadoop Mapper (Tokenizer) as MapFunction
  .flatMap(new HadoopMapFunction<LongWritable, Text, Text, LongWritable>(
    new Tokenizer()
  ))
  .groupBy(0)
  // use Hadoop Reducer (Counter) as Reduce- and CombineFunction
  .reduceGroup(new HadoopReduceCombineFunction<Text, LongWritable, Text, LongWritable>(
    new Counter(), new Counter()
  ));

// Set up the Hadoop TextOutputFormat.
HadoopOutputFormat<Text, IntWritable> hadoopOF =
  new HadoopOutputFormat<Text, IntWritable>(
    new TextOutputFormat<Text, IntWritable>(), job
  );
hadoopOF.getConfiguration().set("mapreduce.output.textoutputformat.separator", " ");
TextOutputFormat.setOutputPath(job, new Path(outputPath));

// Emit data using the Hadoop TextOutputFormat.
result.output(hadoopOF);

// Execute Program
env.execute("Hadoop WordCount");
~~~

{% top %}

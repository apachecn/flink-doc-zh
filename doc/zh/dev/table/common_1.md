---
title: "Concepts & Common API"
nav-parent_id: tableapi
nav-pos: 0
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

Table API and SQL 通过joint API集成在一起。这个API的核心概念是`Table`，Table可以作为查询的输入和输出。该文档将展示Table API and SQL 查询程序的共性结构，例如，如何注册表，如何查询表，及如何输出表。

* This will be replaced by the TOC
{:toc}

Table API和SQL的程序结构
---------------------------------------

无论针对是批处理还是流处理，Table API和SQL 的所有程序都遵从相同的模式。下面的程序实例会展示Table API和SQL程序的共性结构。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// for batch programs use ExecutionEnvironment instead of StreamExecutionEnvironment
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// create a TableEnvironment
// for batch programs use BatchTableEnvironment instead of StreamTableEnvironment
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register a Table
tableEnv.registerTable("table1", ...)            // or
tableEnv.registerTableSource("table2", ...);     // or
tableEnv.registerExternalCatalog("extCat", ...);

// create a Table from a Table API query
Table tapiResult = tableEnv.scan("table1").select(...);
// create a Table from a SQL query
Table sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ... ");

// emit a Table API result Table to a TableSink, same for SQL result
tapiResult.writeToSink(...);

// execute
env.execute();

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// for batch programs use ExecutionEnvironment instead of StreamExecutionEnvironment
val env = StreamExecutionEnvironment.getExecutionEnvironment

// create a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// register a Table
tableEnv.registerTable("table1", ...)           // or
tableEnv.registerTableSource("table2", ...)     // or
tableEnv.registerExternalCatalog("extCat", ...) 

// create a Table from a Table API query
val tapiResult = tableEnv.scan("table1").select(...)
// Create a Table from a SQL query
val sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ...")

// emit a Table API result Table to a TableSink, same for SQL result
tapiResult.writeToSink(...)

// execute
env.execute()

{% endhighlight %}
</div>
</div>

**注意：**Table API和SQL查询可以很容易地被集成并嵌入到DataStream或者SataSet程序中。可以参考[Integration with DataStream and DataSet API](#integration-with-datastream-and-dataset-api)章节，学习如何将DataStream和DataSet与表进行相互转换。

{% top %}

创建一个TableEnvironment
-------------------------

TableEnvironment是Table API和SQL集成的核心概念，负责以下内容：

* 在内部目录下注册 `Table` 
* 注册一个外部目录 
* 执行SQL查询
* 注册一个用户自定义函数（scala，table 或者 aggregation）
* 将一个`DataStream`或者`DataSet`转为一个`Table`
* 提供 `ExecutionEnvironment` 或 `StreamExecutionEnvironment`两种选择

每个 `Table` 都会关联一个具体的`TableEnvironment`。在同一个查询中，不可能结合不同TableEnvironment下的表，例如，join或者union表。

一个`TableEnvironment`可以通过调用带有一个`StreamExecutionEnvironment`或者`ExecutionEnvironment`的`TableEnvironment.getTableEnvironment()`的静态方法和一个可选的`TableConfig`创建。其中，`TableConfig`可用于配置`TableEnvironment`，或者定制查询优化及翻译过程（参考[Query Optimization](#query-optimization)）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// ***************
// STREAMING QUERY
// ***************
StreamExecutionEnvironment sEnv = StreamExecutionEnvironment.getExecutionEnvironment();
// create a TableEnvironment for streaming queries
StreamTableEnvironment sTableEnv = TableEnvironment.getTableEnvironment(sEnv);

// ***********
// BATCH QUERY
// ***********
ExecutionEnvironment bEnv = ExecutionEnvironment.getExecutionEnvironment();
// create a TableEnvironment for batch queries
BatchTableEnvironment bTableEnv = TableEnvironment.getTableEnvironment(bEnv);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// ***************
// STREAMING QUERY
// ***************
val sEnv = StreamExecutionEnvironment.getExecutionEnvironment
// create a TableEnvironment for streaming queries
val sTableEnv = TableEnvironment.getTableEnvironment(sEnv)

// ***********
// BATCH QUERY
// ***********
val bEnv = ExecutionEnvironment.getExecutionEnvironment
// create a TableEnvironment for batch queries
val bTableEnv = TableEnvironment.getTableEnvironment(bEnv)
{% endhighlight %}
</div>
</div>

{% top %}

在目录中注册Tables
-------------------------------

一个`TableEnvironment`维护着一个以名字注册的表的目录。有input tables和output tables两种类型的表。input tables可以在Table API和SQL查询中被引用并提供输入数据。output tables可以将Table API和SQL查询的结果输出到一个外部系统。

一个input table可以从各种source进行注册：

* 一个现存的`Table`对象，通常是Table API和SQL查询的结果
* 一个`TableSource`，访问外部数据，例如file，database，或者messaging system
* 来自DataStream或者DataSet程序的`DataStream`或者`DataSet`，关于注册一个`DataStream`或者`DataSet`可以参考[Integration with DataStream and DataSet API](#integration-with-datastream-and-dataset-api)

output table可以通过一个`TableSink`注册。

### 注册一个Table

一个 `Table` 在一个`TableEnvironment`下注册程序，如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table is the result of a simple projection query 
Table projTable = tableEnv.scan("X").project(...);

// register the Table projTable as table "projectedX"
tableEnv.registerTable("projectedTable", projTable);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// Table is the result of a simple projection query 
val projTable: Table = tableEnv.scan("X").project(...)

// register the Table projTable as table "projectedX"
tableEnv.registerTable("projectedTable", projTable)
{% endhighlight %}
</div>
</div>

**注意：**注册表的处理方式与`VIEW`被引用到关系数据库系统中的处理方式类似，即定义table的未被优化的查询被内联，当有其他查询引用注册表时。如果有多个查询引用相同的注册表时，该定义表的查询会将针对每个引用查询进行内联并执行多次，即注册表的结果不会共享。

{% top %}

### 注册一个TableSource

一个`TableSource`提供对外部数据的访问，外部数据是指存储在例如数据库（MySQL,HBase,...），具有特定编码的文件（CSV,Apache [Parquet, Avro, ORC], …），或者信息传送系统（Apache Kafka, RabbitMQ, …）。

Flink旨在为常见的数据格式和存储系统提供`TableSources`，有关获取支持的`TableSource`列表及如何构造一个自定义`TableSource`的说明，可查阅[Table Source and Sinks]({{ site.baseurl }}/dev/table/sourceSinks.html)。

`TableSource`在`TableEnvironment`注册程序，如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSource
TableSource csvSource = new CsvTableSource("/path/to/file", ...);

// register the TableSource as table "CsvTable"
tableEnv.registerTableSource("CsvTable", csvSource);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// create a TableSource
val csvSource: TableSource = new CsvTableSource("/path/to/file", ...)

// register the TableSource as table "CsvTable"
tableEnv.registerTableSource("CsvTable", csvSource)
{% endhighlight %}
</div>
</div>

{% top %}

### 注册一个TableSink

已注册的`TableSink`可以将Table API或者SQL查询的结果输出到一个外部存储系统，例如一个数据库，键值存储，消息队列或者文件系统（不同编码方式，例如CSV,Apache[Parquet,Avro,ORC],...）参见[emit the result of a Table API or SQL query](common.html#emit-a-table)。

Flink旨在为常见的数据格式和存储系统提供`TableSinks`。关于可用的sinks和如何实现自定义说明的细节，参阅[Table Sources and Sinks]({{ site.baseurl }}/dev/table/sourceSinks.html)。

`TableEnvironment`下注册`TableSink`的程序，如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSink
TableSink csvSink = new CsvTableSink("/path/to/file", ...);

// define the field names and types
String[] fieldNames = {"a", "b", "c"};
TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};

// register the TableSink as table "CsvSinkTable"
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, csvSink);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// create a TableSink
val csvSink: TableSink = new CsvTableSink("/path/to/file", ...)

// define the field names and types
val fieldNames: Array[String] = Array("a", "b", "c")
val fieldTypes: Array[TypeInformation[_]] = Array(Types.INT, Types.STRING, Types.LONG)

// register the TableSink as table "CsvSinkTable"
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, csvSink)
{% endhighlight %}
</div>
</div>

{% top %}

注册一个外部目录
----------------------------

外部目录可用提供关于外部数据库和表的信息，例如名称，模式，统计信息及关于如何获取存在外部数据库，表或者文件的信息。

外部目录可用通过实现`ExternalCatalog`接口创建。

`TableEnvironment`下注册外部目的程序，如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create an external catalog
ExternalCatalog catalog = new InMemoryExternalCatalog();

// register the ExternalCatalog catalog
tableEnv.registerExternalCatalog("InMemCatalog", catalog);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// create an external catalog
val catalog: ExternalCatalog = new InMemoryExternalCatalog

// register the ExternalCatalog catalog
tableEnv.registerExternalCatalog("InMemCatalog", catalog)
{% endhighlight %}
</div>
</div>

在`TableEnvironment`下注册后，在`ExternalCatalog`中定义的所有表都可以通过指定其完整路径，例如`catalog.database.table`，利用Table API或者SQL查询进行访问。

目前，Flink提供`InMemoryExternalCatalog`，用于演示和测试。然而，`ExternalCatalog`接口也可将类似HCatalog或者Metastore的目录连接到Table API。

{% top %}

查询表
-------------

### Table API

Table API是用于Scala和Java的语言集成查询API。相较于SQL，查询不是用字符串指定，而是以主机语言逐步编写的。

该API是基于表示一个 表（streaming或者batch）的`Table`类，并提供关系操作的应用方法。这些方法返回一个新的`Table`对象，它表示在输入表上应用关系操作的结果。一些关系操作通过调用多个方法组成，例如`table.groupBy(...).select()`, 其中，`groupBy(...)`指定表的分组，`select(...)`指定在表分组中的投影。

关于支持streaming和batch tables的所有Table API操作，参阅[Table API]({{ site.baseurl }}/dev/table/tableApi.html)。

以下示例展示一个简单的Table API聚合查询：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table

// scan registered Orders table
Table orders = tableEnv.scan("Orders");
// compute revenue for all customers from France
Table revenue = orders
  .filter("cCountry === 'FRANCE'")
  .groupBy("cID, cName")
  .select("cID, cName, revenue.sum AS revSum");

// emit or convert Table
// execute query
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// register Orders table

// scan registered Orders table
Table orders = tableEnv.scan("Orders")
// compute revenue for all customers from France
Table revenue = orders
  .filter('cCountry === "FRANCE")
  .groupBy('cID, 'cName)
  .select('cID, 'cName, 'revenue.sum AS 'revSum)

// emit or convert Table
// execute query
{% endhighlight %}

**注意：**Scala Table API使用Scala符号，以单个tick (`'`)起始来引用`Table`的属性。通过，也使用Scala的隐式转换，所以确保已导入`org.apache.flink.api.scala._` 和 `org.apache.flink.table.api.scala._` 。

</div>

</div>

{% top %}

### SQL

Flink的SQL集成是基于[Apache Calcite](https://calcite.apache.org)，它实行SQL标准。SQL查询由常规的字符串指定。

关于描述在streaming和batch tables上的Flink的SQL的支持，参阅[SQL]({{ site.baseurl }}/dev/table/sql.html)。

以下示例展示如何指定查询并将结果作为`Table`返回：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table

// compute revenue for all customers from France
Table revenue = tableEnv.sqlQuery(
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// emit or convert Table
// execute query
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// register Orders table

// compute revenue for all customers from France
Table revenue = tableEnv.sqlQuery("""
  |SELECT cID, cName, SUM(revenue) AS revSum
  |FROM Orders
  |WHERE cCountry = 'FRANCE'
  |GROUP BY cID, cName
  """.stripMargin)

// emit or convert Table
// execute query
{% endhighlight %}

</div>
</div>

以下示例展示如何指定将结果插入注册表的更新查询：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.sqlUpdate(
    "INSERT INTO RevenueFrance " +
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// execute query
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.sqlUpdate("""
  |INSERT INTO RevenueFrance
  |SELECT cID, cName, SUM(revenue) AS revSum
  |FROM Orders
  |WHERE cCountry = 'FRANCE'
  |GROUP BY cID, cName
  """.stripMargin)

// execute query
{% endhighlight %}

</div>
</div>

{% top %}

### 混合Table API和SQL

Table API和SQL查询由于都返回`Table`对象，很容易混和：

* Table API查询可以由SQL查询返回的`Table`对象进行定义
* SQL查询可以由Table API查询的结果定义，通过在`TableEnvironment`下[注册结果Table](#register-a-table)并将其引用到SQL查询的`FORM`条款里

{% top %}

发送Table 
------------

`Table`是通过将其写入到一个`TableSink`里进行发送。`TableSink`是支持各种类型文件格式（例如CDV，Apache Parquet，Apache Avro），存储系统（例如，JDBC，Apache HBase，Apache Cassandra，Elasticsearch），或者消息传递系统（例如，Apache Kafka，RabbitMQ）。

一个batch `Table`只能被写入到`BatchTableSink`，但一个streaming `Table`可写入到`AppendStreamTableSink`，`RetractStreamTableSink`或者是 `UpsertStreamTableSink`。

关于可用sinks及有关如何是实现自定义TableSink的说明的具体详情，参阅[Table Source & Sinks]({{ site.baseurl }}/dev/table/sourceSinks.html)。

有两个发送表的方式：

1. `Table.writeToSink(TableSink sink)`：利用所提供的`TableSink`发送表并自动按表的模式配置sink
2. `Table.insertInto(String sinkTable)`：查找一个`TableSink`，并且其已经以所提供的名称在`TableEnvironment`的目录下注册过。将发射的表的模式根据已注册的`TableSink`的模式进行验证

以下示例会展示如何发射一个`Table`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// compute a result Table using Table API operators and/or SQL queries
Table result = ...

// create a TableSink
TableSink sink = new CsvTableSink("/path/to/file", fieldDelim = "|");

// METHOD 1:
//   Emit the result Table to the TableSink via the writeToSink() method
result.writeToSink(sink);

// METHOD 2:
//   Register the TableSink with a specific schema
String[] fieldNames = {"a", "b", "c"};
TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, sink);
//   Emit the result Table to the registered TableSink via the insertInto() method
result.insertInto("CsvSinkTable");

// execute the program
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// compute a result Table using Table API operators and/or SQL queries
val result: Table = ...

// create a TableSink
val sink: TableSink = new CsvTableSink("/path/to/file", fieldDelim = "|")

// METHOD 1:
//   Emit the result Table to the TableSink via the writeToSink() method
result.writeToSink(sink)

// METHOD 2:
//   Register the TableSink with a specific schema
val fieldNames: Array[String] = Array("a", "b", "c")
val fieldTypes: Array[TypeInformation] = Array(Types.INT, Types.STRING, Types.LONG)
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, sink)
//   Emit the result Table to the registered TableSink via the insertInto() method
result.insertInto("CsvSinkTable")

// execute the program
{% endhighlight %}
</div>
</div>

{% top %}


翻译并执行查询
-----------------------------

根据输入数据是streaming还是batch类型，Table API和SQL查询相应地被翻译为[DataStream]({{ site.baseurl }}/dev/datastream_api.html)或者[DataSet]({{ site.baseurl }}/dev/batch)程序。查询在内部表示一个逻辑查询计划，并分两个阶段翻译：

1. 逻辑计划的优化
2. 翻译为DataStream或者DataSet程序

在下列情况Table API或SQL查询会被翻译：

- `Table`被发射到`TableSink`，即，当`Table.writeToSink()`或者`Table.insertInto()`被调用
- 指定一个SQL更新查询，即，当`TableEnvironment.sqlUpdate()`被调用
- 表被转为`DataStream`或`DataSet`，参见[Integration with DataStream and DataSet API](#integration-with-dataStream-and-dataSet-api)。

一旦被翻译，Table API或SQL查询就会像常规DataStream或DataSet程序一样处理，且当调用StreamEnvironment.execute( )或ExecutionEnvironment.execute( )时，会被执行。

{% top %}

与DataStream和DataSet API集成
-------------------------------------------

Table API和SQL查询可以很容易地被集成并嵌入到[DataStream]({{ site.baseurl }}/dev/datastream_api.html)和[DataSet]({{ site.baseurl }}/dev/batch)程序。例如，可以查询外部表（例如来自RDBMS），执行一些预处理，例如筛选，投影，聚合或者添加元数据，然后用 DataStream或DataSet API(及建立在这些API之上的任何库，例如 CEP或Gelly)进一步处理数据。相反，Table API或SQL查询 也可以应用于DataStream 或DataSet程序的结果。

这种互动可以通过`DataStream`或`DataSet`与`Table`的相互转换实现。接下来，我们将介绍如何进行转换。

### 隐式转换Scala

The Scala Table API具有隐式转换为`DataSet`，`DataStream`和`Table`类的功能。这些转换是通过导入`org.apache.flink.table.api.scala._ `包实现的，另外，对于Scala DataStream API还需要导入`org.apache.flink.api.scala._`包。

### 将DataStream或DataSet作为Table注册

`DataStream`或`DataSet`可以在`TableEnvironment`下作为表进行注册。结果表的模式取决于已注册的`DataStream`或`DataSet`的数据类型，详情参阅[mapping of data types to table schema](#mapping-of-data-types-to-table-schema)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// register the DataStream as Table "myTable" with fields "f0", "f1"
tableEnv.registerDataStream("myTable", stream);

// register the DataStream as table "myTable2" with fields "myLong", "myString"
tableEnv.registerDataStream("myTable2", stream, "myLong, myString");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get TableEnvironment 
// registration of a DataSet is equivalent
val tableEnv = TableEnvironment.getTableEnvironment(env)

val stream: DataStream[(Long, String)] = ...

// register the DataStream as Table "myTable" with fields "f0", "f1"
tableEnv.registerDataStream("myTable", stream)

// register the DataStream as table "myTable2" with fields "myLong", "myString"
tableEnv.registerDataStream("myTable2", stream, 'myLong, 'myString)
{% endhighlight %}
</div>
</div>

**注意：**`DataStream` `Table`的名称不能与 `^_ DataStreamTable_ [0-9]+`模式匹配，且`DataSet` `Table`的名称不能与`^_ DataSetTable_[0-9]+`模式匹配。这些模式仅供内部使用。

{% top %}

### 将DataStream或DataSet转为Table

不同于在`TableEnvironment`下注册`DataStream`或`DataSet`，它可以直接转换为`Table`。如果你想在Table API查询中使用Table，这将会很方便。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// Convert the DataStream into a Table with default fields "f0", "f1"
Table table1 = tableEnv.fromDataStream(stream);

// Convert the DataStream into a Table with fields "myLong", "myString"
Table table2 = tableEnv.fromDataStream(stream, "myLong, myString");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get TableEnvironment
// registration of a DataSet is equivalent
val tableEnv = TableEnvironment.getTableEnvironment(env)

val stream: DataStream[(Long, String)] = ...

// convert the DataStream into a Table with default fields '_1, '_2
val table1: Table = tableEnv.fromDataStream(stream)

// convert the DataStream into a Table with fields 'myLong, 'myString
val table2: Table = tableEnv.fromDataStream(stream, 'myLong, 'myString)
{% endhighlight %}
</div>
</div>

{% top %}

### 将Table转化为DataStream或DataSet

`Table`可以被转为`DataStream`或`DataSet`。通过这种方式，自定义的DataStream或DataSet程序就可以在Table API或SQL查询结果中运行。

当将`Table`转为`DataStream`或`DataSet`时，你需要指定`DataStream`或`DataSe`t的结果数据类型，即，`Table`的行要转换的数据类型。通常，最方便的转换类型是`ROW`。以下列表对不同选项的功能进行了概述：

- **ROW**：字段按位置映射，支持任意数量字段，支持`null`值，无类型安全访问
- **POJO**：字段按名称映射(POJO字段必须命名为`Table`字段)，支持任意数量字段，支持空值，有类型安全访问
- **Case Class**：字段按位置映射，不支持`null`值，有类型安全访问
- **Tuple**：字段按位置映射，字段数量限制为22(Scala)或者25(Java)，不支持空值，有类型安全访问
- **Atomic Type**：`Table`必须具有独立字段，不支持`null`值，有类型安全访问

#### 将Table转化为DataStream

作为streaming查询结果的`Table`会自动更新，即，当有新记录到达查询输入流时，它会发生变化。因此，像自动查询转换的`DataStream`需要对表的更新进行编码。

有两种模式可以将`Table`转换为`DataStream`：

1. **Append Mode**：这种模式只能用于当动态`Table`仅通过`INSERT`更改进行修改情况下，即，只能追加且先前发出的结果不会更新
2. **Retract Mode**：始终可以使用此模式，它用一个布尔标志对`INSERT`和`DELETE`变更进行编码

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get StreamTableEnvironment. 
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into an append DataStream of Row by specifying the class
DataStream<Row> dsRow = tableEnv.toAppendStream(table, Row.class);

// convert the Table into an append DataStream of Tuple2<String, Integer> 
//   via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataStream<Tuple2<String, Integer>> dsTuple = 
  tableEnv.toAppendStream(table, tupleType);

// convert the Table into a retract DataStream of Row.
//   A retract stream of type X is a DataStream<Tuple2<Boolean, X>>. 
//   The boolean field indicates the type of the change. 
//   True is INSERT, false is DELETE.
DataStream<Tuple2<Boolean, Row>> retractStream = 
  tableEnv.toRetractStream(table, Row.class);

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get TableEnvironment. 
// registration of a DataSet is equivalent
val tableEnv = TableEnvironment.getTableEnvironment(env)

// Table with two fields (String name, Integer age)
val table: Table = ...

// convert the Table into an append DataStream of Row
val dsRow: DataStream[Row] = tableEnv.toAppendStream[Row](table)

// convert the Table into an append DataStream of Tuple2[String, Int]
val dsTuple: DataStream[(String, Int)] dsTuple = 
  tableEnv.toAppendStream[(String, Int)](table)

// convert the Table into a retract DataStream of Row.
//   A retract stream of type X is a DataStream[(Boolean, X)]. 
//   The boolean field indicates the type of the change. 
//   True is INSERT, false is DELETE.
val retractStream: DataStream[(Boolean, Row)] = tableEnv.toRetractStream[Row](table)
{% endhighlight %}
</div>
</div>

**注意：**有关动态表及其属性的详细介绍，参阅[Streaming Queries]({{ site.baseurl }}/dev/table/streaming.html)。

#### 将Table转化为DataSet

以下展示将`Table`转为`DataSet`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get BatchTableEnvironment
BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into a DataSet of Row by specifying a class
DataSet<Row> dsRow = tableEnv.toDataSet(table, Row.class);

// convert the Table into a DataSet of Tuple2<String, Integer> via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataSet<Tuple2<String, Integer>> dsTuple = 
  tableEnv.toDataSet(table, tupleType);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get TableEnvironment 
// registration of a DataSet is equivalent
val tableEnv = TableEnvironment.getTableEnvironment(env)

// Table with two fields (String name, Integer age)
val table: Table = ...

// convert the Table into a DataSet of Row
val dsRow: DataSet[Row] = tableEnv.toDataSet[Row](table)

// convert the Table into a DataSet of Tuple2[String, Int]
val dsTuple: DataSet[(String, Int)] = tableEnv.toDataSet[(String, Int)](table)
{% endhighlight %}
</div>
</div>

{% top %}

### 数据类型到表模式的映射

Flink的DataStream和DataSet APIs支持多种类型，例如Tuples (built-in Scala and Flink Java tuples), POJOs, case classes,和atomic types。下文中，将会描述Table API是如何将这些类型转换为内部行表示及展示将`DataStream`转为`Table`的示例。

数据类型映射到表模式可以通过两种方式：**基于字段位置**或**基于字段名称**。

**基于位置的映射**

基于位置的映射可为字段提供更有意义的名称，同时保持字段顺序。此映射可用于*具有定义字段顺序*的复合数据类型以及原子类型。例如元组，行和包含大小写这样的复合数据类型，都有字段顺序。但是，POJO的字段必须根据字段名映射（请参阅下一节）。

当定义一个基于位置的映射时，指定的名称不得存在于输入数据类型中，否则API会假定该映射是基于字段名发生。如果未指定字段名称，则使用复合类型的默认字段名称和默认字段顺序，若是原子类型则使用`f0`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field names "myLong" and "myInt"
Table table = tableEnv.fromDataStream(stream, "myLong, myInt");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

val stream: DataStream[(Long, Int)] = ...

// convert DataStream into Table with default field names "_1" and "_2"
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with field names "myLong" and "myInt"
val table: Table = tableEnv.fromDataStream(stream, 'myLong 'myInt)
{% endhighlight %}
</div>
</div>

**基于名称的映射**

基于名称的映射使用与任何数据类型，包括POJOs。它也是最灵活的定义表模式映射的方法。所有映射字段通过名称进行引用，也可用alias `as`重命名。字段可以被重新排序并投射出去。

如果没有指定字段名，对于符合类型，会使用默认字段名及字段顺序，对于atomic类型，使用`f0`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field "f1" only
Table table = tableEnv.fromDataStream(stream, "f1");

// convert DataStream into Table with swapped fields
Table table = tableEnv.fromDataStream(stream, "f1, f0");

// convert DataStream into Table with swapped fields and field names "myInt" and "myLong"
Table table = tableEnv.fromDataStream(stream, "f1 as myInt, f0 as myLong");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

val stream: DataStream[(Long, Int)] = ...

// convert DataStream into Table with default field names "_1" and "_2"
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with field "_2" only
val table: Table = tableEnv.fromDataStream(stream, '_2)

// convert DataStream into Table with swapped fields
val table: Table = tableEnv.fromDataStream(stream, '_2, '_1)

// convert DataStream into Table with swapped fields and field names "myInt" and "myLong"
val table: Table = tableEnv.fromDataStream(stream, '_2 as 'myInt, '_1 as 'myLong)
{% endhighlight %}
</div>
</div>

#### Atomic Types

Flink将原语(`Integer`, `Double`, `String`)或者通用类型(无法被分析和分解的类型)做为原子类型。`DataStream`或`DataSet`的原子类型被转换为具有单一属性的表。该属性的类型是从原子类型推断的，并且必须指定属性名称。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Long> stream = ...

// convert DataStream into Table with default field name "f0"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field name "myLong"
Table table = tableEnv.fromDataStream(stream, "myLong");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

val stream: DataStream[Long] = ...

// convert DataStream into Table with default field name "f0"
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with field name "myLong"
val table: Table = tableEnv.fromDataStream(stream, 'myLong)
{% endhighlight %}
</div>
</div>

#### Tuples (Scala and Java) and Case Classes (Scala only)

Flink支持Scala的内置元组，并为Java提供私有元组类。DataStreams和DataSets的这两种元组可以被转换为表。提供所有字段(基于位置映射)的名称后，可为字段重命名（基于位置映射）。如果没有指定字段名，则使用默认字段名。如果使用原始字段名 (`f0`, `f1`, ... for Flink Tuples and `_1`, `_2`, ... for Scala Tuples) ， API则认为映射是基于名称，而非基于位置。基于名称的映射允许重新对字段排序及用alias(`as`)投影。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// convert DataStream into Table with default field names "f0", "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myLong", "myString" (position-based)
Table table = tableEnv.fromDataStream(stream, "myLong, myString");

// convert DataStream into Table with reordered fields "f1", "f0" (name-based)
Table table = tableEnv.fromDataStream(stream, "f1, f0");

// convert DataStream into Table with projected field "f1" (name-based)
Table table = tableEnv.fromDataStream(stream, "f1");

// convert DataStream into Table with reordered and aliased fields "myString", "myLong" (name-based)
Table table = tableEnv.fromDataStream(stream, "f1 as 'myString', f0 as 'myLong'");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

val stream: DataStream[(Long, String)] = ...

// convert DataStream into Table with renamed default field names '_1, '_2
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with field names "myLong", "myString" (position-based)
val table: Table = tableEnv.fromDataStream(stream, 'myLong, 'myString)

// convert DataStream into Table with reordered fields "_2", "_1" (name-based)
val table: Table = tableEnv.fromDataStream(stream, '_2, '_1)

// convert DataStream into Table with projected field "_2" (name-based)
val table: Table = tableEnv.fromDataStream(stream, '_2)

// convert DataStream into Table with reordered and aliased fields "myString", "myLong" (name-based)
val table: Table = tableEnv.fromDataStream(stream, '_2 as 'myString, '_1 as 'myLong)

// define case class
case class Person(name: String, age: Int)
val streamCC: DataStream[Person] = ...

// convert DataStream into Table with default field names 'name, 'age
val table = tableEnv.fromDataStream(streamCC)

// convert DataStream into Table with field names 'myName, 'myAge (position-based)
val table = tableEnv.fromDataStream(streamCC, 'myName, 'myAge)

// convert DataStream into Table with reordered and aliased fields "myAge", "myName" (name-based)
val table: Table = tableEnv.fromDataStream(stream, 'age as 'myAge, 'name as 'myName)

{% endhighlight %}
</div>
</div>

#### POJO (Java and Scala)

Flink支持POJO作为复合类型。针对什么决定POJO的规则，可参阅[here]({{ site.baseurl }}/dev/api_concepts.html#pojos)。

当转换POJO DataStream或DataSet为表且没有指定字段名时，将使用原始的POJO字段名。对原始的POJO字段名重命名时要求使用关键字AS，因为POJO字段名没有固定顺序，名称映射要求原始名称且不能通过位置完成。对原始的POJO字段名重命名时要求使用alias(关键字AS)，reordered, 和 projected。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Person is a POJO with fields "name" and "age"
DataStream<Person> stream = ...

// convert DataStream into Table with default field names "age", "name" (fields are ordered by name!)
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed fields "myAge", "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "age as myAge, name as myName");

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, "name");

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// Person is a POJO with field names "name" and "age"
val stream: DataStream[Person] = ...

// convert DataStream into Table with default field names "age", "name" (fields are ordered by name!)
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with renamed fields "myAge", "myName" (name-based)
val table: Table = tableEnv.fromDataStream(stream, 'age as 'myAge, 'name as 'myName)

// convert DataStream into Table with projected field "name" (name-based)
val table: Table = tableEnv.fromDataStream(stream, 'name)

// convert DataStream into Table with projected and renamed field "myName" (name-based)
val table: Table = tableEnv.fromDataStream(stream, 'name as 'myName)
{% endhighlight %}
</div>
</div>

#### Row

`Row`数据类型支持任意数量字段，且支持`null`值。字段名可通过`RowTypeInfo`指定或者当`Row` `DataStream`或`DataSet`转换为表(基于位置)来指定。

The row type supports mapping of fields by position and by name. Fields can be renamed by providing names for all fields (mapping based on position) or selected individually for projection/ordering/renaming (mapping based on name).

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// DataStream of Row with two fields "name" and "age" specified in `RowTypeInfo`
DataStream<Row> stream = ...

// convert DataStream into Table with default field names "name", "age"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myName", "myAge" (position-based)
Table table = tableEnv.fromDataStream(stream, "myName, myAge");

// convert DataStream into Table with renamed fields "myName", "myAge" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName, age as myAge");

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, "name");

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// DataStream of Row with two fields "name" and "age" specified in `RowTypeInfo`
val stream: DataStream[Row] = ...

// convert DataStream into Table with default field names "name", "age"
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with renamed field names "myName", "myAge" (position-based)
val table: Table = tableEnv.fromDataStream(stream, 'myName, 'myAge)

// convert DataStream into Table with renamed fields "myName", "myAge" (name-based)
val table: Table = tableEnv.fromDataStream(stream, 'name as 'myName, 'age as 'myAge)

// convert DataStream into Table with projected field "name" (name-based)
val table: Table = tableEnv.fromDataStream(stream, 'name)

// convert DataStream into Table with projected and renamed field "myName" (name-based)
val table: Table = tableEnv.fromDataStream(stream, 'name as 'myName)
{% endhighlight %}
</div>
</div>

{% top %}


查询优化
------------------

Apache Flink利用Apache Calcite来优化和翻译查询。当前执行的优化包括投影和过滤器下推，非相关子查询，及其他类型的查询重写。Flink尚未优化连接的顺序，但会按照查询(`FROM`字句中表的顺序和/或`WHERE`字句中联接谓词的顺序)中定义的顺序执行。

通过提供`CalciteConfig`对象，可以调整在不同阶段应用的一组优化规则。可以通过调用`CalciteConfig.createBuilder()`的builder来创建，且通过调用`tableEnv.getConfig.setCalciteConfig(calciteConfig)`提供给`TableEnvironment`

### 解释Table

Table API提供了一种机制解释计算`Table`的逻辑和优化查询，这是通过`TableEnvironment.explain(table)`方法。它返回一个解释以下3个计划的字符串：

1. 关系查询的Abstract Syntax Tree，即，未优化的逻辑查询计划
2. 优化的逻辑查询计划
3. 物理执行计划

以下代码显示了一个示例和相应输出：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Integer, String>> stream1 = env.fromElements(new Tuple2<>(1, "hello"));
DataStream<Tuple2<Integer, String>> stream2 = env.fromElements(new Tuple2<>(1, "hello"));

Table table1 = tEnv.fromDataStream(stream1, "count, word");
Table table2 = tEnv.fromDataStream(stream2, "count, word");
Table table = table1
  .where("LIKE(word, 'F%')")
  .unionAll(table2);

String explanation = tEnv.explain(table);
System.out.println(explanation);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tEnv = TableEnvironment.getTableEnvironment(env)

val table1 = env.fromElements((1, "hello")).toTable(tEnv, 'count, 'word)
val table2 = env.fromElements((1, "hello")).toTable(tEnv, 'count, 'word)
val table = table1
  .where('word.like("F%"))
  .unionAll(table2)

val explanation: String = tEnv.explain(table)
println(explanation)
{% endhighlight %}
</div>
</div>

{% highlight text %}
== Abstract Syntax Tree ==
LogicalUnion(all=[true])
  LogicalFilter(condition=[LIKE($1, 'F%')])
    LogicalTableScan(table=[[_DataStreamTable_0]])
  LogicalTableScan(table=[[_DataStreamTable_1]])

== Optimized Logical Plan ==
DataStreamUnion(union=[count, word])
  DataStreamCalc(select=[count, word], where=[LIKE(word, 'F%')])
    DataStreamScan(table=[[_DataStreamTable_0]])
  DataStreamScan(table=[[_DataStreamTable_1]])

== Physical Execution Plan ==
Stage 1 : Data Source
  content : collect elements with CollectionInputFormat

Stage 2 : Data Source
  content : collect elements with CollectionInputFormat

  Stage 3 : Operator
    content : from: (count, word)
    ship_strategy : REBALANCE

    Stage 4 : Operator
      content : where: (LIKE(word, 'F%')), select: (count, word)
      ship_strategy : FORWARD
    
      Stage 5 : Operator
        content : from: (count, word)
        ship_strategy : REBALANCE
{% endhighlight %}

{% top %}



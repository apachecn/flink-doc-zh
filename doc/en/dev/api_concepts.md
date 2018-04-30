---
title: "Basic API Concepts"
nav-parent_id: dev
nav-pos: 1
nav-show_overview: true
nav-id: api-concepts
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

Flink程序是实现分布式集合转换操作（例如，filtering，mapping，updating state，joining，grouping，defining windows，aggregating）的一般程序.
集合最初是由一些source源（例如，读取一些文件、kafka topics、本地文件、内存中已有的集合）创建的. 结果经由sink接收器返回，例如可以将数据写入到（分布式）文件中，也可以作为标准输出（例如直接打印到命令行终端）.

Flink程序有多种运行的方式，可直接standalone单机操作，亦可嵌入到其他程序中。因此可由本地JVM或在多台机器组成的集群中执行。

根据不同的数据源类型，既有界或无界，你可以编写批处理程序或流失处理程序，其中DataSet API是应用于批处理场景，DataStream API是应用于流式处理场景。本指南将会介绍这两种API通用的基本概念，但涉及到API使用的具体信息，请参阅我们的[Streaming Guide]({{ site.baseurl }}/dev/datastream_api.html) and
[Batch Guide]({{ site.baseurl }}/dev/batch/index.html) 。

**注:**针对`StreamingExecutionEnvironment` and the `DataStream` API，我们将用实用的例子来做具体展示。这些概念也是适用于`DataSet` API，只是将被`ExecutionEnvironment` and `DataSet`取代.

* This will be replaced by the TOC
{:toc}

DataSet 和 DataStream
----------------------

在一个程序中，Flink可由一些特殊的`DataSet` and `DataStream`类来表示数据。你也可以将这些表示数据的类视为一些包含多个副本的不可变数据集合。在`DataSet` 数据有限的情况下，`DataStream` 元素的数量也可以是无限的。

这些集合在一些关键的方面是有区别于一般的Java集合。首先，它们是不可变的，即意味着一旦它们被创建那么就不能添加或删除元素.也不能轻易的去检查其内的元素.

一个集合的可由一个Flink程序通过添加相关source源来创建，也可以用一些 API方法来构建，例如 map、filter等等。

剖析一个Flink程序
--------------------------

Flink程序类似于数据集合转换的常规程序。这样的程序一般都由相同的一些基础部分构成：

1.  有一个`execution environment`（可执行环境）,
2.  加载/创建初始数据,
3.  按照指定的转换规则，对数据进行转换,
4.  将计算结果存放到指定的地方,
5.  触发程序执行

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

现在我们将会详述每一步的操作步骤，更多细节请参阅每个章节部分。值得注意的是，Java DataSet的每一个核心类都能在{% gh_link /flink-java/src/main/java/org/apache/flink/api/java "org.apache.flink.api.java" %}包中找到，Java DataSet API则是位于{% gh_link /flink-streaming-java/src/main/java/org/apache/flink/streaming/api "org.apache.flink.streaming.api" %}包中。

作为Flink程序的最基本组成部分 `StreamExecutionEnvironment` ，可以用以下静态类来获取之：

{% highlight java %}
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(String host, int port, String... jarFiles)
{% endhighlight %}

一般来说，你只需用到`getExecutionEnvironment()`，因为这样可以视context(环境)来适当调整程序：如果你在IDE内像执行一般的java程序那样执行你的Flink程序，那么它会生成一个本地环境来让你的程序运行在你的本地机器上。如果你通过打jar包，[命令行]({{ site.baseurl }}/ops/cli.html)调用的方式运行，那么Flink集群会执行你的主方法以及通过`getExecutionEnvironment()` 生成一个跟集群适配的执行环境执行环境，来使得你的程序允许在集群上。

对于指定的数据源，在可执行环境中可以有多种方式从文件中读取：可以逐行读取它，获取通过CSV文件的方式，也可以完全使用自定义的输入格式。如果只是将text文本文件作为连续的行读取，可以使用：

{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = env.readTextFile("file:///path/to/file");
{% endhighlight %}

这将会生成一个DataStream，然后你可以将其一些转换操作结合来衍生出成新的DataStreams。

通过在DataStream调用转换函数来应用这些转换操作。例如，一个映射的转换操作如下：

{% highlight java %}
DataStream<String> input = ...;

DataStream<Integer> parsed = input.map(new MapFunction<String, Integer>() {

```
@Override
public Integer map(String value) {
    return Integer.parseInt(value);
}
```

});
{% endhighlight %}

这将会把初始集合中的每个String字符串转化为Integer，从而生成一个新的DataStream。

如果你得到了包含你需要的最终结果的DataStream，你可以将其写入到外部系统或创建一个sink容器。下面就是一些创建sink容器的例子：

{% highlight java %}
writeAsText(String path)

print()
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

现在我们将会详述每一步的操作步骤，更多细节请参阅每个章节部分。值得注意的是，Scala DataSet API的每一个核心类都能在{{% gh_link /flink-scala/src/main/scala/org/apache/flink/api/scala "org.apache.flink.api.scala" %}包中找到，Scala DataStream API 则是位于{% gh_link /flink-streaming-scala/src/main/scala/org/apache/flink/streaming/api/scala "org.apache.flink.streaming.api.scala" %}包中。

作为Flink程序的最基本组成部分 `StreamExecutionEnvironment` ，可以用以下静态类来获取之：

{% highlight scala %}
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(host: String, port: Int, jarFiles: String*)
{% endhighlight %}

一般来说，你只需用到`getExecutionEnvironment()`，因为这样可以视context(环境)来适当调整程序：如果你在IDE内像执行一般的java程序那样执行你的Flink程序，那么它会生成一个本地环境来让你的程序运行在你的本地机器上。如果你通过打jar包，[命令行]({{ site.baseurl }}/ops/cli.html)调用的方式运行，那么Flink集群会执行你的主方法以及通过`getExecutionEnvironment()` 生成一个跟集群适配的执行环境执行环境，来使得你的程序允许在集群上。

对于指定的数据源，在可执行环境中可以有多种方式从文件中读取：可以逐行读取它，获取通过CSV文件的方式，也可以完全使用自定义的输入格式。如果只是将text文本文件作为连续的行读取，可以使用：

{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val text: DataStream[String] = env.readTextFile("file:///path/to/file")
{% endhighlight %}

这将会生成一个DataStream，然后你可以将其一些转换操作结合来衍生出成新的DataStreams。

通过在DataStream调用转换函数来应用这些转换操作。例如，一个映射的转换操作如下：

{% highlight scala %}
val input: DataSet[String] = ...

val mapped = input.map { x => x.toInt }
{% endhighlight %}

这将会把初始集合中的每个String字符串转化为Integer，从而生成一个新的DataStream。

如果你得到了包含你需要的最终结果的DataStream，你可以将其写入到外部系统或创建一个sink容器。下面就是一些创建sink容器的例子：

{% highlight scala %}
writeAsText(path: String)

print()
{% endhighlight %}

</div>
</div>

一旦你完成了整个程序的编写，那么需要在`StreamExecutionEnvironment`调用`execute()` 函数来**trigger the program execution**操作。根据不同的`ExecutionEnvironment` 会选择执行在本机或提交程序到集群上执行。

该 `execute()` 函数会返回一个`JobExecutionResult`，其中包含执行时间和累加器结果。

请参阅[流处理指南]({{ site.baseurl }}/dev/datastream_api.html)来了解关于流数据源和sink容器的更多信息以及更深入的了解关于在DataStream上可进行的的转换操作。

可以查看[批处理指南]({{ site.baseurl }}/dev/batch/index.html)来获取更多有关批处理数据源和sink容器的的信息以及更深入的了解在DataSet上可进行的一些转换操作。


{% top %}

惰性计算
---------------

所有的Flink程序都是惰性执行的：当已经执行了程序的主方法时，数据和转换操作不会被直接加载进来的。相反，每个操作都会被创建或添加到程序的计划中。当执行操作被`execute()` 函数显式触发后其请求执行环境，这些操作才会被真正执行。程序是在本地执行还是在集群执行取决于执行环境类型。

惰性计算可以让你构建复杂的程序并让Flink将其作为一个整体单元去执行计算。

{% top %}

Specifying Keys
---------------

一些转换操作（join，coGroup，KeyBy，groupBy）需要将一组元素定义成一个键。其他转换操作（Reduce、GroupReduce，Aggregate，Windows）允许数据在应用之前在键上进行分组。

一个DataSet被分组如

{% highlight java %}
DataSet<...> input = // [...]
DataSet<...> reduced = input
  .groupBy(/*define key here*/)
  .reduceGroup(/*do something*/);
{% endhighlight %}

而一个键在DataStream上被指定则要用
{% highlight java %}
DataStream<...> input = // [...]
DataStream<...> windowed = input
  .keyBy(/*define key here*/)
  .window(/*window specification*/);
{% endhighlight %}

Flink的数据模型不是基于键值对的，因此，你不需要将数据集打包成键值对。键是一种抽象的概念：它们经由实际数据调用一些函数来定义，以此来指示一些分组操作。

**注意:** 在下面的讨论中，我们将使用`DataStream`和`keyBy`的相关API。对于DataSet API 你只需用`DataSet` and `groupBy`将其替换。

### 用键来定义元祖
{:.no_toc}

最简单的一种情况就是给元祖中一个或多个字段进行分组。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0)
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(Int, String, Long)] = // [...]
val keyed = input.keyBy(0)
{% endhighlight %}
</div>
</div>

这个元祖就被分组到第一个字段上了（其中的整数字段）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0,1)
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataSet[(Int, String, Long)] = // [...]
val grouped = input.groupBy(0,1)
{% endhighlight %}
</div>
</div>

下面我们根据第一和第二个字段组成的组合字段对元祖进行分组。

关于嵌套元祖需要注意的是：如果你有一个组合键DatsStream，如下所示：

{% highlight java %}
DataStream<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
{% endhighlight %}

指定`keyBy(0)`将会导致程序使用整个`Tuple2`作为键（就是使用整数或浮点数作为键）。如果你想使用嵌套的`Tuple2`作为键，那么你就要用到下面的字段表达式键。

Specifying `keyBy(0)` will cause the system to use the full `Tuple2` as a key (with the Integer and Float being the key). If you want to "navigate" into the nested `Tuple2`, you have to use field expression keys which are explained below.

### 使用字段表示式键
{:.no_toc}

你可以使用 基于字符串字段表达式来引用嵌套字段并对其定义grouping、sorting、joining以及coGrouping。

字段表达式可以很容易的选择(嵌套)复合类型的字段，例如[Tuple](#tuples-and-case-classes)和[POJO](#pojos)类型。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

在下面的例子中，我们有个包含了“word”与“count”字段的wc对象。我们只要将字段名称传给`keyBy()`函数，就能完成对字段的分组。

{% highlight java %}
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word;
  public int count;
}
DataStream<WC> words = // [...]
DataStream<WC> wordCounts = words.keyBy("word").window(/*window specification*/);
{% endhighlight %}

**字段表达式语法**:

- 从对象的字段值中选择相应的字段。例如`"user"`指的是POJO类型中的“用户”字段
- 通过偏移量字段索引和字段名称来选择元祖字段。例如`"f0"` 和`"5"`分别就是Java Tuple类型中的第一和第五个字段。
- 在POJO类型和元祖中你也可以选择嵌套字段。例如`"user.zip"`指的就是POJO中的“zip”字段，其存在于“user”字段类型中。任意嵌套的或混合的POJO对象都支持类似于`"f1.user.zip"` 或 `"user.f3.1.zip"`这种表示形式。
- 你也可以使用通配符`"*"` 来表示完整的字段。这也适用于非元祖和POJO的类型。

**字段表达式示例**:

{% highlight java %}
public static class WC {
  public ComplexNestedClass complex; //nested POJO
  private int count;
  // getter / setter for private field (count)
  public int getCount() {
    return count;
  }
  public void setCount(int c) {
    this.count = c;
  }
}
public static class ComplexNestedClass {
  public Integer someNumber;
  public float someFloat;
  public Tuple3<Long, Long, String> word;
  public IntWritable hadoopCitizen;
}
{% endhighlight %}

以上示例代码的有效字段表达式如下:

- `"count"`: wc类中的count字段
- `"complex"`:递归地选择POJO类型complexneteclass的字段复合体的所有字段
- `"complex.word.f2"`:选择嵌套字段`Tuple3`中的所有字段
- `"complex.hadoopCitizen"`:选择Hadoop中的`IntWritable` 类型

</div>
<div data-lang="scala" markdown="1">

在下面的例子中，我们有个包含了“word”与“count”字段的wc对象。我们只要将字段名称传给`keyBy()`函数，就能完成对字段的分组。
{% highlight java %}
// some ordinary POJO (Plain old Java Object)
class WC(var word: String, var count: Int) {
  def this() { this("", 0L) }
}
val words: DataStream[WC] = // [...]
val wordCounts = words.keyBy("word").window(/*window specification*/)

// or, as a case class, which is less typing
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val wordCounts = words.keyBy("word").window(/*window specification*/)
{% endhighlight %}

**字段表达式语法**:

- 从对象的字段值中选择相应的字段。例如`"user"`指的是POJO类型中的“用户”字段
- 通过偏移量字段索引和字段名称来选择元祖字段。例如`"f0"` 和`"5"`分别就是Scala Tuple类型中的第一和第五个字段。
- 在POJO类型和元祖中你也可以选择嵌套字段。例如`"user.zip"`指的就是POJO中的“zip”字段，其存在于“user”字段类型中。任意嵌套的或混合的POJO对象都支持类似于`"f1.user.zip"` 或 `"user.f3.1.zip"`这种表示形式。
- 你也可以使用通配符`"*"` 来表示完整的字段。这也适用于非元祖和POJO的类型。

**字段表达式示例**:

{% highlight scala %}
class WC(var complex: ComplexNestedClass, var count: Int) {
  def this() { this(null, 0) }
}

class ComplexNestedClass(
    var someNumber: Int,
    someFloat: Float,
    word: (Long, Long, String),
    hadoopCitizen: IntWritable) {
  def this() { this(0, 0, (0, 0, ""), new IntWritable(0)) }
}
{% endhighlight %}

- 以上示例代码的有效字段表达式如下:
  - `"count"`: wc类中的count字段
  - `"complex"`:递归地选择POJO类型complexneteclass的字段复合体的所有字段
  - `"complex.word.f2"`:选择嵌套字段`Tuple3`中的所有字段
  - `"complex.hadoopCitizen"`:选择Hadoop中的`IntWritable` 类型

</div>
</div>

### 使用键选择器函数定义键
{:.no_toc}

另一种定义键的方法是“键选择器”函数。键选择器函数将单个元素作为输入并返回该元素的键。键可以是任何类型，并可以从一定计算之后推导出。

下面的例子就展示了一个简单的拥有一个返回对象的键选择器函数：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// some ordinary POJO
public class WC {public String word; public int count;}
DataStream<WC> words = // [...]
KeyedStream<WC> keyed = words
  .keyBy(new KeySelector<WC, String>() {
     public String getKey(WC wc) { return wc.word; }
   });
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// some ordinary case class
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val keyed = words.keyBy( _.word )
{% endhighlight %}
</div>
</div>

{% top %}

指定转换函数
--------------------------

大部分的转换操作都需要用户来定义一些函数。本节列出了定义这些函数的多种方式。

Most transformations require user-defined functions. This section lists different ways
of how they can be specified

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

#### 实现一个接口

最基本的方法就是实现已知的接口：

{% highlight java %}
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
data.map(new MyMapFunction());
{% endhighlight %}

#### 匿名函数

你可以将一个函数作为匿名类传递：

{% highlight java %}
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

#### Java 8 Lambdas

Flink也支持Java API中的 Java 8 Lambda表达式。可以参阅完整的[Java 8 Guide]({{ site.baseurl }}/dev/java8.html)来了解。

{% highlight java %}
data.filter(s -> s.startsWith("http://"));
{% endhighlight %}

{% highlight java %}
data.reduce((i1,i2) -> i1 + i2);
{% endhighlight %}

#### Rich functions

所有经由用户定义的转换操作函数都可以作为一个*rich* function的参数。例如，这种函数你不能这样写

{% highlight java %}
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
{% endhighlight %}

而是要这样写

{% highlight java %}
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
{% endhighlight %}

并将此函数传递给`map`转换操作就像其他传递给map的函数一样：

{% highlight java %}
data.map(new MyMapFunction());
{% endhighlight %}

Rich functions也可以定义成一个匿名类：

{% highlight java %}
data.map (new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">


#### Lambda 函数

正如前面所看到的，所有的操作都可用lambda函数进行描述：

{% highlight scala %}
val data: DataSet[String] = // [...]
data.filter { _.startsWith("http://") }
{% endhighlight %}

{% highlight scala %}
val data: DataSet[Int] = // [...]
data.reduce { (i1,i2) => i1 + i2 }
// or
data.reduce { _ + _ }
{% endhighlight %}

#### Rich functions

所有经由用户定义的转换操作函数都可以作为一个*rich* function的参数。例如，这种函数你不能这样写

{% highlight scala %}
data.map { x => x.toInt }
{% endhighlight %}

而是要这样写

{% highlight scala %}
class MyMapFunction extends RichMapFunction[String, Int] {
  def map(in: String):Int = { in.toInt }
};
{% endhighlight %}

并将此函数传递给`map`转换操作就像其他传递给map的函数一样：

{% highlight scala %}
data.map(new MyMapFunction())
{% endhighlight %}

Rich functions也可以定义成一个匿名类：
{% highlight scala %}
data.map (new RichMapFunction[String, Int] {
  def map(in: String):Int = { in.toInt }
})
{% endhighlight %}
</div>

</div>

rich 函数提供除了用户自定义的函数（map、reduce等）外，还有四个方法：`open`, `close`, `getRuntimeContext`, and
`setRuntimeContext`。这些对于参数化的函数很有用（请参阅 [Passing Parameters to Functions]({{ site.baseurl }}/dev/batch/index.html#passing-parameters-to-functions)），这些函数如，创建和确定本地状态，访问广播变量(请参阅
[Broadcast Variables]({{ site.baseurl }}/dev/batch/index.html#broadcast-variables))，以及访问运行时的信息，例如累加器和计数器还有迭代(参见 [Iterations]({{ site.baseurl }}/dev/batch/iterations.html))。

{% top %}

支持的数据类型
--------------------

Flink对诸如存在于 DataSet 或 DataStream的一些元素类型有一些限制条件。这也是系统在分析了一些确定有效的执行策略后而得出的一个原因。

有六种不同的数据类型:

1. **Java 元祖** and **Scala Case 类**
2. **Java POJOs**
3. **原始类型**
4. **常规类**
5. **值**
6. **实现了Hadoop的序列化类**
7. **特殊类型**

#### 元祖和Case类

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

元组是由多种固定数目且不同类型字段组成的复合字段。Java API 从`Tuple1` 到 `Tuple25`提供了多种类。每一个元组的里面都可以包含任意的Flink类型甚至包含元组型字段，从而生成嵌套元祖。元组中的每一个字段都可以通过字段的名字直接访问，例如`tuple.f4`，，或者是用一般的getter方法，例如`tuple.getField(int position)`。字段的索引从0开始。请注意，这里与Scala元祖是相反的，但它更符合Java的一般索引表示形式。

{% highlight java %}
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public Integer map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});

wordCounts.keyBy(0); // also valid .keyBy("f0")


{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

Scala case类型（Scala元组是case类的一种特殊的情况），是由多种固定数目且不同类型字段组成的复合字段。元组的字段值的获取是通过偏移量来解决的，例如 `_1`就代表第一个字段。case类字段可以由他们的名字访问。

{% highlight scala %}
case class WordCount(word: String, count: Int)
val input = env.fromElements(
    WordCount("hello", 1),
    WordCount("world", 2)) // Case Class Data Set

input.keyBy("word")// key by field expression "word"

val input2 = env.fromElements(("hello", 1), ("world", 2)) // Tuple2 Data Set

input2.keyBy(0, 1) // key by field positions 0 and 1
{% endhighlight %}

</div>
</div>

#### POJO类型数据

在满足以下几点的情况下，Java和Scala类将会被Flink程序视为特殊的POJO数据类型处理：

- 类是由public修饰的的。
- 必须有一个无参的public的构造方法（默认构造方法）。
- 所有字段都是public的或都可由getter、setter方法访问。对于一个名为`foo`的字段，其getter和setter方法必须命名为`getFoo()` 和 `setFoo()`。
- 字段的类型必须被Flink程序支持读写。目前，Flink使用[Avro](http://avro.apache.org)来序列化任意对象（例如`Date`）。

Flink程序通过分析POJP数据类型的构造方法，了解了POJO的各个字段的概况。因此POJO类型比一般的数据类型更容易使用。此外，Flink程序处理POJP类型数据也比一般数据类型更便捷的处理。

下面就展示了一个包含了两个public字段的POJO对象的简单例子。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class WordWithCount {

    public String word;
    public int count;
    
    public WordWithCount() {}
    
    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}

DataStream<WordWithCount> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));

wordCounts.keyBy("word"); // key by field expression "word"

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
class WordWithCount(var word: String, var count: Int) {
    def this() {
      this(null, -1)
    }
}

val input = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2)) // Case Class Data Set

input.keyBy("word")// key by field expression "word"

{% endhighlight %}
</div>
</div>

#### 原始数据类型

Flink程序支持所有的Java和Scala原始数据类型，例如`Integer`, `String`, 和 `Double`。

#### General Class Types

Flink程序支持大部分的Java和Scala类。（内置API和自定义类）。主要是对一些包含了无法序列化字段的类有访问限制，例如文件指针、I/O流、或者一些本地类库里面的类。Flink程序也可以很好的处理那些遵循了Java Bean的规范的类。

所有未被被确定为非POJO的类（参考上面的对POJO类型的要求）都会被视为一般的class类。Flink程序就像对待黑盒子一样对待这些数据类型，无法访问其中的内容（例如对其进行高效的排序）通常是用[Kryo](https://github.com/EsotericSoftware/kryo)序列化框架对一些类型进行序列化。

#### Values

*值*类型需要手动的去描述他们的序列化和反序列化。并不是通过通用的序列化框架来实现，而是通过实现`org.apache.flinktypes.Value` 接口以及其中的`read` and `write`方法来对这些操作提供固定的代码块。当无法通过使用通用的序列化框架对其进行序列化操作的时候，这种Value类型是非常合适的。示例是一个将稀疏的向量元素作为数组实现的数据类型，已经数组中大多元素都为0，可以为这些非0元素套用特殊的编码，而通用的序列化框架将会很容易的对其进行写入操作。

`org.apache.flinktypes.CopyableValue`接口也是用类似的方式于使用手动实现的内部克隆逻辑。

Flink附带了与基本数据类型对应的预定义值类型，例如：`ByteValue`,
`ShortValue`, `IntValue`, `LongValue`, `FloatValue`, `DoubleValue`, `StringValue`, `CharValue`,
`BooleanValue`，这些Value类型可以视为基本数据类型的可变变体：它们的值是可改的，允许使用者重用这些对象并减轻垃圾回收器的压力。


#### Hadoop Writables

你也可以使用继承了`org.apache.hadoop.Writable` 接口的数据类型。定义在`write()`and `readFields()`里面的序列化逻辑将会被用于序列化操作。

#### 特殊类型

你也可以使用一些特殊类型，包括Scala的 `Either`, `Option`, 和 `Try`。Java API也自定义实现了`Either`。与Scala的`Either`类似，其代表了 *Left* or *Right* 两种可能的值。`Either` 可以应用于错误的操作或者是需要输出两种不同类型的值的记录。

#### 类型擦除和类型判断

*注意: 这一章节仅与Java相关.*

Java编译器在编译后将会抛出许多的泛型类型信息。这在Java中被称为*类型擦除*。这就意味着在程序运行期间，对象实例将不再考虑泛型类型。举个例子，`DataStream<String>` 和 `DataStream<Long>`的实例从JVM的角度来看是如此一致。

Flink需要一些类型信息，在其执行程序的运行的时候（此时主方法被程序调用）。Fink的Java API试图重建那些以各种方式丢失的类型信息并将其显式的存在到数据集和运算符中。你可以通过`DataStream.getType()`得到这些类型。这个方法将会返回一个`TypeInformation`的实例，这是Flink内部表示类型的方式。

在某些情况下，类型推断还是存在自身的局限性，因此这时候就需要与开发者进行“合作”推断。这样的例子可以是从集合中创建数据集的方法，比如`ExecutionEnvironment.fromCollection(),`在那里你可以传递描述类型的参数。但是如泛型函数如`MapFunction<I, O>`就需要一些额外的类型信息。

The
{% gh_link /flink-core/src/main/java/org/apache/flink/api/java/typeutils/ResultTypeQueryable.java "ResultTypeQueryable" %}
interface can be implemented by input formats and functions to tell the API
explicitly about their return type. The *input types* that the functions are invoked with can
usually be inferred by the result types of the previous operations.

{% top %}

累加器和计时器
---------------------------

累加器的结构简单且有**添加操作**和**最终结果累计**的功能，是在任务结束后使用。

最直接的累加器就是一个**计数器r**：你可以使用```Accumulator.add(V value)```方法来让其递增。在任务执行的最后要统计（汇聚）下局部结果的总数，然后将结果发送到客户端。在debugging的时候或者你想快速的找到有关于你数据的更多信息的时候，累加器是很有用的。

Flink目前有如下的 **内置累加器**。它们都实现了{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java "Accumulator" %}的接口。

- {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/IntCounter.java "__IntCounter__" %},
  {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/LongCounter.java "__LongCounter__" %}
  and {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/DoubleCounter.java "__DoubleCounter__" %}:

  请参阅下面有关于累加器的示例。

- {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Histogram.java "__Histogram__" %}:

  一个关于多个离线的且有多个的箱子的直方图实现。其内部只是由Integer到Integer的映射。你可以用它来计算值的分布情况，例如，在一个wordcount程序中的每一行值的分布。

__如何使用累加器:__

首先当你使用它的时候你需要在用户定义的转换函数里面创建一个累加器对象（这是一个计数器）。

{% highlight java %}
private IntCounter numLines = new IntCounter();
{% endhighlight %}

第二步你需要注册这个累加器对象，一般来说是在*rich*函数的```open()```方法里。这里你也可以定义它的名字。

{% highlight java %}
getRuntimeContext().addAccumulator("num-lines", this.numLines);
{% endhighlight %}

你可以在操作函数的任意地方使用这个累加器，包括 ```open()``` 和
```close()```方法中。

{% highlight java %}
this.numLines.add(1);
{% endhighlight %}

最终的总的结果是存放在```JobExecutionResult```对象中，该对象是从可执行的环境下经由`execute()`方法返回的（现在只有在为完成的一个job在执行等待的时候才会起作用）。

{% highlight java %}
myJobExecutionResult.getAccumulatorResult("num-lines")
{% endhighlight %}

所有的累加器都有一个单独的名称空间。然后你可以在任务的不同操作函数中使用同一个累加器。Flink会在内部对所有的同一个名称的累加器的结果进行整合。

关于累加器和迭代器的说明：

当前累加器的结果仅仅适用于在整体的任务已经结束的前提下。我们还有计划在下一个迭代中可以实现之前迭代的产生的结果。你可以用{% gh_link /flink-java/src/main/java/org/apache/flink/api/java/operators/IterativeDataSet.java#L98 "Aggregators" %}来计算每次迭代产生的统计数据，并在这些数据的基础上终止迭代。

__自定义累加器:__

要实现你自己的累加器，你只需要编写实现累加器的接口即可。如果你的自定义累加器是与Flink程序一起产生的，那么需要随时创建一个pull请求。

你可以选择实现{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java "Accumulator" %}
或 {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/SimpleAccumulator.java "SimpleAccumulator" %}.

```Accumulator<V,R>```是相当灵活的：它定义了一个可以增加的类型```V```的值，以及一个类型为```R```的最终结果。例如 对于一个直方图，```V``` 就是表示的数字，```R```就是直方图。```SimpleAccumulator```也适用于两种类型的相同情况。例如：计数器。

```

```

{% 回到顶部 %}

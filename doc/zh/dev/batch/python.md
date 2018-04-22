---
title: "Python Programming Guide"
is_beta: true
nav-title: Python API
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

Flink中的分析程序是实施数据集转换（例如filtering, mapping, joining, grouping）的常规程序。
数据集最初是从某些来源创建的（例如，通过读取文件或从集合中创建）。结果通过 sink 返回，例如可以将数据写入（分布式）文件，或者写入标准输出（例如命令行终端）。
Flink程序可以在各种情况下运行，可以独立运行，也可以嵌入其他程序中。执行可以发生在本地JVM或许多机器的集群中。

为了创建您自己的Flink程序，我们鼓励您从[程序骨架](#program-skeleton)开始，逐步添加自己的[transformations](#transformations)。
其余部分充当其他算子和高级功能的参考。

* This will be replaced by the TOC
{:toc}

示例程序
---------------

以下程序是WordCount的完整工作示例。您可以复制并粘贴代码以在本地运行。

{% highlight python %}
from flink.plan.Environment import get_environment
from flink.functions.GroupReduceFunction import GroupReduceFunction

class Adder(GroupReduceFunction):
  def reduce(self, iterator, collector):
    count, word = iterator.next()
    count += sum([x[0] for x in iterator])
    collector.collect((count, word))

env = get_environment()
data = env.from_elements("Who's there?",
 "I think I hear them. Stand, ho! Who's there?")

data \
  .flat_map(lambda x, c: [(1, word) for word in x.lower().split()]) \
  .group_by(1) \
  .reduce_group(Adder(), combinable=True) \
  .output()

env.execute(local=True)
{% endhighlight %}

{% top %}

程序骨架
----------------

正如我们在例子中已经看到的，Flink程序看起来像普通的Python程序。
每个程序由相同的基本部分组成：

1. 获取一个 `Environment`,
2. 加载/创建 初始数据,
3. 指定此数据的transformations,
4. 指定放置计算结果的位置, 并且
5. 执行你的程序。

我们现在将对每个步骤进行概述，但请参阅各个部分了解更多详细信息。


 `Environment` 对于所有 Flink 程序来说是最基础的。你可以在`Environment`类上使用这些静态方法获得一个：

{% highlight python %}
get_environment()
{% endhighlight %}

为了指定数据源，执行环境有几种从文件读取的方法。
要仅将文本文件作为一系列行读取，可以使用：

{% highlight python %}
env = get_environment()
text = env.read_text("file:///path/to/file")
{% endhighlight %}

这会给你一个DataSet，然后你可以应用转换。 有关数据源和输入格式的更多信息，
请参阅[Data Sources](#data-sources)。


一旦你有了一个DataSet，你可以应用转换来创建一个新的DataSet，然后你可以写入一个文件，再次转换或与其他DataSet结合。
您可以使用您自己的自定义转换函数通过调用DataSet上的方法来应用转换。 例如，Map转换如下所示：

{% highlight python %}
data.map(lambda x: x*2)
{% endhighlight %}

这将通过将原始DataSet中的每个值加倍来创建一个新的DataSet。
有关更多信息和所有转换的列表，请参阅[Transformations](#transformations)。


一旦你有一个需要被写入磁盘的数据集，你可以在DataSet上调用其中的一个方法：

{% highlight python %}
data.write_text("<file-path>", WriteMode=Constants.NO_OVERWRITE)
write_csv("<file-path>", line_delimiter='\n', field_delimiter=',', write_mode=Constants.NO_OVERWRITE)
output()
{% endhighlight %}


最后一种方法只对本地机器上的开发/调试很有用，它会将DataSet的内容输出到标准输出。
（请注意，在群集中，结果将转到群集节点的标准输出流，并结束于worker节点的*.out* 文件中）。
前两个顾名思义。 有关写入文件的更多信息，请参阅[Data Sinks](#data-sinks)。


一旦你指定完整的程序，你需要调用环境上的执行。
这可以在您的本地机器上执行，也可以将您的程序提交给群集执行，具体取决于Flink的启动方式。
您可以使用`execute(local=True)`强制执行本地执行。

{% top %}

Project setup
---------------

除了设置Flink之外，不需要额外的工作。python包可以在Flink发行版的/resource文件夹中找到。运行作业时，flink软件包以及计划和可选软件包将通过HDFS自动分发到群集中。

Python API已在安装了Python 2.7或3.4的Linux / Windows系统上进行测试。

默认情况下，Flink将通过调用“python”来启动python进程。通过设置flink-conf.yaml中的“python.binary.path”键，您可以修改此行为以使用您选择的二进制文件。

{% top %}

Lazy Evaluation
---------------

所有Flink程序都会被懒执行：当程序的主要方法被执行时，数据加载和转换不会直接发生。
相反，每个算子都会创建并添加到程序的计划中。当在Environment对象上调用一个`execute()`方法时，这些算子实际上被执行。
程序是在本地执行还是在群集上执行取决于程序的环境。

lazy evaluation 让您可以构建Flink作为整体计划单元执行的复杂程序。

{% top %}


Transformations
---------------

Data transformations将一个或多个数据集转换为新的数据集。 程序可以将多个转换组合到复杂的程序集中。


本节简要介绍可用的transformations。[transformations 文档](dataset_transformations.html)用示例详细描述了所有transformations。

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>Map</strong></td>
      <td>
        <p>Takes one element and produces one element.</p>
{% highlight python %}
data.map(lambda x: x * 2)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>Takes one element and produces zero, one, or more elements. </p>
{% highlight python %}
data.flat_map(
  lambda x,c: [(1,word) for word in line.lower().split() for line in x])
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
        <p>Transforms a parallel partition in a single function call. The function get the partition
        as an `Iterator` and can produce an arbitrary number of result values. The number of
        elements in each partition depends on the parallelism and previous operations.</p>
{% highlight python %}
data.map_partition(lambda x,c: [value * 2 for value in x])
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>Evaluates a boolean function for each element and retains those for which the function
        returns true.</p>
{% highlight python %}
data.filter(lambda x: x > 1000)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>Combines a group of elements into a single element by repeatedly combining two elements
        into one. Reduce may be applied on a full data set, or on a grouped data set.</p>
{% highlight python %}
data.reduce(lambda x,y : x + y)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>Combines a group of elements into one or more elements. ReduceGroup may be applied on a
        full data set, or on a grouped data set.</p>
{% highlight python %}
class Adder(GroupReduceFunction):
  def reduce(self, iterator, collector):
    count, word = iterator.next()
    count += sum([x[0] for x in iterator)      
    collector.collect((count, word))

data.reduce_group(Adder())
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>Performs a built-in operation (sum, min, max) on one field of all the Tuples in a
        data set or in each group of a data set. Aggregation can be applied on a full dataset
        or on a grouped data set.</p>
{% highlight python %}
# This code finds the sum of all of the values in the first field and the maximum of all of the values in the second field
data.aggregate(Aggregation.Sum, 0).and_agg(Aggregation.Max, 1)

# min(), max(), and sum() syntactic sugar functions are also available
data.sum(0).and_agg(Aggregation.Max, 1)
{% endhighlight %}
      </td>
    </tr>

    </tr>
      <td><strong>Join</strong></td>
      <td>
        Joins two data sets by creating all pairs of elements that are equal on their keys.
        Optionally uses a JoinFunction to turn the pair of elements into a single element.
        See <a href="#specifying-keys">keys</a> on how to define join keys.
{% highlight python %}
# In this case tuple fields are used as keys.
# "0" is the join field on the first tuple
# "1" is the join field on the second tuple.
result = input1.join(input2).where(0).equal_to(1)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>CoGroup</strong></td>
      <td>
        <p>The two-dimensional variant of the reduce operation. Groups each input on one or more
        fields and then joins the groups. The transformation function is called per pair of groups.
        See <a href="#specifying-keys">keys</a> on how to define coGroup keys.</p>
{% highlight python %}
data1.co_group(data2).where(0).equal_to(1)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>Builds the Cartesian product (cross product) of two inputs, creating all pairs of
        elements. Optionally uses a CrossFunction to turn the pair of elements into a single
        element.</p>
{% highlight python %}
result = data1.cross(data2)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>Produces the union of two data sets.</p>
{% highlight python %}
data.union(data2)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>ZipWithIndex</strong></td>
      <td>
        <p>Assigns consecutive indexes to each element. For more information, please refer to
        the [Zip Elements Guide](zip_elements_guide.html#zip-with-a-dense-index).</p>
{% highlight python %}
data.zip_with_index()
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

{% top %}


Specifying Keys
-------------

某些转换（如Join或CoGroup）要求在其参数DataSets上定义一个键，
而其他转换（Reduce，GroupReduce）则允许DataSet在应用之前在键上进行分组。

A DataSet is grouped as
{% highlight python %}
reduced = data \
  .group_by(<define key here>) \
  .reduce_group(<do something>)
{% endhighlight %}

Flink的数据模型不基于键值对。因此，您不需要将数据集类型物理地打包成键和值。
建是“虚拟的”：它们被定义为实际数据上的函数来指导分组算子。

###为Tuples定义key
{:.no_toc}

最简单的情况是通过元组的一个或多个字段上分组Tuple的数据集：
{% highlight python %}
reduced = data \
  .group_by(0) \
  .reduce_group(<do something>)
{% endhighlight %}

数据集被分组在元组的第一个字段上。
group-reduce函数将因此接收第一个字段中具有相同值的元组的组。

{% highlight python %}
grouped = data \
  .group_by(0,1) \
  .reduce(/*do something*/)
{% endhighlight %}

数据集按照由第一和第二个字段组成的组合键进行分组，因此reduce函数将接收两个字段具有相同值的组。

关于嵌套元组的一个注意事项：如果你有一个带有嵌套元组的数据集，指定`group_by(<index of tuple>)`将导致系统使用完整的元组作为键。
{% top %}


将Functions传递给Flink
--------------------------

某些算子需要用户定义的函数，而所有这些算子都接受lambda函数和rich函数作为参数。

{% highlight python %}
data.filter(lambda x: x > 5)
{% endhighlight %}

{% highlight python %}
class Filter(FilterFunction):
    def filter(self, value):
        return value > 5

data.filter(Filter())
{% endhighlight %}

rich函数允许使用导入的函数，提供对广播变量的访问，可以使用 __init__()进行参数化，并且是复杂函数的前往选项。
它们也是为reduce 算子 定义可选的“combine”功能的唯一方法。

Lambda函数允许轻松插入 one-liners。请注意，如果算子可以返回多个值，则lambda函数必须返回一个迭代器。（所有接收收集器参数的函数）

{% top %}

数据类型
----------

Flink的Python API目前仅提供对原始python类型（int，float，bool，string）和字节数组的原生支持。

类型支持可以通过将序列化器，反序列化器和类型类传递给环境来扩展。
{% highlight python %}
class MyObj(object):
    def __init__(self, i):
        self.value = i


class MySerializer(object):
    def serialize(self, value):
        return struct.pack(">i", value.value)


class MyDeserializer(object):
    def _deserialize(self, read):
        i = struct.unpack(">i", read(4))[0]
        return MyObj(i)


env.register_custom_type(MyObj, MySerializer(), MyDeserializer())
{% endhighlight %}

#### 元祖/列表

您可以将元组（或列表）用于复合类型。Python元组映射到Flink Tuple类型，
其中包含固定数量的各种类型的字段（最多25个）。元组的每个字段都可以是一个基本类型 - 包括嵌套元组。

{% highlight python %}
word_counts = env.from_elements(("hello", 1), ("world",2))

counts = word_counts.map(lambda x: x[1])
{% endhighlight %}

在处理需要用于分组或匹配记录的键的算子时，元组让您只需指定要用作键的字段的位置。
您可以指定多个位置来使用组合键（请参见 [数据Transformations](#transformations)）。

{% highlight python %}
wordCounts \
    .group_by(0) \
    .reduce(MyReduceFunction())
{% endhighlight %}

{% top %}

数据源
------------

数据源创建初始数据集，例如从文件或集合中创建。

基于文件:

- `read_text(path)` - 读取文件行并将其作为字符串返回
- `read_csv(path, type)` - 解析逗号（或其他字符）分隔字段的文件。返回元组的数据集。 支持将基本的Java类型及其Value对应项作为字段类型。


基于集合:

- `from_elements(*args)` - 从Seq创建数据集。所有元素
- `generate_sequence(from, to)` -并行生成给定间隔内的数字序列。

**示例**

{% highlight python %}
env  = get_environment

\# read text file from local files system
localLiens = env.read_text("file:#/path/to/my/textfile")

\# read text file from a HDFS running at nnHost:nnPort
hdfsLines = env.read_text("hdfs://nnHost:nnPort/path/to/my/textfile")

\# read a CSV file with three fields, schema defined using constants defined in flink.plan.Constants
csvInput = env.read_csv("hdfs:///the/CSV/file", (INT, STRING, DOUBLE))

\# create a set from some given elements
values = env.from_elements("Foo", "bar", "foobar", "fubar")

\# generate a number sequence
numbers = env.generate_sequence(1, 10000000)
{% endhighlight %}

{% top %}

Data Sinks
----------

Data sinks 消费 DataSets 并用于存储或返回它们：

- `write_text()` - 将元素按行方式编写为字符串。字符串通过调用每个元素的*str()* 方法获得。
- `write_csv(...)` -将元组写为逗号分隔的值文件。行和字段分隔符是可配置的。每个字段的值来自对象的*str()* 方法。
- `output()` - 打印标准输出中每个元素的*str()* 值。


DataSet可以输入到多个算子。程序可以编写或打印一个数据集，同时对它们进行额外的转换。

**示例**

标准的 data sink 方法:

{% highlight scala %}
 write DataSet to a file on the local file system
textData.write_text("file:///my/result/on/localFS")

 write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.write_text("hdfs://nnHost:nnPort/my/result/on/localFS")

 write DataSet to a file and overwrite the file if it exists
textData.write_text("file:///my/result/on/localFS", WriteMode.OVERWRITE)

 tuples as lines with pipe as the separator "a|b|c"
values.write_csv("file:///path/to/the/result/file", line_delimiter="\n", field_delimiter="|")

 this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.write_text("file:///path/to/the/result/file")
{% endhighlight %}

{% top %}

Broadcast Variables
-------------------

除了算子的常规输入，广播变量还允许您为所有算子的并行实例创建一个数据集。
这对辅助数据集或数据相关参数化非常有用。数据集将作为集合在算子处访问。

- **Broadcast**:广播集合通过`with_broadcast_set(DataSet, String)`按名称注册
- **Access**: 在目标算子中通过`self.context.get_broadcast_variable(String)` 访问

{% highlight python %}
class MapperBcv(MapFunction):
    def map(self, value):
        factor = self.context.get_broadcast_variable("bcv")[0][0]
        return value * factor

# 1. The DataSet to be broadcast
toBroadcast = env.from_elements(1, 2, 3)
data = env.from_elements("a", "b")

# 2. Broadcast the DataSet
data.map(MapperBcv()).with_broadcast_set("bcv", toBroadcast)
{% endhighlight %}

在注册和访问广播数据集时，确保名称（前面例子中的'bcv'）匹配。

**Note**: 由于广播变量的内容保存在每个节点的内存中，因此它不应该变得太大。
对于简单的事情，如标量值，你可以简单地参数化rich 函数。

{% top %}

并行执行
------------------

本节介绍如何在Flink中配置程序的并行执行。
Flink程序由多个任务组成（算子，数据源和sink）。
一个任务被分成几个并行实例来执行，每个并行实例处理任务输入数据的一个子集。
一个任务的并行实例的数量被称为其*并行度*或*并行度（DOP）*。

任务的并行度可以在Flink中不同的级别指定。

### 执行环境级别

Flink程序在[执行环境](#program-skeleton)的上下文中执行。执行环境为所有算子，数据源和sink定义了一个默认的并行度。
通过显式配置算子的并行度，可以覆盖执行环境并行度。

执行环境的默认并行度可以通过调用`set_parallelism()`方法来指定。
要执行并行度为3的[WordCount](#example-program)示例程序的所有算子，数据源和数据sink，
请按如下方式设置执行环境的默认并行度：

{% highlight python %}
env = get_environment()
env.set_parallelism(3)

text.flat_map(lambda x,c: x.lower().split()) \
    .group_by(1) \
    .reduce_group(Adder(), combinable=True) \
    .output()

env.execute()
{% endhighlight %}

### 系统级别

可以通过在`./conf/flink-conf.yaml`中设置`parallelism.default` 属性来定义所有执行环境的全系统默认并行度。
详细信息请参阅[配置]({{ site.baseurl }}/ops/config.html)文档。

{% top %}

执行计划
---------------

要使用Flink运行计划，请转至您的Flink，然后从/bin文件夹运行pyflink.sh脚本。
包含该计划的脚本必须作为第一个参数传递，后面跟着一些附加的python包，最后将被提供给脚本的附加参数由 - 分隔开。

{% highlight python %}
./bin/pyflink.sh <Script>[ <pathToPackage1>[ <pathToPackageX]][ - <param1>[ <paramX>]]
{% endhighlight %}

{% top %}

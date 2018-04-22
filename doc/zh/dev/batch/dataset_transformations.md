---
title: "DataSet Transformations"
nav-title: Transformations
nav-parent_id: batch
nav-pos: 1
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

本文深入介绍了DataSet上可用的转换。 有关Flink Java API的一般介绍，请参阅[编程指南](index.html)。

要在索引密集的数据集中 zip 元素，请参阅[Zip Elements Guide](zip_elements_guide.html)。

* This will be replaced by the TOC
{:toc}

### Map

Map转换在DataSet的每个元素上应用用户定义的映射函数。它实现了一对一的映射，也就是说，函数必须返回一个元素。

以下代码将Integer对的DataSet转换为Integer的DataSet：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// MapFunction that adds two integer values
public class IntAdder implements MapFunction<Tuple2<Integer, Integer>, Integer> {
  @Override
  public Integer map(Tuple2<Integer, Integer> in) {
    return in.f0 + in.f1;
  }
}

// [...]
DataSet<Tuple2<Integer, Integer>> intPairs = // [...]
DataSet<Integer> intSums = intPairs.map(new IntAdder());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val intPairs: DataSet[(Int, Int)] = // [...]
val intSums = intPairs.map { pair => pair._1 + pair._2 }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 intSums = intPairs.map(lambda x: sum(x))
~~~

</div>
</div>

### FlatMap

FlatMap转换在DataSet的每个元素上应用自定义的 FlatMap 函数。 它（map函数的变体）可以为每个输入元素返回任意多个结果元素（包括none）。

以下代码将基于文本行的DataSet转换为基于单词的DataSet：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// FlatMapFunction that tokenizes a String by whitespace characters and emits all String tokens.
public class Tokenizer implements FlatMapFunction<String, String> {
  @Override
  public void flatMap(String value, Collector<String> out) {
    for (String token : value.split("\\W")) {
      out.collect(token);
    }
  }
}

// [...]
DataSet<String> textLines = // [...]
DataSet<String> words = textLines.flatMap(new Tokenizer());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val textLines: DataSet[String] = // [...]
val words = textLines.flatMap { _.split(" ") }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 words = lines.flat_map(lambda x,c: [line.split() for line in x])
~~~

</div>
</div>

### MapPartition

MapPartition在单个函数调用中转换一个并行分区。
map-partition函数以Iterable的形式获取分区，并可以生成任意数量的结果值。
每个分区中的元素数量取决于并行度和以前的算子。

以下代码将基于文本行的DataSet转换为每个分区单词数的DataSet：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class PartitionCounter implements MapPartitionFunction<String, Long> {

  public void mapPartition(Iterable<String> values, Collector<Long> out) {
    long c = 0;
    for (String s : values) {
      c++;
    }
    out.collect(c);
  }
}

// [...]
DataSet<String> textLines = // [...]
DataSet<Long> counts = textLines.mapPartition(new PartitionCounter());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val textLines: DataSet[String] = // [...]
// Some is required because the return value must be a Collection.
// There is an implicit conversion from Option to a Collection.
val counts = texLines.mapPartition { in => Some(in.size) }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 counts = lines.map_partition(lambda x,c: [sum(1 for _ in x)])
~~~

</div>
</div>

### Filter

Filter转换是在DataSet的每个元素上应用用户定义的过滤器函数，并仅保留函数返回`true`的元素。

以下代码从DataSet中删除所有小于零的整数：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// FilterFunction that filters out all Integers smaller than zero.
public class NaturalNumberFilter implements FilterFunction<Integer> {
  @Override
  public boolean filter(Integer number) {
    return number >= 0;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> naturalNumbers = intNumbers.filter(new NaturalNumberFilter());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val intNumbers: DataSet[Int] = // [...]
val naturalNumbers = intNumbers.filter { _ > 0 }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 naturalNumbers = intNumbers.filter(lambda x: x > 0)
~~~

</div>
</div>

**IMPORTANT:** 系统假定该函数不会修改应用了谓词的元素。违反这一假设可能会导致错误的结果。

### Projection of Tuple DataSet

Project转换会删除或移动元组数据集的元组字段。 project(int...)方法通过索引保留的Tuple字段，并在输出Tuple中定义它们的顺序。

Projections  不需要自定义函数。

以下代码显示了在DataSet上应用Project转换的不同方法：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<Integer, Double, String>> in = // [...]
// converts Tuple3<Integer, Double, String> into Tuple2<String, Integer>
DataSet<Tuple2<String, Integer>> out = in.project(2,0);
~~~

#### 带类型提示的Projection

请注意，Java编译器无法推断`project` operator 的返回类型。如果您对`project` operator 的结果调用另一个 operator，这会导致问题，例如：

~~~java
DataSet<Tuple5<String,String,String,String,String>> ds = ....
DataSet<Tuple1<String>> ds2 = ds.project(0).distinct(0);
~~~

这个问题可以通过提示`project`运算符的返回类型来解决：

~~~java
DataSet<Tuple1<String>> ds2 = ds.<Tuple1<String>>project(0).distinct(0);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
Not supported.
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
out = in.project(2,0);
~~~

</div>
</div>

### 分组数据集上的转换

reduce算子可以在分组数据集上运行。指定要用于分组的 key 可以通过多种方式指定：

- key 表达式
- key-selector 函数
- 一个或多个字段位置 (只适用于Tuple DataSet)
- Case Class 字段 (只适用于Case Classes )



请查看关于reduce的示例以了解如何指定分组key。

### 在分组数据集上的Reduce

应用于分组数据集上的Reduce转换使用用户定义的reduce函数将每个组缩减为单个元素。
对于每组输入元素，reduce函数将成对的元素连续组合为一个元素，直到每个组只剩下一个元素。

请注意，对于`ReduceFunction` ，返回对象的key字段应与输入值的key匹配。这是因为reduce可以隐式组合，
并且从合并operator发出的对象在传递给reduce算子时会再次按键分组。

#### 按 key 表达式分组的数据集 Reduce

Key 表达式可以指定DataSet的每个元素的一个或多个字段。每个key表达式都是public字段的名称或getter方法。
点可以用来深入到对象中。 key表达式“*” 选择所有字段。以下代码显示了如何使用关key表达式对POJO DataSet进行分组，
并使用reduce函数对其进行reduce计算。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some ordinary POJO
public class WC {
  public String word;
  public int count;
  // [...]
}

// ReduceFunction that sums Integer attributes of a POJO
public class WordCounter implements ReduceFunction<WC> {
  @Override
  public WC reduce(WC in1, WC in2) {
    return new WC(in1.word, in1.count + in2.count);
  }
}

// [...]
DataSet<WC> words = // [...]
DataSet<WC> wordCounts = words
                         // DataSet grouping on field "word"
                         .groupBy("word")
                         // apply ReduceFunction on grouped DataSet
                         .reduce(new WordCounter());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// some ordinary POJO
class WC(val word: String, val count: Int) {
  def this() {
    this(null, -1)
  }
  // [...]
}

val words: DataSet[WC] = // [...]
val wordCounts = words.groupBy("word").reduce {
  (w1, w2) => new WC(w1.word, w1.count + w2.count)
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~
</div>
</div>

#### 按KeySelector函数分组的DataSet上的Reduce

key-selector函数从DataSet的每个元素中提取键值。提取的键值用于对DataSet进行分组。
以下代码显示了如何使用key-selector函数对POJO DataSet进行分组，并使用reduce函数对其进行reduce计算。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some ordinary POJO
public class WC {
  public String word;
  public int count;
  // [...]
}

// ReduceFunction that sums Integer attributes of a POJO
public class WordCounter implements ReduceFunction<WC> {
  @Override
  public WC reduce(WC in1, WC in2) {
    return new WC(in1.word, in1.count + in2.count);
  }
}

// [...]
DataSet<WC> words = // [...]
DataSet<WC> wordCounts = words
                         // DataSet grouping on field "word"
                         .groupBy(new SelectWord())
                         // apply ReduceFunction on grouped DataSet
                         .reduce(new WordCounter());

public class SelectWord implements KeySelector<WC, String> {
  @Override
  public String getKey(Word w) {
    return w.word;
  }
}
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// some ordinary POJO
class WC(val word: String, val count: Int) {
  def this() {
    this(null, -1)
  }
  // [...]
}

val words: DataSet[WC] = // [...]
val wordCounts = words.groupBy { _.word } reduce {
  (w1, w2) => new WC(w1.word, w1.count + w2.count)
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
class WordCounter(ReduceFunction):
    def reduce(self, in1, in2):
        return (in1[0], in1[1] + in2[1])

words = // [...]
wordCounts = words \
    .group_by(lambda x: x[0]) \
    .reduce(WordCounter())
~~~
</div>
</div>

#### 按字段位置 key 分组的数据集上的reduce（仅适用于元组数据集）

字段位置key 指定元组数据集的一个或多个字段用作分组 key 。
以下代码显示如何使用字段位置 key 并应用reduce函数

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<String, Integer, Double>> tuples = // [...]
DataSet<Tuple3<String, Integer, Double>> reducedTuples = tuples
                                         // group DataSet on first and second field of Tuple
                                         .groupBy(0, 1)
                                         // apply ReduceFunction on grouped DataSet
                                         .reduce(new MyTupleReducer());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val tuples = DataSet[(String, Int, Double)] = // [...]
// group on the first and second Tuple field
val reducedTuples = tuples.groupBy(0, 1).reduce { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 reducedTuples = tuples.group_by(0, 1).reduce( ... )
~~~

</div>
</div>

#### 按Case Class 字段分组的数据集reduce

使用Case Classes 时，您可以使用字段的名称指定分组键：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
Not supported.
~~~
</div>
<div data-lang="scala" markdown="1">

~~~scala
case class MyClass(val a: String, b: Int, c: Double)
val tuples = DataSet[MyClass] = // [...]
// group on the first and second field
val reducedTuples = tuples.groupBy("a", "b").reduce { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~
</div>
</div>

### 分组数据集上的GroupReduce

应用于分组数据集的 GroupReduce 为每个组调用自定义的 group-reduce 函数。
这与Reduce的区别在于用户定义函数一次获取整个组。该函数在组的所有元素上使用Iterable进行调用，
并且可以返回任意数量的结果元素。

#### 按字段位置 key 分组的DataSet上的GroupReduce（仅适用于元组数据集）

以下代码显示了如何从按Integer分组的数据集中删除重复的字符串。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class DistinctReduce
         implements GroupReduceFunction<Tuple2<Integer, String>, Tuple2<Integer, String>> {

  @Override
  public void reduce(Iterable<Tuple2<Integer, String>> in, Collector<Tuple2<Integer, String>> out) {

    Set<String> uniqStrings = new HashSet<String>();
    Integer key = null;

    // add all strings of the group to the set
    for (Tuple2<Integer, String> t : in) {
      key = t.f0;
      uniqStrings.add(t.f1);
    }

    // emit all unique strings.
    for (String s : uniqStrings) {
      out.collect(new Tuple2<Integer, String>(key, s));
    }
  }
}

// [...]
DataSet<Tuple2<Integer, String>> input = // [...]
DataSet<Tuple2<Integer, String>> output = input
                           .groupBy(0)            // group DataSet by the first tuple field
                           .reduceGroup(new DistinctReduce());  // apply GroupReduceFunction
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String)] = // [...]
val output = input.groupBy(0).reduceGroup {
      (in, out: Collector[(Int, String)]) =>
        in.toSet foreach (out.collect)
    }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class DistinctReduce(GroupReduceFunction):
   def reduce(self, iterator, collector):
     dic = dict()
     for value in iterator:
       dic[value[1]] = 1
     for key in dic.keys():
       collector.collect(key)

 output = data.group_by(0).reduce_group(DistinctReduce())
~~~

</div>
</div>

#### 根据key表达式，KeySelector函数或Case Class字段分组的DataSet上的GroupReduce

与通过[key表达式](#reduce-on-dataset-grouped-by-key-expression),
[KeySelector函数](#reduce-on-dataset-grouped-by-keyselector-function),
and [Case Class字段](#reduce-on-dataset-grouped-by-case-class-fields) 进行*Reduce* 转换计算类似。


#### 对已排序的组进行GroupReduce

group-reduce函数使用Iterable访问组的元素。可选地，Iterable可以按照指定的顺序发布组的元素。
在许多情况下，这可以帮助降低用户定义的group-reduce 函数的复杂性并提高效率。

以下代码显示了另一个示例，该示例说明如何删除由Integer分组并按字符串排序的DataSet中的重复字符串。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// GroupReduceFunction that removes consecutive identical elements
public class DistinctReduce
         implements GroupReduceFunction<Tuple2<Integer, String>, Tuple2<Integer, String>> {

  @Override
  public void reduce(Iterable<Tuple2<Integer, String>> in, Collector<Tuple2<Integer, String>> out) {
    Integer key = null;
    String comp = null;

    for (Tuple2<Integer, String> t : in) {
      key = t.f0;
      String next = t.f1;

      // check if strings are different
      if (com == null || !next.equals(comp)) {
        out.collect(new Tuple2<Integer, String>(key, next));
        comp = next;
      }
    }
  }
}

// [...]
DataSet<Tuple2<Integer, String>> input = // [...]
DataSet<Double> output = input
                         .groupBy(0)                         // group DataSet by first field
                         .sortGroup(1, Order.ASCENDING)      // sort groups on second tuple field
                         .reduceGroup(new DistinctReduce());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String)] = // [...]
val output = input.groupBy(0).sortGroup(1, Order.ASCENDING).reduceGroup {
      (in, out: Collector[(Int, String)]) =>
        var prev: (Int, String) = null
        for (t <- in) {
          if (prev == null || prev != t)
            out.collect(t)
            prev = t
        }
    }

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class DistinctReduce(GroupReduceFunction):
   def reduce(self, iterator, collector):
     dic = dict()
     for value in iterator:
       dic[value[1]] = 1
     for key in dic.keys():
       collector.collect(key)

 output = data.group_by(0).sort_group(1, Order.ASCENDING).reduce_group(DistinctReduce())
~~~


</div>
</div>

**Note:** 如果在reduce操作之前使用基于排序的执行策略建立的分组，则GroupSort一般是没开销的。

#### 组合 GroupReduceFunctions

与reduce函数相比，group-reduce函数不是隐式组合的。
为了使group-reduce 函数可组合，它必须实现GroupCombineFunction接口。

**Important**: `GroupCombineFunction`接口的通用输入和输出类型必须等于`GroupReduceFunction`的通用输入类型，如以下示例所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// Combinable GroupReduceFunction that computes a sum.
public class MyCombinableGroupReducer implements
  GroupReduceFunction<Tuple2<String, Integer>, String>,
  GroupCombineFunction<Tuple2<String, Integer>, Tuple2<String, Integer>>
{
  @Override
  public void reduce(Iterable<Tuple2<String, Integer>> in,
                     Collector<String> out) {

    String key = null;
    int sum = 0;

    for (Tuple2<String, Integer> curr : in) {
      key = curr.f0;
      sum += curr.f1;
    }
    // concat key and sum and emit
    out.collect(key + "-" + sum);
  }

  @Override
  public void combine(Iterable<Tuple2<String, Integer>> in,
                      Collector<Tuple2<String, Integer>> out) {
    String key = null;
    int sum = 0;

    for (Tuple2<String, Integer> curr : in) {
      key = curr.f0;
      sum += curr.f1;
    }
    // emit tuple with key and sum
    out.collect(new Tuple2<>(key, sum));
  }
}
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala

// Combinable GroupReduceFunction that computes two sums.
class MyCombinableGroupReducer
  extends GroupReduceFunction[(String, Int), String]
  with GroupCombineFunction[(String, Int), (String, Int)]
{
  override def reduce(
    in: java.lang.Iterable[(String, Int)],
    out: Collector[String]): Unit =
  {
    val r: (String, Int) =
      in.asScala.reduce( (a,b) => (a._1, a._2 + b._2) )
    // concat key and sum and emit
    out.collect (r._1 + "-" + r._2)
  }

  override def combine(
    in: java.lang.Iterable[(String, Int)],
    out: Collector[(String, Int)]): Unit =
  {
    val r: (String, Int) =
      in.asScala.reduce( (a,b) => (a._1, a._2 + b._2) )
    // emit tuple with key and sum
    out.collect(r)
  }
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class GroupReduce(GroupReduceFunction):
   def reduce(self, iterator, collector):
     key, int_sum = iterator.next()
     for value in iterator:
       int_sum += value[1]
     collector.collect(key + "-" + int_sum))

   def combine(self, iterator, collector):
     key, int_sum = iterator.next()
     for value in iterator:
       int_sum += value[1]
     collector.collect((key, int_sum))

data.reduce_group(GroupReduce(), combinable=True)
~~~

</div>
</div>

### 分组数据集上的GroupCombine

GroupCombine转换是可组合的GroupReduceFunction中组合步骤的一般形式。它的意义在于，
它允许将输入类型`I` 与任意输出类型`O`相结合。相反，GroupReduce中的组合步骤仅允许从输入类型`I` 到输出类型`I` 的组合。
这是因为GroupReduceFunction中的reduce步骤需要输入类型“I”。

在某些应用程序中，希望在执行其他转换（例如reduce数据大小）之前将DataSet组合成中间格式。
这可以通过CombineGroup转换以很低的成本实现。

**Note:** 分组数据集上的GroupCombine在内存中以贪婪策略执行，该策略可能不会一次处理所有数据，而是以多个步骤处理。
          它也可以在单个分区上执行，而无需像GroupReduce转换那样进行数据交换。这可能会导致部分结果。

以下示例演示如何将CombineGroup转换用于实现WordCount。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<String> input = [..] // The words received as input

DataSet<Tuple2<String, Integer>> combinedWords = input
  .groupBy(0) // group identical words
  .combineGroup(new GroupCombineFunction<String, Tuple2<String, Integer>() {

    public void combine(Iterable<String> words, Collector<Tuple2<String, Integer>>) { // combine
        String key = null;
        int count = 0;

        for (String word : words) {
            key = word;
            count++;
        }
        // emit tuple with word and count
        out.collect(new Tuple2(key, count));
    }
});

DataSet<Tuple2<String, Integer>> output = combinedWords
  .groupBy(0)                              // group by words again
  .reduceGroup(new GroupReduceFunction() { // group reduce with full data exchange

    public void reduce(Iterable<Tuple2<String, Integer>>, Collector<Tuple2<String, Integer>>) {
        String key = null;
        int count = 0;

        for (Tuple2<String, Integer> word : words) {
            key = word;
            count++;
        }
        // emit tuple with word and count
        out.collect(new Tuple2(key, count));
    }
});
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[String] = [..] // The words received as input

val combinedWords: DataSet[(String, Int)] = input
  .groupBy(0)
  .combineGroup {
    (words, out: Collector[(String, Int)]) =>
        var key: String = null
        var count = 0

        for (word <- words) {
            key = word
            count += 1
        }
        out.collect((key, count))
}

val output: DataSet[(String, Int)] = combinedWords
  .groupBy(0)
  .reduceGroup {
    (words, out: Collector[(String, Int)]) =>
        var key: String = null
        var sum = 0

        for ((word, sum) <- words) {
            key = word
            sum += count
        }
        out.collect((key, sum))
}

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

以上替代WordCount实现演示GroupCombine在执行GroupReduce转换之前如何组合单词。
上面的例子只是一个概念证明。请注意，组合步骤如何改变DataSet的类型，
在执行GroupReduce之前通常需要额外的Map转换。

### 在分组元组数据集上的聚合操作

有一些常用的 Aggregate  操作。 Aggregate 转换提供以下内置聚合函数：

- Sum,
- Min, and
- Max.

Aggregate 转换只能应用于元组数据集，并且仅支持用字段位置 key来分组。

以下代码显示如何在按字段位置 key 分组的数据集上应用 Aggregation 转换：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input
                                   .groupBy(1)        // group DataSet on second field
                                   .aggregate(SUM, 0) // compute sum of the first field
                                   .and(MIN, 2);      // compute minimum of the third field
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String, Double)] = // [...]
val output = input.groupBy(1).aggregate(SUM, 0).and(MIN, 2)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
from flink.functions.Aggregation import Sum, Min

input = # [...]
output = input.group_by(1).aggregate(Sum, 0).and_agg(Min, 2)
~~~

</div>
</div>

要在DataSet上应用多个聚合，那么在第一个聚合之后要使用`.and()`函数，这意味着`.aggregate(SUM, 0).and(MIN, 2)` 对原始数据集的字段0进行求和对字段2求最小值。
相反， `.aggregate(SUM, 0).aggregate(MIN, 2)`将在聚合上在聚合。在给出的示例中，在计算按字段1分组对字段0求和之后，在产生字段2的最小值。

**Note:** 聚合函数集将在未来扩展。

### MinBy / MaxBy on Grouped Tuple DataSet

The MinBy (MaxBy) transformation selects a single tuple for each group of tuples. The selected tuple is the tuple whose values of one or more specified fields are minimum (maximum). The fields which are used for comparison must be valid key fields, i.e., comparable. If multiple tuples have minimum (maximum) fields values, an arbitrary tuple of these tuples is returned.

The following code shows how to select the tuple with the minimum values for the `Integer` and `Double` fields for each group of tuples with the same `String` value from a `DataSet<Tuple3<Integer, String, Double>>`:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input
                                   .groupBy(1)   // group DataSet on second field
                                   .minBy(0, 2); // select tuple with minimum values for first and third field.
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Double)] = input
                                   .groupBy(1)  // group DataSet on second field
                                   .minBy(0, 2) // select tuple with minimum values for first and third field.
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

### 在全数据集上Reduce计算

Reduce转换将用户定义的reduce函数应用于DataSet的所有元素。reduce函数随后将成对的元素合并成一个元素，直到只剩下一个元素。

以下代码显示了如何对整数数据集的所有元素求和：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// ReduceFunction that sums Integers
public class IntSummer implements ReduceFunction<Integer> {
  @Override
  public Integer reduce(Integer num1, Integer num2) {
    return num1 + num2;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> sum = intNumbers.reduce(new IntSummer());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val intNumbers = env.fromElements(1,2,3)
val sum = intNumbers.reduce (_ + _)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 intNumbers = env.from_elements(1,2,3)
 sum = intNumbers.reduce(lambda x,y: x + y)
~~~

</div>
</div>

使用Reduce转换来对整个DataSet进行reduce计算意味着最终的Reduce操作不能并行执行。但是，reduce函数是自动组合的，因此Reduce转换不会限制大多数用例的可伸缩性。

### 在全数据集上进行GroupReduce

GroupReduce转换在DataSet的所有元素上应用自定义的group-reduce 函数。
group-reduce可以迭代DataSet的所有元素并返回任意数量的结果元素。

以下示例显示如何在全数据集上应用GroupReduce转换：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Integer> input = // [...]
// apply a (preferably combinable) GroupReduceFunction to a DataSet
DataSet<Double> output = input.reduceGroup(new MyGroupReducer());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[Int] = // [...]
val output = input.reduceGroup(new MyGroupReducer())
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 output = data.reduce_group(MyGroupReducer())
~~~

</div>
</div>

**Note:** 如果group-reduce函数不可组合，则整个DataSet上的GroupReduce转换不能并行完成。
          因此，这可能是一个非常密集的计算操作。请参阅上文 “Combinable GroupReduceFunctions” 的段落，
          以了解如何实现可组合的group-reduce函数。

### GroupCombine on a full DataSet

全数据集上的GroupCombine与分组数据集上的GroupCombine类似。数据在所有节点上进行分区，
然后以贪婪的方式进行组合（即只有拟合到内存中的数据被一次合并）。

### 在全数据集上的Aggregate

有一些常用的聚合操作。聚合转换提供以下内置聚合函数：

- Sum,
- Min, and
- Max.

聚合转换只能应用于元组数据集。

以下代码显示如何在完整的DataSet上应用聚合转换：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, Double>> input = // [...]
DataSet<Tuple2<Integer, Double>> output = input
                                     .aggregate(SUM, 0)    // compute sum of the first field
                                     .and(MIN, 1);    // compute minimum of the second field
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String, Double)] = // [...]
val output = input.aggregate(SUM, 0).and(MIN, 2)

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
from flink.functions.Aggregation import Sum, Min

input = # [...]
output = input.aggregate(Sum, 0).and_agg(Min, 2)
~~~

</div>
</div>

**Note:** 扩展支持的聚合函数集在我们的计划中。

### 在全元组数据集上的MinBy / MaxBy

 MinBy (MaxBy) 变换从由元组组成的DataSet中选择一个元组。所选元组的一个或多个指定字段的值是在所有元组中最小（最大）。用于比较的字段必须是有效的字段，即可比较的字段。如果多个元组具有最小（最大）字段，则返回这些元组的任意元组。

以下代码显示了如何从数据集`DataSet<Tuple3<Integer, String, Double>>`中选择具有 `Integer` 和 `Double`字段最大值的元组：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input
                                   .maxBy(0, 2); // select tuple with maximum values for first and third field.
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Double)] = input                          
                                   .maxBy(0, 2) // select tuple with maximum values for first and third field.
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

### Distinct

TDistinct转换对DataSet进行去重计算。
以下代码从DataSet中删除所有重复的元素：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, Double>> input = // [...]
DataSet<Tuple2<Integer, Double>> output = input.distinct();

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String, Double)] = // [...]
val output = input.distinct()

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

还可以使用以下方法对DataSet中元素的进行去重：

- 一个或多个字段位置上的key（仅适用于元组数据集），
- 一个key-selector 函数，或者
- 一个 key 表达式。

#### 使用field position keys 进行Distinct

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, Double, String>> input = // [...]
DataSet<Tuple2<Integer, Double, String>> output = input.distinct(0,2);

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, Double, String)] = // [...]
val output = input.distinct(0,2)

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

#### 使用KeySelector 函数 进行Distinct

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
private static class AbsSelector implements KeySelector<Integer, Integer> {
private static final long serialVersionUID = 1L;
	@Override
	public Integer getKey(Integer t) {
    	return Math.abs(t);
	}
}
DataSet<Integer> input = // [...]
DataSet<Integer> output = input.distinct(new AbsSelector());

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[Int] = // [...]
val output = input.distinct {x => Math.abs(x)}

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

#### 使用key expression 进行Distinct

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some ordinary POJO
public class CustomType {
  public String aName;
  public int aNumber;
  // [...]
}

DataSet<CustomType> input = // [...]
DataSet<CustomType> output = input.distinct("aName", "aNumber");

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// some ordinary POJO
case class CustomType(aName : String, aNumber : Int) { }

val input: DataSet[CustomType] = // [...]
val output = input.distinct("aName", "aNumber")

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

也可以通过通配符使用所有字段：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<CustomType> input = // [...]
DataSet<CustomType> output = input.distinct("*");

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// some ordinary POJO
val input: DataSet[CustomType] = // [...]
val output = input.distinct("_")

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

### Join

Join 转换将两个数据集join成一个数据集。两个DataSet的元素可以使用一个或多个可以指定的key进行连接

- 一个 key expression
- 一个 key-selector function
- 一个或多个 field position keys (只适用于Tuple DataSet).
- Case Class 字段

有几种不同的方法可以执行Join转换，如下所示。

#### 默认的 Join (Join into Tuple2)

默认的Join转换产生一个带有两个字段的新的Tuple DataSet。每个元组保存第一个输入DataSet的联合元素和第二个输入DataSet的匹配元素。

以下代码显示了使用 field position keys 来进行默认的Join转换：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public static class User { public String name; public int zip; }
public static class Store { public Manager mgr; public int zip; }
DataSet<User> input1 = // [...]
DataSet<Store> input2 = // [...]
// result dataset is typed as Tuple2
DataSet<Tuple2<User, Store>>
            result = input1.join(input2)
                           .where("zip")       // key of the first input (users)
                           .equalTo("zip");    // key of the second input (stores)
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[(Int, String)] = // [...]
val input2: DataSet[(Double, Int)] = // [...]
val result = input1.join(input2).where(0).equalTo(1)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 result = input1.join(input2).where(0).equal_to(1)
~~~

</div>
</div>

####  使用 Join Function 进行 Join 操作

Join转换还可以调用用户定义的连接函数来处理 join的元组。
一个连接函数接收第一个输入DataSet的一个元素和第二个输入DataSet的一个元素，并返回一个元素。

以下代码使用key-selector 函数执行DataSet与自定义java对象和Tuple DataSet的连接，并演示如何使用用户定义的连接函数：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some POJO
public class Rating {
  public String name;
  public String category;
  public int points;
}

// Join function that joins a custom POJO with a Tuple
public class PointWeighter
         implements JoinFunction<Rating, Tuple2<String, Double>, Tuple2<String, Double>> {

  @Override
  public Tuple2<String, Double> join(Rating rating, Tuple2<String, Double> weight) {
    // multiply the points and rating and construct a new output tuple
    return new Tuple2<String, Double>(rating.name, rating.points * weight.f1);
  }
}

DataSet<Rating> ratings = // [...]
DataSet<Tuple2<String, Double>> weights = // [...]
DataSet<Tuple2<String, Double>>
            weightedRatings =
            ratings.join(weights)

                   // key of the first input
                   .where("category")

                   // key of the second input
                   .equalTo("f0")

                   // applying the JoinFunction on joining pairs
                   .with(new PointWeighter());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
case class Rating(name: String, category: String, points: Int)

val ratings: DataSet[Ratings] = // [...]
val weights: DataSet[(String, Double)] = // [...]

val weightedRatings = ratings.join(weights).where("category").equalTo(0) {
  (rating, weight) => (rating.name, rating.points * weight._2)
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class PointWeighter(JoinFunction):
   def join(self, rating, weight):
     return (rating[0], rating[1] * weight[1])
       if value1[3]:

 weightedRatings =
   ratings.join(weights).where(0).equal_to(0). \
   with(new PointWeighter());
~~~

</div>
</div>

####  使用 Flat-Join Function进行Join

类似于Map和FlatMap，FlatJoin的行为方式与Join相同，但不是返回一个元素，而是返回（集合），零个，一个或多个元素。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class PointWeighter
         implements FlatJoinFunction<Rating, Tuple2<String, Double>, Tuple2<String, Double>> {
  @Override
  public void join(Rating rating, Tuple2<String, Double> weight,
	  Collector<Tuple2<String, Double>> out) {
	if (weight.f1 > 0.1) {
		out.collect(new Tuple2<String, Double>(rating.name, rating.points * weight.f1));
	}
  }
}

DataSet<Tuple2<String, Double>>
            weightedRatings =
            ratings.join(weights) // [...]
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
case class Rating(name: String, category: String, points: Int)

val ratings: DataSet[Ratings] = // [...]
val weights: DataSet[(String, Double)] = // [...]

val weightedRatings = ratings.join(weights).where("category").equalTo(0) {
  (rating, weight, out: Collector[(String, Double)]) =>
    if (weight._2 > 0.1) out.collect(rating.name, rating.points * weight._2)
}

~~~

</div>
<div data-lang="python" markdown="1">
Not supported.
</div>
</div>

####  使用 Projection (只适用于Java/Python )进行Join

Join转换可以使用 projection 构造结果元组，如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<Integer, Byte, String>> input1 = // [...]
DataSet<Tuple2<Integer, Double>> input2 = // [...]
DataSet<Tuple4<Integer, String, Double, Byte>
            result =
            input1.join(input2)
                  // key definition on first DataSet using a field position key
                  .where(0)
                  // key definition of second DataSet using a field position key
                  .equalTo(0)
                  // select and reorder fields of matching tuples
                  .projectFirst(0,2).projectSecond(1).projectFirst(1);
~~~

`projectFirst(int...)`和`projectSecond(int...)`选择应该组合成输出元组的第一个和第二个连接输入的字段。
索引的顺序定义了输出元组中字段的顺序。 join projection也适用于非元组数据集。
在这种情况下，必须调用不带参数的`projectFirst()` 或`projectSecond()`才能将联合元素添加到输出元组中。

</div>
<div data-lang="scala" markdown="1">

~~~scala
Not supported.
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 result = input1.join(input2).where(0).equal_to(0) \
  .project_first(0,2).project_second(1).project_first(1);
~~~

`projectFirst(int...)`和`projectSecond(int...)`选择应该组合成输出元组的第一个和第二个连接输入的字段。
索引的顺序定义了输出元组中字段的顺序。 join projection也适用于非元组数据集。
在这种情况下，必须调用不带参数的`projectFirst()` 或`projectSecond()`才能将联合元素添加到输出元组中。

</div>
</div>

#### 使用DataSet Size Hint进行Join

为了引导优化器选择正确的执行策略，您可以提示要加入的DataSet的大小，如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>>
            result1 =
            // hint that the second DataSet is very small
            input1.joinWithTiny(input2)
                  .where(0)
                  .equalTo(0);

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>>
            result2 =
            // hint that the second DataSet is very large
            input1.joinWithHuge(input2)
                  .where(0)
                  .equalTo(0);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[(Int, String)] = // [...]
val input2: DataSet[(Int, String)] = // [...]

// hint that the second DataSet is very small
val result1 = input1.joinWithTiny(input2).where(0).equalTo(0)

// hint that the second DataSet is very large
val result1 = input1.joinWithHuge(input2).where(0).equalTo(0)

~~~

</div>
<div data-lang="python" markdown="1">

~~~python

 #hint that the second DataSet is very small
 result1 = input1.join_with_tiny(input2).where(0).equal_to(0)

 #hint that the second DataSet is very large
 result1 = input1.join_with_huge(input2).where(0).equal_to(0)

~~~

</div>
</div>

#### 使用Algorithm Hints进行Join

Flink运行时可以以各种方式执行 join 。在不同的情况下，每种可能的方式都胜过其他方式。
系统会尝试自动选择合理的方式，但如果您想强制执行join的特定方式，则允许您手动选择策略。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<SomeType> input1 = // [...]
DataSet<AnotherType> input2 = // [...]

DataSet<Tuple2<SomeType, AnotherType> result =
      input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
            .where("id").equalTo("key");
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[SomeType] = // [...]
val input2: DataSet[AnotherType] = // [...]

// hint that the second DataSet is very small
val result1 = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST).where("id").equalTo("key")

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

以下hints可用：

* `OPTIMIZER_CHOOSES`: 相当于根本不给提示，将选择权留给系统。

* `BROADCAST_HASH_FIRST`: 广播第一个输入并从中建立一个哈希表，由第二个输入进行探测。当第一个输入是非常小的时候，这是一个好的策略。

* `BROADCAST_HASH_SECOND`: 广播第二个输入并从中建立一个哈希表，由第第个输入进行探测。当第二个输入是非常小的时候，这是一个好的策略。

* `REPARTITION_HASH_FIRST`: 系统对每个输入进行分区（shuffles）（除非输入已经分区），并从第一个输入构建一个散列表。
                            如果第一个输入小于第二个输入，并且这两个输入仍然很大，则此策略很好。
                            注意：这是系统使用的默认回退策略，如果不能进行大小估计并且不能重新使用预先存在的分区和排序。

* `REPARTITION_HASH_SECOND`:系统对每个输入进行分区（shuffles）（除非输入已经被分区）并且从第二个输入建立一个散列表。
                            如果第二个输入小于第一个输入，并且这两个输入仍然很大，则此策略很好。

* `REPARTITION_SORT_MERGE`: 系统对每个输入进行分区（shuffles）（除非输入已被分区）并对每个输入进行排序（除非它已被排序）。
                            输入通过一个已排序输入的streamed merge来连接。 如果其中一个或两个输入已经排序，则该策略是好的。


### OuterJoin

OuterJoin转换在两个数据集上执行左，右或全外连接。外部连接与常规（内部）连接类似，并创建所有在其键上相同的元素对。另外，如果在另一侧没有找到匹配的键，则保留“outer”侧（左侧，右侧或两者都满）的记录。匹配一对元素（或一个元素和另一个输入的null 值）被赋予JoinFunction以将该对元素转换为单个元素或FlatJoinFunction以将该对元素变为任意多个（包括你none）元素。

两个DataSet的元素都可以使用一个或多个可以指定的key进行连接

- 一个 key expression
- 一个 key-selector 函数
- 一个或多个 field position keys (只适用于Tuple DataSet).
- Case Class 字段

**OuterJoins 只支持 Java 和 Scala DataSet API.**


####  通过 Join 函数 进行OuterJoin


OuterJoin转换调用用户定义的连接函数来处理连接的元组。一个连接函数接收第一个输入DataSet的一个元素和第二个输入DataSet的一个元素，并返回一个元素。根据外连接的类型(left, right, full) ，连接函数的两个输入元素之一可以为null。


以下代码使用  key-selector 函数执行自定义java对象DataSet和Tuple DataSet的左外部联接 ，并演示如何使用用户定义的 join 函数：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some POJO
public class Rating {
  public String name;
  public String category;
  public int points;
}

// Join function that joins a custom POJO with a Tuple
public class PointAssigner
         implements JoinFunction<Tuple2<String, String>, Rating, Tuple2<String, Integer>> {

  @Override
  public Tuple2<String, Integer> join(Tuple2<String, String> movie, Rating rating) {
    // Assigns the rating points to the movie.
    // NOTE: rating might be null
    return new Tuple2<String, Double>(movie.f0, rating == null ? -1 : rating.points;
  }
}

DataSet<Tuple2<String, String>> movies = // [...]
DataSet<Rating> ratings = // [...]
DataSet<Tuple2<String, Integer>>
            moviesWithPoints =
            movies.leftOuterJoin(ratings)

                   // key of the first input
                   .where("f0")

                   // key of the second input
                   .equalTo("name")

                   // applying the JoinFunction on joining pairs
                   .with(new PointAssigner());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
case class Rating(name: String, category: String, points: Int)

val movies: DataSet[(String, String)] = // [...]
val ratings: DataSet[Ratings] = // [...]

val moviesWithPoints = movies.leftOuterJoin(ratings).where(0).equalTo("name") {
  (movie, rating) => (movie._1, if (rating == null) -1 else rating.points)
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

#### 通过 Flat-Join 函数 进行OuterJoin

类似于Map和FlatMap，具有 flat-join函数的OuterJoin的行为与具有 join函数的OuterJoin的行为相同，但是不是返回一个元素，而是返回（集合）零个，一个或多个元素。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class PointAssigner
         implements FlatJoinFunction<Tuple2<String, String>, Rating, Tuple2<String, Integer>> {
  @Override
  public void join(Tuple2<String, String> movie, Rating rating
    Collector<Tuple2<String, Integer>> out) {
  if (rating == null ) {
    out.collect(new Tuple2<String, Integer>(movie.f0, -1));
  } else if (rating.points < 10) {
    out.collect(new Tuple2<String, Integer>(movie.f0, rating.points));
  } else {
    // do not emit
  }
}

DataSet<Tuple2<String, Integer>>
            moviesWithPoints =
            movies.leftOuterJoin(ratings) // [...]
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
Not supported.
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

#### Join Algorithm Hints

Flink运行时可以以各种方式执行外部连接。在不同的情况下，每种可能的方式都胜过其他方式。系统会自动尝试选择合理的方式，但如果您想强制执行外部联接的特定方式，则允许您手动选择策略。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<SomeType> input1 = // [...]
DataSet<AnotherType> input2 = // [...]

DataSet<Tuple2<SomeType, AnotherType> result1 =
      input1.leftOuterJoin(input2, JoinHint.REPARTITION_SORT_MERGE)
            .where("id").equalTo("key");

DataSet<Tuple2<SomeType, AnotherType> result2 =
      input1.rightOuterJoin(input2, JoinHint.BROADCAST_HASH_FIRST)
            .where("id").equalTo("key");
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[SomeType] = // [...]
val input2: DataSet[AnotherType] = // [...]

// hint that the second DataSet is very small
val result1 = input1.leftOuterJoin(input2, JoinHint.REPARTITION_SORT_MERGE).where("id").equalTo("key")

val result2 = input1.rightOuterJoin(input2, JoinHint.BROADCAST_HASH_FIRST).where("id").equalTo("key")

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

The following hints are available.

* `OPTIMIZER_CHOOSES`: 相当于根本不给提示，将选择权留给系统。

* `BROADCAST_HASH_FIRST`:广播第一个输入并从中建立一个哈希表，由第二个输入进行探测。当第一个输入是非常小的时候，这是一个好的策略。

* `BROADCAST_HASH_SECOND`: 广播第二个输入并从中建立一个哈希表，由第第个输入进行探测。当第二个输入是非常小的时候，这是一个好的策略。

* `REPARTITION_HASH_FIRST`: 系统对每个输入进行分区（shuffles）（除非输入已经分区），并从第一个输入构建一个散列表。
                            如果第一个输入小于第二个输入，并且这两个输入仍然很大，则此策略很好。

* `REPARTITION_HASH_SECOND`: 系统对每个输入进行分区（shuffles）（除非输入已经被分区）并且从第二个输入建立一个散列表。
                             如果第二个输入小于第一个输入，并且这两个输入仍然很大，则此策略很好。

* `REPARTITION_SORT_MERGE`: 系统对每个输入进行分区（shuffles）（除非输入已被分区）并对每个输入进行排序（除非它已被排序）。
                            输入通过一个已排序输入的streamed merge来连接。 如果其中一个或两个输入已经排序，则该策略是好的。

**NOTE:** 不是所有的执行策略都受到每个OuterJoin类型的支持。

* `LeftOuterJoin` supports:
  * `OPTIMIZER_CHOOSES`
  * `BROADCAST_HASH_SECOND`
  * `REPARTITION_HASH_SECOND`
  * `REPARTITION_SORT_MERGE`

* `RightOuterJoin` supports:
  * `OPTIMIZER_CHOOSES`
  * `BROADCAST_HASH_FIRST`
  * `REPARTITION_HASH_FIRST`
  * `REPARTITION_SORT_MERGE`

* `FullOuterJoin` supports:
  * `OPTIMIZER_CHOOSES`
  * `REPARTITION_SORT_MERGE`


### Cross

Cross 转换将两个数据集组合成一个数据集。它构建了两个输入数据集的所有成对组合，即它构建了一个笛卡尔积。Cross 转换要么在每对元素上调用用户定义的交叉函数，要么输出一个Tuple2。
两种模式如下所示。

**Note:** Cross可能是一个*非常*计算密集型的操作，它甚至可以挑战大型计算集群！

####  带自定义函数的Cross

Cross转换可以调用用户定义的Cross函数。 Cross函数接收第一个输入的一个元素和第二个输入的一个元素，并返回一个结果元素。

以下代码显示了如何使用Cross函数对两个数据集应用Cross转换：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class Coord {
  public int id;
  public int x;
  public int y;
}

// CrossFunction computes the Euclidean distance between two Coord objects.
public class EuclideanDistComputer
         implements CrossFunction<Coord, Coord, Tuple3<Integer, Integer, Double>> {

  @Override
  public Tuple3<Integer, Integer, Double> cross(Coord c1, Coord c2) {
    // compute Euclidean distance of coordinates
    double dist = sqrt(pow(c1.x - c2.x, 2) + pow(c1.y - c2.y, 2));
    return new Tuple3<Integer, Integer, Double>(c1.id, c2.id, dist);
  }
}

DataSet<Coord> coords1 = // [...]
DataSet<Coord> coords2 = // [...]
DataSet<Tuple3<Integer, Integer, Double>>
            distances =
            coords1.cross(coords2)
                   // apply CrossFunction
                   .with(new EuclideanDistComputer());
~~~

####  带 Projection 的 Cross

Cross 转换也可以使用 projection  构造结果元组，如下所示：

~~~java
DataSet<Tuple3<Integer, Byte, String>> input1 = // [...]
DataSet<Tuple2<Integer, Double>> input2 = // [...]
DataSet<Tuple4<Integer, Byte, Integer, Double>
            result =
            input1.cross(input2)
                  // select and reorder fields of matching tuples
                  .projectSecond(0).projectFirst(1,0).projectSecond(1);
~~~

Cross projection 中的字段选择与Join结果的 projection 方式相同。

</div>
<div data-lang="scala" markdown="1">

~~~scala
case class Coord(id: Int, x: Int, y: Int)

val coords1: DataSet[Coord] = // [...]
val coords2: DataSet[Coord] = // [...]

val distances = coords1.cross(coords2) {
  (c1, c2) =>
    val dist = sqrt(pow(c1.x - c2.x, 2) + pow(c1.y - c2.y, 2))
    (c1.id, c2.id, dist)
}
~~~


</div>
<div data-lang="python" markdown="1">

~~~python
 class Euclid(CrossFunction):
   def cross(self, c1, c2):
     return (c1[0], c2[0], sqrt(pow(c1[1] - c2.[1], 2) + pow(c1[2] - c2[2], 2)))

 distances = coords1.cross(coords2).using(Euclid())
~~~

#### 带 Projection 的 Cross

Cross 转换也可以使用 projection  构造结果元组，如下所示：

~~~python
result = input1.cross(input2).projectFirst(1,0).projectSecond(0,1);
~~~

Cross projection 中的字段选择与Join结果的 projection 方式相同。

</div>
</div>

####  带 DataSet Size Hint的Cross

为了引导优化器选择正确的执行策略，可以提示DataSet的大小，如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple4<Integer, String, Integer, String>>
            udfResult =
                  // hint that the second DataSet is very small
            input1.crossWithTiny(input2)
                  // apply any Cross function (or projection)
                  .with(new MyCrosser());

DataSet<Tuple3<Integer, Integer, String>>
            projectResult =
                  // hint that the second DataSet is very large
            input1.crossWithHuge(input2)
                  // apply a projection (or any Cross function)
                  .projectFirst(0,1).projectSecond(1);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[(Int, String)] = // [...]
val input2: DataSet[(Int, String)] = // [...]

// hint that the second DataSet is very small
val result1 = input1.crossWithTiny(input2)

// hint that the second DataSet is very large
val result1 = input1.crossWithHuge(input2)

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 #hint that the second DataSet is very small
 result1 = input1.cross_with_tiny(input2)

 #hint that the second DataSet is very large
 result1 = input1.cross_with_huge(input2)

~~~

</div>
</div>

### CoGroup

CoGroup转换共同处理两个DataSet的组。两个数据集都被分组到一个定义的键上，并且共享相同键的两个数据组的组被共同递交给用户定义的co-group函数。
如果对于特定的键只有一个数据集有一个组，则共同组功能用这个组和一个空组来调用。co-group 函数可以分别遍历两个组的元素并返回任意数量的结果元素。

与Reduce, GroupReduce, 和 Join 相似， keys可以通过不同的 key-selection 方法定。

#### 在DataSets上的CoGroup

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

该示例显示如何按Field Position Keys（仅Tuple DataSets）进行分组。您可以对Pojo类型和key expressions进行相同的操作。

~~~java
// Some CoGroupFunction definition
class MyCoGrouper
         implements CoGroupFunction<Tuple2<String, Integer>, Tuple2<String, Double>, Double> {

  @Override
  public void coGroup(Iterable<Tuple2<String, Integer>> iVals,
                      Iterable<Tuple2<String, Double>> dVals,
                      Collector<Double> out) {

    Set<Integer> ints = new HashSet<Integer>();

    // add all Integer values in group to set
    for (Tuple2<String, Integer>> val : iVals) {
      ints.add(val.f1);
    }

    // multiply each Double value with each unique Integer values of group
    for (Tuple2<String, Double> val : dVals) {
      for (Integer i : ints) {
        out.collect(val.f1 * i);
      }
    }
  }
}

// [...]
DataSet<Tuple2<String, Integer>> iVals = // [...]
DataSet<Tuple2<String, Double>> dVals = // [...]
DataSet<Double> output = iVals.coGroup(dVals)
                         // group first DataSet on first tuple field
                         .where(0)
                         // group second DataSet on first tuple field
                         .equalTo(0)
                         // apply CoGroup function on each pair of groups
                         .with(new MyCoGrouper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val iVals: DataSet[(String, Int)] = // [...]
val dVals: DataSet[(String, Double)] = // [...]

val output = iVals.coGroup(dVals).where(0).equalTo(0) {
  (iVals, dVals, out: Collector[Double]) =>
    val ints = iVals map { _._2 } toSet

    for (dVal <- dVals) {
      for (i <- ints) {
        out.collect(dVal._2 * i)
      }
    }
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class CoGroup(CoGroupFunction):
   def co_group(self, ivals, dvals, collector):
     ints = dict()
     # add all Integer values in group to set
     for value in ivals:
       ints[value[1]] = 1
     # multiply each Double value with each unique Integer values of group
     for value in dvals:
       for i in ints.keys():
         collector.collect(value[1] * i)


 output = ivals.co_group(dvals).where(0).equal_to(0).using(CoGroup())
~~~

</div>
</div>


### Union

生成两个DataSet的union ，它们必须是相同的类型。 超过两个数据集的union 可以用多个union 调用实现，如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> vals1 = // [...]
DataSet<Tuple2<String, Integer>> vals2 = // [...]
DataSet<Tuple2<String, Integer>> vals3 = // [...]
DataSet<Tuple2<String, Integer>> unioned = vals1.union(vals2).union(vals3);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val vals1: DataSet[(String, Int)] = // [...]
val vals2: DataSet[(String, Int)] = // [...]
val vals3: DataSet[(String, Int)] = // [...]

val unioned = vals1.union(vals2).union(vals3)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 unioned = vals1.union(vals2).union(vals3)
~~~

</div>
</div>

### Rebalance
均匀重新平衡DataSet的并行分区以消除数据倾斜。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<String> in = // [...]
// rebalance DataSet and apply a Map transformation.
DataSet<Tuple2<String, String>> out = in.rebalance()
                                        .map(new Mapper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[String] = // [...]
// rebalance DataSet and apply a Map transformation.
val out = in.rebalance().map { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>


### Hash-Partition


通过数据集上给定的key进行Hash-partitions。
key可以为position keys，expression keys和 key selector functions （请参阅 [Reduce 示例](#reduce-on-grouped-dataset) 以了解如何指定键）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> in = // [...]
// hash-partition DataSet by String value and apply a MapPartition transformation.
DataSet<Tuple2<String, String>> out = in.partitionByHash(0)
                                        .mapPartition(new PartitionMapper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[(String, Int)] = // [...]
// hash-partition DataSet by String value and apply a MapPartition transformation.
val out = in.partitionByHash(0).mapPartition { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

### Range-Partition


通过数据集上给定的key进行Range-partitions。
key可以为position keys，expression keys和 key selector functions （请参阅 [Reduce 示例](#reduce-on-grouped-dataset) 以了解如何指定键）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> in = // [...]
// range-partition DataSet by String value and apply a MapPartition transformation.
DataSet<Tuple2<String, String>> out = in.partitionByRange(0)
                                        .mapPartition(new PartitionMapper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[(String, Int)] = // [...]
// range-partition DataSet by String value and apply a MapPartition transformation.
val out = in.partitionByRange(0).mapPartition { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>


### Sort Partition

在指定的字段中按指定的顺序在本地对DataSet的所有分区进行排序。
可以将字段指定为字段表达式或字段位置（请参阅 [Reduce 示例](#reduce-on-grouped-dataset)以了解如何指定键）。
可以通过多次`sortPartition()` 调用将分区排序用在多个字段上。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> in = // [...]
// Locally sort partitions in ascending order on the second String field and
// in descending order on the first String field.
// Apply a MapPartition transformation on the sorted partitions.
DataSet<Tuple2<String, String>> out = in.sortPartition(1, Order.ASCENDING)
                                        .sortPartition(0, Order.DESCENDING)
                                        .mapPartition(new PartitionMapper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[(String, Int)] = // [...]
// Locally sort partitions in ascending order on the second String field and
// in descending order on the first String field.
// Apply a MapPartition transformation on the sorted partitions.
val out = in.sortPartition(1, Order.ASCENDING)
            .sortPartition(0, Order.DESCENDING)
            .mapPartition { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

### First-n

返回DataSet的前n个（任意）元素。First-n可以应用于常规的DataSet，分组的DataSet或分组排序的DataSet。分组键可以指定为key-selector 函数或key-selector（请参阅[Reduce 示例](#reduce-on-grouped-dataset) 以了解如何指定键）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> in = // [...]
// Return the first five (arbitrary) elements of the DataSet
DataSet<Tuple2<String, Integer>> out1 = in.first(5);

// Return the first two (arbitrary) elements of each String group
DataSet<Tuple2<String, Integer>> out2 = in.groupBy(0)
                                          .first(2);

// Return the first three elements of each String group ordered by the Integer field
DataSet<Tuple2<String, Integer>> out3 = in.groupBy(0)
                                          .sortGroup(1, Order.ASCENDING)
                                          .first(3);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[(String, Int)] = // [...]
// Return the first five (arbitrary) elements of the DataSet
val out1 = in.first(5)

// Return the first two (arbitrary) elements of each String group
val out2 = in.groupBy(0).first(2)

// Return the first three elements of each String group ordered by the Integer field
val out3 = in.groupBy(0).sortGroup(1, Order.ASCENDING).first(3)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

{% top %}

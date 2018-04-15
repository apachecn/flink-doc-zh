---
title: "Flink DataSet API Programming Guide"
nav-id: batch
nav-title: Batch (DataSet API)
nav-parent_id: dev
nav-pos: 30
nav-show_overview: true
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

Flink中的DataSet程序是实现数据集转换的常规程序 (例如： filtering, mapping, joining, grouping)。
DataSet从某些数据源初始化(例如, 通过读取文件, 或者本地数据集合)。 结果通过 sink 返回，比如将
文件写入（分布式）文件或写入标准输出（如命令行终端）。Flink程序可以在各种情况下运行，独立
运行或嵌入其他程序中。执行可以发生在本地JVM或许多机器的集群中。

请参阅[基本概念]（{{site.baseurl}}/dev/api_concepts.html）以了解Flink API的基本概念。

为了创建您自己的 Flink DataSet 程序, 我们鼓励您从 [Flink 程序的解刨]({{ site.baseurl }}/dev/api_concepts.html#anatomy-of-a-flink-program)开始
并逐渐添加自己的 [transformations](#dataset-transformations)。其余部分充当其他操作和高级功能的参考。

* This will be replaced by the TOC
{:toc}

示例程序
---------------

以下程序是WordCount的完整程序。您可以在本地运行， 您只需将正确的Flink的库包含到您的项目中
(参考[Linking with Flink]({{ site.baseurl }}/dev/linking_with_flink.html))。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
public class WordCountExample {
    public static void main(String[] args) throws Exception {
        final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        DataSet<String> text = env.fromElements(
            "Who's there?",
            "I think I hear them. Stand, ho! Who's there?");

        DataSet<Tuple2<String, Integer>> wordCounts = text
            .flatMap(new LineSplitter())
            .groupBy(0)
            .sum(1);

        wordCounts.print();
    }

    public static class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String line, Collector<Tuple2<String, Integer>> out) {
            for (String word : line.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }
}
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

object WordCount {
  def main(args: Array[String]) {

    val env = ExecutionEnvironment.getExecutionEnvironment
    val text = env.fromElements(
      "Who's there?",
      "I think I hear them. Stand, ho! Who's there?")

    val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
      .map { (_, 1) }
      .groupBy(0)
      .sum(1)

    counts.print()
  }
}
{% endhighlight %}
</div>

</div>

{% top %}

DataSet Transformations
-----------------------

Data transformations 把一个或多个 DataSet 转换成一个新的 DataSet。程序可以把多个 transformations 组合成一个复杂的程序集。

本节将简要介绍可用的transformation。
 [transformation 文档](dataset_transformations.html)里包含的关于所有 transformation 的完整描述。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

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
{% highlight java %}
data.map(new MapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>Takes one element and produces zero, one, or more elements. </p>
{% highlight java %}
data.flatMap(new FlatMapFunction<String, String>() {
  public void flatMap(String value, Collector<String> out) {
    for (String s : value.split(" ")) {
      out.collect(s);
    }
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
        <p>Transforms a parallel partition in a single function call. The function gets the partition
        as an <code>Iterable</code> stream and can produce an arbitrary number of result values. The number of
        elements in each partition depends on the degree-of-parallelism and previous operations.</p>
{% highlight java %}
data.mapPartition(new MapPartitionFunction<String, Long>() {
  public void mapPartition(Iterable<String> values, Collector<Long> out) {
    long c = 0;
    for (String s : values) {
      c++;
    }
    out.collect(c);
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>Evaluates a boolean function for each element and retains those for which the function
        returns true.<br/>

        <strong>IMPORTANT:</strong> The system assumes that the function does not modify the elements on which the predicate is applied. Violating this assumption
        can lead to incorrect results.
        </p>
{% highlight java %}
data.filter(new FilterFunction<Integer>() {
  public boolean filter(Integer value) { return value > 1000; }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>Combines a group of elements into a single element by repeatedly combining two elements
        into one. Reduce may be applied on a full data set or on a grouped data set.</p>
{% highlight java %}
data.reduce(new ReduceFunction<Integer> {
  public Integer reduce(Integer a, Integer b) { return a + b; }
});
{% endhighlight %}
        <p>If the reduce was applied to a grouped data set then you can specify the way that the
        runtime executes the combine phase of the reduce by supplying a <code>CombineHint</code> to
        <code>setCombineHint</code>. The hash-based strategy should be faster in most cases,
        especially if the number of different keys is small compared to the number of input
        elements (eg. 1/10).</p>
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>Combines a group of elements into one or more elements. ReduceGroup may be applied on a
        full data set or on a grouped data set.</p>
{% highlight java %}
data.reduceGroup(new GroupReduceFunction<Integer, Integer> {
  public void reduce(Iterable<Integer> values, Collector<Integer> out) {
    int prefixSum = 0;
    for (Integer i : values) {
      prefixSum += i;
      out.collect(prefixSum);
    }
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>Aggregates a group of values into a single value. Aggregation functions can be thought of
        as built-in reduce functions. Aggregate may be applied on a full data set, or on a grouped
        data set.</p>
{% highlight java %}
Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.aggregate(SUM, 0).and(MIN, 2);
{% endhighlight %}
	<p>You can also use short-hand syntax for minimum, maximum, and sum aggregations.</p>
	{% highlight java %}
	Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.sum(0).andMin(2);
	{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Distinct</strong></td>
      <td>
        <p>Returns the distinct elements of a data set. It removes the duplicate entries
        from the input DataSet, with respect to all fields of the elements, or a subset of fields.</p>
{% highlight java %}
data.distinct();
{% endhighlight %}
        <p>Distinct is implemented using a reduce function. You can specify the way that the
        runtime executes the combine phase of the reduce by supplying a <code>CombineHint</code> to
        <code>setCombineHint</code>. The hash-based strategy should be faster in most cases,
        especially if the number of different keys is small compared to the number of input
        elements (eg. 1/10).</p>
      </td>
    </tr>

    <tr>
      <td><strong>Join</strong></td>
      <td>
        Joins two data sets by creating all pairs of elements that are equal on their keys.
        Optionally uses a JoinFunction to turn the pair of elements into a single element, or a
        FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)
        elements. See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight java %}
result = input1.join(input2)
               .where(0)       // key of the first input (tuple field 0)
               .equalTo(1);    // key of the second input (tuple field 1)
{% endhighlight %}
        You can specify the way that the runtime executes the join via <i>Join Hints</i>. The hints
        describe whether the join happens through partitioning or broadcasting, and whether it uses
        a sort-based or a hash-based algorithm. Please refer to the
        <a href="dataset_transformations.html#join-algorithm-hints">Transformations Guide</a> for
        a list of possible hints and an example.</br>
        If no hint is specified, the system will try to make an estimate of the input sizes and
        pick the best strategy according to those estimates.
{% highlight java %}
// This executes a join by broadcasting the first data set
// using a hash table for the broadcast data
result = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
               .where(0).equalTo(1);
{% endhighlight %}
        Note that the join transformation works only for equi-joins. Other join types need to be expressed using OuterJoin or CoGroup.
      </td>
    </tr>

    <tr>
      <td><strong>OuterJoin</strong></td>
      <td>
        Performs a left, right, or full outer join on two data sets. Outer joins are similar to regular (inner) joins and create all pairs of elements that are equal on their keys. In addition, records of the "outer" side (left, right, or both in case of full) are preserved if no matching key is found in the other side. Matching pairs of elements (or one element and a <code>null</code> value for the other input) are given to a JoinFunction to turn the pair of elements into a single element, or to a FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)         elements. See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight java %}
input1.leftOuterJoin(input2) // rightOuterJoin or fullOuterJoin for right or full outer joins
      .where(0)              // key of the first input (tuple field 0)
      .equalTo(1)            // key of the second input (tuple field 1)
      .with(new JoinFunction<String, String, String>() {
          public String join(String v1, String v2) {
             // NOTE:
             // - v2 might be null for leftOuterJoin
             // - v1 might be null for rightOuterJoin
             // - v1 OR v2 might be null for fullOuterJoin
          }
      });
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>CoGroup</strong></td>
      <td>
        <p>The two-dimensional variant of the reduce operation. Groups each input on one or more
        fields and then joins the groups. The transformation function is called per pair of groups.
        See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define coGroup keys.</p>
{% highlight java %}
data1.coGroup(data2)
     .where(0)
     .equalTo(1)
     .with(new CoGroupFunction<String, String, String>() {
         public void coGroup(Iterable<String> in1, Iterable<String> in2, Collector<String> out) {
           out.collect(...);
         }
      });
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>Builds the Cartesian product (cross product) of two inputs, creating all pairs of
        elements. Optionally uses a CrossFunction to turn the pair of elements into a single
        element</p>
{% highlight java %}
DataSet<Integer> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<Tuple2<Integer, String>> result = data1.cross(data2);
{% endhighlight %}
      <p>Note: Cross is potentially a <b>very</b> compute-intensive operation which can challenge even large compute clusters! It is advised to hint the system with the DataSet sizes by using <i>crossWithTiny()</i> and <i>crossWithHuge()</i>.</p>
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>Produces the union of two data sets.</p>
{% highlight java %}
DataSet<String> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<String> result = data1.union(data2);
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Rebalance</strong></td>
      <td>
        <p>Evenly rebalances the parallel partitions of a data set to eliminate data skew. Only Map-like transformations may follow a rebalance transformation.</p>
{% highlight java %}
DataSet<String> in = // [...]
DataSet<String> result = in.rebalance()
                           .map(new Mapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Hash-Partition</strong></td>
      <td>
        <p>Hash-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionByHash(0)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Range-Partition</strong></td>
      <td>
        <p>Range-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionByRange(0)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Custom Partitioning</strong></td>
      <td>
        <p>Manually specify a partitioning over the data.
          <br/>
          <i>Note</i>: This method works only on single field keys.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionCustom(Partitioner<K> partitioner, key)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Sort Partition</strong></td>
      <td>
        <p>Locally sorts all partitions of a data set on a specified field in a specified order.
          Fields can be specified as tuple positions or field expressions.
          Sorting on multiple fields is done by chaining sortPartition() calls.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.sortPartition(1, Order.ASCENDING)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>First-n</strong></td>
      <td>
        <p>Returns the first n (arbitrary) elements of a data set. First-n can be applied on a regular data set, a grouped data set, or a grouped-sorted data set. Grouping keys can be specified as key-selector functions or field position keys.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
// regular data set
DataSet<Tuple2<String,Integer>> result1 = in.first(3);
// grouped data set
DataSet<Tuple2<String,Integer>> result2 = in.groupBy(0)
                                            .first(3);
// grouped-sorted data set
DataSet<Tuple2<String,Integer>> result3 = in.groupBy(0)
                                            .sortGroup(1, Order.ASCENDING)
                                            .first(3);
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

----------

以下 transformation 可用于 Tuples 的数据集：

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Project</strong></td>
      <td>
        <p>Selects a subset of fields from the tuples</p>
{% highlight java %}
DataSet<Tuple3<Integer, Double, String>> in = // [...]
DataSet<Tuple2<String, Integer>> out = in.project(2,0);
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>MinBy / MaxBy</strong></td>
      <td>
        <p>Selects a tuple from a group of tuples whose values of one or more fields are minimum (maximum). The fields which are used for comparison must be valid key fields, i.e., comparable. If multiple tuples have minimum (maximum) field values, an arbitrary tuple of these tuples is returned. MinBy (MaxBy) may be applied on a full data set or a grouped data set.</p>
{% highlight java %}
DataSet<Tuple3<Integer, Double, String>> in = // [...]
// a DataSet with a single tuple with minimum values for the Integer and String fields.
DataSet<Tuple3<Integer, Double, String>> out = in.minBy(0, 2);
// a DataSet with one tuple for each group with the minimum value for the Double field.
DataSet<Tuple3<Integer, Double, String>> out2 = in.groupBy(2)
                                                  .minBy(1);
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

</div>
<div data-lang="scala" markdown="1">
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
{% highlight scala %}
data.map { x => x.toInt }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>Takes one element and produces zero, one, or more elements. </p>
{% highlight scala %}
data.flatMap { str => str.split(" ") }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
        <p>Transforms a parallel partition in a single function call. The function get the partition
        as an `Iterator` and can produce an arbitrary number of result values. The number of
        elements in each partition depends on the degree-of-parallelism and previous operations.</p>
{% highlight scala %}
data.mapPartition { in => in map { (_, 1) } }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>Evaluates a boolean function for each element and retains those for which the function
        returns true.<br/>
        <strong>IMPORTANT:</strong> The system assumes that the function does not modify the element on which the predicate is applied.
        Violating this assumption can lead to incorrect results.</p>
{% highlight scala %}
data.filter { _ > 1000 }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>Combines a group of elements into a single element by repeatedly combining two elements
        into one. Reduce may be applied on a full data set, or on a grouped data set.</p>
{% highlight scala %}
data.reduce { _ + _ }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>Combines a group of elements into one or more elements. ReduceGroup may be applied on a
        full data set, or on a grouped data set.</p>
{% highlight scala %}
data.reduceGroup { elements => elements.sum }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>Aggregates a group of values into a single value. Aggregation functions can be thought of
        as built-in reduce functions. Aggregate may be applied on a full data set, or on a grouped
        data set.</p>
{% highlight scala %}
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Double)] = input.aggregate(SUM, 0).aggregate(MIN, 2)
{% endhighlight %}
  <p>You can also use short-hand syntax for minimum, maximum, and sum aggregations.</p>
{% highlight scala %}
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Double)] = input.sum(0).min(2)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Distinct</strong></td>
      <td>
        <p>Returns the distinct elements of a data set. It removes the duplicate entries
        from the input DataSet, with respect to all fields of the elements, or a subset of fields.</p>
      {% highlight scala %}
         data.distinct()
      {% endhighlight %}
      </td>
    </tr>

    </tr>
      <td><strong>Join</strong></td>
      <td>
        Joins two data sets by creating all pairs of elements that are equal on their keys.
        Optionally uses a JoinFunction to turn the pair of elements into a single element, or a
        FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)
        elements. See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight scala %}
// In this case tuple fields are used as keys. "0" is the join field on the first tuple
// "1" is the join field on the second tuple.
val result = input1.join(input2).where(0).equalTo(1)
{% endhighlight %}
        You can specify the way that the runtime executes the join via <i>Join Hints</i>. The hints
        describe whether the join happens through partitioning or broadcasting, and whether it uses
        a sort-based or a hash-based algorithm. Please refer to the
        <a href="dataset_transformations.html#join-algorithm-hints">Transformations Guide</a> for
        a list of possible hints and an example.</br>
        If no hint is specified, the system will try to make an estimate of the input sizes and
        pick the best strategy according to those estimates.
{% highlight scala %}
// This executes a join by broadcasting the first data set
// using a hash table for the broadcast data
val result = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
                   .where(0).equalTo(1)
{% endhighlight %}
          Note that the join transformation works only for equi-joins. Other join types need to be expressed using OuterJoin or CoGroup.
      </td>
    </tr>

    <tr>
      <td><strong>OuterJoin</strong></td>
      <td>
        Performs a left, right, or full outer join on two data sets. Outer joins are similar to regular (inner) joins and create all pairs of elements that are equal on their keys. In addition, records of the "outer" side (left, right, or both in case of full) are preserved if no matching key is found in the other side. Matching pairs of elements (or one element and a `null` value for the other input) are given to a JoinFunction to turn the pair of elements into a single element, or to a FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)         elements. See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight scala %}
val joined = left.leftOuterJoin(right).where(0).equalTo(1) {
   (left, right) =>
     val a = if (left == null) "none" else left._1
     (a, right)
  }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>CoGroup</strong></td>
      <td>
        <p>The two-dimensional variant of the reduce operation. Groups each input on one or more
        fields and then joins the groups. The transformation function is called per pair of groups.
        See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define coGroup keys.</p>
{% highlight scala %}
data1.coGroup(data2).where(0).equalTo(1)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>Builds the Cartesian product (cross product) of two inputs, creating all pairs of
        elements. Optionally uses a CrossFunction to turn the pair of elements into a single
        element</p>
{% highlight scala %}
val data1: DataSet[Int] = // [...]
val data2: DataSet[String] = // [...]
val result: DataSet[(Int, String)] = data1.cross(data2)
{% endhighlight %}
        <p>Note: Cross is potentially a <b>very</b> compute-intensive operation which can challenge even large compute clusters! It is advised to hint the system with the DataSet sizes by using <i>crossWithTiny()</i> and <i>crossWithHuge()</i>.</p>
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>Produces the union of two data sets.</p>
{% highlight scala %}
data.union(data2)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Rebalance</strong></td>
      <td>
        <p>Evenly rebalances the parallel partitions of a data set to eliminate data skew. Only Map-like transformations may follow a rebalance transformation.</p>
{% highlight scala %}
val data1: DataSet[Int] = // [...]
val result: DataSet[(Int, String)] = data1.rebalance().map(...)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Hash-Partition</strong></td>
      <td>
        <p>Hash-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.partitionByHash(0).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Range-Partition</strong></td>
      <td>
        <p>Range-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.partitionByRange(0).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    </tr>
    <tr>
      <td><strong>Custom Partitioning</strong></td>
      <td>
        <p>Manually specify a partitioning over the data.
          <br/>
          <i>Note</i>: This method works only on single field keys.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in
  .partitionCustom(partitioner: Partitioner[K], key)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Sort Partition</strong></td>
      <td>
        <p>Locally sorts all partitions of a data set on a specified field in a specified order.
          Fields can be specified as tuple positions or field expressions.
          Sorting on multiple fields is done by chaining sortPartition() calls.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.sortPartition(1, Order.ASCENDING).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>First-n</strong></td>
      <td>
        <p>Returns the first n (arbitrary) elements of a data set. First-n can be applied on a regular data set, a grouped data set, or a grouped-sorted data set. Grouping keys can be specified as key-selector functions,
        tuple positions or case class fields.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
// regular data set
val result1 = in.first(3)
// grouped data set
val result2 = in.groupBy(0).first(3)
// grouped-sorted data set
val result3 = in.groupBy(0).sortGroup(1, Order.ASCENDING).first(3)
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

----------

The following transformations are available on data sets of Tuples:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>MinBy / MaxBy</strong></td>
      <td>
        <p>Selects a tuple from a group of tuples whose values of one or more fields are minimum (maximum). The fields which are used for comparison must be valid key fields, i.e., comparable. If multiple tuples have minimum (maximum) field values, an arbitrary tuple of these tuples is returned. MinBy (MaxBy) may be applied on a full data set or a grouped data set.</p>
{% highlight java %}
val in: DataSet[(Int, Double, String)] = // [...]
// a data set with a single tuple with minimum values for the Int and String fields.
val out: DataSet[(Int, Double, String)] = in.minBy(0, 2)
// a data set with one tuple for each group with the minimum value for the Double field.
val out2: DataSet[(Int, Double, String)] = in.groupBy(2)
                                             .minBy(1)
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

通过匿名模式匹配从tuples，case classes和collections中提取，如下所示：
{% highlight scala %}
val data: DataSet[(Int, String, Double)] = // [...]
data.map {
  case (id, name, temperature) => // [...]
}
{% endhighlight %}
不支持API开箱即用。要使用此功能，您应该使用<a href="{{site.baseurl }}/dev/scala_api_extensions.html"> Scala API扩展</a>。

</div>
</div>

transformation 的 [并行度]({{ site.baseurl }}/dev/parallel.html) 可以通过 `setParallelism(int)`设置，而通过
`name(String)` 可以给 transformation 分配一个自定义的名称，这有助于调试.。
 [Data Sources](#data-sources) 和 [Data Sinks](#data-sinks) 也是如此。

`withParameters(Configuration)`  传递 Configuration 对象, 可以从用户函数内的  `open()` 方法访问。

{% top %}

数据源
------------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

数据源创建初始 DataSet, 例如冲文件或 Java 集合。创建 DataSet 的一般机制是在
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}后面抽象出来的。
 Flink带有几种内置格式，可以从常用文件格式创建 DataSet 。很多在 *ExecutionEnvironmen* 上都有快捷方式。

基于文件:

- `readTextFile(path)` / `TextInputFormat` - 以行的方式读取文件并将其作为字符串返回。

- `readTextFileWithValue(path)` / `TextValueInputFormat` - 以行的方式读取文件并将其作为 StringValues 返回。
  StringValues 是可变的字符串。

- `readCsvFile(path)` / `CsvInputFormat` - 解析逗号（或其他字符）分隔字段的文件。返回元组或POJO的数据集。字段类型支持基本的Java类型及其Value对应项。

- `readFileOfPrimitives(path, Class)` / `PrimitiveInputFormat` -解析新行（或其他字符序列）分隔的原始数据类型（如String或Integer）的文件。

- `readFileOfPrimitives(path, delimiter, Class)` / `PrimitiveInputFormat` - 使用给定的分隔符解析新行（或其他字符序列）分隔的基本数据类型（如“String”或“Integer”）的文件。

- `readHadoopFile(FileInputFormat, Key, Value, path)` / `FileInputFormat` - 创建JobConf并使用指定的FileInputFormat，Key类和Value类从指定路径读取文件，并将它们作为Tuple2<Key,Value>返回。

- `readSequenceFile(Key, Value, path)` / `SequenceFileInputFormat` - 创建JobConf并从SequenceFileInputFormat，Key类和Value类的指定路径中读取文件，并将它们作为Tuple2 <Key，Value>返回。


基于集合:

- `fromCollection(Collection)` - 从Java Java.util.Collection 创建 DataSet。在集合中所有元素的数据类型必须一样.

- `fromCollection(Iterator, Class)` -  从迭代器创建 DataSet 。 Class 指定迭代器返回的元素的数据类型。

- `fromElements(T ...)` - 根据给定的对象序列创建一个 DataSet。所有对象必须是相同的类型。

- `fromParallelCollection(SplittableIterator, Class)` -  并行创建迭代器中的数据集。Class指定迭代器返回的元素的数据类型。

- `generateSequence(from, to)` - 并行生成给定间隔内的数字序列。

通用:

- `readFile(inputFormat, path)` / `FileInputFormat` -接受文件输入格式。

- `createInput(inputFormat)` / `InputFormat` -  接受通用输入格式。

**示例**

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// read text file from local files system
DataSet<String> localLines = env.readTextFile("file:///path/to/my/textfile");

// read text file from a HDFS running at nnHost:nnPort
DataSet<String> hdfsLines = env.readTextFile("hdfs://nnHost:nnPort/path/to/my/textfile");

// read a CSV file with three fields
DataSet<Tuple3<Integer, String, Double>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
	                       .types(Integer.class, String.class, Double.class);

// read a CSV file with five fields, taking only two of them
DataSet<Tuple2<String, Double>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                               .includeFields("10010")  // take the first and the fourth field
	                       .types(String.class, Double.class);

// read a CSV file with three fields into a POJO (Person.class) with corresponding fields
DataSet<Person>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                         .pojoType(Person.class, "name", "age", "zipcode");


// read a file from the specified path of type TextInputFormat
DataSet<Tuple2<LongWritable, Text>> tuples =
 env.readHadoopFile(new TextInputFormat(), LongWritable.class, Text.class, "hdfs://nnHost:nnPort/path/to/file");

// read a file from the specified path of type SequenceFileInputFormat
DataSet<Tuple2<IntWritable, Text>> tuples =
 env.readSequenceFile(IntWritable.class, Text.class, "hdfs://nnHost:nnPort/path/to/file");

// creates a set from some given elements
DataSet<String> value = env.fromElements("Foo", "bar", "foobar", "fubar");

// generate a number sequence
DataSet<Long> numbers = env.generateSequence(1, 10000000);

// Read data from a relational database using the JDBC input format
DataSet<Tuple2<String, Integer> dbData =
    env.createInput(
      JDBCInputFormat.buildJDBCInputFormat()
                     .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                     .setDBUrl("jdbc:derby:memory:persons")
                     .setQuery("select name, age from persons")
                     .setRowTypeInfo(new RowTypeInfo(BasicTypeInfo.STRING_TYPE_INFO, BasicTypeInfo.INT_TYPE_INFO))
                     .finish()
    );

// Note: Flink's program compiler needs to infer the data types of the data items which are returned
// by an InputFormat. If this information cannot be automatically inferred, it is necessary to
// manually provide the type information as shown in the examples above.
{% endhighlight %}

#### 配置CSV解析

Flink为CSV解析提供了许多配置选项：

- `types(Class ... types)` 指定要解析的字段的类型。**必须配置解析字段的类型。** 在类型为Boolean.class的情况下，“True”（不区分大小写），“False”（区分大小写），“1”和“0”被视为布尔值。

- `lineDelimiter(String del)` 指定单个记录的分隔符。默认的行分隔符是换行符“\n”。

- `fieldDelimiter(String del)` 指定分隔记录字段的分隔符。默认的字段分隔符是逗号字符`','`。

- `includeFields(boolean ... flag)`, `includeFields(String mask)`, or `includeFields(long bitMask)` 定义从输入文件中读取哪些字段（以及要忽略哪些字段）。默认情况下，前n个字段（由 types()定义的类型数量）被解析。

- `parseQuotedStrings(char quoteChar)` 启用带引号的字符串解析。如果字符串字段的第一个字符是引号字符（引号或尾部的空格是*不*修剪的），则字符串将被解析为带引号的字符串。引号字符串中的字段分隔符将被忽略。如果引用字符串字段的最后一个字符不是引号字符，或者引号字符出现在某个不是引号字符串字段开始或结尾的位置，则引号字符串解析失败（除非引号字符使用'\'）。如果启用了带引号的字符串解析并且该字段的第一个字符是*非*引号字符串，则该字符串将被解析为未加引号的字符串。默认情况下，引号字符串解析被禁用。

- `ignoreComments(String commentPrefix)` 指定一个注释前缀。所有以指定的注释前缀开头的行都不会被解析和忽略。默认情况下，不会忽略任何行。

- `ignoreInvalidLines()` 允许宽松的解析，即忽略不能被正确解析的行。默认情况下，lenient解析被禁用，无效行将引发异常。

- `ignoreFirstLine()` 将InputFormat配置为忽略输入文件的第一行。默认情况下，不会忽略任何行。


#### 递归遍历输入目录

对于基于文件的输入，当输入路径是目录时，默认情况下嵌套文件未被枚举。相反，只有基本目录内的文件被读取，而嵌套文件被忽略。通过`recursive.file.enumeration`配置参数可以启用嵌套文件的递归枚举，如下例所示。

{% highlight java %}
// enable recursive enumeration of nested input files
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// create a configuration object
Configuration parameters = new Configuration();

// set the recursive enumeration parameter
parameters.setBoolean("recursive.file.enumeration", true);

// pass the configuration to the data source
DataSet<String> logs = env.readTextFile("file:///path/with.nested/files")
			  .withParameters(parameters);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

数据源创建初始 DataSet, 例如冲文件或 Java 集合。创建 DataSet 的一般机制是在
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}后面抽象出来的。
 Flink带有几种内置格式，可以从常用文件格式创建 DataSet 。很多在 *ExecutionEnvironmen* 上都有快捷方式。

File-based:

- `readTextFile(path)` / `TextInputFormat` - 以行的方式读取文件并将其作为字符串返回。

- `readTextFileWithValue(path)` / `TextValueInputFormat` - 以行的方式读取文件并将其作为 StringValues 返回。
    StringValues 是可变的字符串。

- `readCsvFile(path)` / `CsvInputFormat` - 解析逗号（或其他字符）分隔字段的文件。返回元组,case class objects或POJO的 DataSet。字段类型支持基本的Java类型及其Value对应项。


- `readFileOfPrimitives(path, delimiter)` / `PrimitiveInputFormat` - 解析新行（或其他字符序列）分隔的原始数据类型（如String或Integer）的文件。


- `readHadoopFile(FileInputFormat, Key, Value, path)` / `FileInputFormat` - 创建JobConf并使用指定的FileInputFormat，Key类和Value类从指定路径读取文件，并将它们作为Tuple2<Key,Value>返回。


- `readSequenceFile(Key, Value, path)` / `SequenceFileInputFormat` -  创建JobConf并从SequenceFileInputFormat，Key类和Value类的指定路径中读取文件，并将它们作为Tuple2 <Key，Value>返回。


基于集合:

- `fromCollection(Seq)` - 从 Seq 创建 DataSet。在集合中所有元素的数据类型必须一样。


- `fromCollection(Iterator)` - 从迭代器创建 DataSet 。 Class 指定迭代器返回的元素的数据类型。


- `fromElements(elements: _*)` - 根据给定的对象序列创建一个 DataSet。所有对象必须是相同的类型。


- `fromParallelCollection(SplittableIterator)` - 并行创建迭代器中的数据集。Class指定迭代器返回的元素的数据类型。


- `generateSequence(from, to)` - 并行生成给定间隔内的数字序列。并行生成给定间隔内的数字序列。


通用:

- `readFile(inputFormat, path)` / `FileInputFormat` - 接受文件输入格式。

- `createInput(inputFormat)` / `InputFormat` - 接受通用输入格式。

**示例**

{% highlight scala %}
val env  = ExecutionEnvironment.getExecutionEnvironment

// read text file from local files system
val localLines = env.readTextFile("file:///path/to/my/textfile")

// read text file from a HDFS running at nnHost:nnPort
val hdfsLines = env.readTextFile("hdfs://nnHost:nnPort/path/to/my/textfile")

// read a CSV file with three fields
val csvInput = env.readCsvFile[(Int, String, Double)]("hdfs:///the/CSV/file")

// read a CSV file with five fields, taking only two of them
val csvInput = env.readCsvFile[(String, Double)](
  "hdfs:///the/CSV/file",
  includedFields = Array(0, 3)) // take the first and the fourth field

// CSV input can also be used with Case Classes
case class MyCaseClass(str: String, dbl: Double)
val csvInput = env.readCsvFile[MyCaseClass](
  "hdfs:///the/CSV/file",
  includedFields = Array(0, 3)) // take the first and the fourth field

// read a CSV file with three fields into a POJO (Person) with corresponding fields
val csvInput = env.readCsvFile[Person](
  "hdfs:///the/CSV/file",
  pojoFields = Array("name", "age", "zipcode"))

// create a set from some given elements
val values = env.fromElements("Foo", "bar", "foobar", "fubar")

// generate a number sequence
val numbers = env.generateSequence(1, 10000000)

// read a file from the specified path of type TextInputFormat
val tuples = env.readHadoopFile(new TextInputFormat, classOf[LongWritable],
 classOf[Text], "hdfs://nnHost:nnPort/path/to/file")

// read a file from the specified path of type SequenceFileInputFormat
val tuples = env.readSequenceFile(classOf[IntWritable], classOf[Text],
 "hdfs://nnHost:nnPort/path/to/file")

{% endhighlight %}

#### 配置CSV解析

Flink为CSV解析提供了许多配置选项：

- `lineDelimiter: String` 指定单个记录的分隔符。默认的行分隔符是换行符“\n”。

- `fieldDelimiter: String` 指定分隔记录字段的分隔符。默认的字段分隔符是逗号字符`','`。

- `includeFields: Array[Int]` 定义从输入文件中读取哪些字段（以及要忽略哪些字段）。默认情况下，前n个字段（由 types()定义的类型数量）被解析。

- `pojoFields: Array[String]` 指定映射到CSV字段的POJO的字段。根据POJO字段的类型和顺序自动初始化CSV字段的解析器。

- `parseQuotedStrings: Character` 启用带引号的字符串解析。如果字符串字段的第一个字符是引号字符（引号或尾部的空格是*不*修剪的），则字符串将被解析为带引号的字符串。引号字符串中的字段分隔符将被忽略。如果引用字符串字段的最后一个字符不是引号字符，或者引号字符出现在某个不是引号字符串字段开始或结尾的位置，则引号字符串解析失败（除非引号字符使用'\'）。如果启用了带引号的字符串解析并且该字段的第一个字符是*非*引号字符串，则该字符串将被解析为未加引号的字符串。默认情况下，引号字符串解析被禁用。

- `ignoreComments: String` 指定一个注释前缀。所有以指定的注释前缀开头的行都不会被解析和忽略。默认情况下，不会忽略任何行。

- `lenient: Boolean` 允许宽松的解析，即忽略不能被正确解析的行。默认情况下，lenient解析被禁用，无效行将引发异常。

- `ignoreFirstLine: Boolean` 将InputFormat配置为忽略输入文件的第一行。默认情况下，不会忽略任何行。

#### 递归遍历输入目录

对于基于文件的输入，当输入路径是目录时，默认情况下嵌套文件未被枚举。相反，只有基本目录内的文件被读取，而嵌套文件被忽略。通过`recursive.file.enumeration`配置参数可以启用嵌套文件的递归枚举，如下例所示。

{% highlight scala %}
// enable recursive enumeration of nested input files
val env  = ExecutionEnvironment.getExecutionEnvironment

// create a configuration object
val parameters = new Configuration

// set the recursive enumeration parameter
parameters.setBoolean("recursive.file.enumeration", true)

// pass the configuration to the data source
env.readTextFile("file:///path/with.nested/files").withParameters(parameters)
{% endhighlight %}

</div>
</div>

### 读取压缩文件

Flink目前支持对输入文件进行透明解压缩，如果这些文件标有适当的文件扩展名。这意味着不需要进一步配置输入格式，任何`FileInputFormat`都支持压缩，包括自定义输入格式。请注意，压缩文件可能无法并行读取，从而影响作业的可伸缩性。

下表列出了当前支持的压缩方式。

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">压缩方式</th>
      <th class="text-left">文件扩展名</th>
      <th class="text-left" style="width: 20%">是否可以并行读取</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>DEFLATE</strong></td>
      <td><code>.deflate</code></td>
      <td>no</td>
    </tr>
    <tr>
      <td><strong>GZip</strong></td>
      <td><code>.gz</code>, <code>.gzip</code></td>
      <td>no</td>
    </tr>
    <tr>
      <td><strong>Bzip2</strong></td>
      <td><code>.bz2</code></td>
      <td>no</td>
    </tr>
    <tr>
      <td><strong>XZ</strong></td>
      <td><code>.xz</code></td>
      <td>no</td>
    </tr>
  </tbody>
</table>


{% top %}

Data Sinks
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

Data sinks 消费 DataSets 并且 存储或者返回计算结果。 Data sink 操作使用
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/OutputFormat.java "OutputFormat" %} 进行描述。
Flink 内置了各种内置输出格式，这些输出格式被封装在DataSet上的操作之后：

- `writeAsText()` / `TextOutputFormat` - 将元素按行方式编写为字符串。字符串是通过调用每个元素的* toString()*方法获得的。
- `writeAsFormattedText()` / `TextOutputFormat` -将元素按行方式编写为字符串。字符串是通过调用为每个元素自定义的* format()*方法获得的。
- `writeAsCsv(...)` / `CsvOutputFormat` - 将元组写为逗号分隔的值文件。行和字段分隔符是可配置的。每个字段的值来自对象的*toString()*方法。
- `print()` / `printToErr()` / `print(String msg)` / `printToErr(String msg)` - 打印标准输出 / 标准错误流上每个元素的* toString()*值。可以提供前缀（msg）(可选)，该前缀先输出。
这可以帮助区分不同的*print*的调用。如果并行度大于1，则输出还会与生成输出的任务的标识符一起作为前缀。
- `write()` / `FileOutputFormat` - 自定义文件输出的方法和基类。支持自定义对象到字节的转换。
- `output()`/ `OutputFormat` - 对于非基于文件的data sink（例如将结果存储在数据库中），最通用的输出方法。

DataSet可以输入到多个operation。程序可以输出或打印一个数据集，同时对它们进行额外的转换。


**示例**

标准的data sink 方法:

{% highlight java %}
// text data
DataSet<String> textData = // [...]

// write DataSet to a file on the local file system
textData.writeAsText("file:///my/result/on/localFS");

// write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.writeAsText("hdfs://nnHost:nnPort/my/result/on/localFS");

// write DataSet to a file and overwrite the file if it exists
textData.writeAsText("file:///my/result/on/localFS", WriteMode.OVERWRITE);

// tuples as lines with pipe as the separator "a|b|c"
DataSet<Tuple3<String, Integer, Double>> values = // [...]
values.writeAsCsv("file:///path/to/the/result/file", "\n", "|");

// this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.writeAsText("file:///path/to/the/result/file");

// this writes values as strings using a user-defined TextFormatter object
values.writeAsFormattedText("file:///path/to/the/result/file",
    new TextFormatter<Tuple2<Integer, Integer>>() {
        public String format (Tuple2<Integer, Integer> value) {
            return value.f1 + " - " + value.f0;
        }
    });
{% endhighlight %}

使用自定义的输出格式:

{% highlight java %}
DataSet<Tuple3<String, Integer, Double>> myResult = [...]

// write Tuple DataSet to a relational database
myResult.output(
    // build and configure OutputFormat
    JDBCOutputFormat.buildJDBCOutputFormat()
                    .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                    .setDBUrl("jdbc:derby:memory:persons")
                    .setQuery("insert into persons (name, age, height) values (?,?,?)")
                    .finish()
    );
{% endhighlight %}

#### 本地排序输出

可以使用[tuple 字段位置]({{ site.baseurl }}/dev/api_concepts.html#define-keys-for-tuples) 或者 [字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)可以按照指定的顺序在指定的字段上对 data sink的输出进行本地排序。这适用于每种输出格式。

以下示例展示了如何使用这个特性:

{% highlight java %}

DataSet<Tuple3<Integer, String, Double>> tData = // [...]
DataSet<Tuple2<BookPojo, Double>> pData = // [...]
DataSet<String> sData = // [...]

// sort output on String field in ascending order
tData.sortPartition(1, Order.ASCENDING).print();

// sort output on Double field in descending and Integer field in ascending order
tData.sortPartition(2, Order.DESCENDING).sortPartition(0, Order.ASCENDING).print();

// sort output on the "author" field of nested BookPojo in descending order
pData.sortPartition("f0.author", Order.DESCENDING).writeAsText(...);

// sort output on the full tuple in ascending order
tData.sortPartition("*", Order.ASCENDING).writeAsCsv(...);

// sort atomic type (String) output in descending order
sData.sortPartition("*", Order.DESCENDING).writeAsText(...);

{% endhighlight %}

全局排序现在还不支持。

</div>
<div data-lang="scala" markdown="1">
Data sinks 消费 DataSets 并且 存储或者返回计算结果。 Data sink 操作使用
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/OutputFormat.java "OutputFormat" %} 进行描述。
Flink 内置了各种内置输出格式，这些输出格式被封装在DataSet上的操作之后：

- `writeAsText()` / `TextOutputFormat` -  将元素按行方式编写为字符串。字符串是通过调用每个元素的* toString()*方法获得的。
- `writeAsCsv(...)` / `CsvOutputFormat` - 将元组写为逗号分隔的值文件。行和字段分隔符是可配置的。每个字段的值来自对象的*toString()*方法。
- `print()` / `printToErr()` - 打印标准输出 / 标准错误流上每个元素的*toString()*值。
- `write()` / `FileOutputFormat` -自定义文件输出的方法和基类。支持自定义对象到字节的转换。
- `output()`/ `OutputFormat` - 对于非基于文件的data sink（例如将结果存储在数据库中），最通用的输出方法。

DataSet可以输入到多个operation。程序可以输出或打印一个数据集，同时对它们进行额外的转换。

**示例**

标准的data sink 方法:

{% highlight scala %}
// text data
val textData: DataSet[String] = // [...]

// write DataSet to a file on the local file system
textData.writeAsText("file:///my/result/on/localFS")

// write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.writeAsText("hdfs://nnHost:nnPort/my/result/on/localFS")

// write DataSet to a file and overwrite the file if it exists
textData.writeAsText("file:///my/result/on/localFS", WriteMode.OVERWRITE)

// tuples as lines with pipe as the separator "a|b|c"
val values: DataSet[(String, Int, Double)] = // [...]
values.writeAsCsv("file:///path/to/the/result/file", "\n", "|")

// this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.writeAsText("file:///path/to/the/result/file")

// this writes values as strings using a user-defined formatting
values map { tuple => tuple._1 + " - " + tuple._2 }
  .writeAsText("file:///path/to/the/result/file")
{% endhighlight %}


####本地排序输出

可以使用[tuple 字段位置]({{ site.baseurl }}/dev/api_concepts.html#define-keys-for-tuples) 或者 [字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)可以按照指定的顺序在指定的字段上对 data sink的输出进行本地排序。这适用于每种输出格式。

以下示例展示了如何使用这个特性:

{% highlight scala %}

val tData: DataSet[(Int, String, Double)] = // [...]
val pData: DataSet[(BookPojo, Double)] = // [...]
val sData: DataSet[String] = // [...]

// sort output on String field in ascending order
tData.sortPartition(1, Order.ASCENDING).print()

// sort output on Double field in descending and Int field in ascending order
tData.sortPartition(2, Order.DESCENDING).sortPartition(0, Order.ASCENDING).print()

// sort output on the "author" field of nested BookPojo in descending order
pData.sortPartition("_1.author", Order.DESCENDING).writeAsText(...)

// sort output on the full tuple in ascending order
tData.sortPartition("_", Order.ASCENDING).writeAsCsv(...)

// sort atomic type (String) output in descending order
sData.sortPartition("_", Order.DESCENDING).writeAsText(...)

{% endhighlight %}

全局排序现在还不支持。

</div>
</div>

{% top %}


Iteration Operators
-------------------

Iterations 在Flink程序中实现循环。迭代运算符封装程序的一部分并重复执行它。
将一次迭代的结果（部分解）反馈到下一次迭代中。Flink有两种类型的迭代：**BulkIteration**和**DeltaIteration**。

本节提供了有关如何使用两个operators的简单示例。查看[迭代介绍](iterations.html)页面以获取更详细的介绍。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

#### Bulk Iterations

为了创建一个BulkIteration调用，迭代应该从DataSet的iterate(int)方法开始。这将返回一个`IterativeDataSet`，
它可以通过常规操作符进行转换。iterate(int)的单个参数指定最大迭代次数。

要指定迭代结束,调用IterativeDataSet上的`closeWith(DataSet)`方法来指定哪个变换应该反馈到下一次迭代。
您可以选择使用`closeWith(DataSet,DataSet)`指定终止标准，如果此DataSet为空，则该标准将评估第二个DataSet并终止迭代。
如果未指定终止标准，则迭代在给定的最大次数迭代之后终止。

以下示例迭代估计数字Pi。目标是计算落入单位圆圈的随机点的数量。在每次迭代中，挑选一个随机点。如果这一点位于单位圆内，
我们会增加计数。 然后Pi被估计为结果计数除以迭代次数乘以4。

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Create initial IterativeDataSet
IterativeDataSet<Integer> initial = env.fromElements(0).iterate(10000);

DataSet<Integer> iteration = initial.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer i) throws Exception {
        double x = Math.random();
        double y = Math.random();

        return i + ((x * x + y * y < 1) ? 1 : 0);
    }
});

// Iteratively transform the IterativeDataSet
DataSet<Integer> count = initial.closeWith(iteration);

count.map(new MapFunction<Integer, Double>() {
    @Override
    public Double map(Integer count) throws Exception {
        return count / (double) 10000 * 4;
    }
}).print();

env.execute("Iterative Pi Example");
{% endhighlight %}

你也可以看看 {% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java "K-Means example" %}。
它使用BulkIteration对一组未标记的点进行聚类。

#### Delta Iterations

Delta迭代利用了这样的事实：某些算法不会在每次迭代中更改解决方案的每个数据点。

除了在每次迭代中反馈的部分解决方案（称为工作集）之外，增量迭代在迭代中保持状态（称为解决方案集），可以通过增量更新。
迭代计算的结果是最后迭代之后的状态。请参阅[迭代介绍](terations.html)，了解Delta迭代的基本原理。

定义DeltaIteration与定义BulkIteration类似。对于增量迭代，两个数据集形成每次迭代（工作集和解决方案集）的输入，
并且在每次迭代中产生两个数据集作为结果（新工作集，解集delta）。

要创建一个DeltaIteration，调用iterateDelta(DataSet,int,int)（或分别调用iterateDelta(DataSet,int,int []））。
此方法在初始解决方案集上调用。参数是初始增量集，最大迭代次数和关键位置。
返回的DeltaIteration对象使您可以通过方法iteration.getWorkset()和iteration.getSolutionSet()
访问表示工作集和解决方案集的DataSet。

下面是一个 delta iteration 语法的示例

{% highlight java %}
// read the initial data sets
DataSet<Tuple2<Long, Double>> initialSolutionSet = // [...]

DataSet<Tuple2<Long, Double>> initialDeltaSet = // [...]

int maxIterations = 100;
int keyPosition = 0;

DeltaIteration<Tuple2<Long, Double>, Tuple2<Long, Double>> iteration = initialSolutionSet
    .iterateDelta(initialDeltaSet, maxIterations, keyPosition);

DataSet<Tuple2<Long, Double>> candidateUpdates = iteration.getWorkset()
    .groupBy(1)
    .reduceGroup(new ComputeCandidateChanges());

DataSet<Tuple2<Long, Double>> deltas = candidateUpdates
    .join(iteration.getSolutionSet())
    .where(0)
    .equalTo(0)
    .with(new CompareChangesToCurrent());

DataSet<Tuple2<Long, Double>> nextWorkset = deltas
    .filter(new FilterByThreshold());

iteration.closeWith(deltas, nextWorkset)
	.writeAsCsv(outputPath);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
#### Bulk Iterations

要创建一个BulkIteration调用，迭代应该从DataSet的iterate(int)方法开始，同时指定一个step函数。
step函数获取当前迭代的输入DataSet，并且必须返回一个新的DataSet。iterate(int)的参数是最大迭代次数。

还有`iterateWithTermination(int)`函数接受一个返回两个数据集的步骤函数：迭代步骤的结果和终止标准。
一旦终止标准DataSet为空，迭代就会停止。

以下示例迭代估计数字Pi。目标是计算落入单位圆圈的随机点的数量。在每次迭代中，挑选一个随机点。如果这一点位于单位圆内，
我们会增加计数。 然后Pi被估计为结果计数除以迭代次数乘以4。

{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()

// Create initial DataSet
val initial = env.fromElements(0)

val count = initial.iterate(10000) { iterationInput: DataSet[Int] =>
  val result = iterationInput.map { i =>
    val x = Math.random()
    val y = Math.random()
    i + (if (x * x + y * y < 1) 1 else 0)
  }
  result
}

val result = count map { c => c / 10000.0 * 4 }

result.print()

env.execute("Iterative Pi Example")
{% endhighlight %}

你也可以看看  {% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/clustering/KMeans.scala "K-Means example" %}。
它使用BulkIteration对一组未标记的点进行聚类。

#### Delta Iterations

Delta迭代利用了这样的事实：某些算法不会在每次迭代中更改解决方案的每个数据点。

除了在每次迭代中反馈的部分解决方案（称为工作集）之外，增量迭代在迭代中保持状态（称为解决方案集），可以通过增量更新。
迭代计算的结果是最后迭代之后的状态。请参阅[迭代介绍](terations.html)，了解Delta迭代的基本原理。

定义DeltaIteration与定义BulkIteration类似。对于增量迭代，两个数据集形成每次迭代（工作集和解决方案集）的输入，
并且在每次迭代中产生两个数据集作为结果（新工作集，解集delta）。

要在初始解决方案集上创建一个DeltaIteration调用`iterateDelta(initialWorkset，maxIterations，key)`。
step函数有两个参数：(solutionSet，workset)，并且必须返回两个值：（solutionSetDelta，newWorkset）。

下面是一个 delta iteration 语法的示例

{% highlight scala %}
// read the initial data sets
val initialSolutionSet: DataSet[(Long, Double)] = // [...]

val initialWorkset: DataSet[(Long, Double)] = // [...]

val maxIterations = 100
val keyPosition = 0

val result = initialSolutionSet.iterateDelta(initialWorkset, maxIterations, Array(keyPosition)) {
  (solution, workset) =>
    val candidateUpdates = workset.groupBy(1).reduceGroup(new ComputeCandidateChanges())
    val deltas = candidateUpdates.join(solution).where(0).equalTo(0)(new CompareChangesToCurrent())

    val nextWorkset = deltas.filter(new FilterByThreshold())

    (deltas, nextWorkset)
}

result.writeAsCsv(outputPath)

env.execute()
{% endhighlight %}

</div>
</div>

{% top %}

在函数中操作数据对象
--------------------------------------

Flink的运行时间以Java对象的形式与用户函数交换数据。函数接收来自运行时的输入对象作为方法参数，并返回输出对象作为结果。由于这些对象是由用户函数和运行时代码访问的，因此理解并遵循有关用户代码如何访问（即读取和修改）这些对象的规则是非常重要的。

用户函数接收来自Flink运行时的对象，既可以是常规的方法参数（如MapFunction），也可以是Iterable参数（如GroupReduceFunction）。我们将运行时传递给用户函数的对象称为*输入对象*。 户函数可以将对象作为方法返回值（如`MapFunction`）或`Collector`（如`FlatMapFunction`）发送给Flink运行时。我们将用户函数发出的对象称为*输出对象*。

Flink的DataSet API具有两种模式，它们在Flink的运行时间如何创建或重新使用输入对象方面有所不同。此行为会影响用户函数如何与输入和输出对象交互的保证和约束。以下部分定义了这些规则并给出编写指导方针以编写安全的用户功能代码。

### 禁用对象重用（DEFAULT）

默认情况下，Flink以对象重用禁用模式运行。该模式确保函数始终在函数调用中接收新的输入对象。禁用对象的模式提供了更好的保证，并且更安全。但是，它会带来一定的处理开销，并可能导致更高的Java垃圾收集活动。下表说明了用户函数如何在禁用对象的模式下访问输入和输出对象。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Guarantees and Restrictions</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Reading Input Objects</strong></td>
      <td>
        Within a method call it is guaranteed that the value of an input object does not change. This includes objects served by an Iterable. For example it is safe to collect input objects served by an Iterable in a List or Map. Note that objects may be modified after the method call is left. It is <strong>not safe</strong> to remember objects across function calls.
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Input Objects</strong></td>
      <td>You may modify input objects.</td>
   </tr>
   <tr>
      <td><strong>Emitting Input Objects</strong></td>
      <td>
        You may emit input objects. The value of an input object may have changed after it was emitted. It is <strong>not safe</strong> to read an input object after it was emitted.
      </td>
   </tr>
   <tr>
      <td><strong>Reading Output Objects</strong></td>
      <td>
        An object that was given to a Collector or returned as method result might have changed its value. It is <strong>not safe</strong> to read an output object.
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Output Objects</strong></td>
      <td>You may modify an object after it was emitted and emit it again.</td>
   </tr>
  </tbody>
</table>

**对象重用禁用（默认）模式的编码准则：**

- 不要在方法调用中记住和读取输入对象。
- 在你释放对象之后，不要读取对象。


### 启用对象重用

在启用对象重用的模式下，Flink的运行时将对象实例的数量减到最少。这可以提高性能并降低Java垃圾收集压力。通过调用`ExecutionConfig.enableObjectReuse()`来激活启用对象重用模式。下表说明用户功能如何在启用对象重用模式下访问输入和输出对象。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Guarantees and Restrictions</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Reading input objects received as regular method parameters</strong></td>
      <td>
        Input objects received as regular method arguments are not modified within a function call. Objects may be modified after method call is left. It is <strong>not safe</strong> to remember objects across function calls.
      </td>
   </tr>
   <tr>
      <td><strong>Reading input objects received from an Iterable parameter</strong></td>
      <td>
        Input objects received from an Iterable are only valid until the next() method is called. An Iterable or Iterator may serve the same object instance multiple times. It is <strong>not safe</strong> to remember input objects received from an Iterable, e.g., by putting them in a List or Map.
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Input Objects</strong></td>
      <td>You <strong>must not</strong> modify input objects, except for input objects of MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse).</td>
   </tr>
   <tr>
      <td><strong>Emitting Input Objects</strong></td>
      <td>
        You <strong>must not</strong> emit input objects, except for input objects of MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse).</td>
      </td>
   </tr>
   <tr>
      <td><strong>Reading Output Objects</strong></td>
      <td>
        An object that was given to a Collector or returned as method result might have changed its value. It is <strong>not safe</strong> to read an output object.
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Output Objects</strong></td>
      <td>You may modify an output object and emit it again.</td>
   </tr>
  </tbody>
</table>

**已启用对象重用的编码准则：**

- 不要记住从Iterable接收到的输入对象。
- 不要在方法调用中记住和读取输入对象。
- 除了`MapFunction`，`FlatMapFunction`，`MapPartitionFunction`，`GroupReduceFunction`，`GroupCombineFunction`，`CoGroupFunction`和`InputFormat.next（重用）`的输入对象之外，不要修改或发出输入对象。
- 为了减少对象实例化，您可以始终发出一个重复修改但从不读取的专用输出对象。

{% top %}

Debugging
---------

在分布式集群中的大型数据集上运行数据分析程序之前，确保实施的算法按需运行是个不错的主意。因此，实施数据分析程序通常是检查结果，调试和改进的增量过程。

Flink提供了一些很好的功能，通过支持IDE内的本地调试，注入测试数据和收集结果数据，大大简化了数据分析程序的开发过程。
本节提供了一些提示，如何简化Flink程序的开发。

### 本地执行环境

LocalEnvironment在创建它的相同JVM进程中启动Flink系统。如果从IDE启动LocalEnvironment，则可以在代码中设置断点并轻松地调试程序。

LocalEnvironment被创建和使用如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

DataSet<String> lines = env.readTextFile(pathToTextFile);
// build your program

env.execute();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = ExecutionEnvironment.createLocalEnvironment()

val lines = env.readTextFile(pathToTextFile)
// build your program

env.execute()
{% endhighlight %}
</div>
</div>

### 收集 Data Sources 和 Sinks

通过创建输入文件和读取输出文件完成分析程序的输入并检查其输出是非常麻烦的。Flink提供特殊的Data source和Sink。
这些Data source和Sink由Java集合提供支持以简化测试。一旦程序经过测试，Data source和Sink就可以很容易地由读/写外部数据存储（如HDFS）的Data source和Sink代替

收集数据源可以如下使用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

// Create a DataSet from a list of elements
DataSet<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataSet from any Java collection
List<Tuple2<String, Integer>> data = ...
DataSet<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataSet from an Iterator
Iterator<Long> longIt = ...
DataSet<Long> myLongs = env.fromCollection(longIt, Long.class);
{% endhighlight %}

Sink收集如下被指定:

{% highlight java %}
DataSet<Tuple2<String, Integer>> myResult = ...

List<Tuple2<String, Integer>> outData = new ArrayList<Tuple2<String, Integer>>();
myResult.output(new LocalCollectionOutputFormat(outData));
{% endhighlight %}

**Note:** 目前，收集数据Sink仅限于本地执行，作为调试工具。

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.createLocalEnvironment()

// Create a DataSet from a list of elements
val myInts = env.fromElements(1, 2, 3, 4, 5)

// Create a DataSet from any Collection
val data: Seq[(String, Int)] = ...
val myTuples = env.fromCollection(data)

// Create a DataSet from an Iterator
val longIt: Iterator[Long] = ...
val myLongs = env.fromCollection(longIt)
{% endhighlight %}
</div>
</div>

**Note:** 目前，收集数据源要求数据类型和迭代器实现Serializable。此外，收集数据源不能并行执行（parallelism = 1）。

{% top %}

语义注解
-----------

语义注解可以用来给Flink提示关于函数的行为。他们告诉系统，函数需要读取和计算的函数输入的哪些字段，
以及哪些未经修改的字段从输入转发到输出。语义注解是加速执行的强大手段，
因为它们允许系统推理重复使用多个操作中的排序顺序或分区。
使用语义标注最终可以避免不必要的数据混洗或不必要的排序，并显著提高程序的性能。

**Note:**  语义注解的使用是可选的。但是，保守地使用语义注解至关重要！不正确的语义注解会导致Flink对您的程序做出不正确的判断。
最终可能会导致错误的结果。如果operator的行为不能明确预测，则不应提供注释。请仔细阅读文档。

目前支持以下语义注解。

#### Forwarded Fields Annotation

转发字段信息声明了未被函数更改的输入字段转发到输出中的相同位置或另一个位置。优化器使用此信息来推断某个数据属性（如排序或分区）是否由函数保留。
对于对诸如GroupReduce，GroupCombine，CoGroup和MapPartition等对输入元素组进行操作的函数，
所有定义为转发字段的字段必须始终从同一个输入元素共同转发。由分组函数发出的每个元素的转发字段可能来自函数输入组的不同元素。

字段转发信息是使用[字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)指定的。
转发到输出中相同位置的字段可以通过它们的位置来指定。指定的位置必须对输入和输出数据的类型要相同。
例如，字符串“f2”声明Java输入元组的第三个字段总是等于输出元组中的第三个字段。

通过在输入中指定源字段并在输出中指定目标字段作为字段表达式，来声明未经修改而转发到输出中另一位置的字段。
字符串 `"f0->f2"` 表示Java输入元组的第一个字段会被无更改的复制到Java输出元组的第三个字段。
通配符表达式`*`可以用来指整个输入或输出类型，即`"f0->*"`表示函数的输出总是等于其Java输入元组的第一个字段。

多个转发字段可以在单个字符串中声明，方法是用分号分隔为 `"f0; f2->f1; f3->f2"`或单独的字符串`"f0", "f2->f1", "f3->f2"`。
 指定转发字段时，不需要声明所有转发字段，但所有声明都必须正确。

转发的字段信息可以通过在函数类定义上附加Java注解或在调用DataSet上的函数后将其作为 operator 参数传递来声明，如下所示。

##### 函数类注解

* `@ForwardedFields` 用于单个输入的函数，如 Map 和 Reduce。
* `@ForwardedFieldsFirst` 用于具有两个输入的函数的第一个输入 如 Join 和 CoGroup。
* `@ForwardedFieldsSecond` 用于具有两个输入的函数的第二个输入 如 Join 和 CoGroup。

##### Operator 参数

* `data.map(myMapFnc).withForwardedFields()` 用于单个输入的函数，如 Map 和 Reduce。
* `data1.join(data2).where().equalTo().with(myJoinFnc).withForwardFieldsFirst()` 用于具有两个输入的函数的第一个输入 如 Join 和 CoGroup。
* `data1.join(data2).where().equalTo().with(myJoinFnc).withForwardFieldsSecond()` 用于具有两个输入的函数的第二个输入 如 Join 和 CoGroup。

请注意，无法通过 operator 参数覆盖类注解的字段转发信息。

##### 示例

以下示例显示如何使用函数类注解声明转发的字段信息：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@ForwardedFields("f0->f2")
public class MyMap implements
              MapFunction<Tuple2<Integer, Integer>, Tuple3<String, Integer, Integer>> {
  @Override
  public Tuple3<String, Integer, Integer> map(Tuple2<Integer, Integer> val) {
    return new Tuple3<String, Integer, Integer>("foo", val.f1 / 2, val.f0);
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@ForwardedFields("_1->_3")
class MyMap extends MapFunction[(Int, Int), (String, Int, Int)]{
   def map(value: (Int, Int)): (String, Int, Int) = {
    return ("foo", value._2 / 2, value._1)
  }
}
{% endhighlight %}

</div>
</div>

#### 非转发字段


非转发字段信息声明了不保存在函数输出中相同位置的所有字段。那么其他字段的值被认为保存在输出中的相同位置。因此，非转发字段信息与转发字段信息相反。
组合运算符（如GroupReduce，GroupCombine，CoGroup和MapPartition）的未转发字段信息必须满足与转发字段信息相同的要求。

**IMPORTANT**: 非转发字段信息的规范是可选的。但是，一旦使用，
**ALL!** 必须指定非转发字段，因为所有其他字段都被认为是就地转发的。将转发字段声明为未转发是安全的。

未转发的字段被指定为[字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)的列表。
该列表可以作为单个字符串提供(字段表达式用分号分隔)或作为多个字符串。例如，“f1; f3”和“f1”,“f3”都声明Java元组的第二个和第四个字段没有保留在原位，
而其他字段都保留在原位。只能为具有相同输入和输出类型的函数指定非转发字段信息。

使用以下注解将未转发字段信息指定为函数类注解：

* `@NonForwardedFields` 用于单个输入的函数，如 Map 和 Reduce。
* `@NonForwardedFieldsFirst` 用于具有两个输入的函数的第一个输入 如 Join 和 CoGroup。
* `@NonForwardedFieldsSecond` 用于具有两个输入的函数的第二个输入 如 Join 和 CoGroup。

##### 示例

以下示例显示如何声明未转发的字段信息：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@NonForwardedFields("f1") // second field is not forwarded
public class MyMap implements
              MapFunction<Tuple2<Integer, Integer>, Tuple2<Integer, Integer>> {
  @Override
  public Tuple2<Integer, Integer> map(Tuple2<Integer, Integer> val) {
    return new Tuple2<Integer, Integer>(val.f0, val.f1 / 2);
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@NonForwardedFields("_2") // second field is not forwarded
class MyMap extends MapFunction[(Int, Int), (Int, Int)]{
  def map(value: (Int, Int)): (Int, Int) = {
    return (value._1, value._2 / 2)
  }
}
{% endhighlight %}

</div>
</div>

#### 读取字段

读取字段信息声明由函数访问和评估的所有字段，即由函数用来计算其结果的所有字段。
例如，在指定读取字段信息时，在条件语句中计算或用于计算的字段必须标记为read。
只有未经修改转发给输出而未评估其值或不能被访问的字段不会被读取。

**IMPORTANT**: 读取字段信息的指定是可选的。但是，一旦使用，
**ALL!** 读取字段必须被指。将非读取字段声明为已读是安全的。

读字段被指定为 [字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)的列表。
该列表可以作为单个字符串提供(字段表达式用分号分隔)或作为多个字符串。
例如，“f1; f3”和“f1”,“f3”声明Java 元祖的第二个和第四个字段被读取和评估。

使用以下注解将读取字段信息指定为函数类注解：

* `@ReadFields` 用于单个输入的函数，如 Map 和 Reduce。
* `@ReadFieldsFirst` 用于具有两个输入的函数的第一个输入 如 Join 和 CoGroup。
* `@ReadFieldsSecond` 用于具有两个输入的函数的第二个输入 如 Join 和 CoGroup。

##### 示例

以下示例显示如何声明读取字段信息：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@ReadFields("f0; f3") // f0 and f3 are read and evaluated by the function.
public class MyMap implements
              MapFunction<Tuple4<Integer, Integer, Integer, Integer>,
                          Tuple2<Integer, Integer>> {
  @Override
  public Tuple2<Integer, Integer> map(Tuple4<Integer, Integer, Integer, Integer> val) {
    if(val.f0 == 42) {
      return new Tuple2<Integer, Integer>(val.f0, val.f1);
    } else {
      return new Tuple2<Integer, Integer>(val.f3+10, val.f1);
    }
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@ReadFields("_1; _4") // _1 and _4 are read and evaluated by the function.
class MyMap extends MapFunction[(Int, Int, Int, Int), (Int, Int)]{
   def map(value: (Int, Int, Int, Int)): (Int, Int) = {
    if (value._1 == 42) {
      return (value._1, value._2)
    } else {
      return (value._4 + 10, value._2)
    }
  }
}
{% endhighlight %}

</div>
</div>

{% top %}


广播变量
-------------------

除了作为operator 的 常规输入外，广播变量还允许您为一个operator所有并行实例创建一个可靠的数据集。
这对辅助数据集或数据相关参数化非常有用。数据集将作为集合在operator处被访问。

- **Broadcast**: 广播集是使用名称通过withBroadcastSet(DataSet,String)注册
- **Access**: 广播变量在目标 operator中通过 `getRuntimeContext().getBroadcastVariable(String)`访问。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// 1. The DataSet to be broadcast
DataSet<Integer> toBroadcast = env.fromElements(1, 2, 3);

DataSet<String> data = env.fromElements("a", "b");

data.map(new RichMapFunction<String, String>() {
    @Override
    public void open(Configuration parameters) throws Exception {
      // 3. Access the broadcast DataSet as a Collection
      Collection<Integer> broadcastSet = getRuntimeContext().getBroadcastVariable("broadcastSetName");
    }


    @Override
    public String map(String value) throws Exception {
        ...
    }
}).withBroadcastSet(toBroadcast, "broadcastSetName"); // 2. Broadcast the DataSet
{% endhighlight %}

确保在注册和访问广播数据集时，名称（前一示例中的broadcastSetName）匹配。
有关完整的示例程序，请查看{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java "K-Means算法" %}。
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
// 1. The DataSet to be broadcast
val toBroadcast = env.fromElements(1, 2, 3)

val data = env.fromElements("a", "b")

data.map(new RichMapFunction[String, String]() {
    var broadcastSet: Traversable[String] = null

    override def open(config: Configuration): Unit = {
      // 3. Access the broadcast DataSet as a Collection
      broadcastSet = getRuntimeContext().getBroadcastVariable[String]("broadcastSetName").asScala
    }

    def map(in: String): String = {
        ...
    }
}).withBroadcastSet(toBroadcast, "broadcastSetName") // 2. Broadcast the DataSet
{% endhighlight %}

确保在注册和访问广播数据集时，名称（前一示例中的broadcastSetName）匹配。
有关完整的示例程序，请查看{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java "K-Means算法" %}。
</div>
</div>

**Note**: 由于广播变量的内容保存在每个节点的内存中，因此它不应该太大。
对于标量值等简单的事情，您可以简单地将参数作为函数闭包的一部分，或使用withParameters(...)方法传入配置。

{% top %}

分布式缓存
-------------------

Flink提供了一个类似于Apache Hadoop的分布式缓存，使用户函数的并行实例可以在本地访问文件。该功能可用于共享包含静态外部数据的文件，例如字典或机器学习回归模型。

缓存工作如下。程序在ExecutionEnvironment中以特定名称将[本地或远程文件系统（如HDFS或S3）]({{ site.baseurl }}/dev/batch/connectors.html#reading-from-file-systems)的文件或目录注册为缓存文件。当程序执行时，Flink自动将文件或目录复制到所有 worker 节点的本地文件系统。 用户函数可以在指定名称下查找文件或目录，并可从 worker 节点的本地文件系统访问它。

分布式缓存使用如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

在ExecutionEnvironment中注册文件或目录。

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// register a file from HDFS
env.registerCachedFile("hdfs:///path/to/your/file", "hdfsFile")

// register a local executable file (script, executable, ...)
env.registerCachedFile("file:///path/to/exec/file", "localExecFile", true)

// define your program and execute
...
DataSet<String> input = ...
DataSet<Integer> result = input.map(new MyMapper());
...
env.execute();
{% endhighlight %}

访问用户函数中的缓存文件或目录（这里是一个MapFunction）。该函数必须扩展一个[RichFunction]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)类，因为它需要访问RuntimeContext。

{% highlight java %}

// extend a RichFunction to have access to the RuntimeContext
public final class MyMapper extends RichMapFunction<String, Integer> {

    @Override
    public void open(Configuration config) {

      // access cached file via RuntimeContext and DistributedCache
      File myFile = getRuntimeContext().getDistributedCache().getFile("hdfsFile");
      // read the file (or navigate the directory)
      ...
    }

    @Override
    public Integer map(String value) throws Exception {
      // use content of cached file
      ...
    }
}
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

在ExecutionEnvironment中注册文件或目录。

{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

// register a file from HDFS
env.registerCachedFile("hdfs:///path/to/your/file", "hdfsFile")

// register a local executable file (script, executable, ...)
env.registerCachedFile("file:///path/to/exec/file", "localExecFile", true)

// define your program and execute
...
val input: DataSet[String] = ...
val result: DataSet[Integer] = input.map(new MyMapper())
...
env.execute()
{% endhighlight %}

访问用户函数中的缓存文件或目录（这里是一个MapFunction）。该函数必须扩展一个[RichFunction]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)类，因为它需要访问RuntimeContext。

{% highlight scala %}

// extend a RichFunction to have access to the RuntimeContext
class MyMapper extends RichMapFunction[String, Int] {

  override def open(config: Configuration): Unit = {

    // access cached file via RuntimeContext and DistributedCache
    val myFile: File = getRuntimeContext.getDistributedCache.getFile("hdfsFile")
    // read the file (or navigate the directory)
    ...
  }

  override def map(value: String): Int = {
    // use content of cached file
    ...
  }
}
{% endhighlight %}

</div>
</div>

{% top %}

将参数传递给函数
-------------------

可以使用构造函数或withParameters(Configuration)方法将参数传递给函数。参数被序列化为函数对象的一部分，并发送到所有并行任务实例。

请查看[如何将命令行参数传递给函数的最佳实践指南]({{ site.baseurl }}/dev/best_practices.html#parsing-command-line-arguments-and-passing-them-around-in-your-flink-application)。

#### 通过构造函数

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataSet<Integer> toFilter = env.fromElements(1, 2, 3);

toFilter.filter(new MyFilter(2));

private static class MyFilter implements FilterFunction<Integer> {

  private final int limit;

  public MyFilter(int limit) {
    this.limit = limit;
  }

  @Override
  public boolean filter(Integer value) throws Exception {
    return value > limit;
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val toFilter = env.fromElements(1, 2, 3)

toFilter.filter(new MyFilter(2))

class MyFilter(limit: Int) extends FilterFunction[Int] {
  override def filter(value: Int): Boolean = {
    value > limit
  }
}
{% endhighlight %}
</div>
</div>

####  通过`withParameters(Configuration)`

这个方法接受一个Configuration对象作为参数，它将被传递给RichFunction的open（）方法。Configuration 对象是从字符串键到不同值类型的映射。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataSet<Integer> toFilter = env.fromElements(1, 2, 3);

Configuration config = new Configuration();
config.setInteger("limit", 2);

toFilter.filter(new RichFilterFunction<Integer>() {
    private int limit;

    @Override
    public void open(Configuration parameters) throws Exception {
      limit = parameters.getInteger("limit", 0);
    }

    @Override
    public boolean filter(Integer value) throws Exception {
      return value > limit;
    }
}).withParameters(config);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val toFilter = env.fromElements(1, 2, 3)

val c = new Configuration()
c.setInteger("limit", 2)

toFilter.filter(new RichFilterFunction[Int]() {
    var limit = 0

    override def open(config: Configuration): Unit = {
      limit = config.getInteger("limit", 0)
    }

    def filter(in: Int): Boolean = {
        in > limit
    }
}).withParameters(c)
{% endhighlight %}
</div>
</div>

#### 通过全局的`ExecutionConfig`

Flink还允许将自定义的配置值传递到environment的ExecutionConfig接口。由于可以在所有（rich）用户函数中访问配置，因此自定义配置将在所有函数中全局可用。


**设置自定义全局配置**

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

Configuration conf = new Configuration();
conf.setString("mykey","myvalue");
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(conf);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment
val conf = new Configuration()
conf.setString("mykey", "myvalue")
env.getConfig.setGlobalJobParameters(conf)
{% endhighlight %}
</div>
</div>

请注意，您也可以自定义一个类(继承ExecutionConfig.GlobalJobParameters类)作为全局作业参数传递到执行配置。该接口允许实现 Map<String, String> toMap() 方法，在Web前端该方法将依次显示中配置的值。


**从全局配置访问值**

全局作业参数中的对象可以在系统中的许多地方访问。所有实现“RichFunction”接口的用户函数都可以通过运行时上下文进行访问。

{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    private String mykey;
    @Override
    public void open(Configuration parameters) throws Exception {
      super.open(parameters);
      ExecutionConfig.GlobalJobParameters globalParams = getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
      Configuration globConf = (Configuration) globalParams;
      mykey = globConf.getString("mykey", null);
    }
    // ... more here ...
{% endhighlight %}

{% top %}

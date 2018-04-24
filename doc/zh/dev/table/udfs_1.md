---
title: "User-defined Functions"
nav-parent_id: tableapi
nav-pos: 50
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

因为用户自定义函数扩展了查询的表达能力，所以是一个重要功能。

* This will be replaced by the TOC
{:toc}

注册用户自定义函数
-------------------------------
大多数情况下，一个用户自定义函数必须要先注册，才能用于查询，但对于Scala `Table API`，不需要注册函数。

在`TableEnvironment`下调用`registerFunction()`方法注册函数。当用户自定义函数被注册后，它会被添加到`TableEnvironment`的函数目录，这样`Table API`或`SQL`解析器可以对它进行识别并正确翻译。

下面示例会展示如何注册及如何调用每个类型的用户自定义函数（`ScalarFunction`, `TableFunction`, and `AggregateFunction`）。


{% top %}

Scalar函数
----------------

如果在内置函数中没有所需的scalar函数，则可以为Table API 和 SQL自定义scalar函数。用户自定义scalar函数可将0,1或多个scalar值映射为一个新的scalar值。

为了定义scalar函数，需要扩展`org.apache.flink.table.functions`中的`ScalarFunction`基础类和应用多个评价方法。scalar函数的行为取决于评估方法，且该评估方法需要声明为公开并命名为`eval`。评估方法的参数类型及返回类型也决定了scalar函数的参数类型及返回类型。你也可以通过应用名为`eval`的多种方法来加载评估方法。评估方法支持各种可变参数，例如`eval（String. . . strs）`。

以下示例展示如何定义你的hash code 函数，在TableEnvironment下进行注册，并在查询中调用。注意你可以在scalar函数注册前，通过constructor配置它：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class HashCode extends ScalarFunction {
  private int factor = 12;

  public HashCode(int factor) {
      this.factor = factor;
  }

  public int eval(String s) {
      return s.hashCode() * factor;
  }
}

BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register the function
tableEnv.registerFunction("hashCode", new HashCode(10));

// use the function in Java Table API
myTable.select("string, string.hashCode(), hashCode(string)");

// use the function in SQL API
tableEnv.sqlQuery("SELECT string, HASHCODE(string) FROM MyTable");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// must be defined in static/object context
class HashCode(factor: Int) extends ScalarFunction {
  def eval(s: String): Int = {
    s.hashCode() * factor
  }
}

val tableEnv = TableEnvironment.getTableEnvironment(env)

// use the function in Scala Table API
val hashCode = new HashCode(10)
myTable.select('string, hashCode('string))

// register and use the function in SQL
tableEnv.registerFunction("hashCode", new HashCode(10))
tableEnv.sqlQuery("SELECT string, HASHCODE(string) FROM MyTable")
{% endhighlight %}
</div>
</div>

默认情况下，评估方法的结果类型取决于Flink的类型提取工具，这对于基础类型或简单的POJOs是足够的，但对于更复杂，自定义或符合类的，就会出错。在这种情况下，可通过重写`ScalarFunction#getResultType()`，并手动定义`TypeInformation`的结果类型。

以下示例展示一个采用内部时间戳表达形式并将其作为长整型值返回的高级示例。通过重写`ScalarFunction#getResultType()`，我们利用代码生成器定义返回的长整型值应被解释为Types.TIMESTAMP。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public static class TimestampModifier extends ScalarFunction {
  public long eval(long t) {
    return t % 1000;
  }

  public TypeInformation<?> getResultType(signature: Class<?>[]) {
    return Types.TIMESTAMP;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
object TimestampModifier extends ScalarFunction {
  def eval(t: Long): Long = {
    t % 1000
  }

  override def getResultType(signature: Array[Class[_]]): TypeInformation[_] = {
    Types.TIMESTAMP
  }
}
{% endhighlight %}
</div>
</div>

{% top %}

Table函数
---------------

与用户自定义scalar函数类似，用户自定义table函数将0,1或多个scalar数值作为输入参数，但不同于scalar函数返回单个值，它以返回任意数量的行作为输出，返回行可以包括一个或多个列。

为了定义table函数，需要扩展`org.apache.flink.table.functions`里的`TableFunction`基本类，并采用一个或多个评估方法。table函数的行为取决于评估方法，且该评估方法需要声明为公开并命名为`eval`。你也可以通过应用名为eval的多种方法来加载`TableFunction`。评估函数的参数类型决定了所有table函数的有效参数。评估方法也支持例如`eval（String. . . strs）`的可变参数。返回表的类型取决于`TableFunction`泛型类型。评估方法利用受保护的`collect(T)`方法发射输出行。

在Table API，对于Scala用户，表函数伴随 `.join(Expression)`或 `.leftOuterJoin(Expression)`使用，对于Java用户，表函数伴随 `.join(String)`或` .leftOuterJoin(String)`使用。`join operator`连接`outer table`的每一行（operator左侧的table）与由table-valued函数生成的所有行（operator右侧的table）。`leftOuterJoin operator`连接outer table的每一行与由table-valued函数生成的所有行，并保护表函数返回一个空表的outer rows。在SQL，利用`LATERAL TABLE(<TableFunction>) `和`CROSS JOIN `及伴随有一个`ON TRUE`的 `LEFT JOIN`，来join condition。

以下示例展示如何定义table-valued函数，在`TableEnvironment`下注册，并在查询中调用。注意在table函数被注册前，你可以通过构造器配置它： 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// The generic type "Tuple2<String, Integer>" determines the schema of the returned table as (String, Integer).
public class Split extends TableFunction<Tuple2<String, Integer>> {
    private String separator = " ";
    
    public Split(String separator) {
        this.separator = separator;
    }
    
    public void eval(String str) {
        for (String s : str.split(separator)) {
            // use collect(...) to emit a row
            collect(new Tuple2<String, Integer>(s, s.length()));
        }
    }
}

BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);
Table myTable = ...         // table schema: [a: String]

// Register the function.
tableEnv.registerFunction("split", new Split("#"));

// Use the table function in the Java Table API. "as" specifies the field names of the table.
myTable.join("split(a) as (word, length)").select("a, word, length");
myTable.leftOuterJoin("split(a) as (word, length)").select("a, word, length");

// Use the table function in SQL with LATERAL and TABLE keywords.
// CROSS JOIN a table function (equivalent to "join" in Table API).
tableEnv.sqlQuery("SELECT a, word, length FROM MyTable, LATERAL TABLE(split(a)) as T(word, length)");
// LEFT JOIN a table function (equivalent to "leftOuterJoin" in Table API).
tableEnv.sqlQuery("SELECT a, word, length FROM MyTable LEFT JOIN LATERAL TABLE(split(a)) as T(word, length) ON TRUE");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// The generic type "(String, Int)" determines the schema of the returned table as (String, Integer).
class Split(separator: String) extends TableFunction[(String, Int)] {
  def eval(str: String): Unit = {
    // use collect(...) to emit a row.
    str.split(separator).foreach(x -> collect((x, x.length))
  }
}

val tableEnv = TableEnvironment.getTableEnvironment(env)
val myTable = ...         // table schema: [a: String]

// Use the table function in the Scala Table API (Note: No registration required in Scala Table API).
val split = new Split("#")
// "as" specifies the field names of the generated table.
myTable.join(split('a) as ('word, 'length)).select('a, 'word, 'length)
myTable.leftOuterJoin(split('a) as ('word, 'length)).select('a, 'word, 'length)

// Register the table function to use it in SQL queries.
tableEnv.registerFunction("split", new Split("#"))

// Use the table function in SQL with LATERAL and TABLE keywords.
// CROSS JOIN a table function (equivalent to "join" in Table API)
tableEnv.sqlQuery("SELECT a, word, length FROM MyTable, LATERAL TABLE(split(a)) as T(word, length)")
// LEFT JOIN a table function (equivalent to "leftOuterJoin" in Table API)
tableEnv.sqlQuery("SELECT a, word, length FROM MyTable LEFT JOIN TABLE(split(a)) as T(word, length) ON TRUE")
{% endhighlight %}
**IMPORTANT:** Do not implement TableFunction as a Scala object. Scala object is a singleton and will cause concurrency issues.
</div>
</div>

注意POJO类没有确定的字段顺序，因此，你不能利用表函数的`AS`重命名POJO返回字段。

默认情况下，`TableFunction`的结果类型取决于Flink自动类型提取工具，这对于基础类型和简单的POJOs有用，但对于更复杂，自定义或符合类型的会出错。在这种情况下，可以通过重写返回`TypeInformation`的`TableFunction#getResultType()`函数，并手动指定结果类型。

以下展示了一个返回要求显示类型信息的`Row`类型的`TableFunction`示例。我们通过重写`TableFunction#getResultType()`，定义返回的表类型为`RowTypeInfo(String, Integer)`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class CustomTypeSplit extends TableFunction<Row> {
    public void eval(String str) {
        for (String s : str.split(" ")) {
            Row row = new Row(2);
            row.setField(0, s);
            row.setField(1, s.length);
            collect(row);
        }
    }

    @Override
    public TypeInformation<Row> getResultType() {
        return Types.ROW(Types.STRING(), Types.INT());
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CustomTypeSplit extends TableFunction[Row] {
  def eval(str: String): Unit = {
    str.split(" ").foreach({ s =>
      val row = new Row(2)
      row.setField(0, s)
      row.setField(1, s.length)
      collect(row)
    })
  }

  override def getResultType: TypeInformation[Row] = {
    Types.ROW(Types.STRING, Types.INT)
  }
}
{% endhighlight %}
</div>
</div>

{% top %}


聚合函数
---------------------

User-Defined Aggregate Functions (UDAGGs)聚合一个表（一行或多行，且有一个或多个属性）为一个scalar值。

<center>
<img alt="UDAGG mechanism" src="{{ site.baseurl }}/fig/udagg-mechanism.png" width="80%">
</center>

上图展示了一个聚合示例。假设你有一个包含饮料数据的表，其中，表包含3列，为`id`，`name`和`price`，及5行。如果你需要找到表中价格最高的饮料，即执行`max()`聚合。你需要检查5行的每一行，结果输出一个数字值。

用户自定义aggregation函数通过扩展`AggregateFunction`类来实现。一个`AggregateFunction`工作方式如下。首先，需要一个`accumulator`数据结构来保存聚合的中间结果。一个空的accumulator可调用`AggregateFunction`的`createAccumulator()`方法创建。其次，调用函数的`accumulate()`方法，为每个输入行更新其accumulator。一旦所有行被处理完，就会调用函数的`getValue()`方法计算并返回最终结果。

**以下方法对每个AggregateFunction都是强制性的：**

- `createAccumulator()`
- `accumulate()`
- `getValue()`

Flink的类型提取工具可能无法识别复杂的数据类型，例如，如果不是基础类型或简单的POJOs。类似于`ScalarFunction`和`TableFunction`，`AggregateFunction`提供指定结果类型的`TypeInformation(AggregateFunction#getResultType())`和accumulator类型(`AggregateFunction#getAccumulatorType()`)的方法。

除了上述方法，也有一些可选应用的contracted方法。尽管一些方法允许系统更高效的执行查询，但在某些特定用例，一些方法是强制性的。例如，如果aggregation函数被用于会话组窗口的上下文中（当观察到要“连接”他们的行时，两个会话窗口的累积器需要被连接），`merge()`方法是强制要用的。

**根据用例，需要以下的AggregateFunction方法：**

- `retract()`：对于有界OVER窗口上的聚合是必需的
- `merge()`：在很多批式聚合和会话窗口的聚合是必需的
- `resetAccumulator()`：在很多批式聚合中是必需的

所有的`AggregateFunction`方法必须声明为public，而不是static，且按上面提及的名字命名。`createAccumulator`，`getValue`，`getResultType`和`getAccumulatorType`方法定义为`AggregateFunction`的抽象类，其他的为contracted方法。为了定义聚合函数，需要扩展`org.apache.flink.table.functions.AggregateFunction`的基本类，并应用一个或多个accumulate方法。accumulate方法可用不同参数类型加载，并支持可变参数。

以下为`AggregateFunction`所有方法的详细文档： 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
/**
  * Base class for aggregation functions. 
    *
  * @param <T>   the type of the aggregation result
  * @param <ACC> the type of the aggregation accumulator. The accumulator is used to keep the
  *             aggregated values which are needed to compute an aggregation result.
  *             AggregateFunction represents its state using accumulator, thereby the state of the
  *             AggregateFunction must be put into the accumulator.
    */
public abstract class AggregateFunction<T, ACC> extends UserDefinedFunction {

  /**
    * Creates and init the Accumulator for this [[AggregateFunction]].
    *
    * @return the accumulator with the initial value
    */
  public ACC createAccumulator(); // MANDATORY

  /** Processes the input values and update the provided accumulator instance. The method
    * accumulate can be overloaded with different custom types and arguments. An AggregateFunction
    * requires at least one accumulate() method.
    *
    * @param accumulator           the accumulator which contains the current aggregated results
    * @param [user defined inputs] the input value (usually obtained from a new arrived data).
    */
  public void accumulate(ACC accumulator, [user defined inputs]); // MANDATORY

  /**
    * Retracts the input values from the accumulator instance. The current design assumes the
    * inputs are the values that have been previously accumulated. The method retract can be
    * overloaded with different custom types and arguments. This function must be implemented for
    * datastream bounded over aggregate.
    *
    * @param accumulator           the accumulator which contains the current aggregated results
    * @param [user defined inputs] the input value (usually obtained from a new arrived data).
    */
  public void retract(ACC accumulator, [user defined inputs]); // OPTIONAL

  /**
    * Merges a group of accumulator instances into one accumulator instance. This function must be
    * implemented for datastream session window grouping aggregate and dataset grouping aggregate.
    *
    * @param accumulator  the accumulator which will keep the merged aggregate results. It should
    *                     be noted that the accumulator may contain the previous aggregated
    *                     results. Therefore user should not replace or clean this instance in the
    *                     custom merge method.
    * @param its          an [[java.lang.Iterable]] pointed to a group of accumulators that will be
    *                     merged.
    */
  public void merge(ACC accumulator, java.lang.Iterable<ACC> its); // OPTIONAL

  /**
    * Called every time when an aggregation result should be materialized.
    * The returned value could be either an early and incomplete result
    * (periodically emitted as data arrive) or the final result of the
    * aggregation.
    *
    * @param accumulator the accumulator which contains the current
    *                    aggregated results
    * @return the aggregation result
    */
  public T getValue(ACC accumulator); // MANDATORY

  /**
    * Resets the accumulator for this [[AggregateFunction]]. This function must be implemented for
    * dataset grouping aggregate.
    *
    * @param accumulator  the accumulator which needs to be reset
    */
  public void resetAccumulator(ACC accumulator); // OPTIONAL

  /**
    * Returns true if this AggregateFunction can only be applied in an OVER window.
    *
    * @return true if the AggregateFunction requires an OVER window, false otherwise.
    */
  public Boolean requiresOver = false; // PRE-DEFINED

  /**
    * Returns the TypeInformation of the AggregateFunction's result.
    *
    * @return The TypeInformation of the AggregateFunction's result or null if the result type
    *         should be automatically inferred.
    */
  public TypeInformation<T> getResultType = null; // PRE-DEFINED

  /**
    * Returns the TypeInformation of the AggregateFunction's accumulator.
    *
    * @return The TypeInformation of the AggregateFunction's accumulator or null if the
    *         accumulator type should be automatically inferred.
    */
  public TypeInformation<T> getAccumulatorType = null; // PRE-DEFINED
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
/**
  * Base class for aggregation functions. 
    *
  * @tparam T   the type of the aggregation result
  * @tparam ACC the type of the aggregation accumulator. The accumulator is used to keep the
  *             aggregated values which are needed to compute an aggregation result.
  *             AggregateFunction represents its state using accumulator, thereby the state of the
  *             AggregateFunction must be put into the accumulator.
    */
abstract class AggregateFunction[T, ACC] extends UserDefinedFunction {
    /**
    * Creates and init the Accumulator for this [[AggregateFunction]].
    *
    * @return the accumulator with the initial value
    */
  def createAccumulator(): ACC // MANDATORY

  /**
    * Processes the input values and update the provided accumulator instance. The method
    * accumulate can be overloaded with different custom types and arguments. An AggregateFunction
    * requires at least one accumulate() method.
    *
    * @param accumulator           the accumulator which contains the current aggregated results
    * @param [user defined inputs] the input value (usually obtained from a new arrived data).
    */
  def accumulate(accumulator: ACC, [user defined inputs]): Unit // MANDATORY

  /**
    * Retracts the input values from the accumulator instance. The current design assumes the
    * inputs are the values that have been previously accumulated. The method retract can be
    * overloaded with different custom types and arguments. This function must be implemented for
    * datastream bounded over aggregate.
    *
    * @param accumulator           the accumulator which contains the current aggregated results
    * @param [user defined inputs] the input value (usually obtained from a new arrived data).
    */
  def retract(accumulator: ACC, [user defined inputs]): Unit // OPTIONAL

  /**
    * Merges a group of accumulator instances into one accumulator instance. This function must be
    * implemented for datastream session window grouping aggregate and dataset grouping aggregate.
    *
    * @param accumulator  the accumulator which will keep the merged aggregate results. It should
    *                     be noted that the accumulator may contain the previous aggregated
    *                     results. Therefore user should not replace or clean this instance in the
    *                     custom merge method.
    * @param its          an [[java.lang.Iterable]] pointed to a group of accumulators that will be
    *                     merged.
    */
  def merge(accumulator: ACC, its: java.lang.Iterable[ACC]): Unit // OPTIONAL

  /**
    * Called every time when an aggregation result should be materialized.
    * The returned value could be either an early and incomplete result
    * (periodically emitted as data arrive) or the final result of the
    * aggregation.
    *
    * @param accumulator the accumulator which contains the current
    *                    aggregated results
    * @return the aggregation result
    */
  def getValue(accumulator: ACC): T // MANDATORY

  h/**
    * Resets the accumulator for this [[AggregateFunction]]. This function must be implemented for
    * dataset grouping aggregate.
    *
    * @param accumulator  the accumulator which needs to be reset
    */
  def resetAccumulator(accumulator: ACC): Unit // OPTIONAL

  /**
    * Returns true if this AggregateFunction can only be applied in an OVER window.
    *
    * @return true if the AggregateFunction requires an OVER window, false otherwise.
    */
  def requiresOver: Boolean = false // PRE-DEFINED

  /**
    * Returns the TypeInformation of the AggregateFunction's result.
    *
    * @return The TypeInformation of the AggregateFunction's result or null if the result type
    *         should be automatically inferred.
    */
  def getResultType: TypeInformation[T] = null // PRE-DEFINED

  /**
    * Returns the TypeInformation of the AggregateFunction's accumulator.
    *
    * @return The TypeInformation of the AggregateFunction's accumulator or null if the
    *         accumulator type should be automatically inferred.
    */
  def getAccumulatorType: TypeInformation[ACC] = null // PRE-DEFINED
}
{% endhighlight %}
</div>
</div>

以下示例展示如何：

- 定义一个`AggregateFunction`，可以计算给定列的加权平均值
- 在`TableEnvironment`下注册函数
- 在查询中应用函数

为了计算加权平均值，accumulator需要存储加权总和及所有已被计算的数据的计数。在示例中，我们定义了一个`WeightedAvgAccum`类作为accumulator。由于Flink的检查点机制，accumulators自动备份，并进行存储以防在失败情况下能保证准确的语义。

我们`WeightedAvg` `AggregateFunction`的`accumulate()`方法有3种输入。第一个是`WeightedAvgAccum` accumulator，其它两个是自定义输入：输入值`ivalue`和输入权重`iweight`。虽对大多数聚合类型来说，`retract()`， `merge()`和`resetAccumulator()`方法不是强制性的，但我们在以下示例会用到。注意由于Flink类型提取对Scala并非都适用，我们在使用Java类型时，定义`getResultType()`，但在Scala下，定义`getAccumulatorType()`方法。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
/**
 * Accumulator for WeightedAvg.
 */
public static class WeightedAvgAccum {
    public long sum = 0;
    public int count = 0;
}

/**
 * Weighted Average user-defined aggregate function.
 */
public static class WeightedAvg extends AggregateFunction<Long, WeightedAvgAccum> {

    @Override
    public WeightedAvgAccum createAccumulator() {
        return new WeightedAvgAccum();
    }

    @Override
    public Long getValue(WeightedAvgAccum acc) {
        if (acc.count == 0) {
            return null;
        } else {
            return acc.sum / acc.count;
        }
    }

    public void accumulate(WeightedAvgAccum acc, long iValue, int iWeight) {
        acc.sum += iValue * iWeight;
        acc.count += iWeight;
    }

    public void retract(WeightedAvgAccum acc, long iValue, int iWeight) {
        acc.sum -= iValue * iWeight;
        acc.count -= iWeight;
    }
    
    public void merge(WeightedAvgAccum acc, Iterable<WeightedAvgAccum> it) {
        Iterator<WeightedAvgAccum> iter = it.iterator();
        while (iter.hasNext()) {
            WeightedAvgAccum a = iter.next();
            acc.count += a.count;
            acc.sum += a.sum;
        }
    }
    
    public void resetAccumulator(WeightedAvgAccum acc) {
        acc.count = 0;
        acc.sum = 0L;
    }
}

// register function
StreamTableEnvironment tEnv = ...
tEnv.registerFunction("wAvg", new WeightedAvg());

// use function
tEnv.sqlQuery("SELECT user, wAvg(points, level) AS avgPoints FROM userScores GROUP BY user");

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import java.lang.{Long => JLong, Integer => JInteger}
import org.apache.flink.api.java.tuple.{Tuple1 => JTuple1}
import org.apache.flink.api.java.typeutils.TupleTypeInfo
import org.apache.flink.table.api.Types
import org.apache.flink.table.functions.AggregateFunction

/**
 * Accumulator for WeightedAvg.
 */
class WeightedAvgAccum extends JTuple1[JLong, JInteger] {
    sum = 0L
    count = 0
}

/**
 * Weighted Average user-defined aggregate function.
 */
class WeightedAvg extends AggregateFunction[JLong, CountAccumulator] {

  override def createAccumulator(): WeightedAvgAccum = {
    new WeightedAvgAccum
  }

  override def getValue(acc: WeightedAvgAccum): JLong = {
    if (acc.count == 0) {
        null
    } else {
        acc.sum / acc.count
    }
  }

  def accumulate(acc: WeightedAvgAccum, iValue: JLong, iWeight: JInteger): Unit = {
    acc.sum += iValue * iWeight
    acc.count += iWeight
  }

  def retract(acc: WeightedAvgAccum, iValue: JLong, iWeight: JInteger): Unit = {
    acc.sum -= iValue * iWeight
    acc.count -= iWeight
  }
    
  def merge(acc: WeightedAvgAccum, it: java.lang.Iterable[WeightedAvgAccum]): Unit = {
    val iter = it.iterator()
    while (iter.hasNext) {
      val a = iter.next()
      acc.count += a.count
      acc.sum += a.sum
    }
  }

  def resetAccumulator(acc: WeightedAvgAccum): Unit = {
    acc.count = 0
    acc.sum = 0L
  }

  override def getAccumulatorType: TypeInformation[WeightedAvgAccum] = {
    new TupleTypeInfo(classOf[WeightedAvgAccum], Types.LONG, Types.INT)
  }

  override def getResultType: TypeInformation[JLong] = Types.LONG
}

// register function
val tEnv: StreamTableEnvironment = ???
tEnv.registerFunction("wAvg", new WeightedAvg())

// use function
tEnv.sqlQuery("SELECT user, wAvg(points, level) AS avgPoints FROM userScores GROUP BY user")

{% endhighlight %}
</div>
</div>


{% top %}

Best Practices for Implementing UDFs
------------------------------------

Table API和SQL内部代码生成是尽可能使用原始值。用户自定义函数可以通过object creation，casting及boxing引入更多开销。因此，强烈建议将参数和结果类型声明为原始类型，而不是他们的boxed类。`Types.DATE`和`Types.TIME`可表示为`int`，`Types.TIMESTAMP`可表示为`long`。

我们建议写自定义函数时，尽量用Java代替Scala，因为Scala类型对Flink类型提取器有要求。

{% top %}

Integrating UDFs with the Runtime
---------------------------------

有时对于自定义函数，在实际工作前，需要获取到全局运行时间信息。用户自定义函数提供`open()`和`close()`方法用于重写和提供与在DataSet或DataStream API中`RichFunction`方法类似的功能。

`open()`方法在评估方法之前被调用一次。`close()`方法在评估方法最后一次调用后被调用。

`open()`方法提供`FunctionContext`，其包含有关用户自定义函数被执行的上下文的信息，例如 the metric group, the distributed cache files, 或 the global job parameters。

通过调用`FunctionContext`的相关方法，可获得以下信息：

| Method                                | Description                                            |
| :------------------------------------ | :----------------------------------------------------- |
| `getMetricGroup()`                    | Metric group for this parallel subtask.                |
| `getCachedFile(name)`                 | Local temporary file copy of a distributed cache file. |
| `getJobParameter(name, defaultValue)` | Global job parameter value associated with given key.  |

以下示例片段展示如何在一个scalar函数中使用`FunctionContext`，以获取全局作业参数：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class HashCode extends ScalarFunction {

    private int factor = 0;
    
    @Override
    public void open(FunctionContext context) throws Exception {
        // access "hashcode_factor" parameter
        // "12" would be the default value if parameter does not exist
        factor = Integer.valueOf(context.getJobParameter("hashcode_factor", "12")); 
    }
    
    public int eval(String s) {
        return s.hashCode() * factor;
    }
}

ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// set job parameter
Configuration conf = new Configuration();
conf.setString("hashcode_factor", "31");
env.getConfig().setGlobalJobParameters(conf);

// register the function
tableEnv.registerFunction("hashCode", new HashCode());

// use the function in Java Table API
myTable.select("string, string.hashCode(), hashCode(string)");

// use the function in SQL
tableEnv.sqlQuery("SELECT string, HASHCODE(string) FROM MyTable");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
object hashCode extends ScalarFunction {

  var hashcode_factor = 12

  override def open(context: FunctionContext): Unit = {
    // access "hashcode_factor" parameter
    // "12" would be the default value if parameter does not exist
    hashcode_factor = context.getJobParameter("hashcode_factor", "12").toInt
  }

  def eval(s: String): Int = {
    s.hashCode() * hashcode_factor
  }
}

val tableEnv = TableEnvironment.getTableEnvironment(env)

// use the function in Scala Table API
myTable.select('string, hashCode('string))

// register and use the function in SQL
tableEnv.registerFunction("hashCode", hashCode)
tableEnv.sqlQuery("SELECT string, HASHCODE(string) FROM MyTable")
{% endhighlight %}

</div>
</div>

{% top %}


---
title: "Working with State"
nav-parent_id: streaming_state
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

本文档解释了如何在开发一个应用时使用Flink的状态。

* ToC
{:toc}

## 键值状态和算子状态
Flink中有两种基本状态：`键值状态`和`算子状态`。

### 键值状态

*键值状态*总是和键有关，并且只能用在`键值流`的函数和算子中。

你可以将键值状态看做分区或分片的算子状态，每一个键值都有唯一的一个状态分区，每一个键值状态在逻辑上都与一个唯一的<并行操作符实例,键值>相关联，并且由于每一个键值有属于唯一的一个并行的带有键值的算子的实例，我们可以将其简单的看做<算子，键值>的形式。

键值状态接着被放到*键组*中进行管理，键组是Flink用来重新分配键值状态的原子单位；键组数量与并行数量相同，在执行过程中每一个带有键值的算子的并行实例对应一个或多个键组中的键值。

### 算子状态

使用*算子状态*(或者成为*非键值状态*)时，每一个算子状态都与一个并行实例绑定。
The [Kafka 连接器]({{ site.baseurl }}/dev/connectors/kafka.html)是Flink中使用算子状态的一个很好的实例。每一个Kafka消费者的并行实例中都保存着一个主题分片和偏移量的map来作为它的算子状态。
算子状态结构支持在并行化发生改变时对并行算子进行重新分配，且有不同的方案来实施这个重分布操作。

## 原生与托管状态
*键值状态*和*算子状态*有两种存在形式：*托管*和*原生*。

*托管状态*在由Flink运行时控制的数据结构中得到体现，例如内部的哈希表，或是RocksDB。例子有"ValueState"，"ListState"等。Flink的运行时会对这些状态进行编码并将它们写入到检查点中。

*原生状态*是算子在其自己的数据结构中存储的状态。当被写成检查点的时候，它们之写入一些字节序列到检查点中，Flink只会看到这些原始字节序列，而对状态的数据结构一无所知。

所有的数据流函数都可以使用托管状态，但是原生状态结构只能用在对算子进行实现时。相比于原生状态，Flink更加推荐使用托管状态，因为Flink可以通过托管状态来实现并行化发生改变时对状态的自动重分布，也能使Flink对内存进行更好的管理。

<span class="label label-danger">注意</span>如果你的托管状态需要定制序列化逻辑，请参考 [相应的指南](custom_serialization.html) 以保证向后兼容。Flink的默认序列化操作不需要再进行任何特殊的配置。

## 使用托管的键值状态
托管的键值状态接口提供对当前输入元素的键值不同类型的状态的访问，这意味着这种类型的状态只能被用在`键值流`上，，可以通过`stream.keyBy(…)`进行创建。
现在，我们将先看一看不同类型的可用状态，然后我们再看他们是如何被引用到一个程序中，有以下这些基本的可用状态：
* `ValueState<T>`：它保存一个可以被更新和获取(上面提到的输入元素的键范围内，每个操作所能看见的键都能对应一个值)的值。这个值可以通过`update(T)`设置，并通过`T value()`获取。
* `ListState<T>`：它保存一个元素列表，你可以在列表中追加元素，并恢复一个包含所有当前易存储元素的`可迭代对象`。使用`add(T)`添加元素，使用`Iterable<T> get()`获取可迭代对象。
* `ReducingState<T>`：它存储一个单独的值，这个值代表所有添加到状态中的值的聚合体。结构与`ListState`相同，但使用`add(T)`添加的元素会使用特定的`ReduceFunction`reduce到一个聚合体中。

* `AggregatingState<IN, OUT>`: 它存储一个单独的值，这个值代表所有添加到状态中的值的聚合体。与 `ReducingState`相反的是，聚合的类型可能与添加到状态中的元素类型不同，其接口与`ListState`相同，但是通过`add(IN)`方法添加的元素会通过特定的`AggregateFunction`进行聚合。
* `FoldingState<T, ACC>`：它存储一个单独的值，这个值代表所有添加到状态中的值的聚合体。与 `ReducingState`相反的是，聚合的类型可能与添加到状态中的元素类型不同，其接口与`ListState`相同，但是通过`add(T)`方法添加的元素会通过特定的`FoldFunction`折叠到一个聚合体中。
* `MapState<UK, UV>`：它存储一个映射列表。你可以将键-值对放入状态中，并能获得一个当前所有已存储映射的可迭代对象。映射关系通过`put(UK, UV)`或`putAll(Map<UK, UV>)`方法添加。与用户键相关联的值可以通过`get(UK)`方法获取。映射关系，键，值的可迭代视图可以通过使用`entries()`, `keys()` 和 `values()`方法分别获取。

所有的状态类型也都有一个共同的方法`clear()` ，它可以清除当前激活键的状态，例如输入元素的键。

<span class="label label-danger">注意</span>`FoldingState`和`FoldingStateDescriptor`已在Flink1.4中启用，并将在以后的版本中被完全移除。请使用`AggregatingState` 和 `AggregatingStateDescriptor` 来代替。

需要记住的是，这些状态对象仅用在与状态连接时。状态不是必须要存储在内部中的，但至少应该存储在磁盘或是别的什么地方。

另一个需要记住的是，你从状态中获取的值取决于输入元素的键。所以你从你的一次用户函数调用中获取的值可能与另一次调用相同函数获得的值不同，当然，前提是所包含的键也不同。

为了对状态进行操作，你需要创建一个`StateDescriptor`，它存储着状态的名称(我们在之后将会看到，你可以创建好几个状态，它们都有着独一无二的名字，你可以通过名字来引用它)，状态保存的值的类型，也可能存储着一个用户指定的函数，例如一个`ReduceFunction`函数。这取决于你想获取什么样的类型，你可以创建`ValueStateDescriptor`，`ListStateDescriptor`，`ReducingStateDescriptor`，`FoldingStateDescriptor`，`MapStateDescriptor`中的任意一个。

我们通过`RuntimeContext`方法获取状态，所以这只能在*rich functions*中实现。请参阅[这里]({{ site.baseurl }}/dev/api_concepts.html#rich-functions) 获取有关*rich functions*的信息，我们马上也会看到一个实例。`RuntimeContext`可以在`RichFunction`中获得，`RuntimeContext`中有一下这些方法来访问状态：

* `ValueState<T> getState(ValueStateDescriptor<T>)`
* `ReducingState<T> getReducingState(ReducingStateDescriptor<T>)`
* `ListState<T> getListState(ListStateDescriptor<T>)`
* `AggregatingState<IN, OUT> getAggregatingState(AggregatingState<IN, OUT>)`
* `FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)`
* `MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)`

这是一个`FlatMapFunction`的示例，它展示了这些部分是如何协调工作的：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;
    
    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {
    
        // 获取状态值
        Tuple2<Long, Long> currentSum = sum.value();
    
        // 更新计数
        currentSum.f0 += 1;
    
        // 添加输入值的第二个字段
        currentSum.f1 += input.f1;
    
        // 更新状态
        sum.update(currentSum);
    
        // 如果计数累计到了2，emit平均值并清除状态
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }
    
    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // 状态名称
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // 类型信息
                        Tuple2.of(0L, 0L)); // 状态缺省值
        sum = getRuntimeContext().getState(descriptor);
    }
}



// 这个可以像这样用在流式程序中 (假设我们有一个StreamExecutionEnvironment环境)
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// 打印出来的将会是(1,4) 和 (1,5)
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CountWindowAverage extends RichFlatMapFunction[(Long, Long), (Long, Long)] {

  private var sum: ValueState[(Long, Long)] = _

  override def flatMap(input: (Long, Long), out: Collector[(Long, Long)]): Unit = {

    // 获取状态值
    val tmpCurrentSum = sum.value
    
    // 如果之前从未被使用，则为null
    val currentSum = if (tmpCurrentSum != null) {
      tmpCurrentSum
    } else {
      (0L, 0L)
    }
    
    // 更新计数
    val newSum = (currentSum._1 + 1, currentSum._2 + input._2)
    
    // 更新状态
    sum.update(newSum)
    
    // 如果计数累积到2，emit平均值并清除状态
    if (newSum._1 >= 2) {
      out.collect((input._1, newSum._2 / newSum._1))
      sum.clear()
    }
  }

  override def open(parameters: Configuration): Unit = {
    sum = getRuntimeContext.getState(
      new ValueStateDescriptor[(Long, Long)]("average", createTypeInformation[(Long, Long)])
    )
  }
}


object ExampleCountWindowAverage extends App {
  val env = StreamExecutionEnvironment.getExecutionEnvironment

  env.fromCollection(List(
    (1L, 3L),
    (1L, 5L),
    (1L, 7L),
    (1L, 4L),
    (1L, 2L)
  )).keyBy(_._1)
    .flatMap(new CountWindowAverage())
    .print()
  // 打印出来的将会是 (1,4) 和 (1,5)

  env.execute("ExampleManagedState")
}
{% endhighlight %}
</div>
</div>

这个实例实现了一个简单的计数窗口。我们把元祖第一项作为键(在例子中所有的键都是1)。我们在方法中的ValueState里面存储了一个计数值和一个不断累计着的总数和。一旦计数值达到2它就会计算平均数并且清除状态以便我们重新从0开始。需要注意的是如果我们的元组的第一项值不同的话，那么对于不同的键，状态值也会不同。

### 在Scala数据流API中使用状态
除了上述的接口以外，Scala API针对在有单一ValueState的KeyedStream上运用有状态的的map() 或者 flatMap()方法还有一个快捷方法。用户定义的函数在一个Option中得到ValueState的当前值，然后 返回一个用来更新状态的已经更新的值。

{% highlight scala %}
val stream: DataStream[(String, Int)] = ...

val counts: DataStream[(String, Int)] = stream
  .keyBy(_._1)
  .mapWithState((in: (String, Int), count: Option[Int]) =>
    count match {
      case Some(c) => ( (in._1, c), Some(c + in._2) )
      case None => ( (in._1, 0), Some(in._2) )
    })
{% endhighlight %}

## 使用托管的算子状态
要想运用托管的算子状态（managed operator state）,我们可以通过实现一个更一般化的CheckpointedFunction 接口或者实现ListCheckpointed<T extends Serializable>接口来达到我们的目的。

#### 检查点函数

通过CheckpointedFunction接口我们可以存取一个无键的拥有重分布模式的状态。它需要实现两个方法：

{% highlight java %}
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
{% endhighlight %}


无论何时要用到检查点，都要调用snapshotState()。而计数部分:initializeState()是在每次用户自定义函数初始化的时候被调用，无论是 函数第一次被初始化还是函数真的从之前的检查点恢复过来而进行的初始化，这个函数都要被调用。由此，initializeState()不仅仅是一个用来初始化不同 类型的状态的地方，它里面还包含着状态恢复逻辑。

现在，Flink已经支持列表形式的托管的算子状态了。这个状态是一个包含一些序列化对象的列表，这些对象彼此相互独立, 因此很适合重新调整分布。换句话说，这些对象是非键值状态用来重分布的最小粒度。根据不同的存取状态的方法，我们可以定义以下的几种分布模式：

  - **均分重分布：**每一个算子返回一个包含状态元素的列表。整个状态(state)是所有列表的一个串联.在恢复/重分布 的时候，这个列表会均匀分布为多个子列表，数量与并行算子的数量相同。每个算子得到一个子列表，它可能为空，或者包含一个或多个元素。举个栗子， 如果并行度为1的时候，这个算子的检查点状态(checkpointed state)含有元素1element1和元素2element2，那么当并行度增加至2时，元素1element1 可能被分给算子0，而元素2element2可能会去算子1。

  - **联合重分布：**每一个算子返回一个包含状态元素的列表。整个状态(state)是所有列表的一个串联。在恢复/重分布的时候， 每个算子得到完整的包含状态元素的列表。

下面是一个有状态的的SinkFunction运用CheckpointedFunction在将元素送出去之前先将它们缓存起来的例子。它表明了基本的均分重分布型列表状态：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction {

    private final int threshold;
    
    private transient ListState<Tuple2<String, Integer>> checkpointedState;
    
    private List<Tuple2<String, Integer>> bufferedElements;
    
    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }
    
    @Override
    public void invoke(Tuple2<String, Integer> value) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // 将其发送到sink中
            }
            bufferedElements.clear();
        }
    }
    
    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }
    
    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));
    
        checkpointedState = context.getOperatorStateStore().getListState(descriptor);
    
        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class BufferingSink(threshold: Int = 0)
  extends SinkFunction[(String, Int)]
    with CheckpointedFunction
    with CheckpointedRestoring[List[(String, Int)]] {

  @transient
  private var checkpointedState: ListState[(String, Int)] = _

  private val bufferedElements = ListBuffer[(String, Int)]()

  override def invoke(value: (String, Int)): Unit = {
    bufferedElements += value
    if (bufferedElements.size == threshold) {
      for (element <- bufferedElements) {
        // 将其发送到sink中
      }
      bufferedElements.clear()
    }
  }

  override def snapshotState(context: FunctionSnapshotContext): Unit = {
    checkpointedState.clear()
    for (element <- bufferedElements) {
      checkpointedState.add(element)
    }
  }

  override def initializeState(context: FunctionInitializationContext): Unit = {
    val descriptor = new ListStateDescriptor[(String, Int)](
      "buffered-elements",
      TypeInformation.of(new TypeHint[(String, Int)]() {})
    )

    checkpointedState = context.getOperatorStateStore.getListState(descriptor)
    
    if(context.isRestored) {
      for(element <- checkpointedState.get()) {
        bufferedElements += element
      }
    }
  }

  override def restoreState(state: List[(String, Int)]): Unit = {
    bufferedElements ++= state
  }
}
{% endhighlight %}
</div>
</div>

initializeState 方法将FunctionInitializationContext作为参数，用来初始化非键值状态的”容器”。 ListState的这个容器用来根据检查点而存储非键值状态对象。

请注意一下状态被初始化的方式，与键值状态类似，也是用一个带有状态名字和状态存储的值的类型的StateDescriptor来初始化的。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val descriptor = new ListStateDescriptor[(String, Long)](
    "buffered-elements",
    TypeInformation.of(new TypeHint[(String, Long)]() {})
)

checkpointedState = context.getOperatorStateStore.getListState(descriptor)

{% endhighlight %}
</div>
</div>

存取状态的方法的命名规则是这样的：“重分布模式”后面加上“状态结构”。例如， 如果在恢复时要用列表状态的联合重分布模式，就用getUnionListState(descriptor)来存取它， 如果名字里面没有包含重分布模式，e.g. getListState(descriptor)，这就意味着您将使用基本的均分重分布模式了。

初始化容器过后，我们用环境中的isRestored()方法来检查我们是否在发生错误后做了恢复，例如如果结果为true，那就说明我们做了恢复，恢复逻辑被应用了。

就像在代码中被修改过的BufferingSink例子展示的那样，这个在状态初始化时做了恢复的ListState被保存在一个类变量中 以便将来供snapshotState()使用。这样以来这个ListState中的所有元素，包括之前的检查点都被会被清除掉，然后用我们想增加的新检查点来填充它。

补充说明一点，键值状态也可以用initializeState()方法来初始化，这个可以用Flink提 供的FunctionInitializationContext来完成。

#### 列表检查点

ListCheckpointed 接口是CheckpointedFunction的一个具有更多约束的变量，它只支持在恢复时 使用列表类型的状态并且是均分重分布模式的情况。它也需要实现以下两个方法：

{% highlight java %}
List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

void restoreState(List<T> state) throws Exception;
{% endhighlight %}

在snapshotState()方法中算子需要返回一个列表给检查点，在restoreState中要处理这个列表。如果状态没有被重新分片，那你总是可以在snapshotState()中返回Collections.singletonList(MY_STATE)。

### 带有状态的源函数

带有状态的源和其他算子相比需要多注意一下。为了更新状态并且保证输出原子化(需要在错误/恢复时 保证且只有一个语义),用户必须从源环境中获得一个锁。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements ListCheckpointed<Long> {

    /**  对exactly once语义的当前偏移 */
    private Long offset;
    
    /** 作业取消的标记 */
    private volatile boolean isRunning = true;
    
    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();
    
        while (isRunning) {
            // 输出和状态更新是原子操作
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }
    
    @Override
    public void cancel() {
        isRunning = false;
    }
    
    @Override
    public List<Long> snapshotState(long checkpointId, long checkpointTimestamp) {
        return Collections.singletonList(offset);
    }
    
    @Override
    public void restoreState(List<Long> state) {
        for (Long s : state)
            offset = s;
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CounterSource
       extends RichParallelSourceFunction[Long]
       with ListCheckpointed[Long] {

  @volatile
  private var isRunning = true

  private var offset = 0L

  override def run(ctx: SourceFunction.SourceContext[Long]): Unit = {
    val lock = ctx.getCheckpointLock

    while (isRunning) {
      // 输出和状态更新是原子操作
      lock.synchronized({
        ctx.collect(offset)
    
        offset += 1
      })
    }
  }

  override def cancel(): Unit = isRunning = false

  override def restoreState(state: util.List[Long]): Unit =
    for (s <- state) {
      offset = s
    }

  override def snapshotState(checkpointId: Long, timestamp: Long): util.List[Long] =
    Collections.singletonList(offset)

}
{% endhighlight %}
</div>
</div>

当Flink知道一个检查点在与外部进行通信时，一些算子可能需要一些信息。在这种情况下您可以查看org.apache.flink.runtime.state.CheckpointListener接口，来获得更多信息。

{% top %}

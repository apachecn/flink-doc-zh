---
title: "Queryable State"
nav-parent_id: streaming_state
nav-pos: 3
is_beta: true
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

* ToC
{:toc}

<div class="alert alert-warning">
  <strong>注:</strong> 可查询状态的客户端API正处于迭代期，所以 <strong>不保证 </strong>接口的 稳定性。即将到来的版本中API有可能会发生重大变化。
</div>
简而言之，此功能允许用户从外部查询Flink托管的分区的状态(参考[使用状态]({{ site.baseurl }}/dev/stream/state/state.html))。在某些场景，可查询状态取消了需要和外部系统进行分布式交互的依赖，例如在实践中经常是瓶颈的键值存储。除此之外，此功能会在调试中十分有用。

<div class="alert alert-warning">
  <strong>注意:</strong> 可查询状态并发访问键值状态（keyed state）比同步访问更有可能阻碍其操作。因为某些状态存储使用的是java的堆空间，<i>例如</i>：<code>内存状态后端</code>， <code>Fs状态后端</code>，所以内存不直接使用复制当检索值,而是引用存储值,可能会导致读-修改-写模式是不安全的， 且并发修改将会导致可查询状态服务失败。对于这些问题<code>RocksDB状态后端</code>是安全的。


## 架构

在展示如何使用可查询状态前，有必要简单描述一下构成它的一些实体，k额查询状态特性有三个主要实体构成：
 1. `QueryableStateClient`，它(可能)运行在Flink集群外，提交用户的查询。
 2. `QueryableStateClientProxy`，它运行在每一个`TaskManager`上(*也就是*在Flink集群内)，负责接收客户端的查询，从它所代表的响应Task Manager中取出请求的状态，并将其返回给客户端。
 3. `QueryableStateServer`，它运行在每一个`TaskManager`上，并负责为本地存储的状态提供服务。

简而言之，客户端会连接到这些代理之一并会为状态发送一个与特定的键相关联的请求，`k`.如同在[使用状态]({{ site.baseurl }}/dev/stream/state/state.html)中声明的那样，键值状态被有组织地存储在*键组*中，并且每一个`TaskManager`都会被分配到好几个键组。为了找出哪个`TaskManager`对存有`k`的检组负责，代理会询问`JobManager`，根据答案，再为与`k`相关联的状态去查询运行在那个 `TaskManager`上的`QueryableStateServer`，并将返回的信息转发给客户端。

## 激活可查询状态

想要在你的Flink集群上启用可查询状态，你只需将`flink-queryable-state-runtime{{ site.scala_version_suffix }}-{{site.version }}.jar`从下载的[Flink](https://flink.apache.org/downloads.html "Apache Flink: Downloads")的`opt/`目录拷贝到`lib/`目录。否则，可查询状态将不会被启动。为了确认你的集群开启了可查询状态，请检查每一个task manager的log信息，查看是否存在`"Started the Queryable State Proxy Server @ ..."`输出。

## 使状态可查询

既然你已经在你的集群上启动了可查询状态，那么是时候看一下如何使用它了。为了让状态外部可见，它需要明确的使用`QueryableStateStream`对象或者调用`stateDescriptor.setQueryable(String queryableStateName)`方法来使其可查询。前者是一个行为与sink相同的方便的对象，它将它读取的值作为可查询状态提供出来，后者使状态描述器代表的键值状态变成可查询的。

下面的章节会解释这两种方式的具体使用：

### 可查询状态流
在`KeyedStream`上调用`.asQueryableState(stateName, stateDescriptor)`方法会返回一个`QueryableStateStream`，它将它的值以可查询状态的方式提供出来。根据状态类型的不同，`asQueryableState()`方法会有下列不同的变量：

{% highlight java %}
// ValueState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ValueStateDescriptor stateDescriptor)

// Shortcut for explicit ValueStateDescriptor variant
QueryableStateStream asQueryableState(String queryableStateName)

// FoldingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    FoldingStateDescriptor stateDescriptor)

// ReducingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ReducingStateDescriptor stateDescriptor)
{% endhighlight %}


<div class="alert alert-info">
  <strong>注:</strong> 没有可查询的 <code>ListState</code> sink，因为它会产生一个持续增长的list，并且有可能不会被清理，因此会最终消耗过多内存。
</div>

返回的`QueryableStateStream`可以被看做一个sink并且后续**无法**转换。在内部，`QueryableStateStream`会被转化成一个使用所有接受记录来更新可查询状态实例的算子。更新逻辑隐含在调用`asQueryableState`提供的`StateDescriptor`的类型中。
在如同下面键值流的所有记录一个程序中，键值流的所有记录都会通过`ValueState.update(value)`被用来更新状态实例：

{% highlight java %}
stream.keyBy(0).asQueryableState("query-name")
{% endhighlight %}

这个行为与Scala API的`flatMapWithState`相似。

### 托管的键值状态

算子的托管键值状态(参考 [Using Managed Keyed State]({{ site.baseurl }}/dev/stream/state/state.html#using-managed-keyed-state))也可以是可查询的，只需通过`StateDescriptor.setQueryable(String queryableStateName)`使适当的状态描述器变成可查询的，如同下面的示例：
{% highlight java %}
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
                "average", // 状态名称
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // 类型信息
                Tuple2.of(0L, 0L)); // 状态缺省值
descriptor.setQueryable("query-name"); // queryable state name
{% endhighlight %}

<div class="alert alert-info">
  <strong>注:</strong> <code>queryableStateName</code>参数可以任意选择，而且仅可在查询中使用。且不需与状态自己的名称相同。

哪种状态类型可以是可查询的不会限制这种变体，这意味着他可以用于任何`ValueState`, `ReduceState`, `ListState`, `MapState`, `AggregatingState`，以及当前被弃用的

## 查询状态

到目前为止，你已经配置好你的集群使用可查询状态了，并且你已经将你的(部分)状态声明为可查询。现在改看看如何查询这个状态了。

你可以使用`QueryableStateClient`帮助类查询状态。这个类在`flink-queryable-state-client`jar包中，你需要在你项目的`pom.xml`文件中将其作为依赖引入：

<div data-lang="java" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-queryable-state-client-java_{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>

获取更多有关信息，你可以查阅如何 [配置一个Flink程序]({{ site.baseurl }}/dev/linking_with_flink.html).

`QueryableStateClient`将你的查询提交到内部代理，内部代理会处理你的查询并将最终结果返回。初始化这个client你只需提供一个有效的`TaskManager` 主机名(记住，每个task manager上都运行着一个可查询状态代理)和监听端口。有关配置代理和状态服务器端口的更多信息请参阅[配置部分](#Configuration).

{% highlight java %}
QueryableStateClient client = new QueryableStateClient(tmHostname, proxyPort);
{% endhighlight %}

当客户端准备完毕，你可以使用下列方法来查询与类型键`k`相关联的类型状态`v`：

{% highlight java %}
CompletableFuture<S> getKvState(
    final JobID jobId,
    final String queryableStateName,
    final K key,
    final TypeInformation<K> keyTypeInfo,
    final StateDescriptor<S, V> stateDescriptor)
{% endhighlight %}

上面的梗罚会返回一个`CompletableFuture`来最终为可查询状态实例保存状态值。实例通过`jobID`标记的作业的`queryableStateName`进行标记。`key`表示你感兴趣的状态的键，`keyTypeInfo`会告诉Flink如何序列化/反序列化它。最终，`stateDescriptor`会包含被请求状态的必要信息，也就是它的类型(`Value`, `Reduce`等)和如何序列化/反序列化它的信息。 

细心的读者可能会发现返回的future包含类型`S`的值，*也就是说*一个`State`对象包含了真正的值。它可以是任何Flink支持的状态类型：`ValueState`, `ReduceState`, `ListState`, `MapState`,
`AggregatingState`, 和当前被弃用的 `FoldingState`. 

<div class="alert alert-info">
  <strong>注:</strong>这些状态对象不允许对内置状态进行修改。可以使用它们来获取状态真正的值，
  These state objects do not allow modifications to the contained state. You can use them to get  <i>例如</i>使用<code>valueState.get()</code>，或者对内置<code><K, V></code实体进行迭代，<i>例如</i>使用<code>mapState.entries()</code>，但是你不可以修改它们。比如，在返回的list上调用<code>add()</code>方法会抛出<code>UnsupportedOperationException</code>。
</div>

<div class="alert alert-info">
  <strong>注:</strong>客户端是一步的，可以被多个线程共享。它需要使用<code>QueryableStateClient.shutdown()</code>来关闭，以在未被使用时释放资源。
</div>

### 实例
下面的案例继承了`CountWindowAverage`，案例 (参考[使用托管的键值状态]({{ site.baseurl }}/dev/stream/state/state.html#using-managed-keyed-state)) 如何使它可查询，且如何查询这个值:

{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    private transient ValueState<Tuple2<Long, Long>> sum; // 包含计数和求和的元祖
    
    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {
        Tuple2<Long, Long> currentSum = sum.value();
        currentSum.f0 += 1;
        currentSum.f1 += input.f1;
        sum.update(currentSum);
    
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
        descriptor.setQueryable("query-name");
        sum = getRuntimeContext().getState(descriptor);
    }
}
{% endhighlight %}

一旦在job中使用，您可以检索作业ID，然后从该算子中查询任何键的当前状态：

{% highlight java %}
QueryableStateClient client = new QueryableStateClient(tmHostname, proxyPort);

// 将要获取的状态描述器
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
          "average",
          TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}),
          Tuple2.of(0L, 0L));

CompletableFuture<ValueState<Tuple2<Long, Long>>> resultFuture =
        client.getKvState(jobId, "query-name", key, BasicTypeInfo.LONG_TYPE_INFO, descriptor);

// 现在处理返回值
resultFuture.thenAccept(response -> {
        try {
            Tuple2<Long, Long> res = response.get();
        } catch (Exception e) {
            e.printStackTrace();
        }
});
{% endhighlight %}

## 配置
以下配置参数会影响可查询状态服务器和客户端的行为。它们定义在`QueryableStateOptions`中。

### 状态服务端
* `query.server.ports`: 可查询状态服务器的端口范围，这个可以避免同一台机器上多个task manager的端口冲突。具体的范围可以是：一个端口："9123"，你可端口范围："50100-50200"，或者一个范围和具体数值混合的列表："50100-50200,50300-50400,51234"。默认端口是9067。
* `query.server.network-threads`: 状态服务器接收进来的请求的网络(时事件循环)线程数(0 => #slots)。
* `query.server.query-threads`: 状态服务器处理进来的请求的线程数(0 => #slots)。


### 代理
* `query.proxy.ports`: 可查询状态代理的服务器端口范围，这个可以避免同一台机器上多个task manager的端口冲突。具体的范围可以是：一个端口："9123"，一个端口范围："50100-50200"，或者一个范围和具体数值混合的列表："50100-50200,50300-50400,51234"。默认端口是9069。

* `query.proxy.network-threads`: 客户端代理接收进来的请求的网络(时事件循环)线程数(0 => #slots)。
* `query.proxy.query-threads`: 客户端代理处理进来的请求的线程数(0 => #slots)。

## 限制
* 可查询状态的生命周期受限于job的生命周期，*例如*，任务启动时注册可查询状态，并在清理时注销它。在将来的版本中，最好是将其解耦，以便在任务完成后允许查询，并通过状态加速恢复通过状态复制。
* 有关KvState的通知可以通过一个简单的说明来实现。 未来这个应该需要改善，以便实现更强大的询问和确认。
* 服务器和客户端跟踪查询的统计信息。 因为它们不会在任何地方暴露出来，所以默认是禁用的。 一旦可以通过度 量系统更好的发布这些数字，我们应该启用统计。


{% top %}

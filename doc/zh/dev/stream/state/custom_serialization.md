---
title: "Custom Serialization for Managed State"
nav-title: "Custom Serialization"
nav-parent_id: streaming_state
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

如果你的应用使用了Flink的托管状态，你可能需要实现自定义的序列化逻辑来来满足特殊用例。
本页是专门为有自定义序列化需求的人编写的参考文档，它涵盖了如何为状态提供一个自定义序列化器，以及如何处理序列化器的兼容性升级。如果你知识简单地使用Flink自带的序列化器，那么你可以选择跳过此文。

### 使用自定义序列化器

如同上面的例子演示的那样，当注册一个托管算子状态或键值状态时，你需要使用`StateDescriptor`来确定状态的名称和有关状态类型的信息。Flink的
[类型信息框架](../../types_serialization.html)将会使用这些类型信息来为状态创建合适的序列化器。

你也可以完全跳过这些步骤，让Flink使用你自己自定义的序列化器来序列化托管状态，只需简单的使用你自己的`TypeSerializer`实现来初始化`StateDescriptor`实例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class CustomTypeSerializer extends TypeSerializer<Tuple2<String, Integer>> {...};

ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "state-name",
        new CustomTypeSerializer());

checkpointedState = getRuntimeContext().getListState(descriptor);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CustomTypeSerializer extends TypeSerializer[(String, Integer)] {...}

val descriptor = new ListStateDescriptor[(String, Integer)](
    "state-name",
    new CustomTypeSerializer)
)

checkpointedState = getRuntimeContext.getListState(descriptor)
{% endhighlight %}
</div>
</div>

需要注意的是，Flink在写入状态序列化器时，也会将状态作为元数据一并写入。在某些恢复状态(参考后面部分中的信息)的情况中，写入的序列化器需要先被反序列化再使用。因此，我们建议你避免使用匿名类作为你的状态序列化器。匿名类生成的类名是不确定的，他可能在不同的编译器中发生改变，而且还取决于它在封闭类中实例化的顺序，这很容易导致以前写入的序列化器没法被读取(因为原始类无法再在classpath中找到)

### 处理序列化器的升级与兼容性

Flink允许改变用于读取和写入托管状态的序列化器，所以用户不会被限制一直只能使用一个特定的序列化操作。当状态被恢复时，为这个状态注册的新序列化器(例如`StateDescriptor`产生的用于读取恢复的作业中的状态的序列器)会被进行兼容性方面的检查，并会为状态替换掉旧的序列化器。

一个具有兼容性的序列化器意味着它能够读取以前版本对状态序列化产生的字节，并且写入的二进制格式也应该保持一致。`TypeSerializer`接口提供了下面两个方法来检查新版本的序列化器的兼容性：

{% highlight java %}
public abstract TypeSerializerConfigSnapshot snapshotConfiguration();
public abstract CompatibilityResult ensureCompatibility(TypeSerializerConfigSnapshot configSnapshot);
{% endhighlight %}

简单来说，每次使用检查点时，`snapshotConfiguration`方法就会被调用，它会创建一个point-in-time状态序列化器配置的视图。返回的配置快照和检查点一起作为状态的元数据进行存储。当检查点被用于恢复作业时，那个序列化器配置快照就会通过对口方法`ensureCompatibility`被提供给相同状态的新的序列化器，这一列的操作都是为了确保新序列化器的兼容性。无论新的序列化器是否是兼容的，这个方法都会当作检查被执行，当出现不兼容的情况时，这个方法也会提供最新序列化器进行重新配置的可能。

需要注意的是，Flink自带的序列化器就是这样实现的，这保证了它们至少与他们本上是兼容的，例如当我们在恢复作业的状态上使用同样的序列化器，序列化器将会对自己进行重新配置以对以前版本的配置进行兼容。

接下来的部分中举例说明了如何在使用自定义序列化器时实现这两个方法。

#### 实现 `snapshotConfiguration` 方法
序列化器的配置快照应该捕捉足够的信息，这样在恢复状态时，为状态的新序列化器传递的信息才足够判定其是否是兼容的。它可能会包含一些关于序列化数据的序列化器参数或二进制格式的代表性信息；这些代表性信息通常是可以决定新的序列化器是否可用于读取以前版本的序列化字节以及是否以相同的二进制格式写入数据相关的任何信息。

你可以对序列化的状态快照从检查点的写入和读取过程进行完全的自定义。所有的序列化器配置快照都要事先这个基础类：`TypeSerializerConfigSnapshot`。

{% highlight java %}
public abstract TypeSerializerConfigSnapshot extends VersionedIOReadableWritable {
  public abstract int getVersion();
  public void read(DataInputView in) {...}
  public void write(DataOutputView out) {...}
}
{% endhighlight %}

`read` 和 `write`方法定义了如何如何读取和写入到检查点中的。最基础的实现包含了读取和写入配置快照版本的逻辑，所以他应该被扩展而不是完全的重写

配置快照的版本可以通过`getVersion`方法获取。序列化器配置快照的版本控制是维护兼容性配置的手段，因为配置中的信息可能会随着时间而改变。默认情况下，配置快照只于当前版本(`getVersion`返回的版本)兼容，如果要明确指出这个配置可以兼容其他版本，你需要重写`getCompatibleVersions`来返回更多的兼容版本。当从检查点中读取时，你可以使用`getReadVersion`方法来指定特定的版本来读取和写入配置。

<span class="label label-danger">注意</span> 序列化器的配置快照版本与升级序列化器**无关**。完全相同的序列化器可以有其配置快照的不同实现，例如当添加更多信息到配置中来允许未来更加便于理解的兼容性检查时。

实现`TypeSerializerConfigSnapshot`的一个限制是必须提供一个无参构造函数。当从检查点中读取配置快照时会用到这个无参构造函数。

#### 实现`ensureCompatibility`方法

`ensureCompatibility`方法应该包含对`TypeSerializerConfigSnapshot`中的历史版本序列化器的信息进行兼容性检查的逻辑代码，该方法一般会完成下列事情之一：

  * 当序列化器在必要的情况下对自身进行配置时检查序列化器是否是兼容的。如果兼容的话便通知Flink这个序列器是是兼容的。
  * 当序列化器不兼容时，在Flink使用新的序列化器处理数据之前通知其序列化器是不兼容的，并且需要对状态进行迁移。

我们可以把上诉两种情况翻译成代码，只需从`ensureCompatibility`返回下列两者之一即可：

  * **`CompatibilityResult.compatible()`**：这个会通知Flink新的序列化器是兼容的，或者已经被重新配置到可兼容了，Flink可以放心使用这个新序列化器来处理作业。
  * **`CompatibilityResult.requiresMigration()`**：这个会通知Flink新的序列化器时不兼容的，或者无法被重新配置到可兼容，在这个新的序列化可是被使用之前需要对状态进行迁移。先要使用旧版本的序列化器将序列化的状态字节还原成对象，，然后再用新的序列化器进行序列化操作。
  * **`CompatibilityResult.requiresMigration(TypeDeserializer deserializer)`**：这种通知和`CompatibilityResult.requiresMigration()`有着相同的语义，但是发生历史版本序列化器无法被找到或加载，导致无法完成对序列化数据进行反序列化迁移操作的情况时，它会提供一个`TypeDeserializer`当作替补方案。

<span class="label label-danger">注意</span>在当前的Flink版本(1.3)中，当兼容性检查的结果时需要对状态进行迁移，任务会因为状态迁移功能在当前版本(1.3)中还未加入导致其无法从检查点中恢复而直接失败。状态迁移功能将在后续版本中加入。

### 在用户代码中管理`TypeSerializer` 和 `TypeSerializerConfigSnapshot` 类

因为`TypeSerializer` and `TypeSerializerConfigSnapshot`是和状态值一起作为checkpoints的一部分被写入的，所以在classpth的范围内，这些类的功能可能会影响恢复行为。

`TypeSerializer`是使用Java对象序列化直接写入到检查点中的，当新的序列化器告知其不兼容并需要状态迁移时，便需要提供它来读取恢复的状态字节。因此，如果因为对状态的序列化器进行升级导致原始的序列化器类不再存在或被修改(这会导致`serialVersionUID`的改变)，恢复操作将无法进行。当使用`CompatibilityResult.requiresMigration(TypeDeserializer deserializer)`来通知需要进行状态迁移时，我们可以通过提供`TypeDeserializer`作为替补来满足此需求。

恢复的检查点中的`TypeSerializerConfigSnapshot`类必须存在于classpath中，因为它们是对升级的序列化器进行安全检查的基本组件，当没有提供这些类就无法完成恢复。由于配置快照是使用自定义序列化操作被写入到检查点中的，所以可以随意改变类的实现，只要配置改变的兼容性已被版本控制机制 `TypeSerializerConfigSnapshot`处理。

{% top %}

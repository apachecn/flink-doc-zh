---
title:  "Task 生命周期"
nav-title: Task 生命周期
nav-parent_id: internals
nav-pos: 5
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

Task是Flink基本的执行单元。Task是operator并行任务中执行的地方。举个例子，一个operator的并行度设置为5，它的每个实例都会被一个单独的task实例来执行。 

`StreamTask`是flink 流计算引擎中所有不同task的基类。这个文档描述了`StreamTask`不同的生命周期，生命周期中不同阶段的方法。

{:toc}


## Operator 生命周期概括

因为task是operator实例并行执行的实体，它的生命周期与它对应的operator的生命周期紧密相关。所以，在深入了解`StreamTask`之前，先简要描述一下operator的生命周期。这个列表按照每个方法的调用顺序列出了所有的方法。给定一个operator可以有一个用户定义的函数(*UDF*)。在每个operator方法下，我们列出了这些方法调用UDF的生命周期。如果你的operator继承了`AbstractUdfStreamOperator`，那这些方法就会生效，`AbstractUdfStreamOperator`是所有执行udfs的operator的基类。

        // initialization phase
        OPERATOR::setup
            UDF::setRuntimeContext
        OPERATOR::initializeState
        OPERATOR::open
            UDF::open
        
        // processing phase (called on every element/watermark)
        OPERATOR::processElement
            UDF::run
        OPERATOR::processWatermark
        
        // checkpointing phase (called asynchronously on every checkpoint)
        OPERATOR::snapshotState
                
        // termination phase
        OPERATOR::close
            UDF::close
        OPERATOR::dispose
    
简而言之，调用`setup()`方法是为了做一些operator的初始化，比如`RuntimeContext`和metric收集的数据结构。在这之后，方法`initializeState()`给出了一个operator的状态初始化，然后`open()`方法执行所有operator的初始化，比如开启`AbstractUdfStreamOperator`的udf。

<span class="label label-danger">注意</span> 方法`initializeState()` 包含了状态初始化时候的逻辑（比如注册keyed state），也包含了在故障后从checkpoint中恢复的逻辑。本文中关于这部分有后面更详细的解释。


现在万事具备，operator已经准备好了处理输入的数据。输入的数据可以是如下这些东西:input elements（具体的输入啥数据）, watermark(处理乱序的水位线), 和 checkpoint barriers(用于生成ha的checkpoint)。Elements会被`processElement()`方法处理，watermarks是`processWatermark()`,checkpoint barriers 跟踪一个检查点的 invokes (asynchronously) 异步调用`snapshotState()` 方法，我们会在后面描述。对于每个输入的element，根据它的类型来决定上述的哪个方法被调用。注意,`processElement()`也是UDF逻辑被调用的地方， *举例* `MapFunction`中的 `map()`方法。

最后，在正常的情况下，operator的无故障中断（*举例* 如果一个流计算是有限的，当它到达了它的终点的时候),`close()`方法被调用执行，最后会关闭operator逻辑上需要关闭的动作（*举例* 关闭掉opertaor 执行期间所有的连接或输入输出流）,`dispose()`方法是在operator释放了占用的资源后调用的(*举例* operator数据占用掉的本机内存)。

由于故障或者人工取消中断的情况下,执行会直接跳转到`dispose()`,故障发生时候operator所处的阶段到`dispose()`之间的所有阶段都会被跳过。

**Checkpoints:** 当operator接受到checkpoint barrier的时候,operator 的`snapshotState()` 方法会被异步调用。Checkpoints在程序处理阶段执行，也就是在open之后，close之前。这个方法的责任是存储operator的状态，以达到当作业出现故障后重启，知道从哪里开始恢复[state backend]({{ site.baseurl }}/ops/state/state_backends.html)。以下给出Flink checkpoint机制的简单介绍，关于checkpoint更详细的机制介绍，请阅读相关文档:[Data Streaming Fault Tolerance]({{ site.baseurl }}/internals/stream_checkpointing.html)。

## Task 生命周期

上章是operator主要阶段的简介，这章更详细的描述集群上一个task执行期间是如何调用对应的方法的。各阶段的方法顺序描述主要包含在`StreamTask`的`invoke()`方法中。本文剩下的章节中包括两部分，一部分描述阶段的正常情况，一部分是task的无故障执行 (查看 [正常执行](#normal-execution)),另一部分描述task被取消的情况下遵循的不同过程(查看 [中断执行](#interrupted-execution)), 取消可以是人工取消或者其他原因，*举例* 执行期间有异常抛出.

### Normal Execution

### 正常执行

task正常执行完成没有被中断的情况下包含如下步骤:

	    TASK::setInitialState
	    TASK::invoke
    	    create basic utils (config, etc) and load the chain of operators
    	    setup-operators
    	    task-specific-init
    	    initialize-operator-states
       	    open-operators
    	    run
    	    close-operators
    	    dispose-operators
    	    task-specific-cleanup
    	    common-cleanup

如上所示，在任务恢复配置和初始化一些重要参数后，第一步就是恢复任务的初始化，任务范围状态。该步骤在`setInitialState()`中完成，这在下面两种情况下尤其重要:

1. 当task出现故障后，从最后一个成功的检查点恢复并重启
2. 当从一个[savepoint]({{ site.baseurl }}/ops/state/savepoints.html)恢复。

如果task是第一次执行，那么task的初始化状态是空的。

在所有的状态初始化恢复后，task进入`invoke()`方法。然后,它首先通过调用Operators的`setup()`方法来调用每一个本地的`init()`方法来完成初始化task的初始化。task初始化的详细内容,依赖于task的类型(`SourceTask`,`OneInputStreamTask` 或者`TwoInputStreamTask`等),这个步骤可能是不同的，但不管在什么情况下，这里都是获得必要的任务范围资源的地方。举个例子，`OneInputStreamTask`表示一个需要单独输入流的task，初始化将连接本地任务关联输入流不同分区的位置。


获得必要的资源后，就到了operator和udf从task-wide状态中获取他们各自的状态的时候。这在`initializeState()` 方法中完成,该方法会调用每个operator自己的`initializeState()` 方法.这个方法会被各个状态下的operator重载并包含状态的伙计初始化，不管是job第一次执行还是task从故障中通过检查点恢复都是这样。

现在task中所有的Operators都完成了初始化,`StreamTask`的`openAllOperators()`方法调用所有operator各自的`open()`方法。这个方法执行了所有的操作初始化，比如向定时服务器注册恢复的定时服务。一个单独的任务可能被多个operators执行，其中一个Operator消耗它前驱的输出。在这种情况下,`open()`方法由最后一个operator调用,*即* 这个operator的输出就是该task的输出。这么做使得当第一个Operator处理task的输入时候，所有的下游operator就开始准备接受它的输出。

<span class="label label-danger">注意</span> task中连续的opertaor被打开的顺序是从最后一个到第一个。

现在task可以重新(从故障中恢复)开始执行，operators可以处理新的输入数据。这时候task的`run()`方法被调用。这个方法会一直运行直到没有输入数据（有限的输入流）或者task被取消(手动取消或者非手动取消)。这个方法也是opeartor的 `processElement()` 和 `processWatermark()` 被调用的地方。

在运行完成的情况下,*即* 没有输入数据需要处理，在`run()`方法退出之后，这个任务进入关闭阶段。首先，定时服务停止新的定时器注册(*举例* 取消掉已经完成的定时器),清楚掉所有还未启动的定时器，等待当前正在执行的定时器完成。然后`closeAllOperators()`方法会尝试通过调用每个operator各自的`close()`方法来优雅的关闭operators。然后，所有的输出缓存数据被flushed，使这些数据可以被下游task处理，最后task尝试释放掉自己占用的资源，通过调用占用资源的operator各自的`dispose()`方法来释放。我们提到过，当打开operator的时候，顺序是最后一个operator到第一个。而关闭operator动作相反，是从第一个到最后一个。

<span class="label label-danger">注意</span> 关闭task中operator的顺序是从最后一个关到第一个。

最后，当所有的operator被关闭并且它们的资源被释放，task关闭它的定时器服务，执行它自己的清理任务,*举例* 清楚所有的内部缓存，然后执行task的通用任务清理，其中包含了关闭输出流管道和清楚输出缓存等。


**Checkpoints:** 前面我们说在执行`initializeState()`的时候,当从故障中恢复的情况，task和它所有的Operator和functions会恢复状态，从故障出现前最后一个被持久化存储起来的成功的checkpoint中恢复。在flink中Checkpoints被用户配置好后周期化的执行,这是在与主线程不同的线程中完成的。这就是为何checkpoints不在task生命周期的主要阶段中。简而言之,`CheckpointBarriers`这个特殊元素是在job的数据源task中，获取输入数据的时候被注入进去的，然后跟着实际的数据从source一直流传到sink中。source task在运行模式下注入这些barrier，并假定`CheckpointCoordinator`也在运行。不管何时一个task接受到一个barrier，它都会调度一个checkpoint线程去执行任务,这会调用task的operator的`snapshotState()`方法。当checkpoint被调用的时候，任务依然可以接收输入数据，但是这些数据被缓存起来，只有到checkpoint成功完成后数据才会被发送给下游并被处理。

### 异常中断
在前面的部分我们叙述了task从运行到结束的生命周期。有一种情况是task在任意一个点被取消，然后正常执行会被中断，以下操作会被执行:定时器服务关闭，task清理，operator资源回收，正常的任务清理，这些过程在上面都有描述。

{% top %}

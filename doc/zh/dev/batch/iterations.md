---
title: Iterations
nav-parent_id: batch
nav-pos: 2
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

迭代算法出现在数据分析的许多领域，例如机器学习或图形分析。这些算法对于实现大数据从数据中提取有意义的信息的承诺至关重要。随着对在非常大的数据集上运行这些算法的兴趣越来越大，需要以大规模并行方式执行迭代。

Flink程序通过定义**阶梯函数**并将其嵌入到特殊的迭代算子中来实现迭代算法。该运算符有两种变体：**Iterate**和**DeltaIterate**。两个算子都重复调用当前迭代状态的step函数，直到达到某个终止条件。

在这里，我们提供了两种算子的背景并概述了它们的用法。[编程指南](index.html) 解释了如何在Scala和Java中实现算子。我们还通过Flink的graph  API [Gelly]({{site.baseurl}}/dev/libs/gelly/index.html)，支持** vertex-centric和gather-sum-apply迭代**。

下表对两种算子的概述：

<table class="table table-striped table-hover table-bordered">
	<thead>
		<th></th>
		<th class="text-center">Iterate</th>
		<th class="text-center">Delta Iterate</th>
	</thead>
	<tr>
		<td class="text-center" width="20%"><strong>Iteration Input</strong></td>
		<td class="text-center" width="40%"><strong>Partial Solution</strong></td>
		<td class="text-center" width="40%"><strong>Workset</strong> and <strong>Solution Set</strong></td>
	</tr>
	<tr>
		<td class="text-center"><strong>Step Function</strong></td>
		<td colspan="2" class="text-center">Arbitrary Data Flows</td>
	</tr>
	<tr>
		<td class="text-center"><strong>State Update</strong></td>
		<td class="text-center">Next <strong>partial solution</strong></td>
		<td>
			<ul>
				<li>Next workset</li>
				<li><strong>Changes to solution set</strong></li>
			</ul>
		</td>
	</tr>
	<tr>
		<td class="text-center"><strong>Iteration Result</strong></td>
		<td class="text-center">Last partial solution</td>
		<td class="text-center">Solution set state after last iteration</td>
	</tr>
	<tr>
		<td class="text-center"><strong>Termination</strong></td>
		<td>
			<ul>
				<li><strong>Maximum number of iterations</strong> (default)</li>
				<li>Custom aggregator convergence</li>
			</ul>
		</td>
		<td>
			<ul>
				<li><strong>Maximum number of iterations or empty workset</strong> (default)</li>
				<li>Custom aggregator convergence</li>
			</ul>
		</td>
	</tr>
</table>


* This will be replaced by the TOC
{:toc}

Iterate 算子
----------------

**iterate 算子*覆盖了*简单形式的迭代*：在每次迭代中，**step function**消耗**整个输入**（*前一次迭代的结果*或*初始数据集*），并计算**下一版本的部分解决方案**（例如`map`, `reduce`, `join`等）。

<p class="text-center">
    <img alt="Iterate Operator" width="60%" src="{{site.baseurl}}/fig/iterations_iterate_operator.png" />
</p>

  1. **Iteration Input**: *第一次 iteration*的初始输入，可能来自于 *data source* 或 *前一次 算子*。
  2. **Step Function**: step函数将在每次迭代中执行。 它是一个由诸如`map`，`reduce`，`join`等算子组成的任意数据流，并且取决于您的具体任务。
  3. **Next Partial Solution**: 在每次迭代中，step函数的输出将被反馈到*下一次iteration*。
  4. **Iteration Result**: *最后一次迭代*的输出写入*sink*或用作*后面的算子*的输入。

有多种选项可以为迭代指定**终止条件**：

  - **Maximum number of iterations**: 没有任何进一步的条件，迭代将被执行多次。
  - **Custom aggregator convergence**: 迭代允许指定*自定义聚合器*和*收敛标准*像sum一样聚合记录的数量（聚合器），如果此数目为零（收敛标准），则终止。

您也可以在伪代码中的思考Iteration算子 ：

~~~java
IterationState state = getInitialState();

while (!terminationCriterion()) {
	state = step(state);
}

setFinalState(state);
~~~

<div class="panel panel-default">
	<div class="panel-body">
	有关详细信息和代码示例，请参阅 <strong><a href="index.html">编程指南</a> </strong> </div>
</div>

### 示例: 递增数字

在下列示例中,我们**迭代增加一组数字**：

<p class="text-center">
    <img alt="Iterate Operator Example" width="60%" src="{{site.baseurl}}/fig/iterations_iterate_operator_example.png" />
</p>

  1. **Iteration Input**: 初始输入是从数据源中读取的，由五个单字段记录(integers `1` to `5`)组成。
  2. **Step function**: step函数是一个`map`运算符，它将整数字段从`i`递增到`i + 1`。 它将应用于每个输入记录。
  3. **Next Partial Solution**:step函数的输出将是map 算子的输出，即带有递增整数的记录。
  4. **Iteration Result**: 经过10次迭代后，初始数字将增加10次，从而得到整数`11`到`15`。

~~~
// 1st           2nd                       10th
map(1) -> 2      map(2) -> 3      ...      map(10) -> 11
map(2) -> 3      map(3) -> 4      ...      map(11) -> 12
map(3) -> 4      map(4) -> 5      ...      map(12) -> 13
map(4) -> 5      map(5) -> 6      ...      map(13) -> 14
map(5) -> 6      map(6) -> 7      ...      map(14) -> 15
~~~

请注意， **1**, **2**, 和 **4**可以是任意数据流。


Delta Iterate 算子
----------------------

**delta iterate 算子** 涵盖了**增量迭代**的情况。增量迭代**有选择地修改它们**解**的元素，并发展解决方案而不是完全重新计算它。

在适用的情况下，这会导致**更高效的算法**，因为解决方案集中的每个元素都不会在每次迭代中发生变化。这可以**将注意力集中在解决方案的热部件**上，并保持**冷部件不变**。通常情况下，大部分解决方案都比较快速地进行冷却，后面的迭代只对一小部分数据进行操作。

<p class="text-center">
    <img alt="Delta Iterate Operator" width="60%" src="{{site.baseurl}}/fig/iterations_delta_iterate_operator.png" />
</p>

  1. **Iteration Input**: 初始工作集和解决方案集从 *数据源*  或  *前一个 算子*中读取，作为第一次迭代的输入。
  2. **Step Function**: step函数将在每次迭代中执行。 它是一个由诸如`map`，`reduce`，`join`等算子组成的任意数据流，并且取决于您的具体任务。
  3. **Next Workset/Update Solution Set**: *next workset* 驱动迭代计算，并将被反馈到*next workset* 中。此外，solution set 将被更新并隐式转发（不需要重建）。这两个数据集可以由不同的阶梯函数的算子更新。
  4. **Iteration Result**: *最后一次迭代*后，*solution set*写入**data sink*或用作*下一个算子*的输入。

delta迭代的默认**终止条件**由**空工作集合收敛准则**和**最大迭代次数**指定。当生成的*next workset*为空或达到最大迭代次数时，迭代将终止。也可以指定**自定义聚合器**和**收敛准则**。

你也可以通过下面的伪代码了解 iterate 算子:

~~~java
IterationState workset = getInitialState();
IterationState solution = getInitialSolution();

while (!terminationCriterion()) {
	(delta, workset) = step(workset, solution);

	solution.update(delta)
}

setFinalState(solution);
~~~

<div class="panel panel-default">
	<div class="panel-body">
	有关详细信息和代码示例，请参阅 <strong><a href="index.html">编程指南</a> </strong> </div>
</div>

### 示例：在图中传播最小值

在下面的例子中，每个顶点都有一个** ID **和一个**颜色**。每个顶点将其顶点ID传播到相邻的顶点。**目标**是*将最小ID分配给子图中的每个顶点*。如果收到的ID小于当前的ID，则它将变为带有收到的ID的顶点颜色。基于此算法的应用可以在*社区分析*或*连接组件*计算中找到。

<p class="text-center">
    <img alt="Delta Iterate Operator Example" width="100%" src="{{site.baseurl}}/fig/iterations_delta_iterate_operator_example.png" />
</p>
颜色可视化了**solution set的演变**。每次迭代时，最小ID的颜色在相应的子图中扩展。同时，每次迭代都会减少工作量（交换和比较顶点ID）。这对应于**workset的减小的大小**，在三次迭代之后从七个顶点变为零个，此时迭代终止。值得重要观察的是，下面的子图在上面子图之前收敛，而delta迭代能够用workset抽象来捕获它。

**初始输入**被设置为**workset 和solution set**。在上图中，
在上面的子图**ID1**（*橙色*）是**最小ID**。**在第一次迭代**中，它将传播到顶点2，顶点2随后将其颜色更改为橙色。顶点3和4将接收**ID2**（以*黄色*）作为其当前最小ID并更改为黄色。由于*vertex1*的颜色在第一次迭代中没有变化，因此可以在下一个 workset中跳过它。

在下面的子图中**ID 5**（*青色*）是**最小ID **。子图的所有顶点将在第一次迭代中接收它。再次，我们可以为下一个workset跳过未改变的顶点（*顶点5 *）。

在**2nd iteration**中，workset 大小已经从七个元素减少到五个元素（顶点2,3,4,6和7）。这些是迭代的一部分，并进一步传播其当前最小ID。在这次迭代之后，下面的子图已经收敛（图的**冷部分**），因为它在工作集中没有元素，而上半部分的子图需要为剩余的两个工作集元素（顶点3和4）进一步迭代（图的**热部分**）。

当**第3次迭代后工作集为空**时，迭代**终止**。

<a href="#supersteps"></a>

Superstep Synchronization
-------------------------

我们将迭代算子的stepfunction的每次执行称为*单次迭代*。在并行设置中，在迭代状态的不同分区上**多个阶梯函数实例并行计算**。在许多设置中，对所有并行实例的阶梯函数的一次评估形成了所谓的**superstep**，这也是同步的粒度。因此，迭代中的*所有*并行任务需要完成superstep，然后才能初始化下一个superstep。**终止标准**也将在superstep 屏障处进行评估。

<p class="text-center">
    <img alt="Supersteps" width="50%" src="{{site.baseurl}}/fig/iterations_supersteps.png" />
</p>

{% top %}

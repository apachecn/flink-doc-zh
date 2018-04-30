---
title: "配置依赖，连接器和库"
nav-parent_id: start
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

每一个 Flink 应用程序都依赖一系列 Flink 库。换言之，应用程序至少要会依赖于 Flink APIs.许多应用程序还依赖于某些连接器库（如 Kafka，Cassandra 等）。在运行 Flink 应用程序时（无论是在分布式中进行部署，还是在IDE中进行测试）， Flink 运行时库也必须可用。


## Flink 核心依赖和应用程序依赖

与大多数运行用户定义应用程序的系统一样，Flink 中有两大类依赖关系和库：

  - **Flink 核心依赖**: Flink 本身包含一系列运行系统所需的类和依赖项，例如调度，网络，检查点，故障转移，API，操作（例如窗口化），资源管理等。这些类和依赖关系构成 Flink 运行时的核心，并且在启动 Flink 应用程序时必须存在。

    这些核心类和依赖项都打包在 `flink-dist` jar 文件中. 他们是 Flink `lib` 文件夹的一部分，也是 Flink 基本容器镜像的一部分。 我们可以把这些依赖想象成包含 `String` 和 `List`等类的Java的核心库 (`rt.jar`, `charsets.jar`等)。

    Flink 核心依赖不包含任何连接器或类似CEP，SQL，ML的库，是为了避免默认情况下在类路径中包含过多的依赖项和类。事实上，我们希望核心依赖精简而强大，保持默认的类路径小，从而避免依赖冲突。

  - **用户应用程序依赖**: 用户应用程序依赖是特定用户应用程序所需的所有连接器，格式或库。
	用户应用程序通常打包到 *应用程序jar包* 中，其中包含应用程序代码和必需的连接器和库依赖关系。
	
	用户应用程序依赖性不包含Flink DataSet / DataStream API和运行时依赖项，因为这些已经是Flink核心依赖关系的一部分。


## 开始构建一个项目: 基本依赖
每个 Flink 应用程序都需要最低限度的 API 依赖来方便开发。 对于 Maven 项目，您可以使用 [Java Project Template] 或 Scala 项目模板来创建具有这些初始依赖关系的程序框架。

每个 Flink 应用程序都需要最低限度的 API 依赖来方便开发。 对于 Maven 项目，您可以使用 [Java 项目模板]({{ site.baseurl }}/quickstart/java_api_quickstart.html) or [Scala 项目模板]({{ site.baseurl }}/quickstart/scala_api_quickstart.html) 来创建具有这些初始依赖关系的程序框架。

当手动设置项目时，您需要为Java / Scala API添加以下依赖项（这里以 Maven 语法表示，其他构建工具也适用同样的依赖项（Gradle，SBT等））。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-java</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
{% endhighlight %}
</div>
</div>

关于IntelliJ的注意事项：为了使应用程序在IntelliJ IDEA中运行，Flink依赖项需要在编译范围内声明，而不是提供。否则IntelliJ不会将它们添加到类路径中，并且in-IDE执行将失败，并出现NoClassDefFountError。为了避免必须将依赖范围声明为compile（这是不推荐的，请参阅上文），上述链接的Java和Scala项目模板使用一个技巧：它们添加一个配置文件，当应用程序在IntelliJ中运行时，在不影响JAR文件打包的情况下，将依赖关系提升为范围编译。

**重要提示:** 请注意，所有这些依赖项需要设置scpoe变量为 *provided* 。
这意味着它们需要进行编译，而不是把它们打包到项目的应用程序jar包中 —— 这些依赖项是Flink 核心依赖，它们在任何设置中都可用。

强烈建议在把依赖设置为 *provided* 。 如果它们没有设置为需要*provided*，最好的情况是生成的JAR包变得过大，因为它也包含所有Flink核心依赖关系。最糟糕的情况是，添加到应用程序jar包中的 Flink核心依赖关系与您自己的某些依赖版本冲突（通常只能通过反向类加载来避免）。

**关于IntelliJ的注意事项:** 为了使应用程序在IntelliJ IDEA中运行，Flink依赖项需要将 scope 设置为*compile*，而不是*provided*。 否则IntelliJ不会将它们添加到类路径中，并且IDE内部将执行失败，报错 `NoClassDefFountError`。为了避免把依赖设置为需要编译时提供（这是不推荐的，请参阅上文），上述链接的Java和Scala项目模板使用了一个技巧：它们添加一个配置文件，当应用程序在IntelliJ中运行时，在不影响JAR文件打包的情况下，将依赖关系更改为编译时提供。

## 添加连接器和库依赖项

大多数应用程序需要特定的连接器或库来运行，例如Kafka，Cassandra等的连接器。这些连接器不是Flink核心依赖项的一部分，因此必须作为依赖项添加到应用程序中。

下面是一个添加Kafka 0.10连接器作为依赖项的例子（Maven语法）：
{% highlight xml %}
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.10{{ site.scala_version_suffix }}</artifactId>
    <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

我们建议将应用程序代码和所有必需的依赖关系打包成一个*包含依赖关系的jar包*，我们称之为*应用程序jar*。 应用程序jar可以提交给已经运行的Flink集群，或者添加到Flink应用程序容器镜像中。

从[Java 项目模板]({{ site.baseurl }}/quickstart/java_api_quickstart.html) 或者
[Scala 项目模板]({{ site.baseurl }}/quickstart/scala_api_quickstart.html)创建的项目被配置为在运行`mvn clean package`时自动将应用程序依赖项包含到应用程序jar中。 对于未从这些模板设置的项目，我们建议添加Maven Shade插件（见附录）来构建具有所有必需依赖项的应用程序jar。

重要说明：对于Maven（和其他构建工具），想要将依赖项正确打包到应用程序jar中，必须在编译时指定这些应用程序依赖项（不像核心依赖项，它必须在提供的作用域中指定）。

**重要提示:** 对于Maven（和其他构建工具），想要将依赖项正确打包到应用程序jar中，必须将这些应用程序依赖项指定为*compile*（不像核心依赖项，它必须指定为*provided*）。

## Scala 版本

Scala 不同版本 (2.10, 2.11, 2.12等)的二进制文件是互不兼容的。 因此，Flink for Scala 2.11不能与使用Scala 2.12的应用程序一起使用。

类似地，所有Flink依赖的后缀是所构建的Scala版本，例如`flink-streaming-scala_2.11`。

有关如何为特定Scala版本构建Flink的详细信息，请参阅[构建指南]({{ site.baseurl }}/start/building.html#scala-versions)

**注意：** 由于Scala 2.12中的重大变更，Flink 1.5目前仅针对Scala 2.11构建。 我们的目标是在下一个版本中增加对Scala 2.12的支持。
只使用Java的开发者可以选择任何Scala版本，Scala开发者需要选择与其应用程序的Scala版本相匹配的Scala版本。

## Hadoop依赖

**一般规则：不应该直接将Hadoop依赖项添加到您的应用程序中。**
*（唯一的例外是在Flink的Hadoop兼容包中使用现有的Hadoop输入/输出格式）*
如果您希望将Flink与Hadoop一起使用，则需要设置 Flink 使其包含Hadoop依赖项，而不是将Hadoop作为应用程序依赖项直接添加。 有关详细信息，请参阅[Hadoop安装指南]({{ site.baseurl }}/ops/deployment/hadoop.html)。


请配置这些依赖性类似于要测试或提供的依赖性范围。

这种设计有两个主要原因：

  - 某些Hadoop交互发生在Flink内核中，可能在用户应用程序启动之前发生，例如，为HDFS设置checkpoint，通过Hadoop的Kerberos token进行身份验证，或在YARN上进行部署。

  - Flink的反向类加载方法隐藏了许多来自核心依赖关系的传递依赖关系。这不仅适用于Flink自己的核心依赖关系，还适用于Hadoop的依赖关系。这样，应用程序就可以使用不同版本的相同依赖关系，而不会陷入依赖冲突（相信我们，这很重要，因为Hadoops依赖关系树非常庞大。）

如果您在IDE中进行测试或开发时，需要Hadoop相关依赖（例如访问HDFS），请将这些依赖的 scope 配置为 *test* 或者 *provided*。

## 附录: 构建一个含有依赖关系的 Jar 包的模板

要构建包含已声明的连接器和库所需的所有依赖项的应用程序JAR，可以使用以下shade插件定义：

{% highlight xml %}
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-shade-plugin</artifactId>
			<version>3.0.0</version>
			<executions>
				<execution>
					<phase>package</phase>
					<goals>
						<goal>shade</goal>
					</goals>
					<configuration>
						<artifactSet>
							<excludes>
								<exclude>com.google.code.findbugs:jsr305</exclude>
								<exclude>org.slf4j:*</exclude>
								<exclude>log4j:*</exclude>
							</excludes>
						</artifactSet>
						<filters>
							<filter>
								<!-- Do not copy the signatures in the META-INF folder.
								Otherwise, this might cause SecurityExceptions when using the JAR. -->
								<artifact>*:*</artifact>
								<excludes>
									<exclude>META-INF/*.SF</exclude>
									<exclude>META-INF/*.DSA</exclude>
									<exclude>META-INF/*.RSA</exclude>
								</excludes>
							</filter>
						</filters>
						<transformers>
							<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
								<mainClass>my.prorgams.main.clazz</mainClass>
							</transformer>
						</transformers>
					</configuration>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
{% endhighlight %}

{% top %}

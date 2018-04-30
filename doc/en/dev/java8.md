---
title: "Java 8"
nav-parent_id: api-concepts
nav-pos: 20
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

Java8新增了几种新的语言特性，旨在实现更快速、更清晰的编程实现。伴随着最重要特性就是常说的“Lambda表达式”的发布，Java8亦开启了函数式编程的大门。Lambda表达式允许以一种直截了当的方式实现和传递函数，而无需声明额外的（匿名）类。

最新版本的Flink对Java API的所有操作都支持使用Lambda表达式。本文档展示了如何使用Lambda表达式和当前应用局限性的描述。有关于Flink API的整体介绍介绍，可以参考[Programming Guide]({{ site.baseurl }}/dev/api_concepts.html)。

* TOC
{:toc}

### 例子

下面的示例阐述了如何实现一个简单的例子，使用一个Lambda表达式来符合内联`map()`函数的输入。输入函数`i`的类型 和`map()`函数的输出参数是不用去声明的，因为可由Java 8编译器来推断出。

~~~java
env.fromElements(1, 2, 3)
// returns the squared i
.map(i -> i*i)
.print();
~~~

接下来的两个例子展示了一个函数的不同实现，它使用`收集器`来作为输出。一些函数，例如`flatMap()`,为了类型安全需要为`计数器`定义一个输出类型（这个例子中是 `String`）。如果`收集器`的类型无法从周围的上下文中推断出，那就需要手动地通过Lambda表达式的参数列表中去声明。否则输出会被视为将会导致一些非法的行为的对象类型。

~~~java
DataSet<Integer> input = env.fromElements(1, 2, 3);

// collector type must be declared
input.flatMap((Integer number, Collector<String> out) -> {
    StringBuilder builder = new StringBuilder();
    for(int i = 0; i < number; i++) {
        builder.append("a");
        out.collect(builder.toString());
    }
})
// returns (on separate lines) "a", "a", "aa", "a", "aa", "aaa"
.print();
~~~

~~~java
DataSet<Integer> input = env.fromElements(1, 2, 3);

// collector type must not be declared, it is inferred from the type of the dataset
DataSet<String> manyALetters = input.flatMap((number, out) -> {
    StringBuilder builder = new StringBuilder();
    for(int i = 0; i < number; i++) {
       builder.append("a");
       out.collect(builder.toString());
    }
});

// returns (on separate lines) "a", "a", "aa", "a", "aa", "aaa"
manyALetters.print();
~~~

The following code demonstrates a word count which makes extensive use of Lambda Expressions.

~~~java
DataSet<String> input = env.fromElements("Please count", "the words", "but not this");

// filter out strings that contain "not"
input.filter(line -> !line.contains("not"))
// split each line by space
.map(line -> line.split(" "))
// emit a pair <word,1> for each array element
.flatMap((String[] wordArray, Collector<Tuple2<String, Integer>> out)
    -> Arrays.stream(wordArray).forEach(t -> out.collect(new Tuple2<>(t, 1)))
    )
// group and sum up
.groupBy(0).sum(1)
// print
.print();
~~~

### Compiler Limitations
Currently, Flink only supports jobs containing Lambda Expressions completely if they are **compiled with the Eclipse JDT compiler contained in Eclipse Luna 4.4.2 (and above)**.

Only the Eclipse JDT compiler preserves the generic type information necessary to use the entire Lambda Expressions feature type-safely.
Other compilers such as the OpenJDK's and Oracle JDK's `javac` throw away all generic parameters related to Lambda Expressions. This means that types such as `Tuple2<String, Integer>` or `Collector<String>` declared as a Lambda function input or output parameter will be pruned to `Tuple2` or `Collector` in the compiled `.class` files, which is too little information for the Flink compiler.

How to compile a Flink job that contains Lambda Expressions with the JDT compiler will be covered in the next section.

However, it is possible to implement functions such as `map()` or `filter()` with Lambda Expressions in Java 8 compilers other than the Eclipse JDT compiler as long as the function has no `Collector`s or `Iterable`s *and* only if the function handles unparameterized types such as `Integer`, `Long`, `String`, `MyOwnClass` (types without Generics!).

#### Compile Flink jobs with the Eclipse JDT compiler and Maven

If you are using the Eclipse IDE, you can run and debug your Flink code within the IDE without any problems after some configuration steps. The Eclipse IDE by default compiles its Java sources with the Eclipse JDT compiler. The next section describes how to configure the Eclipse IDE.

If you are using a different IDE such as IntelliJ IDEA or you want to package your Jar-File with Maven to run your job on a cluster, you need to modify your project's `pom.xml` file and build your program with Maven. The [quickstart]({{site.baseurl}}/quickstart/setup_quickstart.html) contains preconfigured Maven projects which can be used for new projects or as a reference. Uncomment the mentioned lines in your generated quickstart `pom.xml` file if you want to use Java 8 with Lambda Expressions.

Alternatively, you can manually insert the following lines to your Maven `pom.xml` file. Maven will then use the Eclipse JDT compiler for compilation.

~~~xml
<!-- put these lines under "project/build/pluginManagement/plugins" of your pom.xml -->

<plugin>
    <!-- Use compiler plugin with tycho as the adapter to the JDT compiler. -->
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <compilerId>jdt</compilerId>
    </configuration>
    <dependencies>
        <!-- This dependency provides the implementation of compiler "jdt": -->
        <dependency>
            <groupId>org.eclipse.tycho</groupId>
            <artifactId>tycho-compiler-jdt</artifactId>
            <version>0.21.0</version>
        </dependency>
    </dependencies>
</plugin>
~~~

If you are using Eclipse for development, the m2e plugin might complain about the inserted lines above and marks your `pom.xml` as invalid. If so, insert the following lines to your `pom.xml`.

~~~xml
<!-- put these lines under "project/build/pluginManagement/plugins/plugin[groupId="org.eclipse.m2e", artifactId="lifecycle-mapping"]/configuration/lifecycleMappingMetadata/pluginExecutions" of your pom.xml -->

<pluginExecution>
    <pluginExecutionFilter>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <versionRange>[3.1,)</versionRange>
        <goals>
            <goal>testCompile</goal>
            <goal>compile</goal>
        </goals>
    </pluginExecutionFilter>
    <action>
        <ignore></ignore>
    </action>
</pluginExecution>
~~~

#### Run and debug Flink jobs within the Eclipse IDE

First of all, make sure you are running a current version of Eclipse IDE (4.4.2 or later). Also make sure that you have a Java 8 Runtime Environment (JRE) installed in Eclipse IDE (`Window` -> `Preferences` -> `Java` -> `Installed JREs`).

Create/Import your Eclipse project.

If you are using Maven, you also need to change the Java version in your `pom.xml` for the `maven-compiler-plugin`. Otherwise right click the `JRE System Library` section of your project and open the `Properties` window in order to switch to a Java 8 JRE (or above) that supports Lambda Expressions.

The Eclipse JDT compiler needs a special compiler flag in order to store type information in `.class` files. Open the JDT configuration file at `{project directory}/.settings/org.eclipse.jdt.core.prefs` with your favorite text editor and add the following line:

~~~
org.eclipse.jdt.core.compiler.codegen.lambda.genericSignature=generate
~~~

If not already done, also modify the Java versions of the following properties to `1.8` (or above):

~~~
org.eclipse.jdt.core.compiler.codegen.targetPlatform=1.8
org.eclipse.jdt.core.compiler.compliance=1.8
org.eclipse.jdt.core.compiler.source=1.8
~~~

After you have saved the file, perform a complete project refresh in Eclipse IDE.

If you are using Maven, right click your Eclipse project and select `Maven` -> `Update Project...`.

You have configured everything correctly, if the following Flink program runs without exceptions:

~~~java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.fromElements(1, 2, 3).map((in) -> new Tuple1<String>(" " + in)).print();
env.execute();
~~~

{% top %}

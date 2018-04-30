---
title: "IDE 设置"
nav-parent_id: start
nav-pos: 3
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

* Replaced by the TOC
{:toc}

下面的部分介绍了如何将Flink项目导入可以开发Flink的IDE。要编写Flink程序， 请参阅[Java API]({{ site.baseurl }}/quickstart/java_api_quickstart.html)
 和 [Scala API]({{ site.baseurl }}/quickstart/scala_api_quickstart.html)
快速入门指南。

**注意:** 如果IDE中的某些内容不起作用，请首先尝试使用Maven命令行(`mvn clean package -DskipTests`) 。因为它可能是您的IDE有错误或未正确设置）。

## 准备开始

要开始，请首先从我们的某一个 [repositories](https://flink.apache.org/community.html#source-code)克隆 Flink 原文件，例如
{% highlight bash %}
git clone https://github.com/apache/flink.git
{% endhighlight %}

## IntelliJ IDEA

以下是关于如何设置IntelliJ IDEA IDE以开发Flink核心功能的简要指南。由于Eclipse存在混合Scala和Java项目的问题，因此越来越多的贡献者正在迁移到IntelliJ IDEA。

以下文档介绍了设置用于开发 Flink 的IntelliJ IDEA 2016.2.5（https://www.jetbrains.com/idea/download/）的安装步骤。

### 安装 Scala 插件

IntelliJ设置支持安装Scala插件。如果未安装，请在导入Flink之前按照以下说明启用对Scala项目和文件的支持：

1. 进入IntelliJ插件设置（IntelliJ IDEA - >首选项 - >插件）并点击“安装Jetbrains插件...”。
2. 选择并安装“Scala”插件。
3. 重新启动IntelliJ


### 导入Flink项目

1. 启动IntelliJ IDEA并选择“Import Project”
2. 选择Flink存储库的根文件夹
3. 选择“Import project from external model”，并选择“Maven”
4. 不改变默认选项并点击“next”，直到选择SDK。
5. 如果没有SDK，请在左上角创建一个带有“+”符号的文件夹，然后单击“JDK”，选择您的JDK主目录并单击“OK”。如果有 SDK，只需选择您的SDK。
6. 继续点击“next”并完成导入。
7. 右键单击导入的Flink项目 - > Maven - > Generate Sources and Update Folders。请注意，这会将Flink库安装在本地Maven存储库中，即 "/home/*-your-user-*/.m2/repository/org/apache/flink/".
   另外，`mvn clean package -DskipTests` 命令还可以为IDE创建必要的文件，但不安装库。
8. 构建项目 (Build -> Make Project)

### Checkstyle
IntelliJ 通过 Checkstyle-IDEA 支持 IDE 中 checkstyle。

1. 从IntelliJ插件库中安装“Checkstyle-IDEA”插件。
2. 点击 Settings -> Other Settings -> Checkstyle 来配置插件。
3. 将 "Scan Scope" 设置为 仅 Java源——"Only Java sources (including tests)"。
4. 在“Checkstyle Version”下拉列表中选择 _8.4_，然后单击“apply”。**这一步很重要，不要跳过它！**
5. 在“Configuration File”窗口中，使用加号图标添加新配置：
    1. 把 "Description" 设置为 "Flink"。
    2. 选择 "Use a local Checkstyle file", 并且将它指向您的库 `"tools/maven/checkstyle.xml"` 中。
    3. 选中 "Store relative to project location", 并且点击"Next"。
    4. 将 "checkstyle.suppressions.file" 属性配置为 `"suppressions.xml"`, 然后点击"Next", "Finish"。
6. 选择 "Flink" 作为唯一活动配置文件，然后点击"Apply" 和 "OK"。
7. 如果代码不符合编码规则，Checkstyle现在将会在编辑器中发送警告。

一旦安装了插件，您可以直接通过 Settings -> Editor -> Code Style -> Java -> Scheme旁边的设置图标 导入`"tools/maven/checkstyle.xml"`。比如可以自动调整为导入的 style 。

您可以通过打开Checkstyle工具窗口并单击“Check Module”按钮来扫描整个模块。扫描应报告没有错误。

<span class="label label-info">注意</span> 有些模块并没有完全经过 Checkstyle 检查，包括 `flink-core`, `flink-optimizer`, 和 `flink-runtime`。尽管如此，请确保您添加或者修改的代码符合 Checkstyle 的规则。

## Eclipse

**NOTE:** 根据我们的经验，由于旧版Eclipse版本与Scala IDE 3.0.3捆绑在一起的缺陷，或者由于版本与Scala IDE 4.4.1中捆绑的Scala版本不兼容，所以我们不推荐将 Eclipse 用于 Flink 。

**我们推荐您使用 IntelliJ 来代替 Eclipse ([参见上文](#intellij-idea))**

{% top %}

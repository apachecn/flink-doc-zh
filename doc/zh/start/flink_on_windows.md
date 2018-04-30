---
title:  "在Windows上运行Flink"
nav-parent_id: start
nav-pos: 12
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

如果您想在Windows计算机上本地运行Flink，则需要 [下载](http://flink.apache.org/downloads.html)并解压Flink二进制分发包。 之后，您可以使用 **Windows批处理文件** (`.bat`), 也可以使用 **Cygwin** 运行Flink Jobmanager。

## 使用Windows批处理文件来开始

要通过 *Windows批处理文件*来运行 Flink , 打开 Windows 命令提示符, 并指向 Flink 文件夹下的 `bin/` 目录并运行 `start-local.bat`。

注意：JRE文件夹下的``bin`` 必须提前添加到Windows的``%PATH%``变量中。按照本指南将Java添加到%PATH%变量中。按照这个[指南](http://www.java.com/en/download/help/path.xml) 将Java添加到Windows的``%PATH%``变量。

~~~bash
$ cd flink
$ cd bin
$ start-local.bat
Starting Flink job manager. Web interface by default on http://localhost:8081/.
Do not close this batch window. Stop job manager by pressing Ctrl+C.
~~~

之后，您需要打开第二个终端运行`flink.bat`才能使用它来运行作业。

{% top %}

## 使用Cygwin和Unix脚本来开始

使用 *Cygwin* 您需要打开 Cygwin 终端, 指向您的 Flink 目录并运行 `start-local.sh` 脚本:

~~~bash
$ cd flink
$ bin/start-local.sh
Starting jobmanager.
~~~

{% top %}

## 从Git安装Flink

如果您使用的是Windows git shell，并希望从git存储库安装Flink，则Cygwin可能会产生与此类似的故障：

~~~bash
c:/flink/bin/start-local.sh: line 30: $'\r': command not found
~~~

发生此错误的原因是，Git在Windows中运行时，会自动将UNIX换行符转换为Windows换行符。而Cygwin只能处理UNIX样式的换行符。解决方法是通过以下三个步骤调整Cygwin设置来正确处理换行符：

1. 启动 Cygwin shell

2. 输入以下来进入您的 home 目录

    ~~~bash
    cd; pwd
    ~~~

    这将返回 Cygwin 的根目录。

3. 使用记事本，写字板或其他文本编辑器打开 home 目录下的 `.bash_profile` 文件并添加以下内容:(如果文件不存在，您需要创建它）

~~~bash
export SHELLOPTS
set -o igncr
~~~

保存并打开一个新的 bash shell
{% top %}

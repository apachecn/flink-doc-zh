---
title:  "MapR Setup"
nav-title: MapR
nav-parent_id: deployment
nav-pos: 7
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

This documentation provides instructions on how to prepare Flink for YARN
executions on a [MapR](https://mapr.com/) cluster.

* This will be replaced by the TOC
{:toc}

## 在YARN（MapR版本）上运行Flink

以下文档基于MapR v5.2.0版本. 该文档将指导你如何向  [Flink on YARN]({{ site.baseurl }}/ops/deployment/yarn_setup.html)
集群(MapR)中提交任务或启动Flink会话。

### MapR上编译Flink

在MapR上运行Flink之前，需要在MapR的Hadoop 和 Zookeeper 版本的上编译. 可以用 Maven 在项目根目录以下命令编译Flink：

```
mvn clean install -DskipTests -Pvendor-repos,mapr \
    -Dhadoop.version=2.7.0-mapr-1607 \
    -Dzookeeper.version=3.4.5-mapr-1604
```

编译参数`vendor-repos`将添加MapR的类库以便编译过程中提取Hadoop/Zookeeper依赖包。 编译参数`mapr`另外解决了编译过程中Flink与MapR之间的一些依赖包冲突。同时，保证原本MapR类库依然在MapR节点上被使用. 两个配置文件必须先被激活。

默认情况下，`mapr`配置文件使用Hadoop / Zookeeper依赖关系构建 基于MapR版本5.2.0，所以不需要覆盖`hadoop.version`和`zookeeper.version`属性。 对于不同的MapR版本，只需覆盖这些属性即可。 对应不同MapR发行版的Hadoop / Zookeeper可以在MapR文档中找到，例如：[here](http://maprdocs.mapr.com/home/DevelopmentGuide/MavenArtifacts.html).。

### Job提交客户端设置

向MapR提交Flink任务的客户端需要通过以下设置。

确保MapR的JAAS配置文件被加载而避免登录失败： 

```
export JVM_ARGS=-Djava.security.auth.login.config=/opt/mapr/conf/mapr.login.conf
```

Make sure that the `yarn.nodemanager.resource.cpu-vcores` property is set in `yarn-site.xml`:

~~~xml
<!-- in /opt/mapr/hadoop/hadoop-2.7.0/etc/hadoop/yarn-site.xml -->

<configuration>
...

<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>...</value>
</property>

...
</configuration>
~~~

同时记住设置环境变量`YARN_CONF_DIR`或`HADOOP_CONF_DIR`指定`yarn-site.xml`文件所在路径：

```
export YARN_CONF_DIR=/opt/mapr/hadoop/hadoop-2.7.0/etc/hadoop/
export HADOOP_CONF_DIR=/opt/mapr/hadoop/hadoop-2.7.0/etc/hadoop/
```

确保在类路径中找到MapR本地库：

```
export FLINK_CLASSPATH=/opt/mapr/lib/*
```

如果使用`yarn-session.sh`在YARN上启动Flink会话，那么必需的做以下设置：

```
export CC_CLASSPATH=/opt/mapr/lib/*
```

## 在安全的MapR群集运行Flink

*注意：在Flink 1.2.0版本中，Flink的Kerberos身份验证有个漏洞，导致它无法与MapR的安全系统兼容。 请升级到更新的Flink版本，以便将Flink与安全的MapR群集一起使用。 更多细节，请查阅 [FLINK-5949](https://issues.apache.org/jira/browse/FLINK-5949)*

Flink 的  [Kerberos authentication]({{ site.baseurl }}/ops/security-kerberos.html) 和
[MapR's Security authentication](http://maprdocs.mapr.com/home/SecurityGuide/Configuring-MapR-Security.html).
是独立的. , 有了上述编译过程和环境变量设置，Flink不需要任何其他配置就可以与MapR Security配合使用。

用户只需使用MapR的`maprlogin`身份验证工具登录即可。未获得MapR登录凭证的用户将无法提交Flink job，并将报以下错误:

```
java.lang.Exception: unable to establish the security context
Caused by: o.a.f.r.security.modules.SecurityModule$SecurityInstallException: Unable to set the Hadoop login user
Caused by: java.io.IOException: failure to login: Unable to obtain MapR credentials
```

{% top %}

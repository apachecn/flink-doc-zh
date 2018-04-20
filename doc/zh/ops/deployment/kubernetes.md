---
title:  "Kubernetes Setup"
nav-title: Kubernetes
nav-parent_id: deployment
nav-pos: 4
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

[Kubernetes](https://kubernetes.io) 是一个容器编配系统。

* This will be replaced by the TOC
{:toc}

## 简单的Kubernetes Flink 集群

Kubernetes中的基本Flink集群部署包含三个组件:

* 单个Jobmanager的部署
* 一个Taskmanagers池的部署
* 一个暴露Jobmanager的RPC和UI端口的服务

### 启动群集

使用 [resource definitions found below](#simple-kubernetes-flink-cluster-
resources)，使用以下`kubectl`命令启动群集：

    kubectl create -f jobmanager-deployment.yaml
    kubectl create -f taskmanager-deployment.yaml
    kubectl create -f jobmanager-service.yaml

然后您可以通过`kubectl proxy`访问Flink UI：

1. 在终端运行`kubectl proxy`
2. 在你的浏览器中访问 [http://localhost:8001/api/v1/proxy/namespaces/default/services/flink-jobmanager:8081
](http://localhost:8001/api/v1/proxy/namespaces/default/services/flink-
jobmanager:8081) 

### 删除群集

使用`kubectl`删除群集：

    kubectl delete -f jobmanager-deployment.yaml
    kubectl delete -f taskmanager-deployment.yaml
    kubectl delete -f jobmanager-service.yaml

## 高级群集部署

在GitHub上有一个 [Flink Helm chart](https://github.com/docker-flink/
examples) 的早期版本。

## 附录

### 简单的Kubernetes Flink集群资源

`jobmanager-deployment.yaml`
{% highlight yaml %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: flink-jobmanager
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: flink
        component: jobmanager
    spec:
      containers:
      - name: jobmanager
        image: flink:latest
        args:
        - jobmanager
        ports:
        - containerPort: 6123
          name: rpc
        - containerPort: 6124
          name: blob
        - containerPort: 6125
          name: query
        - containerPort: 8081
          name: ui
        env:
        - name: JOB_MANAGER_RPC_ADDRESS
          value: flink-jobmanager
{% endhighlight %}

`taskmanager-deployment.yaml`
{% highlight yaml %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: flink-taskmanager
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: flink
        component: taskmanager
    spec:
      containers:
      - name: taskmanager
        image: flink:latest
        args:
        - taskmanager
        ports:
        - containerPort: 6121
          name: data
        - containerPort: 6122
          name: rpc
        - containerPort: 6125
          name: query
        env:
        - name: JOB_MANAGER_RPC_ADDRESS
          value: flink-jobmanager
{% endhighlight %}

`jobmanager-service.yaml`
{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager
spec:
  ports:
  - name: rpc
    port: 6123
  - name: blob
    port: 6124
  - name: query
    port: 6125
  - name: ui
    port: 8081
  selector:
    app: flink
    component: jobmanager
{% endhighlight %}

{% top %}

---
title: 把 Airflow 搬进 Kubernetes
subtitle: develop-etl-via-airflow-on-k8s
tags:
  - Airflow
  - Kubernetes
  - Helm
  - DevOps
  - Cloud-Native
categories: 一只代码狗的自我修养
---

## 背景介绍

稳定高效的进行数据处理几乎是如今每一家互联网公司都要面临的课题，尤其是对于[专注于气象数据研究的我司](https://www.seniverse.com/)而言，做数据分析和 [ETL](https://zh.wikipedia.org/wiki/ETL) 的工作是整个公司业务很重要的一部分。在脱离了原始的「刀耕火种」的时代之后，我们内部一直在使用 [Airflow](https://airflow.apache.org/) 作为数据处理流程的框架来管理日常的数据流任务。其实我们也调研了很多其他的方案，最后还是选定了看起来相对比较可靠也比较符合我们业务需求的开源项目 Airflow 来做这个事情，虽然在使用的过程中确实也遇到了不少坑。当时这个项目还在 [Apache Incubator](https://zh.wikipedia.org/wiki/Apache_Incubator)，目前已经顺利毕业了。

简单介绍一下 Airflow，一般由 WebServer（一套完整的 UI 界面用于随时查看任务的执行状态并可以手动执行一些操作）、Scheduler（用来做任务的调度和管理）、Worker（真正执行任务的部分，可能由很多个）组成。这是一个常见的分布式架构，你只需要把任务流的 [DAG](https://airflow.apache.org/concepts.html#dags) 用 Python 代码写好，然后配置好触发条件就可以让它长期运行下去。在实际生产环境中，我们大量使用了 [Celery Executor](https://airflow.apache.org/howto/executor/use-celery.html) 来把任务动态分布到多个 Worker 上执行。

<!-- more -->

如果数据处理任务长期不变，这样的系统已经可以满足我们的需求了。但实际上随着越来越多的数据任务被添加到整个系统，任务负载变得越来越重，很多时候固定数量的 Worker 已经不能及时的处理完被 scheduled 的任务，造成任务队列堆积，一段时间后如果一直不能改善负载情况甚至会拖垮整个系统。这种情况下，只能手动增加更多的 Worker 来分担任务处理工作。然后不同类型的数据处理越多，Worker 所需要安装的依赖也越多，每手工增加一个 Worker 的成本也越来越高。甚至对于气象数据的处理而言，有很多非常古老的数据处理工具（很多还是 Fortran 写的），经常出现依赖相互冲突的情况（版本冲突，编译通不过等等）。如果隔离各个任务之间的运行环境，以及如何根据负载需求动态的伸缩 Worker 的数据量日益成为了这个系统的一个痛点。

动态伸缩，环境隔离，自然让人联想到 [Docker](https://www.docker.com/) 和 [Kubernetes](https://kubernetes.io/zh/) 这样的技术。好在 Airflow 1.10 版本引入了 [Kubernetes Executor](https://airflow.apache.org/kubernetes.html) 允许为每一个任务创建新的 Pod 来处理，而执行完之后新创建的 Pod 会被清理掉，并且每一个任务都可以指定不同的 Docker image 来处理，这样看起来就可以完全解决我们前面的问题。

目前这部分功能似乎还很不稳定，官方文档和讨论都还不多，这篇博客也是为了记录下我们的踩坑过程。

## 在 Kubernetes 上搭建 Airflow

我们使用 [Helm](https://helm.sh/) 来管理在 Kubernetes 上个各个应用以及它们的依赖，目前官方也已经给出了 [stable/airflow 的 Chart](https://github.com/helm/charts/tree/master/stable/airflow)，我们可以直接使用或部分参考。下面我们介绍一下从零开始的搭建过程。

1. 安装 Kubernetes

   本地安装的可以选择 [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)，主要它解决了跨平台的问题。我在 macOS 上使用的是 [Docker Desktop for Mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac) 里[自带的 Kubernetes 集群](https://www.docker.com/blog/docker-mac-kubernetes/)。如果你安装遇到了问题可以看看[这里](https://github.com/gotok8s/k8s-docker-desktop-for-mac)是不是解决了你的问题，或者自行 Google 也可。

2. 安装 Helm

   Helm 3 已经推出了正式版，可惜我们线上还在使用 Helm 2，而版本不兼容是没法使用的。为了跟线上保持一致，我本地也是安装的 Helm 2，所以后面都是以 Helm 2 的操作来执行的，第一次进行的朋友可以直接用 Helm 3。当然安装过程也[很可能出现问题](https://github.com/helm/helm/issues/4730)，可以自行 Google 解决。

3. 在 Kubernetes 上用 Helm 安装 Airflow Chart

   直接根据[官方的 Airflow Chart](https://github.com/helm/charts/tree/master/stable/airflow)步骤执行 `helm install --namespace "airflow" --name "airflow" stable/airflow` 即可在 Kubernetes 上安装一个标准的 Airflow 集群，之后可以查看各个 Pod 的状态是不是已经 ready（一切正常的情况下会在 `airflow` 的 Namespace 下安装 airflow-web、airflow-scheduler、airflow-postgresql、airflow-flower、airflow-redis）。还可以根据 NOTES 的提示在浏览器中查看 Airflow 的 UI 界面是不是也启动良好：

   ```text
   NOTES:
   Congratulations. You have just deployed Apache Airflow
      export POD_NAME=$(kubectl get pods --namespace airflow -l "component=web,app=airflow" -o jsonpath="{.items[0].metadata.name}")
      echo http://127.0.0.1:8080
      kubectl port-forward --namespace airflow $POD_NAME 8080:8080

    Open Airflow in your web browser
   ```

4. 根据自身业务情况自定义一些配置

   之后可以根据 Helm 的 官方 Airflow Chart 提供的配置方式进行一些自定义配置，比如把默认的 Airflow 镜像 `puckel/docker-airflow` 替换成自己根据自身业务需求构建的（比如已经安装了实现业务需求所有必要底层依赖的 Airflow 镜像，比如开启 Kubernetes Executor and operator 必需的 `apache-airflow[kubernetes]`，这样就不用每次部署更新都安装这些依赖了）、更改 `executor` 的类型（我们改为 `Kubernetes`）等等。官方的 Chart 目录下也提供了[一个例子](https://github.com/helm/charts/blob/master/stable/airflow/examples/minikube-values.yaml)可供参考。

   值得额外注意的是我们如何做数据的持久化。这既包括 DAGs、日志 logs，也包括 Airflow 的运行态数据 —— 存储在 Postgres 或 MySQL 中的数据 —— 如何持久化。官方的 README 对此[已经有介绍](https://github.com/helm/charts/tree/master/stable/airflow#dags-deployment)，同样如果有额外的自定义配置直接写到我们自己的 `values` YAML 文件中即可。[`airflow.cfg`](https://github.com/apache/airflow/blob/1.10.6/airflow/config_templates/default_airflow.cfg) 中 `[kubernetes]` 对应的每一项配置也应该过一遍并做相应的修改，尤其是关于 `namespace` 和 `dags_in_image` 的部分。对我司而言，我们自己的 Kubernetes 集群运行在阿里云上，直接使用一个外挂的 NAS 作为 DAGs 和 logs 的共享存储即可（通过 `extraVolumeMounts` 和 `extraVolumes` 挂载和声明，也要注意跟 `dags.path` 路径保持一致），以后我们自己开发的 DAGs 可以直接通过 CI/CD 更新到 NAS 上相应的目录。数据库采用一个已经存在的外部 Postgres。

   假设我们将自定义的 `values` 配置写成 `minikube-values.yaml` 的本地 YAML 文件，就可以根据 `helm install --namespace "airflow" --name "airflow" stable/airflow -f minikube-values.yaml` 启动一个经过自定义修改后的 Airflow 集群（可以先把之前安装的集群通过 `helm delete --purge "airflow"` 清除）。

因为业务需求和基础设施现状的不同，中间可能有不同的架构选择，但总体上经过这几步之后，一个采用 `KubernetesExecutor` 并运行在 Kubernetes 上的 Airflow 集群就基本搭建好了。

## 开发 DAGs

参考 Airflow 官方文档中 Kubernetes Operator 的例子可以开发适用于自身业务需求的 DAG。之后就可以访问 Airflow WebServer 来开启相应的 DAG 任务，并观察集群中 Task 的运行行为是否与预期的一致。

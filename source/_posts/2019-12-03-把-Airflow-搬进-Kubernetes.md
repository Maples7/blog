---
title: 把 Airflow 搬进 Kubernetes
tags:
  - Airflow
  - Kubernetes
  - Helm
  - DevOps
  - Cloud-Native
subtitle: develop-etl-via-airflow-on-k8s
categories: 一只代码狗的自我修养
date: 2019-12-03 00:35:32
---

## 背景介绍

稳定高效的进行数据处理几乎是如今每一家互联网公司都要面临的课题，尤其是对于[专注于气象数据研究的我司](https://www.seniverse.com/)而言，做数据分析和 [ETL](https://zh.wikipedia.org/wiki/ETL) 的工作是整个公司业务很重要的一部分。在脱离了原始的「刀耕火种」的时代之后，我们内部一直在使用 [Airflow](https://airflow.apache.org/) 作为数据处理流程的框架来管理日常的数据流任务。其实我们也调研了很多其他的方案，最后还是选定了看起来相对比较可靠也比较符合我们业务需求的开源项目 Airflow 来做这个事情，虽然在使用的过程中确实也遇到了不少坑。当时这个项目还在 [Apache Incubator](https://zh.wikipedia.org/wiki/Apache_Incubator)，目前已经顺利毕业了。

简单介绍一下 Airflow，一般由 WebServer（一套完整的 UI 界面用于随时查看任务的执行状态并可以手动执行一些操作）、Scheduler（用来做任务的调度和管理）、Worker（真正执行任务的部分，可能有很多个）组成。这是一个常见的分布式架构，你只需要把任务流的 [DAG](https://airflow.apache.org/concepts.html#dags) 用 Python 代码写好，然后配置好触发条件就可以让它长期运行下去。在实际生产环境中，我们大量使用了 [Celery Executor](https://airflow.apache.org/howto/executor/use-celery.html) 来把任务动态分布到多个 Worker 上执行。

<!-- more -->

如果数据处理任务长期不变，这样的系统已经可以满足我们的需求了。但实际上随着越来越多的数据任务被添加到整个系统，任务负载变得越来越重，很多时候固定数量的 Worker 已经不能及时的处理完被 scheduled 的任务，造成任务队列堆积，一段时间后如果一直不能改善负载情况甚至会拖垮整个系统。这种情况下，只能手动增加更多的 Worker 来分担任务处理工作。然后不同类型的数据处理越多，Worker 所需要安装的依赖也越多，每手工增加一个 Worker 的成本也越来越高。甚至对于气象数据的处理而言，有很多非常古老的数据处理工具（很多还是 Fortran 写的），经常出现依赖相互冲突的情况（版本冲突，编译通不过等等）。如何隔离各个任务之间的运行环境，以及如何根据负载需求动态的伸缩 Worker 的数量日益成为了这个系统的一个痛点。

动态伸缩，环境隔离，自然让人联想到 [Docker](https://www.docker.com/) 和 [Kubernetes](https://kubernetes.io/zh/) 这样的技术。好在 Airflow 1.10 版本引入了 [Kubernetes Executor 和 Kubernetes Operator](https://airflow.apache.org/kubernetes.html) 允许为每一个任务创建新的 Pod 来处理，而执行完之后新创建的 Pod 会被清理掉，并且每一个任务都可以指定不同的 Docker image 来处理，这样看起来就可以完全解决我们前面的问题。

目前这部分功能似乎还很不稳定，官方文档和讨论都还不多，这篇博客也是为了记录下我们的踩坑过程。

## 在 Kubernetes 上搭建 Airflow

我们使用 [Helm](https://helm.sh/) 来管理在 Kubernetes 上个各个应用以及它们的依赖，目前官方也已经给出了 [stable/airflow 的 Chart](https://github.com/helm/charts/tree/master/stable/airflow)，我们可以直接使用或部分参考。下面我们介绍一下从零开始的搭建过程。

1. 安装 Kubernetes

   本地安装的可以选择 [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)，主要它解决了跨平台的问题。我在 macOS 上使用的是 [Docker Desktop for Mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac) 里[自带的 Kubernetes 集群](https://www.docker.com/blog/docker-mac-kubernetes/)，如果你安装遇到了问题可以看看[这里](https://github.com/gotok8s/k8s-docker-desktop-for-mac)是不是解决了你的问题，或者自行 Google 也可。

2. 安装 Helm

   Helm 3 已经是正式的稳定版本，可惜我们线上还在使用 Helm 2，而 Helm 2 的版本不兼容是没法使用的。为了跟线上保持一致，我本地也是安装的 Helm 2，所以后面都是以 Helm 2 的操作来执行的，第一次进行的朋友可以直接用 Helm 3。当然安装过程也[很可能出现问题](https://github.com/helm/helm/issues/4730)，可以自行 Google 解决。

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

   之后可以根据 Helm 的 官方 Airflow Chart 提供的配置方式进行一些自定义配置，比如把默认的 Airflow 镜像 `puckel/docker-airflow` 替换成自己根据自身业务需求构建的（比如已经安装了实现业务需求所有必要底层依赖的 Airflow 镜像，或者对于 Kubernetes Executor and operator 必需的 `apache-airflow[kubernetes]`，这样就不用每次部署更新都安装这些依赖了）、更改 `executor` 的类型（我们改为 `Kubernetes`）等等。官方的 Chart 目录下也提供了[一个例子](https://github.com/helm/charts/blob/master/stable/airflow/examples/minikube-values.yaml)可供参考。

   值得额外注意的是我们如何做数据的持久化。这既包括 DAGs、日志 logs，也包括 Airflow 的运行态数据 —— 存储在 Postgres 或 MySQL 中的数据 —— 如何持久化。官方的 README 对此[已经有介绍](https://github.com/helm/charts/tree/master/stable/airflow#dags-deployment)，同样如果有额外的自定义配置直接写到我们自己的 `values` YAML 文件中即可。[`airflow.cfg`](https://github.com/apache/airflow/blob/1.10.6/airflow/config_templates/default_airflow.cfg) 中 `[kubernetes]` 对应的每一项配置也应该过一遍并做相应的修改，尤其是关于 `namespace` 和 `dags_in_image` 的部分。对我司而言，我们自己的 Kubernetes 集群运行在阿里云上，直接使用一个外挂的 NAS 作为 DAGs 和 logs 的共享存储即可（通过 `extraVolumeMounts` 和 `extraVolumes` 挂载和声明，也要注意跟 `dags.path` 路径保持一致），以后我们自己开发的 DAGs 可以直接通过 CI/CD 更新到 NAS 上相应的目录。数据库采用一个已经存在的外部 Postgres。

   假设我们将自定义的 `values` 配置写成 `minikube-values.yaml` 的本地 YAML 文件，就可以用 `helm install --namespace "airflow" --name "airflow" stable/airflow -f minikube-values.yaml` 启动一个经过自定义修改后的 Airflow 集群（可以先把之前启动的集群通过 `helm delete --purge "airflow"` 清除）。

因为业务需求和基础设施现状的不同，中间可能有不同的架构选择，但总体上经过这几步之后，一个采用 `KubernetesExecutor` 并运行在 Kubernetes 上的 Airflow 集群就基本搭建好了。

## 开发 DAGs

参考 Airflow 官方文档中 Kubernetes Operator 的例子可以开发适用于自身业务需求的 DAG。之后就可以访问 Airflow WebServer 来开启相应的 DAG 任务，并观察集群中 Task 的运行行为是否与预期的一致，并逐步迭代。

## 悄悄告诉你，真相其实是这样……

实际上，在经过我的一番折腾之后，最终我们线上使用的版本并没有使用官方的 Helm Chart，而是完全基于我们自己 build 的 Airflow 镜像，从头搭建了我们自己的 Helm Release Chart，这样整个系统拥有最高的可定制度（有些问题不在代码层面定制根本没法绕过去，下文有详述），同时也剔除了很多 Helm 官方 Chart 里有但我们不需要的东西。跟官方的 Chart 相比，我们主要做了这些更改：

- 没有使用 Airflow 官方的 Kubernetes Executor，而是自己继承 `LocalExecutor` 类定制我们自己的 `KubeExecutor`，实际做的事情也很简单，就是实际在执行任务的时候不直接像 `LocalExecutor` 那样在本地执行命令，而用 `kubectl` 把任务指定在某一个 Worker Pod 或使用了 `KubernetesPodOperator` 启动的临时 Pod 里运行。这样做主要考虑的是官方的 `KubernetesPodOperator` 为了一些通用性功能从而进行了较为复杂的流程设计，我们团队自身完全可以结合在阿里云里 Kubernetes 集群的具体特点改造 `LocalExecutor`，这样不仅在运行时可以节省更多的机器资源，而且对于我们最核心的需求 —— 稳定的生产数据 —— 来说是更有利的，毕竟在运行时动态操作 Kubernetes 集群资源的流程变得简单了很多。

- 阿里云还有一个很大的坑是在它上面构建的官方 Kubernetes 集群产品不提供阿里云的 CA 根证书校验，而 [Kubernetes 官方明确指明了需要校验 CA bundle](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/#directly-accessing-the-rest-api-1)，导致我们无法通过 API 与 Kubernetes Server 进行交互，进而在使用 `KubernetesPodOperator` 时会因为证书校验失败而无法创建 Pod。而 `KubernetesPodOperator` 也并没有提供参数让我们将 `verify_ssl` 设置为 False。所以最终我们也没有直接用官方的 `KubernetesPodOperator`，而是自己改了一个类似的把 `verify_ssl` 设置为 False 的 Operator 的版本来使用。同时这里也呼吁一下阿里云能按照技术社区标准来提供自己的技术服务与产品（我们提了工单 argue 这个事情，最终客服给我提供了根证书并叮嘱不要扩散，并不清楚阿里云不能对外公开根证书是基于什么样的考量；在其他项目中我们使用了客服提供的根证书来进行校验是没有问题的）。

- 去掉了不必要的通用型适配的各种选择，完全按自身需求合理定制。

虽然我们做了定制，需要考虑的一些核心问题是没有变的，它们是：如何共享 DAG 的 Python 代码、如何做日志和数据库数据的持久化、是否使用 [XComs](https://airflow.apache.org/docs/stable/concepts.html?highlight=xcom#xcoms) 来做 Task 之间的消息通信（同时在集群环境下如何实现这一点）、如何注入依赖（官方 Helm Chart 的 `requirements.txt` 方式）等等。由于这个尝试和定制的过程着实复杂和不那么让人愉快，甚至需要阅读一些 Airflow 的源码和各个云服务组件的接口参数设计，所以一篇博客无法一一详尽，我自己也不想写那些我们是如何绕过由于各方面的设计缺陷或开源产品不稳定导致的问题的技术细节，它们可能时刻会变，也没有技术深度，更多的是枯燥的云服务运维细节，所以不会在这里赘述太多而分散更大层面的宏观框架上的注意力。

## 结语

在我们实际使用的过程中其实还遇到了很多 Airflow 的坑，作为一个 Apache 基金会的开源项目，它确实弥补了 ETL 场景下工具和框架的缺失，也提供了非常丰富的功能，不过尽管目前版本号已经到了 1.10.6，但还是有很多明显可感知的 BUG 存在，实际的使用感受是能用但总存在一些小问题，偶尔对任务的调度和管理还可能会失灵，不那么可靠，不算是一个高质量的开源项目，所以在使用时加上适当的重试机制是很有必要的。不过目前社区里在这部分并没有更好的替代品，Airflow 已经是我们考查过的最契合管理 ETL 流程的框架了。如果有新的开源项目能弥补这方面的空缺我们会很乐意去尝试，我们自己团队也在考虑是否在更长期的计划中开发一套自己的 ETL 流程框架。

几周不断折腾尝试的搭建、使用和实际开发的过程下来，其实涉及了很多开发[云原生服务](https://pivotal.io/cloud-native)常用的 DevOps 工具和组件，比如 Helm、Kubernetes 和 Docker，能得心应手的使用这些工具需要一段时间的学习实践来积累知识和经验。要想在 Kubernetes 环境下玩转 Airflow，了解这些周边支撑的工具也是不可或缺极其重要的一部分。

---
title: ELK 从搭建到使用
subtitle: elk-starter
tags:
  - ELK
  - 日志系统
categories: 一只代码狗的自我修养
---
## 简介
ELK 是 Elasticsearch + Logstash + Kibana 套件的简称，用它们可以搭建强大的日志管理管理系统。    
其中，各个组件的功能如下：  
- Elasticsearch：分布式、可扩展、实时、多用户能力、RESTful 接口的搜索与数据分析引擎。它能从项目一开始就赋予你的数据以搜索、分析和探索的能力。
- Logstash：可以对你的日志进行收集、清洗、过滤和转发，并将其存储供以后使用。
- Kibana：使数据可视化，清晰的展示日志信息。

本文所使用的相关宿主环境和版本信息如下：
- 宿主环境：
```
vagrant@jessie:~$ uname -a
Linux jessie 3.16.0-4-amd64 #1 SMP Debian 3.16.39-1 (2016-12-30) x86_64 GNU/Linux
```
- Elasticsearch 版本：5.0.0
- Kibana 版本：5.0.0

下述相关命令请注意宿主环境和软件版本的差异。

## 安装
由于众所周知的网络原因，首先从[官网](https://www.elastic.co/start)通过各种更快速的方式把 Elasticsearch 和 Kibana 的安装包下载好，然后通过 `tar -zxvf elasticsearch-5.0.0.tar.gz` 和 `tar -zxvf kibana-5.0.0-darwin-x86_64.tar.gz` 解压。

### 安装 JRE
由于 Elasticsearch 是用 Java 写的，所以需要（至少）安装 Java 运行环境 JRE。而新版本的 Elasticsearch 似乎需要 Java 8 的版本，所以用 `java -version` 检查你的 Java 版本，如果不符合要求或者本身就没有安装 JRE，那么查看这篇文章安装并配置 Java 环境：[How to Install JAVA 8 (JDK/JRE 8u121) on Debian 8 & 7 via PPA](https://tecadmin.net/install-java-8-on-debian/)。

### 安装 X-Pack
X-Pack 是官方推荐安装的插件，它把安全、监控、报警、报告和图形化功能集成到了一个包中。

分别在 Elasticsearch 和 Kibana 解压出的目录下运行 `bin/elasticsearch-plugin install x-pack` 和 `bin/kibana-plugin install x-pack`

当然，你也同样可以自己下载 X-Pack 安装包后再从本地安装，可以参考这里：[Installing X-Pack on Offline Machines](https://www.elastic.co/guide/en/x-pack/current/installing-xpack.html#xpack-installing-offline)。

### 

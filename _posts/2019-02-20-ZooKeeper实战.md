---
layout: post
title: ZooKeeper实战
category: Blog
tags: [分布式锁,ZooKeeper,微服务]
---

## 概述

ZooKeeper是一种为分布式应用所设计的高可用、高性能且一致的开源协调服务，它提供了一项基本服务：**分布式锁服务**。由于ZooKeeper的开源特性，后来我们的开发者在分布式锁的基础上，摸索了出了其他的使用方法：**配置维护、组服务、分布式消息队列**、**分布式通知/协调**等。

### ZooKeeper特性

- **简单**

Zookeeper的核心是一个精简的文件系统，它支持一些简单的操作和一些抽象操作，例如，排序和通知。

- **丰富**

Zookeeper的原语操作是很丰富的，可实现一些协调数据结构和协议。例如，分布式队列、分布式锁和一组同级别节点中的“领导者选举”。

- **高可靠**

Zookeeper支持集群模式，可以很容易的解决单点故障问题。

- **松耦合交互**

不同进程间的交互不需要了解彼此，甚至可以不必同时存在，某进程在zookeeper中留下消息后，该进程结束后其它进程还可以读这条消息。

- **资源库**

Zookeeper实现了一个关于通用协调模式的开源共享存储库，能使开发者免于编写这类通用协议。

### ZooKeeper的安装

- **独立模式安装**

Zookeeper的运行环境是需要java的，建议安装oracle的java6.

可去官网下载一个稳定的版本，然后进行安装：[http://zookeeper.apache.org/](http://zookeeper.apache.org/)

解压后在zookeeper的conf目录下创建配置文件zoo.cfg，里面的配置信息可参考统计目录下的zoo_sample.cfg文件，我们这里配置为：

```properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/zookeeper-data/
clientPort=2181
```

**tickTime**：指定了ZooKeeper的基本时间单位（以毫秒为单位）；

**initLimit**：指定了启动zookeeper时，zookeeper实例中的随从实例同步到领导实例的初始化连接时间限制，超出时间限制则连接失败（以tickTime为时间单位）；

**syncLimit**：指定了zookeeper正常运行时，主从节点之间同步数据的时间限制，若超过这个时间限制，那么随从实例将会被丢弃；

**dataDir**：zookeeper存放数据的目录；

**clientPort**：用于连接客户端的端口。

- **启动一个本地的ZooKeeper实例**

```bash
% zkServer.sh start
```

检查ZooKeeper是否正在运行

```bash
echo ruok | nc localhost 2181
```

若是正常运行的话会打印“imok”。

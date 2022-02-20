---
title: 多集群支持
---

Orleans v.1.3.0增加了对联合多个Orleans集群的支持，使其成为像单个服务一样的松散连接的*多集群*。

多集群有利于*地理分布式*的服务，也就是说，使一个Orleans的应用程序更容易在世界各地的多个数据中心运行。另外，多集群也可以在一个数据中心内运行，以获得更好的故障与性能隔离。

所有机制的设计都特别注意：(1)尽量减少集群之间的通信，(2)即使其他集群故障或无法到达，也能让每个集群自主地运行。

## 配置与操作

下面我们介绍如何配置和操作一个多集群。

[**通信**](GossipChannels.md)。集群间通过与集群内相同的Silo对Silo的连接进行通信。为了交换状态和配置信息，集群使用一个Gossip机制和Gossip通道实现。

[**Silo配置**](SiloConfiguration.md)。Silo需要配置，以便它们知道自己属于哪个集群（每个集群由一个独特的字符串标识）。此外，每个Silo都需要配置连接字符串，使它们在启动时能够连接到一个或多个[Gossip频道](GossipChannels.md)。

[**多集群配置注入**](MultiClusterConfiguration.md)。在运行时，服务操作者可以指定和/或改变*多集群配置*，其中包含一个集群ID的列表，以指定哪些集群是当前多集群的一部分。这是通过调用任何一个集群中的管理Grain来完成的。

## 多集群Grains

下面我们介绍如何在应用层面上使用多集群功能。

[**全局单例Grains**](GlobalSingleInstance.md)。开发者可以指出集群何时以及如何协调他们的Grain目录与一个特定的Grain类的关系。`[GlobalSingleInstance]`特性意味着我们希望与在单个全局集群中运行Orleans时的行为相同：即把所有调用路由到Grain的单个激活中。相反，`[OneInstancePerCluster]`特性表示每个集群可以有自己的独立激活。如果不希望集群之间的通信，这会很合适。

**Log-view Grains**  _(v.1.3.0中不提供)_。一种特殊类型的Grain，使用新的API，类似于事件溯源，用于同步或持久化Grain状态。它可以用来自动和有效地在集群之间和与存储同步Grain的状态。因为它的同步算法可以安全地用于可重入Grain，并被优化为使用批处理和Grain复制，所以当一个Grain在多个集群中被频繁访问，和/或被频繁写入存储时，它可以比标准Grain表现得更好。对Log-view Grain的支持还不是主分支的一部分。我们在[geo-orleans分支](https://github.com/sebastianburckhardt/orleans/tree/geo-samples)中有一个包含示例和一些文档的预发布版本。目前，它正由一个早期采用者在生产中进行评估。
---
title: 多集群通信
---

网络必须以如下方式配置：所有Orleans Silo都可以通过TCP/IP连接到任何其他Orleans Silo，无论它位于世界何处。具体如何实现这一点不在Orleans的考虑范围之内，因为这取决于Silo的部署方式和位置。
 
例如，在Windows Azure上，我们可以使用VNETs来连接一个区域内的多个部署，并使用网关来连接不同区域的VNETs。

### 集群ID

每个集群都有自己独特的集群ID。集群ID必须在全局配置中指定。

集群ID不可为空，也不能包含逗号。另外，如果使用Azure Table存储，集群ID不能包含行键所禁止的字符（/, \, #, ?）。

我们建议使用非常短的字符串作为集群ID，因为集群ID经常被来回传输，并且可能被一些日志视图提供者存储在存储器中。

### 集群网关

每个集群自动指定其活动Silo的一个子集作为*集群网关*。集群网关直接向其他集群公布其IP地址，因此可以作为"首次接触点"。默认情况下，最多有10个Silo（可以通过 `MaxMultiClusterGateways`进行配置）会被指定为集群网关。

不同集群中的Silo之间的通信*并不总是*通过网关。一旦Silo了解并缓存了Grain激活的位置（无论在哪个集群），它就直接向该Silo发送消息，即使该Silo不是集群网关。

### Gossip

Gossip是一种集群间共享配置和状态信息的机制。顾名思义，Gossip（流言）是分散的和双向的：每个Silo直接与同一集群和其他集群中的其他Silo进行通信，以双向交换信息。

##### 内容
Gossip包含以下部分或全部信息：
- 当前有时间戳的[多集群配置](MultiClusterConfiguration.md)。
- 一个包含集群网关信息的字典。键是Silo地址，值包含（1）时间戳，（2）集群ID，（3）状态，即集群是否活跃。

##### 快速和慢速传播
当一个网关改变其状态时，或者当操作员注入一个新的配置时，这个Gossip信息会立即发送到所有Silo、集群和Gossip通道。这发生得很快，但并不可靠。如果信息由于任何原因（如竞争、套接字损坏、Silo故障）而丢失，我们的定期后台Gossip会确保信息最终传达，尽管速度较慢。所有的信息最终都会传达到，并且对偶发的信息丢失和故障有很强的弹性。

所有Gossip数据都有时间戳，这确保了较新的信息会取代较旧的信息，而不考虑消息的相对时间。例如，较新的多集群配置取代较旧的配置，网关的较新信息取代网关的旧信息。关于Gossip数据表示的更多细节，请看`MultiClusterData`类。它有一个`Merge`方法来组合Gossip数据，并使用时间戳解决冲突。

### Gossip通道

当Silo第一次启动，或在失败后重新启动时，它需要有一种方法来**启动Gossip**。这就是*Gossip通道*的作用，可以在[Silo配置](SiloConfiguration.md)中配置。在启动时，Silo会从Gossip通道获取所有信息。启动后，Silo保持定期Gossip，以30秒（可通过`BackgroundGossipInterval`进行配置）为间隔。每次它都会与一个从所有集群网关和Gossip通道中随机选择的伙伴同步其Gossip信息。

注意:  
- 虽然不严格要求，但我们建议始终在不同的地区配置至少两个Gossip通道，以提高可用性。
- 同Gossip通道进行通信的延迟并不关键。
- 只要ServiceId Guid（由其各自的配置指定）是不同的，多个不同的服务可以不受干扰地使用同一个Gossip通道。
- 不严格要求所有Silo使用相同的Gossip通道，只要这些通道足以让一个Silo在启动时与“Gossip社区”初步连接即可。但是，如果一个Gossip通道不是Silo配置的一部分，而该Silo是一个网关，那么它不会将其状态更新推送到通道（快速传播），所以可能需要更长的时间才能通过定期的后台Gossip（慢速传播）抵达通道。

#### 基于Azure Table的Gossip通道

我们已经实现了一个基于Azure表的Gossip通道。该配置指定了用于Azure账户的标准连接字符串。例如，一个配置可以指定两个具有独立Azure存储账户`usa`和`europe`的Gossip通道，如下所示:

```csharp
var silo = new SiloHostBuilder()
  [...]
  .Configure<MultiClusterOptions>(options => 
  {
    [...]
    options.GossipChannels.Add("AzureTable", "DefaultEndpointsProtocol=https;AccountName=usa;AccountKey=...");
    options.GossipChannels.Add("AzureTable", "DefaultEndpointsProtocol=https;AccountName=europe;AccountKey=...")
    [...]
  })
  [...]
```

只要它们各自的配置所指定的ServiceId GUID是不同的，多个不同的服务就可以使用同一个Gossip通道而互不干扰。

#### 其他Gossip通道的实现

我们正在开发其他Gossip通道提供者，类似于为许多不同的存储后端打包的提醒器。 
---
title: 全局单例Grains
---

### Grain协调特性

Developers can indicate when and how clusters should coordinate their grain directories with respect to a particular grain class. The  `[GlobalSingleInstance]` attribute means we want the same behavior as when running Orleans in a single global cluster: that is, route all calls to a single activation of the grain. Conversely, the `[OneInstancePerCluster]` attribute indicates that each cluster can have its own independent activation. This is appropriate if communication between clusters is undesired.
开发者可以指定集群何时以及如何协调他们的Grain目录与特定的Grain类。`[GlobalSingleInstance]`属性意味着我们希望与在单个全局集群中运行Orleans时的行为相同：即把所有调用路由到Grain的单一激活。相反，`[OneInstancePerCluster]`属性表示每个集群可以有自己的独立激活。这适用于不期望集群之间的通信的情况。

这些特性需要加注在Grain实现上。例如：
```csharp
using Orleans.MultiCluster;

[GlobalSingleInstance]
public class MyGlobalGrain : Orleans.Grain, IMyGrain {
   ...
}

[OneInstancePerCluster]
public class MyLocalGrain : Orleans.Grain, IMyGrain {
   ...
}
```

如果一个Grain类没有指定这些特性中的任何一个，那它默认为`[OneInstancePerCluster]`，如果配置参数`UseGlobalSingleInstanceByDefault`被设置为`true`，则为`[GlobalSingleInstance]`。

#### 用于全局单例Grain的协议

当访问一个全局单例（GSI）Grain时，如果没有已知的激活，在激活一个新的实例之前会执行一个特殊的GSI激活协议。具体来说，会向当前[多集群配置](MultiClusterConfiguration.md)中的所有其他集群发送一个请求，以检查它们是否已经为该Grain进行了激活。如果所有的回应都是否定的，那么在这个集群中就会创建一个新的激活。否则，将使用远程激活（并将其引用缓存在本地目录中）。

#### 用于每个集群一个实例（One-Instance-Per-Cluster）的Grain的协议

对于每个集群一个实例的Garin，则不发生集群间的通信。它们只会在每个集群内独立使用标准的Orleans机制。注意，在Orleans框架内部，以下Grain类被标记为`[OneInstancePerCluster]`属性：`ManagementGrain`，`GrainBasedMembershipTable`和`GrainBasedReminderTable`。

#### 可疑激活

如果GSI协议在3次重试（可由配置参数`GlobalSingleInstanceNumberRetries`指定）后没有收到来自所有集群的结论性的响应，它将乐观地创建一个新的本地“可疑”激活，这倾向于可用性而非一致性。

可疑激活可能是重复的（因为一些在GSI协议激活期间没有响应的远程集群可能仍然有这个Grain的激活）。因此，每隔30秒（可由配置参数`GlobalSingleInstanceRetryInterval`指定）定期对所有可疑激活再次执行GSI协议。这确保了一旦集群之间的通信恢复，重复的激活可以被检测并删除。

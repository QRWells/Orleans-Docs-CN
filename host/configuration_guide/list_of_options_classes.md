---
title: 选项类列表
---

所有用于配置Orleans的选项类都在`Orleans.Configuration`命名空间下。很多都在`Orleans.Hosting`命名空间中有辅助方法。

## IClientBuilder和ISiloHostBuilder的通用核心选项

| 选项                           | 用途                                             |
| ------------------------------ | ------------------------------------------------ |
| `ClusterOptions`               | 设置`ClusterId`和`ServiceId`                     |
| `NetworkingOptions`            | 为套接字和打开的连接设置超时时间                 |
| `SerializationProviderOptions` | 设置序列化提供者                                 |
| `TypeManagementOptions`        | 设置类型映射的刷新周期（见异构Silo和版本管理）。 |

## IClientBuilder特有选项

| 选项                               | 用途                                         |
| ---------------------------------- | -------------------------------------------- |
| `ClientMessagingOptions`           | 设置保持开放的连接数，并指定使用何种网络接口 |
| `ClientStatisticsOption`           | 设置与统计数据输出有关的各种设置             |
| `GatewayOptions`                   | 设置可用网关列表的刷新周期                   |
| `StaticGatewayListProviderOptions` | 设置客户端用来连接到集群的URI                |

## ISiloHostBuilder特有选项

| Option type                  | 用于                                                 |
| ---------------------------- | ---------------------------------------------------- |
| `ClusterMembershipOptions`   | 集群成员的设置                                       |
| `ConsistentRingOptions`      | 一致散列算法的配置选项，用于平衡整个集群的资源分配。 |
| `EndpointOptions`            | 设置Silo终结点选项                                   |
| `GrainCollectionOptions`     | 用于Grain垃圾回收的选项                              |
| `GrainVersioningOptions`     | 管理异构部署中的Grain实现的选择                      |
| `LoadSheddingOptions`        | 负载削减配置的设置。必须有`IHostEnvironmentStatistics`的已注册的实现，例如通过`builder.UsePerfCounterEnvironmentStatistics()`（仅适用于Windows），才能使`LoadShedding`发挥作用。    |
| `MultiClusterOptions`        | 配置多集群支持的选项                                 |
| `PerformanceTuningOptions`   | 性能调整选项（网络、线程数）。                       |
| `ProcessExitHandlingOptions` | 配置进程退出时Silo的行为                             |
| `SchedulingOptions`          | 配置调度器行为                                       |
| `SiloMessagingOptions`       | 配置与Silo相关的全局消息传递选项。                   |
| `SiloOptions`                | 设置Silo名称                                         |
| `SiloStatisticsOptions`      | 设置与统计数据输出有关的各种设置                     |
| `TelemetryOptions`           | 设置遥测消费者设置                                   |



















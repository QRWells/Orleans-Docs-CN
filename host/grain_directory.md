---
title: Grain目录
---

## 什么是Grain目录？

Grain有稳定的逻辑标识，在应用程序的生命周期中可能会被激活（实例化）和停用很多次，但在任何时间点上最多存在一个激活的Grain。
每次Grain被激活时，它可能被放在集群中的不同Silo上。
当一个Grain在集群中被激活时，它将被注册到全局注册表，即Grain目录。
这确保了该Grain的后续调用将被交付给该Grain的激活，并且该Grain的其他激活（实例）将不会被创建。
Grain目录负责保持Grain标识及其当前激活位置（Silo）之间的映射。

默认情况下，Orleans使用一个内置的分布式内存目录。
这个目录最终是一致的，并以分布式哈希表的形式在集群的所有Silo中进行分区。

从3.2.0版本开始，Orleans也支持Grain目录的可插拔实现。

在3.2.0版本中包含了两个这样的插件。

- 一个Azure表的实现。`Microsoft.Orleans.GrainDirectory.AzureStorage`(beta)
- 一个Redis存储实现。`Microsoft.Orleans.GrainDirectory.Redis`(beta)

你可以在每个Grain类型的基础上配置要使用的Grain目录实现，你甚至可以注入你自己的实现。

## 你该使用哪个Grain目录？

我们建议总是从默认的开始（内置的内存分布式目录）。
尽管它最终是一致的，并且在集群不稳定时允许偶尔的重复激活，但内置目录是自给自足的，没有外部依赖，不需要任何配置，并且一直在生产环境中使用。
当你对Orleans有一些经验，并且对Grain目录有额外需求，比如，有更强的单次激活保证，和/或想尽量减少集群中的Silo关闭时被停用的Grain数量，可以考虑使用基于存储的Grain目录的实现，如Redis的实现。
首先尝试将其用于一个或几个Grain类型，可以从那些寿命长、有大量状态或昂贵的初始化过程的Grain开始。

## 配置

### 默认Grain目录配置

你不需要做任何事情；内存中的Grain目录将被自动启用，并在集群中进行分区。

### 非默认Grain目录配置

你需要通过Grain类的一个特性来指定要使用的目录插件的名称，并在Silo配置过程中用这个名称注入目录插件。

#### Grain配置

用`GrainDirectory`特性指定Grain目录的插件名称：

```csharp
[GrainDirectory(GrainDirectoryName = "my-grain-directory")]
public class MyGrain : Grain, IMyGrain
{
    [...]
}
```

#### Silo配置

这里我们配置Redis Grain目录的实现：

```csharp
siloBuilder.AddRedisGrainDirectory(
    "my-grain-directory",
    options => options.ConfigurationOptions = redisConfiguration);
```

Azure Grain目录可以像这样配置:

```csharp
siloBuilder.AddAzureTableGrainDirectory(
    "my-grain-directory",
    options => options.ConnectionString =  = azureConnectionString);
```

你可以配置多个具有不同名称的目录，用于不同的Grain类别：

```csharp
siloBuilder
    .AddRedisGrainDirectory(
        "redis-directory-1",
        options => options.ConfigurationOptions = redisConfiguration1)
    .AddRedisGrainDirectory(
        "redis-directory-2",
        options => options.ConfigurationOptions = redisConfiguration2)
    .AddAzureTableGrainDirectory(
        "azure-directory",
        options => options.ConnectionString =  = azureConnectionString);
```
---
title: 部署新版本的Grain
description:
---

### 滚动升级

使用这种方法，你可以直接在你的环境上部署较新的Silo。
这是最简单的方法，但要中断正在进行的部署以及回滚可能很困难。

推荐的配置:
- `DefaultCompatibilityStrategy`设置为`BackwardCompatible`
- `DefaultVersionSelectorStrategy`设置为`AllCompatibleVersions`

```csharp
var silo = new SiloHostBuilder()
  [...]
  .Configure<GrainVersioningOptions>(options => 
  {
    options.DefaultCompatibilityStrategy = nameof(BackwardCompatible);
    options.DefaultVersionSelectorStrategy = nameof(AllCompatibleVersions);
  })
  [...]
```

当使用这种配置时，"老"客户端将能够与两个版本的Silo上的激活进行对话。
而较新的客户端和Silo将只触发较新的Silo上的新激活。

![Rolling gif](rolling.gif)

### 使用staging环境

In this method you will need a second environment (Staging environment),
on which you will deploy newer silos before stopping the Production environment.
The Production and the Staging silos and clients will be __part of the same
cluster__. It is important that silos from both environment can talk to each other.
使用这种方法，你将需要第二个环境（staging环境），在停止生产环境之前，你将在该环境中部署较新的Silo。
生产和staging环境中的Silo和客户端将是 __同一个集群的一部分__。重要的是，两个环境中的Silo都能相互对话。

推荐的配置:
- `DefaultCompatibilityStrategy`设置为`BackwardCompatible`
- `DefaultVersionSelectorStrategy`设置为`MinimumVersion`

```csharp
var silo = new SiloHostBuilder()
  [...]
  .Configure<GrainVersioningOptions>(options => 
  {
    options.DefaultCompatibilityStrategy = nameof(BackwardCompatible);
    options.DefaultVersionSelectorStrategy = nameof(MinimumVersion);
  })
  [...]
```

建议的部署步骤：

1. "V1" Silo和客户端已经部署，并在生产环境中运行。
2. "V2" Silo和客户开始在Staging环境启动。他们将加入与生产环境相同的集群。到目前为止，仍未创建"V2"的激活。
3. 一旦在Staging环境中的部署完成，开发者可以将一些流量重定向到V2客户端（灰度测试，面向测试用户等）。这将创建V2激活，但由于Grains是向后兼容的，而且所有Silo都在同一个集群中，因此不会创建重复的激活。
4. 如果验证成功，就可以继续进行VIP swap。否则，你可以安全地关闭Staging集群：现有的V2激活将被销毁，如果需要，V1激活将被创建。
5. V1的激活最终会自然"迁移"到V2的Silo。你可以安全地关闭V1Silo。

> [!警告]
> 请记住，无状态worker是没有版本的，流的代理也将在暂存环境中启动。

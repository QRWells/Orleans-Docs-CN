---
description: Grain Placement
---

# Grain安置

Orleans保证当一个Grain被调用时，集群中的某些服务器的内存中有可用的该Grain的实例用以处理请求。
如果当前集群中Grain没有激活，Orleans会选择一个服务器来激活该Grain。这称为Grain安置。
安置也是均衡负载的一种方式：繁忙Grains的均匀安置有助于平衡整个集群的工作负载。

Orleans的安置过程是完全可配置的：开发者可以从一组开箱即用的放置策略中选择，如随机、偏好本地和基于负载，也可以配置自定义逻辑。
这使我们可以完全灵活地决定Grains的创建位置。
例如，Grains可以被安置在靠近它们需要操作的资源的服务器上，或者靠近与它们通信的其他Grains。
默认情况下，Orleans会选择一个随机的兼容的服务器。

Orleans使用的安置策略可以全局配置，也可以按Grain类配置。

## 内置的安置策略

### 随机安置

从兼容的服务器中随机选择一个服务器。

这个安置策略可以通过向Grain添加`[RandomPlacement]`特性来配置。

### 本地安置

如果本地服务器兼容，就选择本地服务器，否则随机选择一个服务器。

这个安置策略可以通过向Grain添加`[PreferLocalPlacement]`特性来配置。

### 基于哈希的安置

将Grain的ID哈希为一个非负整数，然后对兼容的服务器的数量取模，
然后从按地址排列的兼容服务器列表中选择相应的服务器。
注意，这并不保证在集群成员变化时仍然保持稳定。
具体来说，添加、删除或重新启动服务器可以改变给定Grain ID所对应的服务器。
因为使用这种策略安置的Grains是在Grain目录中注册的，这种随着成员变化而变化的安置选择通常不会有明显的影响。

这个安置策略可以通过向Grain添加`[HashBasedPlacement]`特性来配置。

### 基于激活数量的安置

这个安置策略的目的，是根据最近繁忙的Grain的数量，将新的Grain激活放在负载最小的服务器上。
它包含一个机制，即所有服务器定期向所有其他服务器发布其总激活数。
然后，安置管理器通过检查最近报告的激活数量，以及根据安置管理器对当前服务器的最近激活计数进行的预测，选择一个预计激活数最少的服务器。
在进行预测时，管理器会随机选择一些服务器，以避免多个独立的服务器使同一服务器过载。
默认情况下，会随机选择两个服务器，但这个数可以通过`ActivationCountBasedPlacementOptions`进行配置。

这个算法是基于Michael David Mitzenmacher的论文[*The Power of Two Choices in Randomized Load Balancing*](https://www.eecs.harvard.edu/~michaelm/postscripts/mythesis.pdf)，并且也在Nginx中用于分布式负载均衡，如文章[*NGINX and the "Power of Two Choices" Load-Balancing Algorithm*](https://www.nginx.com/blog/nginx-power-of-two-choices-load-balancing-algorithm/)中所述。

这个安置策略可以通过向Grain添加`[ActivationCountBasedPlacement]`特性来配置。

### 无状态worker的安置

这是[*无状态的worker* grains](stateless_worker_grains.md)使用的特殊安置材策略。
这与本地安置的操作几乎相同，只是每个服务器可以有同一个的Grain的多个激活，并且Grains不在Grain目录中注册，因为没有必要。

这个安置策略可以通过向Grain添加`[StatelessWorker]`特性来配置。

## 配置默认安置策略

Orleans将使用随机安置，除非默认值被覆盖。
默认的安置策略可以通过在配置过程中注册一个`PlacementStrategy`的实现来覆盖：

``` csharp
siloBuilder.ConfigureServices(services =>
  services.AddSingleton<PlacementStrategy, MyPlacementStrategy>());
```

## 为一个Grain配置默认安置策略

Grain类的安置策略是通过在Grain类上添加特性来配置的。
相关的属性如[内置的安置策略](#内置的安置策略)部分所述。

## 自定义安置策略的示例

首先定义一个实现了`IPlacementDirector`接口的类，它需要实现一个方法。
在这个例子中，我们假设你已经定义了一个函数`GetSiloNumber`，对于将要创建的Grain的GUID，它将返回Silo的编号。

``` csharp
public class SamplePlacementStrategyFixedSiloDirector : IPlacementDirector
{

    public Task<SiloAddress> OnAddActivation(PlacementStrategy strategy, PlacementTarget target, IPlacementContext context)
    {
        var silos = context.GetCompatibleSilos(target).OrderBy(s => s).ToArray();
        int silo = GetSiloNumber(target.GrainIdentity.PrimaryKey, silos.Length);
        return Task.FromResult(silos[silo]);
    }
}
```

接下来，为了把Grain类分配到策略中，你需要定义两个类：

```csharp
[Serializable]
public class SamplePlacementStrategy : PlacementStrategy
{
}

[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public sealed class SamplePlacementStrategyAttribute : PlacementAttribute
{
    public SamplePlacementStrategyAttribute() :
        base(new SamplePlacementStrategy())
        {
        }
}
```

然后，你只需要给想使用这个策略的任意Grain类加上特性注解：

``` csharp
[SamplePlacementStrategy]
public class MyGrain : Grain, IMyGrain
{
    ...
}
```

最后，在你建立Silo时注册该策略：

``` csharp
private static async Task<ISiloHost> StartSilo()
{
    ISiloHostBuilder builder = new SiloHostBuilder()
        // normal configuration methods omitted for brevity
        .ConfigureServices(ConfigureServices);

    var host = builder.Build();
    await host.StartAsync();
    return host;
}


private static void ConfigureServices(IServiceCollection services)
{
    services.AddSingletonNamedService<PlacementStrategy, SamplePlacementStrategy>(nameof(SamplePlacementStrategy));
    services.AddSingletonKeyedService<Type, IPlacementDirector, SamplePlacementStrategyFixedSiloDirector>(typeof(SamplePlacementStrategy));
}
```

关于另一个展示进一步使用安置上下文的简单例子，请参考[Orleans源代码](https://github.com/dotnet/orleans/blob/master/src/Orleans.Runtime/Placement/PreferLocalPlacementDirector.cs)中的`PreferLocalPlacementDirector`。

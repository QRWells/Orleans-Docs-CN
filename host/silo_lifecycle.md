---
title: Silo生命周期
---

# Silo生命周期

# 总览

Orleans silo使用一个可观察（observable）的生命周期（见[Orleans生命周期](/implementation/orleans_lifecycle.md)）来有序地启动和关闭Orleans系统以及应用层组件。

## 生命周期的阶段

Orleans Silo和集群客户端使用一套共同的服务生命周期的阶段。

```csharp
public static class ServiceLifecycleStage
{
    public const int First = int.MinValue;
    public const int RuntimeInitialize = 2000;
    public const int RuntimeServices = 4000;
    public const int RuntimeStorageServices = 6000;
    public const int RuntimeGrainServices = 8000;
    public const int ApplicationServices = 10000;
    public const int BecomeActive = Active-1;
    public const int Active = 20000;
    public const int Last = int.MaxValue;
}
```

- 起始 - 服务生命周期的起始阶段
- 运行时初始化 - 初始化运行时环境。Silo初始化线程。
- 运行时服务 - 启动运行时服务。Silo初始化网络和各种代理。
- 运行时存储服务 - 初始化运行时存储。
- 运行时Grain服务 - 启动Grain的运行时服务。 这包括Grain类型管理、成员服务和Grain目录。
- 应用服务 – 应用层服务。
- 预激活 – Silo加入集群。
- 激活 – Silos在集群中处于活动状态，准备接受工作负载。
- 最终 - 服务生命周期的最终阶段

## 记录日志

由于控制反转，导致这个过程是参与者加入生命周期，而不是生命周期进行一些集中的初始化步骤，所以并非总能从代码中明确启动/关闭的顺序如何。
为了解决这个问题，在Silo启动前增加了日志记录，以报告每个阶段有哪些组件在参与。
这些日志以信息级日志被记录在`Orleans.Runtime.SiloLifecycleSubject`日志器上。例如：

_Information, Orleans.Runtime.SiloLifecycleSubject, “Stage 2000: Orleans.Statistics.PerfCounterEnvironmentStatistics, Orleans.Runtime.InsideRuntimeClient, Orleans.Runtime.Silo”_

_Information, Orleans.Runtime.SiloLifecycleSubject, “Stage 4000: Orleans.Runtime.Silo”_

_Information, Orleans.Runtime.SiloLifecycleSubject, “Stage 10000: Orleans.Runtime.Versions.GrainVersionStore, Orleans.Storage.AzureTableGrainStorage-Default, Orleans.Storage.AzureTableGrainStorage-PubSubStore”_

此外，每个组件的时间和错误信息也同样按阶段进行记录。例如：

_Information, Orleans.Runtime.SiloLifecycleSubject, “Lifecycle observer Orleans.Runtime.InsideRuntimeClient started in stage 2000 which took 33 Milliseconds.”_

_Information, Orleans.Runtime.SiloLifecycleSubject, “Lifecycle observer Orleans.Statistics.PerfCounterEnvironmentStatistics started in stage 2000 which took 17 Milliseconds.”_

## 参与Silo生命周期

应用逻辑可以通过在Silo的服务容器中注册一个参与服务来参与Silo的生命周期。 该服务必须被注册为`ILifecycleParticipant<ISiloLifecycle>`。

```csharp
public interface ISiloLifecycle : ILifecycleObservable
{
}

public interface ILifecycleParticipant<TLifecycleObservable>
    where TLifecycleObservable : ILifecycleObservable
{
    void Participate(TLifecycleObservable lifecycle);
}
```

在Silo启动时，容器中的所有参与者（`ILifecycleParticipant<ISiloLifecycle>`）将有机会通过调用他们的`Participate(.)`行为来参与生命周期。
一旦所有参与者都可以参与，Silo的可观察生命周期将按顺序启动所有阶段。

## 示例

随着Silo生命周期的引入，引导提供者变得不再必要，它在过去用于在初始化阶段注入逻辑，因为现在可以在Silo启动的任何阶段注入应用逻辑。
尽管如此，我们还是添加了一个“启动任务”封装，以帮助那些一直使用引导提供者的开发者进行过渡。
作为一个如何开发参与Silo生命周期的组件的例子，我们来看看启动任务的封装：

启动任务只需要继承自`ILifecycleParticipant<ISiloLifecycle>`，并在指定阶段将应用逻辑订阅给Silo生命周期。

```csharp
class StartupTask : ILifecycleParticipant<ISiloLifecycle>
{
    private readonly IServiceProvider serviceProvider;
    private readonly Func<IServiceProvider, CancellationToken, Task> startupTask;
    private readonly int stage;

    public StartupTask(
        IServiceProvider serviceProvider,
        Func<IServiceProvider, CancellationToken, Task> startupTask,
        int stage)
    {
        this.serviceProvider = serviceProvider;
        this.startupTask = startupTask;
        this.stage = stage;
    }

    public void Participate(ISiloLifecycle lifecycle)
    {
        lifecycle.Subscribe<StartupTask>(
            this.stage,
            cancellation => this.startupTask(this.serviceProvider, cancellation));
    }
}
```

从上面的实现中，我们可以看到，在启动任务的`Participate(..)`调用中，它在指定的阶段订阅了Silo的生命周期，传入应用程序的回调，而非自己的初始化逻辑。

需要在特定阶段初始化的组件会提供自己的回调，但模式是一样的。

现在我们有了一个启动任务，它将确保应用程序的钩子在指定的阶段被调用，我们需要确保启动任务参与到Silo的生命周期中。
为此，我们只需要在容器中注册它。

我们通过`SiloHost Builder`上的一个扩展函数来实现：

```csharp
public static ISiloHostBuilder AddStartupTask(
    this ISiloHostBuilder builder,
    Func<IServiceProvider, CancellationToken, Task> startupTask,
    int stage = ServiceLifecycleStage.Active)
{
    builder.ConfigureServices(services =>
        services.AddTransient<ILifecycleParticipant<ISiloLifecycle>>(sp =>
            new StartupTask(
                sp,
                startupTask,
                stage)));
    return builder;
}
```

通过在Silo的服务容器中注册启动任务来作为标记接口`ILifecycleParticipant<ISiloLifecycle>`，就会向Silo发出信号，表明这个组件需要参与Silo的生命周期。
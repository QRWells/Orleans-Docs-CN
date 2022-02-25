---
title: Orleans生命周期
---

## 概述

Orleans的一些行为足够复杂，需要有序的启动和关闭。
一些具有这种行为的组件包括Grain、Silo和客户端。
为了解决这个问题，我们引入了一个通用的组件生命周期模式。
该模式由一个可观察的生命周期和生命周期观察者组成，前者负责对组件的启动和关闭阶段发出信号，后者负责在特定阶段执行启动或关闭操作。

也可参见[Grain生命周期](../grains/grain_lifecycle.md)和[Silo生命周期](../host/silo_lifecycle.md)。

## 可观察生命周期

需要有序启动和关闭的组件可以使用一个可观察的生命周期，它允许其他组件观察其生命周期，并在启动或关闭期间的某个阶段接收通知。

```csharp
    public interface ILifecycleObservable
    {
        IDisposable Subscribe(string observerName, int stage, ILifecycleObserver observer);
    }
```

订阅调用注册了一个观察者，以便在启动或停止期间到达某个阶段时进行通知。观察者的名称用于报告。“阶段”表示在启动/关闭序列中的何处，观察者将被通知。生命周期的每个阶段都是可观察的。在启动和停止期间到达该阶段时，所有观察者都会收到通知。各个阶段是按升序启动，降序停止的。观察者可以通过释放返回的`IDisposable`来取消订阅。

## 生命周期观察者

需要参与另一个组件的生命周期的组件需要为其启动和关闭行为提供钩子，并订阅可观察的生命周期的某个阶段。

```csharp
    public interface ILifecycleObserver
    {
        Task OnStart(CancellationToken ct);
        Task OnStop(CancellationToken ct);
    }
```

`OnStart/OnStop`将在启动/关闭过程期间到达订阅的阶段时被调用。

## 辅助工具

为方便起见，我们为常见的生命周期使用模式创建了帮助函数。

### 扩展函数

我们提供了用于订阅可观察生命周期的扩展函数，它不要求订阅组件实现`ILifecycleObserver`。相反，这些函数允许组件传入lambda表达式或成员函数，以便在订阅阶段调用。

```csharp
IDisposable Subscribe(this ILifecycleObservable observable, string observerName, int stage, Func<CancellationToken, Task> onStart, Func<CancellationToken, Task> onStop);

IDisposable Subscribe(this ILifecycleObservable observable, string observerName, int stage, Func<CancellationToken, Task> onStart);
```

类似的扩展函数允许使用泛型参数来代替观察者名称。

```csharp
IDisposable Subscribe<TObserver>(this ILifecycleObservable observable, int stage, Func<CancellationToken, Task> onStart, Func<CancellationToken, Task> onStop);

IDisposable Subscribe<TObserver>(this ILifecycleObservable observable, int stage, Func<CancellationToken, Task> onStart);
```

### 参与生命周期

一些可扩展点需要一种方法来识别哪些组件对参与生命周期感兴趣。为此，我们引入了一个生命周期参与者标记接口。在探索Silo和Grain生命周期时，将涉及更多关于如何使用这个接口的内容。

```csharp
    public interface ILifecycleParticipant<TLifecycleObservable>
        where TLifecycleObservable : ILifecycleObservable
    {
        void Participate(TLifecycleObservable lifecycle);
    }
```

## 示例

从我们的生命周期测试来看，下面是一个组件在生命周期的多个阶段参加可观察的生命周期的例子：

```csharp
enum TestStages
{
    Down,
    Initialize,
    Configure,
    Run,
}

class MultiStageObserver : ILifecycleParticipant<ILifecycleObservable>
{
    public Dictionary<TestStages,bool> Started { get; } = new Dictionary<TestStages, bool>();
    public Dictionary<TestStages, bool> Stopped { get; } = new Dictionary<TestStages, bool>();

    private Task OnStartStage(TestStages stage)
    {
        this.Started[stage] = true;
        return Task.CompletedTask;
    }

    private Task OnStopStage(TestStages stage)
    {
        this.Stopped[stage] = true;
        return Task.CompletedTask;
    }

    public void Participate(ILifecycleObservable lifecycle)
    {
        lifecycle.Subscribe<MultiStageObserver>((int)TestStages.Down, ct => OnStartStage(TestStages.Down), ct => OnStopStage(TestStages.Down));
        lifecycle.Subscribe<MultiStageObserver>((int)TestStages.Initialize, ct => OnStartStage(TestStages.Initialize), ct => OnStopStage(TestStages.Initialize));
        lifecycle.Subscribe<MultiStageObserver>((int)TestStages.Configure, ct => OnStartStage(TestStages.Configure), ct => OnStopStage(TestStages.Configure));
        lifecycle.Subscribe<MultiStageObserver>((int)TestStages.Run, ct => OnStartStage(TestStages.Run), ct => OnStopStage(TestStages.Run));
    }
}
```


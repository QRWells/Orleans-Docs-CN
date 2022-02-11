---
title: Grain生命周期
description: 
---

## 总览

Orleans grains使用可观察的生命周期（见[Orleans生命周期](../implementation/orleans_lifecycle.md)）进行有序激活和停用。
这使Grain逻辑、系统组件和应用逻辑在Grain激活和回收期间得以有序地启动和停止。
### 阶段

预定义的Grain生命周期阶段如下所示。

```csharp
public static class GrainLifecycleStage
{
    public const int First = int.MinValue;
    public const int SetupState = 1000;
    public const int Activate = 2000;
    public const int Last = int.MaxValue;
}
```

- `First` - Grain生命周期的最初阶段
- `SetupState` – 在激活之前，设置Grain状态。对于有状态的Grain，这个阶段会从存储中加载状态。
- `Activate` – 在此阶段`OnActivateAsync`与`OnDeactivateAsync`会被调用
- `Last` - Grain生命周期的最后阶段

虽然Grain生命周期将在Grain激活期间使用，但由于Grain在某些错误情况下并不总是被停用（如Silo崩溃），应用不应假设Grain生命周期总会在Grain停用期间被执行。

### 参与Grain生命周期

应用逻辑可以通过两种方式参与Grain的生命周期。
Grain可以参与其自身的生命周期，以及（或）组件可以通过Grain激活上下文来访问生命周期（见`IGrainActivationContext.ObservableLifecycle`）。

Grain总是参与自身的生命周期，因此可以通过重写`participate`方法引入应用逻辑。

### 示例1

```csharp
public override void Participate(IGrainLifecycle lifecycle)
{
    base.Participate(lifecycle);
    lifecycle.Subscribe(this.GetType().FullName, GrainLifecycleStage.SetupState, OnSetupState);
}
```

在上面的例子中，`Grain<T>`重写了`Participate`方法，告诉生命周期在`SetupState`阶段调用其`OnSetupState`方法。

在Grain构造过程中创建的组件也可以参与到生命周期中，不需要任何额外的Grain逻辑。
由于Grain的激活上下文（`IGrainActivationContext`），包括Grain的生命周期（`IGrainActivationContext.ObservableLifecycle`），是在Grain被创建之前创建的，任何由容器注入Grain的组件都可以参与Grain的生命周期。

### 示例2

下面的组件在使用其工厂函数`Create(..)`创建时参与了Grain的生命周期。
这个逻辑可以存在于组件的构造函数中，但这有可能使组件在完全构造之前就被加入到生命周期中，可能是不安全的。

```csharp
public class MyComponent : ILifecycleParticipant<IGrainLifecycle>
{
    public static MyComponent Create(IGrainActivationContext context)
    {
        var component = new MyComponent();
        component.Participate(context.ObservableLifecycle);
        return component;
    }

    public void Participate(IGrainLifecycle lifecycle)
    {
        lifecycle.Subscribe<MyComponent>(GrainLifecycleStage.Activate, OnActivate);
    }

    private Task OnActivate(CancellationToken ct)
    {
        // Do stuff
    }
}
```

通过使用`Create(..)`工厂函数在服务容器中注册上述组件，任何以该组件为依赖所构造的Grain将有该组件参与其生命周期，而不需要Grain中的任何特殊逻辑。

#### 在容器中注册组件

```csharp
    services.AddTransient<MyComponent>(sp =>
        MyComponent.Create(sp.GetRequiredService<IGrainActivationContext>());
```

#### 以组件作为依赖的Grain

```csharp
public class MyGrain : Grain, IMyGrain
{
    private readonly MyComponent component;

    public MyGrain(MyComponent component)
    {
        this.component = component;
    }
}
```

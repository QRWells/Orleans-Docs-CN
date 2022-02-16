---
title: 开发一个Grain
description: 本节将讲述开发一个Grain所需的基本知识
---

## 基本设置

在你写代码实现Grain类之前，先创建一个新的类库项目，
目标框架可以是.NET Standard或.Net Core（首选）或.NET Framework 4.6.1或更高（如果你由于依赖关系不能使用.NET Standard或.NET Core）。
Grain接口和Grain类可以定义在同一个类库项目中，也可以定义在两个不同的项目中，以便更好地将接口和实现分开。
无论哪种情况，项目都需要引用`Microsoft.Orleans.Core.Abstractions`和`Microsoft.Orleans.CodeGenerator.MSBuild`这两个NuGet包。

更详细的说明，请参见[教程 1 - Orleans 基础](/tutorials_and_samples/tutorial_1.md)
里的[项目设置](/tutorials_and_samples/tutorial_1.md#project-setup)部分。

## Grain接口和类

通过在Grains各自的接口中声明的方法，Grains之间可以进行交互，以及从外部被调用。
一个Grain类实现了一个或多个预先声明的Grain接口。
Grain接口里的方法的返回值应当是`Task`（对于`void`方法）、`Task<T>`或`ValueTask<T>`（对于返回`T`类型值的方法）。

下面的代码来自Orleans 1.5版本的Presence Service示例：

```csharp
//一个Grain接口的例子
public interface IPlayerGrain : IGrainWithGuidKey
{
  Task<IGameGrain> GetCurrentGame();
  Task JoinGame(IGameGrain game);
  Task LeaveGame(IGameGrain game);
}

//一个实现了Grain接口的Grain类例子
public class PlayerGrain : Grain, IPlayerGrain
{
    private IGameGrain currentGame;

    // Player现在所处的Game。可能是null.
    public Task<IGameGrain> GetCurrentGame()
    {
       return Task.FromResult(currentGame);
    }

    // Game的Grain通过调用这个方法来通知玩家加入游戏。
    public Task JoinGame(IGameGrain game)
    {
       currentGame = game;
       Console.WriteLine(
           "Player {0} joined game {1}",
           this.GetPrimaryKey(),
           game.GetPrimaryKey());

       return Task.CompletedTask;
    }

   // Game的Grain通过调用这个方法来通知玩家离开游戏。
   public Task LeaveGame(IGameGrain game)
   {
       currentGame = null;
       Console.WriteLine(
           "Player {0} left game {1}",
           this.GetPrimaryKey(),
           game.GetPrimaryKey());

       return Task.CompletedTask;
   }
}
```

## 从Grain方法中返回值

一个返回`T`类型值的Grain方法，在Grain接口中被定义为返回`Task<T>`。
对于没有标注`async`关键字的Grain方法，当返回值可用时，通常会通过以下语句返回。

```csharp
public Task<SomeType> GrainMethod1()
{
    ...
    return Task.FromResult(<表示结果的变量或常量>);
}
```

一个没有返回值的Grain方法，或者说void方法，在Grain接口中被定义为返回一个`Task`。
返回的`Task`表示方法的异步执行和完成。
对于没有标注`async`关键字的Grain方法，当一个“void”方法执行完成时，需要返回`Task.CompletedTask`的特殊值：

```csharp
public Task GrainMethod2()
{
    ...
    return Task.CompletedTask;
}
```

标注有“async”的Grain方法可以直接返回值：

```csharp
public async Task<SomeType> GrainMethod3()
{
    ...
    return <表示结果的变量或常量>;
}
```

一个标注了`async`的“void”Grain方法，如果没有返回任何值，就会在其执行结束时返回：

```csharp
public async Task GrainMethod4()
{
    ...
    return;
}
```
如果一个Grain方法收到了另一个异步方法调用的返回值，无论是否是向一个Grain发起的，并且不需要对该调用进行错误处理，它可以简单地返回它从该异步调用收到的`Task`：

```csharp
public Task<SomeType> GrainMethod5()
{
    ...
    Task<SomeType> task = CallToAnotherGrain();
    return task;
}
```

类似的，一个“void”Grain方法可以返回一个，由另一个调用返回给它的`Task`，而不是等待它。

```csharp
public Task GrainMethod6()
{
    ...
    Task task = CallToAsyncAPI();
    return task;
}
```

可以用`ValueTask<T>`代替`Task<T>`。

### Grain引用

Grain引用是一个代理对象，它实现了与对于Grain类同样的Grain接口。
它封装了目标Grain的逻辑标识（类型和唯一键）。
Grain引用用于发起对目标Grain的调用。
每个Grain引用对应一个Grain（一个Grain类的实例），但是我们可以对同一个Grain创建多个独立的引用。

由于Grain引用反映了目标Grain的逻辑标识，所以它与Grain的物理位置无关，即使在系统完全重启后仍保持有效。
开发者可以像其他任何.NET对象一样使用Grain引用。
它可以被传递给一个方法，或者作为一个方法的返回值，甚至可以被保存到持久化存储器。

可以通过将Grain的标识传递给`GrainFactory.GetGrain<T>(key)`方法来获取其Grain引用，
其中`T`是Grain接口，`key`是Grain在此类型中的唯一键。

以下是如何获得上文中定义的`IPlayerGrain`接口的Grain引用的例子。

在Grain类中:

```csharp
    //构造指定player的Grain引用
    IPlayerGrain player = GrainFactory.GetGrain<IPlayerGrain>(playerId);
```

在Orleans客户端代码中.

```csharp
    IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);
```

### Grain方法的调用

Orleans的编程模型基于[异步编程](https://docs.microsoft.com/zh-cn/dotnet/csharp/async)。

使用前面例子中的Grain引用，下面是如何执行Grain方法的调用:

```csharp
//异步地调用一个Grain方法。
Task joinGameTask = player.JoinGame(this);
//await关键字使方法的其余部分在稍后的时间点被异步地执行（在被等待的任务完成后）而不阻塞线程。
await joinGameTask;
//下一行将在joinGameTask完成后执行。
players.Add(playerId);

```

可以连接两个或更多`Task`，连接操作会创建一个新的`Task`，这个`Task`会在组成它的所有`Task`都完成后被解决。
当一个Grain需要启动多个计算并等待所有的计算完成后再继续进行的时候，这会是一个有用的方法。
例如，一个Grain，其生成一个由很多部分组成的网页，对与每个部分，它都可能对后端发起一次调用，并收到包含结果的`Task`。
这个Grain会await所有这些`Task`的“连接”，当这个连接`Task`被解决后，每个`Task`都已经完成，并且已经收到了组成网页所需的所有数据。

例子:

``` csharp
List<Task> tasks = new List<Task>();
Message notification = CreateNewMessage(text);

foreach (ISubscriber subscriber in subscribers)
{
   tasks.Add(subscriber.Notify(notification));
}

// WhenAll 连接一个集合里的任务，并返回连接后的任务，该任务将在所有单独的通知任务完成后被解决。
Task joinedTask = Task.WhenAll(tasks);
await joinedTask;

// 在joinTask被解决后，此方法的剩余部分将继续异步地执行。
```

### 虚方法

Grain类可以选择重写`OnActivateAsync`和`OnDeactivateAsync`这两个虚方法，它们会在此类的每个Grain被激活和停用时被Orleans运行时所调用。
这使得我们可以在Grains的代码中执行额外的初始化和清理操作。
`OnActivateAsync`抛出的异常会导致激活失败，虽然`OnActivateAsync`（如果被重写）总是作为Grain激活过程中的一步被调用，
但`OnDeactivateAsync`并不能保证在所有情况下都会被调用，例如，在服务器故障或其他异常事件中就不会被调用。
因此，应用不应依赖`OnDeactivateAsync`来执行关键的操作，如状态的变化的持久化。
应用应该只在尽力而为操作(best-effort operations)中使用这一函数。
---
title: 观察者
description: 本节介绍了Orleans中的观察者（Observer）
---

# 观察者

在有些情况下，简单的消息/响应模式是不够的，客户端需要接收异步通知。
例如，用户可能希望在朋友发布新的即时消息时得到通知。

客户端观察者是一种可以异步地通知客户端的机制。
观察者是继承自`IGrainObserver`的单向的异步接口，其所有方法都必须是无返回值的（`void`）。
Grains通过调用观察者的方法来向其发送通知，就像调用Grain接口方法一样，只不过观察者方法没有返回值，所以Grain不依赖于其结果。
Orleans运行时会保证通知的单向传递。
发布这种通知的Grain应该提供一个API同于增加或删除观察者。
另外，通常情况下，对外暴露一个允许取消现有订阅的方法会很方便。

Grain开发者可以使用诸如[`ObserverManager<T>`](https://github.com/dotnet/orleans/blob/e997335d2d689bb39e67f6bcf6fd70862a22c02f/test/Grains/TestGrains/ObserverManager.cs#L12)这样的工具类来简化被观察的（observed）Grain类型的开发。
Grain会在故障后根据需要自动地重新激活，与之不同的是，客户端不具有容错性：故障的客户端可能永远不会恢复。
出于这个原因，`ObserverManager<T>`会在配置好的持续时间后删除订阅。
活跃的客户端应该利用定时器来重新订阅，以保持订阅的活跃。

要订阅一个通知，客户端必须首先创建一个实现了观察者接口的本地对象。
然后调用观察者工厂的一个方法`CreateObjectReference()`，将该对象变成一个Grain引用，然后可以将这个Grain引用传给要通知的Grain上的订阅方法。

其他Grains也可以用这个模型来接收异步通知。
Grains也可以实现`IGrainObserver`接口。
与客户端订阅不同，订阅的Grain只是实现了观察者接口并传入了自身的引用（例如`this.AsReference<IMyGrainObserverInterface>()`）。
因为Grain已然可寻址，所以没必要调用`CreateObjectReference()`了。

## 代码示例

假设我们有一个定期向客户端发送消息的Grain。
为了简单起见，我们的例子中的消息将是一个字符串。我们首先在客户端定义接收消息的接口。

这个接口看起来像这样：

``` csharp
public interface IChat : IGrainObserver
{
    void ReceiveMessage(string message);
}

```

最特别的是这个接口应继承自`IGrainObserver`。
现在，任何想要观察这些消息的客户端都应该有一个实现`IChat`的类。

最简单的情况是这样的：

``` csharp
public class Chat : IChat
{
    public void ReceiveMessage(string message)
    {
        Console.WriteLine(message);
    }
}
```

接下来，我们在服务器上应该有一个Grain，向客户端发送这些聊天消息。
Grain还应该有一个机制，让客户端可以自己订阅和退订通知。
对于订阅，Grain可以使用工具类[`ObserverManager<T>`](https://github.com/dotnet/orleans/blob/e997335d2d689bb39e67f6bcf6fd70862a22c02f/test/Grains/TestGrains/ObserverManager.cs#L12)。

``` csharp
class HelloGrain : Grain, IHello
{
    private readonly ObserverManager<IChat> _subsManager;

    public HelloGrain(ILogger<HelloGrain> logger)
    {
        _subsManager = new ObserverManager<IChat>(TimeSpan.FromMinutes(5), logger, "subs");
    }

    // Clients call this to subscribe.
    public Task Subscribe(IChat observer)
    {
        _subsManager.Subscribe(observer, observer);
        return Task.CompletedTask;
    }

    //Clients use this to unsubscribe and no longer receive messages.
    public Task UnSubscribe(IChat observer)
    {
        _subsManager.Unsubscribe(observer, observer);
        return Task.CompletedTask;
    }
}
```

为了向客户端发送消息，可以使用`ObserverManager<IChat>`实例的`Notify`方法。
该方法接受一个`Action<T>`方法或lambda表达式（其中`T`在这里是`IChat`类型）。
你可以调用接口上的任何方法来将消息发送给客户端。
在我们的例子中，我们只有一个方法，`ReceiveMessage`，我们在服务器上的发送代码是这样的：

``` csharp
public Task SendUpdateMessage(string message)
{
    _subsManager.Notify(s => s.ReceiveMessage(message));
    return Task.CompletedTask;
}
```

现在我们的服务器有一个向观察者客户端发送消息的方法，两个用于订阅/取消订阅的方法，而客户端已经实现了一个能够观察Grain消息的类。
最后一步是使用我们之前实现的`Chat`类在客户端上创建一个观察者引用，并在订阅它后让他接收消息。

代码如下：

``` csharp
//First create the grain reference
var friend = _grainFactory.GetGrain<IHello>(0);
Chat c = new Chat();

//Create a reference for chat, usable for subscribing to the observable grain.
var obj = await _grainFactory.CreateObjectReference<IChat>(c);

//Subscribe the instance to receive messages.
await friend.Subscribe(obj);
```

现在，每当我们服务器上的Grain调用`SendUpdateMessage`方法时，所有订阅的客户端都会收到该消息。
在我们的客户端代码中，变量`c`中的`Chat`实例将收到消息并将其输出到控制台。

**注意：** 传递给`CreateObjectReference`的对象是被一个[`WeakReference<T>`](https://docs.microsoft.com/dotnet/api/system.weakreference)持有的，因此如果没有其他的引用存在，将会被垃圾回收。
用户应该为每个不希望被收集的观察者维护一个引用。

**注意：** 观察者本质上是不可靠的，因为你不会得到任何响应，来得知消息是否被接收和处理，或者仅仅是由于分布式系统中可能出现的任何状况而故障。
正因为如此，你的观察者应该定期轮询Grains或使用任何其他机制来确保他们收到所有它们应该收到的消息。
在某些情况下，你可以忍受一些消息的丢失且不需要任何额外的机制，但是如果你需要确保所有的观察者总是能收到消息，并且能收到所有的消息，定期重新订阅和轮询观察者Grain都可以帮助确保所有消息最终得到处理。

## 执行模型

`IGrainObserver`的实现是通过调用`IGrainFactory.CreateObjectReference`来注册的，每次调用该方法都会创建一个新的引用，指向该实现。
Orleans将逐一执行发送到这些引用的请求，直到完成。
观察者是不可重入的，因此对观察者的并发请求将不会被Orleans交错。
如果有多个观察者并发地接收请求，这些请求可以并行执行。
观察者方法的执行不受诸如`[AlwaysInterleave]`或`[Reentrant]`等特性的影响：开发者不能自定义执行模型。
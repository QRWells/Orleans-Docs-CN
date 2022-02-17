---
title: Orleans流编程API
---

应用通过API与流进行交互，这些API与众所周知的[.NET响应式扩展（Rx）](https://github.com/dotnet/reactive)非常相似。主要的区别是，Orleans流扩展是**异步**的，以使Orleans的分布式和可扩展的计算结构中的处理更高效。

## 异步流

应用通过利用*流提供者*获得一个流的句柄来启动。在[这里](stream_providers.md)你可以阅读更多关于流提供者的信息，但现在你可以简单地把它看作是流的工厂，允许实现者定制流的行为和语义：

``` csharp
IStreamProvider streamProvider = base.GetStreamProvider("SimpleStreamProvider");
IAsyncStream<T> stream = streamProvider.GetStream<T>(Guid, "MyStreamNamespace");
```

应用程序可以通过调用`Grain`类中的`GetStreamProvider`方法，或者通过调用`GrainClient.GetStreamProvider()`方法，在客户端获得流提供者的引用。

[**`Orleans.Streams.IAsyncStream<T>`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncStream.cs)
是一个逻辑强类型的虚拟流的句柄。它的设计精神类似于Orleans Grain引用。对`GetStreamProvider`和`GetStream`的调用是单纯的本地调用。`GetStream`的参数是一个GUID和一个额外的字符串，我们称之为流命名空间（可以为空）。GUID和命名空间字符串共同组成了流的标识（与`GrainFactory.GetGrain`的参数相似）。GUID和命名空间字符串的组合在确定流标识时提供了额外的灵活性。就像Grain 7可能是`PlayerGrain`类型的，而另一个不同的Grain 7可能是`ChatRoomGrain`类型的，流123可能存在于流命名空间`PlayerEventsStream`中，而另一个不同的流123可能存在于流命名空间`ChatRoomMessagesStream`中。

## 生产与消费

`IAsyncStream<T>`同时实现了
[**`Orleans.Streams.IAsyncObserver<T>`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncObserver.cs)和
[**`Orleans.Streams.IAsyncObservable<T>`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncObservable.cs)接口。
这样一来，应用就可以通过`Orleans.Streams.IAsyncObserver<T>`来向流中生产事件，或通过`Orleans.Streams.IAsyncObservable<T>`来订阅或消费流中的事件。

``` csharp
public interface IAsyncObserver<in T>
{
    Task OnNextAsync(T item, StreamSequenceToken token = null);
    Task OnCompletedAsync();
    Task OnErrorAsync(Exception ex);
}

public interface IAsyncObservable<T>
{
    Task<StreamSubscriptionHandle<T>> SubscribeAsync(IAsyncObserver<T> observer);
}
```

要向流中生产事件，应用只需要调用

``` csharp
await stream.OnNextAsync<T>(event)
```

要订阅一个流，应用需要调用

``` csharp
StreamSubscriptionHandle<T> subscriptionHandle = await stream.SubscribeAsync(IAsyncObserver)
```

`SubscribeAsync`的参数可以是一个实现`IAsyncObserver`接口的对象，或者是一个组合的
lambda函数来处理传入的事件。`SubscribeAsync`的更多选项可以通过[**`AsyncObservableExtensions`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Extensions/AsyncObservableExtensions.cs)类获得。
`SubscribeAsync`返回一个[**`StreamSubscriptionHandle<T>`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/StreamSubscriptionHandle.cs)，这是一个不透明的句柄，可以用来取消流的订阅（精神上类似于异步版本的`IDisposable`）。

``` csharp
await subscriptionHandle.UnsubscribeAsync()
```

需要注意的是，**订阅是针对Grain的，而不是针对Grain的单个激活的**。一旦Grain代码订阅到了流，这个订阅就会超过这个激活的寿命，并一直保持持久，直到Grain代码（可能在不同的激活中）显式取消订阅。这就是**虚拟流抽象**的核心：不仅所有的流在逻辑上总是存在的，而且流的订阅也是持久的，其寿命超越了创建订阅的物理激活。

## Multiplicity多重性

一个Orleans流可以有多个生产者和多个消费者。一个生产者发布的消息将被传递给所有在消息发布前订阅了该流的消费者。

In addition, the consumer can subscribe to the same stream multiple times. Each time it subscribes it gets back a unique `StreamSubscriptionHandle<T>`. If a grain (or client) is subscribed X times to the same stream, it will receive the same event X times, once for each subscription. The consumer can also unsubscribe from an individual subscription. It can find all its current subscriptions by calling:
此外，消费者可以多次订阅同一个流。每次订阅都会得到一个唯一的`StreamSubscriptionHandle<T>`。如果一个Grain（或客户端）对同一个流订阅了$n$次，它将收到相同的事件$n$次。消费者也可以退订一个单独的订阅。它可以通过调用这个方法找到它当前所有的订阅：

``` csharp
IList<StreamSubscriptionHandle<T>> allMyHandles = await IAsyncStream<T>.GetAllSubscriptionHandles()
```

## 从故障中恢复

如果一个流的生产者死亡（或者它的Grain被停用），我们不需要做任何事。下次这个Grain想产生更多的事件时，它可以再次获得流的句柄，并以同样的方式生产新的事件。

消费者逻辑要更多一点。正如我们之前所说的，一旦一个消费者Grain订阅到一个流，这个订阅就会一直有效，直到该Grai显式取消订阅。如果流的消费者死亡（或其Grain被停用），并且流上产生了一个新的事件，消费者Grain将被自动重新激活（正如其他Grain在有消息发送到它时被自动激活）。现在Grain代码唯一需要做的是提供一个`IAsyncObserver<T>`来处理数据。消费者需要重新附上处理逻辑到`OnActivateAsync`方法。要做到这一点，它可以调用：

``` csharp
StreamSubscriptionHandle<int> newHandle = await subscriptionHandle.ResumeAsync(IAsyncObserver)
```

消费者使用它第一次订阅时得到的上一个句柄，以便“恢复处理”。注意，`ResumeAsync`只是用`IAsyncObserver`的新实例更新现有的订阅逻辑，并没有改变这个消费者已经订阅了这个流的事实。

消费者如何获得一个旧的`subscriptionHandle`？这里有2种选择。消费者可能已经持久化了从最初的`SubscribeAsync`操作中获得的句柄，现在可以直接使用它。或者，如果消费者没有这个句柄，它可以通过调用`IAsyncStream<T>`来询问所有活动的订阅句柄：

``` csharp
IList<StreamSubscriptionHandle<T>> allMyHandles = await IAsyncStream<T>.GetAllSubscriptionHandles()
```

现在消费者可以恢复所有的订阅，如果愿意，也可以取消订阅。

> 如果消费者Grain直接实现了`IAsyncObserver`接口（`public class MyGrain<T> : Grain, IAsyncObserver<T>`），理论上它应该不需要重新附上`IAsyncObserver`，因此也不需要调用`ResumeAsync`。流运行时应该能够自动发现Grain已经实现了`IAsyncObserver`，并直接调用`IAsyncObserver`的方法。然而，流运行时目前不支持这一点，即使Grain直接实现了`IAsyncObserver`，Grain代码仍然需要显式调用`ResumeAsync`。对于这一特性的支持已经在我们的TODO列表中。


## 显式与隐式订阅

默认情况下，流消费者必须显式地订阅流。这种订阅通常是由Grain（或客户端）收到的一些外部消息触发的，这些消息指示它进行订阅。例如，在一个聊天服务中，当一个用户加入一个聊天室时，他的Grain会收到一个带有聊天名称的`JoinChatGroup`消息，这使得用户Grain订阅这个聊天流。

此外，Orleans流还支持 **“隐式订阅”**。在这个模型中，Grain不显式地订阅流。这个Grain是自动地、隐式地订阅的，仅基于它的Grain标识和一个`ImplicitStreamSubscription`特性。隐式订阅的主要作用是允许流活动自动触发Grain的激活（从而触发订阅）。例如，使用SMS流，如果一个Grain想生产到一个流，另一个Grain处理这个流，生产者需要知道消费者Grain的标识，并向它发出一个Grain调用，告诉它订阅这个流。只有在这之后，它才能开始发送事件。相反，使用隐式订阅，生产者可以直接开始向流生产事件，而消费者Grain将自动被激活并订阅到流。在这种情况下，生产者根本不关心谁在读取这些事件。

`MyGrainType`类型的Grain实现类可以声明一个特性`[ImplicitStreamSubscription("MyStreamNamespace")]`。这告诉流运行时，当一个事件在标识是GUID为XXX且命名空间为`"MyStreamNamespace"`的流上产生时，它应该被传递给标识是GUID为XXX的`MyGrainType`类型的Grain。也就是说，运行时将流`<XXX, MyStreamNamespace>`映射到消费者Grain`<XXX, MyGrainType>`。

`ImplicitStreamSubscription`的存在使流运行时自动将此Grain订阅到一个流，并将流的事件传递给它。然而，Grain代码仍然需要告诉运行时它希望事件如何被处理。最主要的是，它需要附上`IAsyncObserver`。因此，当Grain被激活时，`OnActivateAsync`内的Grain代码需要调用：

``` csharp
IStreamProvider streamProvider = base.GetStreamProvider("SimpleStreamProvider");
IAsyncStream<T> stream = streamProvider.GetStream<T>(this.GetPrimaryKey(), "MyStreamNamespace");
StreamSubscriptionHandle<T> subscription = await stream.SubscribeAsync(IAsyncObserver<T>);
```

## 编写订阅逻辑

下面是如何为各种情况编写订阅逻辑的指南：显式和隐式订阅，可回溯和不可回溯的流。显式订阅和隐式订阅的主要区别在于，对于隐式订阅，Grain总是对每个流命名空间都有只有一个隐式订阅，无法创建多个订阅（没有订阅多重性），且无法取消订阅，Grain逻辑只需要附上处理逻辑。这也意味着，对于隐式订阅，永远不需要恢复订阅。
另一方面，对于显式订阅，则需要恢复订阅，否则如果Grain再次订阅，会导致其订阅的重复。

### 隐式订阅

对于隐式订阅，Grain需要为订阅附上处理逻辑。这应该在Grain的`OnActivateAsync`方法中完成。Grain应该在其`OnActivateAsync`方法中执行`await stream.SubscribeAsync(OnNext ...)`。这会使得这个激活附上`OnNext`函数来处理该流。Grain可以选择指定`StreamSequenceToken`作为`SubscribeAsync`的参数，这会使这个隐式订阅从该token开始消费。隐式订阅永远不需要调用`ResumeAsync`。

``` csharp
public async override Task OnActivateAsync()
{
    var streamProvider = GetStreamProvider(PROVIDER_NAME);
    var stream = streamProvider.GetStream<string>(this.GetPrimaryKey(), "MyStreamNamespace");
    await stream.SubscribeAsync(OnNextAsync)
}
```

### 显式订阅

对于显式订阅，Grain必须调用`SubscribeAsync`来订阅流。 这将创建一个订阅，并附上处理逻辑。
显式订阅将一直存在，直到Grain取消订阅，所以即使Grain被停用并重新激活，Grain仍然是显式订阅的，但没有附上处理逻辑。在这种情况下，Grain需要重新附上处理逻辑。要做到这一点，在其`OnActivateAsync`中，Grain首先需要调用`stream.GetAllSubscriptionHandles()`找出它有哪些订阅。Grain必须在它希望继续处理的每个句柄上执行`ResumeAsync`，或者在它已经完成的任何句柄上执行`UnsubscribeAsync`。Grain也可以选择指定`StreamSequenceToken`作为`ResumeAsync`调用的参数，这使得这个显式订阅从该token开始消费。

``` csharp
public async override Task OnActivateAsync()
{
    var streamProvider = GetStreamProvider(PROVIDER_NAME);
    var stream = streamProvider.GetStream<string>(this.GetPrimaryKey(), "MyStreamNamespace");
    var subscriptionHandles = await stream.GetAllSubscriptionHandles();
    if (!subscriptionHandles.IsNullOrEmpty())
        subscriptionHandles.ForEach(async x => await x.ResumeAsync(OnNextAsync));
}
```

## 流的顺序和序列令牌

单个生产者和单个消费者之间的事件交付顺序取决于流提供者。

在SMS中，生产者通过控制生产者发布事件的方式，显式控制消费者看到事件的顺序。默认情况下（如果SMS提供者的`FireAndForget`选项被设置为`false`），如果生产者等待每一个`OnNextAsync`调用，事件会以FIFO顺序到达。在SMS中，由生产者决定如何处理交付失败，这将由`OnNextAsync`调用所返回的中断的`Task`来表示。

Azure队列流不保证FIFO顺序，因为底层的Azure队列不保证故障情况下的顺序。(在无故障的执行中，它们确实保证了FIFO的顺序。)当生产者将事件生产到Azure队列时，如果入队操作失败，生产者会尝试再一次入队，随后处理潜在的重复消息。在交付方面，Orleans流运行时将事件从队列中取出，并尝试将其交付给消费者进行处理。Orleans流运行时只有在处理成功后才会将事件从队列中删除。如果交付或处理失败，该事件不会从队列中删除，并会在之后自动重新出现在队列中。流运行时会尝试再次交付，因此可能会破坏先进先出的顺序。上述行为符合Azure队列的正常语义。

**应用定义的顺序**：为了处理上述排序问题，应用可以选择性地指定自己的排序。这是通过[**`StreamSequenceToken`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/StreamSubscriptionHandle.cs)实现的，它是一个不透明的`IComparable`对象，可用于排序事件。
生产者可以传递一个可选的`StreamSequenceToken`给`OnNext`调用。这个`StreamSequenceToken`将被一直传递到消费者那里，并与事件一起交付。这样一来，应用程序可以独立于流运行时来推理并重建其顺序。

## 可回溯流

有些流只允许应用程序从最新的时间点开始订阅，而有些流则允许”回溯“。后者的能力取决于底层队列技术和特定的流供应商。例如，Azure队列只允许消费最新入队的事件，而EventHub允许从任意时间点（直到某个到期时间）重放事件。支持回溯时间的流被称为**可回溯流**。

可回溯流的消费者可以向`SubscribeAsync`调用传递一个`StreamSequenceToken`。运行时将从该`StreamSequenceToken`开始向其交付事件。一个空的令牌意味着消费者希望从最新的事件开始接收。

在恢复场景中，回溯流的能力是非常有用的。例如，考虑一个订阅了流并定期检查其状态以及最新序列令牌的Grain。当从故障中恢复时，Grain可以从最新的检查点（序列令牌）重新订阅同一个流，从而在恢复时不会丢失自上次检查点以来产生的任何事件。

[Event Hubs提供者](https://www.nuget.org/packages/Microsoft.Orleans.OrleansServiceBus/)是可回溯的。
你可以在[这里](https://github.com/dotnet/orleans/tree/master/src/Azure/Orleans.Streaming.EventHubs)找到其代码。
[SMS](https://www.nuget.org/packages/Microsoft.Orleans.OrleansProviders/)和[Azure队列](https://www.nuget.org/packages/Microsoft.Orleans.Streaming.AzureStorage/)提供者则是不可回溯的。

## 无状态自动扩展处理

默认情况下，Orleans流的目标是支持大量相对较小的流，每个流由一个或多个有状态的Grain处理。总的来说，所有流的处理都是由大量的普通（有状态的）Grain来分片处理的。应用代码通过分配流的ID和Grain的ID以及显式订阅来控制这种分片。**目的是分片的有状态处理**。

但是，也有一种有趣的情况，即**无状态自动扩展处理**。在这种情况下，应用有少量的流（甚至是单个大的流），目标是无状态处理。一个例子是一个全局性的事件流，其中的处理包括对每个事件进行解码，并可能将其转发到其他流，以便进一步进行有状态处理。无状态的扩展流处理在Orleans中可以通过无状态Worker Grain来支持。

**无状态自动扩展处理当前的进度：**：
我们还未实现这一特性。如果试图从一个无状态Worker Grain中订阅一个流将导致未定义行为。[我们正在考虑支持这个选项](https://github.com/dotnet/orleans/issues/433)。

## Grains和Orleans客户端

Orleans流**在Grain和Orleans客户端间统一地工作**。也就是说，完全相同的API可以在Grain和Orleans客户端中使用，以产生和消费事件。这大大简化了应用逻辑，使特殊的客户端API，如Grain观察者，变得冗余。

## 完全可控且可靠的Streaming Pub-Sub

为了跟踪流的订阅，Orleans使用了一个叫做**Streaming Pub-Sub**的运行时组件，它作为流消费者和流生产者的汇合点。Pub-Sub跟踪所有的流订阅，将其持久化，并将流消费者与流生产者相匹配。

Applications can choose where and how the Pub-Sub data is stored. The Pub-Sub component itself is implemented as grains (called `PubSubRendezvousGrain`), which use Orleans Declarative Persistence. `PubSubRendezvousGrain` uses the storage provider named `PubSubStore`. As with any grain, you can designate an implementation for a storage provider.  For Streaming Pub-Sub you can change the implementation of the `PubSubStore` at silo construction time using the silo host builder:
应用程序可以选择Pub-Sub数据的存储位置和方式。Pub-Sub组件本身被实现为Grain（称为`PubSubRendezvousGrain`），它使用Orleans声明式持久化。`PubSubRendezvousGrain`使用名为`PubSubStore`的存储提供者。与任何Grain一样，你可以为存储提供者指定一个实现。对于流Pub-Sub，你可以在Silo构建时使用Silo Host Builder修改`PubSubStore`的实现：

下面的代码配置Pub-Sub将其状态存储在Azure表中。

``` csharp
hostBuilder.AddAzureTableGrainStorage("PubSubStore", 
    options=>{ options.ConnectionString = "Secret"; });
```

这样，Pub-Sub数据将持久地存储在Azure表中。
对于初始开发，你也可以使用内存存储。
除了Pub-Sub之外，Orleans流运行时会将事件从生产者交付给消费者，管理分配给活跃流的所有运行时资源，并透明地从未使用的流中回收运行时资源。

## 配置

为了使用流，你需要通过Silo host或集群客户端builder启用流提供者。你可以在[这里](stream_providers.md)浏览更多关于流提供者的信息。下面是示例：

``` csharp
hostBuilder.AddSimpleMessageStreamProvider("SMSProvider")
  .AddAzureQueueStreams<AzureQueueDataAdapterV2>("AzureQueueProvider",
    optionsBuilder => optionsBuilder.Configure(
      options=>{ options.ConnectionString = "Secret"; }))
  .AddAzureTableGrainStorage("PubSubStore",
    options=>{ options.ConnectionString = "Secret"; });
```
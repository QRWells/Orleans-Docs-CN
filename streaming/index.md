---
title: Orleans Streams
---

# Orleans流

Orleans v.1.0.0 向编程模型中加入了对流扩展的支持。流扩展提供了一套抽象和一系列APIs，是得思考和处理流变得更加简单可靠。
流扩展允许开发者编写响应式应用，以结构化的方式对事件序列进行操作。流提供者的可扩展模型使编程模型与各种现有的各种队列技术兼容，并且可移植，例如
[Event Hubs](http://azure.microsoft.com/en-us/services/event-hubs/)，[ServiceBus](http://azure.microsoft.com/en-us/services/service-bus/)，[Azure Queues](http://azure.microsoft.com/en-us/documentation/articles/storage-dotnet-how-to-use-queues/)，和[Apache Kafka](http://kafka.apache.org/)。
我们不需要编写额外的代码或者进行额外操作就能与这些队列交互。

## 我该关注什么？

如果你以及对[流处理](https://confluentinc.wordpress.com/2015/01/29/making-sense-of-stream-processing/)有所了解，并且熟悉[Event Hubs](http://azure.microsoft.com/en-us/services/event-hubs/)，[Kafka](http://kafka.apache.org/)，[Azure Stream Analytics](http://azure.microsoft.com/en-us/services/stream-analytics/)，[Apache Storm](https://storm.apache.org/)，[Apache Spark Streaming](https://spark.apache.org/streaming/)以及[Reactive Extensions (Rx) in .NET](https://msdn.microsoft.com/en-us/data/gg577609.aspx)等技术，你可能会问：”我为什么应该关注流？“ **我们为什么需要另一个流处理系？统Actor与流的关系如何？**
["为什么是Orleans流？"](streams_why.md)就是为了回答这个问题。


## 编程模型

在Orleans流编程模型背后有这几条原则：

1. Orleans流是*虚拟的*。也就是说，流总是存在。它不会被显式地创建与销毁，也不会发生故障。
2. 流可以通过ID来*识别*，ID只是GUID和字符串组成的*逻辑名称*。
3. Orleans流可以让你*在时间和空间上将数据的生成和处理解耦*。这意味着流生产者和流消费者可能在不同的服务器上或处在不同的时区，且能经受住故障。
4. Orleans流是*轻量且动态的*。Orleans流运行时被设计用来处理大量的流，这些流会以很快的速度来来去去。
5. Orleans流的*绑定是动态的*。Orleans Stream Runtime被设计用来处理Grains高速连接到和断开流的情况。
6. Orleans流运行时*透明地管理流消费的生命周期*。在一个应用订阅了一个流之后，它就会收到该流的事件，即使在出现故障的情况下。
7. Orleans流*在Grains和Orleans客户端之间统一地工作*。


## 编程APIs

应用通过API与流进行交互，这些API与熟知的[.NET中的响应式扩展(Rx)](https://msdn.microsoft.com/en-us/data/gg577609.aspx)中的非常相似。
即通过使用实现了
[`Orleans.Streams.IAsyncObserver<T>`](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncObserver.cs)和
[`Orleans.Streams.IAsyncObservable<T>`](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncObservable.cs)接口的
[`Orleans.Streams.IAsyncStream<T>`](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncStream.cs)。

在下面一个典型的例子中，
一个设备生成了一些数据，
这些数据被作为一个HTTP请求发送到运行在云端的服务。
运行在前端服务器中的Orleans客户端收到这个HTTP调用，并将数据发布到一个匹配的设备流中：

``` csharp
public async Task OnHttpCall(DeviceEvent deviceEvent)
{
     // Post data directly into the device's stream.
     IStreamProvider streamProvider = GrainClient.GetStreamProvider("MyStreamProvider");
     IAsyncStream<DeviceEventData> deviceStream = streamProvider.GetStream<DeviceEventData>(deviceEvent.DeviceId, "MyNamespace");
     await deviceStream.OnNextAsync(deviceEvent.Data);
}
```

在下面的另一个例子中，一位用户（实现为Orleans Grain）加入了一个聊天室，
得到了这个房间里所有其他用户产生的聊天消息流的句柄，并订阅了它。
注意，用户不需要知道聊天室Grain本身（在我们的系统中可能没有这样的Grain），也不需要知道该组中其他生产消息的用户。
显然，要发布到聊天流，用户不需要知道谁目前订阅了聊天流。这展示了聊天室用户如何在时间和空间上完全解耦。

``` csharp
public class ChatUser: Grain
{
    public async Task JoinChat(Guid chatGroupId)
    {
       IStreamProvider streamProvider = base.GetStreamProvider("MyStreamProvider");
       IAsyncStream<string> chatStream = streamProvider.GetStream<string>(chatGroupId, "MyNamespace");
       await chatStream.SubscribeAsync(async (message, token) => Console.WriteLine(message))
    }
}
```

-----------------------------------

## 快速入门示例

The [Quick Start Sample](streams_quick_start.md) is a good quick overview of the overall workflow of using streams in the application.
After reading it, you should read the [Streams Programming APIs](streams_programming_APIs.md) to get a deeper understanding of the concepts.

[快速入门示例](streams_quick_start.md)是对在应用程序中使用流的整体工作流程的很好的快速概览。
读完后，你应该阅读[流编程APIs](streams_programming_APIs.md)来深入了解这些概念。

## 流编程APIs

[流编程APIs](streams_programming_APIs.md)提供了编程API的详细描述。

## 流提供者

More details on Stream Providers can be found at [Stream Providers](stream_providers.md).
流可以通过各种形状和形式的物理通道来实现，并且可以有不同的语义。Orleans流的设计是通过**流提供者**的概念来提供这种多样性，这是系统中的一个可扩展点。Orleans目前有两个流提供者的实现。基于TCP的**简易消息流提供者**和基于Azure队列的**Azure Queue流提供者**。
关于流提供者的更多细节可以在[流提供者](stream_providers.md)中找到。

## 流语义

**流订阅语义**：
Orleans保证了流的订阅操作的顺序一致性。具体来说，当消费者订阅一个流时，一旦代表订阅操作的`Task`被成功解决，消费者将看到所有在它订阅后产生的事件。
此外，可倒带流允许你通过使用`StreamSequenceToken`从过去的任意时间点进行订阅（更多细节可参见[这里](stream_providers.md)）。

**独立流事件的投递保证**：
独立事件投递保证依赖于独立流提供者。有些只提供尽力而为（best-effort）的至多一次（at-most-once）的投递（如简单消息流（SMS）），而另一些提供至少一次的投递（比如Azure队列流）。
甚至可以创造一个流提供者，保证精确的一次投递（我们还没有这样的提供者，但有可能创建一个）。

**事件投递顺序**：
事件的顺序同样取决于流提供者。在SMS流中，生产者通过控制事件发布的方式来显式地控制消费者看到的事件的顺序。Azure队列流不保证先入先出（FIFO）的顺序，因为底层Azure队列不保证故障情况下的顺序。
应用也可以通过使用`StreamSequenceToken`来控制自己的流的投递顺序。

## 流的实现

[Orleans流的实现](~/docs/implementation/streams_implementation/index.md)展示了内部实现的高层次概述。

## 代码示例

更多关于如何在Grains中使用流API的例子可以在[这里](https://github.com/dotnet/orleans/blob/master/test/Grains/TestGrains/SampleStreamingGrain.cs)找到。
我们计划在未来创建更多的示例。


## 更多材料

* [Orleans Virtual Meetup about Streams](https://www.youtube.com/watch?v=eSepBlfY554)
* [Orleans Streaming Presentation from Virtual Meetup](~/docs/resources/presentations/Orleans%20Streaming%20-%20Virtual%20meetup%20-%205-22-2015.pptx)

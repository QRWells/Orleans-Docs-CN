---
title: Orleans流 快速入门
description: 本指南将向你展示快速设置和使用Orleans流的方法。要了解更多关于流的细节，请阅读本文档的其他部分。
---

## 必要的配置

在本指南中，我们将使用一个基于简单消息的流，它使用Grain消息发送流数据给订阅者。我们将使用内存存储提供者来存储订阅列表，但是对于真正的生产应用来说，这不是一个好的选择。

在Silo上，其中`hostBuilder`是一个`ISiloHostBuilder`：

``` csharp
hostBuilder.AddSimpleMessageStreamProvider("SMSProvider")
           .AddMemoryGrainStorage("PubSubStore");
```

在集群客户端上,其中`clientBuilder`是一个`IClientBuilder`：

``` csharp
clientBuilder.AddSimpleMessageStreamProvider("SMSProvider");
```

---
**注意**

默认情况下，通过简单消息流传递的消息被认为是不可改变的，并且可以通过引用传递给其他Grain。要关闭这种行为，你必须配置SMS提供者来关闭`OptimizeForImmutableData`。

```csharp
siloBuilder
    .AddSimpleMessageStreamProvider("SMSProvider", (options) => options.OptimizeForImmutableData = false);
```

---

现在我们可以创建流，使用它们作为生产者发送数据，也可以作为订阅者接收数据。

## 生产事件

为流生产事件是相对容易的。首先应该访问你在上面的配置中定义的流提供者（`SMSProvider`），然后选择一个流并向其推送数据。

``` csharp
//Pick a GUID for a chat room grain and chat room stream
var guid = some guid identifying the chat room
//Get one of the providers which we defined in our config
var streamProvider = GetStreamProvider("SMSProvider");
//Get the reference to a stream
var stream = streamProvider.GetStream<int>(guid, "RANDOMDATA");
```

如你所见，我们的流有一个GUID和一个命名空间。这使我们很容易识别唯一的流。例如，一个聊天室的命名空间可以是"Rooms"，GUID可以是RoomGrain的GUID。

这里我们使用一些已知聊天室的GUID。使用流的`OnNext`方法，我们可以向它推送数据。让我们使用随机数在一个定时器内执行。你也可以为流使用任何其他数据类型。

``` csharp
RegisterTimer(s =>
        {
            return stream.OnNextAsync(new System.Random().Next());
        }, null, TimeSpan.FromMilliseconds(1000), TimeSpan.FromMilliseconds(1000));
```

## 订阅和接收流数据

对于接收数据，我们可以使用隐式/显式订阅，这在文档的其他页面有更详细的描述。这里我们使用隐式订阅，它更容易。当一个Grain类型想要隐式订阅一个流时，使用属性`ImplicitStreamSubscription (namespace)]`。

在我们的例子中，我们将这样定义一个`ReceiverGrain`：

``` csharp
[ImplicitStreamSubscription("RANDOMDATA")]
public class ReceiverGrain : Grain, IRandomReceiver
```

每当数据被推送到命名空间为RANDOMDATA的流中，就像我们在定时器中一样，一个与流的GUID相同的`ReceiverGrain`类型的Grain将收到该消息。即使目前没有已激活的Grain存在，运行时也会自动创建一个新的激活并将消息发送给它。

为了使其有效工作，我们需要通过设置接收数据的`OnNext`方法来完成订阅过程。要做到这一点，我们的`ReceiverGrain`应该在`OnActivateAsync`中调用类似这样的方法：

``` csharp
//Create a GUID based on our GUID as a grain
var guid = this.GetPrimaryKey();
//Get one of the providers which we defined in config
var streamProvider = GetStreamProvider("SMSProvider");
//Get the reference to a stream
var stream = streamProvider.GetStream<int>(guid, "RANDOMDATA");
//Set our OnNext method to the lambda which simply prints the data. This doesn't make new subscriptions, because we are using implicit subscriptions via [ImplicitStreamSubscription].
await stream.SubscribeAsync<int>(async (data, token) => Console.WriteLine(data));
```

全部设置好了! 现在只需要一个东西来触发我们的生产者Grain的创建，然后它将注册定时器并开始向所有感兴趣的人发送随机字节。

同样，本指南跳过了很多细节，只展示了大概。阅读文档的其他部分和RX上的其他资源，可以很好地了解新特性以及用法。

响应式编程可能是解决许多问题的一个非常强大的方法。例如，你可以在订阅者中使用LINQ来过滤数字，并做各种有趣的事情。
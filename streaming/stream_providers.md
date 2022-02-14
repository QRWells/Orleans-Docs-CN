---
title: Orleans流提供者
description: 本节介绍了Orleans内置的两种流提供者以及创建流提供者的快捷方法
---

流可以有不同的形态和形式。
一些流可能通过直接的TCP链接投递事件，有些流则通过持久的队列投递事件。
不同的流类型可能使用不同的批处理策略，不同的缓存算法，或不同的回压过程。
为了避免将流应用限制在这些行为选择上，**流提供者**是Orleans流运行时的扩展点，允许用户实现任何类型的流。
这个扩展点与[Orleans存储提供者](https://github.com/dotnet/orleans/wiki/Custom%20Storage%20Providers)的设计精神相似。
Orleans目前内置许多流提供者，包括：[简单消息流提供者](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core/Streams/SimpleMessageStream/SimpleMessageStreamProvider.cs) 和[Azure队列流提供者](https://github.com/dotnet/orleans/tree/master/src/Azure/Orleans.Streaming.AzureStorage/Providers/Streams/AzureQueue)。

## 简单消息流提供者

简单消息流提供者，也被称为SMS提供者，通过利用常规的Orleans Grain消息，在TCP上传递事件。
由于SMS中的事件是通过不可靠的TCP链路传递的，SMS**不保证**可靠的事件传递，也不会自动重发SMS流的失败消息。
默认情况下，生产者对`stream.OnNextAsync`的调用会返回一个`Task`，代表流消费者的处理状态，它告诉生产者，消费者是否成功接收并处理了事件。
如果这个任务失败，生产者可以决定再次发送同一事件，从而实现应用层面的可靠性。
尽管流消息的传递是尽力而为的，但SMS流本身是可靠的。
也就是说，由发布-订阅实现的订阅者到生产者的绑定是完全可靠的。

## Azure 队列 流提供者

在生产者一侧，AQ流提供者将事件直接压入Azure队列。
在消费者一侧，AQ流提供者管理一组**拉取代理**，从一组Azure队列中拉取事件，并将其交付给消费它们的应用代码。
我们可以把拉取代理看作是一个分布式的“微服务”--一个分区的、高可用的、弹性的分布式组件。
拉取代理在托管应用Grain的相同Silo内运行。
因此，不需要运行单独的Azure worker来从队列中提取。
拉取代理的存续、它们的管理、回压、平衡它们之间的队列以及将队列从一个故障的代理移交给另一个代理，都完全由Orleans流运行时管理，这对使用流的应用代码是透明的。
## 队列适配器

通过持久队列投递事件的不同流提供者表现出类似的行为，并受制于类似的实现。
因此，我们提供了一个通用的可扩展的[`PersistentStreamProvider`](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core/Streams/PersistentStreams/PersistentStreamProvider.cs)，允许开发者插入不同类型的队列，而不需要从头编写一个全新的流提供者。
`PersistentStreamProvider`使用一个[`IQueueAdapter`](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core/Streams/QueueAdapters/IQueueAdapter.cs)组件，它抽象了具体的队列实现细节，并提供`enqueue`和`dequeue`事件的方法。
其余的都是由`PersistentStreamProvider`内部的逻辑来处理。
上面提到的Azure队列提供者也是这样实现的：它是`PersistentStreamProvider`的一个实例，使用了`AzureQueueAdapter`。

## 下一步

[Orleans流实现细节](../implementation/streams_implementation/index.md)

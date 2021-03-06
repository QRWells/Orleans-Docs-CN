---
title: 单向请求
description: 本节介绍单向请求的相关内容
---

Grain异步地方式执行请求（方法调用），因此所有Grain接口方法必须返回一个异步类型，如`Task`。
通过等待从Grain调用中返回的`Task`的完成，调用者会被通知请求已经完成，任何异常或返回值都可以被传回调用者，以便它们可以被处理。
在有些情况下，调用者只是想*通知*一个Grain某些事件已经发生，而不需要接收任何异常或完成信号。
对于这些情况，Orleans支持*单向请求*。

单向请求立即返回给调用者，并且不发出失败或完成的信号。
单向请求甚至不能保证调用者收到请求。
单向请求的主要优点是，其节省了向调用者发回响应的信息传递成本，因此在一些特殊情况下可以提高性能。
单向请求是一种高级的性能特性，应该谨慎使用，只有当开发者确定单向请求是有益的。
建议首选常规的双向请求，它发出完成信号并将错误传回给调用者。

通过用`[OneWay]`特性标记Grain接口方法，可以使请求成为单向的，像这样：


``` csharp
public interface IOneWayGrain : IGrainWithGuidKey
{
    [OneWay]
    Task Notify(MyData data);
}
```

单向请求必须返回`Task`或`ValueTask`，并且不能返回这些类型的泛型（`Task<T>`和`ValueTask<T>`）。
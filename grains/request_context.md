---
title: 请求上下文
description: 本节介绍了Orleans中的请求上下文
---

请求上下文是Orleans的一个特性，它允许应用元数据与请求一起传播，如追踪ID。应用元数据可以在客户端添加，它将与Orleans请求一起传给接收Grain。

该特性由Orleans命名空间中的静态类`RequestContext`实现。这个类暴露了两个简单的方法：

`void Set(string key, object value)`用于在请求上下文中存储一个值。该值可以任意**可序列化**类型。`Object Get(string key)`用于从当前请求上下文中获取一个值。

请求上下文的底层存储是**线程静态(thread-static)**的。当一个线程（无论是客户端还是Orleans内部）发送一个请求时，发送线程的请求上下文的内容会被包含在请求的Orleans消息中，当Grain代码收到请求时，该元数据可以从本地的请求上下文中访问。如果Grain代码不修改请求上下文，那么它向任何Grain发出的请求都会收到相同的元数据，以此类推。

当你使用`StartNew`或者`ContinueWith`调度一个`Future`的计算时，应用元数据也会保持原样。在这两种情况下，计算续体将以与调度代码在计算被调度时相同的元数据执行（也就是说，系统会复制一个当前的元数据并将其传递给计算续体，所以调用`StartNew`或者`ContinueWith`后的变化不会被计算续体看到）。

注意，应用元数据不会随着响应而传回；也就是说，由于收到响应而运行的代码，无论是在`ContinueWith`计算续体中还是在调用`Wait`或`GetValue`之后，仍将在由原始请求设置的当前上下文中运行。

例如，要将客户端的追踪ID设置为一个新的GUID，可以调用：

``` csharp
RequestContext.Set("TraceId", new Guid());
```

在Grain代码（或其他在调度器线程上的Orleans内部运行的代码）中，原始客户端请求的追踪ID可以被使用，例如，写入日志：

``` csharp
Logger.Info("Currently processing external request {0}", RequestContext.Get("TraceId"));
```

虽然任何可序列化的对象都可以作为应用元数据发送，但值得一提的是，大型或复杂的对象可能会明显加大消息序列化的时间开销。因此，建议使用简单的类型（字符串、GUID或数字类型）。
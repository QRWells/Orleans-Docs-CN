---
title: Grain取消令牌
description: Orleans运行时提供了名为“Grain取消令牌”的机制，使开发者能够取消正在执行的Grain操作。
---

## 简介
**Grain取消令牌** 是对标准 .NET`System.Threading.CancellationToken`的封装，它可以实现线程、线程池工作项或`Task`对象之间的合作取消，并可以作为Grain方法参数传递。

`GrainCancellationTokenSource`是通过其Token属性提供取消令牌的对象，并通过调用其`Cancel`方法发送取消消息。 

## 用法

* 实例化一个`CancellationTokenSource`对象，它管理和发送取消通知给各个取消令牌。

``` csharp
        var tcs = new GrainCancellationTokenSource();
```
* 将`GrainCancellationTokenSource.Token`属性所返回的令牌传递给每个监听取消事件的Grain方法。

``` csharp
        var waitTask = grain.LongIoWork(tcs.Token, TimeSpan.FromSeconds(10));
```
* A cancellable grain operation needs to handle underlying **CancellationToken** property of **GrainCancellationToken** just like it would do in any other .NET code.一个可取消的Grain操作需要处理`GrainCancellationToken`底层的`CancellationToken`属性，正如其在其他.NET代码中一样。

``` csharp
        public async Task LongIoWork(GrainCancellationToken tc, TimeSpan delay)
        {
            while(!tc.CancellationToken.IsCancellationRequested)
            {
                 await IoOperation(tc.CancellationToken);
            }
        }
```
* 调用`GrainCancellationTokenSource.Cancel`方法来启动取消过程。

``` csharp
        await tcs.Cancel();
```
* 当你完成对`GrainCancellationTokenSource`对象的处理后，调用`Dispose`方法。

``` csharp
        tcs.Dispose();
```


 #### 重要考虑:

* `GrainCancellationTokenSource.Cancel`方法返回`Task`，为了确保能够取消，其调用必须在通信失败的情况下重试。
* 在底层`System.Threading.CancellationToken`中注册的回调受制于它们被注册的Grain激活中的单线程执行保证。
* 每个`GrainCancellationToken`都可以通过多个方法调用来传递。


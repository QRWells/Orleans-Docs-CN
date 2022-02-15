---
title: 异构Silos
---
## 总览

在一个给定的集群上，Silo可以支持一组不同的Grain类型。
![](https://github.com/dotnet/orleans-docs/blob/main/src/images/heterogeneous.png?raw=true)

在这个例子中，集群支持`A`，`B`，`C`，`D`，`E`类型的Grain。
* Grain类型`A`和`B`可以放在1号和2号Silo上。
* Grain类型`C`可以放在1、2、3号Silo上。
* Grain类型`D`只能放置在3号Silo上
* Grain类型`E`只能放置在4号Silo上。

所有Silo都应该引用集群中所有Grain类型的接口，但Grain类只应被承载它们的Silo所引用。

客户端不知道哪个Silo可以支持某个特定的Grain类型。

**一个给定的Grain类型的实现在支持它的每个Silo上都必须是相同的。以下情况是无效的：**

在1号和2号Silo上:

``` csharp
public class C: Grain, IMyGrainInterface
{
   public Task SomeMethod() { … }
}
```

在3号Silo上：

``` csharp
public class C: Grain, IMyGrainInterface, IMyOtherGrainInterface
{
   public Task SomeMethod() { … }
   public Task SomeOtherMethod() { … }
}
```

## 配置

无需配置，你可以在集群的每个Silo上部署不同的二进制文件。
然而，如果有必要，你可以改变Silo和客户端检查类型变化的时间间隔，通过特性`TypeMapRefreshInterval`，它来自`TypeManagementOptions`。

为了测试，你可以使用`GrainClassOptions`中的特性`ExcludedGrainTypes`，这是一个你想在Silo上排除的类型的名称列表。

## 限制条件

* 如果支持的Grain类型集合发生变化，连接的客户端并不会被通知。在前面的例子中：
	* 如果4号Silo离开集群，客户端仍将尝试对`E`类型的Grain进行调用。它会使运行时抛出`OrleansException`而故障。
	* 如果客户端在4号Silo加入集群之前已经连接到集群，客户端将不能调用`E`类型的晶粒。它将以`ArgumentException`故障。
* 不支持无状态Grain：集群中的所有Silo必须支持同一组无状态Grain。
* 不支持`ImplicitStreamSubscription`，因此只有["显式订阅"](../streaming/streams_programming_APIs.md)可以在Orleans流中使用。
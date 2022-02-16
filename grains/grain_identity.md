---
title: Grain标识
---

在面向对象的环境中，一个对象的标识很难与它的引用区分开来。
因此，当使用new创建一个对象时，你得到的引用代表了其标识的所有特征，除了那些将该对象映射到它所代表的一些外部实体的部分。

在分布式系统中，对象的引用不能代表实例的标识，因为引用通常只限于一个地址空间。
.NET的引用就是这样的。
此外，一个Grain必须有一个标识，无论它是否处于激活状态，这样我们才可以按需激活它。
因此，Grains有一个主键。
主键可以是一个全局唯一标识符（GUID），一个长整数或一个字符串。

主键的作用范围是Grain的类型。
因此，一个Grain的完整标识是由该Grain的类型和它的键组成的。

Grain的调用者决定该使用哪个方案。
选项可以是:

* long
* GUID
* string
* GUID + string
* long + string

因为底层数据是相同的，这些方案可以互换使用。
当使用长整型时，实际上创建了一个GUID，并以零填充。

在需要单例Grain的情况下，如字典或注册表，可以使用`Guid.Empty`作为其键。
这只是一个惯例，但通过遵守这个惯例，在调用者那里就可以清楚地看到一个单例Grain正在被使用，正如我们在第一个教程中看到的那样。

## 使用GUID

当有多个进程请求同一个Grain时，GUID会是一个很好的选择，比如web农场里的一些web服务器。
你不需要统筹key的分配，因为这可能会在系统中引入一个单点故障，也可能引入对一个资源的系统级锁，这可能会导致性能瓶颈。
GUID发生碰撞的几率非常低，所以在组建Orleans系统时，GUID会是一个不差的选择。

在客户端代码中通过GUID引用一个Grain：

``` csharp
var grain = grainFactory.GetGrain<IExample>(Guid.NewGuid());
```

从Grain代码中获取主键：

``` csharp
public override Task OnActivateAsync()
{
    Guid primaryKey = this.GetPrimaryKey();
    return base.OnActivateAsync();
}
```

## 使用长整型

长整型也可以用作主键，如果Grain被持久化到一个关系型数据库中，这会很有用，因为在那里数字索引比GUID更受欢迎。

在客户端代码中通过长整型引用一个Grain：

``` csharp
var grain = grainFactory.GetGrain<IExample>(1);
```

从Grain代码中获取主键：

``` csharp
public override Task OnActivateAsync()
{
    long primaryKey = this.GetPrimaryKeyLong();
    return base.OnActivateAsync();
}
```

## 使用字符串

字符串也可以用作主键。

在客户端代码中通过字符串引用一个Grain：

``` csharp
var grain = grainFactory.GetGrain<IExample>("myGrainKey");
```

从Grain代码中获取主键：

``` csharp
public override Task OnActivateAsync()
{
    string primaryKey = this.GetPrimaryKeyString();
    return base.OnActivateAsync();
}
```
## 使用复合主键

如果你的系统不适用GUID或长整型，你可以选择复合主键。
通过复合主键，你可以使用GUID或长整型，以及字符串的组合来引用一个Grain。

你可以继承`IGrainWithGuidCompoundKey`或`IGrainWithIntegerCompoundKey`接口，像这样：

``` csharp
public interface IExampleGrain : Orleans.IGrainWithIntegerCompoundKey
{
    Task Hello();
}
```

在客户端代码中，这使得Grain工厂的`GetGrain`方法有了第二个参数：

``` csharp
var grain = grainFactory.GetGrain<IExample>(0, "a string!", null);
```
为了访问Grain中的复合主键，我们可以调用重载的`GetPrimaryKey`方法：

``` csharp
public class ExampleGrain : Orleans.Grain, IExampleGrain
{
    public Task Hello()
    {
        string keyExtension;
        long primaryKey = this.GetPrimaryKeyLong(out keyExtension);
        Console.WriteLine("Hello from " + keyExtension);
        Task.CompletedTask;
    }
}
```


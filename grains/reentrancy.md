---
title: 可重入性
description: 本节本节介绍了Orleans中的请求调度机制
---

# 请求调度

Grain激活有一个*单线程*的执行模式，默认情况下，在开始处理下一个请求之前，会从开始到完成处理每个请求。
在某些情况下，当一个请求在等待异步操作完成时，一个激活可能需要处理其他请求。
出于这个原因和其他一些原因，Orleans给开发者提供了一些对请求交错行为的控制，如下文[可重入性](#可重入性)部分所述。
下面是一个非重入请求调度的例子，这是Orleans的默认行为。

我们最开始的例子主要关注下面的`PingGrain`的定义：

``` csharp
public interface IPingGrain : IGrainWithStringKey
{
    Task Ping();
    Task CallOther(IPingGrain other);
}

public class PingGrain : Grain, IPingGrain
{
    private readonly ILogger<PingGrain> _logger;

    public PingGrain(ILogger<PingGrain> logger) => _logger = logger;

    public Task Ping() => Task.CompletedTask;

    public async Task CallOther(IPingGrain other)
    {
        _logger.LogInformation("1");
        await other.Ping();
        _logger.LogInformation("2");
    }
}
```

在我们的例子中涉及到两个类型为`PingGrain`的Grain，即*A*和*B*。
一个调用者发起以下调用：

``` csharp
var a = grainFactory.GetGrain("A");
var b = grainFactory.GetGrain("B");
await a.CallOther(b);
```

![](https://raw.githubusercontent.com/dotnet/orleans-docs/main/src/images/scheduling_7.png)

执行流程如下：

1. 调用到达*A*，它记录下`"1"`，然后发起对*B*的调用
2. *B*立即从`Ping()`返回到*A*
3. *A*记录下`"2"`并返回到原始调用者

当*A*在等待对*B*的调用时，它不能处理其他传入的请求。
因此，如果*A*和*B*同时调用对方，它们可能会在等待调用完成时陷入*死锁*。
下面是一个例子，基于客户端发出的以下调用：

``` csharp
var a = grainFactory.GetGrain("A");
var b = grainFactory.GetGrain("B");

// A 调用 B 的同时 B 调用 A。
// 这可能会陷入死锁，这取决于事件的非确定性时机。
await Task.WhenAll(a.CallOther(b), b.CallOther(a));
```

## 情况1: 调用没有引发死锁

![](https://raw.githubusercontent.com/dotnet/orleans-docs/main/src/images/scheduling_8.png)

在这个例子中：

1. 来自*A*的`Ping()`调用在`CallOther(a)`调用到达*B*之前到达了*B*
2. 因此，*B*在`CallOther(a)`调用之前处理`Ping()`调用
3. 因为*B*处理了`Ping()`调用，所以*A*能够返回到调用者那里
4. 当*B*向*A*发出`Ping()`调用时，*A*仍在忙于记录其信息（`"2"`），所以该调用需要等待很短的时间，但它很快就能被处理
5. *A*处理了`Ping()`调用并返回到*B*，*B*再返回到原来的调用者

现在，我们将讨论一系列不那么幸运的事件：由于时机上略有不同，相同的代码引发了*死锁*。

## 情况2: 调用引发了死锁

![](https://raw.githubusercontent.com/dotnet/orleans-docs/main/src/images/scheduling_5.png)

在这个例子中：

1. `CallOther`到达各自的Grain，并同时进行处理
2. 两个Grain都记录下了`"1"`然后运行`await other.Ping()`
3. 因为两个Grains都在*忙碌*（处理尚未完成的`CallOther`请求），`Ping()`会等待
4. 经过一段时间后，Orleans认为调用已经**超时**，并且两个`Ping()`调用都会抛出异常
5. 这个异常没有被`CallOther`方法体处理，所以它抛给了原始调用者

下一节描述了如何通过允许多个请求相互交叉执行来防止死锁。

## 可重入性

Orleans默认选择一个安全的执行流程：一个Grain的内部状态不会被多个请求并发地修改。
内部状态的并发修改会使逻辑变得复杂，给开发者带来更大的负担。
这种对那些并发性Bug的保护是有代价的，正如我们在上面看到的，主要是*活跃性*：某些调用模式会导致死锁。
避免死锁的一个方法是确保Grain调用永远不会形成一个回环。
很多时候，要写出无回环且不会死锁的代码是很困难的。
在处理下一个请求之前，等待每个请求从开始运行到完成也会影响性能。
例如，在默认情况下，如果一个Grain方法对数据库服务执行一些异步请求，那么Grain将暂停请求的执行，直到数据库的响应到达Grain。

在后文中会讨论每种情况。
因此，Orleans为开发者提供了一些选项，运行部分或全部请求可以被*并发地*执行，通过交叉其执行过程。
在Orleans中，这称之为*可重入*和*交叉*。
通过并发执行请求，执行异步操作的Grains可以在更短的时间内处理更多的请求。

在以下情况下，多个请求可以交叉执行：

* Grain类带有`[Reentrant]`特性
* 接口方法带有`[AlwaysInterleave]`特性
* Grain的`[MayInterleave(x)]`谓词返回`true`。

有了可重入，下面的情况就可以有效执行，上述死锁的可能性也消除了。

### 情况3: Grain或方法是可重入的

![](https://raw.githubusercontent.com/dotnet/orleans-docs/main/src/images/scheduling_6.png)

在这个例子中，Grain *A*和*B*能够同时相互调用，而不会出现任何潜在的请求调度死锁，因为这两个晶粒都是*可重入的*。
下面几节将提供更多关于可重入的细节。

### 可重入Grains

`Grain`实现类可以用`[Reentrant]`特性进行标记，以表明不同的请求可以自由交叉进行。

换句话说，一个可重入的激活可以在前一个请求还没有处理完的时候开始执行另一个请求。
执行仍然被限制在一个线程内，所以激活仍然是一次执行一个回合，每个回合只代表激活的一个请求来执行。

可重入的Grain代码永远不会并行地运行多份Grain代码（Grain代码的执行永远是单线程的），但可重入的Grain**可能**会看到不同请求的代码交叉执行。
也就是说，来自不同请求的连续回合可能会交叉执行。

例如下面的伪代码，`Foo()`和`Bar()`是同一个Grain类的2个方法：

``` csharp
Task Foo()
{
    await task1;    // line 1
    return Do2();   // line 2
}

Task Bar()
{
    await task2;   // line 3
    return Do2();  // line 4
}
```

如果这个Grain被标记为`[Reentrant]`，`Foo()`和`Bar()`的执行可能会交叉。

例如，以下的执行顺序：

第一行，第三行，第二行和第四行。
也就是说，来自不同请求的回合是交叉的。

如果Grain不是可重入的，唯一可能的执行是：第1行，第2行，第3行，第4行；第3行，第4行，第1行，第2行（新的请求不能在前一个请求完成之前开始）。

在选择可重入和不可重入的Grain时，主要的权衡因素是使交叉正确工作的代码复杂性，以及推理它的难度。

在平凡的情况下，当Grain是无状态的，且逻辑简单时，少量的（但不能太少，以便所有的硬件线程都被利用）可重入Grain，通常会使效率略高一些。

如果代码比较复杂，那么更多的非可重入Grain，即使整体效率略低，也能为你省去很多解决非明显交叉问题的麻烦。

总的来说，答案还是取决于应用的具体细节。

### 交叉方法

标有`[AlwaysInterleave]`的Grain接口方法将被交叉使用，无论Grain本身是否是可重入的。请看下面的例子：

``` csharp
public interface ISlowpokeGrain : IGrainWithIntegerKey
{
    Task GoSlow();

    [AlwaysInterleave]
    Task GoFast();
}

public class SlowpokeGrain : Grain, ISlowpokeGrain
{
    public async Task GoSlow()
    {
        await Task.Delay(TimeSpan.FromSeconds(10));
    }

    public async Task GoFast()
    {
        await Task.Delay(TimeSpan.FromSeconds(10));
    }
}
```

现在考虑由下述客户请求发起的调用流程：

``` csharp
var slowpoke = client.GetGrain<ISlowpokeGrain>(0);

// A) 此操作耗时约20秒
await Task.WhenAll(slowpoke.GoSlow(), slowpoke.GoSlow());

// B) 此操作耗时约10秒
await Task.WhenAll(slowpoke.GoFast(), slowpoke.GoFast(), slowpoke.GoFast());
```

对`GoSlow`的调用将不会交叉执行，所以两个`GoSlow()`的调用的执行将需要大约20秒。
另一方面，由于`GoFast`被标记为`[AlwaysInterleave]`，对它的三个调用将被同时执行，总共将在大约10秒内完成，而不是需要至少30秒才能完成。

### 使用谓词的可重入性

Grain类可以指定一个谓词，通过检查请求，在逐个调用的基础上确定是否交叉执行。
`[MayInterleave(string methodName)]`特性提供了这个功能。
该特性的参数是Grain类中一个静态方法的名称，该方法接受一个`InvokeMethodRequest`对象，并返回一个`bool`，表示该请求是否应该交叉执行。

下面是一个例子，如果请求参数类型有`[Interleave]`特性，则允许交叉：

``` csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Struct)]
public sealed class InterleaveAttribute : Attribute { }

// 指定may-interleave谓词
[MayInterleave(nameof(ArgHasInterleaveAttribute))]
public class MyGrain : Grain, IMyGrain
{
    public static bool ArgHasInterleaveAttribute(InvokeMethodRequest req)
    {
        // 返回true表示这个调用应该与其他调用交叉进行
        // 返回false则反之
        return req.Arguments.Length == 1
            && req.Arguments[0]?.GetType().GetCustomAttribute<InterleaveAttribute>() != null;
    }

    public Task Process(object payload)
    {
        // Process the object.
    }
}
```

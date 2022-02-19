---
title: 客户端的托管
---

客户端允许非Grain代码与Orleans集群交互。
客户端允许应用程序代码与集群中托管的Grain和流进行通信。
有两种方法可以获取客户端，这取决于客户端代码的托管位置：与Silo在同一个进程中，或者在一个单独的进程中。
本文将讨论这两种选择，首先是我们推荐的选择：将客户端代码与Grain代码共同托管在同一进程中。

## Co-hosted clients共同托管的客户端

如果客户端代码与Grain代码托管在同一进程中，那么客户端可以直接从托管应用程序的依赖注入容器中获取。
在这种情况下，客户端直接与它所依附的Silo进行通信，可以利用Silo所知的关于集群的额外信息。

这样做有几个好处，包括减少网络和CPU的开销，以及减少延迟，增加吞吐量和可靠性。
客户端利用Silo对集群拓扑结构和状态的了解，不需要使用单独的网关。
也避免了网络跳转和序列化/反序列化的往返。
因此增加了可靠性，因为在客户端和Grain之间所需的节点数量是最小的。
如果Grain是一个[无状态Worker Grain](../grains/stateless_worker_grains.md)，或者恰好在客户端被托管的Silo上激活，那么根本不需要进行序列化或网络通信，客户端可以获得额外的性能和可靠性收益。
共同托管客户端和Grain代码也简化了部署和应用程序拓扑结构，因为不需要部署和监控两个不同的应用程序二进制文件。

但是这种方法也有缺点，主要是Grain代码不再与客户端进程隔离。
因此，客户端代码中的问题，如阻塞IO或锁争夺导致的线程饥饿，会影响到Grain代码的性能。
即使没有上述的代码缺陷，仅仅通过让客户代码与Grain代码在同一处理器上执行，也会产生*noisy neighbor*效应，给CPU缓存带来额外的压力，并对一般的本地资源产生额外的争夺。
此外，定位这些问题的来源现在更加困难，因为监控系统无法区分逻辑上的客户代码和Grain代码。

尽管有这些缺点，将客户端代码与Grain代码共同托管仍是一种流行的选择，也是大多数应用程序的推荐方法。
详细说来，上述的不利因素在实践中是很少的，原因如下:

* 客户端代码通常是非常*少*的，例如将传入的HTTP请求翻译成Grain调用，因此，*noisy neighbor*效应的影响是很小的，与其他所需的网关成本相当。
* 在出现性能问题时，开发人员的典型工作流程涉及到CPU分析器和调试器等工具，尽管在同一进程中同时执行客户和Grain代码，但这些工具在快速确定问题的来源方面仍然有效。换句话说，指标变得更加粗略，不能够精确地识别问题的来源，但更详细的工具仍然有效。

### 从主机处获取客户端

如果使用[.NET 通用主机](https://docs.microsoft.com/zh-cn/aspnet/core/fundamentals/host/generic-host)托管，客户端会自动在主机的[依赖注入](https://docs.microsoft.com/zh-cn/aspnet/core/fundamentals/dependency-injection)容器中可用，并且可以注入到诸如[ASP.NET控制器](https://docs.microsoft.com/zh-cn/aspnet/core/mvc/controllers/actions)或[`IHostedService`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostedservice)的实现里。

另外，可以从`IHost`或`ISiloHost`获取`IGrainFactory`或`IClusterClient`这样的客户端接口：

``` csharp
var client = host.Services.GetService<IClusterClient>();
await client.GetGrain<IMyGrain>(0).Ping();
```

## 外部客户端

客户端代码可以在托管Grain代码的Orleans集群之外运行。
因此，外部客户端会作为一个连接器或管道，连接到集群和应用程序的所有Grain。

![](https://github.com/dotnet/orleans-docs/blob/main/src/images/frontend_cluster.png?raw=true)

通常情况下，客户端在前端Web服务器上用于连接到Orleans集群，该集群作为中间层，其Grain执行业务逻辑。
在一个典型的设置中，一个前端的Web服务器：
* 接收网络请求
* 进行必要的认证和授权验证
* 决定哪个（哪些）Grain应处理该请求
* 使用Grain客户端向那些Grain发起一个或多个方法调用。
* 处理成功或失败的Grain调用以及任何返回值
* 为网络请求发回一个响应

### Grain客户端的初始化

在Grain客户端可以用于调用Orleans集群中托管的Grain之前，它需要被配置、初始化，并连接到集群。

配置是通过`ClientBuilder`和一些补充选项类提供的，这些选项类包含一个配置属性的层次结构，用于以编程方式配置客户端。

更多信息可以参阅[客户端配置指南](configuration_guide/client_configuration.md)。

下面是客户端配置的一个例子：

```csharp

var client = new ClientBuilder()
    // Clustering information
    .Configure<ClusterOptions>(options =>
    {
        options.ClusterId = "my-first-cluster";
        options.ServiceId = "MyOrleansService";
    })
    // Clustering provider
    .UseAzureStorageClustering(options => options.ConnectionString = connectionString)
    // Application parts: just reference one of the grain interfaces that we use
    .ConfigureApplicationParts(parts => parts.AddApplicationPart(typeof(IValueGrain).Assembly))
    .Build();

```

最后，我们需要在构建好的客户端对象上调用`Connect()`方法，使其连接到Orleans集群。这是一个异步方法，返回一个`Task`。所以我们需要用`await`或`.Wait()`来等待它的完成。

```csharp

await client.Connect();

```

### 向Grain发起调用

从客户端调用Grain与[从Grain代码中调用](../grains/index.md)其实没有什么区别。
同样的`GetGrain<T>(key)`方法，其中`T`是目标grain接口，在两种情况下都是用来[获得Grain引用](../grains/index.md#grain%E5%BC%95%E7%94%A8)的。
稍有不同的是，我们通过不一样的工厂对象来调用`GetGrain`。
在客户端代码中，我们通过连接的客户端对象做到这一点：

``` csharp
IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);
Task t = player.JoinGame(game)
await t;
```

根据[Grain接口规则](~/docs/grains/index.md)的要求，对Grain方法的调用会返回一个`Task`或`Task<T>`。
客户端可以使用`await`关键字来异步等待返回的`Task`而不阻塞线程，或者在某些情况下使用`Wait()`方法来阻塞当前执行线程。

从客户端代码调用Grain和从另一个Grain内部调用Grain的主要区别是Grain的单线程执行模式。
Grain被Orleans运行时限制为单线程，而客户端可能是多线程的。
Orleans并没有在客户端提供这样的保证，因此要由客户端使用适合其环境的同步结构来管理自己的并发——锁、事件、`Tasks`等等。

### 收取通知

在有些情况下，简单的请求-响应模式是不够的，客户端需要接收异步通知。
例如，一个用户可能希望在她所关注的人发布新消息时得到通知。

[观察者](../grains/observers.md)就是这样一种机制，它可以将客户端对象暴露为类似Grain的目标，以获得Grain的调用。
对观察者的调用不提供任何成功或失败的指示，因为它们是作为单向的尽力而为消息发送的。
因此，在必要时，应用程序代码需要在观察者之上建立一个更高层次的可靠性机制。

另一种可用于向客户端传递异步消息的机制是[流](~/docs/streaming/index.md)。流暴露了单个消息传递的成功或失败的指示，因此能够实现到客户端的可靠通信。

### 客户端连接 

在这两种情况下，集群客户端会遇到连接问题：

* 当`IClusterClient.Connect`方法最初被调用时。
* 当在从连接的集群客户端处获取的Grain引用上发起调用时。

在第一种情况下，`Connect`方法将抛出一个异常，以表明哪里出了问题。那通常是（但不一定）一个`SiloUnavailableException`。如果发生这种情况，集群客户端实例就无法使用，应该被释放掉。重试过滤功能可以选择性地提供给`Connect`方法，例如，在进行另一次尝试之前，可以等待一段时间。如果没有提供重试过滤器，或者重试过滤器返回`false`，客户端将放弃。

如果`Connect`成功返回，集群客户端保证可以使用，直到它被释放掉。这意味着，即使客户端遇到连接问题，它也会无限期地尝试恢复。确切的恢复行为可以在`ClientBuilder`提供的`GatewayOptions`对象上进行配置，例如：

```csharp
var client = new ClientBuilder()
    // ...
    .Configure<GatewayOptions>(opts => opts.GatewayListRefreshPeriod = TimeSpan.FromMinutes(10)) // Default is 1 min.
    .Build();
```

在第二种情况下，如果在Grain调用过程中发生连接问题，将在客户端抛出一个`SiloUnavailableException`。这可以像这样处理：

``` csharp
IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);

try
{
    await player.JoinGame(game);
}
catch (SiloUnavailableException)
{
    // Lost connection to the cluster...
}
```

在这种情况下，Grain引用不会失效，之后可以在同一引用上重试调用，那时可能已经重新建立了连接。

### 依赖注入

在使用.NET通用主机的程序中创建外部客户端的推荐方法是通过依赖注入来注入一个`IClusterClient`单例，然后它可以作为托管的服务、ASP.NET控制器等的构造器参数被接受。

**注意：**
当在将要连接到Orleans Silo的同一进程中共同托管该Silo时，*没有*必要手动创建一个客户端；Orleans将自动提供一个客户端并适当地管理其寿命。

当连接到不同进程中的集群时（例如在不同的机器上），一个常见的模式是创建一个像这样的托管服务。

```csharp
public class ClusterClientHostedService : IHostedService
{
    public IClusterClient Client { get; }

    public ClusterClientHostedService(ILoggerProvider loggerProvider)
    {
        Client = new ClientBuilder()
            // Appropriate client configuration here, e.g.:
            .UseLocalhostClustering()
            .ConfigureLogging(builder => builder.AddProvider(loggerProvider))
            .Build();
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // A retry filter could be provided here.
        await Client.Connect();
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await Client.Close();

        Client.Dispose();
    }
}
```

然后，像这样注册该服务：

```csharp
public class Program
{
    static Task Main()
    {
        return new HostBuilder()
            .ConfigureServices(services =>
            {
                services.AddSingleton<ClusterClientHostedService>();
                services.AddSingleton<IHostedService>(sp => sp.GetService<ClusterClientHostedService>());
                services.AddSingleton<IClusterClient>(sp => sp.GetService<ClusterClientHostedService>().Client);
                services.AddSingleton<IGrainFactory>(sp => sp.GetService<ClusterClientHostedService>().Client);
            })
            .ConfigureLogging(builder => builder.AddConsole())
            .RunConsoleAsync();
    }
}
```

在这一点上，`IClusterClient`实例可以在任何支持依赖注入的地方被消费，例如ASP.NET控制器：

```csharp
public class HomeController : Controller
{
    readonly IClusterClient _client;

    public HomeController(IClusterClient client) => _client = client;

    public IActionResult Index()
    {
        var grain = _client.GetGrain<IMyGrain>();
        var model = grain.GetModel();

        return View(model);
    }
}
```

### 示例

这是上面给出的例子的扩展版本，一个客户端程序连接到Orleans，找到玩家账户，用观察者订阅玩家所在的游戏会话的更新，并打印出通知，直到程序被手动终止。

```csharp
namespace PlayerWatcher
{
    class Program
    {
        /// <summary>
        /// Simulates a companion application that connects to the game
        /// that a particular player is currently part of, and subscribes
        /// to receive live notifications about its progress.
        /// </summary>
        static void Main(string[] args)
        {
            RunWatcher().Wait();
            // Block main thread so that the process doesn't exit.
            // Updates arrive on thread pool threads.
            Console.ReadLine();
        }

        static async Task RunWatcher()
        {
            try

            {
            var client = new ClientBuilder()
                // Clustering information
                .Configure<ClusterOptions>(options =>
                {
                    options.ClusterId = "my-first-cluster";
                    options.ServiceId = "MyOrleansService";
                })
                // Clustering provider
                .UseAzureStorageClustering(options => options.ConnectionString = connectionString)
                // Application parts: just reference one of the grain interfaces that we use
                .ConfigureApplicationParts(parts => parts.AddApplicationPart(typeof(IValueGrain).Assembly))
                .Build();

                // Hardcoded player ID
                Guid playerId = new Guid("{2349992C-860A-4EDA-9590-000000000006}");
                IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);
                IGameGrain game = null;

                while (game == null)
                {
                    Console.WriteLine("Getting current game for player {0}...", playerId);

                    try
                    {
                        game = await player.GetCurrentGame();
                        if (game == null) // Wait until the player joins a game
                        {
                            await Task.Delay(5000);
                        }
                    }
                    catch (Exception exc)
                    {
                        Console.WriteLine("Exception: ", exc.GetBaseException());
                    }
                }

                Console.WriteLine("Subscribing to updates for game {0}...", game.GetPrimaryKey());

                // Subscribe for updates
                var watcher = new GameObserver();
                await game.SubscribeForGameUpdates(
                    await client.CreateObjectReference<IGameObserver>(watcher));

                Console.WriteLine("Subscribed successfully. Press <Enter> to stop.");
            }
            catch (Exception exc)
            {
                Console.WriteLine("Unexpected Error: {0}", exc.GetBaseException());
            }
        }
    }

    /// <summary>
    /// Observer class that implements the observer interface. Need to pass a grain reference to an instance of this class to subscribe for updates.
    /// </summary>
    class GameObserver : IGameObserver
    {
        // Receive updates
        public void UpdateGameScore(string score)
        {
            Console.WriteLine("New game score: {0}", score);
        }
    }
    }
}
```

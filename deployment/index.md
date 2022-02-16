---
title: 运行应用
---

### Orleans应用

一个典型的Orleans应用程序由一个服务器进程（Silos）集群和一组客户端进程组成，通常是Web服务器，它们接收外部请求，将其转化为Grain方法调用，并将结果返回。
因此，运行一个Orleans应用的第一件事就是启动一个Silo集群。
出于测试的目的，一个集群可以由单个Silo组成。
对于可靠的生产部署来说，我们显然希望在一个集群里有多个Silo，以实现容错和扩展。

一旦集群运行起来，我们可以启动一个或多个客户端进程，连接到集群，并可以向Grain发送请求。
客户端连接到Silo上一个特殊的TCP端点--网关。
默认情况下，集群中的每个Silo都启用了客户端网关。
因此，客户端可以并行地连接到所有Silo，以获得更好的性能和弹性。

### 配置并启动一个Silo

Silo可以通过`ClusterConfiguration`对象以编程方式进行配置。
可以直接实例化它并应用，或从文件中加载设置，或者用一些辅助方法创建，以适应不同的部署环境。
对于本地测试，最简单的方法是使用`ClusterConfiguration.LocalhostPrimarySilo()`辅助方法。
然后将配置对象传递给`SiloHost`类的一个新实例，之后就可以初始化并启动。

你可以创建一个空的控制台应用程序项目，目标是.NET Framework 4.6.1或更高版本，用于托管一个Silo。
并在项目中添加`Microsoft.Orleans.Server` NuGet包。

```powershell
PM> Install-Package Microsoft.Orleans.Server
```

下面的例子展示了如何启动本地Silo：

```csharp
var siloConfig = ClusterConfiguration.LocalhostPrimarySilo(); 
var silo = new SiloHost("Test Silo", siloConfig); 
silo.InitializeOrleansSilo(); 
silo.StartOrleansSilo();

Console.WriteLine("Press Enter to close."); 
// wait here
Console.ReadLine(); 

// shut the silo down after we are done.
silo.ShutdownOrleansSilo();
```

### 配置客户端并连接到集群

连接到Silo集群并向Grain发送请求的客户端可以通过`ClientConfiguration`对象和`ClientBuilder`以编程方式进行配置。
可以直接实例化`ClientConfiguration`并应用，或从文件中加载设置，或者用一些辅助方法创建，以适应不同的部署环境。
对于本地测试，最简单的方法是使用`ClientConfiguration.LocalhostSilo()`辅助方法。
然后，配置对象被传递给`ClientBuilder`类的新实例。

`ClientBuilder`提供了更多的方法来配置额外的客户端功能。
之后，调用`ClientBuilder`对象的`Build`方法来获得`IClusterClient`接口的实现。
最后，我们在返回的对象上调用`Connect()`方法来连接到集群。

你可以创建一个针对.NET Framework 4.6.1或更高版本的空控制台应用项目来运行客户端，或者重新使用前文中创建的控制台应用程序项目。
你需要在项目中添加`Microsoft.Orleans.Client` NuGet包。

```powershell
PM> Install-Package Microsoft.Orleans.Client
```

下面的例子展示了客户端如何连接到本地Silo：

```csharp
var config = ClientConfiguration.LocalhostSilo();
var builder = new ClientBuilder().UseConfiguration(config).
var client = builder.Build();
await client.Connect();
```

### 生产环境配置

我们在这里使用的配置例子是用于测试Silo和客户端在同一台机器上运行的情况（即`localhost`）。
在生产环境中，Silo和客户端通常运行在不同的服务器上，并使用可靠的集群配置选项之一进行配置。
你可以在[配置指南](../host/configuration_guide/index.md)和[集群管理](../implementation/cluster_management.md)的描述中找到更多信息。
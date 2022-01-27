---
description: Orleans是一个用于构建健壮、可伸缩的分布式应用程序的跨平台框架
---

# 概述

Orleans建立在.NET开发者生产力的基础上，并将其带入了分布式应用程序的世界，例如云服务。 Orleans可从单个本地服务器扩展到云中全局分布的高可用性应用。

Orleans采用了对象、接口、async/await和try/catch等熟悉的概念，并将其扩展到多服务器环境。
这样，它可以帮助具有单服务器应用程序经验的开发者过渡到构建弹性、可扩展的云服务和其他分布式应用程序。 因此，Orleans通常被称为“分布式.NET”。

它由[Microsoft Research](http://research.microsoft.com/projects/orleans/)创建，引入了[Virtual Actor模型](http://research.microsoft.com/apps/pubs/default.aspx?id=210931)
作为一种构建面向云时代的新一代分布式系统的新方法。
Orleans的核心贡献是它的编程模型，其在不限制功能或对开发者施加繁重约束的情况下，降低了高并发分布式系统固有的复杂性。

## Grains

![一个Grain由一个稳定的标识, 表现和状态构成](https://raw.githubusercontent.com/dotnet/orleans-docs/main/src/images/grain_formulation.svg)

所有Orleans应用的基本构件都是*grain*。
Grains是由用户定义的身份、行为和状态所组成的实体。
Grain标识是由用户定义的键值，它使Grains始终可供调用。
Grains可以通过强类型通信接口(contracts)被其他Grains或Web前端等外部客户端调用。
每个Grain都是一个实现了一个或多个这些接口的类的实例。

Grains可以具有非持久化和（或）可以存储在任何存储系统中的持久化状态。
因此，Grains隐式地划分应用状态，实现自动可扩展性并简化故障恢复。
当Grain处于活动状态时，其状态被保存在内存中，以获得更低的延迟与数据存储负载。

![](https://raw.githubusercontent.com/dotnet/orleans-docs/main/src/images/managed_lifecycle.svg)

Grains的实例化是由Orleans运行时自动按需进行的。暂时不用的Grains会自动从内存中删除以释放资源。
它们稳定的标识使之成为可能，因为标识可以用于调用Grains，无论它们是否已经加载到内存中。 
这还允许透明地从失败中恢复，因为调用方不需要知道在哪个服务器上的何时实例化了一个Grain。
Grains有一个受管理的生命周期，Orleans运行时负责激活/停用Grains，并根据需要存储/定位Grains。
这使得开发者可以基于一个假设——所有的Grains总是在内存中，来编写代码。

综合来看，稳定的标识、有状态和受管理的生命周期是使构建在Orleans之上的系统具有可扩展性、高性能和可靠性的核心因素，而不会迫使开发者编写复杂的分布式系统代码。

### 示例: 物联网云后端

考虑一个[物联网](https://zh.wikipedia.org/zh/%E7%89%A9%E8%81%94%E7%BD%91)系统的云后端。
这个应用需要处理传入的设备数据，过滤、汇总并处理这些信息，而且能够向设备发送命令。
在Orleans中，很自然地，对于每一个设备，我们都可以用一个Grain去模拟，这个Grain就是它所对应的物理设备的*数字映射*。
这些Grains将最新的设备数据保存在内存中，这样就可以快速查询和处理，而不需要与物理设备直接通信。
通过观察来自设备的时间序列数据，Grains可以检测到条件的变化，比如，测量值超过阈值时触发某个动作。

一个简单的恒温器可以模拟如下：

``` csharp
public interface IThermostat : IGrainWithStringKey
{
  Task<List<Command>> OnUpdate(ThermostatStatus update);
}
```

从Web前端中的恒温器到达的事件可以通过调用`OnUpdate`方法发送给其对应的Grain，该方法可以选择性地将一个命令发回给设备。

``` csharp
var thermostat = client.GetGrain<IThermostat>(id);
return await thermostat.OnUpdate(update);
```

同一个恒温器Grain可以实现一个单独的接口，供控制系统交互使用：

``` csharp
public interface IThermostatControl : IGrainWithStringKey
{
  Task<ThermostatStatus> GetStatus();

  Task UpdateConfiguration(ThermostatConfiguration config);
}
```

这两个接口（`IThermostat` 和 `IThermostatControl`）被同一个类实现:

``` csharp
public class ThermostatGrain : Grain, IThermostat, IThermostatControl
{
  private ThermostatStatus _status;
  private List<Command> _commands;

  public Task<List<Command>> OnUpdate(ThermostatStatus status)
  {
    _status = status;
    var result = _commands;
    _commands = new List<Command>();
    return Task.FromResult(result);
  }

  public Task<ThermostatStatus> GetStatus() => Task.FromResult(_status);
  
  public Task UpdateConfiguration(ThermostatConfiguration config)
  {
    _commands.Add(new ConfigUpdateCommand(config));
    return Task.CompletedTask;
  }
}
```

上述的grain类并没有持久化它的状态。
在[文档](grains/grain_persistence/index.md)中可以找到更多展示状态持久化的例子。

## Orleans运行时

Orleans运行时实现了应用程序的编程模型。运行时的主要组件是*silo*，它负责托管Grains。
通常情况下，一组Silos作为一个Cluster（集群）运行，以获得可扩展性和容错能力。
当作为一个Cluster运行时，Silos间相互协调来分配工作，检测和恢复故障。
运行时使Cluster中托管的Grains能够相互通信，就像它们在同一个进程中一样。

除了核心编程模型外，Silo还为Grains提供了一套运行时服务，如计时器、提醒器（持续计时器）、持久化、事务、流等。
更多细节见下面的[特性](#特性)。

Web前端和其他外部客户端可以使用客户端库，来调用Cluster中的Grains，这个库能够自动管理网络通信。
简单起见，客户端也可以和Silos在同一进程中共同托管。

Orleans与.NET Standard 2.0及更高版本兼容，可以运行在Windows、Linux和macOS上，通过完整的.NET Framework或.NET Core。

## 特性

### 持久化

Orleans提供了一个简单的持久化模型，以确保在请求被处理前，状态对Grains可用且保持一致性。
Grains可以有多个具名的持久化数据对象，例如，一个名为“profile”的对象用于存储用户信息，一个名为“inventory”的对象用于存储用户的库存。状态可以存储在任意存储系统中。
例如，用户信息可以存储在一个数据库中，而库存可以存储在另一个数据库中。
当Grain运行时，这个状态存储在内存中，这样就可以在不访问存储系统的情况下处理读取请求。
当一个Grain更新了它的状态，`state.WriteStateAsync()` 调用确保其对应存储也被更新，以保证持久性和一致性。
更多信息请参见[Grain持久化](grains/grain_persistence/index.md)的文档。

### 分布式ACID事务

除了上述的简单的持久化模型，Grains可以持有*事务状态*。
多个Grains可以共同参与[ACID](https://zh.wikipedia.org/wiki/ACID)事务，无论它们的状态最终存储在哪里。
Orleans中的事务是分布式去中心化的（没有中央事务管理器或协调器），并且有[可串行化的隔离级别](https://zh.wikipedia.org/wiki/%E4%BA%8B%E5%8B%99%E9%9A%94%E9%9B%A2#%E5%8F%AF%E4%B8%B2%E8%A1%8C%E5%8C%96)。
关于Orleans中事务的更多信息，请参见[文档](grains/transactions.md)以及[Microsoft Research technical report](https://www.microsoft.com/en-us/research/publication/transactions-distributed-actors-cloud-2/)。

### 流

流可以帮助开发者接近实时地处理一系列数据。
Orleans中的流是*受管理的*：在Grains或客户端发布或订阅到一个流之前，不需要创建或注册流。
这使得流的生产者和消费者之间以及与基础设施之间更高程度的解耦。
流的处理是可靠的：Grains可以储存检查点（cursors）并在激活期间或之后的任意时间重置到已存储的检查点。
流支持消息分批向消费者交付，以提高效率和恢复性能。
流由Azure Event Hubs, Amazon Kinesis等队列服务所支持。
任意数量的流可以被复用至更小数量的队列上，这些队列的处理将在集群上平均分配。

### 计时器&提醒器

提醒器是Grains的一种持久化调度机制。
它们用于确保某些动作在未来的某个时间点完成，即使Grain在那时并未激活。
计时器是与提醒器相对于的非持久化调度机制，可以用于不要求可靠性的高频事件。
更多信息，请参见[计时器与提醒器](grains/timers_and_reminders.md)的文档。

### 灵活的Grain安置

当一个Grain在Orleans中被激活时，会由运行时来决定在哪个服务器（Silo）上激活这个Grain。
这称为Grain安置。
Orleans中的安置过程是完全可配置的：开发者可以从一组开箱即用的安置策略中选择，如随机、偏好本地和基于负载，或者配置自定义逻辑。
决定Grain的创建位置因此变得非常灵活。例如，Grains可以被安置在靠近它们需要操作的资源或靠近与之通信的其他Grains的服务器上。
更多信息请参见[Grain安置](grains/grain_placement.md)的文档。

### Grains版本化&异构集群

应用的代码会随时间推移发生变化，以安全的方式升级在线生产系统将会是一个挑战，尤其是有状态的系统。
Orleans中的Grain接口能够可选地进行版本控制。
Cluster维护了一个映射，其中包括Cluster中哪个Silo上有哪些Grain实现，以及这些实现的版本。
Orleans运行时会将这个版本信息与安置策略一起使用，以便在路由调用到Grains时做出安置决定。
除了安全地更新处于版本控制中的Grains外，版本化还可以实现异构Clusters，即Silos各自可以使用不同的一套Grains实现。
更多信息，请参见[Grain版本化](grains/grain_versioning/grain_versioning.md)的文档。

### 弹性可扩展性&容错

Orleans的设计使其可以弹性扩展。当一个Silo加入Cluster时，它能够接受新的（Grains的）激活，当一个Silo离开Cluster时（由于规模缩减或机器故障），在该Silo上激活的Grains将按需在其他Silos上重新激活。
一个Orleans Cluster可以被缩减为一个Silo。实现弹性扩展的同时也实现了容错：集群可以自动检测并快速恢复故障。

### 到处运行

Orleans可以在任何支持.NET Core或.NET Framework的地方运行。
包括在Linux、Windows和macOS上托管，部署到Kubernetes、虚拟机或物理机上，在本地或云上，以及PaaS服务上，如Azure云。

### 无状态Workers

无状态workers是具有特殊标记的Grains，其不带有任何状态，可以同时在多个Silos上激活。
这可以提高无状态函数的并行性。更多信息，请参见[无状态Worker Grains](grains/stateless_worker_grains.md)的文档。

### Grain调用过滤器

Grains之间共同的逻辑可以通过[Grain调用过滤器](grains/interceptors.md)来表达。
Orleans支持传入和传出调用的过滤器。过滤器的一些常见用例是：授权、日志和遥测、以及错误处理。

### 请求上下文

元数据和其他信息可以通过使用[请求上下文](grains/request_context.md)沿着一系列的请求传递。
请求上下文可用于保存分布式跟踪信息或任何其他用户定义的值。

## 入门

请参见[入门教程](tutorials_and_samples/tutorial_1.md)。

### 构建

在Windows上，运行`build.cmd`脚本以在本地构建NuGet包，然后从`/Artifacts/Release/*`引用所需的NuGet包。
你可以运行`Test.cmd`来运行所有BVT测试，`TestAll.cmd`还可以运行功能测试。

在Linux和macOS上，运行`build.sh`脚本或`dotnet build ./OrleansCrossPlatform.sln`来构建Orleans。

## 官方构建

最新的稳定、可用于生产环境的版本可以在[这里](https://github.com/dotnet/orleans/releases/latest)找到。

Nightly构建版发布在https://orleans.pkgs.visualstudio.com/orleans-public/_packaging/orleans-builds/nuget/v3/index.json。
这些构建通过了所有功能性测试，但没有像发布到NuGet的稳定版构建或预发布构建那样进行彻底的测试。

### 在项目中使用Nightly构建包

要在你的项目中使用Nightly构建，请使用以下任一方法添加MyGet feed：

1. 修改.csproj文件使其包含如下部分：

```xml
  <RestoreSources>
    $(RestoreSources);
    https://orleans.pkgs.visualstudio.com/orleans-public/_packaging/orleans-builds/nuget/v3/index.json;
  </RestoreSources>
```

或者

1. 在解决方案目录下创建一个包含如下内容的`NuGet.config`文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
 <packageSources>
  <clear />
  <add key="orleans-ci" value="https://orleans.pkgs.visualstudio.com/orleans-public/_packaging/orleans-builds/nuget/v3/index.json" />
  <add key="nuget" value="https://api.nuget.org/v3/index.json" />
 </packageSources>
</configuration>
```

## 社区

* 通过[在Github上创建issue](https://github.com/dotnet/orleans/issues)或通过[Stack Overflow](https://stackoverflow.com/questions/ask?tags=orleans)进行提问。
* [在Discord上交流](https://aka.ms/orleans-discord)
* 关注[@msftorleans](https://twitter.com/msftorleans)Twitter账户获取有关Orleans的公告。
* [OrleansContrib - 用于Orleans的社区插件的GitHub组织](https://github.com/OrleansContrib/)各种社区项目，包括监控、设计模式、Storage Provider等。
* 为希望[为Orleans贡献代码](resources/contributing.md)的开发者提供的指南。
* 我们同样鼓励你通过在GitHub上创建一个新的[会话](https://github.com/dotnet/orleans/issues)来报告错误或进行技术讨论。

## 许可

本项目采用[MIT许可证](https://github.com/dotnet/orleans/blob/master/LICENSE)授权。

## 快捷链接

* [Microsoft Research project home](http://research.microsoft.com/projects/orleans/)
* 技术报告: [Distributed Virtual Actors for Programmability and Scalability](http://research.microsoft.com/apps/pubs/default.aspx?id=210931)
* [Orleans文档](http://dotnet.github.io/orleans/)

## Orleans的起源

Orleans[创建于微软研究院并设计用于云计算](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/)。
自2011年以来，它已被多个微软产品群广泛地应用于云端和本地，其中最著名的是游戏工作室，
如343 Industries和The Coalition将其用于《光环4》、《光环5》以及《战争机器4》背后的云服务平台，除此之外也被许多其他公司采用。

Orleans于2015年1月开源，吸引了许多开发者，是[.NET生态中最具活力的开源社区之一](http://mattwarren.org/2016/11/23/open-source-net-2-years-later/)。
在开发者社区和微软Orleans团队的积极合作下，每天都有特性的添加和改进。
微软研究院将继续与Orleans团队合作，推出新的主要特性，如[geo-distribution](https://www.microsoft.com/en-us/research/publication/geo-distribution-actor-based-services/)，
[indexing](https://www.microsoft.com/en-us/research/publication/indexing-in-an-actor-oriented-database/)
和[分布式事务](https://www.microsoft.com/en-us/research/publication/transactions-distributed-actors-cloud-2/)，这些功能正在推动技术的发展。
Orleans已经成为许多.NET开发者构建分布式系统和云服务的首选框架。

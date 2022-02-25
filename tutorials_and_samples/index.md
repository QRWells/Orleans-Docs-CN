---
title: 示例
---

## [Hello, World!](./Hello-World.md)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/HelloWorld/code.png?sanitize=true" />
</p>

一个*Hello, World!*应用，演示了如何创建和使用你的第一个Grain。
*Hello, World!*应用是程序员的一种仪式，这就是基于Orleans的我们的*Hello, World!*示例。
该示例由单个项目构成，它启动基于Orleans的应用程序，向一个Grain发送消息，打印响应，并在用户按键时终止。

**这个程序演示了：**

* 如何开始接触Orleans
* 如何定义并实现一个Grain接口
* 如何获取Grain的引用，以及如何调用Grain

## [Adventure](https://github.com/dotnet/orleans/blob/main/samples/Adventure/README.md)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/Adventure/assets/BoxArt.jpg?sanitize=true" />
</p>

在有图形用户界面（GUI）之前，在游戏机和大规模多人游戏时代之前，我们有VT100终端，有[巨洞冒险](https://zh.wikipedia.org/wiki/%E5%B7%A8%E6%B4%9E%E5%86%92%E9%9A%AA)、[魔域](https://zh.wikipedia.org/wiki/%E9%AD%94%E5%9F%9F)和[微软冒险](https://en.wikipedia.org/wiki/Microsoft_Adventure)。
以今天的标准来看，这些可能很差劲，但在那时，那是一个神奇的世界，有怪物，有鸟叫，你还可以捡起东西。
它是这个示例的灵感来源。

**这个程序演示了：**

* 如何使用Grain来构造一个应用（一个游戏，在这个例子里）。
* 如何从外部客户端连接到Orleans集群（`ClientBuilder`）。

## [Chirper](https://github.com/dotnet/orleans/raw/main/samples/Chirper/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/Chirper/screenshot.png?sanitize=true" />
</p>

一个社交网络的发布/订阅系统，用以在用户之间发送简短的文本信息。
发布者向任何关注他们的其他用户发送简短的 *"Chirp"* 消息（不要与 *"Tweets"* 混淆，因为存在各种法律因素）。

**这个程序演示了：**

* 如何使用Orleans创建一个简单的社交媒体/社交网络应用
* 如何使用Grain持久化（`IPersistentState<T>`）在Grain中存储状态。
* 实现多个Grain接口的Grain
* 可重入Grain，其允许多个Grain调用同时执行，以单线程、交错的方式进行。
* 使用*Grain观察者*（`IGrainObserver`）来接收Grain的推送通知

## [GPS Tracker](https://github.com/dotnet/orleans/raw/main/samples/GPSTracker/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/GPSTracker/screenshot.jpeg?sanitize=true" />
</p>

一个在地图上追踪装有GPS的[物联网](https://zh.wikipedia.org/wiki/%E7%89%A9%E8%81%94%E7%BD%91)设备的服务。
设备的位置使用*SignalR*进行近乎实时的更新，因此这个示例展示了将Orleans与SignalR集成的一种方法。
设备的更新来自于*设备网关*，这是用一个单独的进程实现的，它连接到主服务并模拟一些设备以伪随机的方式在旧金山的某个区域移动。

**这个程序演示了：**

* 如何使用Orleans创建一个[物联网](https://zh.wikipedia.org/wiki/%E7%89%A9%E8%81%94%E7%BD%91)应用
* Orleans如何与[ASP.NET Core SignalR](https://docs.microsoft.com/aspnet/core/signalr/introduction)共同托管和整合
* 如何使用Orleans和SignalR将Grain的实时更新广播给一组用户

## [HanBaoBao](https://github.com/ReubenBond/hanbaobao-web)

<p align="center">
    <img src="https://github.com/ReubenBond/hanbaobao-web/blob/main/assets/demo-1.png?raw=true" />
</p>

一个英语-普通话词典的Web应用，展示了如何部署到Kubernetes、分流Grain调用以及请求节流。

**这个程序演示了：**

* 如何使用Orleans构建一个真实的应用
* 如何将基于Orleans的应用部署到Kubernetes上
* 如何将Orleans与ASP.NET Core和[*单页应用*](https://en.wikipedia.org/wiki/Single-page_application)的JavaScript框架（[Vue.js](https://vuejs.org/)）结合起来
* 如何实现漏斗式的请求节流
* 如何从数据库中加载和查询数据
* 如何懒且临时地缓存结果
* 如何将请求分流到许多Grain中并收集结果

## [Presence Service](https://github.com/dotnet/orleans/raw/main/samples/Presence/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/Presence/screenshot.png?sanitize=true" />
</p>

一个游戏在线服务，类似于为[Halo](https://www.xbox.com/games/halo)创建的一个基于Orleans的服务。
在线服务以近乎实时的方式追踪玩家和游戏会话。

**这个程序演示了：**

* 在现实世界中的应用的简化版本
* 使用*Grain观察者*（`IGrainObserver`）来接收Grain的推送通知


## [Tic Tac Toe](https://github.com/dotnet/orleans/raw/main/samples/TicTacToe/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/TicTacToe/logo.png?sanitize=true"/>
</p>

一个使用ASP.NET MVC、JavaScript和Orleans的基于Web的[井字棋](https://en.wikipedia.org/wiki/Tic-tac-toe)游戏。

**这个程序演示了：**

* 如何使用Orleans创建一个网络游戏
* 如何创建一个基本的游戏大厅系统
* 如何从ASP.NET Core MVC应用中访问Orleans Grain

## [Voting](https://github.com/dotnet/orleans/raw/main/samples/Voting/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/Voting/screenshot.png?sanitize=true"/>
</p>

一个用于对一组选项进行投票的Web应用。这个例子演示了在Kubernetes上的部署。
该应用使用[.NET通用主机](https://docs.microsoft.com/zh-cn/dotnet/core/extensions/generic-host)在同一进程中共同托管[ASP.NET Core](https://docs.microsoft.com/zh-cn/aspnet/core/)和Orleans以及[Orleans Dashboard](https://github.com/OrleansContrib/OrleansDashboard) 。

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/Voting/dashboard.png?sanitize=true"/>
</p>

**这个程序演示了：**

* 如何将基于Orleans的应用部署到Kubernetes上
* 如何配置[Orleans Dashboard](https://github.com/OrleansContrib/OrleansDashboard)

## [Chat Room](https://github.com/dotnet/orleans/raw/main/samples/ChatRoom/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/ChatRoom/screenshot.png?sanitize=true" />
</p>

使用[Orleans流](../streaming/index.md)构建的基于终端的聊天应用。

**这个程序演示了：**

* 如何使用Orleans创建聊天应用
* 如何使用[Orleans流](../streaming/index.md)

## [Bank Account](https://github.com/dotnet/orleans/raw/main/samples/BankAccount/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/BankAccount/assets/BankClient.png?sanitize=true"/>
</p>

模拟银行账户，使用ACID事务在一组账户之间转移随机金额。

**这个程序演示了：**

* 如何使用Orleans事务来安全地执行涉及多个具有ACID保证和可序列化隔离的有状态Grain的操作。

## [Blazor Server](https://github.com/dotnet/orleans/raw/main/samples/Blazor/BlazorServer/#readme) and [Blazor WebAssembly](https://github.com/dotnet/orleans/raw/main/samples/Blazor/BlazorWasm/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/Blazor/BlazorServer/screenshot.jpeg?sanitize=true"/>
</p>

这两个Blazor示例是以[Blazor入门教程](https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro)为基础，经过改编后用于Orleans的。
[Blazor WebAssembly](https://github.com/dotnet/orleans/raw/main/samples/Blazor/BlazorWasm/#readme)示例使用[Blazor WebAssembly托管模式](https://docs.microsoft.com/aspnet/core/blazor/hosting-models#blazor-webassembly)。
[Blazor服务器](https://github.com/dotnet/orleans/raw/main/samples/Blazor/BlazorServer/#readme)示例使用[Blazor服务器托管模型](https://docs.microsoft.com/aspnet/core/blazor/hosting-models#blazor-server)。
它们包含一个交互式计数器、一个TODO列表和一个天气服务。

**这个程序演示了：**

* 如何将ASP.NET Core Blazor服务器与Orleans进行整合
* 如何将ASP.NET Core Blazor WebAssembly（WASM）与Orleans进行整合

## [Stocks](https://github.com/dotnet/orleans/raw/main/samples/Stocks/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/Stocks/screenshot.png?sanitize=true" />
</p>

一个股票价格应用，使用HTTP调用从远程服务中获取价格，并将价格暂时缓存在一个Grain中。
一个[`BackgroundService`](https://docs.microsoft.com/aspnet/core/fundamentals/host/hosted-services#backgroundservice-base-class)定期从不同的`StockGrain`Grain中获取最新的股票价格，这些Grain对应着一组股票代号。

**这个程序演示了：**

* 如何在一个[`BackgroundService`](https://docs.microsoft.com/aspnet/core/fundamentals/host/hosted-services#backgroundservice-base-class)内使用Orleans。
* 如何在一个Grain中使用定时器
* 如何使用.NET的`HttpClient`进行外部服务调用，并将结果缓存在一个Grain中。

## [Transport Layer Security](https://github.com/dotnet/orleans/raw/main/samples/TransportLayerSecurity/#readme)

<p align="center">
    <img src="https://raw.githubusercontent.com/dotnet/orleans/main/samples/TransportLayerSecurity/screenshot.png?sanitize=true" />
</p>

一个*Hello, World!*应用，配置为使用双向[*传输层安全协议*](https://zh.wikipedia.org/wiki/%E5%82%B3%E8%BC%B8%E5%B1%A4%E5%AE%89%E5%85%A8%E6%80%A7%E5%8D%94%E5%AE%9A)来保证每个服务器之间的安全网络通信。

**这个程序演示了：**

* 如何Orleans配置双向TLS（mTLS）认证

## [Visual Basic Hello World](https://github.com/dotnet/orleans/raw/main/samples/VBHelloWorld/#readme)

一个使用Visual Basic的*Hello, World！*应用。

**这个程序演示了：**

* 如何使用Visual Basic开发基于Orleans的应用

## [F# Hello World](https://github.com/dotnet/orleans/raw/main/samples/FSharpHelloWorld/#readme)

一个使用F#的*Hello, World！*应用。

**这个程序演示了：**

* 如何使用F#开发基于Orleans的应用

## [Streaming: Pub/Sub Streams over Azure Event Hubs](https://github.com/dotnet/orleans/raw/main/samples/Streaming/Simple/#readme)

一个使用Orleans流的应用，以[Azure Event Hubs](https://azure.microsoft.com/services/event-hubs/)作为提供者和隐式订阅者。

**这个程序演示了：**

* 如何使用[Orleans流](../streaming/index.md)
* 如何使用`[ImplicitStreamSubscription(namespace)]`特性来隐式地将一个Grain订阅到具有相应id的流上
* 如何配置Orleans流以便与[Azure Event Hubs](https://azure.microsoft.com/services/event-hubs/)一起使用

## [Streaming: Custom Data Adapter](https://github.com/dotnet/orleans/raw/main/samples/Streaming/CustomDataAdapter/#readme)

一个使用Orleans流的应用，一个非Orleans的发布者推送到一个流，Grain通过*自定义数据适配器*消费这个流，这个适配器告诉Orleans如何解释流信息。

**这个程序演示了：**

* 如何使用[Orleans流](../streaming/index.md)
* 如何使用`[ImplicitStreamSubscription(namespace)]`属性来隐式地将一个Grain订阅到具有相应id的流上
* 如何配置Orleans流以便与[Azure Event Hubs](https://azure.microsoft.com/services/event-hubs/)一起使用
* 如何通过提供一个自定义的`EventHubDataAdapter`实现（一个自定义的数据适配器）来消费非Orleans发布者发布的流消息

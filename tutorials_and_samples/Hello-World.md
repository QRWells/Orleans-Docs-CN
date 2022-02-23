---
title: Hello World 示例应用
---

为了开始我们的Orleans之旅，首先打开[`IHelloGrain.cs`](https://github.com/dotnet/orleans/blob/main/samples/HelloWorld/IHelloGrain.cs)，你会发现以下接口声明：

``` csharp
public interface IHelloGrain : IGrainWithStringKey
{
    Task<string> SayHello(string greeting);
}
```

它定义了`IHelloGrain`Grain接口。
我们知道它是一个Grain接口，因为它实现了`IGrainWithStringKey`。
这意味着当我们想获得对实现了`IHelloGrain`的Grain的引用时，我们将使用一个字符串值来标识Grain实例。
在我们的例子中，正如你将要在后面的[`Program.cs`](https://github.com/dotnet/orleans/blob/main/samples/HelloWorld/Program.cs)中看到的，我们将使用字符串`"friend"`来标识我们希望与之通信的Grain，它也可以是任意字符串：

``` csharp
var friend = grainFactory.GetGrain<IHelloGrain>("friend");
```

现在，打开[`HelloGrain.cs`](https://github.com/dotnet/orleans/blob/main/samples/HelloWorld/HelloGrain.cs)，我们将看到`IHelloGrain`接口的实现：

``` csharp
public class HelloGrain : Grain, IHelloGrain
{
    public Task<string> SayHello(string greeting) => Task.FromResult($"Hello, {greeting}!");
}
```

我们知道`HelloGrain`是一个grain的实现，因为它继承了`Grain`基类。
该类是用来识别应用程序中的Grain类。

`HelloGrain`通过返回一个简单的字符串实现了`IHelloGrain`：`$"Hello, {greeting}"`。

打开[`Program.cs`](https://github.com/dotnet/orleans/blob/main/samples/HelloWorld/Program.cs)，看看Orleans是如何配置的：

``` csharp
using var host = new HostBuilder()
    .UseOrleans(builder =>
    {
        builder.UseLocalhostClustering();
    })
    .Build();

await host.StartAsync();

// Get the grain factory
var grainFactory = host.Services.GetRequiredService<IGrainFactory>();

// Get a reference to the HelloGrain grain with the key "friend".
var friend = grainFactory.GetGrain<IHelloGrain>("friend");

// Call the grain and print the result to the console
var result = await friend.SayHello("Good morning!"); 
Console.WriteLine("\n\n{0}\n\n", result);

Console.WriteLine("Orleans is running.\nPress Enter to terminate...");
Console.ReadLine();
Console.WriteLine("Orleans is stopping...");

await host.StopAsync();
```

这个程序创建了一个新的[`HostBuilder`](https://docs.microsoft.com/zh-cn/dotnet/core/extensions/generic-host)，并通过调用`UseOrleans`扩展方法将Orleans加入其中。
在该调用中，它将Orleans配置为使用本地主机集群，这一般用于开发和测试场景。
然后程序启动主机，并从服务提供者那里获取`IGrainFactory`实例。
使用`IGrainFactory`，我们可以得到一个Grain的*引用*。
在本例中，我们想要一个对名为`"friend"`的`HelloGrain`实例的引用，因此我们调用`grainFactory.GetGrain<IHelloGrain>("friend")`。
一旦我们有了一个引用，我们就可以使用它，调用`friend.SayHello("Good morning!")`，将结果打印到控制台。

是时候让这个示例运行起来了。通过在终端窗口执行以下命令来实现：

``` powershell
dotnet run
```

You should see `Hello, Good morning!!` printed to the console.
你应该看到控制台里打印出`Hello, Good morning!!`。

Orleans在我们第一次调用`HelloGrain`时（`friend.SayHello(...)`）自动为我们实例化了`"friend"`的实例。
作为开发者，我们不需要管理这些Grain的生命期。Orleans在需要时激活它们，当它们变得空闲时，就会停用它们。
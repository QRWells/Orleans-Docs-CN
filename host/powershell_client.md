---
title: PowerShell客户端模块
---

Orleans PowerShell客户端模块是一套[PowerShell Cmdlets](https://docs.microsoft.com/zh-cn/powershell/scripting/developer/cmdlet/cmdlet-overview?view=powershell-7.2)，它将[Grain客户端](https://github.com/dotnet/orleans/blob/master/src/Orleans/Core/GrainClient.cs)包装为一套易用的指令，使之不仅可以与[ManagementGrain](https://github.com/dotnet/orleans/blob/master/src/Orleans.Runtime/Core/ManagementGrain.cs)，还可以与任意`IGrain`使用PowerShell脚本进行交互，就像一般的Orleans应用一样。

这些Cmdlets通过Powershell脚本，实现了从启动维护任务、测试、监控或其他类型的自动化的一系列场景。

以下是使用示例：

## 安装模块

### 从源代码安装
你可以从源码构建`OrleansPSUtils`项目，并直接导入它：

``` powershell
PS> Import-Module .\projectOutputDir\Orleans.psd1

```

尽管你可以这样做，但有一个更简单和有趣的方法，即从**PowerShell Gallery**安装它。

### 从PowerShell Gallery安装

现今的Powershell模块就像Nuget包一样易于分享，但并不是在[nuget.org](nuget.org)，而是托管在[PowerShell Gallery](https://www.powershellgallery.com/)。

* 要将其安装到指定目录，只要运行：

``` powershell
PS> Save-Module -Name OrleansPSUtils -Path <path>

```

* 要将其安装到你的PowerShell模块路径（**推荐的方式**），只要运行：

``` powershell
PS> Install-Module -Name OrleansPSUtils

```

* 如果你打算在[Azure Automation](https://azure.microsoft.com/en-us/services/automation/)上使用这个模块，只需点击下面的按钮：
<button style="border:none;background-image:none; background-color:transparent " type="button" title="Deploy this module to Azure Automation." onclick="window.open('https://www.powershellgallery.com/packages/Orleans/DeployItemToAzureAutomation?itemType=PSModule', target = '_blank')">
	<img src="https://www.powershellgallery.com/Content/Images/DeployToAzureAutomationButton.png">
</button>

## 使用模块

Regardless of the way you decide to install it, the first thing you need to do in order to actually use it is import the module on the current PowerShell session so the Cmdlets get available by running this:
无论以何种方式安装它，要实际使用它，你需要做的第一件事是在当前的PowerShell会话中导入该模块，这样就可以使用Cmdlets：

``` powershell
PS> Import-Module OrleansPSUtils
```

**注意**:
如果从源码构建，你必须按照安装部分的建议导入，使用`.psd1`的路径而不是使用模块名称，因为它不会在`$env:PSModulePath`的PowerShell运行时变量上。
同样，强烈建议你从PowerShell Gallery安装。

导入模块后（这意味着它被加载到PowerShell会话中），你将有以下Cmdlets可用：

* `Start-GrainClient`
* `Stop-GrainClient`
* `Get-Grain`

#### Start-GrainClient

该模块是对`GrainClient.Initialize()`及其重载的封装。

**用法**:

* __`Start-GrainClient`__

  * 与调用`GrainClient.Initialize()`相同，它将查找已知的Orleans客户端配置文件名称

* __`Start-GrainClient [-ConfigFilePath] <string> [[-Timeout] <timespan>]`__

  * 将使用所提供的文件路径，与`GrainClient.Initialize(filePath)`一致

* __`Start-GrainClient [-ConfigFile] <FileInfo> [[-Timeout] <timespan>]`__

  * 使用代表配置文件的`System.FileInfo`类的实例，就像`GrainClient.Initialize(fileInfo)`一样

* __`Start-GrainClient [-Config] <ClientConfiguration> [[-Timeout] <timespan>]`__

  * 使用`Orleans.Runtime.Configuration.ClientConfiguration`的实例，就像`GrainClient.Initialize(config)`一样

* __`Start-GrainClient [-GatewayAddress] <IPEndPoint> [[-OverrideConfig] <bool>] [[-Timeout] <timespan>]`__

  * 集群网关地址Endpoint


**注意**:
参数`Timeout`是可选的，如果设置了并且大于`System.TimeSpan.Zero`，它将在内部调用`Orleans.GrainClient.SetResponseTimeout(Timeout)`。

#### Stop-GrainClient

不带任何参数，当调用时，如果`GrainClient`已经被初始化了，将适当地取消初始化。

#### Get-Grain

`GrainClient.GrainFactory.GetGrain<T>()`及其重载的封装。

必需参数是`-GrainType`和`-XXXKey`，用于当前Orleans支持的Grain键类型（`string`, `Guid`, `long`），还有`-KeyExtension`，可用于有复合键的Grain。

这个Cmdlet返回一个由参数`-GrainType`传递的Grain引用。

## 示例:

一个调用`MyInterfacesNamespace.IMyGrain.SayHeloTo`Grain方法的简单例子。

``` powershell
PS> Import-Module OrleansPSUtils
PS> $configFilePath = Resolve-Path(".\ClientConfig.xml").Path
PS> Start-GrainClient -ConfigFilePath $configFilePath
PS> Add-Type -Path .\MyGrainInterfaceAssembly.dll
PS> $grainInterfaceType = [MyInterfacesNamespace.IMyGrain]
PS> $grainId = [System.Guid]::Parse("A4CF7B5D-9606-446D-ACE9-C900AC6BA3AD")
PS> $grain = Get-Grain -GrainType $grainInterfaceType -GuidKey $grainId
PS> $message = $grain.SayHelloTo("Gutemberg").Result
PS> Write-Output $message
Hello Gutemberg!
PS> Stop-GrainClient
```

我们计划在引入更多的Cmdlets，比如在Powershell上更原生地使用观察者、流以及其他Orleans核心功能时，更新这个页面。
我们希望这能帮助人们作为自动化的一个起点。像往常一样，这是一项正在进行中的工作，我们欢迎大家的贡献。:)

注意，我们的目的不是在PowerShell上重新实现整个客户端，而是给IT和DevOps团队提供一种与Grains交互的方式，且不需要实现一个.Net应用程序。
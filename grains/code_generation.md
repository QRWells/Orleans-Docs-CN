---
title: 代码生成
description: Orleans运行时利用生成的代码，以确保在整个集群中使用的类的正确序列化。样板代码的生成也是如此，它抽象出方法运送、异常传播和其他内部运行时概念的实现细节。
---

## 启用代码生成

代码生成可以在你的项目正在构建时或你的应用程序初始化时进行。

### 构建期间

执行代码生成的首选方法在构建期间进行。构建时的代码生成可以通过使用以下包之一来启用：

+ [Microsoft.Orleans.OrleansCodeGenerator.Build](https://www.nuget.org/packages/Microsoft.Orleans.OrleansCodeGenerator.Build/)。一个使用Roslyn进行代码生成并使用.NET Reflection进行分析的包。
+ [Microsoft.Orleans.CodeGenerator.MSBuild](https://www.nuget.org/packages/Microsoft.Orleans.CodeGenerator.MSBuild/)。一个新的代码生成的包，在代码生成和代码分析方面都利用了Roslyn。它不加载应用程序的二进制文件，因此避免了因依赖版本冲突和目标框架不同而引起的问题。新的代码生成器还改进了对增量构建的支持，这有助于缩短构建时间。

这些包中的一个应该被安装到所有包含Grain、Grain接口、自定义序列化器或Grain间发送的类的项目中。安装一个包会在项目中注入一个目标，并在构建时产生代码。

这两个包（`Microsoft.Orleans.CodeGenerator.MSBuild`和`Microsoft.Orleans.OrleansCodeGenerator.Build`）都只支持C#项目。其他语言的项目由下述的`Microsoft.Orleans.OrleansCodeGenerator`包提供支持，或者通过创建一个C#项目来作为由其他语言编写的程序集生成的代码的目标。

通过在目标项目的*csproj*文件中为`OrleansCodeGenLogLevel`指定一个值，可以在构建时进行额外的诊断。例如，`<OrleansCodeGenLogLevel>Trace</OrleansCodeGenLogLevel>`。

### 初始化期间

通过安装`Microsoft.Orleans.OrleansCodeGenerator`包并使用`IApplicationPartManager.WithCodeGeneration`扩展方法，可以在客户端和Silo的初始化期间进行代码生成。

``` csharp
builder.ConfigureApplicationParts(
    parts => parts
        .AddApplicationPart(typeof(IRuntimeCodeGenGrain).Assembly)
        .WithCodeGeneration());
```

在上述例子中，`builder`可以是`ISiloHostBuilder`或`IClientBuilder`的一个实例。一个可选的[`ILoggerFactory`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.iloggerfactory)实例可以传递给`WithCodeGeneration`，以便在代码生成期间启用日志，例如：

``` csharp
ILoggerFactory codeGenLoggerFactory = new LoggerFactory();
codeGenLoggerFactory.AddProvider(new ConsoleLoggerProvider());
builder.ConfigureApplicationParts(
    parts => parts
        .AddApplicationPart(typeof(IRuntimeCodeGenGrain).Assembly)
        .WithCodeGeneration(codeGenLoggerFactory));
```

## 干涉代码生成

### 为一个特定的类生成代码

代码会自动生成Grain接口、Grain类、Grain状态以及在Grain方法中作为参数传递的类。如果一个类不符合这个标准，可以使用以下方法来进一步指导代码生成。

在一个类上添加`[Serializable]`，指示代码生成器为该类生成一个序列化器。

在项目中添加`[assembly: GenerateSerializer(Type)]`，指示代码生成器将该类型视为可序列化的，如果无法为该类型生成序列化器，例如因为该类型不可访问，则会导致错误。如果代码生成被启用，这个错误将中止构建。这个属性也允许从另一个程序集为特定类型生成代码。

`[assembly: KnownType(Type)]`也指示代码生成器包含一个特定的类型（可能来自一个被引用的程序集），但如果该类型是不可访问的，则不会引起异常。

### 为所有子类生成序列化器

将`[KnownBaseType]`添加到一个接口或类中，指示代码生成器为所有继承/实现该类型的类型生成序列化代码。

### 为另一个程序集中的所有类生成代码

在有些情况下，生成的代码在构建时不能被包含在一个特定的程序集中。例如，这可能包括不引用Orleans的共享库，用C#以外的语言编写的程序集，以及开发者没有源代码的程序集。在这些情况下，为这些程序集生成的代码可以放在一个单独的程序集中，并在初始化时被引用。

为了使一个程序集能够这样做：

1. 创建一个C#项目。
2. 安装`Microsoft.Orleans.CodeGenerator.MSBuild`或`Microsoft.Orleans.OrleansCodeGenerator.Build`包。
3. 添加一个对目标程序集的引用。
4. 在C#文件的顶层添加`[assembly: KnownAssembly("OtherAssembly")]`。

`KnownAssembly`属性指示代码生成器检查指定的程序集，并为其中的类型生成代码。该属性可以在一个项目中多次使用。

然后，在初始化过程中，必须将生成的程序集添加到客户端/Silo中：

``` csharp
builder.ConfigureApplicationParts(
    parts => parts.AddApplicationPart("CodeGenAssembly"));
```

在前面的例子中，`builder`可以是`ISiloHostBuilder`或`IClientBuilder`的一个实例。

`KnownAssemblyAttribute`有一个可选的属性，`TreatTypesAsSerializable`，它可以被设置为`true`，以指示代码生成器像该程序集中的所有类型被标记为可序列化一样工作。

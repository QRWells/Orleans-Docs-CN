---
title: Grain调用拦截器
description: 本节介绍了Orleans中的调用拦截器及其相关应用
---

Grain调用过滤器提供了一种拦截Grain调用的方法。过滤器可以在Grain调用之前和之后执行代码。可以同时设置多个过滤器。过滤器是异步的，它可以修改`RequestContext`、参数和被调用方法的返回值。过滤器还可以检查被调用的方法的`MethodInfo`，可以用来抛出或处理异常。

Grain调用过滤器的一些使用示例：

* 授权：过滤器可以检查被调用的方法和参数或`RequestContext`中的一些授权信息，以确定是否允许调用继续执行。
* 日志/遥测：过滤器可以记录信息，捕捉时间数据和其他关于方法调用的统计数据。
* 错误处理：过滤器可以拦截由方法调用抛出的异常，并将其转化为另一个异常或者在异常通过过滤器时进行处理。

过滤器有两种类型：

* 传入调用过滤器
* 传出调用过滤器

传入调用过滤器在接受调用时执行；传出调用过滤器在发起调用时执行。

## 传入调用过滤器

传入Grain调用过滤器应实现`IIncomingGrainCallFilter`接口，其包含一个方法：

``` csharp
public interface IIncomingGrainCallFilter
{
    Task Invoke(IIncomingGrainCallContext context);
}
```

传递给`Invoke`方法的参数`IIncomingGrainCallContext`具体如下：

``` csharp
public interface IIncomingGrainCallContext
{
    /// <summary>
    /// Gets the grain being invoked.
    /// </summary>
    IAddressable Grain { get; }

    /// <summary>
    /// Gets the <see cref="MethodInfo"/> for the interface method being invoked.
    /// </summary>
    MethodInfo InterfaceMethod { get; }

    /// <summary>
    /// Gets the <see cref="MethodInfo"/> for the implementation method being invoked.
    /// </summary>
    MethodInfo ImplementationMethod { get; }

    /// <summary>
    /// Gets the arguments for this method invocation.
    /// </summary>
    object[] Arguments { get; }

    /// <summary>
    /// Invokes the request.
    /// </summary>
    Task Invoke();

    /// <summary>
    /// Gets or sets the result.
    /// </summary>
    object Result { get; set; }
}
```

`IIncomingGrainCallFilter.Invoke(IIncomingGrainCallContext)`方法必须等待或返回`IIncomingGrainCallContext.Invoke()`的结果，以执行下一个配置好的过滤器，最后执行Grain方法本身。在等待`Invoke()`方法后，`Result`属性可以被修改。`ImplementationMethod`属性返回实现类的`MethodInfo`。接口方法的`MethodInfo`可以使用`InterfaceMethod`属性来访问。Grain的调用过滤器对所有Grain方法的调用都生效，也包括Grain扩展（`IGrainExtension`的实现）的调用，这些扩展安装在Grain中。例如，Grain扩展被用来实现流和取消令牌。因此，应该假设`ImplementationMethod`的值不一定是是Grain类本身的一个方法。
### 配置传入调用过滤器

`IIncomingGrainCallFilter`的实现可以通过依赖注入注册为Silo级过滤器，也可以通过一个实现了`IIncomingGrainCallFilter`的Grain直接注册为Grain级过滤器。

#### Silo级Grain调用过滤器

一个委托可以像这样使用依赖注入注册为一个Silo级Grain调用过滤器：

``` csharp
siloHostBuilder.AddIncomingGrainCallFilter(async context =>
{
    // If the method being called is 'MyInterceptedMethod', then set a value
    // on the RequestContext which can then be read by other filters or the grain.
    if (string.Equals(context.InterfaceMethod.Name, nameof(IMyGrain.MyInterceptedMethod)))
    {
        RequestContext.Set("intercepted value", "this value was added by the filter");
    }

    await context.Invoke();

    // If the grain method returned an int, set the result to double that value.
    if (context.Result is int resultValue) context.Result = resultValue * 2;
});
```

同样地，一个类可以使用`AddIncomingGrainCallFilter`方法注册为一个Grain调用过滤器。
下面是一个Grain调用过滤器的例子，它记录了每个Grain方法的结果：

```csharp
public class LoggingCallFilter : IIncomingGrainCallFilter
{
    private readonly Logger log;

    public LoggingCallFilter(Factory<string, Logger> loggerFactory)
    {
        this.log = loggerFactory(nameof(LoggingCallFilter));
    }

    public async Task Invoke(IIncomingGrainCallContext context)
    {
        try
        {
            await context.Invoke();
            var msg = string.Format(
                "{0}.{1}({2}) returned value {3}",
                context.Grain.GetType(),
                context.InterfaceMethod.Name,
                string.Join(", ", context.Arguments),
                context.Result);
            this.log.Info(msg);
        }
        catch (Exception exception)
        {
            var msg = string.Format(
                "{0}.{1}({2}) threw an exception: {3}",
                context.Grain.GetType(),
                context.InterfaceMethod.Name,
                string.Join(", ", context.Arguments),
                exception);
            this.log.Info(msg);

            // If this exception is not re-thrown, it is considered to be
            // handled by this filter.
            throw;
        }
    }
}
```

然后可以使用`AddIncomingGrainCallFilter`扩展方法来注册这个过滤器：

``` csharp
siloHostBuilder.AddIncomingGrainCallFilter<LoggingCallFilter>();
```

另外，也可以不使用扩展方法来注册过滤器：

``` csharp
siloHostBuilder.ConfigureServices(
    services => services.AddSingleton<IIncomingGrainCallFilter, LoggingCallFilter>());
```

#### 逐Grain的Grain调用过滤器

一个Grain类可以将自己注册为Grain调用过滤器，并通过实现`IIncomingGrainCallFilter`来过滤对它的任何调用，就像这样：

```csharp
public class MyFilteredGrain : Grain, IMyFilteredGrain, IIncomingGrainCallFilter
{
    public async Task Invoke(IIncomingGrainCallContext context)
    {
        await context.Invoke();

        // Change the result of the call from 7 to 38.
        if (string.Equals(context.InterfaceMethod.Name, nameof(this.GetFavoriteNumber)))
        {
            context.Result = 38;
        }
    }

    public Task<int> GetFavoriteNumber() => Task.FromResult(7);
}
```

在上面的例子中，所有对`GetFavoriteNumber`方法的调用将返回`38`而不是`7`，因为返回值已经被过滤器修改了。

过滤器的另一个使用场景是访问控制，像下面这个例子：

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class AdminOnlyAttribute : Attribute { }

public class MyAccessControlledGrain : Grain, IMyFilteredGrain, IIncomingGrainCallFilter
{
    public Task Invoke(IIncomingGrainCallContext context)
    {
        // Check access conditions.
        var isAdminMethod = context.ImplementationMethod.GetCustomAttribute<AdminOnlyAttribute>();
        if (isAdminMethod && !(bool) RequestContext.Get("isAdmin"))
        {
            throw new AccessDeniedException($"Only admins can access {context.ImplementationMethod.Name}!");
        }

        return context.Invoke();
    }

    [AdminOnly]
    public Task<int> SpecialAdminOnlyOperation() => Task.FromResult(7);
}
```

在上面的例子中，只有在`RequestContext`中`"isAdmin"`被设置为`true`时，才能调用`SpecialAdminOnlyOperation`方法。通过这种方式，Grain调用过滤器可用于授权。在这个例子中，调用者负责确保`"isAdmin"`值被正确设置，并正确进行认证。请注意，`[AdminOnly]`特性是在Grain类方法上指定的。这是因为`ImplementationMethod`属性返回实现的`MethodInfo`，而不是接口的。过滤器也可以检查`InterfaceMethod`属性。

### Grain调用过滤器的顺序

Grain调用过滤器遵循定义好的顺序：

1. 在依赖注入容器中配置的`IIncomingGrainCallFilter`实现会按照它们被注册的顺序。
2. 实现了`IIncomingGrainCallFilter`的Grain级过滤器。
3. Grain方法的实现或Grain扩展方法的实现。

对`IIncomingGrainCallContext.Invoke()`的每次调用都封装了下一个定义好的过滤器，这样每个过滤器都可以在过滤器链中的下一个过滤器前后执行代码，并最终执行Grain方法本身。

## 传出调用过滤器

传出Grain调用过滤器类似于传入Grain调用过滤器，主要区别在于它们是在调用者（客户端）而不是被调用者（Grain）上被调用。

传出Grain调用过滤器应实现`IOutgoingGrainCallFilter`接口，其包含一个方法：

``` csharp
public interface IOutgoingGrainCallFilter
{
    Task Invoke(IOutgoingGrainCallContext context);
}
```

传递给`Invoke`方法的参数`IOutgoingGrainCallContext`具体如下：

``` csharp
public interface IOutgoingGrainCallContext
{
    /// <summary>
    /// Gets the grain being invoked.
    /// </summary>
    IAddressable Grain { get; }

    /// <summary>
    /// Gets the <see cref="MethodInfo"/> for the interface method being invoked.
    /// </summary>
    MethodInfo InterfaceMethod { get; }

    /// <summary>
    /// Gets the arguments for this method invocation.
    /// </summary>
    object[] Arguments { get; }

    /// <summary>
    /// Invokes the request.
    /// </summary>
    Task Invoke();

    /// <summary>
    /// Gets or sets the result.
    /// </summary>
    object Result { get; set; }
}
```

`IOutgoingGrainCallFilter.Invoke(IOutgoingGrainCallContext)`方法必须等待或返回`IOutgoingGrainCallContext.Invoke()`的结果，以执行下一个配置好的过滤器，最后执行Grain方法本身。在等待`Invoke()`方法后，`Result`属性可以被修改。`ImplementationMethod`属性返回实现类的`MethodInfo`。接口方法的`MethodInfo`可以使用`InterfaceMethod`属性来访问。传出Grain调用过滤器对调用Grain的方法都生效，也包括Orleans对系统方法的调用。

### 配置传出Grain调用过滤器

`IOutgoingGrainCallFilter`的实现可以通过依赖注入的方式在Silo以及客户端上注册。

一个委托可以像这样注册为一个调用过滤器：

``` csharp
builder.AddOutgoingGrainCallFilter(async context =>
{
    // If the method being called is 'MyInterceptedMethod', then set a value
    // on the RequestContext which can then be read by other filters or the grain.
    if (string.Equals(context.InterfaceMethod.Name, nameof(IMyGrain.MyInterceptedMethod)))
    {
        RequestContext.Set("intercepted value", "this value was added by the filter");
    }

    await context.Invoke();

    // If the grain method returned an int, set the result to double that value.
    if (context.Result is int resultValue) context.Result = resultValue * 2;
});
```

在上述代码中，`builder`可以是`ISiloHostBuilder`或`IClientBuilder`的实例。

同样地，一个类可以被注册为一个传出Grain调用过滤器。
下面是一个Grain调用过滤器的例子，它记录了每个Grain方法的结果：

```csharp
public class LoggingCallFilter : IOutgoingGrainCallFilter
{
    private readonly Logger log;

    public LoggingCallFilter(Factory<string, Logger> loggerFactory)
    {
        this.log = loggerFactory(nameof(LoggingCallFilter));
    }

    public async Task Invoke(IOutgoingGrainCallContext context)
    {
        try
        {
            await context.Invoke();
            var msg = string.Format(
                "{0}.{1}({2}) returned value {3}",
                context.Grain.GetType(),
                context.InterfaceMethod.Name,
                string.Join(", ", context.Arguments),
                context.Result);
            this.log.Info(msg);
        }
        catch (Exception exception)
        {
            var msg = string.Format(
                "{0}.{1}({2}) threw an exception: {3}",
                context.Grain.GetType(),
                context.InterfaceMethod.Name,
                string.Join(", ", context.Arguments),
                exception);
            this.log.Info(msg);

            // If this exception is not re-thrown, it is considered to be
            // handled by this filter.
            throw;
        }
    }
}
```

然后可以使用`AddOutgoingGrainCallFilter`扩展方法注册这个过滤器：

``` csharp
builder.AddOutgoingGrainCallFilter<LoggingCallFilter>();
```

另外，也可以不使用扩展方法来注册过滤器：

``` csharp
builder.ConfigureServices(
    services => services.AddSingleton<IOutgoingGrainCallFilter, LoggingCallFilter>());
```

与委托调用过滤器的例子一样，`builder`可以是`ISiloHostBuiler`或`IClientBuilder`的一个实例。

### 使用案例

#### 异常转换

当一个从服务器抛出的异常在客户端被反序列化时，有时你会得到后述的异常，而不是真正的那个异常：`TypeLoadException: Could not find Whatever.dll.`。

如果包含异常的程序集对客户端不可用，就会发生这种情况。例如，假设你在你的Grain实现中使用了Entity Framework，那么就有可能抛出一个`EntityException`。另一方面，客户端不会（也不应该）引用`EntityFramework.dll`，因为它不访问底层的数据访问层。

当客户端试图反序列化`EntityException`时，由于缺少DLL，它无法反序列化，于是是抛出了一个隐藏了原始`EntityException`的`TypeLoadException`。

可能有人会说这很好，因为客户端永远不会处理`EntityException`；否则它就必须引用`EntityFramework.dll`。

但是，如果客户端仅仅是想记录这个异常怎么办？问题是，原始的错误信息丢失。解决这个问题的一个方法是拦截服务器端的异常，如果异常类型对于客户端可能是未知的，就用`Exception`类型的普通异常来代替。

然而，有一件重要的事情我们必须记住：**如果调用者是Grain客户端**，我们只想替换一个异常；如果调用者是另一个Grain（或同样在进行Grain调用的Orleans基础设施，例如，在`GrainBasedReminderTable` Grain上），我们不希望替换异常。

在服务器端，这可以通过一个Silo级拦截器来完成：

```csharp
public class ExceptionConversionFilter : IIncomingGrainCallFilter
{
    private static readonly HashSet<string> KnownExceptionTypeAssemblyNames =
        new HashSet<string>
        {
            typeof(string).Assembly.GetName().Name,
            "System",
            "System.ComponentModel.Composition",
            "System.ComponentModel.DataAnnotations",
            "System.Configuration",
            "System.Core",
            "System.Data",
            "System.Data.DataSetExtensions",
            "System.Net.Http",
            "System.Numerics",
            "System.Runtime.Serialization",
            "System.Security",
            "System.Xml",
            "System.Xml.Linq",

            "MyCompany.Microservices.DataTransfer",
            "MyCompany.Microservices.Interfaces",
            "MyCompany.Microservices.ServiceLayer"
        };

    public async Task Invoke(IIncomingGrainCallContext context)
    {
        var isConversionEnabled =
            RequestContext.Get("IsExceptionConversionEnabled") as bool? == true;
        if (!isConversionEnabled)
        {
            // If exception conversion is not enabled, execute the call without interference.
            await context.Invoke();
            return;
        }

        RequestContext.Remove("IsExceptionConversionEnabled");
        try
        {
            await context.Invoke();
        }
        catch (Exception exc)
        {
            var type = exc.GetType();

            if (KnownExceptionTypeAssemblyNames.Contains(type.Assembly.GetName().Name))
            {
                throw;
            }

            // Throw a base exception containing some exception details.
            throw new Exception(
                string.Format(
                    "Exception of non-public type '{0}' has been wrapped."
                    + " Original message: <<<<----{1}{2}{3}---->>>>",
                    type.FullName,
                    Environment.NewLine,
                    exc,
                    Environment.NewLine));
        }
    }
}
```

这个拦截器可以在Silo上注册：

``` csharp
siloHostBuilder.AddIncomingGrainCallFilter<ExceptionConversionFilter>();
```

通过添加一个传出调用过滤器，对客户端的调用启用过滤器：

```csharp
clientBuilder.AddOutgoingGrainCallFilter(context =>
{
    RequestContext.Set("IsExceptionConversionEnabled", true);
    return context.Invoke();
});
```

这样，客户端告诉服务器，它想使用异常转换。

#### 在拦截器中调用Grain

通过在拦截器类中注入`IGrainFactory`，可以在拦截器中进行Grain调用：

``` csharp
private readonly IGrainFactory grainFactory;

public CustomCallFilter(IGrainFactory grainFactory)
{
  this.grainFactory = grainFactory;
}

public async Task Invoke(IIncomingGrainCallContext context)
{
  // Hook calls to any grain other than ICustomFilterGrain implementations.
  // This avoids potential infinite recursion when calling OnReceivedCall() below.
  if (!(context.Grain is ICustomFilterGrain))
  {
    var filterGrain = this.grainFactory.GetGrain<ICustomFilterGrain>(context.Grain.GetPrimaryKeyLong());

    // Perform some grain call here.
    await filterGrain.OnReceivedCall();
  }

  // Continue invoking the call on the target grain.
  await context.Invoke();
}
```
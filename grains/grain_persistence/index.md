---
description: 本节将介绍Orleans中的持久化手段
---

# 持久化

Grains可以有多个与之关联的具名持久化数据对象。这些状态对象会在Grain激活期间从存储器中加载，以便其在请求时可用。
Grains的持久化采用了一个可扩展插件模型，从而可以使用任意数据库的存储提供商。这一持久化模型是为了简易性设计的，并不打算涵盖所有数据访问模式。
Grains也可以不使用Grain持久化模型，直接访问数据库。

![一个Grain可以有多个存储在不同储存系统中的持久化数据对象](https://raw.githubusercontent.com/dotnet/orleans-docs/main/src/images/grain_state_1.png)

在上图中，UserGrain有一个*Profile*状态和一个*Cart*状态，每个状态都存储在一个单独的存储系统中。

## 宗旨

1. 每个Grain有多个具名的持久化数据对象。
2. 多个可配置的存储提供商，每个都可以有不同的配置，并基于不同的存储系统。
3. 存储提供商可以由社区开发并发布。
4. 存储提供商可以完全控制它们如何在持久化底层存储中存储Grain状态的数据。推论：Orleans并没有提供全面的ORM存储解决方案，而是允许自定义存储提供商在需要时支持特定的ORM需求。

## 相关的包

Orleans grain的存储提供商可以在[NuGet](https://www.nuget.org/packages?q=Orleans+Persistence)上找到。官方维护的包有：

* [Microsoft.Orleans.Persistence.AdoNet](https://www.nuget.org/packages/Microsoft.Orleans.Persistence.AdoNet)用于SQL数据库和其他由ADO.NET支持的存储系统。更多信息请参见[ADO.NET Grain持久化](relational_storage.md)。
* [Microsoft.Orleans.Persistence.AzureStorage](https://www.nuget.org/packages/Microsoft.Orleans.Persistence.AzureStorage)用于Azure，包括Azure Blob Storage，Azure Table Storage以及Azure CosmosDB，通过使用Azure Table Storage API。更多信息请参见[Azure Storage Grain持久化](azure_storage.md)。
* [Microsoft.Orleans.Persistence.DynamoDB](https://www.nuget.org/packages/Microsoft.Orleans.Persistence.DynamoDB)用于Amazon DynamoDB。更多信息请参见[Amazon DynamoDB Grain持久化](dynamodb_storage.md)。

## API

Grains使用`IPersistentState<TState>`接口与它们的持久化状态进行交互，其中`TState`是可序列化的状态类型：

``` csharp
public interface IPersistentState<TState> where TState : new()
{
  TState State { get; set; }
  string Etag { get; }
  Task ClearStateAsync();
  Task WriteStateAsync();
  Task ReadStateAsync();
}
```

`IPersistentState<TState>`的实例作为构造函数的参数被注入到Grain中。
这些参数可以用`[PersistentState(stateName, storageName)]`属性来注解，以识别被注入的状态的名称和提供该状态的存储提供商的名称。
下面的例子通过向`UserGrain`构造函数注入两个具名状态来证明这一点。

``` csharp
public class UserGrain : Grain, IUserGrain
{
  private readonly IPersistentState<ProfileState> _profile;
  private readonly IPersistentState<CartState> _cart;

  public UserGrain(
    [PersistentState("profile", "profileStore")] IPersistentState<ProfileState> profile,
    [PersistentState("cart", "cartStore")] IPersistentState<CartState> cart,
    )
  {
    _profile = profile;
    _cart = cart;
  }
}
```

不同的Grain类型可以使用不同的可配置的存储提供商，即使两者是同一类型。例如，两个不同的Azure Table的存储提供商实例，连接到了不同的Azure Storage账户。

### 读取状态

当Grain被激活时，Grain状态将被自动读取，但在必要时，Grain负责显式触发任何发生改变的Grain状态的写入。

如果一个Grain想要显式地从后台存储中重新读取最新的状态，它应该调用`ReadStateAsync()`方法。
这将会通过存储提供商从持久化存储中重新加载Grain状态，当`ReadStateAsync()`这一`Task`完成时，先前的Grain状态的内存副本将被覆盖并替换。

状态的值是通过`State`属性来访问的。例如，下面的方法可以访问上面代码中声明的profile状态：

``` csharp
public Task<string> GetNameAsync() => Task.FromResult(_profile.State.Name);
```

在正常操作中不需要调用`ReadStateAsync()`，状态会在激活时自动加载。但是，`ReadStateAsync()`可以用来刷新被外部修改的状态。

有关错误处理机制的详情，请参见下面的[故障模式](#持久化操作的故障模式)部分。

### 写入状态

状态也可以通过`State`属性进行修改。修改后的状态不会被自动持久化。
相反，开发者可以通过调用`WriteStateAsync()`方法来决定何时持久化状态。
例如，下面的方法更新了`State`上的一个属性并持久化了更新后的状态：

``` csharp
public async Task SetNameAsync(string name)
{
  _profile.State.Name = name;
  await _profile.WriteStateAsync();
}
```

从概念上讲，Orleans运行时将在任何写操作期间对Grain状态数据对象进行深拷贝，供其自身使用。
在某些情况下，运行时 _可能_ 会使用优化规则和启发式方法来避免执行部分或全部的深拷贝，前提是要保留期望的逻辑隔离语义。

有关错误处理机制的详情，请参见下面的[故障模式](#持久化操作的故障模式)部分。

### 清除状态

`ClearStateAsync()`方法会清除存储中的Grain状态。根据Provider的情况，这个操作可以选择完全删除Grain的状态。

## 入门指南

在Grain可以使用持久化之前，必须在Silo上配置一个存储提供商。

首先，配置存储提供商，一个用于profile状态，一个用于cart状态：

``` csharp
var host = new HostBuilder()
  .UseOrleans(siloBuilder =>
  {
    // 使用"profileStore"名称配置Azure Table存储
    siloBuilder.AddAzureTableGrainStorage(
      name: "profileStore",
      configureOptions: options =>
      {
        // 使用JSON来序列化存储里的状态
        options.UseJson = true;

        // 配置存储的连接key
        options.ConnectionString = "DefaultEndpointsProtocol=https;AccountName=data1;AccountKey=SOMETHING1";
      })

      // 使用"cartStore"名称配置Azure Blob存储
      .AddAzureBlobGrainStorage(
        name: "cartStore",
        configureOptions: options =>
        {
            // 使用JSON来序列化存储里的状态
            options.UseJson = true;

            // 配置存储的连接key
            options.ConnectionString = "DefaultEndpointsProtocol=https;AccountName=data2;AccountKey=SOMETHING2";
        });
    // -- 其他选项
  })
  .Build();
```

现在我们已经配置了一个名为`"profileStore"`的存储提供商，我们可以从一个Grain中访问这个Provider。

持久化状态主要通过两种方式添加到Grain中：

1. 将`IPersistentState<TState>`注入到Grain的构造函数中
2. 继承`Grain<TState>`

我们推荐的方法是将`IPersistentState<TState>`注入到Grain的构造函数中，并使用相关联的`[PersistentState("stateName", "providerName")]`特性来向Grain添加存储。
有关`Grain<TState>`的详细信息，见[下文](#使用graintstate将存储加进grain)。这种方法仍然受支持，但被视为老旧的方法。 

声明一个类来容纳我们的Grain的状态：

``` csharp
[Serializable]
public class ProfileState
{
  public string Name { get; set; }

  public Date DateOfBirth
}
```

将`IPersistentState<TState>`注入到Grain的构造函数中

``` csharp
public class UserGrain : Grain, IUserGrain
{
  private readonly IPersistentState<ProfileState> _profile;

  public UserGrain([PersistentState("profile", "profileStore")] IPersistentState<ProfileState> profile)
  {
    _profile = profile;
  }
}
```

注意：profile状态在被注入构造函数时不会被加载，所以在那时是无法访问它的。
状态将在`OnActivateAsync`被调用之前被加载。

现在Grain有了持久化状态，我们可以添加方法来读取和写入状态：

``` csharp
public class UserGrain : Grain, IUserGrain
{
  private readonly IPersistentState<ProfileState> _profile;

  public UserGrain([PersistentState("profile", "profileStore")] IPersistentState<ProfileState> profile)
  {
    _profile = profile;
  }

  public Task<string> GetNameAsync() => Task.FromResult(_profile.State.Name);

  public async Task SetNameAsync(string name)
  {
    _profile.State.Name = name;
    await _profile.WriteStateAsync();
  }
}
```

## 持久化操作的故障模式

### 读操作的故障模式

在最初读取某个Grain的状态数据时，存储提供商返回的故障将导致该Grain的激活操作失败。
在这种情况下，将 _不会_ 对该Grain的生命周期回调方法`OnActivateAsync()`进行任何调用。
发送给Grain并引发其激活的原始请求会把故障返回给调用者，这像Grain激活期间的其他故障一样。
存储提供商在为某个Grain读取状态数据时遇到的故障会使`ReadStateAsync()` `Task`抛出异常。
Grain可以选择处理或忽略`Task`异常，就像Orleans的其他`Task`一样。

若一个Grain在Silo启动时因缺少或损坏的存储提供商配置而无法加载，那么向该Grain发送消息将会返回永久错误`Orleans.BadProviderConfigException`。

### 写操作的故障模式

存储提供商在为某一Grain写入状态数据时遇到的故障将导致`WriteStateAsync()` `Task`抛出一个异常。
这通常意味着，只要`WriteStateAsync()` `Task`被正确地连接进这个Grain方法最终返回的`Task`，Grain调用异常就会被抛回给客户端调用者。
然而，在某些进阶场景中，有可能编写Grain代码来专门处理这种写入错误，就像它们可以处理任何其他故障的`Task`一样。

执行错误处理或恢复代码的Grains必须捕获异常或故障的`WriteStateAsync()` `Task`，且不重新抛出，以表示它们已经成功地处理了写入错误。

## 建议

### 使用JSON序列化或另一种具有版本容错的序列化格式

代码随着时间的推移而改进，这往往也包括存储类型。
为了适应这些变化，应该配置一个合适的序列化器。
对于大多数存储提供商来说，`UseJson`选项或类似的选项可用于使用JSON作为序列化格式。
确保在改进数据合约时，已经存储的数据仍然可以加载。

## 使用`Grain<TState>`将存储加进Grain

**注意：** 使用`Grain<T>`为Grain添加存储是*老旧*的功能：Grain存储应该使用`IPersistentState<T>`来添加，如之前所述。

继承了`Grain<T>`的Grain类（其中`T`是需要持久化的特定应用状态数据类型）将从指定的存储中自动加载其状态。

这样的Grains被标记为`[StorageProvider]`特性，它指定了一个存储提供商的具名实例，用于读取/写入该Grain的状态数据。

``` csharp
[StorageProvider(ProviderName="store1")]
public class MyGrain : Grain<MyGrainState>, /*...*/
{
  /*...*/
}
```

`Grain<T>`基类定义了如下方法给派生类调用：

``` csharp
protected virtual Task ReadStateAsync() { /*...*/ }
protected virtual Task WriteStateAsync() { /*...*/ }
protected virtual Task ClearStateAsync() { /*...*/ }
```

这些方法的行为对应于前文定义的`IPersistentState<TState>`中的对应方法。

## 创建一个存储提供商

状态持久化API有两部分：通过`IPersistentState<T>`或`Grain<T>`暴露给Grain的API，以及以`IGrainStorage`为中心的存储提供商 API，
这些是存储提供商必须实现的接口：

``` csharp
/// <summary>
/// Interface to be implemented for a storage able to read and write Orleans grain state data.
/// </summary>
public interface IGrainStorage
{
  /// <summary>Read data function for this storage instance.</summary>
  /// <param name="grainType">Type of this grain [fully qualified class name]</param>
  /// <param name="grainReference">Grain reference object for this grain.</param>
  /// <param name="grainState">State data object to be populated for this grain.</param>
  /// <returns>Completion promise for the Read operation on the specified grain.</returns>
  Task ReadStateAsync(string grainType, GrainReference grainReference, IGrainState grainState);

  /// <summary>Write data function for this storage instance.</summary>
  /// <param name="grainType">Type of this grain [fully qualified class name]</param>
  /// <param name="grainReference">Grain reference object for this grain.</param>
  /// <param name="grainState">State data object to be written for this grain.</param>
  /// <returns>Completion promise for the Write operation on the specified grain.</returns>
  Task WriteStateAsync(string grainType, GrainReference grainReference, IGrainState grainState);

  /// <summary>Delete / Clear data function for this storage instance.</summary>
  /// <param name="grainType">Type of this grain [fully qualified class name]</param>
  /// <param name="grainReference">Grain reference object for this grain.</param>
  /// <param name="grainState">Copy of last-known state data object for this grain.</param>
  /// <returns>Completion promise for the Delete operation on the specified grain.</returns>
  Task ClearStateAsync(string grainType, GrainReference grainReference, IGrainState grainState);
}
```

通过实现这个接口并[注册](#注册一个storage-provider)这个实现来创建一个自定义存储提供商。
对于现有的存储提供商的实现的示例，请参见[`AzureBlobGrainStorage`](https://github.com/dotnet/orleans/blob/af974d37864f85bfde5dc02f2f60bba997f2162d/src/Azure/Orleans.Persistence.AzureStorage/Providers/Storage/AzureBlobStorage.cs)。

### 存储提供商语义

不透明的，因provider而异（provider-specific）的`Etag`值（`string`）_可能_ 会被存储提供商设为Grain状态的一部分，并在读取时填进Grain状态的元数据里。
有些存储提供商可能不使用`Etag`，并将其设为`null`。

当存储提供商检测到`Etag`的约束被违反时，所有试图进行写操作的尝试都 _应该_ 引发写入`Task`的瞬时错误`Orleans.InconsistentStateException`，并封装
底层存储异常。

``` csharp
public class InconsistentStateException : OrleansException
{
  public InconsistentStateException(
    string message,
    string storedEtag,
    string currentEtag,
    Exception storageException)
    : base(message, storageException)
  {
    this.StoredEtag = storedEtag;
    this.CurrentEtag = currentEtag;
  }

  public InconsistentStateException(string storedEtag, string currentEtag, Exception storageException)
    : this(storageException.Message, storedEtag, currentEtag, storageException)
  { }

  /// <summary>The Etag value currently held in persistent storage.</summary>
  public string StoredEtag { get; private set; }
  
  /// <summary>The Etag value currently held in memory, and attempting to be updated.</summary>
  public string CurrentEtag { get; private set; }
}
```

任何其他来自存储操作的故障条件 _必须_ 引发返回的`Task`的中止，连同一个指示底层存储问题的异常。
在很多情况下，这个异常会被抛回给调用者，调用者通过调用Grain上的方法触发了存储操作。
考量调用者是否能够反序列化这个异常是很重要的。例如，客户端可能没有加载包含该异常类型的某个持久化库。
出于这个原因，建议将异常转换为可以传播回给调用者的异常。

### 数据映射

存储提供商应该决定如何最好地存储Grain状态——blob（各种格式/序列化形式）或字段-列格式会是首选。

### 注册一个存储提供商

当一个Grain被创建时，Orleans运行时将从Service provider（`IServiceProvider`）解析一个存储提供商。运行时将解析出一个`IGrainStorage`的实例。
如果存储提供商被命名了，例如使用`[PersistentState(stateName, storageName)]`特性，那么`IGrainStorage`的一个具名实例将被解析。

要注册`IGrainStorage`的具名实例，使用`IServiceCollection.AddSingletonNamedService`扩展方法，像[AzureTableGrainStorage的示例](https://github.com/dotnet/orleans/blob/af974d37864f85bfde5dc02f2f60bba997f2162d/src/Azure/Orleans.Persistence.AzureStorage/Hosting/AzureTableSiloBuilderExtensions.cs#L78)一样。
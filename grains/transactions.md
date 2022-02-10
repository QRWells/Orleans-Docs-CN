---
title: Orleans 2.0中的事务
description: Orleans支持针对持久性Grain状态的分布式ACID事务。
---

## 设置

Orleans的事务是可选的。
Silo必须被配置为使用事务。
如果不进行配置，任何对Grain上的事务方法的调用都会收到`OrleansTransactionsDisabledException`。
要在Silo上启用事务，请在Silo host builder上调用`UseTransactions()`。

```csharp
var builder = new SiloHostBuilder().UseTransactions();
```
### 事务状态存储

为了使用事务，用户需要配置一个数据存储。
为了支持带有事务的各种数据存储，我们引入了存储抽象`ITransactionalStateStorage`。
这个抽象是专门针对事务需求的，与一般的Grain物存储（`IGrainStorage`）不同。
为了使用事务特化的存储，用户可以使用`ITransactionalStateStorage`的实现来配置他们的Silo，例如Azure（`AddAzureTableTransactionalStateStorage`）。

示例:

```csharp
var builder = new SiloHostBuilder()
    .AddAzureTableTransactionalStateStorage("TransactionStore", options =>
    {
        options.ConnectionString = ”YOUR_STORAGE_CONNECTION_STRING”;
    })
    .UseTransactions();
```

出于开发目的，如果你需要的数据存储没有事务特化的存储，可以使用`IGrainStorage`的实现来代替。
对于任何没有配置存储的事务状态，事务将尝试使用桥接，失效转移到Grain存储。
通过桥接到Grain存储来访问事务状态的效率较低，而且我们不打算长期支持这种模式，因此建议只用于开发目的。

## 编程模型

### Grain接口

为了让Grain支持事务，Grain接口上的事务方法必须使用`[Transaction]`特性来标记为事务的一部分。
该特性需要通过下面的事务选项来指示Grain调用在事务环境中的行为：

- `TransactionOption.Create` - 事务性调用。其总是创建一个新的事务上下文（即启动一个新事务），即使是在现有的事务上下文中进行调用。
- `TransactionOption.Join` - 事务性调用。但只能在现存事务的上下文中调用。
- `TransactionOption.CreateOrJoin` - 事务性调用。如果在一个事务上下文中调用，它将使用该上下文，否则它将创建一个新的上下文。
- `TransactionOption.Suppress` - 非事务性调用。可以从一个事务中调用。如果在一个事务上下文中调用，上下文将不会被传递给调用。
- `TransactionOption.Supported` - 非事务性调用。其支持事务。如果在一个事务上下文中调用，上下文将被传递给调用。
- `TransactionOption.NotAllowed` - 非事务性调用。不能从一个事务中调用。如果在一个事务的上下文中调用，它将抛出一个`NotSupportedException`。

调用可以被标记为“创建”，这意味着该调用将总是启动它自己的事务。
例如，下面的ATM Grain中的转账操作将总是启动一个新的事务，包含到两个用到的账户。

```csharp
public interface IATMGrain : IGrainWithIntegerKey
{
    [Transaction(TransactionOption.Create)]
    Task Transfer(Guid fromAccount, Guid toAccount, uint amountToTransfer);
}
```

账户Grain上的事务操作`Withdraw`和`Deposit`被标记为`TransactionOption.Join`，表明它们只能在现有事务上下文中被调用，即在`IATMGrain.Transfer(...)`中被调用。
`GetBalance`的调用被标记为`TransactionOption.CreateOrJoin`，所以它可以从现有的事务中调用，比如通过`IATMGrain.Transfer(...)`，或者单独调用。

```csharp

public interface IAccountGrain : IGrainWithGuidKey
{
    [Transaction(TransactionOption.Join)]
    Task Withdraw(uint amount);

    [Transaction(TransactionOption.Join)]
    Task Deposit(uint amount);

    [Transaction(TransactionOption.CreateOrJoin)]
    Task<uint> GetBalance();
}

```

#### 重要考虑

注意，`OnActivateAsync`不能被标记为事务性调用，因为所有这样的调用都需要在调用前进行适当的设置。它只存在于Grain应用API。这意味着，在这些方法中试图读取事务性状态，会在运行时引发异常。

### Grain实现

Grain实现需要使用`ITransactionalState`分面（见分面系统）来通过ACID事务管理Grain状态。

```csharp
    public interface ITransactionalState<TState>
        where TState : class, new()
    {
        Task<TResult> PerformRead<TResult>(Func<TState, TResult> readFunction);
        Task<TResult> PerformUpdate<TResult>(Func<TState, TResult> updateFunction);
    }
```

所有对持久化状态的读写访问必须通过传递给事务状态分面的同步函数来执行。
这允许事务系统以事务方式执行或取消这些操作。
要在Grain中使用事务状态，只需要定义一个可序列化的状态类，并在Grain的构造函数中用`TransactionalState`特性来声明事务状态。后者声明了状态名称和（可选的）使用的事务状态存储（见[设置](#设置)）。

```csharp
[AttributeUsage(AttributeTargets.Parameter)]
public class TransactionalStateAttribute : Attribute
{
    public TransactionalStateAttribute(string stateName, string storageName = null)
    {
      …
    }
}
```

示例:

```csharp
public class AccountGrain : Grain, IAccountGrain
{
    private readonly ITransactionalState<Balance> balance;

    public AccountGrain(
        [TransactionalState("balance", "TransactionStore")]
        ITransactionalState<Balance> balance)
    {
        this.balance = balance ?? throw new ArgumentNullException(nameof(balance));
    }

    Task IAccountGrain.Deposit(uint amount)
    {
        return this.balance.PerformUpdate(x => x.Value += amount);
    }

    Task IAccountGrain.Withdrawal(uint amount)
    {
        return this.balance.PerformUpdate(x => x.Value -= amount);
    }

    Task<uint> IAccountGrain.GetBalance()
    {
        return this.balance.PerformRead(x => x.Value);
    }
}
```

在上面的例子中，`[TransactionalState]`特性被用来声明构造函数参数`balance`应该与一个名为`"balance"`的事务状态相关。
有了这个声明，Orleans就会注入一个`ITransactionalState`实例，其状态是从名为`"TransactionStore"`的交易状态存储中加载的（见设置）。
该状态可以通过`PerformUpdate`修改，或者通过`PerformRead`读取。
事务基础设施将确保任何此类更改都会作为事务的一部分执行，即使是分布在Orleans集群上的多个Grain，也将全部提交，或者在创建事务的Grain调用完成后全部撤销（上述例子中的`IATMGrain.Transfer`）。

### 调用事务

Grain接口上的事务性方法可以像其他Grain调用一样被调用。

```csharp
    IATMGrain atm = client.GetGrain<IATMGrain>(0);
    Guid from = Guid.NewGuid();
    Guid to = Guid.NewGuid();
    await atm.Transfer(from, to, 100);
    uint fromBalance = await client.GetGrain<IAccountGrain>(from).GetBalance();
    uint toBalance = await client.GetGrain<IAccountGrain>(to).GetBalance();
```

在上述调用中，使用ATM Grain将100单位的货币从一个账户转到另一个账户。
转账完成后，查询两个账户，以获得其当前余额。
货币转账以及两个账户的查询都是作为ACID事务执行的。

正如在上面的例子中所看到的，事务可以在一个`Task`中返回值，就像其他Grain调用一样，但是当调用失败时，它们不会抛出应用程序异常，而是抛出`OrleansTransactionException`或 `TimeoutException`。
如果应用程序在事务过程中抛出一个异常，并且该异常导致事务失败（而不是因为其他系统故障而失败），应用程序的异常将是`OrleansTransactionException`的内部异常。
如果抛出一个`OrleansTransactionAbortedException`类型的事务异常，说明该事务失败且可以重试。
抛出的任何其他异常都表明事务在未知状态下终止了。
由于事务是分布式操作，处于未知状态的事务可能已经成功、失败或仍在进行中。
出于这个原因，在验证状态或重试操作之前，最好容许调用超时（`SiloMessagingOptions.ResponseTimeout`），以避免级联中止。
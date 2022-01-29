---
title: Timers and Reminders
---

# 定时器和提醒器

Orleans运行时提供了两种机制，称为定时器和提醒器，使开发者能够为Grains指定定期动作。

# 定时器

## 简介

**定时器**用于创建无需跨越多个Grain的激活（Grain的实例化）的周期性Grain行为。它与标准.NET中的`System.Threading.Timer`类本质上是相同的。
此外，定时器在它所操作的Grain激活中保证单线程执行。

每个激活可以有零个或多个与之关联的定时器。运行时在它所关联的激活的运行时上下文中执行每个定时器例程。

## 用法

要启动一个定时器，使用`Grain.RegisterTimer`方法,它返回一个`IDisposable`引用:

``` csharp
public IDisposable RegisterTimer(
       Func<object, Task> asyncCallback, // 定时器需要定期执行的函数
       object state,                     // 传递给asyncCallback的object
       TimeSpan dueTime,                 // 在第一次触发之前的等待时间
       TimeSpan period)                  // 定时器的触发周期
```

通过释放（Dispose）定时器来取消它。

如果Grain被停用或发生故障且其Silo崩溃，计时器将停止触发。

## 重要注意事项

* 启用激活回收后，定时器回调的执行不会将激活的状态从闲置改为正在使用。这意味着定时器不能用于推迟原本闲置激活的停用。
* 传递给`Grain.RegisterTimer`的周期是指，从`asyncCallback`返回的`Task`被解决的时刻到下一次`asyncCallback`调用应该发生的时刻所经过的时间。这不仅使对`asyncCallback`的连续调用不可能重叠，还使完成`asyncCallback`的用时可以影响`asyncCallback`被调用的频率。这是与`System.Threading.Timer`的语义的重要不同。
* 每个`asyncCallback`的调用都是在单独的一轮中传递给一个激活，并且永远不会与同一激活的其他轮同时运行。但是请注意，`asyncCallback`的调用不是作为消息传递的，因此不受消息交织语义的影响。这意味着`asyncCallback`的调用应该被认为是在一个可重入Grain上运行，与该Grain的其他消息相比，其行为也是如此。

# 提醒器

## 介绍

提醒器与定时器很相似，但有几个重要的不同点：

* 提醒器是持久化的，在几乎所有情况下（包括部分或全部集群重启）都会持续触发，除非显式取消。
* 提醒器的“定义”被写入存储，但是，对于每个特定的触发，以及其触发时间，缺并非如此。这会带来一个副作用：如果集群在一个特定的提醒器触发时完全关闭，这次触发将会错过，只有提醒器的下一次触发才会发生。
* 提醒器关联到一个Grain，而非某个特定的激活。
* 如果一个Grain在提醒器触发时没有关联的激活，那这个Grain将被创建。如果一个激活变成空闲状态并已停用，关联至同一Grain的提醒器将会在下一次触发时重新激活它。
* 提醒器是通过消息传递的，与所有其他Grain方法一样受制于交织语义。
* 提醒器不应被用于高频率的定时任务--其周期应以分钟、小时或天为单位。

## 配置

提醒器是持久化的，它依靠存储来运行。
在提醒器子系统运行前，你必须指定使用哪个底层存储。
这是通过使用`UseXReminderService`扩展方法来配置其中一个提醒器提供者来实现的，其中`X`是提供者的名称，如`UseAzureTableReminderService`。

Azure Table的配置:

``` csharp
// TODO replace with your connection string
const string connectionString = "YOUR_CONNECTION_STRING_HERE";
var silo = new SiloHostBuilder()
    [...]
    .UseAzureTableReminderService(options => options.ConnectionString = connectionString)
    [...]
```

SQL:

``` csharp
// TODO replace with your connection string
const string connectionString = "YOUR_CONNECTION_STRING_HERE";
const string invariant = "YOUR_INVARIANT";
var silo = new SiloHostBuilder()
    [...]
    .UseAdoNetReminderService(options => 
    {
        options.ConnectionString = connectionString;
        options.Invariant = invariant;
    })
    [...]
```

如果你只需要一个提醒器的占位符实现来工作，不需要设置Azure账户或SQL数据库。这是一个仅用于开发的提醒器系统的实现：

``` csharp
var silo = new SiloHostBuilder()
    [...]
    .UseInMemoryReminderService()
    [...]
```

## 用法

使用提醒器的Grain必须实现`IRemindable.ReceiveReminder`方法。

``` csharp
Task IRemindable.ReceiveReminder(string reminderName, TickStatus status)
{
    Console.WriteLine("Thanks for reminding me-- I almost forgot!");
    return Task.CompletedTask;
}
```

要启动一个提醒器，使用`Grain.RegisterOrUpdateReminder`方法，它返回一个`IGrainReminder`对象：

``` csharp
protected Task<IGrainReminder> RegisterOrUpdateReminder(string reminderName, TimeSpan dueTime, TimeSpan period)
```

* `reminderName`是一个字符串，它必须能在上下文Grain的范围内唯一地识别提醒器。
* `dueTime`指定提醒器在第一次触发之前的等待时间。
* `period`指定提醒器的触发周期。

因为提醒器在任何一个激活的生命周期中都会存活，所以它们必须显式取消（而不是被释放（Dispose））。可以通过调用`Grain.UnregisterReminder`方法来取消一个提醒器：

``` csharp
protected Task UnregisterReminder(IGrainReminder reminder)
```

提醒器是由`Grain.RegisterOrUpdateReminder`返回的句柄对象。

`IGrainReminder`的实例并不保证在激活的有效期之后仍然有效。如果你想以一种持久化的方式识别提醒器，请使用提醒器名称的字符串。

如果你只有提醒器的名称且需要相应的`IGrainReminder`实例，请调用`Grain.GetReminder`方法：

``` csharp
protected Task<IGrainReminder> GetReminder(string reminderName)
```

## 我该使用哪一个？

我们建议你在以下情况下使用定时器：

* 在激活被停用或发生故障时，定时器停止工作也并不重要（或可以接受）。
* 定时器的周期很小（比如秒或分钟级别）。
* 定时器的回调可以从`Grain.OnActivateAsync`里或当Grain方法被调用时启动。

我们建议你在以下情况下使用提醒器：

* 定期行为需要在激活以及任何故障中保持存活。
* 执行不频繁的任务（比如分钟、小时或天级别）

## 结合使用定时器和提醒器

你可以考虑使用提醒器和定时器的组合来达成目的。
例如，如果你需要一个小分辨率的定时器，它需要在不同的激活中存活，你可以使用一个每五分钟运行一次的提醒器，其目的是唤醒一个Grain来重新启动一个可能因停用而失效的本地定时器。
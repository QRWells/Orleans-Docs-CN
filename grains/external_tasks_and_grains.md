---
title: 外部任务和Grains
---

# External Tasks and Grains

在Orleans的设计里，任何从Grain代码中产生的子任务（例如，通过`await`、`ContinueWith`或`Task.Factory.StartNew`产生的）将在与父任务相同的 per-activation [TaskScheduler](https://docs.microsoft.com/dotnet/api/system.threading.tasks.taskscheduler)上进行调度，从而继承了与其他Grain代码相同的*单线程执行模型*。
这是Grain基于回合的并发的单线程执行背后的基点。

在某些情况下，Grain代码可能需要“突破”Orleans的任务调度模型去"做一些特别的事情"，比如将一个`Task`显式指向不同的任务调度器或.NET[线程池](https://docs.microsoft.com/dotnet/api/system.threading.threadpool)。
例如，Grain代码必须执行一个同步的远程阻塞调用（如远程IO）。
在Grain上下文中执行该阻塞调用将阻塞Grain，因此不应该这样做。
相反，Grain代码可以在线程池线程上执行这段阻塞代码，并join（`await`）该执行的完成，并在Grain上下文中继续。
我们期望逃逸出Orleans调度器将是一个非常高级的且很少需要的，超出“正常”使用模式的使用场景。

## 基于Task的API

1. `await`、`Task.Factory.StartNew`（见下文）、`Task.ContinueWith`、`Task.WhenAny`、`Task.WhenAll`、`Task.Delay`都遵循当前任务调度器。
这意味着在不传递不同的任务调度器的情况下，以默认方式使用它们将使它们在Grain上下文中执行。

2. `Task.Run`和`Task.Factory.FromAsync`的`endMethod`委托都*不*遵循当前任务调度器。它们都使用`TaskScheduler.Default`调度器，即.NET线程池任务调度器。因此，`Task.Run`内的代码和`Task.Factory.FromAsync`内的`endMethod`将*始终*运行在Orleans Grain的单线程执行模型之外的.NET线程池上，[详见这里](http://blogs.msdn.com/b/pfxteam/archive/2011/10/24/10229468.aspx)。但是，在`await Task.Run`或`await Task.Factory.FromAsync`之后的所有代码将在任务创建时的调度器下运行，也就是Grain的调度器。

3. `ConfigureAwait(false)`是一个用来显式逃逸出当前任务的调度器的API。它将使等待任务后的代码在`TaskScheduler.Default`调度器上执行，也就是.NET线程池，因此会破坏Grain的单线程执行。一般来说，你应该**不直接在grain代码中使用`ConfigureAwait(false)`**。

4. 签名为`async void`的方法不应该与Grain一起使用。它们是为GUI事件处理程序准备的。`async void`方法如果允许一个异常逃逸，就会立即使当前进程崩溃，从而无法处理这个异常。对于`List<T>.ForEach(async element => ...)`和任何其他接受`Action<T>`的方法也是如此，因为异步委托将被强制变成一个`async void`委托。

### Task.Factory.StartNew和async委托

通常建议，在任何C#程序中调度任务应使用`Task.Run`而不是`Task.Factory.StartNew`。
事实上，在快速谷歌一下`Task.Factory.StartNew()`的使用，会发现[它很危险](https://blog.stephencleary.com/2013/08/startnew-is-dangerous.html)并且[应该总是倾向于`Task.Run`](https://devblogs.microsoft.com/pfxteam/task-run-vs-task-factory-startnew/)。
但是，如果我们想保持谷物的*单线程执行模式*，我们就需要用到它，那么我们如何正确地使用它呢？
使用`Task.Factory.StartNew()`的风险在于，它并不支持原生的异步委托。
这意味着这可能是一个bug： `var notIntendedTask = Task.Factory.StartNew(SomeDelegateAsync)`。
`notIntendedTask` _不是_ 一个在`SomeDelegateAsync`完成时完成的任务。
相反，我们应该 _总是_ 拆包返回的任务： `var task = Task.Factory.StartNew(SomeDelegateAsync).Unwrap()`。

#### 例子

下面的示例代码演示了`TaskScheduler.Current`、`Task.Run`和一个自定义调度器的用法，以摆脱Orleans Grain上下文，以及如何返回到它。

``` csharp
   public async Task MyGrainMethod()
   {
        // Grab the grain's task scheduler
        var orleansTS = TaskScheduler.Current;
        await TaskDelay(10000);

        // Current task scheduler did not change, the code after await is still running
        // in the same task scheduler.
        Assert.AreEqual(orleansTS, TaskScheduler.Current);

        Task t1 = Task.Run( () =>
        {
             // This code runs on the thread pool scheduler, not on Orleans task scheduler
             Assert.AreNotEqual(orleansTS, TaskScheduler.Current);
             Assert.AreEqual(TaskScheduler.Default, TaskScheduler.Current);
        });

        await t1;

        // We are back to the Orleans task scheduler. 
        // Since await was executed in Orleans task scheduler context, we are now back
        // to that context.
        Assert.AreEqual(orleansTS, TaskScheduler.Current);

        // Example of using Task.Factory.StartNew with a custom scheduler to escape from
        // the Orleans scheduler
        Task t2 = Task.Factory.StartNew(() =>
        {
             // This code runs on the MyCustomSchedulerThatIWroteMyself scheduler, not on
            // the Orleans task scheduler
             Assert.AreNotEqual(orleansTS, TaskScheduler.Current);
             Assert.AreEqual(MyCustomSchedulerThatIWroteMyself, TaskScheduler.Current);
        },
        CancellationToken.None,
        TaskCreationOptions.None,
        scheduler: MyCustomSchedulerThatIWroteMyself);

        await t2;

        // We are back to Orleans task scheduler.
        Assert.AreEqual(orleansTS, TaskScheduler.Current);
   }
```

#### 例子 - 从运行在线程池上的代码中调用Grain

另一种情况是，一段Grain代码需要“脱离”Grain的任务调度模型，在线程池（或其他一些非Grain上下文）上运行，但仍然需要调用另一个Grain。
Grain调用可以从非Grain上下文中进行，无需额外的操作。

下面的代码演示了如何从运行在Grain内部但不在Grain上下文中的一段代码中进行Grain调用。

``` csharp
   public async Task MyGrainMethod()
   {
        // Grab the Orleans task scheduler
        var orleansTS = TaskScheduler.Current;
        var fooGrain = this.GrainFactory.GetGrain<IFooGrain>(0);
        Task<int> t1 = Task.Run(async () =>
        {
            // This code runs on the thread pool scheduler,
            // not on Orleans task scheduler
            Assert.AreNotEqual(orleansTS, TaskScheduler.Current);
            int res = await fooGrain.MakeGrainCall();

            // This code continues on the thread pool scheduler,
            // not on the Orleans task scheduler
            Assert.AreNotEqual(orleansTS, TaskScheduler.Current);
            return res;
        });

        int result = await t1;

        // We are back to the Orleans task scheduler.
        // Since await was executed in the Orleans task scheduler context,
        // we are now back to that context.
        Assert.AreEqual(orleansTS, TaskScheduler.Current);
   }
```

## 与其他库合作

你的代码使用的一些外部库可能在其内部使用了`ConfigureAwait(false)`。
事实上，在.NET中[实现通用库时](https://msdn.microsoft.com/magazine/jj991977.aspx)使用`ConfigureAwait(false)`是一种良好且正确的做法。
但这在Orleans中不是问题。
只要Grain中调用库方法的代码是用常规的`await`来等待库的调用，Grain代码就是正确的。
结果将与预期完全一致——库的代码将在默认调度器（由`TaskScheduler.Default`返回的值，这并不能保证计算续体一定会在`ThreadPool`线程上运行，因为计算续体经常被内联在前一个线程中）上运行，而Grain代码将在Grain的调度器上运行。

另一个常见问题是，是否需要用`Task.Run`来执行库的调用？也就是说，是否需要将库的代码显式转移（offload）到`ThreadPool`（对于Grain代码，`Task.Run(() => myLibrary.FooAsync())`）？
答案是否定的。
没有必要将任何代码转移到`ThreadPool`，除非是库的代码正在进行阻塞的同步调用。
通常情况下，任何写得好的、正确的.NET异步库（返回`Task`并以`Async`后缀命名的方法）都不会进行阻塞调用。
因此，除非你怀疑异步库有问题，或者你故意使用同步阻塞库，否则没有必要将任何东西转移到`ThreadPool`。

## 死锁

由于Grain是*单线程*执行的，所以有可能通过同步阻塞来使一个Grain陷入死锁，其需要多个线程来解除阻塞。
这意味着，如果在调用方法或属性时，所提供的任务还没有完成，那么调用以下任何方法和属性的代码都会使Grain陷入死锁：

* `Task.Wait()`
* `Task.Result`
* `Task.WaitAny(...)`
* `Task.WaitAll(...)`
* `task.GetAwaiter().GetResult()`

在任何高并发服务中都应该避免使用这些方法，它们会导致性能低下且不稳定，因为它们阻塞了可能正在进行有效工作的线程，并要求.NET`ThreadPool`注入额外的线程以便完成这些工作，从而使.NET`ThreadPool`陷入饥饿状态。
在执行Grain代码时，如上所述，这些方法会导致Grain出现死锁，因此在Grain代码中也应避免使用这些方法。

如果有一些无法避免的*sync-over-async*的工作，最好将这些工作放到一个单独的调度器中。
最简单的方法是以`await Task.Run(() => task.Wait())`为例。
请注意，我们强烈建议避免*sync-over-async*工作，因为如上所述，它将导致你的应用的可扩展性和性能受到影响。

## 总结：在Orleans中使用Task

| 你想做什么                                                                                                              | 如何实现                                      |
| ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| 在.NET线程池的线程上运行后台工作。不允许有Grain代码或Grain调用。                                             `Task.Run` |
| 在Grain代码中运行异步worker任务，并有Orleans的回合制并发保证（[见上文](#taskfactorystartnew和async委托)）。             | `Task.Factory.StartNew(WorkerAsync).Unwrap()` |
| 在Grain代码中运行同步的worker任务，并有Orleans的回合制并发保证。                                                        | `Task.Factory.StartNew(WorkerSync)`           |
| 执行任务的超时                                                                                                          | `Task.Delay` + `Task.WhenAny`                 |
| 调用异步库方法                                                                                                          | `await`此异步方法                             |
| 使用 `async`/`await`                                                                                                    | 普通的 .NET Task-Async编程模型。受支持且推荐  |
| `ConfigureAwait(false)`                                                                                                 | 不要在Grain代码内使用。只允许在库内使用。     |

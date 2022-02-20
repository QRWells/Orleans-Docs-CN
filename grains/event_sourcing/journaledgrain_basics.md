---
title: 日志式Grain的API
description: 本节将介绍日志式Grain的API及相关操作
---

# 日志式Grain基础

日志式Grains派生自`JournaledGrain<StateType,EventType>`，泛型参数如下：

* `StateType`代表Grain的状态。其必须是一个带`public`默认构造函数的类。
* `EventType`是所有可以为这个Grain引发的事件的共同超类型，可以是类或者接口。

所有的状态和事件对象都应该是可序列化的（因为日志一致性提供者可能需要持久化它们，以及（或）在通知消息中发送它们）。

对于事件是POCO（普通C#对象（plain old C# objects））的Grain，`JournaledGrain<StateType>`可以作为`JournaledGrain<StateType,Object>`的简写。

## 读取Grain状态

要读取当前的Grain状态，并确定其版本号，日志式Grain有以下属性

```csharp
GrainState State { get; }
int Version { get; }
```

版本号总是等于已确认事件的总数，而状态是将所有已确认事件应用于初始状态的结果。初始状态，其版本号为0（因为没有事件被应用到它），由GrainState类的默认构造函数决定。

_重要事项：_ 应用程序不应直接修改由`State`返回的对象。它只用于读取。相反，当应用程序想要修改状态时，它必须通过引发事件来间接地修改。

## 引发事件

通过调用`RaiseEvent`函数来引发事件。例如，一个代表聊天室的Grain可以引发一个`PostedEvent`来表示一个用户提交了一个帖子：

```csharp
RaiseEvent(new PostedEvent() { Guid = guid, User = user, Text = text, Timestamp = DateTime.UtcNow });
```

请注意，`RaiseEvent`启动了对存储访问的写入，但并不等待写入的完成。对于许多应用来说，重要的是要等到我们确认事件已经被持久化。在这种情况下，我们需要通过等待`ConfirmEvents`来进一步确认：

```csharp
RaiseEvent(new DepositTransaction() { DepositAmount = amount, Description = description });
await ConfirmEvents();
```

注意，即使你不显式地调用`ConfirmEvents`，事件最终也会被确认--它会在后台自动进行。

## 实现状态转移

每当事件发生时，运行时都会 _自动_ 更新Grain状态。应用没必要在引发事件后显式地更新状态。然而，应用仍然需要提供代码来指定如何更新状态以响应一个事件。主要有两种方式。

**(a)** Grain状态类可以在`StateType`上实现一个或多个`Apply`方法。通常情况下，人们会创建多个重载，并为事件的运行类型选择最接近的匹配：
```csharp
class GrainState {
   
   Apply(E1 @event)  
   {
     // code that updates the state
   }
   Apply(E2 @event)  
   {
     // code that updates the state
   }
}
```
**(b)** Grain可以重写`TransitionState`函数：
```csharp
protected override void TransitionState(State state, EventType @event)
{
   // code that updates the state
}
```
转移方法被假定为除了修改状态对象外没有其他副作用，并且应该是确定性的（否则，效果是不可预测的）。 如果转移代码抛出一个异常，该异常将被捕获并包含在Orleans日志的警告中，并由日志一致性提供者发布。 

运行时究竟何时调用转移方法，取决于所选择的日志一致性提供者及其配置。对于应用程序来说，最好不要依赖特定的时间，除非是由日志一致性提供者特别保证的。

一些日志一致性提供者，如`LogStorage`，每次加载Grain时都会重播事件序列。因此，只要事件对象仍然可以从存储中正确反序列化，就可以从彻底修改GrainState类和转移方法。但对于其他提供者，如`StateStorage`，只有`GrainState`对象被持久化，所以开发者必须确保从存储中读取时能正确反序列化。

## 引发多个事件

在调用`ConfirmEvents`之前，可以多次调用`RaiseEvent`：

```csharp
RaiseEvent(e1);
RaiseEvent(e2);
await ConfirmEvents();
```
然而，这很可能会导致两次连续的存储访问，而且会产生风险，即Grain在只写入第一个事件后就会故障。因此，通常来说更好的做法是一次引发多个事件，使用

```csharp
RaiseEvents(IEnumerable<EventType> events)
```

这保证了给定的事件序列被原子化地写入存储。请注意，由于版本号总是与事件序列的长度相匹配，引发多个事件会使版本号增长多次。

## 获取事件序列

来自基类`JournaledGrain`的以下方法允许应用程序获取所有确认事件序列的指定片段：

```csharp
Task<IReadOnlyList<EventType>> RetrieveConfirmedEvents(int fromVersion, int toVersion)
```

然而，并不是所有的日志一致性提供者都支持。如果不支持，或者指定的序列段不再可用，就会抛出一个`NotSupportedException`。

要获取到最新确认的版本之前的所有事件，可以调用

```csharp
await RetrieveConfirmedEvents(0, Version);
```
 
只有确认过的事件可以被获取到：如果`toVersion`大于属性`Version`的当前值，就会抛出一个异常。

由于确认的事件永远不会改变，所以即使在存在多个实例或延迟确认也无需担心竞争。但是，在这种情况下，当`await`恢复时，属性`Version`的值有可能大于调用`RetrieveConfirmedEvents`时的值，所以最好将其值保存在一个变量中。也请看[并发性保证](immediate_vs_delayed_confirmation.md#Concurrency-Guarantees)一节。
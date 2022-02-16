---
title: 日志一致性提供者
description: 本节介绍了Orleans内置的三种日志一致性提供者
---

`Microsoft.Orleans.EventSourcing`包含有几个日志一致性提供者，涵盖了适合入门的基本场景，并具有一些扩展性。

### Orleans.EventSourcing.*StateStorage*.LogConsistencyProvider

这个提供者存储*Grain状态快照*，它使用一个可以独立配置的标准存储提供者。

保存在存储中的数据是一个对象，它包含了Grain状态（由`JournaledGrain`的第一个类型参数指定）和一些元数据（版本号，以及一个特殊的标签，用于避免存储访问失败时事件重复）。

由于每次我们访问存储时都会读/写整个Grain状态，所以这个提供者不适合Grain状态非常大的对象。

这个提供者不支持`RetrieveConfirmedEvents`：它不能从存储中检索事件，因为这些事件没有被持久化。

### Orleans.EventSourcing.*LogStorage*.LogConsistencyProvider

这个提供者将*完整的事件序列存储为一个对象*，它使用一个可以独立配置的标准存储提供者。

保存在存储中的数据是一个包含`List<EventType>`对象的对象，以及一些元数据（一个特殊的标签，用于避免存储访问失败时事件重复）。

这个提供者支持`RetrieveConfirmedEvents`。所有的事件都是可用的，并保存在内存中。

由于每次我们访问存储时都要读/写整个事件序列，这个提供者 _不适合在生产环境中使用_，除非可以保证事件序列保持在相当短的水平。这个提供者的主要目的是为了说明事件源的语义，以及用于样例/测试环境。

### Orleans.EventSourcing.*CustomStorage*.LogConsistencyProvider

这个提供者允许开发者接入自己的存储接口，这个接口会在适当的时候由一致性协议调用。这个提供者并不对所存储的是状态快照还是事件做出具体的假设--程序员对这一选项进行控制（可以存储其中之一或两者）。

要使用这个提供者，Grain必须像之前一样派生自`JournaledGrain<StateType,EventType>`，但另外还必须实现以下接口：

```csharp
public interface ICustomStorageInterface<StateType, EventType>
{
   Task<KeyValuePair<int,StateType>> ReadStateFromStorage();

   Task<bool> ApplyUpdatesToStorage(IReadOnlyList<EventType> updates, int expectedversion);
}
```
The consistency provider expects these to behave a certain way. Programmers should be aware that:
一致性提供者希望这些东西能以某种方式表现出来。程序员应该注意：

* 第一个方法，`ReadStateFromStorage`，应该同时返回版本号和读取的状态。如果还没有存储任何东西，它应该返回0，以及一个与`StateType`的默认构造函数相匹配的状态。

* `ApplyUpdatesToStorage`必须在预期版本与实际版本不一致时返回`false`（这类似于e-tag的检查）。

* 如果 `ApplyUpdatesToStorage`因异常而故障，一致性提供者会重试。这意味着如果抛出这样的异常，一些事件可能会重复，但事件实际上被持久化了。开发者需要确保这是安全的：例如，要么通过不抛出异常来避免这种情况，要么确保重复的事件对应用逻辑无害，或者添加一些额外的机制来过滤重复事件。 

这个提供者不支持`RetrieveConfirmedEvents`。当然，由于开发者无论如何都要控制存储接口，他们不需要首选调用这个，而是可以实现自己的事件检索。

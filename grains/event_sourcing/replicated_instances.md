---
title: 复制的Grain
description: 本节介绍了Orleans事件溯源中的复制Grain
---

<!-- Sometimes, there can be multiple instances of the same grain active, such as when operating a multi-cluster, and using the `[OneInstancePerCluster]` attribute. The JournaledGrain is designed to support replicated instances with minimal friction. It relies on *log-consistency providers* to run the necessary protocols to ensure all instances agree on the same sequence of events. In particular, it takes care of the following aspects:  -->

有时，同一Grain可能有多个实例处于活动状态，例如在操作多集群时，或者使用`[OneInstancePerCluster]`特性时。日志式Grain支持摩擦最小的复制实例。它依靠*日志一致性提供者*来运行必要的协议，以确保所有实例认同相同的事件序列。特别地，它照顾到以下几个方面：

* **版本一致**: Grain状态的所有版本（除了临时版本）都基于相同的全局事件序列。特别地，如果两个实例看到的版本号相同，那么它们看到的状态就就相同。

* **竞争事件**: 多个实例可以同时引发一个事件。一致性提供者解决了这种竞争，确保每个实例都认同同一个序列。

* **通知/反应**: 在一个Grain实例引发一个事件后，一致性提供者不仅更新存储，而且还通知所有其他Grain实例。

关于一致性模型的一般讨论，见我们的[技术报告](https://www.microsoft.com/en-us/research/publication/geo-distribution-actor-based-services/)和[GSP论文](https://www.microsoft.com/en-us/research/publication/global-sequence-protocol-a-robust-abstraction-for-replicated-shared-state-extended-version/) (全球序列协议)。

## 条件事件

如果竞争事件有冲突，即由于某种原因不应该同时提交，那么就会出现问题。例如，当从银行账户取钱时，两个实例可能会独立判断有足够的资金用于取款，并发出取款事件。但是这两个事件的组合可能会透支。为了避免这种情况，日志式Grain API太提供了一个`RaiseConditionalEvent`方法。

```csharp
bool success = await RaiseConditionalEvent(new WithdrawalEvent()  { ... });
```

条件事件会检查本地版本是否与存储中的版本相符。如果不相符，这意味着事件序列在此期间已经增长，这意味着这个事件在与其他事件的竞争中失败了。在这种情况下，条件事件*不*附加到日志中，并且`RaiseConditionalEvent`返回false。

这是类似于使用电子标签的条件存储更新，同样也提供了一个简单的机制来避免提交冲突的事件。

对同一个Grain同时使用有条件和无条件的事件是可能的，也是明智的，比如一个“存款事件”和一个“提款事件”。存款不需要条件：即使存款事件在竞争中失利，它也不必被取消，但仍然可以被追加到全局事件序列中。

等待由`RaiseConditionalEvent`返回的`Task`就足以确认该事件，也就是说，没有必要同时调用`ConfirmEvents`。

## 显式同步化

有时，需要确保一个Grain完全跟上最新的版本。这可以通过调用

```csharp
await RefreshNow();
```

它完成了(1)确认所有未确认的事件，(2)从存储中加载最新版本。

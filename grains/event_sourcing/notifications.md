---
title: 通知
description: 
---
# 通知

通常来说，对状态变化做出反应的能力会使开发更加方便。
所有的回调都受制于Orleans的基于回合的保证，参见并发性保证一节。

## 追踪已确认的状态

为了在已确认状态发生变化时获得通知，`JournaledGrain`（日志式Grain）子类可以重写这个方法：

```csharp
protected override void OnStateChanged()
{
   // read state and/or event log and take appropriate action
}
```

`OnStateChanged`在已确认状态发生更新时被调用，也就是说，版本号的增长。这可能发生在：

1. 从存储中加载了一个较新版本的状态。
2. 由该实例引发的事件已成功写入存储。
3. 收到了来自其他实例的通知消息。

请注意，由于从存储中初始加载完成时，所有Grains最初的版本都是0，这意味着，如果初始加载完成时版本号大于0，就会调用`OnStateChanged`。

## 追踪暂定状态

为了在暂定状态发生变化时获得通知，`JournaledGrain`（日志式Grain）子类可以重写这个方法：

```csharp
protected override void OnTentativeStateChanged()
{
   // read state and/or events and take appropriate action
}
```

`OnTentativeStateChanged`在暂定状态发生变化时被调用，即组合序列（已确认事件与未确认事件）（ConfirmedEvents + UnconfirmedEvents）发生变化时。特别是，对`OnTentativeStateChanged()`的回调总是发生在`RaiseEvent`期间。

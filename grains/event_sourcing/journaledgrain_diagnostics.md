---
title: 日志式Grain的诊断
description: 
---

# 日志式Grain的诊断


## 监控连接错误

根据设计，日志一致性提供者在连接错误（包括与存储的连接，以及集群之间的连接）下是可恢复的。但仅仅容错是不够的，因为应用通常需要监控所有此类问题，并在问题严重时提请操作员注意。

日志式Grain子类可以重写以下方法，以便在观察到连接错误以及这些错误被解决时接收通知：

```csharp
protected override void OnConnectionIssue(ConnectionIssue issue) 
{
    /// handle the observed error described by issue             
}
protected override void OnConnectionIssueResolved(ConnectionIssue issue) 
{
    /// handle the resolution of a previously reported issue             
}
```

`ConnectionIssue`是一个抽象类，含有几个描述问题的公共字段，包括自上次连接成功后观察到的次数。连接问题的实际类型是由子类定义的。连接问题被分为不同的类型，如 `PrimaryOperationFailed`或`NotificationFailed`，有时还有额外的键（如`RemoteCluster`）来进一步缩小类别。

如果同一类别的问题发生多次（例如，我们不断收到针对同一`RemoteCluster`的`NotificationFailed`），且每次都由`OnConnectionIssue`报告。一旦这类问题被解决（例如，我们最终成功地向这个`RemoteCluster`发送了通知），那么`OnConnectionIssueResolved`就会被调用一次，使用与`OnConnectionIssue`的上次报告相同的`issue`对象。对于独立的类别，连接问题和它们的解决被独立地报告。

## 简单的统计

We currently offer a simple support for basic statistics (in the future, we will probably replace this with a more standard telemetry mechanism).
Statistics collection can be enabled or disabled for a JournaledGrain by calling
我们目前提供了对基础统计的简单支持（在未来，我们可能会用一个更标准的遥测机制来取代它）。
对一个日志式Grain，统计数据的收集可以通过调用以下方法来启用或禁用：

```csharp
void EnableStatsCollection()
void DisableStatsCollection()
```

可以通过调用以下方法获取统计数据：

 ```csharp
LogConsistencyStatistics GetStats()
```

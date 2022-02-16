---
title: 配置
description: 
---

## 配置项目引用

### Grain接口

和之前一样，接口只依赖于`Microsoft.Orleans.Core`包，因为Grain接口与实现无关。

### Grain实现
 
日志式Grains需要派生自`JournaledGrain<S,E>`或`JournaledGrain<S>`，其定义在`Microsoft.Orleans.EventSourcing`包中。

### 日志一致性提供者

我们目前内置三个日志一致性提供者（用于状态存储、日志存储和自定义存储）。这三个都在`Microsoft.Orleans.EventSourcing`包中。因此，所有的日志式Grains已经可以访问它们。关于这些提供者的作用以及它们的区别，请参见[内置的日志一致性提供者](log_consistency_providers.md)。

## 集群配置

日志一致性提供者的配置与其他Orleans中的提供者一样。
例如，要使用所有三个提供者（当然，你可能只需要其中一个或两个），在配置文件的`<Globals>`标签中添加如下内容：

```xml
<LogConsistencyProviders>
  <Provider Type="Orleans.EventSourcing.StateStorage.LogConsistencyProvider" Name="StateStorage" />
  <Provider Type="Orleans.EventSourcing.LogStorage.LogConsistencyProvider" Name="LogStorage" />
  <Provider Type="Orleans.EventSourcing.CustomStorage.LogConsistencyProvider" Name="CustomStorage" />
</LogConsistencyProviders>
```

这也可以通过代码实现。2.0.0稳定版之后，`ClientConfiguration`和`ClusterConfiguration`已经不存在了！它现在被`ClientBuilder`和`SiloBuilder`所取代（注意，并没有集群的builder）。

```csharp
builder.AddLogStorageBasedLogConsistencyProvider("LogStorage")
```

## Grain类特性

每个有日志式Grain类必须有一个`LogConsistencyProvider`特性来指定日志一致性提供者。一些提供者还需要一个`StorageProvider`属性。
例如:

```csharp
[StorageProvider(ProviderName = "OrleansLocalStorage")]
[LogConsistencyProvider(ProviderName = "LogStorage")]
public class EventSourcedBankAccountGrain : JournaledGrain<BankAccountState>, IEventSourcedBankAccountGrain
{ ... }
```

可见这里"`OrleansLocalStorage`被用来存储Grain状态，而"`LogStorage`"是事件溯源事件的内存存储提供者。

### `LogConsistencyProvider`特性

要指定日志一致性提供者，请在Grain类中添加`[LogConsistencyProvider(ProviderName=...)]`属性，并给出集群配置中配置的提供者的名称。例如：

```csharp
[LogConsistencyProvider(ProviderName = "CustomStorage")]
public class ChatGrain : JournaledGrain<XDocument, IChatEvent>, IChatGrain, ICustomStorage { ... }
```

### `StorageProvider`特性

一些日志一致性提供者（包括`LogStorage`和`StateStorage`）使用标准StorageProvider来与存储进行通信。这个提供者使用另外的`StorageProvider`属性来指定，如下所示：

```csharp
[LogConsistencyProvider(ProviderName = "LogStorage")]
[StorageProvider(ProviderName = "AzureBlobStorage")]
public class ChatGrain : JournaledGrain<XDocument, IChatEvent>, IChatGrain { ... }
```

## 默认提供者

如果配置中指定了默认值，可以省略`LogConsistencyProvider`和/或`StorageProvider`属性。这是通过使用特殊名称`Default`来实现的。例如：

```xml
<LogConsistencyProviders>
  <Provider Type="Orleans.EventSourcing.LogStorage.LogConsistencyProvider" Name="Default" />
</LogConsistencyProviders>
<StorageProviders>
  <Provider Type="Orleans.Storage.MemoryStorage" Name="Default" />
</StorageProviders>
```



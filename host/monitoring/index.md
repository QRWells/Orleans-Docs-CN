---
title: 运行时监控
---

## Orleans日志

Orleans使用[Microsoft.Extensions.Logging](https://www.nuget.org/packages/Microsoft.Extensions.Logging/)来处理所有Silo和客户端的日志。

## 运行时监控

Orleans通过`ITelemetryConsumer`接口输出其运行时的统计数据和指标。
应用程序可以为其Silo和客户端注册一个或多个遥测消费者，以接收Orleans运行时发布的统计数据和指标。
这些消费者可以是流行的遥测分析解决方案的消费者，也可以是为其他目标和目的定制的消费者。
目前Orleans代码库中包括三个遥测消费者。

它们是作为独立的NuGet包发布的：

- `Microsoft.Orleans.OrleansTelemetryConsumers.AI`用于发布到[Application Insights](https://azure.microsoft.com/en-us/services/application-insights/).

- `Microsoft.Orleans.OrleansTelemetryConsumers.Counters`用于发布到Windows性能计数器。
Orleans运行时不断地更新它们中的一部分。
CounterControl.exe工具，包含在[`Microsoft.Orleans.CounterControl`](https://www.nuget.org/packages/Microsoft.Orleans.CounterControl/)NuGet包中，用于注册必要的性能计数器类别。
它必须以较高的权限运行。
性能计数器可以使用任何标准的监控工具进行监控。

- `Microsoft.Orleans.OrleansTelemetryConsumers.NewRelic`用于发布到[New Relic](https://newrelic.com/).

下面介绍如何配置你的Silo和客户端使用遥测消费者，这是Silo配置代码的示例：

```csharp
var siloHostBuilder = new SiloHostBuilder();
//configure the silo with AITelemetryConsumer
siloHostBuilder.AddApplicationInsightsTelemetryConsumer("INSTRUMENTATION_KEY");
```

这是客户端配置代码的示例：

```csharp
var clientBuilder = new ClientBuilder();
//configure the clientBuilder with AITelemetryConsumer
clientBuilder.AddApplicationInsightsTelemetryConsumer("INSTRUMENTATION_KEY");
```

要使用自定义的`TelemetryConfiguration`（可能有`TelemetryProcessors`、`TelemetrySinks`等），Silo配置代码类似这样：

```csharp
var siloHostBuilder = new SiloHostBuilder();
var telemetryConfiguration = TelemetryConfiguration.CreateDefault();
//configure the silo with AITelemetryConsumer
siloHostBuilder.AddApplicationInsightsTelemetryConsumer(telemetryConfiguration);
```

客户端配置代码类似这样：

```csharp
var clientBuilder = new ClientBuilder();
var telemetryConfiguration = TelemetryConfiguration.CreateDefault();
//configure the clientBuilder with AITelemetryConsumer
clientBuilder.AddApplicationInsightsTelemetryConsumer(telemetryConfiguration);
```
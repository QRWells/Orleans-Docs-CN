---
title: 多集群Silo配置
---

为了快速了解概况，我们在下面以XML展示了所有相关的配置参数（包括可选参数）：

```xml
<?xml version="1.0" encoding="utf-8"?>
<OrleansConfiguration xmlns="urn:orleans">
  <Globals>
    <MultiClusterNetwork
      ClusterId="clusterid"
      DefaultMultiCluster="uswest,europewest,useast"
      BackgroundGossipInterval="30s"
      UseGlobalSingleInstanceByDefault="false"
      GlobalSingleInstanceRetryInterval="30s"
      GlobalSingleInstanceNumberRetries="3"
      MaxMultiClusterGateways="10">
         <GossipChannel  Type="..."  ConnectionString="..."/>      
         <GossipChannel  Type="..."  ConnectionString="..."/>      
    </MultiClusterNetwork>
    <SystemStore ... ServiceId="some-guid" .../>
  </Globals>
</OrleansConfiguration>
```

```csharp
var silo = new SiloHostBuilder()
  [...]
  .Configure<ClusterInfo>(options =>
  {
    options.ClusterId = "us3";
    options.ServiceId = "myawesomeservice";
  })
  .Configure<MultiClusterOptions>(options => 
  {
    options.HasMultiClusterNetwork = true;
    options.DefaultMultiCluster = new[] { "us1", "eu1", "us2" };
    options.BackgroundGossipInterval = TimeSpan.FromSeconds(30);
    options.UseGlobalSingleInstanceByDefault = false;
    options.GlobalSingleInstanceRetryInterval = TimeSpan.FromSeconds(30);
    options.GlobalSingleInstanceNumberRetries = 3;
    options.MaxMultiClusterGateways = 10;
    options.GossipChannels.Add("AzureTable", "DefaultEndpointsProtocol=https;AccountName=usa;AccountKey=...");
    options.GossipChannels.Add("AzureTable", "DefaultEndpointsProtocol=https;AccountName=europe;AccountKey=...")
    [...]
  })
  [...]
```

像往常一样，所有的配置设置也可以通过`GlobalConfiguration`类的相应成员，以编程方式进行读写。

`Service Id`是一个任意的ID，用于识别该服务。它对所有集群和所有Silo都必须是相同的。

`MultiClusterNetwork`部分是可选的，如果不指明，则该Silo的所有多集群支持都将被禁用。

在[多集群通信](GossipChannels.md)一节中解释了**必要参数**：`ClusterId`和`GossipChannel`。

可选参数`MaxMultiClusterGateways`和`BackgroundGossipInterval`亦在[多集群通信](GossipChannels.md)一节中解释。

可选参数`DefaultMultiCluster`在[多集群配置](MultiClusterConfiguration.md)一节中解释。

可选参数`UseGlobalSingleInstanceByDefault`、`GlobalSingleInstanceRetryInterval`和`GlobalSingleInstanceNumberRetries`在[全局单例Grains](GlobalSingleInstance.md)一节中解释。

## Orleans客户端配置

Orleans客户端不需要额外的配置。同一个客户端可能无法连接到不同集群中的Silo（在这种情况下，Silo会拒绝连接）。
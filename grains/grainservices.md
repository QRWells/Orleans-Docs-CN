---
title: Grain服务
description:  本节介绍了Orleans中的Grain服务
---

# Grain服务

GrainService是一种特殊的Grain：它没有标识，每个Silo中从启动运行到关闭。

## 创建一个Grain服务

**第一步.** 创建接口。
Grain服务接口与其他Grain接口没有什么不同。

``` csharp
public interface IDataService : IGrainService {
    Task MyMethod();
}
```

**第二步.** 创建DataService这一Grain。
尽可能使Grain服务可重入，这可以提高性能。
注意，这里需要调用基类构造函数。
你也可以注入一个`IGrainFactory`，这样你就可以从你的Grain服务进行Grain调用。

注意：Grain服务不能写到Orleans流，因为它不在Grain任务调度器中工作。
如果你需要Grain服务写到流中，那么你必须把对象送到另一个Grain中，以便写到流中。

``` csharp
[Reentrant]

public class DataService : GrainService, IDataService {

    readonly IGrainFactory GrainFactory;

    public DataService(IServiceProvider services, IGrainIdentity id, Silo silo, ILoggerFactory loggerFactory, IGrainFactory grainFactory) : base(id, silo, loggerFactory) {
        GrainFactory = grainFactory;
    }

    public override Task Init(IServiceProvider serviceProvider) {
        return base.Init(serviceProvider);
    }

    public override async Task Start() {
        await base.Start();
    }

    public override Task Stop() {
        return base.Stop();
    }

    public Task MyMethod() {
    }
}
```

**第三步.** 为Grain服务客户端创建一个接口，供其他Grain连接到Grain服务。

``` csharp
public interface IDataServiceClient : IGrainServiceClient<IDataService>, IDataService {
}
```

**第四步.** 创建Grain服务客户端。
它基本上只是一个数据服务的代理。
但是，你必须手动输入所有的方法映射，但只需要一行。

``` csharp
public class DataServiceClient : GrainServiceClient<IDataService>, IDataServiceClient {

    public DataServiceClient(IServiceProvider serviceProvider) : base(serviceProvider) {
    }

    public Task MyMethod()  => GrainService.MyMethod();
}
```

**第五步.** 
将Grain服务客户端注入到需要它的其他Grain中。
注意，Grain服务客户端并不保证能访问本地Silo上的Grain服务。
你的命令有可能被发送到集群中任何Silo上的Grain服务。

``` csharp
public class MyNormalGrain: Grain<NormalGrainState>, INormalGrain {

    readonly IDataServiceClient DataServiceClient;

    public MyNormalGrain(IGrainActivationContext grainActivationContext, IDataServiceClient dataServiceClient) {
                DataServiceClient = dataServiceClient;
    }
}
```

**第六步.** 
将Grain服务注入到Silo中。
这是为了Silo能够启动Grain服务。

``` csharp
(ISiloHostBuilder builder) => builder .ConfigureServices(services => { services.AddSingleton<IDataService, DataService>(); });

```

## 附加说明

### 说明 1

`ISiloHostBuilder`上有一个扩展方法：`AddGrainService<SomeGrainService>()`。
其类型约束是。`where T : GrainService`。
它最终会调用到这里：

```csharp
return services.AddSingleton<IGrainService>(sp => GrainServiceFactory(grainServiceType, sp));
```

基本上，Silo在启动时从服务提供者那里获取了`IGrainService`类型：

```csharp
var grainServices = this.Services.GetServices<IGrainService>();
```
Grainservice项目应该引用`Microsoft.Orleans.OrleansRuntime` Nuget包。
### 说明 2

为了使其发挥作用，你必须同时注册该服务和它的客户端。
像是这样：
``` csharp
  var builder = new SiloHostBuilder()
      .AddGrainService<DataService>()  // Register GrainService
      .ConfigureServices(s =>
       {
          // Register Client of GrainService
          s.AddSingleton<IDataServiceClient, DataServiceClient>(); 
      })
```
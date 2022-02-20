---
title: 无状态Worker Grains
---

默认情况下，Orleans运行时在集群内对每个Grain创建至多一个激活。
这是对虚拟Actor模型最直观的表达，每个Grain都对应于一个具有独特类型/身份的实体。
然而，在有些情况下，应用程序需要执行与系统中某个特定实体无关的功能性无状态操作。
例如，如果客户端发送的请求带有压缩的有效载荷，需要在它们被路由到目标Grain进行处理之前进行解压缩，这种解压缩/路由逻辑不与应用程序中的特定实体相联系，并且可以轻松地扩展。

当`[StatelessWorker]`特性应用于Grain类时，它向Orleans运行时表明，该类对应的Grain应被视为**无状态Work** Grain。
**无状态Worker** Grain具有以下属性，这使得它们的执行与普通Grain类非常不同。

1. Orleans运行时可以并将在集群的不同Silo上创建同一个无状态Worker Grain的多个激活。
2. 对无状态Worker Grain的请求总是在本地执行，也就是在请求来源的同一个Silo上执行，要么是由Silo上运行的Grain发出的，要么是由Silo的客户端网关接收的。因此，从其他Grain或客户端网关对无状态Worker Grain的调用永远不会产生远程消息。
3. 如果已经存在的无状态Worker Grain繁忙，Orleans运行时将自动创建额外的激活。除非通过可选的`maxLocalWorkers`参数显式指定，否则运行时为每个Silo创建的无状态Worker Grain的最大激活数量默认受限于机器上的CPU核心数。
4. 由于2和3，无状态Worker Grain的激活是不可单独寻址的。对一个无状态Worker Grain的两个后续请求可能会被它的不同激活所处理。

无状态Worker Grain提供了一种直接的方式来创建一个自动管理的Grain激活池，它能根据实际负载自动伸缩。
运行时总是以相同的顺序扫描可用的无状态Worker Grain的激活。
正因为如此，它总是将请求分派给它能找到的第一个空闲的本地激活，并且只有在所有先前的激活都繁忙的情况下才会获取到最后一个。
如果所有的激活都很忙，而且还没有达到激活限制，它就在列表的末尾再创建一个激活，并把请求派发给它。
这意味着，当对无状态Worker Grain的请求率增加，而现有的激活目前都很忙时，运行时会将其激活池扩大到上限。
反之，当负载下降，并且可以由较少数量的无状态Worker Grain的激活来处理时，位于列表尾部的激活将没有请求派发给它们。
它们将进入闲置状态，并最终被标准的激活回收进程停用。
因此，激活池最终会缩减，以匹配负载。

下面的例子定义了一个无状态Worker Grain类`MyStatelessWorkerGrain`，使用默认的最大激活数限制。
``` csharp
[StatelessWorker]
public class MyStatelessWorkerGrain : Grain, IMyStatelessWorkerGrain
{
 ...
}
```

调用一个无状态Worker Grain与调用其他Grain是一样的。
唯一的区别是，在大多数情况下，只使用一个Grain ID，即0或`Guid.Empty`。
当需要有多个无状态Worker Grain池时，可以使用多个Grain ID，每个池一个。

``` csharp
var worker = GrainFactory.GetGrain<IMyStatelessWorkerGrain>(0);
await worker.Process(args);
```

这个定义了一个无状态Worker Grain类，每个Silo不超过一个Grain激活。
``` csharp
[StatelessWorker(1)] // max 1 activation per silo
public class MyLonelyWorkerGrain : ILonelyWorkerGrain
{
 ...
}
```

注意，`[StatelessWorker]`特性不会改变目标Grain类的可重入性。
就像其他Grain一样，无状态Worker Grain默认为非可重入。
它们可以通过给Grain类添加`[Reentrant]`特性来显式指定其为可重入。

## 状态

“无状态Worker”中的“无状态”并不意味着无状态Worker不能有状态或只限于执行功能操作。
就像其他Grain一样，无状态Worker Grain可以加载并在内存中保留它需要的任何状态。
只是因为在集群的相同和不同的Silo上可以创建多个无状态Worker Grain的激活，所以没有一个简单的机制来协调不同激活所持有的状态。

有几种有用的模式用到了持有状态的无状态Worker。

### 规模化的热缓存项目

对于经历高吞吐量的热缓存项目，将每个这样的项目保存在无状态Worker Grain中，使其 
a) 在一个Silo内和集群中的所有Silo中自动扩展。
b) 使数据在通过客户网关接收客户请求的Silo上始终是本地可用的，这样就可以在不额外跳转到另一个Silo的情况下回应请求。

### 缩减式聚合

在某些情况下，应用需要计算集群中所有特定类型的Grain的某些指标，并定期报告汇总结果。
例如，报告每个游戏地图的玩家数量、VoIP电话的平均持续时间等。
如果数以千计或数以百万计的Grain中的每一个都向一个单一的全局聚合器报告他们的指标，聚合器将立即超载，从而无法处理大量的报告。
另一种方法是把这个任务变成两个（或更多）步骤的缩减式聚合。
第一层聚合是由报告Grain发送他们的指标到无状态Worker预聚合Grain来完成的。
Orleans运行时将自动为每个Silo创建多个无状态Worker Grain的激活。
由于所有这些调用将在本地处理，没有远程调用或消息的序列化，这种聚合的成本将大大低于远程的情况。
于是，每个预聚合的无状态Wokrer Grain的激活，都可以独立或与其他本地激活协调，
并且将它们的聚合报告发送到全局的最终聚合器（如果需要的话，也可以发送到下一个还原层），且避免其超载。

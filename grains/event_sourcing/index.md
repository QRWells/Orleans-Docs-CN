---
title: 事件溯源
---

事件溯源提供了一种灵活的方式来管理和持久化Grain状态。与标准Grain相比，事件溯源的（event-sourced）Grain有许多潜在的优势。首先，它可以与许多不同的存储供应者配置一起使用，并支持多个集群的跨地域复制。此外，它将Grain类、Grain状态（由Grain状态对象表示）和Grain更新（由事件对象表示）的定义干净地分离。

这一部分文档的结构如下：

* [日志式Grain基础](journaledgrain_basics.md)解释了如何通过派生自`JournaledGrain`来定义一个事件溯源的Grain，如何访问当前状态，以及如何引发更新状态的事件。

* [复制的实例](replicated_instances.md)解释了事件溯源机制如何处理复制的Grain实例并确保一致性。它讨论了竞争事件和冲突的可能性，以及如何解决这些问题。

* [即时/延时确认](immediate_vs_delayed_confirmation.md)解释了事件的延迟确认和可重入怎样提高可用性和吞吐量。

* [通知](notifications.md)解释了如何订阅通知，其允许Grain对新事件作出反应。

* [事件溯源配置](event_sourcing_configuration.md)解释了如何配置项目、集群和日志一致性提供者。

* [内置的日志一致性提供者](log_consistency_providers.md)解释了目前内置的三种日志一致性提供者如何工作。

* [日志式Grain的诊断](journaledgrain_diagnostics.md)解释了如何监控连接错误，并获得简单的统计数据。


就日志式Grain的API而言，上述的行为是相当稳定的。然而，我们希望能尽快增加或改变日志一致性提供者，以便更容易让开发者接入标准的事件存储系统。


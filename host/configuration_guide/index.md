---
title: Orleans配置指南
---

本配置指南解释了关键的配置参数，以及它们在大多数典型场景下应该如何使用。

Orleans可以在各种配置中使用，以适应不同的使用场景，如用于开发和测试的本地单节点部署、服务器集群、多实例Azure worker role等。

本指南提供了关键配置参数的说明，这些参数是使Orleans在某个目标场景中运行所必需的。还有其他一些配置参数，主要是帮助微调Orleans以获得更好的性能。

Silo和客户端分别通过`SiloHostBuilder`和`ClientBuilder`以及一些补充的选项类进行程序化配置。
Orleans的选项类遵循[ASP.NET选项](https://docs.microsoft.com/zh-cn/aspnet/core/fundamentals/configuration/options/)模式，可以通过文件、环境变量等加载。
请参考[选项模式文档](https://docs.microsoft.com/zh-cn/aspnet/core/fundamentals/configuration/options/)以了解更多信息。

如果你想为本地开发配置一个Silo和一个客户端，请看[本地开发配置](local_development_configuration.md)部分。
本指南的[服务器配置](server_configuration.md)和[客户端配置](client_configuration.md)部分分别涉及配置Silo和客户端。

[典型配置](typical_configurations.md)一节提供了一些常见配置的总结。

可配置的重要核心选项的列表可以在[这一节](list_of_options_classes.md)中找到。

**重要**：确保你正确配置了.NET垃圾回收，详见[配置.NET垃圾回收](configuring_.NET_garbage_collection.md)。
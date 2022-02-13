---
title: 版本选择器策略
description:
---

当集群中存在同一Grain接口的几个版本，并且需要创建一个新的激活，将根据`GrainVersioningOptions.DefaultVersionSelectorStrategy`中定义的策略选择一个[兼容的版本](compatible_grains.md)。

Orleans开箱即用，支持以下策略：

## 所有兼容的版本（默认）

使用这种策略，新的激活的版本将在所有兼容的版本中随机选择。

例如，如果我们有一个给定的Grain接口的2个版本，V1和V2：

  - V2向后兼容V1
  - 在集群中，有2个Silo支持V2，8个支持V1。
  - 该请求是由V1客户端/Silo发起的

在这种情况下，新的激活有20%的机率是V2，80%的机率是V1。

## 最新版本

使用这种策略，新的激活的版本将始终是最新的兼容版本。

例如，如果我们有一个给定的Grain接口的两个版本，V1和V2 (V2是向后或完全兼容V1的），那么所有新的激活都将是V2。

## 最小版本

使用这种策略，新激活的版本将始终是要求的或最小的兼容版本。

For example if we have 2 versions of a given grain interface, V2, V3, all fully 
compatibles:
例如，如果我们有两个版本的谷物接口，V2和V3，都是完全兼容：

  - 如果请求是从V1客户端/Silo发起的，新的激活将是V2
  - 如果请求是从V3客户端/Silo发出的，新的激活也将是V2
---
title: 实现细节
---
# 概览

## [Orleans生命周期](orleans_lifecycle.md)

一些Orleans的行为足够复杂，需要有序的启动和关闭。
为了解决这个问题，我们引入了一个通用的组件生命周期模式。

## [消息交付的保证](messaging_delivery_guarantees.md)

Orleans的消息传递保证默认情况下是**至多一次的**。
也可以配置为超时后重试，Orleans会提供至少一次的交付。

## [调度器](scheduler.md)

Orleans调度器是Orleans运行时中的一个组件，负责执行应用程序代码和部分运行时代码，以确保单线程执行语义。

## [集群管理](cluster_management.md)

Orleans通过一个内置的成员协议提供集群管理，我们有时将其称为Silo成员。
这个协议的目的是让所有Silo（Orleans服务器）与当前存活的Silos达成一致，检测失败的Silo，并允许新Silo加入集群。

## [流的实现](streams_implementation/index.md)

本节提供了Orleans流实现的高层次概述。
它描述了在应用层面上看不到的概念和细节。

## [负载均衡](load_balancing.md)

负载均衡，从广义上讲，是Orleans运行时的支柱之一。

## [单元测试](testing.md)

本节展示了如何对你的Grain进行单元测试，以确保它们的行为正确。
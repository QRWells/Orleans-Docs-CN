---
title: Grain接口版本控制
description:
---

> [!警告]
> 本页介绍了如何使用Grain接口的版本控制。Grain状态的版本控制不在讨论范围之内。

## 概览
在一个给定的集群上，Silo可以支持不同版本的粮食类型。
![具有不同版本的Grain的集群](/grains/grain_versioning/version.png)
在这个例子中，客户端和Silo{1,2,3}使用了版本1，Silo 4使用了版本2的`A`进行编译。

## 限制:

-	无状态worker无法进行版本控制
-	流接口是不分版本的

## 启用版本控制

如果没有显式地将`[Version]`特性添加到Grain接口上，那么Grain的默认版本为0。你可以通过在Grain接口上使用`[Version]`特性对Grain进行版本控制：

``` cs
[Version(X)]
public interface IVersionUpgradeTestGrain : IGrainWithIntegerKey {}
```

其中，`X`是Grain接口的版本号，一般是单调增加的。

## Grain版本兼容性及安置
当一个来自版本化的Grain的调用到达集群时：
- 如果当前没有对应的激活，就创建一个兼容的激活。
- 如果存在对应的激活:
  - 如果其版本不兼容，那它会被停用，一个兼容的激活会被创建（见[版本选择器策略](version_selector_strategy.md)）。
  - 如果其版本兼容（见[兼容的Grain](compatible_grains.md)），则该调用会被正常处理。

默认地:
- 所有版本化的Grain都应向后兼容（参见[向后兼容指南](backward_compatibility_guidelines.md)和[兼容的Grain](compatible_grains.md)）。也就是说，v1的Grain可以调用v2的Grain，但v2的Grain不能调用v1的Grain。
- 当集群中存在多个版本时，新的激活将被随机地放在一个兼容的Silo上。

你可以通过选项`GrainVersioningOptions`来改变这个默认行为：

```csharp
var silo = new SiloHostBuilder()
  [...]
  .Configure<GrainVersioningOptions>(options => 
  {
    options.DefaultCompatibilityStrategy = nameof(BackwardCompatible);
    options.DefaultVersionSelectorStrategy = nameof(MinimumVersion);
  })
  [...]
```

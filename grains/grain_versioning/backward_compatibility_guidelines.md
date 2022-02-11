---
title: 向后兼容指南
description: 
---

编写向后兼容的代码可能很困难，而且难以测试。

## 不要改变现有方法的签名

由于Orleans序列化器的工作方式，你不应该改变现有方法的签名。

下面的例子是正确的：

``` cs
[Version(1)]
public interface IMyGrain : IGrainWithIntegerKey
{
  // First method
  Task MyMethod(int arg);
}
```
``` cs
[Version(2)]
public interface IMyGrain : IGrainWithIntegerKey
{
  // Method inherited from V1
  Task MyMethod(int arg);

  // New method added in V2
  Task MyNewMethod(int arg, obj o);
}
```

这个则不正确:

``` cs
[Version(1)]
public interface IMyGrain : IGrainWithIntegerKey
{
  // First method
  Task MyMethod(int arg);
}
```
``` cs
[Version(2)]
public interface IMyGrain : IGrainWithIntegerKey
{
  // Method inherited from V1
  Task MyMethod(int arg, obj o);
}
```

**注意**：你不应该在你的代码中做这样的改变，因为这个例子展示了导致非常糟糕的副作用的错误做法。

这个例子解释了如果你只是重命名参数名称会发生什么：
假设我们在集群中部署了以下两个接口版本：

``` cs
[Version(1)]
public interface IMyGrain : IGrainWithIntegerKey
{
  // return a - b
  Task<int> Substract(int a, int b);
}
```
``` cs
[Version(2)]
public interface IMyGrain : IGrainWithIntegerKey
{
  // return y - x
  Task<int> Substract(int y, int x);
}
```

这两个方法似乎是相同的。但是，如果客户端是用V1调用的，而请求是由V2的激活处理的：

``` cs
var grain = client.GetGrain<IMyGrain>(0);
var result = await grain.Substract(5, 4); // 会返回"-1"而不是期望的"1"
```

这是由于内部的Orleans序列化器的工作方式造成的。

## 避免改变现有方法的逻辑

这似乎很显然，但是当你修改一个现有方法的方法体时，你要非常小心。
除非你要修复一个错误，否则如果你需要修改代码，最好是直接添加一个新方法。

示例:

``` cs
// V1
public interface MyGrain : IMyGrain
{
  // First method
  Task MyMethod(int arg)
  {
    SomeSubRoutine(arg);
  }
}
```
``` cs
// V2
public interface MyGrain : IMyGrain
{
  // Method inherited from V1
  // Do not change the body
  Task MyMethod(int arg)
  {
    SomeSubRoutine(arg);
  }

  // New method added in V2
  Task MyNewMethod(int arg)
  {
    SomeSubRoutine(arg);
    NewRoutineAdded(arg);
  }
}
```

## 不要从Grain接口中移除方法

除非你确定方法不再被使用，否则你不应该从Grain接口中移除方法。
如果你想移除方法，这应该分两步进行：

1. 部署V2的Grain，同时将V1的方法标记为`Obsolete`：

  ``` cs
  [Version(1)]
  public interface IMyGrain : IGrainWithIntegerKey
  {
    // First method
    Task MyMethod(int arg);
  }
  ```
  ``` cs
  [Version(2)]
  public interface IMyGrain : IGrainWithIntegerKey
  {
    // Method inherited from V1
    [Obsolete]
    Task MyMethod(int arg);

    // New method added in V2
    Task MyNewMethod(int arg, obj o);
  }
  ```

2. 当你确定不再有V1的调用时（实际上V1不再部署在运行的集群中），部署V3并删除V1方法。
  ``` cs
  [Version(3)]
  public interface IMyGrain : IGrainWithIntegerKey
  {
    // New method added in V2
    Task MyNewMethod(int arg, obj o);
  }
  ```

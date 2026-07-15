---
title: C# using 用法
date: 2026-07-12 00:00:00 +0800
categories: [Languages, CSharp]
tags: [csharp, using, idisposable, resource-management]
description: 梳理 C# using 语句与声明的写法、资源释放顺序、所有权约定，以及常见陷阱与实践原则。
media_subpath: /assets/img/posts/2026-07-12-csharp-using
---

# C# `using`

## 1. 核心作用

用于确保实现了 `IDisposable` 的对象在离开作用域时调用 `Dispose()`。

```csharp
using var stream = File.OpenRead(path);
```

本质上近似：

```csharp
var stream = File.OpenRead(path);

try
{
    // 使用 stream
}
finally
{
    stream?.Dispose();
}
```

因此即使发生异常或提前 `return`，资源也会被释放。

------

## 2. 两种写法

### `using` 语句

```csharp
using (var connection = CreateConnection())
{
    Execute(connection);
}
```

离开花括号时立即释放，适合数据库连接、事务、锁等需要尽早释放的资源。

### `using` 声明

```csharp
using var connection = CreateConnection();

Execute(connection);
```

在当前作用域结束时释放，代码更简洁，但可能延长资源生命周期。

选择原则：

> 优先根据释放时机选择，而不是根据缩进多少选择。

------

## 3. 多个资源的释放顺序

```csharp
using var connection = CreateConnection();
using var command = connection.CreateCommand();
using var reader = command.ExecuteReader();
```

释放顺序与声明顺序相反：

```text
reader
command
connection
```

这种顺序正好符合资源依赖关系。

------

## 4. `Dispose()` 不等于回收内存

`Dispose()` 通常用于：

- 关闭文件或网络流；
- 归还数据库连接；
- 释放系统句柄；
- 结束事务、锁或临时作用域。

对象本身占用的托管内存仍由 GC 回收。

------

## 5. 注意资源所有权

通常遵循：

> 谁创建资源，谁负责释放。

```csharp
void Read(Stream stream)
{
    // 通常不要在这里 Dispose
}
```

如果资源由外部传入，除非接口明确表示转移所有权，否则不应擅自释放。

对于包装流，还要关注 `leaveOpen`：

```csharp
using var writer = new StreamWriter(
    stream,
    Encoding.UTF8,
    leaveOpen: true);
```

这样释放 `writer` 时不会关闭底层 `stream`。

------

## 6. 常见陷阱

不要手动释放却缺少异常保护：

```csharp
var stream = File.OpenRead(path);
Process(stream);
stream.Dispose();
```

不要让高成本资源无意义地存活到方法末尾：

```csharp
using var connection = CreateConnection();

Query(connection);
DoLongCalculation(); // 此时连接仍未释放
```

更合适：

```csharp
using (var connection = CreateConnection())
{
    Query(connection);
}

DoLongCalculation();
```

不要继续使用已经释放的对象：

```csharp
var reader = File.OpenText(path);

using (reader)
{
    Process(reader);
}

// reader 仍在作用域中，但已经被释放
```

优先在 `using` 内直接声明变量。

------

## 7. 实践原则

- 文件流、数据库连接、事务等资源应使用 `using`。
- 尽量缩小资源作用域。
- 按依赖顺序声明资源，使其反向释放。
- 不要释放不属于当前代码的资源。
- DI 容器创建的对象通常由容器负责释放。
- `using` 保证调用 `Dispose()`，但不保证 `Dispose()` 本身不会抛异常。

一句话总结：

> `using` 是通过 `try/finally` 语义实现的确定性资源释放机制，重点是明确资源所有权和释放时机。

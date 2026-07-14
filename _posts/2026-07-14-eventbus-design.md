---
title: 事件总线设计
date: 2026-07-14 00:00:00 +0800
categories: [GameDev, Gameplay]
tags: [eventbus, csharp, observer-pattern, architecture, unity, gameplay]
description: 梳理强类型 EventBus 的定位、三层结构、生命周期、Publish 快照、SubscribeOnce 移除时机、Delegate 陷阱与异常隔离，以及「事件 vs 命令」的使用边界。
media_subpath: /assets/img/posts/2026-07-14-eventbus-design
---

# 事件总线设计

## 定位

EventBus 本质还是事件系统，只不过用事件类型（Type）作为事件标识，而不是字符串。它解决的是跨模块通知问题：发送方只负责发布事件，不关心谁会处理；接收方只负责订阅自己关心的事件，不关心事件是谁发出来的。

## 实现结构

整个实现只有三层：

- `IEventBus`：定义接口，方便测试或替换实现。
- `GameEventBus`：真正保存订阅关系并负责事件派发。
- `EventBus`：静态门面，业务层唯一入口。

静态门面和 `Log` 的设计是一致的。业务代码只和静态类交互，真正工作的对象由 Bootstrap 初始化并维护生命周期。

## 生命周期

EventBus 属于基础设施，因此需要和 Logger 一样在 Bootstrap 中初始化。初始化顺序一般放在 Logger 后面，**因为** EventBus 内部可能需要输出 Debug 或 Error 日志。

关闭时也不要直接 Dispose Logger，而是先调用 `EventBus.Shutdown()`，确保清理过程中产生的日志还能正常输出。

这里需要区分 `Clear()` 和 `Shutdown()`。`Clear()` 只是清空所有监听关系，Bus 本身仍然可以继续使用；`Shutdown()` 则表示整个 EventBus 生命周期结束，除了清空监听外，还会释放实例并让静态门面失效。

## Publish 的实现细节

Publish（发布事件） 最大的细节就是不要直接遍历监听列表，而是先复制一份监听者的快照。

原因很简单：Handler 执行过程中完全可能调用 `Subscribe()`、`Unsubscribe()` 或者 `SubscribeOnce()`，如果直接遍历原 List，很容易出现索引错乱、漏调用甚至 `InvalidOperationException`。因此通常会先 `ToArray()`，再遍历数组，这样一次 Publish 的目标集合从开始派发时就已经确定了，本轮过程中发生的订阅或退订，只影响下一轮。

## SubscribeOnce

Once 的关键不是监听一次，而是**移除时机**。

正确顺序应该是：

```text
Remove -> Invoke
```

而不是：

```text
Invoke -> Remove
```

如果 Handler 内再次 Publish 同类型事件，后者会导致 Once 被重复调用。

## Delegate 陷阱

订阅和取消订阅时，要传入同一个委托对象。Unsubscribe 能否成功，取决于 Delegate 是否相等，而不是代码是否长得一样。

下面这种写法永远取消不了：

```c#
Subscribe<PlayerDied>(e => Foo(e));
Unsubscribe<PlayerDied>(e => Foo(e));
```

因为两次 lambda 会生成两个不同的 Delegate。

比较稳妥的方式是使用命名方法，或者把 lambda 保存成字段，再用于 Subscribe 和 Unsubscribe。

## 异常处理

EventBus 一般不会让某个监听器的异常中断整个事件链，而是对每个 Handler 单独进行 `try/catch`。这样即使 A 抛出了异常，B、C 仍然可以继续收到事件。

```c#
foreach (var handler in handlers)
{
    try
    {
        handler(evt);
    }
    catch (Exception e)
    {
        Log.Error(e);
    }
}
```

这里更准确的说法是**隔离异常**，而不是吞异常。异常仍然应该通过 Logger 输出，只是不影响其他监听者。

## EventBus 的边界

项目后期最容易出现的问题不是 EventBus 不够用，而是什么都往 EventBus 里塞。

我的判断标准只有一句话：

> 「发生了什么」用 EventBus；「我要你做什么」用接口或者直接调用。

例如 `PlayerDied`、`SceneLoaded`、`QuestCompleted` 都属于事件；而 `OpenInventory`、`PlaySound`、`MovePlayer` 更像命令，不应该通过 EventBus 发送。

---
title: C# 泛型
date: 2026-07-10 00:00:00 +0800
categories: [Languages, CSharp]
tags: [csharp, generics, covariance, contravariance, type-safety]
description: 梳理 C# 泛型的基本用法、约束、开放与封闭泛型、协变与逆变、性能特点，以及常见应用与设计建议。
media_subpath: /assets/img/posts/2026-07-10-csharp-generics
---

# C# 泛型

泛型允许把类型作为参数传递给类、接口、方法和委托，使同一套代码可以处理不同类型，同时保留编译期类型检查。

它的核心思想是：

> 将具体类型延后到使用时确定，对相同的结构和行为进行类型安全的抽象。

相比使用 `object`，泛型可以减少强制类型转换，避免部分装箱和拆箱，并让很多类型错误提前在编译阶段暴露。

## 泛型的基本使用

### 泛型类

泛型类适合表示结构相同、内部数据类型不同的对象。

```csharp
public class Box<T>
{
    public T Value { get; set; }

    public Box(T value)
    {
        Value = value;
    }
}
```

使用泛型类时，需要提供实际类型：

```csharp
Box<int> intBox = new(10);
Box<string> stringBox = new("Hello");
```

`T` 是类型参数。它只是一个占位符，等到使用 `Box<T>` 时才会被替换成具体类型。

`Box<int>` 和 `Box<string>` 是两个不同的封闭泛型类型。虽然它们来自同一个泛型定义，但类型之间不能直接相互赋值。

泛型可以同时声明多个类型参数：

```csharp
public class Cache<TKey, TValue>
{
    private readonly Dictionary<TKey, TValue> _items = new();
}
```

类型参数名称应尽量体现作用。只有类型含义非常通用时才使用 `T`；多个类型参数更适合使用 `TKey`、`TValue`、`TResult`、`TEntity` 等名称。

### 泛型方法

泛型方法可以定义在普通类中，也可以定义在泛型类中。

```csharp
public static T GetFirst<T>(IEnumerable<T> source)
{
    return source.First();
}
```

调用时，编译器通常可以根据实参推断类型：

```csharp
int number = GetFirst(new[] { 1, 2, 3 });
string name = GetFirst(new[] { "A", "B" });
```

也可以显式指定类型：

```csharp
int number = GetFirst<int>(new[] { 1, 2, 3 });
```

C# 的泛型类型推断主要根据方法参数完成，通常不能只根据返回值推断类型：

```csharp
public static T Create<T>()
{
    return default!;
}

// 无法推断 T
// string value = Create();

string value = Create<string>();
```

### 泛型接口

泛型接口用于描述一组操作方式相同，但处理类型不同的组件。

```csharp
public interface IRepository<T>
{
    T? GetById(long id);
    void Add(T entity);
}
```

实现接口时，需要指定具体类型：

```csharp
public class UserRepository : IRepository<User>
{
    public User? GetById(long id)
    {
        return null;
    }

    public void Add(User entity)
    {
    }
}
```

泛型接口常用于仓储、缓存、序列化器、工厂、处理器和事件系统等基础组件。

## 泛型约束

默认情况下，编译器只知道 `T` 是某种类型，因此不能直接访问它的特定成员。泛型约束使用 `where` 声明类型参数必须满足的条件。

约束的作用不仅是限制可传入的类型，更重要的是告诉编译器：泛型代码可以安全使用哪些能力。

### 常用约束

`class` 表示 `T` 必须是引用类型：

```csharp
public class Repository<T> where T : class
{
}
```

启用可空引用类型后，`class` 通常表示不可空引用类型，`class?` 允许可空引用类型。

`struct` 表示 `T` 必须是非空值类型：

```csharp
public class ValueCache<T> where T : struct
{
}
```

基类约束表示 `T` 必须继承指定类型：

```csharp
public void Save<T>(T entity) where T : Entity
{
    Console.WriteLine(entity.Id);
}
```

接口约束表示 `T` 必须实现指定接口：

```csharp
public void Validate<T>(T value) where T : IValidatable
{
    value.Validate();
}
```

多个接口约束可以组合：

```csharp
public void Process<T>(T value) where T : IEntity, IValidatable
{
}
```

`new()` 表示 `T` 必须具有公开无参构造函数：

```csharp
public T Create<T>() where T : new()
{
    return new T();
}
```

组合多个约束时，`new()` 必须写在最后：

```csharp
public T CreateEntity<T>() where T : Entity, IDisposable, new()
{
    return new T();
}
```

`notnull` 表示类型参数不应为可空类型，常用于字典键和缓存键：

```csharp
public class Cache<TKey, TValue> where TKey : notnull
{
}
```

### 约束的设计意义

约束应该表达泛型逻辑真正依赖的能力。例如，一段代码需要调用 `Validate()`，就应约束 `T` 实现 `IValidatable`，而不是在方法内部通过反射或类型判断寻找该方法。

```csharp
public void Execute<T>(T value) where T : IValidatable
{
    value.Validate();
}
```

约束越准确，泛型代码的意图就越清晰，编译器也能提供更完整的类型检查。

## 泛型的类型特性

### 开放泛型与封闭泛型

没有提供实际类型参数的泛型称为开放泛型：

```csharp
typeof(List<>)
typeof(Dictionary<,>)
```

已经指定实际类型参数的泛型称为封闭泛型：

```csharp
typeof(List<int>)
typeof(Dictionary<string, int>)
```

开放泛型不能直接创建实例，但经常用于反射和依赖注入：

```csharp
services.AddScoped( typeof(IRepository<>), typeof(Repository<>));
```

这表示所有 `IRepository<T>` 都使用对应的 `Repository<T>` 实现。

### 泛型静态成员

泛型类型的静态成员会按照封闭泛型类型分别保存。

```csharp
public class Counter<T>
{
    public static int Value;
}

Counter<int>.Value = 1;
Counter<string>.Value = 2;
```

`Counter<int>.Value` 和 `Counter<string>.Value` 是两份互不影响的数据。

这一特性可以用于按类型缓存：

```csharp
public static class TypeCache<T>
{
    public static readonly Type Type = typeof(T);
}
```

但也要注意，每一种不同的 `T` 都会拥有自己的静态状态。如果泛型参数种类很多，静态字段保存的对象可能长期存活。

### `default` 的含义

泛型代码无法提前确定 `T` 是值类型还是引用类型，因此可以使用：

`T value = default;`

实际结果取决于具体类型：

```text
int        -> 0
bool       -> false
引用类型    -> null
普通结构体  -> 所有字段的默认值
```

启用可空引用类型后，经常会看到：

`return default!;`

`!` 只会抑制编译器的空引用警告，不会改变运行时值。对于引用类型，返回结果仍然可能是 `null`。

## 协变与逆变

泛型默认是不变的。即使 `string` 继承自 `object`，`List<string>` 也不能直接转换为 `List<object>`：

```csharp
List<string> strings = new();

// 编译错误
// List<object> objects = strings;
```

如果允许这种转换，就可以通过 `List<object>` 向原列表加入其他类型，从而破坏原本只允许保存字符串的类型约束。

### 协变

协变使用 `out`，表示类型参数只作为输出使用。

```csharp
public interface IProducer<out T>
{
    T Create();
}
```

例如：

```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;
```

`IEnumerable<T>` 只负责向外提供元素，因此字符串序列可以安全地当作对象序列读取。

可以将协变理解为：

> 更具体的输出，可以当作更宽泛的输出使用。

### 逆变

逆变使用 `in`，表示类型参数只作为输入使用。

```csharp
public interface IConsumer<in T>
{
    void Consume(T value);
}
```

例如：

```csharp
Action<object> printObject = value => Console.WriteLine(value);
Action<string> printString = printObject;
```

能够处理任意 `object` 的方法，自然也能够处理 `string`。

可以将逆变理解为：

> 能接收更宽泛类型的对象，也能用于接收更具体的类型。

`in` 和 `out` 只能声明在泛型接口和泛型委托上，不能直接用于普通泛型类。

简单记忆：

> `out` 负责输出，`in` 负责输入。

## 泛型的性能特点

泛型集合可以避免使用 `object` 时产生的强制转换和部分装箱开销。

**装箱**是把一个值类型转换成 `object` 或它实现的接口类型。

**拆箱**是从已经装箱的对象中取回值类型。

非泛型集合保存值类型时，需要装箱：

```csharp
ArrayList list = new();
list.Add(10);

int number = (int)list[0];
```

使用泛型集合时，元素类型在编译期已经确定：

```csharp
List<int> list = new();
list.Add(10);

int number = list[0];
```

这段代码不需要将 `int` 装箱成 `object`，取出时也不需要强制转换。

不过，泛型并不意味着任何情况下都不会装箱。如果将泛型值转换为 `object`，并且实际类型是值类型，仍然会发生装箱：

```csharp
public void Print<T>(T value)
{
    object boxed = value!;
}
```

泛型的性能优势主要来自保留具体类型信息，而不是一种自动消除所有运行时开销的机制。

## 泛型的常见应用

泛型适合结构一致、行为相似，但处理类型不同的场景。常见应用包括：

- `List<T>`、`Dictionary<TKey, TValue>` 等集合；
- `ObjectPool<T>` 对象池；
- `Cache<TKey, TValue>` 缓存；
- `IRepository<TEntity>` 仓储；
- `IFactory<T>` 工厂；
- `Handler<TRequest, TResponse>` 请求处理器；
- 泛型事件和消息系统；
- 序列化与反序列化；
- 与具体类型无关的算法。

例如一个简单的对象池：

```csharp
public class ObjectPool<T>
    where T : new()
{
    private readonly Stack<T> _items = new();

    public T Get()
    {
        return _items.Count > 0
            ? _items.Pop()
            : new T();
    }

    public void Release(T item)
    {
        _items.Push(item);
    }
}
```

这个实现只适合构造过程简单的对象。实际项目中，特别是 Unity 项目，对象创建可能依赖 `Instantiate`、Prefab、初始化参数或依赖注入，此时更适合传入工厂方法：

```csharp
public class ObjectPool<T>
{
    private readonly Stack<T> _items = new();
    private readonly Func<T> _factory;

    public ObjectPool(Func<T> factory)
    {
        _factory = factory;
    }

    public T Get()
    {
        return _items.Count > 0
            ? _items.Pop()
            : _factory();
    }

    public void Release(T item)
    {
        _items.Push(item);
    }
}
```

## 常见误区与设计建议

泛型不是动态类型。`T` 仍然会在编译期接受严格的类型检查，而 `dynamic` 会将部分类型检查推迟到运行时。

泛型参数之间存在继承关系，也不代表泛型类型之间会自动产生相同的继承关系。即使 `Dog` 继承自 `Animal`，`List<Dog>` 也不是 `List<Animal>`。

不要在泛型内部大量判断具体类型：

```csharp
if (typeof(T) == typeof(User))
{
}
else if (typeof(T) == typeof(Order))
{
}
```

如果泛型实现中充满针对具体类型的分支，通常说明这段逻辑并不真正通用，更适合使用接口、多态、方法重载或独立处理器。

泛型参数也不宜过多：

```csharp
Handler<TRequest, TResponse, TEntity, TKey, TContext>
```

类型参数越多，调用和维护成本越高。如果一个类型需要同时理解请求、响应、实体、主键和上下文，应检查它是否承担了过多职责。

`new()` 约束也应谨慎使用。它只能调用公开无参构造函数，无法向构造函数传递依赖。复杂对象通常更适合通过 `Func<T>` 或工厂接口创建。

泛型适合抽象真正与具体类型无关的逻辑，但不应该为了减少几个重复类，就把业务规则差异很大的对象强行合并。

判断一个泛型设计是否合理，可以考虑三个问题：

- 这段逻辑是否真的与具体类型无关；
- 类型参数必须具备哪些能力；
- 使用泛型是否比接口、多态或重载更清晰。

## 总结

泛型是在编译期类型安全的前提下，将类型作为参数进行抽象。它能够提高代码复用性，减少强制类型转换，并避免部分装箱开销。

泛型类和泛型方法用于复用结构与逻辑，泛型约束用于表达类型必须具备的能力，协变和逆变用于控制泛型接口与委托之间的类型转换关系。

泛型设计的目标并不是让一段代码支持所有类型，而是让一组具有相同结构或能力的类型，安全地复用同一套实现。

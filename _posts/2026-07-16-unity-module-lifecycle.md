---
title: Unity 客户端模块生命周期设计
date: 2026-07-16 00:00:00 +0800
categories: [GameDev, Unity]
tags: [unity, architecture, module-lifecycle, bootstrap, csharp, dependency]
description: 梳理 Unity 客户端模块生命周期管理器的设计目标、接口与基类、依赖拓扑排序、作用域、失败回滚与关闭顺序，以及 Init/Start/Update/Shutdown 的职责边界。
media_subpath: /assets/img/posts/2026-07-16-unity-module-lifecycle
---

# Unity 客户端模块生命周期设计

## 1. 设计目标

模块生命周期管理器用于统一管理客户端中长期存在或具有明确启动、运行、关闭过程的业务模块，例如配置、日志、网络、存档、账号、资源、音频、埋点、战斗和场景流程模块。

它的核心职责应保持稳定，不要随着业务增长变成一个新的“超级管理类”。一个合格的生命周期管理器通常只负责以下事情：

- 注册模块并保存模块实例。
- 校验模块依赖关系。
- 计算稳定的执行顺序。
- 驱动 `Init`、`Start`、`Update` 和 `Shutdown`。
- 管理模块和管理器自身的状态。
- 处理初始化失败、启动失败和关闭异常。
- 提供统一日志、耗时统计和调试信息。
- 管理 Application、Session、Scene 等不同作用域。

具体业务逻辑必须留在模块内部。生命周期管理器不应该知道登录协议、资源加载策略、战斗规则或 UI 流程。

### 1.1 生命周期语义

推荐将模块生命周期定义为：

```text
Created
  ↓
Initializing
  ↓
Initialized
  ↓
Starting
  ↓
Running
  ↓
ShuttingDown
  ↓
Shutdown
```

如果任意阶段失败，则进入 `Faulted` 状态，再执行回滚或关闭。

四个核心接口方法的职责应明确：

```text
Init      建立模块内部状态，准备依赖和资源
Start     在所有依赖已初始化后正式启动业务
Update    推进需要持续运行的逻辑
Shutdown  停止任务、解除订阅并释放资源
```

`Init` 和 `Start` 不应合并。`Init` 解决“模块是否准备好”，`Start` 解决“模块是否开始工作”。拆分后可以避免模块在初始化期间过早调用尚未准备完成的依赖。

### 1.2 模块边界

模块应具备相对清晰的职责和独立生命周期。以下对象通常适合作为模块：

- 网络连接与消息派发系统。
- 配置加载与版本配置系统。
- 存档和本地数据系统。
- 登录、账号和会话系统。
- Addressables 或自定义资源管理系统。
- 音频、埋点、日志和崩溃上报系统。
- 大厅、战斗和关卡流程系统。

以下对象通常不需要成为模块：

- 普通数据结构。
- 无状态工具类。
- 单个 UI View。
- 临时 Presenter 或 Controller。
- 只属于一个 GameObject 的行为组件。
- 生命周期完全由场景对象控制的轻量逻辑。

模块数量不应无限增长。一个模块如果只包装了一个工具方法，或者没有独立状态和关闭过程，通常没有必要接入生命周期管理器。

---

## 2. 核心接口设计

### 2.1 同步生命周期接口

基础版本可以使用同步接口：

```csharp
using System;
using System.Collections.Generic;

public interface IModule
{
    Type ModuleType { get; }

    IReadOnlyCollection<Type> Dependencies { get; }

    void Init(ModuleContext context);

    void Start();

    void Shutdown();
}
```

`ModuleType` 用于模块索引和错误信息。大多数情况下可以直接返回 `GetType()`，也可以在基类中统一实现。

`Dependencies` 用于声明当前模块必须依赖哪些模块。管理器根据它进行缺失依赖检查和拓扑排序。

如果模块需要逐帧更新，应通过独立接口声明，而不是让所有模块都实现空的 `Update`：

```csharp
public interface IModuleTick
{
    void Tick(float deltaTime);
}

public interface IModuleFixedTick
{
    void FixedTick(float fixedDeltaTime);
}

public interface IModuleLateTick
{
    void LateTick(float deltaTime);
}
```

不要在项目初期一次性加入所有 Tick 接口。只有确实存在对应调度需求时再添加。

### 2.2 模块基类

可以提供一个基类统一状态检查和默认依赖：

```csharp
using System;
using System.Collections.Generic;

public abstract class ModuleBase : IModule
{
    private static readonly IReadOnlyCollection<Type> EmptyDependencies =
        Array.Empty<Type>();

    public Type ModuleType => GetType();

    public virtual IReadOnlyCollection<Type> Dependencies =>
        EmptyDependencies;

    public ModuleState State { get; private set; } =
        ModuleState.Created;

    public void Init(ModuleContext context)
    {
        EnsureState(ModuleState.Created);

        State = ModuleState.Initializing;

        try
        {
            OnInit(context);
            State = ModuleState.Initialized;
        }
        catch
        {
            State = ModuleState.Faulted;
            throw;
        }
    }

    public void Start()
    {
        EnsureState(ModuleState.Initialized);

        State = ModuleState.Starting;

        try
        {
            OnStart();
            State = ModuleState.Running;
        }
        catch
        {
            State = ModuleState.Faulted;
            throw;
        }
    }

    public void Shutdown()
    {
        if (State is ModuleState.Created or ModuleState.Shutdown)
            return;

        if (State == ModuleState.ShuttingDown)
            return;

        State = ModuleState.ShuttingDown;

        try
        {
            OnShutdown();
            State = ModuleState.Shutdown;
        }
        catch
        {
            State = ModuleState.Faulted;
            throw;
        }
    }

    protected abstract void OnInit(ModuleContext context);

    protected abstract void OnStart();

    protected abstract void OnShutdown();

    private void EnsureState(ModuleState expected)
    {
        if (State != expected)
        {
            throw new InvalidOperationException(
                $"{ModuleType.Name} expected state {expected}, actual {State}.");
        }
    }
}
```

基类不是必需的。如果项目更偏好组合而不是继承，也可以让每个模块自行管理状态。不过基础设施中的状态检查最好有统一实现，避免不同模块出现不同规则。

### 2.3 模块上下文

`ModuleContext` 用于提供生命周期期间需要的公共环境：

```csharp
public sealed class ModuleContext
{
    public IModuleLogger Logger { get; }

    public IClock Clock { get; }

    public CancellationToken ApplicationToken { get; }

    public ModuleContext(
        IModuleLogger logger,
        IClock clock,
        CancellationToken applicationToken)
    {
        Logger = logger;
        Clock = clock;
        ApplicationToken = applicationToken;
    }
}
```

不要把 `ModuleContext` 设计成万能 Service Locator。模块的强依赖应优先通过构造函数注入，`ModuleContext` 只提供运行环境和跨模块通用能力。

---

## 3. 状态模型

### 3.1 模块状态

模块自身可以使用以下状态：

```csharp
public enum ModuleState
{
    Created,
    Initializing,
    Initialized,
    Starting,
    Running,
    ShuttingDown,
    Shutdown,
    Faulted
}
```

状态约束至少应保证：

- 只有 `Created` 可以进入 `Initializing`。
- 只有 `Initialized` 可以进入 `Starting`。
- 只有 `Running` 模块才参与 Tick。
- `Shutdown` 可以重复调用，但只能执行一次实际清理。
- `Faulted` 模块不能继续启动。
- 关闭阶段不再接收新业务请求。

模块内部暴露业务 API 时，也应检查状态：

```csharp
public void Send(Request request)
{
    if (State != ModuleState.Running)
    {
        throw new InvalidOperationException(
            "NetworkModule is not running.");
    }

    // 发送请求
}
```

这样可以尽早发现调用时机错误，而不是在内部资源为空时产生难以定位的空引用。

### 3.2 管理器状态

管理器也应维护独立状态：

```csharp
public enum ModuleManagerState
{
    Created,
    Initializing,
    Initialized,
    Starting,
    Running,
    ShuttingDown,
    Shutdown,
    Faulted
}
```

不要只使用 `_initialized`、`_started`、`_shutdown` 等多个布尔字段。多个布尔值容易形成非法组合，而枚举能直接表达当前唯一状态。

---

## 4. 模块注册与查询

### 4.1 注册规则

管理器应在初始化前完成所有模块注册：

```csharp
private readonly Dictionary<Type, IModule> _modules = new();

public void Register(IModule module)
{
    if (module == null)
        throw new ArgumentNullException(nameof(module));

    if (State != ModuleManagerState.Created)
    {
        throw new InvalidOperationException(
            "Modules can only be registered before initialization.");
    }

    var moduleType = module.ModuleType;

    if (!_modules.TryAdd(moduleType, module))
    {
        throw new InvalidOperationException(
            $"Module {moduleType.Name} has already been registered.");
    }
}
```

初始化后动态注册模块会让依赖顺序、Tick 列表和关闭顺序变得不稳定。如果业务确实需要动态加载模块，建议新建独立作用域管理器，而不是修改正在运行的管理器。

### 4.2 模块查询

模块查询可以使用泛型接口：

```csharp
public T Get<T>() where T : class, IModule
{
    if (_modules.TryGetValue(typeof(T), out var module))
        return (T)module;

    throw new KeyNotFoundException(
        $"Module {typeof(T).Name} is not registered.");
}
```

如果模块依赖通过构造函数注入，运行期间通常不需要频繁调用 `Get<T>()`。泛型查询更适合启动组装、调试工具和少量动态场景。

避免让业务代码到处持有全局的 `ModuleManager.Instance`。这会让真实依赖重新隐藏起来，最终退化成 Service Locator。

---

## 5. 依赖声明与拓扑排序

### 5.1 显式依赖

模块依赖应由类型表达：

```csharp
public sealed class LoginModule : ModuleBase
{
    private static readonly IReadOnlyCollection<Type> ModuleDependencies =
        new[]
        {
            typeof(ConfigModule),
            typeof(NetworkModule)
        };

    public override IReadOnlyCollection<Type> Dependencies =>
        ModuleDependencies;

    protected override void OnInit(ModuleContext context)
    {
    }

    protected override void OnStart()
    {
    }

    protected override void OnShutdown()
    {
    }
}
```

依赖关系用于决定生命周期顺序，而不是用于自动获取实例。模块实例仍然可以通过构造函数注入：

```csharp
public LoginModule(
    ConfigModule configModule,
    NetworkModule networkModule)
{
    _configModule = configModule;
    _networkModule = networkModule;
}
```

这种设计同时保留了两个信息：

```text
构造函数依赖：对象创建时需要什么
生命周期依赖：启动顺序上必须先准备什么
```

多数情况下二者相同，但不应该默认完全等价。例如一个模块可能持有日志服务，却不需要把日志服务纳入当前作用域的生命周期排序。

### 5.2 依赖校验

排序前应先检查缺失依赖：

```csharp
private void ValidateDependencies()
{
    foreach (var module in _modules.Values)
    {
        foreach (var dependency in module.Dependencies)
        {
            if (!_modules.ContainsKey(dependency))
            {
                throw new InvalidOperationException(
                    $"{module.ModuleType.Name} depends on " +
                    $"{dependency.Name}, but it is not registered.");
            }
        }
    }
}
```

还应禁止模块依赖自身：

```csharp
if (dependency == module.ModuleType)
{
    throw new InvalidOperationException(
        $"{module.ModuleType.Name} cannot depend on itself.");
}
```

### 5.3 拓扑排序

可以使用深度优先搜索完成排序，并同时检测循环依赖：

```csharp
private enum VisitState
{
    None,
    Visiting,
    Visited
}

private List<IModule> SortModules()
{
    var result = new List<IModule>(_modules.Count);
    var states = new Dictionary<Type, VisitState>();

    foreach (var moduleType in _modules.Keys)
        states[moduleType] = VisitState.None;

    foreach (var module in _modules.Values)
        Visit(module, states, result);

    return result;
}

private void Visit(
    IModule module,
    Dictionary<Type, VisitState> states,
    List<IModule> result)
{
    var type = module.ModuleType;
    var state = states[type];

    if (state == VisitState.Visited)
        return;

    if (state == VisitState.Visiting)
    {
        throw new InvalidOperationException(
            $"Circular module dependency detected at {type.Name}.");
    }

    states[type] = VisitState.Visiting;

    foreach (var dependencyType in module.Dependencies)
    {
        var dependency = _modules[dependencyType];
        Visit(dependency, states, result);
    }

    states[type] = VisitState.Visited;
    result.Add(module);
}
```

排序结果满足：

```text
被依赖模块总是在依赖模块之前
```

关闭时直接反向遍历排序结果即可。

### 5.4 稳定排序

多个模块之间没有依赖关系时，字典遍历顺序不应被当成业务约定。为了保证日志、测试和不同平台上的顺序稳定，可以增加次级排序规则，例如注册序号或模块名。

推荐保存注册序号：

```csharp
private readonly Dictionary<Type, int> _registrationOrder = new();
```

拓扑排序遍历模块和依赖时，根据注册序号排序。这样依赖关系决定主顺序，注册顺序决定同级顺序。

---

## 6. 生命周期管理器实现

### 6.1 初始化

初始化应按拓扑排序结果依次执行，并记录已经成功完成的模块：

```csharp
public void Init(ModuleContext context)
{
    EnsureState(ModuleManagerState.Created);

    State = ModuleManagerState.Initializing;

    ValidateDependencies();
    _sortedModules = SortModules();

    var initializedModules = new List<IModule>();

    try
    {
        foreach (var module in _sortedModules)
        {
            context.Logger.Info(
                $"Init started: {module.ModuleType.Name}");

            module.Init(context);
            initializedModules.Add(module);

            context.Logger.Info(
                $"Init completed: {module.ModuleType.Name}");
        }

        State = ModuleManagerState.Initialized;
    }
    catch
    {
        RollbackInitializedModules(initializedModules, context);
        State = ModuleManagerState.Faulted;
        throw;
    }
}
```

初始化失败时，必须只回滚已经成功完成初始化的模块。未完成初始化的模块不能假设具备完整关闭条件。

### 6.2 启动

所有模块完成 `Init` 后再进入 `Start`：

```csharp
public void Start(ModuleContext context)
{
    EnsureState(ModuleManagerState.Initialized);

    State = ModuleManagerState.Starting;

    var startedModules = new List<IModule>();

    try
    {
        foreach (var module in _sortedModules)
        {
            context.Logger.Info(
                $"Start started: {module.ModuleType.Name}");

            module.Start();
            startedModules.Add(module);

            context.Logger.Info(
                $"Start completed: {module.ModuleType.Name}");
        }

        BuildTickLists();
        State = ModuleManagerState.Running;
    }
    catch
    {
        ShutdownModulesReverse(startedModules, context);
        ShutdownInitializedButNotStartedModules(
            startedModules.Count,
            context);

        State = ModuleManagerState.Faulted;
        throw;
    }
}
```

启动失败时有两种处理策略。

第一种策略是直接反向关闭所有已初始化模块，逻辑简单，也更常用。

第二种策略是区分“已启动模块”和“只初始化模块”，分别执行停止和释放。如果接口只有一个 `Shutdown`，两种状态最终都调用同一个方法即可。

### 6.3 Tick

管理器只缓存实现对应接口的模块：

```csharp
private readonly List<IModuleTick> _tickModules = new();
private readonly List<IModuleFixedTick> _fixedTickModules = new();
private readonly List<IModuleLateTick> _lateTickModules = new();

private void BuildTickLists()
{
    _tickModules.Clear();
    _fixedTickModules.Clear();
    _lateTickModules.Clear();

    foreach (var module in _sortedModules)
    {
        if (module is IModuleTick tick)
            _tickModules.Add(tick);

        if (module is IModuleFixedTick fixedTick)
            _fixedTickModules.Add(fixedTick);

        if (module is IModuleLateTick lateTick)
            _lateTickModules.Add(lateTick);
    }
}
```

每帧不应使用 LINQ、反射或重新判断模块类型。启动阶段一次性构建列表即可。

```csharp
public void Tick(float deltaTime)
{
    if (State != ModuleManagerState.Running)
        return;

    for (var i = 0; i < _tickModules.Count; i++)
        _tickModules[i].Tick(deltaTime);
}
```

`FixedTick` 和 `LateTick` 使用相同方式实现。

### 6.4 关闭

关闭必须反向执行：

```csharp
public void Shutdown(ModuleContext context)
{
    if (State is ModuleManagerState.Created or
        ModuleManagerState.Shutdown or
        ModuleManagerState.ShuttingDown)
    {
        return;
    }

    State = ModuleManagerState.ShuttingDown;

    for (var i = _sortedModules.Count - 1; i >= 0; i--)
    {
        var module = _sortedModules[i];

        try
        {
            context.Logger.Info(
                $"Shutdown started: {module.ModuleType.Name}");

            module.Shutdown();

            context.Logger.Info(
                $"Shutdown completed: {module.ModuleType.Name}");
        }
        catch (Exception exception)
        {
            context.Logger.Error(
                $"Shutdown failed: {module.ModuleType.Name}",
                exception);
        }
    }

    ClearRuntimeLists();
    State = ModuleManagerState.Shutdown;
}
```

单个模块关闭失败后，不应停止后续清理。管理器应记录异常并继续关闭其他模块。

如果需要让上层知道关闭过程中发生过异常，可以收集异常并在结束后抛出 `AggregateException`，或者返回包含错误列表的关闭结果。

---

## 7. 异步生命周期

### 7.1 异步接口

包含远端配置、SDK、Addressables、存档和网络连接的项目通常需要异步接口：

```csharp
public interface IAsyncModule
{
    Type ModuleType { get; }

    IReadOnlyCollection<Type> Dependencies { get; }

    Task InitAsync(
        ModuleContext context,
        CancellationToken cancellationToken);

    Task StartAsync(CancellationToken cancellationToken);

    Task ShutdownAsync(CancellationToken cancellationToken);
}
```

生命周期方法不要使用 `async void`。返回 `Task` 才能被等待、取消和捕获异常。

### 7.2 取消

管理器应为一次完整生命周期创建独立的 `CancellationTokenSource`：

```csharp
private CancellationTokenSource _lifecycleCts;

public async Task InitAsync(
    ModuleContext context,
    CancellationToken externalToken)
{
    _lifecycleCts = CancellationTokenSource
        .CreateLinkedTokenSource(
            externalToken,
            context.ApplicationToken);

    var token = _lifecycleCts.Token;

    // 执行初始化
}
```

关闭前先取消生命周期令牌：

```csharp
_lifecycleCts?.Cancel();
```

模块必须把令牌继续传给内部异步调用。只在接口中接收令牌但不向下传递，没有实际取消效果。

### 7.3 超时

第三方 SDK 和网络操作可能永久不返回。可以为每个阶段设置超时：

```csharp
public static async Task WithTimeout(
    Task task,
    TimeSpan timeout,
    CancellationToken cancellationToken)
{
    using var timeoutCts =
        CancellationTokenSource.CreateLinkedTokenSource(
            cancellationToken);

    var delayTask = Task.Delay(timeout, timeoutCts.Token);
    var completedTask = await Task.WhenAny(task, delayTask);

    if (completedTask == delayTask)
        throw new TimeoutException();

    timeoutCts.Cancel();
    await task;
}
```

超时后不能只抛异常，还应确保对应模块能进入可关闭状态。第三方 SDK 如果不支持取消，需要在适配层中增加状态隔离，避免迟到回调重新修改已经关闭的模块。

### 7.4 并行初始化

没有依赖关系的模块理论上可以并行初始化，但不建议一开始就直接对所有模块调用 `Task.WhenAll`。正确的并行方式是先按依赖关系分层：

```text
第一层：Config、Logging
第二层：Resource、Network、Analytics
第三层：Account、Save
第四层：Lobby
```

同一层可以并行，下一层必须等待上一层完成。

并行初始化会增加以下复杂度：

- 日志顺序不再固定。
- 多个异常需要聚合。
- 回滚顺序需要重新定义。
- 模块内部可能争用 Unity 主线程。
- Unity API 大多只能在主线程调用。

只有启动耗时已经成为明确问题，并且已有完整日志和测试时，再引入分层并行。

---

## 8. 失败处理与回滚

### 8.1 初始化失败

初始化失败后应反向关闭已经成功初始化的模块：

```csharp
private void RollbackInitializedModules(
    IReadOnlyList<IModule> initializedModules,
    ModuleContext context)
{
    for (var i = initializedModules.Count - 1; i >= 0; i--)
    {
        try
        {
            initializedModules[i].Shutdown();
        }
        catch (Exception exception)
        {
            context.Logger.Error(
                $"Rollback failed: " +
                initializedModules[i].ModuleType.Name,
                exception);
        }
    }
}
```

回滚异常不应覆盖原始初始化异常。原始异常是导致启动失败的主要原因，回滚异常应作为附加信息记录。

### 8.2 Start 失败

`Start` 失败通常意味着模块已经完成资源准备，但正式业务启动失败。最稳妥的处理是关闭整个作用域，而不是尝试让部分模块继续运行。

例如登录模块启动失败后，网络和配置模块虽然仍可运行，但当前 Session Scope 已经不再具备完整业务能力。继续保留部分模块会让管理器状态和业务状态不一致。

### 8.3 Tick 异常

Tick 异常有三种常见策略：

```text
开发环境：直接抛出，尽早暴露问题
测试环境：记录模块、帧号和调用顺序后抛出
线上环境：记录并上报，按模块重要性决定是否隔离
```

不要默认吞掉所有 Tick 异常。一个关键模块停止更新后，客户端可能继续运行但状态已经错误，这类问题比直接崩溃更难定位。

可以为非关键模块增加隔离策略，但必须明确模块是否允许降级。

---

## 9. 作用域设计

### 9.1 Application Scope

Application Scope 与客户端进程生命周期一致，通常包含：

- 日志。
- 配置。
- 资源系统。
- 音频。
- 本地化。
- 埋点。
- 崩溃上报。
- 平台 SDK 基础层。

该作用域通常由 Bootstrap 场景创建，并通过 `DontDestroyOnLoad` 保持。

### 9.2 Session Scope

Session Scope 从进入账号会话开始，到退出账号结束，通常包含：

- 网络连接。
- 登录状态。
- 玩家数据。
- 好友和聊天。
- 背包和任务。
- 支付会话。
- 服务器时间同步。

退出登录时应完整关闭 Session Scope，然后重新创建新的 Session Scope。不要尝试手动重置几十个全局单例字段。

### 9.3 Scene Scope

Scene Scope 与大厅、战斗或关卡流程绑定，通常包含：

- 场景状态机。
- 战斗规则。
- 地图数据。
- 场景事件。
- 场景 UI 流程。
- 当前关卡资源引用。

场景卸载前应先关闭 Scene Scope，再释放场景资源。否则模块可能在资源已释放后仍然响应事件或执行 Tick。

### 9.4 作用域关系

推荐结构：

```text
ApplicationModuleManager
    └── SessionModuleManager
            └── SceneModuleManager
```

父作用域可以创建和持有子作用域，但子作用域不能反向控制父作用域生命周期。

关闭顺序为：

```text
Scene Scope
    ↓
Session Scope
    ↓
Application Scope
```

---

## 10. Unity 接入

### 10.1 ModuleHost

Unity 侧只保留一个薄适配层：

```csharp
using UnityEngine;

public sealed class ModuleHost : MonoBehaviour
{
    private ModuleLifecycleManager _manager;
    private ModuleContext _context;
    private bool _isQuitting;

    private void Awake()
    {
        DontDestroyOnLoad(gameObject);

        _context = CreateContext();
        _manager = CreateManager();

        _manager.Init(_context);
    }

    private void Start()
    {
        _manager.Start(_context);
    }

    private void Update()
    {
        _manager.Tick(Time.deltaTime);
    }

    private void FixedUpdate()
    {
        _manager.FixedTick(Time.fixedDeltaTime);
    }

    private void LateUpdate()
    {
        _manager.LateTick(Time.deltaTime);
    }

    private void OnApplicationQuit()
    {
        _isQuitting = true;
        _manager?.Shutdown(_context);
    }

    private void OnDestroy()
    {
        if (!_isQuitting)
            _manager?.Shutdown(_context);
    }
}
```

`ModuleHost` 不应包含具体业务逻辑。它只负责：

- 创建上下文。
- 组装模块。
- 转发 Unity 生命周期。
- 在应用退出时执行兜底关闭。

### 10.2 主线程约束

Unity API 通常只能在主线程调用。异步模块如果在后台线程完成 IO，回到 Unity API 前必须切回主线程。

生命周期管理器本身最好保持在主线程调度。模块内部可以将纯计算或文件 IO 放到后台线程，但不能直接在后台线程创建 GameObject、加载 UnityEngine.Object 或修改场景对象。

### 10.3 场景切换

场景切换建议由显式流程控制：

```text
停止 Scene Scope 接收新请求
取消 Scene Scope 异步任务
执行 Scene Scope Shutdown
卸载旧场景
加载新场景
创建新的 Scene Scope
执行 Init 和 Start
```

不要只依赖 `OnDestroy` 被动关闭场景模块。显式流程能保证资源释放和模块关闭顺序可控。

---

## 11. 事件与资源清理

### 11.1 事件订阅

事件订阅应返回 `IDisposable`：

```csharp
private readonly CompositeDisposable _disposables =
    new();

protected override void OnStart()
{
    _eventBus
        .Subscribe<LoginSuccessEvent>(OnLoginSuccess)
        .AddTo(_disposables);
}

protected override void OnShutdown()
{
    _disposables.Dispose();
}
```

如果没有现成的 `CompositeDisposable`，可以实现简单集合：

```csharp
public sealed class DisposableCollection : IDisposable
{
    private readonly List<IDisposable> _items = new();
    private bool _disposed;

    public void Add(IDisposable item)
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(DisposableCollection));

        _items.Add(item);
    }

    public void Dispose()
    {
        if (_disposed)
            return;

        _disposed = true;

        for (var i = _items.Count - 1; i >= 0; i--)
            _items[i].Dispose();

        _items.Clear();
    }
}
```

### 11.2 异步任务

模块应保存自己的 `CancellationTokenSource`，关闭时先取消，再等待关键任务结束：

```csharp
private CancellationTokenSource _cts;
private Task _receiveLoopTask;

protected override void OnStart()
{
    _cts = new CancellationTokenSource();
    _receiveLoopTask = ReceiveLoopAsync(_cts.Token);
}

protected override void OnShutdown()
{
    _cts?.Cancel();
    _cts?.Dispose();
    _cts = null;
}
```

同步 `Shutdown` 无法安全等待异步任务时，应升级为 `ShutdownAsync`，不要用阻塞式 `.Wait()` 或 `.Result`，否则可能导致主线程死锁。

### 11.3 Unity 资源

模块持有 Addressables 句柄、AssetBundle、NativeArray、文件流、Socket 或第三方 SDK 句柄时，应明确所有权。

只有拥有资源所有权的模块负责释放。共享资源应由更高层资源模块统一管理，业务模块只释放引用或租约，避免重复释放。

---

## 12. Update 调度策略

### 12.1 不同频率

不是所有持续逻辑都需要每帧执行。可以提供低频调度：

```csharp
public interface IModuleScheduledTick
{
    float Interval { get; }

    void ScheduledTick(float deltaTime);
}
```

管理器保存累计时间：

```csharp
private sealed class ScheduledEntry
{
    public IModuleScheduledTick Module;
    public float Elapsed;
}
```

适合低频调度的逻辑包括：

- 心跳。
- 超时扫描。
- 缓存清理。
- 统计刷新。
- 弱实时状态检查。

### 12.2 Tick 顺序

Tick 顺序可以沿用模块拓扑顺序。这样被依赖模块先更新，依赖模块后更新。

不过初始化依赖不一定等于更新依赖。如果项目存在明确不同的更新顺序需求，可以单独增加 Tick 依赖或 Tick 优先级，但不要默认引入第二套复杂依赖图。

### 12.3 性能统计

开发版本中可以记录模块耗时：

```csharp
var startTimestamp = Stopwatch.GetTimestamp();

module.Tick(deltaTime);

var elapsed = Stopwatch.GetElapsedTime(startTimestamp);
```

建议记录：

```text
模块名
生命周期阶段
单次耗时
最大耗时
平均耗时
调用次数
异常次数
```

性能统计应支持关闭，避免发布版本中产生不必要开销。

---

## 13. 日志与调试信息

### 13.1 生命周期日志

管理器应统一输出：

```text
[Module] Init started: ConfigModule
[Module] Init completed: ConfigModule, 12 ms
[Module] Start started: NetworkModule
[Module] Start failed: NetworkModule, TimeoutException
[Module] Rollback completed: ConfigModule
```

日志中至少包含：

- 作用域名称。
- 模块名称。
- 生命周期阶段。
- 开始时间。
- 结束时间。
- 耗时。
- 当前状态。
- 异常类型和堆栈。

### 13.2 调试面板

可以提供运行时调试面板，显示：

```text
当前管理器状态
模块排序结果
每个模块状态
依赖列表
最近一次生命周期耗时
最近一次异常
Tick 耗时
作用域创建和关闭时间
```

调试面板不应直接修改模块内部状态。必要的调试操作应通过管理器提供的受控接口完成。

---

## 14. 测试设计

### 14.1 排序测试

至少测试：

- 无依赖模块顺序稳定。
- 单链依赖排序正确。
- 多分支依赖排序正确。
- 缺少依赖时失败。
- 自依赖时失败。
- 循环依赖时失败。
- 重复注册时失败。

可以使用记录模块：

```csharp
public sealed class RecordingModule : IModule
{
    private readonly List<string> _records;

    public Type ModuleType { get; }

    public IReadOnlyCollection<Type> Dependencies { get; }

    public RecordingModule(
        Type moduleType,
        IReadOnlyCollection<Type> dependencies,
        List<string> records)
    {
        ModuleType = moduleType;
        Dependencies = dependencies;
        _records = records;
    }

    public void Init(ModuleContext context)
    {
        _records.Add($"Init:{ModuleType.Name}");
    }

    public void Start()
    {
        _records.Add($"Start:{ModuleType.Name}");
    }

    public void Shutdown()
    {
        _records.Add($"Shutdown:{ModuleType.Name}");
    }
}
```

### 14.2 回滚测试

应验证：

```text
A.Init 成功
B.Init 成功
C.Init 抛异常
```

最终记录必须是：

```text
Init:A
Init:B
Init:C
Shutdown:B
Shutdown:A
```

同时管理器状态应为 `Faulted`。

### 14.3 状态测试

应验证：

- 未初始化不能 Start。
- 非 Running 状态不执行 Tick。
- Shutdown 重复调用不会重复释放。
- Faulted 状态不能重新 Start。
- 初始化后不能继续注册模块。
- 单个 Shutdown 抛异常时，其他模块仍继续关闭。

### 14.4 异步测试

异步版本还应验证：

- 取消令牌能中断初始化。
- 超时会进入失败流程。
- 迟到回调不会重新修改已关闭模块。
- 多个并行异常能被完整记录。
- Shutdown 会取消仍在运行的任务。

---
title: 游戏编程模式
date: 2026-06-18 00:00:00 +0800
categories: [GameDev, Gameplay]
tags: [game-programming-patterns, design-patterns, command-pattern, flyweight-pattern, observer-pattern, prototype-pattern, singleton-pattern, state-pattern, game-loop, unity]
description: 梳理《游戏编程模式》中常见设计模式与序列型模式，包括命令模式、享元模式、观察者模式、原型模式、单例模式、状态模式、游戏循环、更新方法和双缓冲，并结合 Unity 与游戏客户端开发场景理解其应用。
media_subpath: /assets/img/posts/2026-06-18-game-programming-patterns
---

# 游戏编程模式

## 命令模式（Command Pattern）：

 把一次“操作请求”封装成一个命令对象，由调用方创建命令，由执行方统一执行。

它的核心作用是：

> 将“发起操作”和“执行操作”解耦。

在游戏客户端中，命令模式常用于：

- 玩家输入

  ```
  Command* command = inputHandler.handleInput();
  if (command)    
  {      
      command->execute(actor);    
  }
  ```

  将玩家输入与角色操作解耦，输入系统不需要知道角色具体怎么移动、攻击；角色系统也不需要关心输入来自键盘、手柄还是 AI。

- 技能释放

- 战斗指令

- 回放系统

- 撤销/重做

  ```
      function makeMoveUnitCommand(unit, x, y) {
        var xBefore, yBefore;
        return {
          execute: function() {
            xBefore = unit.x();
            yBefore = unit.y();
            unit.moveTo(x, y);
          },
          undo: function() {
            unit.moveTo(xBefore, yBefore);
          }
        };
      }
  ```

- 网络同步

  ![命令模式中的命令流](command-flow.png)

  命令模式把玩家输入、AI 行为、网络消息统一抽象成命令流，再由战斗系统按顺序执行，是多人游戏输入同步和回放系统的重要基础。

------

**反射**：让程序在运行时检查和操作类、对象、字段、方法等类型信息的机制。

**闭包**：一个函数携带了它所引用的外部变量，使这些变量在函数外部仍能被继续访问和修改。

## 享元模式（Flyweight Pattern）

把大量对象中相同的部分提取出来共享，避免重复创建，节省内存。

“享元”可以理解为：**共享的最小对象单元。**

```
    class Terrain
    {
    public:
      Terrain(int movementCost, bool isWater,
             Texture texture)
      : moveCost_(moveCost),
        isWater_(isWater),
        texture_(texture)
      {}

      int getMoveCost() const { return moveCost_; }
      bool isWater() const { return isWater_; }
      const Texture& getTexture() const
      {
      return texture_;
      }

    private:
      int moveCost_;
      bool isWater_;
      Texture texture_;
    };
    
    class World
    {
    public:
      World()
      : grassTerrain_(1, false, GRASS_TEXTURE),
        hillTerrain_(3, false, HILL_TEXTURE),
        riverTerrain_(2, true, RIVER_TEXTURE)
      {}

    private:
      Terrain grassTerrain_;
      Terrain hillTerrain_;
      Terrain riverTerrain_;
      // Other stuff...
    };
```

## 观察者模式（Observer Pattern）

**定义：**
观察者模式用于在对象之间建立一种“一对多”的依赖关系。当一个对象状态发生变化时，所有依赖它的对象都会自动收到通知并更新。

**核心思想：**

> 被观察者不直接依赖具体观察者，而是通过通知机制广播事件。

**主要角色：**

- **Subject / 被观察者**：事件的发布者，负责维护观察者列表，并在状态变化时发送通知。
- **Observer / 观察者**：事件的接收者，收到通知后执行自己的逻辑。

**游戏中的常见场景：**

- 玩家血量变化 → UI 血条刷新
- 玩家死亡 → 播放死亡动画、弹出结算界面
- 背包物品变化 → 背包 UI 更新
- 任务进度变化 → 任务面板刷新
- 网络状态变化 → 提示断线或重连

**优点：**

- 降低对象之间的耦合
- 一个事件可以通知多个系统
- 方便扩展新的响应逻辑

**缺点：**

- 事件链过多时，调用关系不直观
- 需要注意注册和注销，避免内存泄漏或重复监听

**一句话总结：**观察者模式通过“订阅—通知”机制，让一个对象状态变化时自动通知多个相关对象，从而实现系统之间的解耦。

**与事件的区别**：观察者模式描述的是“一个对象变化，通知多个对象”的结构；事件通常是实现这种结构的具体方式。

**友元**就是被类特别授权的函数或类，可以访问这个类的私有成员。

## 原型模式（Prototype Pattern）

**定义：**
原型模式用于通过复制已有对象来创建新对象，而不是每次都从零开始创建。

**核心思想：**

> 先准备一个原型对象，需要新对象时直接克隆它，再修改个别属性。

**主要角色：**

- **Prototype / 原型接口**：定义克隆方法。
- **ConcretePrototype / 具体原型**：实现克隆逻辑的具体对象。
- **Client / 客户端**：通过克隆原型对象来创建新对象。

**游戏中的常见场景：**

- 怪物模板 → 生成多个怪物实例
- 子弹模板 → 发射时生成子弹
- 装备模板 → 掉落时生成具体装备
- 技能模板 → 释放时生成技能实例
- 特效模板 → 播放爆炸、治疗、命中特效
- Unity Prefab → 通过 `Instantiate` 创建对象实例

**优点：**

- 适合批量创建相似对象
- 减少重复初始化代码
- 创建复杂对象更方便
- 可以通过模板快速生成不同实例

**缺点：**

- 深拷贝实现可能比较复杂
- 浅拷贝容易导致对象共享同一份引用数据
- 对象结构复杂时，克隆逻辑不容易维护

**一句话总结：**
原型模式通过“复制已有对象”来创建新对象，适合游戏中怪物、子弹、装备、技能、特效等大量相似对象的生成。

**与对象池的区别：**
原型模式关注“如何根据模板创建对象”；对象池关注“如何复用已经创建好的对象，减少频繁创建和销毁”。

**注意点：**
游戏开发中要区分共享配置和独立状态。模型、基础属性、资源路径可以共享；当前位置、当前血量、当前目标等运行状态必须独立。

## 单例模式（Singleton Pattern）

**定义：**
单例模式用于保证一个类在程序运行期间只有一个实例，并提供一个全局访问点。

**核心思想：**

> 一个类只创建一个对象，其他地方通过同一个入口访问它。

**主要角色：**

- **Singleton / 单例类**：负责创建并保存唯一实例。
- **Instance / 全局访问点**：提供外部访问单例对象的入口。

**游戏中的常见场景：**

- 音频管理器：`AudioManager`
- 输入管理器：`InputManager`
- 配置管理器：`ConfigManager`
- 存档管理器：`SaveManager`
- 网络管理器：`NetworkManager`
- 本地化管理器：`LocalizationManager`

**优点：**

- 保证全局只有一个实例
- 访问方便
- 避免重复创建管理类对象
- 适合管理全局系统

**缺点：**

- 容易变成全局变量
- 会增加系统之间的耦合
- 隐藏对象之间的依赖关系
- 不利于测试和替换
- 生命周期管理不当容易出问题

**一句话总结：**
单例模式通过“唯一实例 + 全局访问点”来管理全局对象，适合游戏中真正唯一、职责明确、生命周期稳定的系统。

**与全局变量的区别：**
单例模式可以控制对象的创建和访问，并封装初始化逻辑；全局变量只是直接暴露数据，缺少封装和控制。

**注意点：**
单例不要为了访问方便而滥用。玩家、敌人、子弹、武器、UI 面板、房间实例等可能存在多个或需要频繁销毁重建的对象，不适合做成单例。

**游戏开发建议：**
真正全局唯一的管理器可以考虑单例；普通游戏对象和战斗实例更适合由上层管理器持有，而不是直接写成单例。

## 状态模式（State Pattern）

**定义：**
状态模式用于让一个对象在内部状态改变时，改变它的行为。对象看起来像是在不同状态下拥有不同的行为逻辑。

**核心思想：**

> 把不同状态下的行为拆分到不同状态类中，对象只保存当前状态，并把行为交给当前状态处理。

**主要角色：**

- **Context / 上下文对象**：拥有状态的对象，例如玩家、敌人、武器。
- **State / 状态接口**：定义所有状态共有的方法。
- **ConcreteState / 具体状态**：具体的状态实现，例如待机、移动、攻击、死亡。

**游戏中的常见场景：**

- 玩家状态：待机、移动、跳跃、攻击、死亡
- 敌人 AI：巡逻、追击、攻击、逃跑、死亡
- 武器状态：待机、开火、换弹、冷却
- UI 状态：打开、关闭、加载中、禁用
- 游戏流程：准备、进行中、暂停、胜利、失败

**优点：**

- 避免大量 `if else` 或 `switch`
- 每个状态的逻辑更清晰
- 状态切换更容易管理
- 方便扩展新的状态

**缺点：**

- 状态类数量可能变多
- 状态切换关系复杂时不容易追踪
- 简单逻辑使用状态模式可能会过度设计

**一句话总结：**
状态模式通过“当前状态对象”来决定对象的行为，适合处理玩家、敌人、武器等具有多种状态的游戏对象。

**与普通条件判断的区别：**
普通写法通常把所有状态逻辑写在一个类里，用 `if else` 判断；状态模式把不同状态拆成不同类，让每个状态只负责自己的行为。

**静态状态与动态实例状态：**

- **静态状态**：所有对象共享同一个状态对象，适合没有独立数据的状态，例如待机、移动、死亡。
- **动态实例状态**：每次进入状态都创建新的状态对象，适合需要保存独立数据的状态，例如换弹、蓄力、眩晕、连击。

**函数指针简化：**
如果状态类没有数据成员，并且只有一个虚函数，可以不用状态类，直接用状态函数表示当前状态。此时 `state_` 不再指向状态对象，而是指向状态函数。

**并发状态机：**
一个对象同时拥有多个状态机，适合避免状态组合爆炸。

例如 FPS 玩家可以同时拥有：

- 移动状态：待机、奔跑、跳跃
- 武器状态：待机、开火、换弹
- 生命状态：存活、死亡、眩晕

**层级状态机：**
状态可以有父子关系，子状态复用父状态的公共逻辑。

例如：

- `Grounded`：在地面状态
  - `Idle`
  - `Run`
  - `Crouch`
- `Airborne`：空中状态
  - `Jump`
  - `Fall`

**下推状态机：**
状态用栈来管理，适合临时打断后返回原状态。

例如 UI 状态：

- `Gameplay`
- Push `PauseMenu`
- Push `SettingsMenu`
- Pop 后回到 `PauseMenu`
- 再 Pop 回到 `Gameplay

**总结：**
状态模式的关键是把“不同状态下的不同行为”拆分开，让对象只关心当前状态。它能让复杂状态逻辑更清晰，但也要避免在简单场景中过度设计。

## 序列型模式（Sequencing Patterns）

**定义：**
序列型模式用于组织游戏中随时间推进的执行顺序，关注输入、逻辑、物理、动画、渲染、显示之间的时序关系。

它不是单一模式，而是一组模式，主要包括：

- **游戏循环（Game Loop）**
- **更新方法（Update Method）**
- **双缓冲（Double Buffer）**

**核心思想：**

> 游戏世界看起来是连续、同步变化的，但计算机实际是顺序执行的。序列型模式的目标是管理这种顺序执行带来的时间推进、状态可见性和帧同步问题。

**问答整理：**

**Q：高 FPS 一定更好吗？**
A：不一定。高 FPS 只代表渲染帧率高，不代表帧时间稳定、输入延迟低、物理稳定或网络同步更好。对实时游戏来说，稳定的帧时间往往比单纯高平均 FPS 更重要。

**Q：浮点数误差会影响解帧或高帧率逻辑吗？**
A：会有影响，但通常不是主要原因。真正容易导致差异的是帧率变化后，Update 调用次数、计时器采样点、动画事件、物理积分步数和输入采样频率发生变化。浮点误差更多是长期累积或边界判断时的次要因素。

**Q：CPU 和 GPU 过载会出现什么现象？**
A：CPU 过载会导致逻辑、物理、输入、网络发包、FixedUpdate 补帧等变慢；GPU 过载会导致渲染排队、输入到显示延迟变大、帧时间波动。两者都会表现为掉帧、卡顿、输入延迟、对枪手感变差。

### 游戏循环（Game Loop）

**定义：**
游戏循环是游戏运行的核心控制结构，负责不断驱动输入、逻辑、物理、动画、渲染等系统运行。

典型结构：

```cpp
while (running) {
    processInput();
    update(deltaTime);
    render();
    present();
}
```

固定时间步结构：

```cpp
accumulator += frameTime;

while (accumulator >= fixedDt) {
    simulate(fixedDt);
    accumulator -= fixedDt;
}

render(accumulator / fixedDt);
```

其中：

- `simulate(fixedDt)` 使用固定时间步，适合物理和核心模拟
- `render(alpha)` 使用插值，适合表现层平滑渲染
- `accumulator` 用于处理真实帧时间和固定模拟步之间的差异

**核心要点：**

- `Update` 通常跟随渲染帧率
- `FixedUpdate` 或固定步长模拟用于物理和稳定模拟
- 渲染帧率不应该直接决定游戏规则
- 物理更新和渲染更新通常需要分离
- 长帧需要限制最大补帧，避免补帧风暴

**问答整理：**

**Q：FixedUpdate 会在某一帧不执行或执行多次吗？**
A：会。FixedUpdate 不是绑定渲染帧，而是绑定固定模拟时间。Unity 会累积真实经过的时间，够一个 fixed timestep 就执行一次，不够就不执行，积压太多就补跑多次。

**Q：如果 FixedUpdate 的理论时间点落在两个 Update 之间怎么办？**
A：它不会立刻插入执行。FixedUpdate 不是独立计时器，而是主循环中的一个阶段。理论时间点到了以后，会等到下一帧的 FixedUpdate 阶段再执行，所以 FixedUpdate 的真实调用间隔会波动。

**Q：什么是补帧风暴？**
A：某一帧过慢后，固定更新积压；下一帧为了追赶模拟时间补跑多个 FixedUpdate，导致这一帧更慢，随后继续积压，形成恶性循环。

**Q：游戏解帧开放帧率的底层原理是什么？**
A：本质是移除或修改原本限制渲染循环频率的机制，例如帧率上限、等待逻辑、VSync 同步或引擎内部的 target frame rate。它主要改变渲染帧率和 Update 调用频率，不一定改变固定模拟频率。

**Q：角色解帧后普通攻击变快的原理是？**
A：通常不是攻击数值被改，而是普攻流程中的输入缓冲、连段检测、动画事件、取消窗口、hitlag 或后摇逻辑受帧率影响。高 FPS 让这些边界更频繁被检测，可能更早进入下一段攻击。

### 更新方法（Update Method）

**定义：**
更新方法让对象或系统在每一帧执行自己的更新逻辑。游戏世界统一调度对象，对象在自己的 `update()` 中处理行为。

典型结构：

```cpp
for (auto& object : objects) {
    object.update(dt);
}
```

**核心思想：**

> 对象只暴露更新入口，主循环统一控制执行时机。

**重难点：**

- 更新是顺序执行的，不是真正同时发生
- 更新顺序会影响结果
- 对象之间可能读取到旧状态或半更新状态
- 遍历时增删对象容易出错
- 每个对象都无脑 Update 会带来性能浪费

### 双缓冲（Double Buffer）

**定义：**
双缓冲通过维护两份数据，一份用于读取，一份用于写入，写入完成后交换，避免外部看到中间状态。

基本结构：

```text
front buffer：当前读取 / 显示
back buffer：当前写入 / 构建
swap：写完后交换
```

伪代码：

```cpp
write(backBuffer);
swap(frontBuffer, backBuffer);
read(frontBuffer);
```

**核心思想：**

> 用状态快照隐藏顺序写入过程，使结果看起来像同时完成。

**常见场景：**

- 图像渲染
- 多线程读写数据
- 当前帧 / 下一帧状态切换

**重难点：**

- 避免读到半更新状态
- 避免更新顺序影响同一帧结果
- 适合“读旧状态，写新状态”的场景
- 需要考虑内存占用、复制成本和同步成本
- 在多线程或渲染管线中，需要保证交换操作的时机安全

## 行为型模式

### 字节码模式（Bytecode Pattern）

**定义：** 字节码模式用于把游戏行为编译成一组简单指令，再交给虚拟机解释执行。它的核心作用是：**将行为描述和底层执行逻辑解耦**。

```cpp
enum OpCode {
    OP_PUSH_CONST,
    OP_GET_TARGET_HP,
    OP_DAMAGE,
    OP_PLAY_EFFECT,
    OP_JUMP_IF_FALSE,
    OP_END
};

class VM {
public:
    void run(const std::vector<int>& code) {
        int ip = 0;
        while (true) {
            OpCode op = static_cast<OpCode>(code[ip++]);
            switch (op) {
            case OP_PUSH_CONST:
                stack_.push(code[ip++]);
                break;
            case OP_DAMAGE: {
                int value = stack_.top();
                stack_.pop();
                dealDamage(value);
                break;
            }
            case OP_PLAY_EFFECT:
                playEffect(code[ip++]);
                break;
            case OP_END:
                return;
            }
        }
    }
private:
    std::stack<int> stack_;
};
```

字节码模式的重点不是 `switch` 怎么写，而是**指令集怎么设计**。指令太细，例如 `PUSH`、`ADD`、`MUL`，通用性强，但执行步数多、调试困难；指令太粗，例如 `CAST_FIREBALL_TO_NEAREST_ENEMY`，执行高效、策划容易理解，但扩展性差。

- **栈式虚拟机**：指令格式简单，编译器容易生成，适合技能、任务、剧情这类中低频逻辑
- **寄存器式虚拟机**：VM 指令数量更少，数据流更清晰，性能更好，但编译器和指令格式更复杂。

在游戏中使用字节码时，最重要的是边界控制。VM 不应该暴露任意原生函数调用，否则等于让外部逻辑绕过了引擎安全层。

**一句话总结：** 字节码模式通过“指令集 + 虚拟机”把游戏行为变成可控的数据，适合技能、AI、任务、剧情等需要热更新、沙盒化和工具化的系统。它的难点不在解释执行，而在指令边界、调试工具、确定性、性能边界和版本兼容。

### 子类沙盒模式（Subclass Sandbox Pattern）

**定义：** 子类沙盒模式让基类提供一组受保护的操作接口，子类只能在这些接口范围内组合自己的行为。它的核心作用是：**允许大量子类自定义行为，同时把危险、复杂、重复的底层细节封装在基类中**。可以理解为，子类可以写特殊逻辑，但只能在父类划好的“沙盒 API”里写。

```cpp
class Skill {
public:
    void cast() {
        checkCost();
        playCastAnimation();
        onCast();
        startCooldown();
    }

protected:
    virtual void onCast() = 0;

    Actor* findNearestEnemy();
    void dealDamage(Actor* target, int value);
    void addBuff(Actor* target, int buffId);
    void playEffect(int effectId);

private:
    void checkCost();
    void playCastAnimation();
    void startCooldown();
};

class FireballSkill : public Skill {
protected:
    void onCast() override {
        Actor* target = findNearestEnemy();
        dealDamage(target, 100);
        addBuff(target, BUFF_BURNING);
        playEffect(EFFECT_FIREBALL);
    }
};
```

**子类沙盒和组件模式的区别**：子类沙盒偏继承，适合少量维度上的特殊行为；组件模式偏组合，适合能力排列组合非常多的对象。

**优点：** 子类代码简洁，公共逻辑复用度高，可以限制子类访问范围，并统一处理日志、资源、网络、冷却和异常。

**缺点：** 基类容易膨胀，子类数量可能爆炸，所有子类都会依赖同一个父类，后期维护成本可能变高。

**一句话总结：** 子类沙盒模式通过“父类提供受控能力，子类组合这些能力”来实现可扩展行为，适合技能、怪物、道具等大量相似但细节不同的对象。关键是沙盒 API 要克制，不能把父类做成万能入口。

### 类型对象模式（Type Object Pattern）

**定义：** 类型对象模式用于把“类型”从类继承层次中抽离出来，变成运行时可创建、可配置、可引用的数据对象。它的核心作用是：**用对象表示类型，而不是用类表示类型**。

```cpp
class MonsterType {
public:
    std::string name;
    int maxHp;
    int attack;
    float moveSpeed;
    std::vector<int> skillIds;
};

class Monster {
public:
    Monster(MonsterType* type)
        : type_(type), currentHp_(type->maxHp) {}

    int attack() const {
        return type_->attack;
    }

private:
    MonsterType* type_;
    int currentHp_;
};
```

类型对象最重要的边界是：**类型对象保存共享配置，实例对象保存运行时状态**。例如 `MonsterType` 可以保存最大生命、基础攻击、移动速度、模型路径、技能列表和掉落表；`Monster` 实例保存当前生命、当前位置、当前目标、AI 状态、技能冷却和身上的 Buff。

**类型对象和享元模式的区别**：但关注点不同。享元模式关注节省内存，把重复数据共享出去；类型对象模式关注建模，把“类型”本身从代码类变成数据对象。

**类型对象和原型模式的区别**：原型模式关注通过复制已有对象创建新对象，类型对象关注实例引用一份共享类型配置。一个是“复制模板”，一个是“引用类型”。

类型对象也可以有继承关系，但要小心继承层次过深、字段覆盖规则不清晰、循环继承、热更新依赖刷新等问题。

**优点：** 减少子类数量，适合数据驱动和热更新，新内容可以通过配置增加，实例共享类型数据也能节省内存。

**缺点：** 类型安全下降，错误更晚暴露，配置校验成本变高；如果行为差异很大，仍然需要代码、脚本或组件系统配合。

**一句话总结：** 类型对象模式通过“用数据对象表示类型”替代大量子类，适合怪物、装备、技能、Buff 等内容量大、参数变化频繁的系统。关键是严格区分共享配置和实例状态。

## 解耦型模式

解耦不是让模块之间完全没有依赖，而是让依赖方向清晰、变化范围可控，并通过稳定接口或数据契约协作。

解耦型模式通常会增加间接调用。接口、消息和组件都能降低直接依赖，但也可能隐藏调用关系。因此在客户端工程中，除了关注类图，还要关注数据所有权、对象生命周期、执行顺序和问题定位能力。

### 组件模式（Component Pattern）

**定义：** 组件模式把复杂对象的不同能力拆分为多个相对独立的组件，再通过组合组件构建对象。它的核心作用是：**用组合代替不断膨胀的继承层次，让能力能够独立复用和替换。**

```cpp
class Component {
public:
    virtual ~Component() = default;
    virtual void onAttach() {}
    virtual void onDetach() {}
};

class GameObject {
public:
    template<class T, class... Args>
    T& addComponent(Args&&... args);

    template<class T>
    T* getComponent();

private:
    std::vector<std::unique_ptr<Component>> components_;
};
```

组件模式真正困难的地方不是把 `Player` 拆成移动、战斗和动画组件，而是拆分后如何管理这些组件之间的关系。

**数据所有权：** 同一份运行时状态应有明确的唯一所有者。例如角色速度可以由移动组件持有，动画组件只读取速度设置动画参数，不应让两个组件分别保存一份“当前速度”。否则组件执行顺序不同，就可能读到旧值或互相覆盖状态。

```cpp
class MovementComponent {
public:
    const Vec3& velocity() const { return velocity_; }

private:
    Vec3 velocity_;
};

class AnimationComponent {
public:
    explicit AnimationComponent(MovementComponent& movement)
        : movement_(movement) {}

    void update() {
        animator_.setFloat("Speed", movement_.velocity().length());
    }

private:
    MovementComponent& movement_;
    Animator animator_;
};
```

组件之间并非不能依赖，关键是让依赖显式、稳定并且可以在装配阶段校验。

常见通信方式包括：

- 构造函数或初始化函数注入，适合必须存在的强依赖。
- 通过容器查询其他组件，使用方便，但依赖容易被隐藏。
- 通过事件发送通知，适合一对多广播，但不适合需要立即返回结果的调用。
- 通过共享上下文或黑板交换少量公共数据，但要避免退化成万能数据容器。

```cpp
class AttackComponent : public Component {
public:
    bool initialize(GameObject& owner) {
        attributes_ = owner.getComponent<AttributeComponent>();
        animation_ = owner.getComponent<AnimationComponent>();
        return attributes_ != nullptr && animation_ != nullptr;
    }

private:
    AttributeComponent* attributes_{};
    AnimationComponent* animation_{};
};
```

对于攻击组件来说，属性组件和动画组件是明确依赖。与其在释放技能时才出现空指针，不如在对象装配完成后统一检查依赖是否合法。

**生命周期：** 组件至少要区分创建、挂载、初始化、启用、禁用、重置和销毁。尤其在对象池、场景切换和异步资源加载中，构造函数执行完成并不代表组件已经可以正常工作。

```text
Construct → Attach → Initialize → Enable → Update
          → Disable → Reset → Detach → Destroy
```

对象池中的组件通常不会真正销毁，因此 `Reset` 不能只重置位置和血量，还要清理事件订阅、计时器、协程、目标引用、动画状态和异步回调。

**更新调度：** 不应默认所有组件都需要每帧执行 `Update`。不同能力适合不同更新方式：

- 输入、相机和表现逻辑通常按渲染帧更新。
- 物理和确定性模拟通常使用固定时间步。
- 远距离 AI、环境检测可以降低更新频率。
- 血量 UI、任务进度和冷却完成更适合事件驱动。

当组件数量增加后，可以由调度器按照阶段运行同类组件，而不是让每个对象自行遍历全部组件。

```text
Input → Gameplay → Physics → Animation → Camera → RenderPrepare
```

这种做法能够让更新顺序显式化，也方便后续进行批量处理和多线程拆分。

**组件粒度：** 组件过粗，只是把一个大类拆成几个较小的大类；组件过细，则会产生大量跨组件查询、消息发送和生命周期管理。一个能力是否值得拆成组件，可以判断它是否拥有相对独立的数据、能否被多个对象复用、是否需要独立启停，以及它与其他能力之间是否存在稳定接口。

**与 ECS 的区别：** 传统组件模式通常仍以 `GameObject` 为中心，组件中可以同时保存数据和行为。ECS 更强调组件保存数据，系统批量处理同类数据，并利用连续存储改善缓存局部性。Unity 的 `MonoBehaviour` 更接近传统组件模型，DOTS/ECS 才更接近数据导向的 ECS。

**优点：** 减少继承层级，能力可以组合和复用，方便编辑器装配，也便于按能力启用或禁用。

**缺点：** 组件依赖和执行顺序容易被隐藏；数据所有权、生命周期和通信方式会成为新的复杂点；大量小对象、虚调用和随机内存访问也可能影响性能。

**一句话总结：** 组件模式通过“按能力组合对象”代替复杂继承。真正需要设计的不是组件数量，而是数据所有权、依赖声明、生命周期和更新调度。

### 事件队列模式（Event Queue Pattern）

**定义：** 事件队列模式把事件的产生和处理在时间上解耦。生产者只负责将事件写入队列，消费者在合适的阶段统一读取并处理。它的核心作用是：**让发送者不必立即调用接收者，也不必知道事件最终何时被处理。**

```cpp
struct PlaySoundEvent {
    SoundId sound;
    Vec3 position;
    float volume;
};

class AudioEventQueue {
public:
    bool push(const PlaySoundEvent& event) {
        if (events_.size() >= kMaxEvents) return false;
        events_.push_back(event);
        return true;
    }

    void flush(AudioSystem& audio) {
        for (const auto& event : events_) {
            audio.play(event.sound, event.position, event.volume);
        }
        events_.clear();
    }

private:
    static constexpr size_t kMaxEvents = 1024;
    std::vector<PlaySoundEvent> events_;
};
```

**与观察者模式的区别：** 观察者模式主要解耦“由谁响应”，通知通常在当前调用栈中同步发生；事件队列进一步解耦“何时响应”。

```text
观察者：发布事件 → 遍历订阅者 → 立即执行回调
事件队列：发布事件 → 写入队列 → 指定阶段统一消费
```

观察者适合局部系统中的即时通知，例如属性变化后立即刷新依赖状态。事件队列适合跨阶段、跨线程、需要削峰或需要保证处理顺序的消息，例如音频请求、网络消息和资源加载结果。

**事件数据：** 事件最好保存值语义的数据快照，或者保存能够重新校验有效性的稳定 ID，而不是直接携带可能失效的裸指针。

```cpp
struct DamageEvent {
    EntityId attacker;
    EntityId target;
    int damage;
    uint32_t frame;
};
```

如果事件保存 `Actor* target`，等到事件被消费时，目标对象可能已经销毁，或者对应内存已经被对象池中的另一个实例复用。使用实体 ID 或带代数的句柄后，消费者可以在处理时重新检查对象是否仍然有效。

**顺序语义：** 引入事件队列后，需要明确以下规则：

- 同一帧内是否严格按照入队顺序处理。
- 消费事件时产生的新事件，是本轮继续消费，还是留到下一轮。
- 同类型事件是否允许重复入队。
- 队列满时是丢弃、覆盖、扩容还是阻塞。
- 高优先级事件是否能够抢占普通事件。

为了避免事件处理过程中不断产生新事件并形成无限递归，可以使用当前队列和待处理队列分离的方式。

```cpp
void EventBus::dispatch() {
    current_.swap(pending_);

    for (const Event& event : current_) {
        handle(event); // 新事件写入 pending_
    }

    current_.clear();
}
```

这样本轮只消费开始时已经存在的事件，处理过程中产生的新事件统一留到下一轮。

**事件语义：** 并不是所有消息都需要逐条、完整地处理。

- `ActorDied`、`ItemAcquired` 属于事实型事件，通常不能随意丢失。
- `RefreshPanel`、`PlayFootstep` 属于请求型事件，可以根据情况去重、合并或降级。
- 网络延迟、加载进度等状态型消息通常只关心最新值，可以用新值覆盖旧值。

例如同一帧连续产生十次“刷新背包界面”请求，最终只刷新一次即可。大量距离很远的脚步声也可以根据优先级和音频通道数量丢弃。

**多线程边界：** 跨线程队列需要处理锁、内存可见性和对象线程安全问题。客户端中的消息量不高时，简单互斥队列或每线程本地队列通常比复杂无锁队列更容易维护。无锁结构只有在锁竞争已经被分析工具确认是瓶颈时才值得使用。

跨线程事件也不应直接携带只能在主线程访问的引擎对象。后台线程可以传递资源 ID、计算结果或不可变数据，再由主线程解析并应用。

**调试能力：** 事件队列会让调用关系从函数调用栈中消失，因此关键事件最好记录事件类型、生产者、帧号、序列号、关键参数和消费阶段。战斗回放、网络同步和线上问题定位都依赖这种事件轨迹。

**适用场景：** 音频请求、网络消息分发、资源加载结果、任务和成就通知、战斗表现事件、遥测和埋点。

**不适用场景：** 调用方必须立即得到返回值；状态必须在当前调用栈中完成修改；事件延迟会破坏确定性模拟。

**优点：** 解耦发送与处理时机，可以批处理、削峰、跨线程，并统一进行优先级、去重和日志记录。

**缺点：** 状态变化会产生延迟；事件顺序和生命周期更复杂；调用链不直观；队列堆积可能把即时错误变成延迟爆发的问题。

**一句话总结：** 事件队列通过“先记录事件，后统一处理”实现时间解耦。关键不只是入队和出队，而是事件语义、对象有效性、顺序、容量和调试能力。

### 服务定位器模式（Service Locator Pattern）

**定义：** 服务定位器为全局或广泛使用的服务提供统一查询入口。调用方依赖服务接口，而不是直接依赖具体实现。它的核心作用是：**集中管理公共服务的访问方式和实现替换。**

```cpp
class IAudioService {
public:
    virtual ~IAudioService() = default;
    virtual void play(SoundId id) = 0;
};

class NullAudioService final : public IAudioService {
public:
    void play(SoundId) override {}
};

class Services {
public:
    static void provideAudio(IAudioService& audio) {
        audio_ = &audio;
    }

    static IAudioService& audio() {
        return *audio_;
    }

private:
    static inline NullAudioService nullAudio_;
    static inline IAudioService* audio_ = &nullAudio_;
};
```

调用方不再依赖 `OpenALAudioService`、`UnityAudioService` 等具体类型，也不需要把音频服务从最上层逐级传递到每一个对象中。

```cpp
void Weapon::fire() {
    Services::audio().play(SoundId::GunShot);
}
```

与直接访问 `AudioManager::Instance()` 相比，服务定位器至少通过接口隔离了具体实现，可以注册真实实现、静音实现或测试替身。

**隐藏依赖：** 服务定位器没有消除依赖，只是把依赖从构造函数中移到了全局查询入口。单看 `Weapon` 的成员和构造函数，无法判断它依赖音频服务。任何对象都能访问定位器，也可能导致调用边界和生命周期失控。

因此，服务定位器更适合职责稳定、作用域明确、调用方很多的基础服务，例如日志、音频、平台接口、遥测、时间源和崩溃上报。

当前玩家、当前关卡、战斗世界或房间实例通常不适合作为进程级全局服务。这些对象存在切换、重建或多实例需求，更适合由玩法上下文、场景上下文或房间对象显式持有。

**空对象：** 空对象可以让服务缺失时安全地执行无操作，避免调用方到处进行空指针检查。

```cpp
Services::audio().play(id);
```

未注册真实音频服务时，调用会落到 `NullAudioService`。但空对象只适合允许降级的能力。存档、支付或资源系统如果静默失败，可能比直接报告错误更危险，这类服务应在初始化阶段强制校验。

**与依赖注入的区别：** 服务定位器由对象主动查询依赖，使用方便但依赖隐藏；依赖注入由外部把依赖传入对象，依赖关系更显式，也更容易控制作用域和编写测试。

```cpp
class Weapon {
public:
    explicit Weapon(IAudioService& audio)
        : audio_(audio) {}

private:
    IAudioService& audio_;
};
```

工程中可以组合使用两者：系统入口通过服务定位器获得一次公共服务，再把接口注入局部业务对象。这样既避免参数从程序最上层传遍整个工程，也不会让每一个小对象都能随意访问全局状态。

**生命周期和线程安全：** 需要明确服务何时注册、何时销毁、运行中是否允许替换，以及替换期间是否仍有对象正在调用。较稳妥的约束是启动阶段注册、运行阶段只读，服务生命周期长于所有使用者，只有测试或编辑器环境允许替换。

如果客户端支持多个游戏世界、多个房间预览或编辑器多场景运行，应使用作用域定位器，而不是所有环境共享一份进程级静态服务。

**与单例模式的区别：** 单例模式关心某个类只能存在一个实例；服务定位器关心调用方如何获得某种服务。定位器中可以注册单例，也可以注册普通对象、代理对象或测试替身。

**优点：** 提供统一访问入口，具体实现可以替换，能够使用空对象和装饰器，也能减少公共服务的层层传参。

**缺点：** 对象依赖被隐藏，容易退化成全局变量集合；初始化顺序、生命周期、多世界隔离和测试状态清理都比较困难。

**一句话总结：** 服务定位器通过“统一入口 + 服务接口”管理公共服务，但它只是受控的全局访问方式，不是消除依赖。使用边界应比普通单例更加严格。

## 优化型模式

优化型模式不是为了让代码看起来更高级，而是针对已经确认的瓶颈改变数据布局、更新策略或对象管理方式。

优化前应先区分问题来自 CPU 计算、缓存未命中、内存带宽、GPU 提交、运行时分配、垃圾回收、锁竞争，还是资源 I/O。模式必须与瓶颈匹配：对象池无法减少 Draw Call，空间分区无法降低材质切换，脏标记也不能弥补错误的算法复杂度。

### 数据局部性模式（Data Locality Pattern）

**定义：** 数据局部性模式通过调整数据在内存中的布局，让同一处理阶段需要的数据尽量连续、紧凑地存放，从而提高 CPU 缓存命中率。它的核心作用是：**减少 CPU 等待内存的时间，而不只是减少执行指令。**

传统面向对象结构通常由对象指针和多层成员指针组成：

```cpp
for (Actor* actor : actors) {
    actor->update();
}
```

每个 `Actor` 可能继续访问 `Transform`、`AIState`、`Animation` 和 `PhysicsBody`。这些对象分散在不同内存位置，CPU 很难提前加载下一次需要的数据，而且一个缓存行中还可能包含当前更新阶段完全不会使用的字段。

按访问阶段组织数据后，移动系统可以只遍历位置和速度：

```cpp
struct MovementData {
    std::vector<Vec3> positions;
    std::vector<Vec3> velocities;
};

void updateMovement(MovementData& data, float dt) {
    const size_t count = data.positions.size();

    for (size_t i = 0; i < count; ++i) {
        data.positions[i] += data.velocities[i] * dt;
    }
}
```

**AoS 与 SoA：** AoS 是结构体数组，单个对象的字段存放在一起；SoA 是数组结构，同一种字段连续存放。

```cpp
struct Particle {
    Vec3 position;
    Vec3 velocity;
    Color color;
    float lifetime;
};

std::vector<Particle> particles;
```

AoS 适合一次处理单个对象的大部分字段。

```cpp
struct Particles {
    std::vector<Vec3> positions;
    std::vector<Vec3> velocities;
    std::vector<float> lifetimes;
};
```

SoA 适合批量处理同一种字段或少量字段，更有利于缓存和 SIMD。二者没有绝对优劣，应根据热点循环的真实访问方式选择。很多系统最终会使用混合布局，例如物理阶段把位置、速度和半径放在一起，而名称、背包和剧情数据保存在另一块冷数据中。

**热数据与冷数据：** 高频访问的小数据应与低频访问的大数据分离。

```cpp
struct ActorHotData {
    Vec3 position;
    Vec3 velocity;
    float radius;
    uint32_t state;
};

struct ActorColdData {
    std::string debugName;
    std::vector<ItemId> inventory;
    DialogueData dialogue;
};
```

战斗循环不应因为读取角色位置，就把名称、背包和对话数据一同载入缓存。

**连续存储的边界：** 数据连续并不代表一定更快，还要考虑访问模式、容器扩容、数据对齐、分支预测、虚函数间接调用和多线程伪共享。

如果系统中只有十几个对象，或每个对象的逻辑分支差异很大，把代码强行改成 SoA 可能只会增加复杂度。数据局部性优化应以性能分析中的缓存未命中、内存带宽或热点循环为依据。

**稳定句柄：** 连续数组删除对象时经常使用交换删除：

```cpp
items[index] = std::move(items.back());
items.pop_back();
```

这种操作会改变元素所在位置，因此外部不应长期保存数组元素的裸地址或不带校验的索引。可以使用“索引 + 代数”的句柄：

```cpp
struct EntityHandle {
    uint32_t index;
    uint32_t generation;
};
```

当槽位被复用时递增 `generation`，旧句柄即使指向相同索引，也会因为代数不同而被判断为失效。

**与组件模式和 ECS 的关系：** 组件模式主要解决代码组织和能力组合，数据局部性主要解决运行时内存访问。传统组件对象可能分散在堆上，结构清晰但缓存较差。ECS 通常把同类组件连续存放，再由系统批量处理，因此同时涉及组件化和数据局部性，但两者不是同一个概念。

**适用场景：** 大量粒子、单位、投射物、动画采样数据、物理代理、可见性判断和 ECS 系统。

**不适用场景：** 对象数量少、更新频率低、逻辑高度异构，或者性能分析没有显示内存访问是瓶颈。

**优点：** 提高缓存命中率，便于批量处理、向量化和多线程分块。

**缺点：** 数据和行为可能分离，代码可读性下降；对象移动、句柄管理和数据迁移更复杂；数据布局会与具体访问模式绑定。

**一句话总结：** 数据局部性模式通过“让一起处理的数据在内存中靠在一起”减少缓存未命中，优化重点从类的结构转向热点循环的真实访问路径。

### 脏标记模式（Dirty Flag Pattern）

**定义：** 脏标记模式用于缓存计算成本较高的派生结果。当原始数据发生变化时，只把缓存标记为失效，在真正需要结果或指定刷新阶段再重新计算。它的核心作用是：**合并多次修改，避免重复计算派生数据。**

```cpp
class Transform {
public:
    void setLocalPosition(const Vec3& value) {
        localPosition_ = value;
        markWorldDirty();
    }

    const Matrix4& worldMatrix() {
        if (worldDirty_) {
            worldMatrix_ = parent_
                ? parent_->worldMatrix() * localMatrix()
                : localMatrix();
            worldDirty_ = false;
        }

        return worldMatrix_;
    }

private:
    void markWorldDirty() {
        if (worldDirty_) return;

        worldDirty_ = true;
        for (Transform* child : children_) {
            child->markWorldDirty();
        }
    }

    bool worldDirty_ = true;
    Vec3 localPosition_{};
    Matrix4 worldMatrix_{};
    Transform* parent_{};
    std::vector<Transform*> children_;
};
```

**原始数据与派生数据：** 使用脏标记前，必须明确哪些数据是权威来源，哪些数据只是可以重新计算的缓存，以及哪些变化会使缓存失效。

例如本地位置、旋转和缩放是原始数据，世界矩阵是派生数据；装备和 Buff 列表是原始数据，最终攻击力和战斗力可能是派生数据。

如果原始数据和派生数据的边界不明确，系统就可能同时修改两份状态，导致缓存失去意义。

**刷新方式：** 脏数据通常有两种重算方式。

按需计算是在第一次读取时重新计算，之后继续使用缓存。它适合派生结果不一定会被访问的场景，但可能把耗时隐藏在任意 getter 中。

批量刷新是在固定阶段集中重建所有脏数据：

```cpp
for (Transform* transform : dirtyTransforms) {
    transform->rebuildWorldMatrix();
}
```

这种方式更容易统计耗时和安排多线程任务，也能确保后续阶段只读取稳定结果。但脏对象列表需要去重，否则同一对象可能被重复加入并重复计算。

**依赖传播：** 层级结构中，父节点变化会使所有后代的世界变换失效。最直接的方式是递归标记所有子节点，但大型层级可能反复遍历相同子树。

另一种方式是使用版本号。子节点记录上次计算时使用的父节点版本，如果当前父版本已经变化，就在读取或批量刷新时重新计算。

```cpp
if (cachedParentVersion_ != parent_->worldVersion()) {
    rebuild();
}
```

复杂缓存依赖中，版本号通常比多个布尔值更容易表达“缓存基于哪一版源数据计算”。

**多线程问题：** 按需计算会让一个看似只读的 getter 实际修改缓存。多个线程同时读取时，可能重复计算，也可能发生数据竞争。

多线程客户端中更稳妥的方式是：在任务阶段批量重建缓存，后续阶段只读。这样能够明确写入边界，也便于设置任务依赖。

**常见错误：**

- 修改了原始数据却忘记标脏，最终读取到过期结果。
- 一个缓存依赖多个数据源，但只监听了部分变化。
- 在 getter 中进行高成本重算，造成难以预测的帧尖峰。
- 脏标记从未清除，系统退化为每次读取都重新计算。
- 原始计算本身非常便宜，却为了缓存增加大量一致性状态。

**适用场景：** Transform 层级、UI 布局、装备属性汇总、包围盒、静态合批数据、导航网格局部更新和序列化快照。

**优点：** 能够合并连续修改，避免无效重算，并把昂贵工作延迟到真正需要时或统一阶段执行。

**缺点：** 增加缓存一致性状态；漏标脏会产生隐蔽错误；按需重算可能导致帧尖峰和线程安全问题。

**一句话总结：** 脏标记通过“变化时只声明缓存失效，使用前再重建”减少重复计算。真正困难的不是一个布尔变量，而是完整维护缓存依赖和刷新时机。

### 对象池模式（Object Pool Pattern）

**定义：** 对象池预先或分批创建一组可复用对象，使用时取出，结束后重置并归还，避免高频创建和销毁。它的核心作用是：**控制运行时分配、内存碎片和垃圾回收抖动。**

```cpp
template<class T>
class ObjectPool {
public:
    template<class... Args>
    T* acquire(Args&&... args) {
        if (free_.empty()) {
            storage_.push_back(std::make_unique<T>());
            free_.push_back(storage_.back().get());
        }

        T* object = free_.back();
        free_.pop_back();
        object->onAcquire(std::forward<Args>(args)...);
        return object;
    }

    void release(T* object) {
        object->onRelease();
        free_.push_back(object);
    }

private:
    std::vector<std::unique_ptr<T>> storage_;
    std::vector<T*> free_;
};
```

对象池不是简单地把 `new` 和 `delete` 藏起来。一个池化对象会经历多次逻辑生命周期：

```text
从未使用 → 第一次激活 → 归还 → 再次激活 → 再次归还
```

**状态重置：** 对象归还时通常需要清理：

- 位置、速度、生命值和计数器。
- 父子节点、场景挂接关系和激活状态。
- 事件订阅、委托、计时器和协程。
- 粒子、动画、拖尾和音频播放状态。
- 目标对象、资源引用和异步回调。
- 网络序列号、帧号、命中列表和伤害记录。

只调用 `SetActive(false)` 并不代表对象已经安全归还。很多对象池问题都来自上一轮生命周期残留的状态。

**容量策略：** 池容量过小会在战斗高峰临时扩容，仍然产生卡顿；容量过大则会长期占用内存。常见策略包括：

- 根据历史峰值预热一部分对象。
- 运行中分批扩容，避免单帧创建大量实例。
- 设置软上限和硬上限。
- 达到上限后丢弃低优先级效果，或回收最旧对象。
- 将场景级对象池与全局对象池分离，场景退出后释放大容量池。

移动端尤其需要注意常驻内存。对象池本质上是用更多长期内存和更复杂的状态管理，换取更稳定的运行时分配成本。

**引用有效性：** 对象被归还后，外部可能仍然保存它的旧引用。该对象再次被取出时，同一地址已经表示新的逻辑实例，旧回调或旧事件可能错误地操作它。

可以使用带代数的池句柄：

```cpp
struct PooledHandle {
    uint32_t index;
    uint32_t generation;
};
```

槽位每次重新分配时递增代数，事件、异步回调和外部引用在访问前检查句柄版本，能够避免旧生命周期继续影响新对象。

**与内存池的区别：** 对象池复用已经构造过的业务对象，重点是对象状态和逻辑生命周期；内存池管理固定大小或特定策略的内存块，重点是分配器开销和内存碎片。对象池可以建立在普通堆上，也可以使用内存池作为底层分配器。

**与原型模式的区别：** 原型模式解决如何从模板创建相似对象，对象池解决如何复用已经创建的实例。实际项目中可以先用原型或配置初始化对象，再将运行时实例放入池中反复使用。

**不适合池化的情况：** 对象创建频率低、生命周期很长；对象内部资源差异很大，重置成本接近重新创建；运行时分配并不是瓶颈；对象池会产生大量闲置内存或复杂悬空引用。

**适用场景：** 子弹、命中特效、飘字、短生命周期 UI 项、敌人波次、音频通道和网络消息缓冲区。

**优点：** 减少高频创建和销毁，降低 GC 抖动和内存碎片，也能够显式控制对象峰值数量。

**缺点：** 状态重置复杂，容量难以估计，旧引用和事件订阅容易污染下一次生命周期，并且会增加常驻内存。

**一句话总结：** 对象池通过“借出—重置—归还”复用短生命周期对象。真正的工程难点是状态隔离、容量策略和引用有效性，而不是维护一个空闲列表。

### 空间分区模式（Spatial Partition Pattern）

**定义：** 空间分区模式按照对象在空间中的位置建立索引，使系统只检查附近或相关区域的对象，而不是每次遍历整个世界。它的核心作用是：**把全量空间查询缩小为局部候选集。**

没有空间分区时，搜索附近敌人通常需要遍历所有对象：

```cpp
for (Actor* actor : allActors) {
    if (distance(actor->position(), center) <= radius) {
        result.push_back(actor);
    }
}
```

单次查询复杂度是 `O(N)`。如果大量单位每帧都进行邻域查询，整体开销可能接近 `O(N²)`。

**均匀网格：** 规则地图和对象分布相对均匀时，均匀网格是实现简单、更新成本较低的选择。

```cpp
class SpatialGrid {
public:
    CellCoord toCell(const Vec2& position) const {
        return {
            static_cast<int>(std::floor(position.x / cellSize_)),
            static_cast<int>(std::floor(position.y / cellSize_))
        };
    }

    void move(EntityId id, const Vec2& oldPosition,
              const Vec2& newPosition) {
        CellCoord oldCell = toCell(oldPosition);
        CellCoord newCell = toCell(newPosition);

        if (oldCell == newCell) return;

        remove(oldCell, id);
        insert(newCell, id);
    }
};
```

半径查询时只遍历圆形范围覆盖的格子，再对候选对象进行精确距离判断。空间结构只负责粗筛，不能替代最终的阵营过滤、包围体测试和距离检测。

**单元格大小：** 格子太大时，单个格子包含大量对象，查询会退化为局部全量遍历；格子太小时，一次范围查询会跨越很多格子，移动对象也更频繁地更新所属格子。

格子大小通常应参考最常用的查询半径、对象尺寸和场景密度，再通过实际运行数据调整，不存在适合所有项目的固定值。

**常见空间结构：**

- 均匀网格或空间哈希适合动态对象多、查询半径相对稳定的场景。
- 四叉树和八叉树适合空间密度差异较大的二维或三维世界。
- BVH 和动态 AABB Tree 常用于碰撞 Broad Phase 与射线查询。
- KD-Tree 适合静态或更新不频繁的最近邻查询。
- 导航网格分区适合路径查询和可行走区域管理。

空间结构的选择取决于查询类型和对象更新频率。静态树结构通常查询效率较高，但频繁移动时，重新插入和维护结构的成本也可能很高。

**大型对象：** 一个大型对象可能覆盖多个格子。如果只按照中心点放入一个格子，查询边界区域时可能漏掉它；如果放入所有覆盖格子，又会产生重复候选。

常见处理方式是按包围盒插入多个格子并在查询时去重，或者把大型对象单独放入更高层级的结构中。

**更新与查询时序：** 对象位置变化后，空间索引必须同步更新。较清晰的阶段顺序是：

```text
移动或物理更新 → 刷新空间索引 → 邻域查询或碰撞检测
```

如果移动和查询交错进行，不同对象可能看到一份半更新状态的空间索引。可以通过阶段化更新、双缓冲索引或读写锁解决，但应根据实际并发需求选择，不要默认使用复杂同步结构。

**过滤顺序：** 高效查询通常先执行便宜且排除率高的判断：

```text
空间格粗筛 → 阵营或类型过滤 → 包围体测试 → 精确距离或碰撞测试
```

把昂贵计算放在最后，可以显著减少需要执行精确检测的对象数量。

**与物理引擎 Broad Phase 的关系：** 物理引擎通常已经维护碰撞粗筛结构。业务系统是否复用物理查询，要根据查询频率和过滤需求决定。偶尔查找附近单位时，直接使用物理接口可能已经足够；AI、技能和兴趣管理每帧进行大量查询时，单独维护业务空间索引通常更可控。

**适用场景：** 附近敌人搜索、AOE 技能、碰撞 Broad Phase、视野和兴趣管理、音频候选筛选、开放世界实体管理和网络同步区域划分。

**优点：** 大幅减少候选对象数量，使大量单位和大世界中的局部查询能够扩展。

**缺点：** 需要持续维护空间索引；参数选择会直接影响性能；大型对象、动态更新和并发访问会增加复杂度；对象分布极端时结构可能退化。

**一句话总结：** 空间分区通过“先按位置建立索引，再只检查相关区域”避免全量遍历。关键是结构选择、单元尺度、动态维护和最终精确过滤。
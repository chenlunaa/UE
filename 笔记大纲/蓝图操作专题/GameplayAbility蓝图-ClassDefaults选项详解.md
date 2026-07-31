# GameplayAbility 蓝图：Class Defaults 选项作用详解

> 基于 UE5.3 学习视频第 10 章第 3 节字幕整理

---

## 一、在哪里设置

打开一个继承自 `GameplayAbility`（或项目自定义 Ability 基类）的蓝图，然后点击工具栏中的：

```text
Class Defaults（类默认值） → Details（细节）面板
```

这里的配置决定能力的标签关系、激活限制、实例化方式、网络执行方式、消耗、冷却和自动触发条件。

> `Class Settings` 用于查看或修改蓝图的父类等类级信息；本篇讨论的是 `Class Defaults` 中属于 Gameplay Ability 的默认配置。

---

## 二、Tags：标签配置

### 2.1 Ability Tags（能力标签）

表示**这个能力自身拥有哪些标签**，用于标识和分类能力。

例如：

```text
Abilities.Attack
Abilities.Spell.Fire
Abilities.Movement.Dash
```

其他能力可以根据这些标签取消或阻止该能力。

> Ability Tags 描述的是能力本身，不等于角色当前拥有的标签。

### 2.2 Cancel Abilities with Tag（取消带有标签的能力）

当当前能力成功激活时，取消其他正在执行且拥有指定 Ability Tag 的能力。

示例：冲刺能力设置取消 `Abilities.Attack`，那么冲刺激活时，可取消角色当前正在执行的攻击能力。

### 2.3 Block Abilities with Tag（阻止带有标签的能力）

当前能力处于激活状态期间，拥有指定 Ability Tag 的其他能力不能激活。

示例：眩晕能力激活期间阻止 `Abilities.Attack` 与 `Abilities.Movement`，角色便无法开始攻击或移动类能力。

### Cancel 与 Block 的区别

| 选项 | 处理对象 | 作用时机 | 结果 |
|---|---|---|---|
| Cancel Abilities with Tag | 已经处于激活状态的其他能力 | 当前能力激活时 | 结束或取消匹配能力 |
| Block Abilities with Tag | 之后尝试激活的其他能力 | 当前能力保持激活期间 | 拒绝匹配能力启动 |

> Cancel 处理“已经在运行的能力”，Block 限制“接下来想启动的能力”。

### 2.4 Activation Owned Tags（激活期间拥有的标签）

当前能力处于激活状态时，把这些标签临时赋予能力拥有者；能力结束后，这些标签随之移除。

示例：攻击能力激活时赋予：

```text
State.Attacking
```

其他能力或 Gameplay Effect 就可以通过 `State.Attacking` 判断角色是否正在攻击。

字幕还提到：这些标签是否复制与 Ability System Globals 中的相关全局设置有关。

---

## 三、能力激活所需与阻挡标签

这组配置用于声明：能力拥有者处于什么状态时，当前能力可以或不可以激活。

### 3.1 Activation Required Tags（激活要求标签）

能力拥有者的 ASC 必须拥有这里配置的**全部标签**，能力才能激活。

```text
要求：State.Weapon.Equipped
结果：只有装备武器时才能激活攻击能力
```

### 3.2 Activation Blocked Tags（激活阻挡标签）

能力拥有者的 ASC 只要拥有这里配置的**任意一个标签**，能力就会被阻止激活。

```text
阻挡：State.Stunned、State.Dead
结果：眩晕或死亡任一状态存在时，都不能激活能力
```

### Required 与 Blocked 的判断规则

| 类型 | 判断方式 |
|---|---|
| Required Tags | 必须全部拥有，属于 All 条件 |
| Blocked Tags | 拥有任意一个就阻止，属于 Any 条件 |

---

## 四、Source 与 Target 标签条件

这两组条件用于根据能力来源和目标的 ASC 标签决定能力能否激活。

### 4.1 Source Required Tags

来源 ASC 必须拥有配置中的全部标签，能力才能激活。

### 4.2 Source Blocked Tags

来源 ASC 只要拥有配置中的任意标签，能力就不能激活。

### 4.3 Target Required Tags

目标 ASC 必须拥有配置中的全部标签，能力才能激活。

示例：某个治疗能力只允许对拥有 `State.Alive` 的目标使用。

### 4.4 Target Blocked Tags

目标 ASC 只要拥有配置中的任意标签，能力就不能激活。

示例：目标拥有 `State.Invulnerable` 时，不允许激活针对它的伤害能力。

| 条件对象 | Required | Blocked |
|---|---|---|
| Ability Owner / Avatar | Activation Required Tags | Activation Blocked Tags |
| Source ASC | Source Required Tags | Source Blocked Tags |
| Target ASC | Target Required Tags | Target Blocked Tags |

> Source、Target 与能力拥有者并不一定是同一个对象，具体数据取决于该能力如何被激活以及是否提供了来源、目标信息。

---

## 五、Input：输入设置

### Replicate Input Directly（直接复制输入）

启用后，输入按下和释放状态会直接发送到服务器。

这可能产生大量 RPC，尤其是每帧处理输入或玩家快速连续点击时，因此课程和引擎注释都明确表示：**通常不建议使用**。

推荐思路是仅在真正需要同步的时刻发送 GAS 的通用复制事件，而不是直接复制原始输入状态。

> 经验法则：保持关闭，不要把它作为 Ability 输入同步的常规方案。

---

## 六、Instancing Policy：实例化策略

实例化策略决定 Ability 对象如何创建、复用以及能否保存状态。

### 6.1 Instanced Per Actor（按角色实例化）

每个能力拥有者只创建一个该 Ability 的实例，之后每次激活都重复使用它。

**优点：**

- 可以保存成员变量和持久数据。
- 可以使用 AbilityTask 并绑定委托。
- 对象会被复用，性能优于每次执行都创建新对象。

**注意：**

- 上一次激活留下的变量不会自动重置。
- 每次激活需要重新初始化的状态，必须手动重置。

**课程推荐：**大多数常规能力使用此选项。

### 6.2 Instanced Per Execution（按执行实例化）

每次激活能力时都创建一个新 Ability 实例，结束后销毁。

**优点：**

- 每次执行拥有独立状态。
- 不会意外保留上一次激活的成员变量。
- 适合需要同一能力多次执行实例彼此独立的场景。

**缺点：**

- 频繁创建和销毁对象，三种策略中开销最高。
- 不会在多次激活之间保留实例数据。

### 6.3 Non-Instanced（非实例化）

不创建 Ability 实例，直接通过类默认对象（CDO）执行。

**优点：**性能开销最低。

**限制：**

- 不能在 Ability 对象中保存每次执行的状态。
- 无法像实例化能力一样绑定 AbilityTask 委托。
- 功能限制较多，可近似理解为调用无状态的静态逻辑。

### 实例化策略对照

| 策略 | 对象生命周期 | 能否保存实例状态 | AbilityTask | 性能 |
|---|---|---|---|---|
| Instanced Per Actor | 每个拥有者一个，反复复用 | 可以，注意手动重置 | 支持 | 较好 |
| Instanced Per Execution | 每次激活创建一个 | 仅本次执行 | 支持 | 开销最高 |
| Non-Instanced | 不创建实例，使用 CDO | 不可以 | 受限，不能绑定任务委托 | 最佳 |

---

## 七、Retrigger Instanced Ability

`Retrigger Instanced Ability` 表示：当一个实例化能力已经处于激活状态时，如果再次尝试激活，是否先结束当前执行，再重新触发能力。

- **关闭（默认）**：已激活时不会通过该选项自动结束并重启。
- **开启**：再次激活会结束当前执行，然后从头重新触发。

> 字幕中的“重新触发不稳定性”是机翻错误，实际选项是 `Retrigger Instanced Ability`。

---

## 八、Net Execution Policy：网络执行策略

网络执行策略决定能力在哪一端运行，以及客户端是否进行预测。

### 8.1 Local Only（仅本地）

只在本地运行，服务器不执行该能力。

适合只影响本地玩家、与权威游戏状态无关的装饰性或本地功能。不要用它处理服务器必须验证的伤害、资源或关键状态。

### 8.2 Local Predicted（本地预测）

拥有客户端先在本地执行，然后把请求发送给服务器；服务器验证并运行权威版本，如果预测无效则进行纠正或回滚。

**优点：**客户端无需等待一次网络往返，操作响应更及时。

**课程建议：**大多数由玩家主动触发的常规能力使用此策略。

### 8.3 Server Only（仅服务器）

能力只在服务器运行，不在拥有客户端执行 Ability 本身。

适合必须由服务器处理的持续逻辑、事件监听或权威性任务。

### 8.4 Server Initiated（服务器启动）

能力先由服务器启动，并使拥有客户端执行对应能力。由于必须先经过服务器，响应性不如本地预测，适合由服务器主动发起的能力。

### 网络执行策略对照

| 策略 | 启动位置 | 客户端预测 | 典型用途 |
|---|---|---|---|
| Local Only | 本地 | 无需 | 纯本地、非权威表现 |
| Local Predicted | 客户端发起，服务器验证 | 有 | 大多数玩家主动技能 |
| Server Only | 服务器 | 无 | 服务器权威持续逻辑 |
| Server Initiated | 服务器发起，并通知拥有客户端 | 无 | 服务器主动触发的能力 |

> Gameplay Ability 不会在模拟代理上执行。其他玩家需要看到的表现或状态，应通过可复制的 Gameplay Effect 和 Gameplay Cue 等系统呈现。

---

## 九、Replication Policy 与远程取消

### 9.1 Replication Policy（复制策略）

它描述 Ability 实例状态和事件的复制方式，但课程明确建议保持默认的 `Do Not Replicate`，无需调整。

原因是：

- 授予的 AbilitySpec 已由服务器同步给拥有客户端。
- `Local Predicted` 等网络执行策略已经负责能力的执行与预测流程。
- 模拟代理上的表现应使用 Gameplay Effect 或 Gameplay Cue，而不是依赖 Ability 实例复制。

> 经验法则：不要为了让多人游戏中的能力“能够同步”就开启 Ability 的 Replication Policy。

### 9.2 Server Respects Remote Ability Cancellation

开启时，如果拥有客户端取消了本地能力，服务器会接受远程取消并终止服务器上的对应能力。

课程不建议依赖此功能，因为服务器应当是权威来源。更稳妥的方向通常是由服务器决定能力是否应当取消，并让客户端服从服务器结果。

### 9.3 Net Security Policy

控制客户端是否有权请求执行、终止等能力操作。字幕只作简要介绍：默认的 `Client or Server` 没有额外安全限制，课程中大多数能力保持默认即可。

---

## 十、Cost Gameplay Effect：能力消耗

Gameplay Ability 的资源消耗通过一个 Gameplay Effect 实现。

在 `Cost Gameplay Effect Class` 中指定成本 GE，该 GE 决定：

- 消耗哪一种属性，例如 Mana、Stamina 或弹药。
- 每次消耗多少。

能力执行时需要在合适的位置提交（Commit）消耗；只有配置 Cost GE 并不代表任意时刻都会自动扣除。

```text
Ability 激活
   ↓
检查资源是否足够
   ↓
提交 Ability Cost
   ↓
应用 Cost Gameplay Effect，扣除属性
```

---

## 十一、Ability Triggers：能力触发器

`Ability Triggers` 是一个数组，可让能力响应事件或标签变化自动尝试激活。

每个元素包含一个 Gameplay Tag 和 Trigger Source。字幕提到三种触发来源：

| Trigger Source | 作用 |
|---|---|
| Gameplay Event | 收到对应标签的 Gameplay Event 时触发 |
| Owned Tag Added | 对应拥有标签被添加时触发 |
| Owned Tag Present | 对应拥有标签存在时触发 |

它适合制作事件驱动或状态驱动的能力，例如收到受击事件时触发反应能力，或角色获得某种状态标签时启动被动能力。

---

## 十二、Cooldown Gameplay Effect：能力冷却

Ability 的冷却同样通过 Gameplay Effect 实现。

`Cooldown Gameplay Effect Class` 指定的 GE 主要负责：

- 定义冷却持续时间。
- 在冷却期间提供用于阻止再次激活的冷却标签。

冷却从什么时候开始，取决于能力何时提交（Commit）冷却，而不单纯取决于能力何时结束或取消。

```text
提交 Ability Cooldown
        ↓
应用 Cooldown Gameplay Effect
        ↓
冷却标签存在，能力无法再次激活
        ↓
GE 到期并移除标签，能力恢复可用
```

---

## 十三、课程阶段推荐配置

对大多数由玩家输入触发、包含 AbilityTask 的普通技能，可先使用：

| 配置项 | 推荐值 | 原因 |
|---|---|---|
| Instancing Policy | `Instanced Per Actor` | 支持状态与 AbilityTask，对象可复用 |
| Net Execution Policy | `Local Predicted` | 保持客户端响应，同时由服务器验证 |
| Replication Policy | `Do Not Replicate` | Ability 不需要靠实例复制完成常规网络流程 |
| Replicate Input Directly | 关闭 | 避免大量输入 RPC |
| Server Respects Remote Ability Cancellation | 通常关闭或谨慎使用 | 保持服务器权威 |
| Cost / Cooldown | 配置对应 Gameplay Effect | 使用 GAS 的标准资源和冷却机制 |

具体能力仍需根据是否由服务器触发、是否需要独立执行实例、是否只有本地表现来调整。

---

## 十四、快速记忆

```text
Ability Tags             标识“我是什么能力”
Cancel Tags              激活时取消谁
Block Tags               激活期间不让谁启动
Activation Owned Tags    激活期间给拥有者什么状态
Required Tags            全部具备才可激活
Blocked Tags             任意具备就拒绝激活
Instancing Policy        Ability 对象如何创建和复用
Net Execution Policy     Ability 在哪一端执行、是否预测
Cost GE                  使用能力要扣什么资源
Cooldown GE              使用后多久才能再次激活
Ability Triggers         什么事件或标签变化会触发能力
```

一句话总结：

> GameplayAbility 的 Class Defaults 不是单纯的属性面板，而是用数据配置能力之间的互斥关系、激活门槛、对象生命周期和网络执行规则。

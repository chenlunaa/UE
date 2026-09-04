# 项目中期总览：UE 运行机制与 Aura 核心流程

> 笔记类型：项目概览阶段记录｜项目中期总览

> 本文是项目完成约一半时的阶段性总览。目标不是重复前 24 篇笔记，而是把已经实现的输入、GAS、UI、伤害、AI、近战和远程攻击串成一套完整的运行模型。

---

## 目录

- [1. 目前已经完成了什么](#section-1)
- [2. 先建立 UE 项目的整体心智模型](#section-2)
- [3. 一般 UE 项目从启动到第一帧](#section-3)
- [5. Aura 玩家生成与 GAS 初始化](#section-5)
- [10. 完整伤害结算管线](#section-10)
- [14. 近战与远程攻击完整闭环](#section-14)
- [18. 网络复制与 RPC 的判断框架](#section-18)
- [23. 中期知识地图](#section-23)

<a id="section-1"></a>
## 1. 目前已经完成了什么

目前的 Aura 已经不再只是“角色能移动、能放技能”的演示，而是具备了动作 RPG 的核心骨架：

- 使用 Enhanced Input 统一处理移动、Shift、鼠标和技能输入；
- 使用 GameplayTag 连接输入、能力、状态、伤害类型、动画事件和 UI 消息；
- 将 ASC 和 AttributeSet 放在 PlayerState 上，支持角色重生及多人复制；
- 建立 Primary、Secondary、Vital、Resistance 和 Meta Attribute；
- 用 GameplayEffect 完成属性初始化、增益、持续效果、周期效果和伤害；
- 用 GameplayAbility 完成技能激活、鼠标 TargetData、蒙太奇和投射物；
- 用 WidgetController 隔离 UMG 与 GAS，完成 Overlay、属性菜单、消息和敌人血条；
- 用 ExecCalc 完成多伤害类型、抗性、格挡、护甲、穿甲和暴击结算；
- 用自定义 EffectContext 携带格挡、暴击等一次伤害的附加结果；
- 用 Behavior Tree、Blackboard 和 EQS 完成敌人索敌、追击、选位和攻击；
- 打通近战命中与远程投射物的伤害闭环。

可以把当前项目概括为五个相互协作的系统：

```text
输入与控制 ──> Gameplay Ability ──> Gameplay Effect / 属性
     │                  │                       │
     │                  ├──> 动画 / 投射物      ├──> 伤害结果
     │                  │                       │
     └──> 移动 / 寻路   └──> GameplayTag <─────┤
                                                │
AI 决策 ──> 选择目标 / 激活能力                 └──> UI 展示
```

它们没有彼此直接写死，而是通过接口、委托、GameplayTag、DataAsset 和 GAS Spec 交换信息。这正是项目目前最有价值的架构成果。

---

<a id="section-2"></a>
## 2. 先建立 UE 项目的整体心智模型

一个 UE 游戏不是从某个 `main.cpp` 开始，由开发者手写循环。引擎已经提供启动、世界管理、对象生命周期、输入、物理、网络和渲染循环；项目代码是在引擎规定的扩展点中注册规则。

可以把 UE 分成五层：

| 层次 | 负责什么 | Aura 中的例子 |
|---|---|---|
| Engine / World | 启动、加载地图、Tick、物理、网络、渲染 | UWorld、Actor、Component |
| Game Framework | 决定规则、玩家和 Pawn 的关系 | GameMode、PlayerController、PlayerState、Character、HUD |
| Gameplay Systems | 实现可复用的游戏规则 | GAS、Enhanced Input、Navigation、AI、Motion Warping |
| Project Gameplay | 项目自己的战斗和交互 | AuraCharacter、AuraASC、ExecCalc、Projectile |
| Content / Presentation | 配置资产和最终表现 | 蓝图、UMG、动画、Niagara、DataAsset、CurveTable |

理解项目时，最好始终问四个问题：

1. 这个对象是谁创建的，存在于服务器、客户端还是两端？
2. 它在生命周期的哪个时刻获得依赖？
3. 数据由谁拥有，谁有权修改，如何复制？
4. 系统间通过直接调用、接口、委托、Tag，还是网络 RPC 通信？

---

<a id="section-3"></a>
## 3. 一般 UE 项目从启动到第一帧

下面是概念上的启动链。真实引擎内部更复杂，但这个模型足够用于定位项目代码：

```text
启动编辑器 PIE 或游戏程序
  ↓
FEngineLoop 初始化平台、Core、Engine 和项目模块
  ↓
加载 Aura Runtime Module
  ↓
创建 Engine / GameInstance 等全局对象
  ↓
AssetManager::StartInitialLoading
  ↓
加载目标 Map，创建 UWorld
  ↓
确定当前 GameMode
  ↓
服务器创建 GameMode、GameState
  ↓
玩家连接，创建 PlayerController、PlayerState
  ↓
GameMode 生成 Pawn，Controller Possess Pawn
  ↓
Actor BeginPlay，世界进入第一帧 Tick
```

Aura 模块的入口是：

```cpp
IMPLEMENT_PRIMARY_GAME_MODULE(FDefaultGameModuleImpl, Aura, "Aura");
```

这不是游戏玩法的入口，而是告诉 UE：“Aura 是本项目的主运行时模块”。模块加载后，类反射信息、静态注册和项目代码才进入引擎体系。

### 3.1 AuraAssetManager 的早期初始化

项目把引擎的 AssetManager 替换为 `UAuraAssetManager`。它在 `StartInitialLoading()` 中完成两件关键工作：

```cpp
FAuraGameplayTags::InitializeNativeGameplayTags();
UAbilitySystemGlobals::Get().InitGlobalData();
```

因此项目的早期主线是：

```text
Engine 启动
  -> 加载 Aura 模块
  -> AuraAssetManager 开始初始加载
  -> 注册原生 GameplayTag
  -> 初始化 GAS 全局数据和自定义 EffectContext
  -> 加载地图与游戏对象
```

必须尽早注册 Tag，因为输入配置、能力、Effect、UI 和伤害计算都把 Tag 当成协议；必须初始化 GAS 全局数据，因为 TargetData、EffectContext 等底层结构依赖它。

### 3.2 Map 和 GameMode 决定“这局游戏是什么”

`GameDefaultMap` 或 PIE 当前地图决定加载哪个 World。GameMode 的选择通常遵循：

```text
启动 URL 明确指定
  > 地图 World Settings 的 GameMode Override
  > Project Settings 的默认 GameMode
```

GameMode 蓝图还能继续配置 Default Pawn、PlayerController、PlayerState、HUD 等具体类。因此，C++ 类写对并不代表游戏一定使用了它，还要检查：

- 当前打开的是哪张 Map；
- World Settings 是否覆盖 GameMode；
- GameMode 蓝图的 Class Defaults；
- Project Settings 中的 Maps & Modes；
- C++ 默认值与蓝图子类是否一致。

当前 `DefaultEngine.ini` 中全局默认仍是引擎的 `GameModeBase`，说明实际 Aura GameMode 很可能由地图或蓝图覆盖。这个配置入口分散，是以后排查“为什么我的 Controller/HUD 没生效”时的第一检查点。

---

<a id="section-4"></a>
## 4. Game Framework 对象各自负责什么

| 对象 | 核心职责 | 服务器 | 所属客户端 | 其他客户端 |
|---|---|:---:|:---:|:---:|
| GameMode | 游戏规则、选择类、生成和重生玩家 | 有 | 无 | 无 |
| GameState | 需要让所有人知道的全局比赛状态 | 有 | 有 | 有 |
| PlayerController | 接收本地输入、控制 Pawn、发起 RPC、管理本地 HUD | 每个连接都有 | 只有自己的 | 通常没有别人的 |
| PlayerState | 玩家长期状态、等级、队伍、分数、ASC Owner | 有 | 有 | 有 |
| Pawn / Character | 玩家在世界中的实体、移动、碰撞、Avatar | 有 | 有 | 有 |
| HUD / Widget | 本地屏幕展示 | 不作为权威玩法数据 | 本地创建 | 各自创建自己的 |

最常见的误区是把这些类都当作“角色类”。它们解决的是不同生命周期问题：

- Character 可能死亡、销毁和重生；
- PlayerState 代表这个连接对应的玩家，通常比 Character 活得久；
- PlayerController 代表控制关系和本地入口；
- GameMode 只存在于服务器，所以客户端不能直接依赖它读取规则；
- Widget 是表现层，不能成为生命值等权威数据的拥有者。

---

<a id="section-5"></a>
## 5. Aura 玩家生成与 GAS 初始化：最重要的一条时序

### 5.1 为什么 ASC 放在 PlayerState

Aura 的玩家 ASC 和 AttributeSet 由 `AAuraPlayerState` 创建，而不是由 Character 创建：

```text
PlayerState
  ├── AbilitySystemComponent
  ├── AttributeSet
  └── Level 等长期玩家数据

Character
  └── 当前在世界中的身体和 ASC Avatar
```

这样做的主要收益是：

- Character 重生时，ASC 不必随身体一起销毁；
- PlayerState 本身就是可复制的玩家状态容器；
- OwnerActor 和 AvatarActor 的职责清晰；
- 能力、属性和效果可以跨 Pawn 生命周期保留或重新绑定。

关键绑定为：

```cpp
ASC->InitAbilityActorInfo(AuraPlayerState, this);
```

此时：

- **OwnerActor = PlayerState**：ASC 的长期所有者；
- **AvatarActor = Character**：当前替 ASC 在世界中行动的身体。

### 5.2 服务器初始化

```text
GameMode 创建 Controller / PlayerState / Character
  -> Controller Possess Character
  -> Character::PossessedBy
  -> Character::InitAbilityActorInfo
  -> ASC::InitAbilityActorInfo(PlayerState, Character)
  -> AbilityActorInfoSet
  -> 初始化默认属性
  -> 服务器授予初始 GameplayAbility
  -> HUD/Overlay 获得 ASC 和 AttributeSet
```

授予能力和修改权威属性通常只能由服务器完成。

### 5.3 客户端初始化

客户端不会执行服务器的 `PossessedBy` 来完成全部初始化，它依赖 PlayerState 的复制通知：

```text
PlayerState 复制到客户端 Character
  -> Character::OnRep_PlayerState
  -> Character::InitAbilityActorInfo
  -> ASC::InitAbilityActorInfo(PlayerState, Character)
  -> 创建和绑定本地 Overlay UI
```

因此 `PossessedBy` 与 `OnRep_PlayerState` 是同一目的的服务器/客户端两个入口。多人游戏里只在 `BeginPlay` 初始化 ASC，往往会遇到对象尚未复制到位的问题。

### 5.4 Mixed 复制模式

PlayerState 的 ASC 使用 Mixed Replication Mode。概念上：

- GameplayTag 和 Cue 等必要状态可以让相关客户端知道；
- 完整 GameplayEffect 信息主要复制给拥有该 ASC 的客户端；
- 相比 Full 模式节省流量，同时满足玩家自己 UI 和状态显示需求。

这也要求 OwnerActor 的所有权链正确，因此 PlayerState 作为 Owner 与 Controller 的关系很重要。

---

<a id="section-6"></a>
## 6. 输入系统：从按键到技能激活

Aura 使用 Enhanced Input 接收物理输入，再用 GameplayTag 把输入映射到能力。

### 6.1 初始化

`AAuraPlayerController::BeginPlay()` 主要完成：

- 向 Local Player 的 Enhanced Input Subsystem 添加 Mapping Context；
- 显示鼠标光标；
- 设置 Game and UI 输入模式；
- 配置鼠标锁定方式。

`SetupInputComponent()` 绑定：

- 移动 Action；
- Shift Action；
- 技能 Action 的 Pressed、Held、Released；
- InputConfig 中每个 Action 对应的 InputTag。

### 6.2 技能输入链

```text
键鼠输入
  -> Enhanced Input Action
  -> AuraInputComponent
  -> InputConfig 查找 InputTag
  -> PlayerController::AbilityInputTagHeld/Released
  -> AuraASC 遍历 ActivatableAbilities
  -> 检查 AbilitySpec.DynamicAbilityTags
  -> TryActivateAbility
  -> GameplayAbility::ActivateAbility
```

能力授予时，能力自身的 `StartupInputTag` 被加入 `AbilitySpec.DynamicAbilityTags`。因此输入系统不知道 FireBolt 的 C++ 类型，它只发布 `InputTag.LMB`；ASC 也不关心键盘具体按键，只寻找拥有该 Tag 的 AbilitySpec。

这实现了三层解耦：

```text
物理按键 <-> InputAction <-> InputTag <-> GameplayAbility
```

换键位、换能力和换角色配置时，不需要让所有系统彼此引用。

### 6.3 鼠标点击为什么既能移动又能放技能

LMB 根据上下文承担两种含义：

- 指向敌人或按住 Shift：把 LMB 交给技能输入；
- 指向地面且短按：计算导航路径并自动移动；
- 指向地面且长按：持续向光标方向输入移动。

短按寻路流程：

```text
LMB Released
  -> NavigationSystem 查找路径
  -> 路径点写入 Spline
  -> PlayerTick 中 AutoRun
  -> 沿 Spline 切线方向 AddMovementInput
  -> 接近终点后停止
```

### 6.4 鼠标高亮

```text
PlayerController::PlayerTick
  -> CursorTrace
  -> Visibility Channel Hit Result
  -> 检查 EnemyInterface
  -> 旧目标 UnHighlight
  -> 新目标 Highlight
  -> Mesh 开启 CustomDepth / Stencil
```

接口的价值在于 Controller 不需要知道命中的是 `AAuraEnemy` 还是以后新增的可交互目标，只需知道该对象是否支持高亮协议。

---

<a id="section-7"></a>
## 7. UI：为什么要有 WidgetController

UI 的核心流程是：

```text
HUD::InitOverlay
  -> CreateWidget(Overlay)
  -> 创建或取得 OverlayWidgetController
  -> 传入 PlayerController / PlayerState / ASC / AttributeSet
  -> BindCallbacksToDependencies
  -> Widget::SetWidgetController
  -> BroadcastInitialValues
  -> AddToViewport
```

WidgetController 是 GAS 数据与 UMG 表现之间的适配层：

```text
ASC / AttributeSet / PlayerState
  -> 原生委托、属性变化委托、Effect Tag
  -> WidgetController
  -> BlueprintAssignable 委托
  -> Widget 更新文字、进度条、动画和提示
```

如果 Widget 直接到处查找 Character、ASC 和 AttributeSet，会产生三个问题：

- Widget 很难复用和测试；
- 初始化时序与空指针问题散落在每个控件；
- 玩法层和表现层双向依赖。

当前架构使 Widget 只负责“如何显示”，WidgetController 负责“从哪里获取数据以及何时广播”。初始值用 `BroadcastInitialValues()` 主动推送，后续变化用 Delegate 增量更新。

GameplayEffect 的 UI 消息也是相同思路：

```text
GameplayEffect 应用到 ASC
  -> OnGameplayEffectAppliedDelegateToSelf
  -> 读取 EffectSpec Asset Tags
  -> AuraASC::EffectAssetTags 广播
  -> OverlayWidgetController 查消息 DataTable
  -> Widget 显示消息
```

这让一个 GE 只需携带例如 `Message.HealthPotion`，不需要直接调用某个 UI 类。

---

<a id="section-8"></a>
## 8. GAS 中四种核心对象的职责

| 对象 | 它回答的问题 | Aura 中的用途 |
|---|---|---|
| AbilitySystemComponent | 谁拥有和运行能力系统？ | 保存能力、效果、Tag，处理激活与复制 |
| AttributeSet | 有哪些可由 GAS 管理的数值？ | Health、Mana、Armor、Resistance、IncomingDamage |
| GameplayAbility | 角色想执行什么动作？ | 攻击、投射物、鼠标目标数据、蒙太奇 |
| GameplayEffect | 这次动作怎样改变属性/Tag？ | 初始属性、Buff、持续伤害、最终伤害 |

一个常用判断是：

- 有过程、有输入、有动画、有等待：放在 Ability；
- 描述数值变化、持续时间和 Tag：放在 Effect；
- 保存可复制、可监听的数值：放在 AttributeSet；
- 需要统一管理、激活、预测和复制：通过 ASC。

---

<a id="section-9"></a>
## 9. 属性初始化、MMC 与 ExecCalc 的区别

项目中存在三类“计算”，它们用途不同。

### 9.1 普通 Modifier / CurveTable

适合直接加减乘除，或者按等级从曲线中取得系数。例如某个等级的基础力量、固定增加 10 点护甲。

特点：简单、数据驱动、易配置，优先使用。

### 9.2 MMC（Modifier Magnitude Calculation）

MMC 的任务是**算出一个 Modifier 的 Magnitude**。它常用于单个派生属性：

```text
读取 Source/Target 属性或等级
  -> 运行一段可复用公式
  -> 返回一个浮点 Magnitude
  -> GE 用这个数修改一个属性
```

Aura 的 `MMC_MaxHealth`、`MMC_MaxMana` 适合表达：最大生命/法力如何由体力、智力、等级等输入派生。

它的重点是“一条 Modifier 的值是多少”，不是一次复杂攻击的所有结算步骤。

### 9.3 ExecCalc（Gameplay Effect Execution Calculation）

ExecCalc 适合一次 Effect 中需要同时读取多项属性、执行复杂规则、输出一个或多个结果的结算：

```text
多种 Damage SetByCaller
  -> 对应 Resistance
  -> BlockChance
  -> ArmorPenetration / Armor
  -> CriticalHit / CriticalResistance
  -> 输出 IncomingDamage
  -> 在 EffectContext 记录暴击和格挡
```

### 9.4 最容易记住的对比

| 问题 | 普通 Modifier | MMC | ExecCalc |
|---|---|---|---|
| 典型用途 | 固定或简单数值变化 | 计算一个 Modifier 的大小 | 一次复杂结算 |
| 输出 | 配置好的数值 | 通常一个 Magnitude | 一个或多个 Execution Output |
| 复杂度 | 低 | 中 | 高 |
| Aura 示例 | +10 Armor | MaxHealth | Damage |
| 是否适合伤害全流程 | 简单伤害可以 | 通常不适合 | 适合 |

实践原则是：能用普通 Modifier 就不要写代码；一个派生值用 MMC；多个捕获属性、概率判断和多阶段规则使用 ExecCalc。

---

<a id="section-10"></a>
## 10. 完整伤害结算管线

Aura 的伤害不是由 Projectile 直接执行 `Health -= Damage`，而是进入 GAS 管线：

```text
GameplayAbility
  -> 创建 Damage GameplayEffect Spec
  -> 按 DamageType Tag 写入 SetByCaller Damage
  -> ApplyGameplayEffectSpecToTarget
  -> ExecCalc_Damage 捕获攻防属性
  -> 伤害类型分别应用对应 Resistance
  -> 计算格挡、护甲/穿甲、暴击
  -> 输出 IncomingDamage Meta Attribute
  -> AttributeSet::PostGameplayEffectExecute
  -> 清空 IncomingDamage 并扣除 Health
  -> 致命：Die
  -> 非致命：激活 HitReact Ability
  -> Client RPC 显示伤害数字
```

### 10.1 为什么使用 IncomingDamage 元属性

`IncomingDamage` 是结算过程的临时收件箱，不是角色长期状态：

1. ExecCalc 只负责算出最终伤害；
2. 结果写入 IncomingDamage；
3. AttributeSet 统一读取并清零；
4. 再处理扣血、死亡、受击和飘字。

这样把“数学公式”与“伤害发生后的游戏事件”分开。以后加入护盾、吸血、免死、处决时，不必把所有副作用塞进 ExecCalc。

### 10.2 自定义 EffectContext 做什么

EffectContext 描述这次 GameplayEffect 的上下文，例如 Instigator、Causer、SourceObject 和 HitResult。Aura 扩展它，用来携带：

- 是否格挡；
- 是否暴击；
- 后续可能需要的击退、死亡冲量、Debuff 等单次结算信息。

这些信息属于“这一次伤害”，不应变成角色永久属性。自定义 Context 在多人游戏中还要正确实现复制序列化和结构识别，否则服务器算出的附加结果无法可靠到达客户端。

---

<a id="section-11"></a>
## 11. GameplayTag 是跨系统协议，不只是标签

Aura 中的 Tag 至少承担五种角色：

| 类别 | 作用 | 示例概念 |
|---|---|---|
| Input Tag | 将 InputAction 映射到 AbilitySpec | InputTag.LMB |
| Ability Tag | 分类、查询和按类别激活能力 | Abilities.Attack |
| State Tag | 表示正在受击、眩晕等状态 | Effects.HitReact |
| Damage / Resistance Tag | 作为伤害数据键，并映射抗性 | Damage.Fire -> Resistance.Fire |
| Message / Event / Montage Tag | UI 消息、动画事件和攻击插槽协议 | Message.*、Montage.* |

Tag 的价值是把“依赖具体类”变成“依赖公共语义”。例如 AI 只要求激活 `Abilities.Attack`，不需要判断当前敌人是食尸鬼还是投石者；职业 DataAsset 决定它实际拥有什么攻击能力。

但 Tag 也需要治理：集中注册、命名有层级、明确读写方、避免字符串拼写和语义重复。

---

<a id="section-12"></a>
## 12. 投射物、TargetData 与网络权威

鼠标位置是本地客户端才直接知道的数据，而伤害必须由服务器权威执行。`TargetDataUnderMouse` AbilityTask 解决的就是这个边界：

```text
本地客户端获取 Cursor HitResult
  -> 包装为 GameplayAbilityTargetData
  -> 预测窗口内发送到服务器
  -> 服务器接收 TargetData
  -> Ability 根据目标位置生成投射物
```

投射物链路为：

```text
Projectile Ability 激活
  -> 播放/等待施法事件
  -> 服务器 Spawn Projectile
  -> 设置 DamageEffectParams / Spec 数据
  -> Projectile 移动与碰撞
  -> 检查敌我关系和有效目标
  -> 将 Damage GE Spec 应用到目标 ASC
  -> GAS 伤害管线
  -> 命中特效、音效、销毁
```

核心原则：

- 客户端负责采集输入和及时表现；
- 服务器负责生成权威投射物与应用伤害；
- 复制负责让其他客户端看到结果；
- 不要把客户端传来的“我打中了、伤害 999”直接当作可信事实。

---

<a id="section-13"></a>
## 13. AI：从感知目标到激活攻击能力

Aura 当前使用经典的 UE AI 技术栈：

```text
AIController：控制入口
Behavior Tree：决策流程
Blackboard：决策共享内存
Service：周期更新信息
Task：执行一个动作
Decorator：判断分支能否执行
EQS：在多个空间候选点中查询和评分
GAS：真正执行攻击
```

### 13.1 启动链

```text
服务器生成 Enemy
  -> AIController Possess Enemy
  -> 初始化 Blackboard
  -> Run Behavior Tree
  -> Service 查找最近玩家
  -> 写入 TargetToFollow / DistanceToTarget
  -> Selector 根据条件选择追击或攻击
```

Behavior Tree 不应该自己实现伤害公式。它负责“现在该做什么”；GameplayAbility 负责“攻击如何执行”；GameplayEffect/ExecCalc 负责“攻击造成什么数值结果”。

### 13.2 EQS 的意义

远程敌人不是简单向目标直线冲刺。EQS 可以：

1. 围绕目标或自身生成候选点；
2. 用 Trace 排除被墙遮挡的点；
3. 用 Distance 等测试评分；
4. 选择最佳位置写入 Blackboard；
5. Behavior Tree MoveTo 该位置后攻击。

这把复杂空间选择从 Behavior Tree 的流程节点中分离出来，便于可视化调试和参数调整。

---

<a id="section-14"></a>
## 14. 近战与远程攻击完整闭环

### 14.1 共同前半段

```text
Behavior Tree Attack Task
  -> TryActivateAbilitiesByTag(Abilities.Attack)
  -> ASC 找到职业已授予的攻击 Ability
  -> Ability 读取 CombatTarget
  -> 选择 TaggedMontage
  -> 设置 Motion Warping 目标
  -> 播放攻击蒙太奇
  -> AnimNotify / Gameplay Event 进入命中时刻
```

职业数据决定敌人获得近战还是远程攻击能力，所以 BT 可以保持通用。

### 14.2 近战后半段

```text
攻击蒙太奇事件
  -> 根据 CombatSocket Tag 取得武器/手部 Socket
  -> 检测近战目标
  -> 构造并应用 Damage Effect
  -> ExecCalc -> IncomingDamage -> Health
```

Motion Warping 让同一段动画根据目标位置调整根运动，减少“挥空但逻辑命中”或“角色瞬移到固定动画落点”的违和感。

### 14.3 远程后半段

```text
攻击蒙太奇事件
  -> 武器 Socket 取得出生点
  -> 朝 CombatTarget 计算方向
  -> 服务器生成 Projectile
  -> Projectile 碰撞目标
  -> 应用 Damage Effect
  -> ExecCalc -> IncomingDamage -> Health
```

Weapon AnimBP 负责武器骨骼的表现同步；GameplayAbility 和 Projectile 负责权威玩法。两者都由蒙太奇事件对齐到正确的发射帧。

---

<a id="section-15"></a>
## 15. C++、蓝图与数据资产应如何分工

| 工具 | 最适合承担 | Aura 中的例子 |
|---|---|---|
| C++ | 底层规则、网络、通用算法、类型安全接口 | ASC、AttributeSet、ExecCalc、Controller |
| Blueprint | 资产组合、动画时序、表现、快速关卡配置 | BT 资产、Widget、Ability 子类、AnimBP |
| DataAsset | 一组有明确类型的数据配置 | CharacterClassInfo、AttributeInfo、InputConfig |
| CurveTable | 随等级变化的连续成长系数 | 属性成长、护甲/暴击系数 |
| GameplayTag | 跨系统公共语义和查询键 | Input、Damage、Ability、Message |
| Interface | 多种 Actor 共享行为协议 | CombatInterface、EnemyInterface |

一个实用分界是：

- 如果改动需要重新编译才能保证类型和规则正确，倾向 C++；
- 如果是动画、特效、音效和资产引用，倾向蓝图；
- 如果设计师需要批量调数值，倾向 DataAsset/CurveTable；
- 如果多个系统只需共享“含义”而不应认识具体类，使用 Tag 或 Interface。

蓝图不是 C++ 的低级替代品。好的项目让 C++ 提供稳定能力，让蓝图组合内容；蓝图中若充满伤害公式、网络权限分支和重复查找，说明边界可能需要重新整理。

---

<a id="section-16"></a>
## 16. UE 对象生命周期：代码为什么不能随便放

| 阶段 | 适合做什么 | 常见风险 |
|---|---|---|
| C++ Constructor | 创建默认子对象、设置类默认值 | World、Controller、网络对象通常还不可用 |
| OnConstruction | 根据编辑器属性构造实例表现 | 可能在编辑器中多次执行 |
| BeginPlay | World 中开始运行后的初始化 | PlayerState 等复制数据可能尚未到位 |
| PossessedBy | 服务器 Pawn 被 Controller 占有 | 客户端不会靠它完成相同初始化 |
| OnRep_PlayerState | 客户端收到 PlayerState | 可能重复触发，初始化要考虑幂等性 |
| Tick | 连续更新必要状态 | 不应每帧做昂贵全局查找 |
| EndPlay / Destroyed | 解绑委托、停止任务、清理表现 | 忘记清理会留下悬空引用或重复回调 |

很多 UE Bug 不是算法错，而是“正确代码放在错误生命周期”。Aura 的 ASC 初始化同时处理 `PossessedBy` 和 `OnRep_PlayerState`，正是典型例子。

---

<a id="section-17"></a>
## 17. 一般 UE 项目每一帧如何运行

概念上的一帧包括：

```text
平台事件和输入采样
  -> PlayerController / InputComponent
  -> UWorld Tick
  -> Actor / Component Tick（受 Tick Group 约束）
  -> CharacterMovement / Physics
  -> AI、Behavior Tree、GameplayTasks、AbilityTask、Timer
  -> 网络复制与 RPC 处理
  -> Animation Update / Evaluate
  -> Slate / UMG
  -> Render Thread / RHI 提交画面
```

这不是严格的逐函数顺序。UE 会使用 Tick Group、任务图和游戏线程/渲染线程并行组织工作。但对项目调试而言，可以这样理解：输入改变意图，玩法系统改变 World 状态，复制传播权威结果，动画和 UI 消费状态，渲染显示最终画面。

Aura 中并非所有逻辑都应写进 Tick：

- 鼠标 Trace、自动寻路方向适合连续更新；
- 属性变化适合 Delegate；
- 持续效果适合 GameplayEffect Duration/Period；
- 延迟和等待动画事件适合 AbilityTask；
- AI 信息可由 BT Service 按合理间隔更新；
- 一次性初始化适合生命周期回调。

选择事件驱动还是逐帧轮询，会直接影响性能与代码清晰度。

---

<a id="section-18"></a>
## 18. 网络复制与 RPC 的判断框架

遇到多人问题时，按以下顺序判断：

1. **这段代码正在服务器还是客户端执行？** 使用 Authority、LocallyControlled 等条件确认。
2. **谁拥有这个 Actor？** Client RPC 和 Server RPC 都依赖正确的网络所有权链。
3. **这是持久状态还是瞬时事件？** 持久状态优先复制属性；瞬时表现可使用 RPC、GameplayCue 或复制 Actor。
4. **谁有权修改结果？** 伤害、生成、能力授予等应由服务器决定。
5. **客户端是否需要预测？** 输入和技能手感可预测，但服务器仍需校验并最终确认。

几个具体例子：

- Health：复制的持久状态；
- 鼠标 HitResult：客户端采集，通过 TargetData 发送；
- Projectile：服务器生成并复制；
- 飘字：服务器确认伤害后通知相关客户端显示；
- HUD Widget：每个本地客户端自己创建，不把 Widget 复制到网络。

---

<a id="section-19"></a>
## 19. 如何从源码读懂一次功能

不要只沿着文件夹顺序阅读，应该沿“事件链”阅读。例如分析一次火球攻击：

```text
输入 Action 在哪里绑定？
  -> InputTag 怎样到 ASC？
  -> 哪个 AbilitySpec 匹配？
  -> Ability 如何获得鼠标 TargetData？
  -> 谁在服务器生成 Projectile？
  -> Projectile 如何找到目标 ASC？
  -> Damage Spec 写入了哪些 SetByCaller？
  -> ExecCalc 捕获了哪些属性？
  -> IncomingDamage 在哪里转成 Health？
  -> 死亡、受击和 UI 从哪里收到事件？
```

推荐调试工具链：

- C++ 断点：观察 `PossessedBy`、`OnRep_PlayerState`、能力激活和伤害执行；
- `showdebug abilitysystem` / GAS 调试信息：查看 Ability、Effect、Tag；
- Gameplay Debugger：观察 AI、目标和导航；
- Behavior Tree Debugger：查看当前运行节点和 Blackboard；
- EQS Testing Pawn：查看候选点、过滤和评分；
- `stat game`、Unreal Insights：定位 Tick 和性能开销；
- 多人 PIE：至少用 Listen Server + Client 检查权限与复制；
- Network Emulation：模拟延迟、丢包，验证预测和表现。

测试时不要只看“能不能打中”，还要观察服务器和客户端两边：是否重复生成、是否重复扣血、其他客户端是否能看到、晚加入客户端能否取得正确状态。

---

<a id="section-20"></a>
## 20. 当前架构做得好的地方

1. **ASC 放在 PlayerState**：为多人、重生和持久玩家状态打下了正确基础。
2. **Tag 驱动输入和能力**：减少具体类之间的硬编码依赖。
3. **WidgetController 隔离 UI**：属性变化、消息和初始状态有统一数据入口。
4. **ExecCalc 与 IncomingDamage 分层**：数学结算和伤害后事件职责清晰。
5. **自定义 EffectContext**：为复杂伤害结果和多人同步提供扩展点。
6. **AI 只决定行为，GAS 执行攻击**：玩家与 AI 可以共享同一套能力和伤害规则。
7. **DataAsset + CurveTable**：职业、属性和等级成长开始数据驱动。
8. **C++ 与蓝图互补**：底层系统稳定，表现和资产组合仍有足够迭代速度。

---

<a id="section-21"></a>
## 21. 当前需要关注的技术债

这些不是“项目做错了”，而是从教学项目继续走向完整项目时值得逐步处理的边界：

- 默认 GameMode 配置仍依赖地图/蓝图覆盖，入口较分散；
- Content 若被 Git 忽略，蓝图、动画、DataAsset 和地图难以审计、备份与复现；
- Blackboard Key 使用硬编码 `FName` 时容易拼写错误，可集中为常量或可配置 Key Selector；
- AI Service 使用全局 Actor 查找，在大场景中成本会升高，可转向感知系统或缓存；
- 伤害 Spec、Curve、ClassInfo、CombatInterface 等关键依赖应补充有效性检查；
- 敌我判断目前若主要依赖 Actor Tag，需要逐步形成明确的 Team/Faction 规则；
- Projectile 的 Owner、Instigator、友伤和销毁语义要在多人情况下持续验证；
- UI 初始化应验证重生、重新 Possess 和切图路径，避免重复绑定 Delegate；
- PlayerController 同时承担输入、Cursor Trace、寻路和自动移动，后期可拆成组件或服务；
- 伤害 Tag 中 `Lighting`/`Lightning` 等命名应尽早统一，避免成为长期协议；
- GameState、GameInstance、SaveGame、关卡流转和存档体系尚未形成完整闭环；
- 死亡目标的 AI 引用清理、远程弹道边界和攻击取消仍需进一步完善。

处理优先级建议是：先保证多人正确性和资产可追踪，再补生命周期边界与空指针防御，最后做性能和架构拆分。

---

<a id="section-22"></a>
## 22. 下一阶段建议

### 22.1 先补齐完整 Gameplay Loop

```text
进入地图
  -> 生成玩家
  -> 探索和战斗
  -> 击杀敌人 / 获得奖励
  -> 经验与升级
  -> 技能解锁或升级
  -> 保存进度
  -> 切换关卡 / 死亡重生
```

系统数量不是完成度，能否形成循环才是。下一阶段每增加一个系统，都应确认它怎样进入这条 Loop。

### 22.2 建立可重复的测试矩阵

至少覆盖：

| 场景 | 需要验证 |
|---|---|
| Standalone | 基本流程和性能 |
| Listen Server | 服务器玩家技能、AI 和 UI |
| Client | 输入预测、TargetData、复制和 RPC |
| 两个客户端互相观察 | 投射物、受击、死亡和飘字是否一致 |
| 高延迟/丢包 | 重复激活、假命中、表现延迟 |
| 重生/切图 | ASC 重绑、Delegate、HUD 和属性初始化 |

### 22.3 逐步建立项目级文档

建议继续维护：

- 启动与对象生命周期图；
- GameplayTag 字典及生产者/消费者；
- 伤害公式和捕获属性表；
- 网络 Authority/RPC/Replication 清单；
- 蓝图资产与 C++ 基类映射；
- 每个重要流程的调试入口和测试用例。

---

<a id="section-23"></a>
## 23. 中期知识地图

```text
UE Engine
├── Module / Config / Plugin / AssetManager
├── World / Map / GameMode
├── Game Framework
│   ├── PlayerController：输入、本地控制、HUD
│   ├── PlayerState：长期玩家状态、ASC Owner
│   ├── Character：世界身体、ASC Avatar
│   └── HUD / Widget：本地表现
├── Enhanced Input
│   └── Action -> InputTag -> AbilitySpec
├── GAS
│   ├── ASC：能力、Effect、Tag、复制
│   ├── AttributeSet：持久属性与 Meta Attribute
│   ├── Ability：动作过程和 AbilityTask
│   ├── GameplayEffect：属性变化规则
│   ├── MMC：一个 Modifier 的派生值
│   ├── ExecCalc：一次复杂结算
│   └── EffectContext：一次 Effect 的上下文
├── UI
│   └── ASC/AS -> WidgetController -> Widget
├── AI
│   └── Controller -> BT/Blackboard/EQS -> GAS Ability
└── Network
    └── Server Authority + Replication + RPC + Prediction
```

---

<a id="section-24"></a>
## 24. 一句话总结整个项目

Aura 的核心不是某个火球、血条或行为树节点，而是：**UE 的 Game Framework 负责对象和生命周期，Enhanced Input 与 AI 产生意图，GAS 把意图变成可复制的能力、效果与属性变化，GameplayTag 充当跨系统协议，WidgetController、动画和特效再把权威游戏状态转化为玩家能看到的反馈。**

当你能沿着“谁创建对象、谁拥有数据、何时初始化、在哪一端执行、通过什么协议通信”这五个问题解释任意一个功能时，就不再是在记课程步骤，而是真正开始理解 UE 项目如何运行。

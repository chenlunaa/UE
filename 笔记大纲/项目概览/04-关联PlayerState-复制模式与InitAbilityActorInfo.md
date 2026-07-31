<a id="section-1"></a>
# UE5 学习笔记 — 第四次提交（续）

> 📦 Commit `3686998`：将 PlayerState 与 Aura 关联起来（设置复制模式 + 初始化 AbilityActorInfo）  
> 📅 日期：2026-06-05  
> 🎬 对应视频：3.7 设置复制模式 / 3.8 初始化 AbilityActorInfo

---

<a id="section-2"></a>
## 目录

- [UE5 学习笔记 — 第四次提交（续）](#section-1)
  - [目录](#section-2)
  - [一、GameplayEffect 复制模式](#section-3)
    - [1.1 什么是复制模式？](#section-4)
    - [1.2 三种复制模式详解](#section-5)
    - [1.3 本项目中的选择](#section-6)
    - [1.4 代码实现](#section-7)
  - [二、Owner Actor 与 Avatar Actor](#section-8)
    - [2.1 两个角色的概念](#section-9)
    - [2.2 敌人：Owner == Avatar](#section-10)
    - [2.3 玩家：Owner ≠ Avatar](#section-11)
  - [三、InitAbilityActorInfo — 初始化 ASC 的演员信息](#section-12)
    - [3.1 为什么需要调用这个函数？](#section-13)
    - [3.2 敌人的初始化（BeginPlay 中）](#section-14)
    - [3.3 玩家的初始化（PossessedBy + OnRep\_PlayerState）](#section-15)
    - [3.4 为什么玩家需要两个入口？](#section-16)
    - [3.5 DRY 原则：提取 InitAbilityActorInfo 私有函数](#section-17)
  - [四、PlayerState 与 Character 的指针桥接](#section-18)
    - [4.1 问题回顾](#section-19)
    - [4.2 桥接代码](#section-20)
    - [4.3 桥接后的完整数据流](#section-21)
  - [五、混合复制模式的注意事项](#section-22)
    - [5.1 Owner Actor 的 Owner 必须是 Controller](#section-23)
    - [5.2 PlayerState 天然满足条件](#section-24)
  - [六、新增 / 修改文件清单](#section-25)
  - [七、知识点总结](#section-26)

---

<a id="section-3"></a>
## 一、GameplayEffect 复制模式

<a id="section-4"></a>
### 1.1 什么是复制模式？

在多人游戏中，**GameplayEffect（GE）** 用于修改属性值（如造成伤害、治疗、Buff/Debuff）。当服务器上发生 GE 时，客户端也需要知道这些变化（比如更新血条显示）。

**复制模式**决定了 GE 信息如何从服务器同步到各个客户端。

`UAbilitySystemComponent` 提供了一个函数来设置复制模式：

```cpp
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::XXX);
```

<a id="section-5"></a>
### 1.2 三种复制模式详解

| 模式          | 枚举值       | GameplayEffect 复制 | GameplayCue 复制 | GameplayTag 复制 | 适用场景     |
| ----------- | --------- | ----------------- | -------------- | -------------- | -------- |
| **Full**    | `Full`    | 复制给**所有**客户端      | ✅ 所有           | ✅ 所有           | 单人游戏     |
| **Mixed**   | `Mixed`   | 只复制给**拥有者**客户端    | ✅ 所有           | ✅ 所有           | 玩家控制的角色  |
| **Minimal** | `Minimal` | **不复制** GE        | ✅ 所有           | ✅ 所有           | AI 控制的角色 |

**逐条解读：**

- **Full（完全复制）**：GE 会复制给所有客户端。这在单人游戏中没问题，但在多人游戏中会造成大量不必要的网络流量——你不需要知道其他玩家的每个 Buff 细节。

- **Mixed（混合复制）**：GE 只复制给"拥有者"客户端（即该 ASC 所属的玩家）。但 GameplayCue（视觉/音效提示）和 GameplayTag 仍然复制给所有客户端。这样：
  
  - 你自己的血条能实时更新（因为 GE 复制给了你）
  - 其他玩家也能看到你身上的特效（因为 GameplayCue 复制给了所有人）

- **Minimal（最小复制）**：GE 完全不复制。适用于 AI 控制的敌人——服务器上计算伤害就够了，客户端不需要知道敌人每个属性的精确变化。但 GameplayCue 和 GameplayTag 仍然会复制（这样客户端能看到敌人的受击特效等）。

> ⚠️ **重要注释（来自引擎源码）**：`Minimal` 模式不适用于"拥有的 ASC"（owned ASC）。对于玩家拥有的 ASC，请使用 `Mixed` 代替。

<a id="section-6"></a>
### 1.3 本项目中的选择

根据 Epic 官方推荐和课程规则：

| 角色类型                        | 复制模式      | 原因                                |
| --------------------------- | --------- | --------------------------------- |
| **玩家控制的角色**（AAuraCharacter） | `Mixed`   | ASC 在 PlayerState 上，玩家需要看到自己的属性变化 |
| **AI 控制的敌人**（AAuraEnemy）    | `Minimal` | ASC 在 Character 上，敌人属性不需要同步给客户端   |

<a id="section-7"></a>
### 1.4 代码实现

**AAuraPlayerState 构造函数（玩家 → Mixed）：**

```cpp
AAuraPlayerState::AAuraPlayerState()
{
    AbilitySystemComponent = CreateDefaultSubobject<UMy_AuraAbilitySystemComponent>("AbilitySystemComponent");
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);  // ← 新增

    AttributeSet = CreateDefaultSubobject<UAuraAttributeSet>("AttributeSet");
    NetUpdateFrequency = 100.f;
}
```

**AAuraEnemy 构造函数（敌人 → Minimal）：**

```cpp
AAuraEnemy::AAuraEnemy()
{
    GetMesh()->SetCollisionResponseToChannel(ECC_Visibility, ECR_Block);

    AbilitySystemComponent = CreateDefaultSubobject<UMy_AuraAbilitySystemComponent>("AbilitySystemComponent");
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);  // ← 新增

    AttributeSet = CreateDefaultSubobject<UAuraAttributeSet>("AttributeSet");
}
```

> 🎯 **设计理念**：多人游戏必须从头设计网络架构。为多人游戏编写的代码可以在单人模式下正常运行，但反过来不行——单人游戏的代码放到多人环境中通常无法正常工作。

---

<a id="section-8"></a>
## 二、Owner Actor 与 Avatar Actor

<a id="section-9"></a>
### 2.1 两个角色的概念

`UAbilitySystemComponent` 内部维护两个重要的 Actor 引用：

| 概念                     | 含义                                 | 类比   |
| ---------------------- | ---------------------------------- | ---- |
| **Owner Actor**（所有者演员） | 实际**拥有**该 ASC 的 Actor，负责 ASC 的生命周期 | "大脑" |
| **Avatar Actor**（化身演员） | ASC 在游戏世界中的**物理表示**，即你在屏幕上看到的角色    | "身体" |

`InitAbilityActorInfo` 函数的签名：

```cpp
void InitAbilityActorInfo(AActor* InOwnerActor, AActor* InAvatarActor);
```

<a id="section-10"></a>
### 2.2 敌人：Owner == Avatar

对于 AI 控制的敌人，ASC 直接放在 Character 上：

```
AAuraEnemy（敌人类）
  ├── 拥有 ASC（Owner = this）
  └── 也是 ASC 的 Avatar（Avatar = this）
```

```cpp
// AAuraEnemy::BeginPlay()
AbilitySystemComponent->InitAbilityActorInfo(this, this);
//                                            ↑     ↑
//                                         Owner  Avatar
//                                       （同一个 Actor）
```

<a id="section-11"></a>
### 2.3 玩家：Owner ≠ Avatar

对于玩家控制的角色，ASC 放在 PlayerState 上：

```
AAuraPlayerState（玩家状态）── 拥有 ASC（Owner）
AAuraCharacter（角色）      ── ASC 的 Avatar（化身）
```

```cpp
// AAuraCharacter::InitAbilityActorInfo()
AuraPlayerState->GetAbilitySystemComponent()->InitAbilityActorInfo(AuraPlayerState, this);
//                                                                  ↑               ↑
//                                                               Owner           Avatar
//                                                          (PlayerState)    (Character)
```

> 💡 **为什么这样设计？** PlayerState 在角色死亡/重生后依然存在，所以 ASC 和属性数据不会丢失。但游戏世界中实际移动、播放动画的是 Character，所以 Character 是 Avatar。

---

<a id="section-12"></a>
## 三、InitAbilityActorInfo — 初始化 ASC 的演员信息

<a id="section-13"></a>
### 3.1 为什么需要调用这个函数？

在 GAS 框架内部，很多功能需要知道"谁是 ASC 的所有者"和"谁是 ASC 的化身"。例如：

- 应用 GE 时需要知道目标 Actor
- 能力激活时需要知道是哪个 Actor 在执行
- 网络复制时需要知道数据发给谁

如果不调用 `InitAbilityActorInfo`，这些功能将无法正常工作。

<a id="section-14"></a>
### 3.2 敌人的初始化（BeginPlay 中）

敌人是最简单的情况——ASC 就在自己身上，在 `BeginPlay` 中直接初始化即可：

```cpp
// AuraEnemy.h — 新增
protected:
    virtual void BeginPlay() override;
```

```cpp
// AuraEnemy.cpp
void AAuraEnemy::BeginPlay()
{
    Super::BeginPlay();
    AbilitySystemComponent->InitAbilityActorInfo(this, this);
}
```

**为什么在 BeginPlay 中调用就够了？**

- 敌人是 AI 控制的，不涉及"玩家登录 → 创建 PlayerState → Possess Pawn"这个复杂流程
- `BeginPlay` 时，敌人的 Controller（AIController）已经设置好了
- Owner 和 Avatar 都是 `this`，不需要等待任何外部依赖

<a id="section-15"></a>
### 3.3 玩家的初始化（PossessedBy + OnRep_PlayerState）

玩家的情况复杂得多，需要在**两个不同的入口**调用初始化：

```cpp
// AuraCharacter.h — 新增
public:
    virtual void PossessedBy(AController* NewController) override;
    virtual void OnRep_PlayerState() override;
private:
    void InitAbilityActorInfo();
```

```cpp
// AuraCharacter.cpp
void AAuraCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    // 服务器端调用
    InitAbilityActorInfo();
}

void AAuraCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();
    // 客户端调用
    InitAbilityActorInfo();
}

void AAuraCharacter::InitAbilityActorInfo()
{
    AAuraPlayerState* AuraPlayerState = GetPlayerState<AAuraPlayerState>();
    check(AuraPlayerState);
    AuraPlayerState->GetAbilitySystemComponent()->InitAbilityActorInfo(AuraPlayerState, this);
    AbilitySystemComponent = AuraPlayerState->GetAbilitySystemComponent();
    AttributeSet = AuraPlayerState->GetAttributeSet();
}
```

<a id="section-16"></a>
### 3.4 为什么玩家需要两个入口？

这是一个经典的 UE 多人游戏模式——**服务器和客户端的初始化时机不同**：
**因为他们的生命周期不同：**
**玩家（Player）的生命周期：异步且分离
玩家登录时，连接是以 PlayerController 为主导的。**

* 服务器先生成 PlayerState 和 PlayerController。
* 服务器切地图或生成 Pawn，然后 Possess（附身）。
* 网络延迟导致步调不一致： 在客户端，Pawn 可能先被复制生成了，但此时 PlayerState 还没复制过来。如果你在客户端 Pawn 的 BeginPlay 里去初始化 GAS，由于拿不到 PlayerState，初始化就会直接失败。

**AI 的生命周期：一体化生成
AI 敌人通常是作为地图的一部分直接放置在关卡中，或者由服务器的 Spawner 统一 SpawnActor 出来的。**

* 服务器端： 当 AI Pawn 被 Spawn 时，它的 AIController 通常会立即（在同一个 CPU 帧内）被自动创建并完成 Possess。
* 客户端端： 客户端通过网络复制（Replication）收到这个 AI Pawn。对客户端而言，它收到的就是一个已经由服务器设定好所有基本状态的、完整的傀儡。 因此，当 AI 在客户端和服务器触发 BeginPlay 时，它的 Controller 已经牢牢绑定好了，没有玩家那种“断断续续”的同步等待期。

```
服务器端流程：
  玩家登录 → 创建 PlayerState → 创建 Pawn → PossessedBy() 被调用
                                            ↑
                                    此时 Controller 和 PlayerState 都已就绪
                                    → 在这里调用 InitAbilityActorInfo()

客户端流程：
  玩家登录 → 创建 PlayerState（服务器） → 复制到客户端
                                         ↓
                                   OnRep_PlayerState() 被触发（RepNotify）
                                         ↑
                                  此时 PlayerState 已复制完成，指针有效
                                  → 在这里调用 InitAbilityActorInfo()
```

| 入口函数                  | 调用时机                    | 调用端     | 前提条件                           |
| --------------------- | ----------------------- | ------- | ------------------------------ |
| `PossessedBy()`       | Pawn 被 Controller 接管时   | **服务器** | Controller 已设置，PlayerState 可访问 |
| `OnRep_PlayerState()` | PlayerState 从服务器复制到客户端时 | **客户端** | PlayerState 已复制完成，指针有效         |

> 🎯 **核心原则**：调用 `InitAbilityActorInfo` 之前，必须确保 Controller 和 PlayerState 都已就绪。

<a id="section-17"></a>
### 3.5 DRY 原则：提取 InitAbilityActorInfo 私有函数

注意 `PossessedBy` 和 `OnRep_PlayerState` 中做的事情完全一样，所以提取为私有函数 `InitAbilityActorInfo()`：

```cpp
private:
    void InitAbilityActorInfo();  // 避免代码重复
```

这遵循了 **DRY 原则（Don't Repeat Yourself）**——同样的逻辑只写一次，两个地方调用。

---

<a id="section-18"></a>
## 四、PlayerState 与 Character 的指针桥接

<a id="section-19"></a>
### 4.1 问题回顾

在第四次提交中，我们留下了一个"悬而未决"的问题：

```
AAuraCharacterBase 中有两个指针：
  - AbilitySystemComponent  → nullptr（对玩家角色）
  - AttributeSet            → nullptr（对玩家角色）

AAuraPlayerState 中也有两个指针：
  - AbilitySystemComponent  → 有效（在构造函数中创建）
  - AttributeSet            → 有效（在构造函数中创建）

问题：AAuraCharacter 的指针怎么指向 AAuraPlayerState 中的对象？
```

<a id="section-20"></a>
### 4.2 桥接代码

在 `InitAbilityActorInfo()` 中完成桥接：

```cpp
void AAuraCharacter::InitAbilityActorInfo()
{
    AAuraPlayerState* AuraPlayerState = GetPlayerState<AAuraPlayerState>();
    check(AuraPlayerState);

    // 1. 初始化 ASC 的 Owner/Avatar 信息
    AuraPlayerState->GetAbilitySystemComponent()->InitAbilityActorInfo(AuraPlayerState, this);

    // 2. 桥接：将 Character 的指针指向 PlayerState 中的对象
    AbilitySystemComponent = AuraPlayerState->GetAbilitySystemComponent();
    AttributeSet = AuraPlayerState->GetAttributeSet();
}
```

现在：

- `AAuraCharacter::AbilitySystemComponent` → 指向 PlayerState 中的 ASC ✅
- `AAuraCharacter::AttributeSet` → 指向 PlayerState 中的 AttributeSet ✅
- 通过 `IAbilitySystemInterface::GetAbilitySystemComponent()` 也能正确返回 ✅

<a id="section-21"></a>
### 4.3 桥接后的完整数据流

```
┌─────────────────────────────────────────────────┐
│                  AAuraPlayerState                 │
│  ┌─────────────────────────────────────────────┐ │
│  │ UMy_AuraAbilitySystemComponent (有效实例)    │ │
│  │ UAuraAttributeSet (有效实例)                 │ │
│  └──────────────┬──────────────────────────────┘ │
│                 │                                 │
└─────────────────┼─────────────────────────────────┘
                  │ 指针桥接
                  ↓
┌─────────────────┼─────────────────────────────────┐
│  AAuraCharacter  │                                 │
│  ┌───────────────┴──────────────────────────────┐ │
│  │ AbilitySystemComponent ──→ 指向 PlayerState  │ │
│  │ AttributeSet           ──→ 指向 PlayerState  │ │
│  └──────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────┘
```

---

<a id="section-22"></a>
## 五、混合复制模式的注意事项

<a id="section-23"></a>
### 5.1 Owner Actor 的 Owner 必须是 Controller

当使用 `Mixed` 复制模式时，有一个**硬性要求**：

> **Owner Actor 的 Owner 必须是 Controller。**

这意味着 `InitAbilityActorInfo(InOwnerActor, InAvatarActor)` 中的 `InOwnerActor` 必须是一个其 `GetOwner()` 返回有效 Controller 的 Actor。

<a id="section-24"></a>
### 5.2 PlayerState 天然满足条件

在本项目中，Owner Actor 是 `AAuraPlayerState`：

- `APlayerState` 的 Owner 会被引擎**自动设置**为对应的 `APlayerController`
- 同样，`APawn` 被 `PossessedBy()` 时，其 Owner 也会自动设置为 Controller

所以本项目中不需要手动调用 `SetOwner()`——引擎已经帮我们处理好了。

> ⚠️ **注意**：如果你的项目中使用了一个**不是 PlayerState 也不是 Pawn** 的类作为 Owner Actor，并且使用了 Mixed 复制模式，你需要手动调用 `InOwnerActor->SetOwner(Controller)` 来满足这个要求。

---

<a id="section-25"></a>
## 六、新增 / 修改文件清单

| 文件                                                | 操作    | 说明                                                             |
| ------------------------------------------------- | ----- | -------------------------------------------------------------- |
| `Source/Aura/Public/Character/AuraCharacter.h`    | ✏️ 修改 | 添加 `PossessedBy`、`OnRep_PlayerState`、`InitAbilityActorInfo` 声明 |
| `Source/Aura/Private/Character/AuraCharacter.cpp` | ✏️ 修改 | 实现上述三个函数，完成 ASC 初始化和指针桥接                                       |
| `Source/Aura/Public/Character/AuraEnemy.h`        | ✏️ 修改 | 添加 `BeginPlay` 声明，整理接口注释                                       |
| `Source/Aura/Private/Character/AuraEnemy.cpp`     | ✏️ 修改 | 实现 `BeginPlay` 初始化 ASC；设置 Minimal 复制模式                         |
| `Source/Aura/Private/Player/AuraPlayerState.cpp`  | ✏️ 修改 | 设置 Mixed 复制模式                                                  |

---

<a id="section-26"></a>
## 七、知识点总结

| 知识点                             | 要点                                                                             |
| ------------------------------- | ------------------------------------------------------------------------------ |
| **三种复制模式**                      | Full（单人）、Mixed（玩家角色，GE 只给拥有者）、Minimal（AI，GE 不复制）                               |
| **复制模式选择规则**                    | 玩家 → Mixed；AI 敌人 → Minimal；不要对拥有的 ASC 使用 Minimal                               |
| **Owner Actor vs Avatar Actor** | Owner = 拥有 ASC 的 Actor（大脑）；Avatar = ASC 在世界中的表示（身体）                            |
| **敌人 ASC 初始化**                  | `BeginPlay` 中调用 `InitAbilityActorInfo(this, this)`，简单直接                        |
| **玩家 ASC 初始化**                  | 两个入口：`PossessedBy`（服务器）+ `OnRep_PlayerState`（客户端）                              |
| **PossessedBy**                 | 服务器端 Pawn 被 Controller 接管时调用，此时 PlayerState 已就绪                                |
| **OnRep_PlayerState**           | 客户端收到复制的 PlayerState 时触发的 RepNotify                                            |
| **指针桥接**                        | 在 `InitAbilityActorInfo` 中将 Character 的 ASC/AttributeSet 指针指向 PlayerState 中的实例 |
| **DRY 原则**                      | 提取 `InitAbilityActorInfo()` 私有函数，避免在 PossessedBy 和 OnRep_PlayerState 中重复代码     |
| **Mixed 模式要求**                  | Owner Actor 的 Owner 必须是 Controller；PlayerState 自动满足此条件                         |
| **多人游戏设计**                      | 必须从一开始就按多人模式设计；为多人编写的代码可在单人下运行，反之不行                                            |

---

> 📝 **下一步预告**：开始定义具体的 Gameplay Attributes（生命值、魔法值、攻击力等），让 AttributeSet 不再是空壳。

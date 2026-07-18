# UE5 学习笔记 — 第四次提交

> 📦 Commit `222ee73`：添加 AbilitySystemComponent 和 AttributeSet  
> 📅 日期：2026-06-04  
> 🎬 对应视频：3.3 创建 PlayerState / 3.4 创建 ASC 和 AttributeSet 类 / 3.6 将 ASC 和 AttributeSet 添加到角色

---

## 目录

- [UE5 学习笔记 — 第四次提交](#ue5-学习笔记--第四次提交)
  - [目录](#目录)
  - [一、GAS（Gameplay Ability System）概述](#一gasgameplay-ability-system概述)
    - [1.1 什么是 GAS？](#11-什么是-gas)
    - [1.2 GAS 的核心组件](#12-gas-的核心组件)
    - [1.3 启用 Gameplay Abilities 插件](#13-启用-gameplay-abilities-插件)
  - [二、创建 PlayerState（玩家状态）](#二创建-playerstate玩家状态)
    - [2.1 为什么需要自定义 PlayerState？](#21-为什么需要自定义-playerstate)
    - [2.2 AAuraPlayerState 的 C++ 实现](#22-aauraplayerstate-的-c-实现)
    - [2.3 NetUpdateFrequency 详解](#23-netupdatefrequency-详解)
    - [2.4 蓝图配置](#24-蓝图配置)
  - [三、创建 AbilitySystemComponent 和 AttributeSet 类](#三创建-abilitysystemcomponent-和-attributeset-类)
    - [3.1 类的命名规范](#31-类的命名规范)
    - [3.2 UMy_AuraAbilitySystemComponent（自定义 ASC）](#32-umy_auraabilitysystemcomponent自定义-asc)
    - [3.3 UAuraAttributeSet（自定义属性集）](#33-uauraattributeset自定义属性集)
  - [四、Build.cs 模块依赖配置](#四buildcs-模块依赖配置)
    - [4.1 模块依赖调整](#41-模块依赖调整)
    - [4.2 Public vs Private 依赖的区别](#42-public-vs-private-依赖的区别)
  - [五、将 ASC 和 AttributeSet 添加到角色体系](#五将-asc-和-attributeset-添加到角色体系)
    - [5.1 两种拥有者模式](#51-两种拥有者模式)
    - [5.2 AuraCharacterBase 添加指针](#52-auracharacterbase-添加指针)
    - [5.3 AAuraEnemy 中构造 ASC 和 AttributeSet](#53-aauraenemy-中构造-asc-和-attributeset)
    - [5.4 AAuraPlayerState 中构造 ASC 和 AttributeSet](#54-aauraplayerstate-中构造-asc-和-attributeset)
  - [六、实现 IAbilitySystemInterface 接口](#六实现-iabilitysysteminterface-接口)
    - [6.1 接口的作用](#61-接口的作用)
    - [6.2 AuraCharacterBase 实现接口](#62-auracharacterbase-实现接口)
    - [6.3 AAuraPlayerState 也实现接口](#63-aauraplayerstate-也实现接口)
    - [6.4 GetAttributeSet 辅助获取器](#64-getattributeset-辅助获取器)
  - [七、新增 / 修改文件清单](#七新增--修改文件清单)
  - [八、知识点总结](#八知识点总结)

---

## 一、GAS（Gameplay Ability System）概述

### 1.1 什么是 GAS？

**Gameplay Ability System（GAS）** 是虚幻引擎内置的一套强大框架，专门用于构建复杂的游戏玩法逻辑。它被广泛应用于 AAA 级游戏（如《堡垒之夜》《Paragon》等），提供了：

- **属性系统**（Attributes）：管理角色的生命值、魔法值、攻击力等数值
- **能力系统**（Abilities）：定义角色可以执行的动作（技能、攻击、跳跃等）
- **效果系统**（Effects）：处理增益/减益（Buff/Debuff）、伤害计算等
- **标签系统**（Tags）：用层级标签来标记和查询游戏状态

### 1.2 GAS 的核心组件

| 组件                           | 类名                        | 作用                            | 数量关系                                              |
| ---------------------------- | ------------------------- | ----------------------------- | ------------------------------------------------- |
| **Ability System Component** | `UAbilitySystemComponent` | GAS 的大脑，管理所有能力的激活、效果的施加、属性的修改 | 一个 Actor 只能有一个 ASC                                |
| **Attribute Set**            | `UAttributeSet`           | 属性的容器，存储角色的各种数值（生命、魔法、攻击力等）   | 一个 ASC 可以挂载多个 不同的 AttributeSet（例如：基础属性集 + 战利品属性集） |

> 💡 **一句话理解**：ASC 是"管理者"，AttributeSet 是"数据仓库"。
> 
> 你永远是在**用 ASC 驱动 GameplayEffect**，去**修改 AttributeSet 里的属性值**。

例子 ：我们来看一个最常见的场景：**玩家受到 20 点火属性伤害**。

1. **触发**：一个火球术技能（GameplayAbility）命中玩家，生成了一个 `GameplayEffect`（暂且叫 `GE_FireDamage`，里面写着“造成 20 点伤害”）。

2. **ASC 出场**：火球把这个 `GE_FireDamage` 投递给玩家的 **ASC**。

3. **ASC 处理**：**ASC** 检查玩家身上有没有“火免疫”的 Tag。如果没有，ASC 开始计算这个 Effect。

4. **走向 AttributeSet**：**ASC** 发现这个 Effect 的目的是修改“生命值（Health）”，于是它转身对挂载在自己身上的 **AttributeSet** 说：“账本，把里面的 Health 减去 20”。

5. **AttributeSet 拦截与规范**：**AttributeSet** 收到指令，在实际扣钱（血）前进行校对（在 `PreAttributeChange` 或 `PostGameplayEffectExecute` 中）：
   
   - *“等等，玩家防具提供了 5 点减伤，实际只扣 15 点。”*
   
   - *“扣完 15 点后，Health 变成 -5 了，不合规，我得把它钳制（Clamp）到最小边界 0。”*

6. **同步**：**AttributeSet** 把最终修改好的 `Health = 0` 写入内存，并自动通过网络同步给所有客户端。

### 1.3 启用 Gameplay Abilities 插件

GAS 是 UE5 的一个**插件**，默认不启用，需要手动开启：

1. 打开 **Edit → Plugins**
2. 搜索 **"Gameplay Abilities"**
3. 勾选复选框
4. **重启编辑器**

---

## 二、创建 PlayerState（玩家状态）

### 2.1 为什么需要自定义 PlayerState？

在 GAS 架构中，**玩家控制的角色**（如主角）的 ASC 和 AttributeSet 通常放在 **PlayerState** 上，而不是直接放在 Character 上。原因：

| 放在 Character 上     | 放在 PlayerState 上       |
| ------------------ | ---------------------- |
| 角色死亡/销毁时 ASC 丢失    | PlayerState 在角色重生后仍然存在 |
| 不适合 MOBA/RPG 的复活机制 | 支持角色死亡后保留属性和能力         |
| 网络同步不够高效           | PlayerState 专为网络复制设计   |

> 🎯 这是 Epic 官方推荐的做法，Lyra 示例项目和《堡垒之夜》都采用这种架构。

而对于 **AI 控制的敌人**，ASC 和 AttributeSet 直接放在 Character 上即可——敌人不需要重生保留状态。

### 2.2 AAuraPlayerState 的 C++ 实现

**头文件** `Source/Aura/Public/Player/AuraPlayerState.h`：

- **`GetAbilitySystemComponent()`**：是为了遵守 GAS 框架的统一契约（接口），实现跨模块的通用访问。

- **成员变量 `AbilitySystemComponent`**：是当前类用来实际存储该组件的载体。
  通过虚函数将两者结合，既满足了引擎的接口规范，又保留了未来代码扩展的灵活性。

```cpp
class AURA_API AAuraPlayerState : public APlayerState, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    AAuraPlayerState();
    // 这里是实现IAbilitySystemInterface的抽象函数
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    // 所以这里没有IAttributeSetFace不用定义接口去实现多态
    UAttributeSet* GetAttributeSet() const { return AttributeSet; }

protected:
    UPROPERTY()
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    UPROPERTY()
    TObjectPtr<UAttributeSet> AttributeSet; // 专门做给Aura的
};
```

**实现** `Source/Aura/Private/Player/AuraPlayerState.cpp`：

```cpp
AAuraPlayerState::AAuraPlayerState()
{
    AbilitySystemComponent = CreateDefaultSubobject<UMy_AuraAbilitySystemComponent>("AbilitySystemComponent");
    AbilitySystemComponent->SetIsReplicated(true);

    AttributeSet = CreateDefaultSubobject<UAuraAttributeSet>("AttributeSet");

    NetUpdateFrequency = 100.f;
}

UAbilitySystemComponent* AAuraPlayerState::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}
```

### 2.3 NetUpdateFrequency 详解

```cpp
NetUpdateFrequency = 100.f;
```

| 属性                   | 默认值            | 本项目值   | 含义                        |
| -------------------- | -------------- | ------ | ------------------------- |
| `NetUpdateFrequency` | ~2 Hz（约 0.5 秒） | 100 Hz | 服务器向客户端同步 PlayerState 的频率 |

**为什么设为 100？**

- 默认的 `NetUpdateFrequency` 很低（约每秒 2 次），适合缓慢变化的玩家数据
- 但 GAS 的属性变化（如血量、魔法值）需要**即时同步**，否则会出现明显的延迟感
- Lyra 项目和《堡垒之夜》中，PlayerState 的 `NetUpdateFrequency` 也设为 100

> ⚠️ 服务器会**尽量**满足这个频率，但受网络带宽和 CPU 限制，实际频率可能低于设定值。

### 2.4 蓝图配置

1. 基于 `AAuraPlayerState` 创建蓝图类 `BP_AuraPlayerState`，放在 `Content/Blueprints/Player/`
2. 打开 `BP_AuraGameMode`，将 **Player State Class** 设为 `BP_AuraPlayerState`
3. 编译并保存

---

## 三、创建 AbilitySystemComponent 和 AttributeSet 类

### 3.1 类的命名规范

在大型 UE 项目中，类名通常带有**项目前缀**，用于：

- 避免与引擎类或其他插件类**命名冲突**
- 一眼识别类的**所属项目**或**开发团队**

| 项目                    | 前缀     | 示例                            |
| --------------------- | ------ | ----------------------------- |
| 本项目（Aura）             | `Aura` | `UAuraAttributeSet`           |
| Lyra 示例               | `Lyra` | `ULyraAbilitySystemComponent` |
| Druid Mechanics（作者公司） | `DM`   | `UDMAbilitySystemComponent`   |

本项目已统一使用 `Aura` 前缀：`AAuraCharacter`、`AAuraPlayerController`、`AAuraPlayerState` 等。

### 3.2 UMy_AuraAbilitySystemComponent（自定义 ASC）

**头文件** `Source/Aura/Public/AbilitySystem/My_AuraAbilitySystemComponent.h`：

```cpp
UCLASS()
class AURA_API UMy_AuraAbilitySystemComponent : public UAbilitySystemComponent
{
    GENERATED_BODY()
};
```

**实现** `Source/Aura/Private/AbilitySystem/My_AuraAbilitySystemComponent.cpp`：

```cpp
#include "AbilitySystem/My_AuraAbilitySystemComponent.h"
```

> 📝 目前只是一个空子类，后续会在其中添加自定义逻辑（如能力授予、输入绑定等）。创建自定义 ASC 是最佳实践——不要直接使用引擎的 `UAbilitySystemComponent`。

### 3.3 UAuraAttributeSet（自定义属性集）

**头文件** `Source/Aura/Public/AbilitySystem/AuraAttributeSet.h`：

```cpp
UCLASS()
class AURA_API UAuraAttributeSet : public UAttributeSet
{
    GENERATED_BODY()
};
```

**实现** `Source/Aura/Private/AbilitySystem/AuraAttributeSet.cpp`：

```cpp
#include "AbilitySystem/AuraAttributeSet.h"
```

> 📝 同样是一个空子类，后续会在其中定义具体的属性（生命值、魔法值、攻击力等）。

---

## 四、Build.cs 模块依赖配置

### 4.1 模块依赖调整

在 `Source/Aura/Aura.Build.cs` 中做了以下调整：

**修改前：**

```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "Engine", "InputCore", "EnhancedInput"
});

PrivateDependencyModuleNames.AddRange(new string[] {
    "GameplayTags", "GameplayTasks", "NavigationSystem", "Niagara", "AIModule"
});
```

**修改后：**

```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "Engine", "InputCore", "EnhancedInput",
    "GameplayAbilities"   // ← 从 Private 移到 Public
});

PrivateDependencyModuleNames.AddRange(new string[] {
    "GameplayTags", "GameplayTasks"
    // 移除了 NavigationSystem, Niagara, AIModule
});
```

**关键变化：**

| 模块                  | 变化                   | 原因                                                                                                             |
| ------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------- |
| `GameplayAbilities` | Private → **Public** | `AAuraCharacterBase` 头文件中公开继承了 `IAbilitySystemInterface`，该接口定义在 `GameplayAbilities` 模块中，因此必须暴露给所有依赖 Aura 模块的代码 |
| `NavigationSystem`  | 移除                   | 当前阶段不需要 AI 寻路功能                                                                                                |
| `Niagara`           | 移除                   | 当前阶段不需要粒子特效                                                                                                    |
| `AIModule`          | 移除                   | 当前阶段不需要 AI 行为树等功能                                                                                              |

> 💡 按需添加模块依赖，减少不必要的编译时间和耦合。

### 4.2 Public vs Private 依赖的区别

| 依赖类型        | 可见性               | 典型用途               |
| ----------- | ----------------- | ------------------ |
| **Public**  | 头文件中的类型被外部模块使用时需要 | 基类、接口、公共数据结构所在的模块  |
| **Private** | 仅在 .cpp 实现文件中使用   | 内部实现细节、不需要暴露给外部的模块 |

**经验法则**：如果你的头文件中 `#include` 了某个模块的类型，那这个模块通常应该在 `PublicDependencyModuleNames` 中。

---

## 五、将 ASC 和 AttributeSet 添加到角色体系

### 5.1 两种拥有者模式

本项目采用**混合架构**：

```
┌─────────────────────────────────────────────┐
│              游戏角色体系                      │
├─────────────────┬───────────────────────────┤
│   玩家控制角色     │     AI 控制敌人             │
│  (AAuraCharacter) │   (AAuraEnemy)            │
├─────────────────┼───────────────────────────┤
│  ASC 在           │  ASC 在                    │
│  PlayerState 上   │  Character 自身上           │
├─────────────────┼───────────────────────────┤
│  优点：           │  优点：                     │
│  • 死亡/重生不丢   │  • 简单直接                 │
│    失 ASC 数据    │  • 敌人不需要重生保留状态     │
│  • 网络同步高效    │                           │
└─────────────────┴───────────────────────────┘
```

### 5.2 AuraCharacterBase 添加指针

在基类 `AAuraCharacterBase` 中添加两个指针，所有角色子类都会继承：

```cpp
// AuraCharacterBase.h — 新增内容
class UAbilitySystemComponent;
class UAttributeSet;

class AURA_API AAuraCharacterBase : public ACharacter, public IAbilitySystemInterface
{
    // ...

protected:
    UPROPERTY()
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    UPROPERTY()
    TObjectPtr<UAttributeSet> AttributeSet;
};
```

> ⚠️ 注意：这些指针在 `AAuraCharacterBase` 中**不初始化**。对于敌人，在 `AAuraEnemy` 构造函数中初始化；对于玩家，在 `AAuraPlayerState` 中初始化。基类只是提供了统一的访问入口。

### 5.3 AAuraEnemy 中构造 ASC 和 AttributeSet

```cpp
AAuraEnemy::AAuraEnemy()
{
    GetMesh()->SetCollisionResponseToChannel(ECC_Visibility, ECR_Block);

    // 在敌人自身构造 ASC 和 AttributeSet
    AbilitySystemComponent = CreateDefaultSubobject<UMy_AuraAbilitySystemComponent>("AbilitySystemComponent");
    AbilitySystemComponent->SetIsReplicated(true);

    AttributeSet = CreateDefaultSubobject<UAuraAttributeSet>("AttributeSet");
}
```

| 关键点                           | 说明                                                          |
| ----------------------------- | ----------------------------------------------------------- |
| `CreateDefaultSubobject<T>()` | 在构造函数中创建子对象（组件），这是 UE 的标准做法                                 |
| `SetIsReplicated(true)`       | 标记 ASC 需要网络复制，这样服务器上的能力激活和属性变化才会同步到客户端                      |
| 子对象名称                         | `"AbilitySystemComponent"` 和 `"AttributeSet"` 是内部名称，用于编辑器显示 |

### 5.4 AAuraPlayerState 中构造 ASC 和 AttributeSet

```cpp
AAuraPlayerState::AAuraPlayerState()
{
    AbilitySystemComponent = CreateDefaultSubobject<UMy_AuraAbilitySystemComponent>("AbilitySystemComponent");
    AbilitySystemComponent->SetIsReplicated(true);

    AttributeSet = CreateDefaultSubobject<UAuraAttributeSet>("AttributeSet");

    NetUpdateFrequency = 100.f;
}
```

> 🎯 **关键理解**：此时 `AAuraCharacter` 中的 `AbilitySystemComponent` 和 `AttributeSet` 指针仍然是 **`nullptr`**！需要在后续步骤中将 PlayerState 的指针"桥接"到 Character 上。

---

## 六、实现 IAbilitySystemInterface 接口

### 6.1 接口的作用

`IAbilitySystemInterface` 是 GAS 框架定义的一个关键接口，只有一个纯虚函数：

```cpp
// 引擎源码（简化）
class IAbilitySystemInterface
{
public:
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const = 0;
};
```

**为什么这个接口如此重要？**

- GAS 系统的许多功能（如 GameplayEffect 的应用、Ability 的授予）需要从一个 Actor 获取其 ASC
- 通过实现这个接口，任何类都可以通过 `Cast<IAbilitySystemInterface>(Actor)` 来获取 ASC
- 这是一个**解耦**的设计：调用者不需要知道 Actor 的具体类型，只需要知道它"有 ASC"

```cpp
// 使用示例
if (IAbilitySystemInterface* ASI = Cast<IAbilitySystemInterface>(SomeActor))
{
    UAbilitySystemComponent* ASC = ASI->GetAbilitySystemComponent();
    // 使用 ASC 做事情...
}
```

### 6.2 AuraCharacterBase 实现接口

```cpp
// AuraCharacterBase.h
class AURA_API AAuraCharacterBase : public ACharacter, public IAbilitySystemInterface
{
    // ...
public:
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    UAttributeSet* GetAttributeSet() const { return AttributeSet; }
};
```

```cpp
// AuraCharacterBase.cpp
UAbilitySystemComponent* AAuraCharacterBase::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}
```

> ⚠️ 对于玩家角色 `AAuraCharacter`，调用 `GetAbilitySystemComponent()` 会返回 `nullptr`，因为指针在 Character 中未初始化。需要在后续步骤中做桥接。

### 6.3 AAuraPlayerState 也实现接口

```cpp
// AuraPlayerState.h
class AURA_API AAuraPlayerState : public APlayerState, public IAbilitySystemInterface
{
    // ...
public:
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    UAttributeSet* GetAttributeSet() const { return AttributeSet; }
};
```

```cpp
// AuraPlayerState.cpp
UAbilitySystemComponent* AAuraPlayerState::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}
```

> 💡 在 PlayerState 上也实现 `IAbilitySystemInterface` 是一种**便利设计**——某些情况下通过 PlayerState 获取 ASC 比通过 Character 更方便。

### 6.4 GetAttributeSet 辅助获取器

虽然不是接口要求，但添加 `GetAttributeSet()` 获取器是一个好习惯：

```cpp
UAttributeSet* GetAttributeSet() const { return AttributeSet; }
```

这使得其他代码可以方便地获取属性集，而不需要手动从 ASC 中查找。

---

## 七、新增 / 修改文件清单

| 文件                                                                    | 操作    | 说明                                                  |
| --------------------------------------------------------------------- | ----- | --------------------------------------------------- |
| `Source/Aura/Aura.Build.cs`                                           | ✏️ 修改 | 调整模块依赖：GameplayAbilities 移到 Public，移除未使用的模块         |
| `Source/Aura/Public/AbilitySystem/My_AuraAbilitySystemComponent.h`    | ➕ 新增  | 自定义 ASC 类声明                                         |
| `Source/Aura/Private/AbilitySystem/My_AuraAbilitySystemComponent.cpp` | ➕ 新增  | 自定义 ASC 类实现                                         |
| `Source/Aura/Public/AbilitySystem/AuraAttributeSet.h`                 | ➕ 新增  | 自定义 AttributeSet 类声明                                |
| `Source/Aura/Private/AbilitySystem/AuraAttributeSet.cpp`              | ➕ 新增  | 自定义 AttributeSet 类实现                                |
| `Source/Aura/Public/Player/AuraPlayerState.h`                         | ➕ 新增  | PlayerState 类声明                                     |
| `Source/Aura/Private/Player/AuraPlayerState.cpp`                      | ➕ 新增  | PlayerState 类实现                                     |
| `Source/Aura/Public/Character/AuraCharacterBase.h`                    | ✏️ 修改 | 添加 ASC/AttributeSet 指针 + 实现 IAbilitySystemInterface |
| `Source/Aura/Private/Character/AuraCharacterBase.cpp`                 | ✏️ 修改 | 实现 GetAbilitySystemComponent()                      |
| `Source/Aura/Public/Character/AuraEnemy.h`                            | ✏️ 修改 | 添加注释                                                |
| `Source/Aura/Private/Character/AuraEnemy.cpp`                         | ✏️ 修改 | 在构造函数中创建 ASC 和 AttributeSet                         |
| `Source/Aura/Private/Player/AuraPlayerController.cpp`                 | ✏️ 修改 | 添加注释                                                |

---

## 八、知识点总结

| 知识点                         | 要点                                                          |
| --------------------------- | ----------------------------------------------------------- |
| **GAS 核心组件**                | ASC（能力系统组件）= 管理者；AttributeSet（属性集）= 数据仓库                    |
| **插件启用**                    | Gameplay Abilities 插件默认不启用，需在 Edit → Plugins 中手动开启          |
| **PlayerState 的 GAS 角色**    | 玩家 ASC 放在 PlayerState 上（支持重生保留），敌人 ASC 放在 Character 上（简单直接） |
| **NetUpdateFrequency**      | 默认 ~2Hz，GAS 项目建议设为 100Hz 以保证属性同步的即时性                        |
| **项目前缀命名**                  | 所有自定义类使用项目前缀（如 `Aura`），避免与引擎类冲突                             |
| **Public vs Private 依赖**    | 头文件中暴露的类型所在模块 → Public；仅在 .cpp 中使用的 → Private               |
| **CreateDefaultSubobject**  | 在构造函数中创建子对象的标准方式                                            |
| **SetIsReplicated(true)**   | 标记组件需要网络复制                                                  |
| **IAbilitySystemInterface** | GAS 的关键接口，提供统一的 ASC 获取方式，实现解耦                               |
| **混合架构**                    | 玩家 = PlayerState 持有 ASC；敌人 = Character 自身持有 ASC             |
| **指针桥接（待完成）**               | AAuraCharacter 的 ASC 指针需要从 PlayerState 桥接过来                 |

---

> 📝 **下一步预告**：将 PlayerState 中的 ASC 和 AttributeSet 桥接到 AAuraCharacter，并设置 ASC 的 Owner 和 Avatar Actor。

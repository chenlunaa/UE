# 使用 ATTRIBUTE_ACCESSORS 改变 AttributeSet 中的元素值

> 📦 Commit `c1759cb`：AttributeSet 属性声明 + 复制 + ATTRIBUTE_ACCESSORS 宏 + 构造函数初始化  
> 📅 日期：2026-06-06  
> 🎬 对应视频：第4章 4.1 / 4.2 / 4.3

---

## 目录

- [使用 ATTRIBUTE_ACCESSORS 改变 AttributeSet 中的元素值](#使用-attribute_accessors-改变-attributeset-中的元素值)
  - [目录](#目录)
  - [一、属性集（AttributeSet）深入理解](#一属性集attributeset深入理解)
    - [1.1 属性集与 ASC 的关系](#11-属性集与-asc-的关系)
    - [1.2 多个属性集 vs 单个属性集](#12-多个属性集-vs-单个属性集)
    - [1.3 什么是属性（Attribute）](#13-什么是属性attribute)
  - [二、客户端预测（Client Prediction）](#二客户端预测client-prediction)
    - [2.1 为什么需要预测](#21-为什么需要预测)
    - [2.2 无预测的工作流程（糟糕体验）](#22-无预测的工作流程糟糕体验)
    - [2.3 有预测的工作流程（流畅体验）](#23-有预测的工作流程流畅体验)
    - [2.4 GAS 内置预测的优势](#24-gas-内置预测的优势)
  - [三、属性的基础值与当前值](#三属性的基础值与当前值)
    - [3.1 基础值（Base Value）](#31-基础值base-value)
    - [3.2 当前值（Current Value）](#32-当前值current-value)
    - [3.3 最大值属性是独立的](#33-最大值属性是独立的)
  - [四、属性声明完整步骤（4.2 核心）](#四属性声明完整步骤42-核心)
    - [4.1 步骤总览](#41-步骤总览)
    - [4.2 第一步：声明变量](#42-第一步声明变量)
    - [4.3 第二步：创建 OnRep 通知函数](#43-第二步创建-onrep-通知函数)
    - [4.4 第三步：实现 OnRep 函数（GAMEPLAYATTRIBUTE_REPNOTIFY）](#44-第三步实现-onrep-函数gameplayattribute_repnotify)
    - [4.5 第四步：注册复制（GetLifetimeReplicatedProps）](#45-第四步注册复制getlifetimereplicatedprops)
    - [4.6 完整代码示例：Health + MaxHealth + Mana + MaxMana](#46-完整代码示例health--maxhealth--mana--maxmana)
  - [五、ATTRIBUTE_ACCESSORS 宏（4.3 核心）](#五attribute_accessors-宏43-核心)
    - [5.1 四个底层宏](#51-四个底层宏)
    - [5.2 ATTRIBUTE_ACCESSORS 的定义](#52-attribute_accessors-的定义)
    - [5.3 使用示例](#53-使用示例)
    - [5.4 在构造函数中初始化属性](#54-在构造函数中初始化属性)
  - [六、验证调试](#六验证调试)
    - [6.1 Show Debug Ability System 命令](#61-show-debug-ability-system-命令)
    - [6.2 调试信息解读](#62-调试信息解读)
  - [七、新增 / 修改文件清单](#七新增--修改文件清单)
  - [八、知识点总结](#八知识点总结)

---

## 一、属性集（AttributeSet）深入理解

### 1.1 属性集与 ASC 的关系

当我们在所有者 Actor 的构造函数内构造属性集时，它会**自动注册**到能力系统组件（ASC）。ASC 可以访问任何已注册的属性集。

```cpp
// 在 PlayerState 构造函数中创建 AttributeSet
// AttributeSet 会自动注册到 ASC，无需手动调用注册函数
```

### 1.2 多个属性集 vs 单个属性集

| 方案        | 说明                                                         |
| --------- | ---------------------------------------------------------- |
| **多个属性集** | 按类别划分（如 VitalAttributes、OffensiveAttributes），每个必须是**不同的类** |
| **单个属性集** | 将所有属性放在一个类中，更简单，尤其当属性间需要相互计算时                              |

> ⚠️ 不能有**两个相同类类型**的属性集注册到同一个 ASC，否则检索时会产生歧义。

本项目采用**单个属性集**策略，包含所有角色使用的属性。

### 1.3 什么是属性（Attribute）

- 属性是与游戏中给定实体（如角色）相关联的**数值量**
- 所有属性都是 **浮点数（float）**
- 它们存在于 `FGameplayAttributeData` 结构中
- 属性存储在属性集中，由属性集紧密监督
- 当属性发生变化时，可以以任何功能响应它

---

## 二、客户端预测（Client Prediction）

### 2.1 为什么需要预测

在多人游戏中，数据在网络上传输需要时间（延迟）。如果没有预测，客户端执行操作后需要等待服务器确认才能看到效果，导致明显的卡顿感。

### 2.2 无预测的工作流程（糟糕体验）

```
客户端请求更改 → [网络延迟] → 服务器接收 → 判断有效性 → [网络延迟] → 客户端收到确认 → 更改生效
```

- 延迟 100ms 或更长时间并不罕见
- 玩家执行操作后要等很久才能看到效果，体验极差

### 2.3 有预测的工作流程（流畅体验）

```
客户端立即更改属性（无需等待） → 同时发送请求给服务器
                                     ↓
服务器判断有效性 → 有效则通知其他客户端 → 无效则回滚更改
```

- 客户端**立即**感知到变化，没有延迟
- 服务器仍然保持**权威**（可以拒绝无效更改并回滚）
- 预测是多人游戏中创造更流畅体验的关键因素

### 2.4 GAS 内置预测的优势

GAS 将预测作为**内置功能**贯穿整个系统。我们不需要自己实现延迟补偿，可以专注于游戏机制的创造。

---

## 三、属性的基础值与当前值

每个属性实际上由**两个值**组成：

### 3.1 基础值（Base Value）

- 属性的**永久值**
- 不受临时效果影响

### 3.2 当前值（Current Value）

- 基础值 + 由游戏效果（Gameplay Effect）引起的**临时修改**
- 例如：一个 Buff 在 10 秒内增加 50 点力量，这 50 点就是临时修改
- 当 Buff 时间结束后，临时修改被取消，属性回到基础值

### 3.3 最大值属性是独立的

> ⚠️ **常见错误**：认为基础值就是属性的最大值。

正确做法：**最大值应该是自己独立的属性**（如 MaxHealth、MaxMana）。

```
Health 百分比 = Health / MaxHealth   ✅ 正确
Health 百分比 = Health / BaseHealth  ❌ 错误
```

这样可以对属性本身和最大属性**分别施加**游戏效果。

---

## 四、属性声明完整步骤（4.2 核心）

### 4.1 步骤总览

为属性集添加一个新属性需要以下 **4 个步骤**：

```
1. 声明变量（UPROPERTY + ReplicatedUsing）
2. 创建 OnRep 通知函数（UFUNCTION）
3. 实现 OnRep 函数（GAMEPLAYATTRIBUTE_REPNOTIFY）
4. 注册复制（DOREPLIFETIME_CONDITION_NOTIFY）
```

### 4.2 第一步：声明变量

在头文件中声明 `FGameplayAttributeData` 类型的属性，使用 `ReplicatedUsing` 标记复制：

```cpp
// AuraAttributeSet.h

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital Attributes")
FGameplayAttributeData Health;
```

关键点：

- `BlueprintReadOnly`：允许蓝图读取，但不允许修改
- `ReplicatedUsing = OnRep_Health`：当属性从服务器复制到客户端时，自动调用 `OnRep_Health` 函数
- `Category = "Vital Attributes"`：在蓝图中的分类名称

### 4.3 第二步：创建 OnRep 通知函数

```cpp
// AuraAttributeSet.h

UFUNCTION()
void OnRep_Health(const FGameplayAttributeData& OldHealth) const;
```

关键点：

- 必须是 `UFUNCTION()`（蓝图函数标记）
- 参数可以是空，或者接收旧值的 `const FGameplayAttributeData&`
- 如果提供了旧值参数，可以在函数中比较新旧值
- 可以标记为 `const`

### 4.4 第三步：实现 OnRep 函数（GAMEPLAYATTRIBUTE_REPNOTIFY）

```cpp
// AuraAttributeSet.cpp

void UAuraAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth) const
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UAuraAttributeSet, Health, OldHealth);
}
```

`GAMEPLAYATTRIBUTE_REPNOTIFY` 宏的作用：

- 通知 GAS 系统该属性已被复制
- 让 GAS 做底层账目记录以保持系统一致性
- 跟踪旧值，以便在需要时**回滚**更改（预测场景）

### 4.5 第四步：注册复制（GetLifetimeReplicatedProps）

```cpp
// AuraAttributeSet.h（声明）
virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

// AuraAttributeSet.cpp（实现）
void UAuraAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, Health, COND_None, REPNOTIFY_Always);
}
```

`DOREPLIFETIME_CONDITION_NOTIFY` 参数说明：

| 参数  | 值                   | 含义                                       |
| --- | ------------------- | ---------------------------------------- |
| 类名  | `UAuraAttributeSet` | 属性所属的类                                   |
| 属性名 | `Health`            | 要注册复制的变量                                 |
| 条件  | `COND_None`         | 无条件复制（不限制只复制给所有者等）                       |
| 通知  | `REPNOTIFY_Always`  | 即使值没变也复制（默认是 REPNOTIFY_OnChanged，值不变不复制） |

> 为什么用 `REPNOTIFY_Always`？因为即使将属性设置为与当前相同的值，我们也可能想对这个"设置行为"做出反应。

### 4.6 完整代码示例：Health + MaxHealth + Mana + MaxMana

**头文件（AuraAttributeSet.h）：**

```cpp
UCLASS()
class AURA_API UAuraAttributeSet : public UAttributeSet
{
    GENERATED_BODY()
public:
    UAuraAttributeSet();
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // === Vital Attributes ===
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital Attributes")
    FGameplayAttributeData Health;

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth, Category = "Vital Attributes")
    FGameplayAttributeData MaxHealth;

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Mana, Category = "Vital Attributes")
    FGameplayAttributeData Mana;

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxMana, Category = "Vital Attributes")
    FGameplayAttributeData MaxMana;

    // === OnRep 通知函数 ===
    UFUNCTION()
    void OnRep_Health(const FGameplayAttributeData& OldHealth) const;

    UFUNCTION()
    void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth) const;

    UFUNCTION()
    void OnRep_Mana(const FGameplayAttributeData& OldMana) const;

    UFUNCTION()
    void OnRep_MaxMana(const FGameplayAttributeData& OldMaxMana) const;
};
```

**CPP 文件（AuraAttributeSet.cpp）：**

```cpp
#include "AbilitySystem/AuraAttributeSet.h"
#include <Net/UnrealNetwork.h>

UAuraAttributeSet::UAuraAttributeSet()
{
    InitHealth(100.f);
    InitMaxHealth(100.f);
    InitMana(100.f);
    InitMaxMana(100.f);
}

void UAuraAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, Mana, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, MaxMana, COND_None, REPNOTIFY_Always);
}

void UAuraAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth) const
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UAuraAttributeSet, Health, OldHealth);
}

void UAuraAttributeSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth) const
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UAuraAttributeSet, MaxHealth, OldMaxHealth);
}

void UAuraAttributeSet::OnRep_Mana(const FGameplayAttributeData& OldMana) const
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UAuraAttributeSet, Mana, OldMana);
}

void UAuraAttributeSet::OnRep_MaxMana(const FGameplayAttributeData& OldMaxMana) const
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UAuraAttributeSet, MaxMana, OldMaxMana);
}
```

---

## 五、ATTRIBUTE_ACCESSORS 宏（4.3 核心）

### 5.1 四个底层宏

GAS 提供了四个底层宏，用于生成属性的访问函数：

| 宏                                                            | 生成的函数                  | 说明                                  |
| ------------------------------------------------------------ | ---------------------- | ----------------------------------- |
| `GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName)` | `GetHealthAttribute()` | 返回 `FGameplayAttribute` 对象（属性本身的引用） |
| `GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName)`               | `GetHealth()`          | 返回 `float`（属性的数值）                   |
| `GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName)`               | `SetHealth(float)`     | 设置属性的**基础值**（Base Value）            |
| `GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)`              | `InitHealth(float)`    | 设置**基础值和当前值**为同一个值                  |

> `Set` 和 `Init` 的区别：`Set` 只设置基础值，`Init` 同时设置基础值和当前值。

### 5.2 ATTRIBUTE_ACCESSORS 的定义

为了减少重复代码，可以定义一个方便的宏：

```cpp
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
```

### 5.3 使用示例

```cpp
// 声明属性
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital Attributes")
FGameplayAttributeData Health;

// 使用 ATTRIBUTE_ACCESSORS 生成访问函数
ATTRIBUTE_ACCESSORS(UAuraAttributeSet, Health);
```

这行宏会生成以下函数：

- `FGameplayAttribute UAuraAttributeSet::GetHealthAttribute()`
- `float UAuraAttributeSet::GetHealth()`
- `void UAuraAttributeSet::SetHealth(float NewVal)`
- `void UAuraAttributeSet::InitHealth(float NewVal)`

### 5.4 在构造函数中初始化属性

```cpp
UAuraAttributeSet::UAuraAttributeSet()
{
    InitHealth(100.f);      // 初始化 Health = 100
    InitMaxHealth(100.f);   // 初始化 MaxHealth = 100
    InitMana(50.f);         // 初始化 Mana = 50
    InitMaxMana(50.f);      // 初始化 MaxMana = 50
}
```

> ⚠️ 注意：构造函数调用的是 `Init` 而不是 `Set`，因为构造函数执行时机太早，`Set` 可能无法正确工作。`Init` 会同时设置基础值和当前值。

---

## 六、验证调试

### 6.1 Show Debug Ability System 命令

在游戏运行时，打开控制台（~键），输入：

```
showdebug abilitysystem
```

（两个单词：`showdebug` + `abilitysystem`）

### 6.2 调试信息解读

调试视图会显示：

| 信息                | 说明                                |
| ----------------- | --------------------------------- |
| **Avatar Actor**  | 当前显示的化身 Actor（如 BP_AuraCharacter） |
| **Authoritative** | 是否为权威版本（单人模式下就是权威）                |
| **Owner Actor**   | 所有者 Actor（如 BP_AuraPlayerState）   |
| **Controller**    | 玩家控制器                             |
| **Pawn**          | 当前控制的 Pawn                        |
| **Owned Tags**    | 拥有的游戏标签                           |
| **属性列表**          | 所有属性及其当前值                         |

- 按 **Page Up / Page Down** 可以在不同目标之间循环
- 可以看到属性值实时更新

---

## 七、新增 / 修改文件清单

| 文件                                                       | 变更类型 | 说明                                                                 |
| -------------------------------------------------------- | ---- | ------------------------------------------------------------------ |
| `Source/Aura/Public/AbilitySystem/AuraAttributeSet.h`    | ✅ 修改 | 添加 Health/MaxHealth/Mana/MaxMana 属性声明、OnRep 函数、ATTRIBUTE_ACCESSORS |
| `Source/Aura/Private/AbilitySystem/AuraAttributeSet.cpp` | ✅ 修改 | 实现构造函数初始化、GetLifetimeReplicatedProps、OnRep 函数                      |
| `Config/DefaultGame.ini`                                 | ✅ 修改 | 注释掉了 AbilitySystemGlobals 和 GameplayCueNotifyPaths 配置              |

---

## 八、知识点总结

| 知识点                     | 关键内容                                                          |
| ----------------------- | ------------------------------------------------------------- |
| **属性集注册**               | 构造函数中创建 AttributeSet 会自动注册到 ASC                               |
| **属性本质**                | `FGameplayAttributeData` 类型，包含 BaseValue + CurrentValue       |
| **最大值设计**               | 最大值应该是独立属性（如 MaxHealth），而非使用 BaseValue                        |
| **客户端预测**               | GAS 内置预测，客户端立即生效，服务器权威验证                                      |
| **属性声明 4 步**            | 声明变量 → OnRep 函数 → GAMEPLAYATTRIBUTE_REPNOTIFY → DOREPLIFETIME |
| **REPNOTIFY_Always**    | 即使值不变也复制，确保对设置行为做出响应                                          |
| **ATTRIBUTE_ACCESSORS** | 一行宏生成 Getter/Setter/Initter 四个访问函数                            |
| **Init vs Set**         | Init 同时设置 BaseValue 和 CurrentValue；Set 只设置 BaseValue          |
| **调试命令**                | `showdebug abilitysystem` 查看属性和标签                             |

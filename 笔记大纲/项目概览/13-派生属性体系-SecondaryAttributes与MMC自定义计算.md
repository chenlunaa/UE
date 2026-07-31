# UE5 学习笔记 — 第十三次提交（第8章：派生属性与自定义计算）

> 📦 Commit `108cb8a`：新增多种属性与多种初始化方式  
> 📦 Commit `7169863`：补充提交 — CombatInterface、MMC、AuraASC 委托系统  
> 📅 日期：2026-07-18  
> 🎬 对应视频：8.3~8.11（第8章核心内容）

---

## 目录

- [一、整体目标：构建派生属性体系](#section-3)
- [二、属性分类体系设计（8.6）](#section-4)
  - [2.1 三类属性的划分](#section-5)
  - [2.2 Primary Attributes（主属性）](#section-6)
  - [2.3 Secondary Attributes（次级属性）](#section-7)
  - [2.4 Vital Attributes（核心属性）](#section-8)
- [三、基于属性的修饰符（Attribute-Based Modifier）（8.3~8.5）](#section-9)
  - [3.1 四种修饰符强度计算类型](#section-10)
  - [3.2 Attribute-Based 的核心参数](#section-11)
  - [3.3 计算公式](#section-12)
  - [3.4 修饰符运算顺序（重要！）](#section-13)
  - [3.5 多修饰符混合运算示例](#section-14)
- [四、无限游戏效果与派生属性（8.7）](#section-15)
  - [4.1 派生属性的核心原理](#section-16)
  - [4.2 无限持续（Infinite）GE 的配置要点](#section-17)
  - [4.3 代码重构：ApplyEffectToSelf 通用函数](#section-18)
  - [4.4 InitializeDefaultAttributes 统一入口](#section-19)
- [五、CombatInterface 战斗接口（8.8~8.9）](#section-20)
  - [5.1 为什么需要 CombatInterface](#section-21)
  - [5.2 接口设计](#section-22)
  - [5.3 PlayerState 存储等级](#section-23)
  - [5.4 AuraEnemy 存储等级](#section-24)
  - [5.5 各角色类的实现](#section-25)
- [六、MMC 自定义计算类（8.8~8.10）](#section-26)
  - [6.1 为什么需要 MMC](#section-27)
  - [6.2 MMC 与 Attribute-Based 的对比](#section-28)
  - [6.3 MMC_MaxHealth 实现](#section-29)
  - [6.4 MMC_MaxMana 实现](#section-30)
  - [6.5 MMC 的使用方式](#section-31)
- [七、AuraAbilitySystemComponent 委托系统（补充提交）](#section-32)
  - [7.1 EffectAssetTags 多播委托](#section-33)
  - [7.2 AbilityActorInfoSet 初始化回调](#section-34)
- [八、Vital Attributes 初始化（8.11）](#section-35)
  - [8.1 设计思路](#section-36)
  - [8.2 GE_VitalAttributes 配置](#section-37)
- [九、新增/修改文件清单](#section-38)
- [十、知识点总结](#section-42)

---

<a id="section-3"></a>
## 一、整体目标：构建派生属性体系

本次提交的核心目标是建立一个完整的 RPG 属性系统，其中属性之间存在依赖/派生关系：

```
Primary Attributes (主属性)
  ├── Strength (力量)
  ├── Intelligence (智力)
  ├── Resilience (韧性)
  └── Vigor (活力)
        │
        ├──派生──→ Secondary Attributes (次级属性)
        │             ├── Armor (护甲) ← 依赖 Resilience
        │             ├── ArmorPenetration (护甲穿透) ← 依赖 Resilience
        │             ├── BlockChance (格挡几率) ← 依赖 Armor
        │             ├── CriticalHitChance (暴击几率) ← 依赖 ArmorPenetration
        │             ├── CriticalHitDamage (暴击伤害) ← 依赖 ArmorPenetration
        │             ├── CriticalHitResistance (暴击抗性) ← 依赖 Armor
        │             ├── HealthRegeneration (生命恢复) ← 依赖 Vigor
        │             ├── ManaRegeneration (法力恢复) ← 依赖 Intelligence
        │             ├── MaxHealth (最大生命) ← 依赖 Vigor + PlayerLevel
        │             └── MaxMana (最大法力) ← 依赖 Intelligence + PlayerLevel
        │
        └──派生──→ Vital Attributes (核心属性)
                      ├── Health ← 初始化为 MaxHealth
                      └── Mana ← 初始化为 MaxMana
```

---

<a id="section-4"></a>
## 二、属性分类体系设计（8.6）

<a id="section-5"></a>
### 2.1 三类属性的划分

| 分类 | 特点 | 初始化方式 |
|------|------|-----------|
| **Primary Attributes** | 独立属性，不依赖其他属性 | 通过 GE 直接设置值（如 Strength=10） |
| **Secondary Attributes** | 依赖主属性或其他次级属性 | 无限 GE + Attribute-Based Modifier 自动派生 |
| **Vital Attributes** | 有当前值/最大值之分 | 瞬时 GE 初始化为对应最大值 |

<a id="section-6"></a>
### 2.2 Primary Attributes（主属性）

```cpp
// 四个主属性，独立存在，不依赖任何其他属性
// 可通过升级、装备等途径提升
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Strength, Category = "Primary Attributes")
FGameplayAttributeData Strength;   // 提升物理伤害

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Intelligence, Category = "Primary Attributes")
FGameplayAttributeData Intelligence; // 提升魔法伤害

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Resilience, Category = "Primary Attributes")
FGameplayAttributeData Resilience;  // 提升护甲和护甲穿透

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Vigor, Category = "Primary Attributes")
FGameplayAttributeData Vigor;       // 提升生命值
```

<a id="section-7"></a>
### 2.3 Secondary Attributes（次级属性）

次级属性共 10 个，每个都通过**无限 GE + 覆盖操作**从其他属性派生：

| 属性 | 依赖属性 | 公式示例 | 作用 |
|------|---------|---------|------|
| Armor | Resilience | `6 + 0.25 × (韧性+2)` | 减少受到的伤害，提升格挡几率 |
| ArmorPenetration | Resilience | `3 + 0.15 × (韧性+1)` | 无视敌人护甲，提升暴击几率 |
| BlockChance | Armor | `4 + 0.25 × 护甲` | 格挡成功减半 incoming 伤害 |
| CriticalHitChance | ArmorPenetration | `2 + 0.25 × 护甲穿透` | 双倍伤害触发概率 |
| CriticalHitDamage | ArmorPenetration | `5 + 1.5 × 护甲穿透` | 暴击时附加的额外伤害 |
| CriticalHitResistance | Armor | `10 + 0.25 × 护甲` | 降低敌人暴击几率 |
| HealthRegeneration | Vigor | `1 + 0.1 × 活力` | 每秒恢复的生命值 |
| ManaRegeneration | Intelligence | `1 + 0.1 × 智力` | 每秒恢复的法力值 |
| MaxHealth | Vigor + Level | MMC 计算 | 生命值上限 |
| MaxMana | Intelligence + Level | MMC 计算 | 法力值上限 |

<a id="section-8"></a>
### 2.4 Vital Attributes（核心属性）

```cpp
// 只有两个核心属性，有当前值/最大值之分
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital Attributes")
FGameplayAttributeData Health;  // 当前生命值

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Mana, Category = "Vital Attributes")
FGameplayAttributeData Mana;    // 当前法力值
```

> **注意**：MaxHealth 和 MaxMana 从 Vital Attributes 移到了 Secondary Attributes，因为它们依赖主属性派生而来。

---

<a id="section-9"></a>
## 三、基于属性的修饰符（Attribute-Based Modifier）（8.3~8.5）

<a id="section-10"></a>
### 3.1 四种修饰符强度计算类型

| 类型 | 说明 | 使用场景 |
|------|------|---------|
| **Scalable Float** | 可缩放浮点数，可关联曲线表 | 简单固定值或按等级缩放 |
| **Attribute Based** | 基于其他属性的值来计算 | 派生属性、属性间关联 |
| **Custom Calculation Class** | 自定义 MMC 类，任意复杂计算 | 需要访问非属性变量（如等级） |
| **Set By Caller** | 由调用者设置 | 运行时动态传入值 |

<a id="section-11"></a>
### 3.2 Attribute-Based 的核心参数

```
基于属性的强度计算有三个关键参数：

┌─────────────────────────────────────────────┐
│  最终值 = (属性值 + 预乘加性值) × 系数 + 后乘加性值  │
└─────────────────────────────────────────────┘
```

| 参数 | 英文名 | 说明 |
|------|--------|------|
| **系数** | Coefficient | 乘法因子，如 0.25 表示取属性的 25% |
| **预乘加性值** | Pre Multiply Additive Value | 在乘以系数前加到属性值上 |
| **后乘加性值** | Post Multiply Additive Value | 在乘以系数后加到结果上 |

<a id="section-12"></a>
### 3.3 计算公式

```
步骤1: X = 捕获的属性值 + Pre Multiply Additive Value
步骤2: Y = X × Coefficient
步骤3: 最终结果 = Y + Post Multiply Additive Value
```

**示例**：活力=9，系数=0.1，预乘=3，后乘=1
```
(9 + 3) × 0.1 + 1 = 12 × 0.1 + 1 = 1.2 + 1 = 2.2
```

<a id="section-13"></a>
### 3.4 修饰符运算顺序（重要！）

**同一个 GE 中的多个修饰符按数组顺序依次执行**，每个修饰符在前一个结果上操作：

```
初始值 → [修饰符1] → 结果1 → [修饰符2] → 结果2 → [修饰符3] → 最终结果
```

**示例**：Health=10，三个修饰符：
- 修饰符1：+Vigor(9) → 19
- 修饰符2：×Strength(10) → 190
- 修饰符3：÷Resilience(12) → 15.83

<a id="section-14"></a>
### 3.5 多修饰符混合运算示例

```
Health=10, Vigor=9, Strength=10, Resilience=12

┌────────────────┬──────────┬──────────────┐
│   修饰符操作    │ 基础属性  │   计算结果     │
├────────────────┼──────────┼──────────────┤
│ 1. Add         │ Vigor    │ 10 + 9 = 19  │
│ 2. Multiply    │ Strength │ 19 × 10 = 190│
│ 3. Divide      │ Resilience│ 190 ÷ 12 ≈ 15.83│
│ 4. Add         │ MaxHealth│ 15.83 + 100 = 115.83│
└────────────────┴──────────┴──────────────┘
```

---

<a id="section-15"></a>
## 四、无限游戏效果与派生属性（8.7）

<a id="section-16"></a>
### 4.1 派生属性的核心原理

派生属性的关键：**无限持续的 GE + 覆盖（Override）操作**

```
原理：
1. 创建一个无限持续的 GE（Infinite Duration）
2. 修饰符操作设为 Override（覆盖，而非 Add）
3. 修饰符强度设为 Attribute-Based，绑定支撑属性
4. 禁用 Snapshot（快照），确保实时响应基础属性变化
5. 每当基础属性变化时，派生属性自动重算
```

<a id="section-17"></a>
### 4.2 无限持续（Infinite）GE 的配置要点

| 配置项 | 设置 | 原因 |
|--------|------|------|
| Duration Policy | Infinite | 始终生效 |
| Modifier Op | **Override** | 覆盖值而非累加 |
| Modifier Magnitude | Attribute Based | 从其他属性获取值 |
| Snapshot | **取消勾选** | 实时响应属性变化 |
| Attribute Source | Target | 从目标自身获取属性 |

<a id="section-18"></a>
### 4.3 代码重构：ApplyEffectToSelf 通用函数

将原先两个几乎一样的函数（`InitializePrimaryAttributes` / `InitializeSecondaryAttributes`）重构为一个通用函数：

```cpp
// AuraCharacterBase.h
void ApplyEffectToSelf(TSubclassOf<UGameplayEffect> GameplayEffectClass, float Level) const;
void InitializeDefaultAttributes() const;

// AuraCharacterBase.cpp
void AAuraCharacterBase::ApplyEffectToSelf(TSubclassOf<UGameplayEffect> const GameplayEffectClass, float const Level) const
{
    check(IsValid(GetAbilitySystemComponent()));
    check(GameplayEffectClass);
    FGameplayEffectContextHandle ContextHandle = GetAbilitySystemComponent()->MakeEffectContext();
    ContextHandle.AddSourceObject(this);
    const FGameplayEffectSpecHandle SpecHandle = GetAbilitySystemComponent()->MakeOutgoingSpec(GameplayEffectClass, Level, ContextHandle);
    GetAbilitySystemComponent()->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), GetAbilitySystemComponent());
}
```

<a id="section-19"></a>
### 4.4 InitializeDefaultAttributes 统一入口

```cpp
void AAuraCharacterBase::InitializeDefaultAttributes() const
{
    ApplyEffectToSelf(DefaultPrimaryAttributes, 1.f);    // 先初始化主属性
    ApplyEffectToSelf(DefaultSecondaryAttributes, 1.f);  // 再初始化次级属性（依赖主属性）
    ApplyEffectToSelf(DefaultVitalAttributes, 1.f);      // 最后初始化核心属性（依赖次级属性）
}
```

> **顺序至关重要**：次级属性依赖主属性，核心属性依赖次级属性，必须按顺序初始化。

---

<a id="section-20"></a>
## 五、CombatInterface 战斗接口（8.8~8.9）

<a id="section-21"></a>
### 5.1 为什么需要 CombatInterface

MMC 自定义计算需要访问**玩家等级**，但等级不是属性（不在 AttributeSet 中）：
- 玩家等级存储在 `PlayerState` 中
- 敌人等级存储在 `AuraEnemy` 中
- MMC 不应该依赖具体类，应依赖抽象（接口）

<a id="section-22"></a>
### 5.2 接口设计

```cpp
// CombatInterface.h
UINTERFACE(MinimalAPI)
class UCombatInterface : public UInterface
{
    GENERATED_BODY()
};

class AURA_API ICombatInterface
{
    GENERATED_BODY()
public:
    virtual int32 GetPlayerLevel();  // 默认返回 0
};
```

<a id="section-23"></a>
### 5.3 PlayerState 存储等级

```cpp
// AuraPlayerState.h
private:
    UPROPERTY(VisibleAnywhere, ReplicatedUsing = OnRep_Level)
    int32 Level = 1;

    UFUNCTION()
    void OnRep_Level(int32 OldLevel);

public:
    FORCEINLINE int32 GetPlayerLevel() const { return Level; }
```

复制配置：
```cpp
void AAuraPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AAuraPlayerState, Level);
}
```

<a id="section-24"></a>
### 5.4 AuraEnemy 存储等级

```cpp
// AuraEnemy.h
protected:
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Character Class Default")
    int32 Level = 1;
```

> 敌人等级不需要复制（Replicated），因为相关计算仅在服务器端执行。

<a id="section-25"></a>
### 5.5 各角色类的实现

```cpp
// AuraCharacterBase — 继承 ICombatInterface
class AURA_API AAuraCharacterBase : public ACharacter, public IAbilitySystemInterface, public ICombatInterface
{ ... };

// AuraEnemy — 直接返回自身 Level
int32 AAuraEnemy::GetPlayerLevel()
{
    return Level;
}

// AuraCharacter — 从 PlayerState 获取 Level
int32 AAuraCharacter::GetPlayerLevel()
{
    const AAuraPlayerState* AuraPlayerState = GetPlayerState<AAuraPlayerState>();
    check(AuraPlayerState);
    return AuraPlayerState->GetPlayerLevel();
}
```

> **命名注意**：函数名从 `GetLevel` 改为 `GetPlayerLevel`，因为 `AActor` 已有 `GetLevel()` 方法，避免隐藏基类函数。

---

<a id="section-26"></a>
## 六、MMC 自定义计算类（8.8~8.10）

<a id="section-27"></a>
### 6.1 为什么需要 MMC

当属性计算需要依赖**非属性变量**（如玩家等级）时，Attribute-Based Modifier 无法满足需求：

```
Attribute-Based 的局限：
  ✅ 可以：用 Vigor 计算 MaxHealth
  ❌ 不能：用 Vigor + PlayerLevel 计算 MaxHealth（Level 不是属性）

MMC 的优势：
  ✅ 可以：访问任何属性
  ✅ 可以：通过接口访问任意变量（如 PlayerLevel）
  ✅ 可以：实现任意复杂的计算逻辑
```

<a id="section-28"></a>
### 6.2 MMC 与 Attribute-Based 的对比

| 特性 | Attribute Based | MMC (Custom Calculation) |
|------|:---:|:---:|
| 依赖其他属性 | ✅ | ✅ |
| 依赖非属性变量 | ❌ | ✅ |
| 任意复杂计算 | ❌ | ✅ |
| 配置复杂度 | 低 | 中 |
| 曲线表支持 | ❌ | ❌（需配合 Scalable Float） |

<a id="section-29"></a>
### 6.3 MMC_MaxHealth 实现

```cpp
// MMC_Max_Health.h
UCLASS()
class AURA_API UMMC_Max_Health : public UGameplayModMagnitudeCalculation
{
    GENERATED_BODY()
public:
    UMMC_Max_Health();
    virtual float CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const override;
private:
    FGameplayEffectAttributeCaptureDefinition VigorDef;
};

// MMC_Max_Health.cpp
UMMC_Max_Health::UMMC_Max_Health()
{
    // 捕获 Vigor 属性
    VigorDef.AttributeToCapture = UAuraAttributeSet::GetVigorAttribute();
    VigorDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
    VigorDef.bSnapshot = false;
    RelevantAttributesToCapture.Add(VigorDef);
}

float UMMC_Max_Health::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const
{
    // 获取 Source/Target Tags
    const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    FAggregatorEvaluateParameters EvaluationParameters;
    EvaluationParameters.SourceTags = SourceTags;
    EvaluationParameters.TargetTags = TargetTags;

    // 获取捕获的属性值
    float Vigor = 0.f;
    GetCapturedAttributeMagnitude(VigorDef, Spec, EvaluationParameters, Vigor);
    Vigor = FMath::Max<float>(Vigor, 0.f);  // 确保非负

    // 通过接口获取玩家等级
    ICombatInterface* CombatInterface = Cast<ICombatInterface>(Spec.GetContext().GetSourceObject());
    const int32 PlayerLevel = CombatInterface->GetPlayerLevel();

    // 核心公式
    return 80.f + 2.5f * Vigor + 10.f * PlayerLevel;
}
```

<a id="section-30"></a>
### 6.4 MMC_MaxMana 实现

```cpp
// MMC_MaxMana.cpp — 结构完全相同，仅参数不同
UMMC_MaxMana::UMMC_MaxMana()
{
    IntDef.AttributeToCapture = UAuraAttributeSet::GetIntelligenceAttribute();
    IntDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
    IntDef.bSnapshot = false;
    RelevantAttributesToCapture.Add(IntDef);
}

float UMMC_MaxMana::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const
{
    // ... (获取 Tags、评估参数)
    float Int = 0.f;
    GetCapturedAttributeMagnitude(IntDef, Spec, EvaluationParameters, Int);
    Int = FMath::Max<float>(Int, 0.f);

    ICombatInterface* CombatInterface = Cast<ICombatInterface>(Spec.GetContext().GetSourceObject());
    const int32 PlayerLevel = CombatInterface->GetPlayerLevel();

    return 50.f + 2.5f * Int + 15.f * PlayerLevel;
}
```

<a id="section-31"></a>
### 6.5 MMC 的使用方式

在 GE 的 Modifier 中：
1. Modifier Magnitude Calculation Type → 选择 **Custom Calculation Class**
2. 在出现的 MMC 下拉中选择对应的 MMC 类（如 `MMC_Max_Health`）

---

<a id="section-32"></a>
## 七、AuraAbilitySystemComponent 委托系统（补充提交）

<a id="section-33"></a>
### 7.1 EffectAssetTags 多播委托

```cpp
// AuraAbilitySystemComponent.h
DECLARE_MULTICAST_DELEGATE_OneParam(FEffectAssetTags, const FGameplayTagContainer& /*AssetTags*/);

UCLASS()
class AURA_API UAuraAbilitySystemComponent : public UAbilitySystemComponent
{
    GENERATED_BODY()
public:
    void AbilityActorInfoSet();
    FEffectAssetTags EffectAssetTags;  // 多播委托，广播 GE 的 Asset Tags

protected:
    void EffectApplied(UAbilitySystemComponent* AbilitySystemComponent, 
                       const FGameplayEffectSpec& EffectSpec, 
                       FActiveGameplayEffectHandle ActiveEffectHandle);
};
```

<a id="section-34"></a>
### 7.2 AbilityActorInfoSet 初始化回调

```cpp
// AuraAbilitySystemComponent.cpp
void UAuraAbilitySystemComponent::AbilityActorInfoSet()
{
    // 绑定 ASC 自带的委托，当 GE 应用到自身时触发
    OnGameplayEffectAppliedDelegateToSelf.AddUObject(this, &UAuraAbilitySystemComponent::EffectApplied);
}

void UAuraAbilitySystemComponent::EffectApplied(UAbilitySystemComponent* AbilitySystemComponent,
                                                const FGameplayEffectSpec& EffectSpec, 
                                                FActiveGameplayEffectHandle ActiveEffectHandle)
{
    FGameplayTagContainer TagContainer;
    EffectSpec.GetAllAssetTags(TagContainer);  // 获取 GE 的所有 Asset Tags
    EffectAssetTags.Broadcast(TagContainer);    // 广播给所有订阅者
}
```

> **数据流**：GE 应用 → ASC 触发 `OnGameplayEffectAppliedDelegateToSelf` → `EffectApplied` 收集 Asset Tags → 广播 `EffectAssetTags` → WidgetController 接收并处理

---

<a id="section-35"></a>
## 八、Vital Attributes 初始化（8.11）

<a id="section-36"></a>
### 8.1 设计思路

核心属性（Health/Mana）需要在 Secondary Attributes 初始化完成后，设置为对应最大值：

```
问题：Health 初始值应该是多少？
  ❌ 硬编码：InitHealth(50.f) — 不灵活
  ✅ GE 初始化：创建瞬时 GE，将 Health 设为 MaxHealth 的值

时序要求：
  1. Primary Attributes GE 先执行 → 设置 Strength/Intelligence/Resilience/Vigor
  2. Secondary Attributes GE 执行 → 根据主属性计算 MaxHealth/MaxMana
  3. Vital Attributes GE 最后执行 → Health = MaxHealth, Mana = MaxMana
```

<a id="section-37"></a>
### 8.2 GE_VitalAttributes 配置

```
GE_VitalAttributes（瞬时 Instant）:
  Modifier 1: Health ← Override ← Attribute Based ← 捕获 MaxHealth (Target)
  Modifier 2: Mana   ← Override ← Attribute Based ← 捕获 MaxMana (Target)
```

代码中通过 `DefaultVitalAttributes` 变量在 `InitializeDefaultAttributes()` 中统一调用。

---

<a id="section-38"></a>
## 九、新增/修改文件清单

### 修改的文件

| 文件 | 主要变更 |
|------|---------|
| `AuraAttributeSet.h` | 新增 10 个 Secondary Attributes 声明 + 对应 OnRep 函数；MaxHealth/MaxMana 从 Vital 移至 Secondary |
| `AuraAttributeSet.cpp` | 新增所有 OnRep 函数实现；移除构造函数中的硬编码初始化；调整 `GetLifetimeReplicatedProps` 中的属性分组 |
| `AuraCharacterBase.h` | 继承 `ICombatInterface`；新增 `DefaultSecondaryAttributes`、`DefaultVitalAttributes`；重构为 `ApplyEffectToSelf()` + `InitializeDefaultAttributes()` |
| `AuraCharacterBase.cpp` | 实现 `ApplyEffectToSelf()` 和 `InitializeDefaultAttributes()` |
| `AuraCharacter.h` | 重写 `GetPlayerLevel()` |
| `AuraCharacter.cpp` | 实现 `GetPlayerLevel()`（从 PlayerState 获取）；调用改为 `InitializeDefaultAttributes()` |
| `AuraEnemy.h` | 新增 `Level` 变量；重写 `GetPlayerLevel()` |
| `AuraEnemy.cpp` | 实现 `GetPlayerLevel()`（直接返回 Level） |
| `AuraPlayerState.h` | 新增 `Level` 复制变量 + `GetPlayerLevel()` 获取器 + `OnRep_Level` |
| `AuraPlayerState.cpp` | 实现 `GetLifetimeReplicatedProps` 和 `OnRep_Level`；`NetUpdateFrequency` 改用 `SetNetUpdateFrequency()` |

### 新增的文件

| 文件 | 说明 |
|------|------|
| `CombatInterface.h` | 战斗接口，定义 `GetPlayerLevel()` |
| `CombatInterface.cpp` | 默认实现，返回 0 |
| `MMC_Max_Health.h/.cpp` | MaxHealth 自定义计算类（Vigor + PlayerLevel） |
| `MMC_MaxMana.h/.cpp` | MaxMana 自定义计算类（Intelligence + PlayerLevel） |
| `AuraAbilitySystemComponent.h/.cpp` | 自定义 ASC，声明 `EffectAssetTags` 委托，广播 GE 的 Asset Tags |

### 蓝图新增

| 蓝图 | 说明 |
|------|------|
| `GE_SecondaryAttributes` | 无限 GE，包含所有次级属性的 Override 修饰符 |
| `GE_VitalAttributes` | 瞬时 GE，将 Health/Mana 初始化为 MaxHealth/MaxMana |

---

<a id="section-42"></a>
## 十、知识点总结

### 核心概念

| 概念 | 要点 |
|------|------|
| **属性分类** | Primary（独立）→ Secondary（派生）→ Vital（当前值） |
| **派生属性** | 无限 GE + Override + Attribute Based + 禁用 Snapshot |
| **Attribute Based Modifier** | 公式：`(属性值 + Pre) × Coefficient + Post` |
| **修饰符顺序** | 同一 GE 内的多个修饰符按数组顺序依次执行 |
| **MMC** | 继承 `UGameplayModMagnitudeCalculation`，重写 `CalculateBaseMagnitude_Implementation` |
| **CombatInterface** | 抽象接口，解耦 MMC 与具体类，通过 `GetSourceObject()` 获取 |
| **初始化顺序** | Primary → Secondary → Vital（顺序不可颠倒） |

### 设计模式

| 模式 | 应用场景 |
|------|---------|
| **接口解耦** | CombatInterface 让 MMC 不依赖具体类 |
| **策略模式** | 不同角色（Player/Enemy）以不同方式实现 `GetPlayerLevel()` |
| **模板方法** | `InitializeDefaultAttributes()` 定义初始化流程，具体 GE 由蓝图配置 |
| **观察者模式** | `EffectAssetTags` 多播委托，WidgetController 订阅 GE 事件 |

### 关键注意事项

1. **派生属性必须用 Override 而非 Add**，否则会叠加而非替换
2. **必须禁用 Snapshot**，否则派生属性不会随基础属性实时更新
3. **初始化顺序至关重要**：Primary → Secondary → Vital
4. **`GetLevel()` 命名冲突**：AActor 已有此方法，改用 `GetPlayerLevel()`
5. **MMC 中通过 `Spec.GetContext().GetSourceObject()` 获取源对象**，再 Cast 到接口
6. **敌人的 Level 不需要复制**，因为相关计算仅在服务器执行

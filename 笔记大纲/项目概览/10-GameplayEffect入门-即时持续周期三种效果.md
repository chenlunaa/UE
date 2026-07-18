# UE5 学习笔记 — 第十次提交

> 📦 Commit `15628f9`：添加 GameplayEffect 蓝图操作（效果应用 + 属性监听扩展 + 持续性/周期性效果）  
> 📅 日期：2026-06-18  
> 🎬 对应视频：6.1 GameplayEffect 概述 / 6.2 重构 AuraEffectActor / 6.3 创建即时 GameplayEffect / 6.4 持续时间效果 / 6.5 周期性效果

---

## 目录

- [一、整体目标：从硬编码属性修改迁移到 GameplayEffect 系统](#一整体目标从硬编码属性修改迁移到-gameplayeffect-系统)
- [二、GameplayEffect 核心概念（6.1 理论）](#二gameplayeffect-核心概念61-理论)
  - [2.1 什么是 GameplayEffect](#21-什么是-gameplayeffect)
  - [2.2 修改器（Modifier）与操作类型](#22-修改器modifier与操作类型)
  - [2.3 幅度计算类型（Magnitude Calculation）](#23-幅度计算类型magnitude-calculation)
  - [2.4 持续时间策略（Duration Policy）](#24-持续时间策略duration-policy)
  - [2.5 GameplayEffectSpec 与 EffectContext](#25-gameplayeffectspec-与-effectcontext)
- [三、重构 AuraEffectActor（6.2 核心）](#三重构-auraeffectactor62-核心)
  - [3.1 移除硬编码组件的原因](#31-移除硬编码组件的原因)
  - [3.2 C++ 与蓝图的职责划分](#32-c-与蓝图的职责划分)
  - [3.3 ApplyEffectToTarget 函数详解](#33-applyeffecttotarget-函数详解)
  - [3.4 两种获取 ASC 的方式对比](#34-两种获取-asc-的方式对比)
  - [3.5 完整调用链路](#35-完整调用链路)
- [四、创建即时 GameplayEffect（6.3 实操）](#四创建即时-gameplayeffect63-实操)
  - [4.1 蓝图创建 GE 的步骤](#41-蓝图创建-ge-的步骤)
  - [4.2 健康药水 + 法力药水的完整配置](#42-健康药水--法力药水的完整配置)
  - [4.3 蓝图侧事件绑定](#43-蓝图侧事件绑定)
- [五、蓝图 vs C++ 应用 GE 的两种方式（6.4 对比）](#五蓝图-vs-c-应用-ge-的两种方式64-对比)
  - [5.1 C++ 方式（我们使用的）](#51-c-方式我们使用的)
  - [5.2 纯蓝图方式](#52-纯蓝图方式)
  - [5.3 两种方式的取舍](#53-两种方式的取舍)
- [六、持续时间与周期性效果（6.4-6.5）](#六持续时间与周期性效果64-65)
  - [6.1 基本值 vs 当前值](#61-基本值-vs-当前值)
  - [6.2 持续时间效果 — 健康水晶](#62-持续时间效果--健康水晶)
  - [6.3 周期性效果 — 持续治疗](#63-周期性效果--持续治疗)
  - [6.4 周期抑制策略](#64-周期抑制策略)
- [七、OverlayWidgetController 扩展法力值监听](#七overlaywidgetcontroller-扩展法力值监听)
- [八、新增/修改文件清单](#八新增修改文件清单)
- [九、知识点总结](#九知识点总结)

---

## 一、整体目标：从硬编码属性修改迁移到 GameplayEffect 系统

在之前的 AuraEffectActor 中，我们直接通过 `const_cast` 获取 AttributeSet 并手动调用 `SetHealth()` 来修改属性值。这种方式存在严重问题：

```
❌ 旧方式：
  重叠检测 → Cast<IAbilitySystemInterface> → GetAttributeSet() → const_cast → SetHealth(+25)
  
  问题：
  1. const_cast 破坏常量保护，违反设计意图
  2. 硬编码数值 (+25)，无法灵活配置
  3. 无法利用 GAS 的预测、网络同步、标签系统
  4. 每种效果类型都需要修改 C++ 代码
```

**本次提交的核心目标**：

```
✅ 新方式：
  重叠检测 → 蓝图调用 ApplyEffectToTarget() → 创建 EffectSpec → ApplyGameplayEffectSpecToSelf
  
  优势：
  1. 效果配置完全在蓝图中，设计师友好
  2. 支持即时/持续/周期/无限四种策略
  3. 自动处理网络同步和预测
  4. 同一 C++ 类可派生出任意类型的效果 Actor
```

---

## 二、GameplayEffect 核心概念（6.1 理论）

### 2.1 什么是 GameplayEffect

> GameplayEffect（简称 GE）是 GAS 中用于**修改属性和游戏标签**的数据资产类型。

**关键特性**：
- GE 是**纯数据**，不添加逻辑代码
- 我们基于 `UGameplayEffect` 类创建**蓝图子类**，而非 C++ 子类
- GE 通过**修改器（Modifier）**和**执行器（Execution）**来改变属性
- GE 足够灵活，几乎永远不需要 C++ 子类化

```
UGameplayEffect (C++ 基类)
    └── GE_PotionHeal (蓝图)  ← 只配置数据，不写代码
    └── GE_PotionMana (蓝图)
    └── GE_CrystalHeal (蓝图)
```

### 2.2 修改器（Modifier）与操作类型

修改器是 GE 中改变属性的核心机制，每个 GE 可以有**多个修改器**（数组），同时影响多个属性。

**四种修改器操作（ModifierOp）**：

| 操作 | 说明 | 示例 |
|------|------|------|
| **Add（加法）** | 将幅度加到属性上 | 生命值 +25 |
| **Multiply（乘法）** | 将属性乘以幅度 | 伤害 ×1.5 |
| **Divide（除法）** | 将属性除以幅度 | 速度 ÷2 |
| **Override（替换）** | 直接将属性替换为幅度值 | 生命值 = 100 |

> 💡 加法中使用负值即可实现减法效果。

### 2.3 幅度计算类型（Magnitude Calculation）

修改器中的"幅度"（Magnitude）可以通过多种方式计算：

| 计算类型 | 说明 | 复杂度 |
|----------|------|--------|
| **Scalable Float（可缩放浮点）** | 硬编码一个值，或通过曲线表按等级缩放 | ⭐ |
| **Attribute Based（基于属性）** | 使用另一个属性的值作为幅度 | ⭐⭐ |
| **Custom Calculation Class（自定义计算类）** | 创建 MMC 类，捕获多个属性进行复杂计算 | ⭐⭐⭐ |
| **Set By Caller（调用者设置）** | 通过代码逻辑在运行时设置幅度（键值对） | ⭐⭐ |

> 本阶段我们只使用最简单的 **Scalable Float**。

### 2.4 持续时间策略（Duration Policy）

| 策略 | 行为 | 修改目标 | 适用场景 |
|------|------|----------|----------|
| **Instant（即时）** | 一次性应用，永久生效 | 基本值（BaseValue） | 药水、永久升级 |
| **Has Duration（有持续时间）** | 持续 N 秒后自动移除 | 当前值（CurrentValue） | 临时 Buff/Debuff |
| **Infinite（无限）** | 永久存在，需手动移除 | 当前值（CurrentValue） | 装备效果、被动技能 |

> 🔑 **关键区别**：Instant 修改**基本值**（永久），Duration/Infinite 修改**当前值**（可撤销）。

### 2.5 GameplayEffectSpec 与 EffectContext

GAS 使用"规格（Spec）"模式来优化 GE 的应用：

```
GameplayEffect (类默认对象/CDO)
    └── GameplayEffectSpec (轻量级规格)
         ├── 效果类引用
         ├── 等级 (Level)
         ├── 幅度信息
         └── GameplayEffectContext (上下文)
              ├── 源对象 (Source Object)
              ├── 发起者 (Instigator)
              ├── 能力引用
              └── 是否命中 (Hit Result)
```

**为什么需要 Spec？**
- CDO 是共享的，不能修改；Spec 是每个应用实例独立的
- Spec 携带上下文信息（谁造成的、目标是谁、什么能力触发的）
- Handle 模式：`FGameplayEffectContextHandle` 和 `FGameplayEffectSpecHandle` 是轻量级包装器，内部用 `TSharedPtr` 持有实际数据

---

## 三、重构 AuraEffectActor（6.2 核心）

### 3.1 移除硬编码组件的原因

**旧代码的问题**：
```cpp
// ❌ 旧 AuraEffectActor — 在 C++ 中硬编码组件
UPROPERTY(VisibleAnywhere)
TObjectPtr<USphereComponent> Sphere;       // 写死了球体

UPROPERTY(VisibleAnywhere)
TObjectPtr<UStaticMeshComponent> Mesh;     // 写死了网格
```

- 如果设计师想要**盒子碰撞**而非球体 → 必须改 C++ 代码
- 如果设计师想要**胶囊碰撞** → 又得改 C++ 代码
- 视觉表现（网格形状）也被锁死在 C++ 中

**新设计理念**：
```
C++ 侧：只保留核心逻辑（应用效果的能力）
蓝图侧：自由选择视觉表现 + 碰撞体积 + 事件绑定
```

### 3.2 C++ 与蓝图的职责划分

| 职责 | C++ | 蓝图 |
|------|-----|------|
| 应用 GameplayEffect 的核心逻辑 | ✅ `ApplyEffectToTarget()` | — |
| 效果类的选择 | ✅ 声明变量类型 | ✅ 具体选择哪个 GE 蓝图 |
| 碰撞体积类型 | — | ✅ 球体/盒子/胶囊任选 |
| 视觉网格 | — | ✅ 药水瓶/水晶/碎片任选 |
| 重叠事件绑定 | — | ✅ 蓝图事件图 |
| 销毁时机 | — | ✅ 蓝图控制 |

```cpp
// ✅ 新 AuraEffectActor — 只保留 SceneComponent 作为根
AAuraEffectActor::AAuraEffectActor()
{
    PrimaryActorTick.bCanEverTick = false;
    SetRootComponent(CreateDefaultSubobject<USceneComponent>("SceneRoot"));
    // 不再有 Sphere、Mesh — 交给蓝图
}
```

### 3.3 ApplyEffectToTarget 函数详解

这是本次提交最核心的 C++ 函数：

```cpp
void AAuraEffectActor::ApplyEffectToTarget(AActor* TargetActor, 
    TSubclassOf<UGameplayEffect> GameplayEffectclass)
{
    // 步骤1：获取目标的能力系统组件（使用蓝图库函数）
    UAbilitySystemComponent* TargetASC = 
        UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(TargetActor);
    if (TargetASC == nullptr) return;  // 没有 ASC 就什么都不做
    
    check(GameplayEffectclass);  // GE 类必须设置，否则故意崩溃
    
    // 步骤2：创建效果上下文（EffectContext）
    FGameplayEffectContextHandle EffectContextHandle = TargetASC->MakeEffectContext();
    EffectContextHandle.AddSourceObject(this);  // 记录源对象（这个药水/水晶）
    
    // 步骤3：创建效果规格（EffectSpec）
    const FGameplayEffectSpecHandle EffectSpecHandle = 
        TargetASC->MakeOutgoingSpec(GameplayEffectclass, 1.f, EffectContextHandle);
    
    // 步骤4：将效果规格应用到自身
    TargetASC->ApplyGameplayEffectSpecToSelf(*(EffectSpecHandle.Data.Get()));
}
```

**逐步解析**：

```
步骤1: GetAbilitySystemComponent(TargetActor)
  └── 内部先尝试 Cast<IAbilitySystemInterface>
  └── 失败则用 FindComponentByClass<UAbilitySystemComponent>()
  └── 返回 UAbilitySystemComponent* 或 nullptr

步骤2: MakeEffectContext()
  └── 创建 FGameplayEffectContextHandle（包装器）
  └── 内部持有 FGameplayEffectContext（实际上下文数据）
  └── AddSourceObject(this) → 设置上下文的源对象

步骤3: MakeOutgoingSpec(GE类, 等级, 上下文)
  └── 返回 FGameplayEffectSpecHandle（包装器）
  └── 内部持有 FGameplayEffectSpec（实际规格数据）
  └── 等级参数：1.f（后续会用到等级缩放）

步骤4: ApplyGameplayEffectSpecToSelf(Spec)
  └── 参数类型: const FGameplayEffectSpec&（引用）
  └── 需要从 Handle 中解包: EffectSpecHandle.Data.Get() → 原始指针 → *解引用
```

> ⚠️ **语法要点**：`EffectSpecHandle.Data` 是 `TSharedPtr<FGameplayEffectSpec>`，所以需要 `.Get()` 获取原始指针，再 `*` 解引用为引用。

### 3.4 两种获取 ASC 的方式对比

| 方式 | 代码 | 优点 | 缺点 |
|------|------|------|------|
| **接口转换** | `Cast<IAbilitySystemInterface>(Actor)->GetAbilitySystemComponent()` | 直接、简单 | 要求 Actor 必须实现接口 |
| **蓝图库函数** | `UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Actor)` | 容错性好，Actor 不实现接口也能找到 | 略微复杂 |

```cpp
// 方式1：接口转换（旧方式）
if (IAbilitySystemInterface* ASCInterface = Cast<IAbilitySystemInterface>(OtherActor))
{
    ASCInterface->GetAbilitySystemComponent()->GetAttributeSet(...);
}

// 方式2：蓝图库函数（新方式，推荐）
UAbilitySystemComponent* TargetASC = 
    UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(TargetActor);
// 内部实现：
//   1. Cast<IAbilitySystemInterface> → 成功则返回 ASC
//   2. 失败则 FindComponentByClass<UAbilitySystemComponent>()
```

> 💡 蓝图库函数是**更健壮**的选择，因为它处理了 Actor 未实现接口的边缘情况。

### 3.5 完整调用链路

```
蓝图事件图                          C++ 核心逻辑
─────────                          ────────────
球体/盒子 OnComponentBeginOverlap
    │
    ├─ OtherActor (重叠的Actor)
    │
    ▼
ApplyEffectToTarget(TargetActor, GameplayEffectClass)  ← 蓝图可调用
    │
    ├─ GetAbilitySystemComponent(TargetActor)           ← 蓝图库函数
    │   └─ 返回 TargetASC 或 nullptr
    │
    ├─ MakeEffectContext()                              ← ASC 方法
    │   └─ AddSourceObject(this)
    │
    ├─ MakeOutgoingSpec(GE类, 1.f, ContextHandle)       ← ASC 方法
    │   └─ 返回 EffectSpecHandle
    │
    └─ ApplyGameplayEffectSpecToSelf(*Spec)             ← ASC 方法
        └─ 修改目标属性值
```

---

## 四、创建即时 GameplayEffect（6.3 实操）

### 4.1 蓝图创建 GE 的步骤

```
1. 右键 → 蓝图类 → 搜索 "GameplayEffect" → 选择基类
2. 命名规范：GE_ 前缀，如 GE_PotionHeal
3. 打开蓝图，在细节面板配置：
   ├── Duration Policy: Instant（即时）
   ├── Modifiers 数组 → 添加元素
   │   ├── Attribute: AuraAttributeSet.Health
   │   ├── ModifierOp: Add
   │   └── Magnitude → Scalable Float → Value: 25.0
   └── Application Chance: 1.0（100% 概率应用）
```

### 4.2 健康药水 + 法力药水的完整配置

**GE_PotionHeal（健康药水效果）**：
| 参数 | 值 |
|------|-----|
| Duration Policy | Instant |
| Modifier[0].Attribute | `AuraAttributeSet.Health` |
| Modifier[0].ModifierOp | Add |
| Modifier[0].Magnitude (ScalableFloat) | 25.0 |

**GE_PotionMana（法力药水效果）**：
| 参数 | 值 |
|------|-----|
| Duration Policy | Instant |
| Modifier[0].Attribute | `AuraAttributeSet.Mana` |
| Modifier[0].ModifierOp | Add |
| Modifier[0].Magnitude (ScalableFloat) | 30.0 |

### 4.3 蓝图侧事件绑定

在 `BP_HealthPotion` 蓝图中：

```
事件图：
  Sphere (球体组件)
    └── OnComponentBeginOverlap
         ├── OtherActor ──▶ TargetActor (ApplyEffectToTarget)
         ├── InstantGameplayEffectClass (变量) ──▶ GameplayEffectClass
         └── 执行后 → DestroyActor
```

> 💡 蓝图可调用函数的 `Target` 参数与蓝图的 `self` 概念冲突，因此将参数重命名为 `TargetActor`。

---

## 五、蓝图 vs C++ 应用 GE 的两种方式（6.4 对比）

### 5.1 C++ 方式（我们使用的）

```cpp
// 封装在 ApplyEffectToTarget 中，蓝图只需一个节点
UFUNCTION(BlueprintCallable)
void ApplyEffectToTarget(AActor* TargetActor, TSubclassOf<UGameplayEffect> GameplayEffectclass);
```

**蓝图侧**：只需一个节点，干净整洁。

### 5.2 纯蓝图方式

```
蓝图节点链：
  GetAbilitySystemComponent(TargetActor)
    └── MakeEffectContext()
    └── MakeOutgoingSpec(GE类, 1.0, Context)
    └── ApplyGameplayEffectSpecToSelf(Spec)
```

**蓝图侧**：需要 4-5 个节点串联，略显杂乱。

### 5.3 两种方式的取舍

| 维度 | C++ 封装 | 纯蓝图 |
|------|----------|--------|
| 蓝图整洁度 | ⭐⭐⭐ 一个节点 | ⭐⭐ 多个节点 |
| 灵活性 | ⭐⭐ 受限于 C++ 暴露的接口 | ⭐⭐⭐ 完全自由组合 |
| 性能 | 几乎无差异 | 几乎无差异 |
| 可调试性 | ⭐⭐ 需要 C++ 断点 | ⭐⭐⭐ 蓝图断点直观 |
| 适用场景 | 标准化流程 | 实验/原型阶段 |

> 🎯 **最佳实践**：将通用逻辑封装在 C++ 中，保留蓝图灵活性用于差异化配置。

---

## 六、持续时间与周期性效果（6.4-6.5）

### 6.1 基本值 vs 当前值

这是理解 GE 持续时间策略的关键概念：

```
属性 = 基本值 (BaseValue) + 临时修改器叠加 (CurrentValue)

Instant GE:
  修改 BaseValue → 永久生效，不可撤销
  
Duration/Infinite GE:
  修改 CurrentValue → 效果存在期间有效
  效果移除 → CurrentValue 回到 BaseValue
  
Periodic GE (周期性):
  每次周期 → 修改 BaseValue → 永久生效（即使效果被移除）
```

```
示例：最大生命值 +100 持续 2 秒

时间轴：
  t=0s:  应用 GE, MaxHealth = 100 + 100 = 200 (CurrentValue)
  t=2s:  GE 自动移除, MaxHealth = 100 (回到 BaseValue)
  
  视觉表现：健康球从 50% 降到 25%（因为分母变大）
```

### 6.2 持续时间效果 — 健康水晶

**GE_CrystalHeal 配置**：
| 参数 | 值 |
|------|-----|
| Duration Policy | Has Duration |
| Duration Magnitude (ScalableFloat) | 2.0 秒 |
| Modifier[0].Attribute | `AuraAttributeSet.MaxHealth` |
| Modifier[0].ModifierOp | Add |
| Modifier[0].Magnitude | 100.0 |

> 演示效果：最大生命值临时翻倍，2 秒后恢复。健康球会先"缩小"再恢复。

### 6.3 周期性效果 — 持续治疗

将 Duration/Infinite GE 的 **Period（周期）** 设为非零值即可变为周期性效果：

**GE_CrystalHeal（周期性版本）**：
| 参数 | 值 |
|------|-----|
| Duration Policy | Has Duration |
| Duration Magnitude | 2.0 秒 |
| **Period** | **0.1 秒** |
| Execute Periodic Effect on Application | ✅ 勾选 |
| Modifier[0].Attribute | `AuraAttributeSet.Health` |
| Modifier[0].ModifierOp | Add |
| Modifier[0].Magnitude | 1.0 |

**效果计算**：
```
2 秒 / 0.1 秒 = 20 个周期
勾选 "Execute on Application" → 立即执行 1 次
总计: 1 + 20 = 21 次 × 1.0 = 21 点生命值恢复
```

> ⚠️ **性能考量**：不要将周期设得太小（如 0.01 秒），因为每次周期都会触发 GAS 的完整预测/同步流程。对于平滑的 UI 表现，应使用**插值渲染**而非提高 GE 频率。

### 6.4 周期抑制策略

当 GE 被标签抑制时，周期性效果的"未执行周期"处理方式：

| 策略 | 行为 |
|------|------|
| Never Reset | 抑制期间周期不执行，恢复后继续 |
| Reset on Successful Application | 抑制期间周期重置 |

> 等学到 GameplayTag 后再深入理解。

---

## 七、OverlayWidgetController 扩展法力值监听

本次提交同时扩展了法力值的 UI 绑定：

**新增委托声明**（`OverlayWidgetController.h`）：
```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnManaChangeSignature, float, NewMana);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMaxManaChangeSignature, float, NewMaxMana);
```

**BroadcastInitialValues 扩展**：
```cpp
void UOverlayWidgetController::BroadcastInitialValues()
{
    // ... 健康值广播 ...
    OnManaChange.Broadcast(AuraAttributeSet->GetMana());
    OnMaxManaChange.Broadcast(AuraAttributeSet->GetMaxMana());
}
```

**BindCallbacksToDependencies 扩展**：
```cpp
void UOverlayWidgetController::BindCallbacksToDependencies()
{
    // ... 健康值绑定 ...
    AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
        AuraAttributeSet->GetManaAttribute()
    ).AddUObject(this, &UOverlayWidgetController::ManaChanged);
    
    AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
        AuraAttributeSet->GetMaxManaAttribute()
    ).AddUObject(this, &UOverlayWidgetController::MaxManaChanged);
}
```

> 同时将法力初始值从 100 改为 50（`InitMana(50.f)`），以便测试法力药水效果。

---

## 八、新增/修改文件清单

| 文件 | 变更类型 | 说明 |
|------|----------|------|
| `Source/Aura/Public/Actor/AuraEffectActor.h` | 修改 | 移除 Sphere/Mesh，新增 GE 变量和 ApplyEffectToTarget |
| `Source/Aura/Private/Actor/AuraEffectActor.cpp` | 重写 | 移除旧重叠逻辑，实现 ApplyEffectToTarget |
| `Source/Aura/Private/AbilitySystem/AuraAttributeSet.cpp` | 修改 | InitMana(50.f) |
| `Source/Aura/Public/UI/WidgetController/OverlayWidgetController.h` | 修改 | 新增法力值委托声明 |
| `Source/Aura/Private/UI/WidgetController/OverlayWidgetController.cpp` | 修改 | 新增法力值广播和回调 |
| `Content/BluePrint/Potions/GE_PotionHeal` | 新增 | 即时健康药水 GE 蓝图 |
| `Content/BluePrint/Potions/GE_PotionMana` | 新增 | 即时法力药水 GE 蓝图 |
| `Content/BluePrint/Potions/BP_ManaPotion` | 新增 | 法力药水 Actor 蓝图 |
| `Content/BluePrint/Potions/GE_CrystalHeal` | 新增 | 持续/周期治疗水晶 GE 蓝图 |
| `Content/BluePrint/Potions/BP_HealthCrystal` | 新增 | 健康水晶 Actor 蓝图 |
| `Content/BluePrint/Potions/GE_CrystalMana` | 新增 | 持续/周期法力水晶 GE 蓝图 |
| `Content/BluePrint/Potions/BP_ManaCrystal` | 新增 | 法力水晶 Actor 蓝图 |

---

## 九、知识点总结

| 知识点 | 关键内容 |
|--------|----------|
| **GameplayEffect 本质** | 纯数据资产，通过蓝图配置，不写逻辑代码 |
| **四种 ModifierOp** | Add / Multiply / Divide / Override |
| **三种 Duration Policy** | Instant（改 BaseValue）/ Has Duration（改 CurrentValue）/ Infinite（改 CurrentValue） |
| **EffectSpec 模式** | CDO → Spec（轻量级副本）→ Handle（包装器）→ Data（TSharedPtr） |
| **EffectContext 作用** | 携带源对象、发起者、能力引用等上下文信息 |
| **蓝图库函数优势** | `GetAbilitySystemComponent()` 比手动 Cast 更健壮 |
| **周期性 GE** | 设置 Period > 0，每次周期永久修改 BaseValue |
| **C++ vs 蓝图分工** | C++ 封装通用逻辑，蓝图负责差异化配置 |
| **UI 属性监听扩展** | 复用委托模式，轻松添加新属性的 UI 绑定 |
| **蓝图灵活性** | 将碰撞体积/视觉网格移到蓝图侧，设计师自由选择 |

---

> 📝 **下一步**：GameplayTag 系统、无限 GE、GE 叠加策略、自定义 Magnitude Calculation

# UE5 学习笔记 — 第十二次提交（第7章全线提交）

> 📦 Commit `ff44c22`：GameplayTag 系统、ASC 委托广播、Widget 消息系统与属性钳制修复  
> 📅 日期：2026-07-10  
> 🎬 对应视频：7.1~7.16（第7章全部内容）

---

## 目录

- [一、整体目标：建立 GameplayTag 驱动的 UI 消息系统](#一整体目标建立-gameplaytag-驱动的-ui-消息系统)
- [二、GameplayTag 基础概念（7.1~7.3）](#二gameplaytag-基础概念7173)
  - [2.1 什么是 GameplayTag](#21-什么是-gameplaytag)
  - [2.2 GameplayTag 的层级结构](#22-gameplaytag-的层级结构)
  - [2.3 创建 GameplayTag 的三种方式](#23-创建-gameplaytag-的三种方式)
  - [2.4 MatchesTag 的匹配规则](#24-matchestag-的匹配规则)
- [三、GameplayEffect 中的标签类型（7.5）](#三gameplayeffect-中的标签类型75)
  - [3.1 资产标签（Asset Tags）vs 授予标签（Granted Tags）](#31-资产标签asset-tagsvs-授予标签granted-tags)
  - [3.2 标签叠加 vs 效果叠加](#32-标签叠加-vs-效果叠加)
  - [3.3 即时效果与标签授予的局限性](#33-即时效果与标签授予的局限性)
- [四、AbilitySystemComponent 委托系统（7.5~7.6）](#四abilitysystemcomponent-委托系统7576)
  - [4.1 ASC 中的关键委托概览](#41-asc-中的关键委托概览)
  - [4.2 OnGameplayEffectAppliedDelegateToSelf](#42-ongameplayeffectapplieddelegatetoself)
  - [4.3 复制策略对委托的影响](#43-复制策略对委托的影响)
- [五、AuraAbilitySystemComponent 自定义委托（7.7~7.8）](#五auraabilitysystemcomponent-自定义委托7778)
  - [5.1 类重命名：My_AuraAbilitySystemComponent → AuraAbilitySystemComponent](#51-类重命名my_auraabilitysystemcomponent--auraabilitysystemcomponent)
  - [5.2 声明 EffectAssetTags 多播委托](#52-声明-effectassettags-多播委托)
  - [5.3 在 OnEffectApplied 中广播资产标签](#53-在-oneffectapplied-中广播资产标签)
  - [5.4 AbilityActorInfoSet 初始化回调](#54-abilityactorinfoset-初始化回调)
  - [5.5 InitAbilityActorInfo 虚函数重构](#55-initabilityactorinfo-虚函数重构)
- [六、OverlayWidgetController 重构（7.8~7.11）](#六overlaywidgetcontroller-重构78711)
  - [6.1 用 Lambda 替换回调函数（7.8 & 7.14）](#61-用-lambda-替换回调函数78--714)
  - [6.2 统一委托签名：FOnAttributeChangedSignature](#62-统一委托签名fonattributechangedsignature)
  - [6.3 FUIWidgetRow 结构体与数据表（7.10~7.11）](#63-fuiwidgetrow-结构体与数据表710711)
  - [6.4 绑定 EffectAssetTags 委托](#64-绑定-effectassettags-委托)
  - [6.5 GetDataTableRowByTag 模板函数](#65-getdatatablerowbytag-模板函数)
  - [6.6 MessageWidgetRowDelegate 广播](#66-messagewidgetrowdelegate-广播)
  - [6.7 MatchesTag 过滤非消息标签](#67-matchestag-过滤非消息标签)
- [七、属性钳制 Bug 修复（7.16）](#七属性钳制-bug-修复716)
  - [7.1 Bug 复现：PreAttributeChange 中 Clamp 的局限性](#71-bug-复现preattributechange-中-clamp-的局限性)
  - [7.2 根本原因：NewValue 是临时值](#72-根本原因newvalue-是临时值)
  - [7.3 解决方案：PostGameplayEffectExecute 中 SetHealth/SetMana](#73-解决方案postgameplayeffectexecute-中-sethealthsetmana)
- [八、新增/修改文件清单](#八新增修改文件清单)
- [九、知识点总结](#九知识点总结)

---

## 一、整体目标：建立 GameplayTag 驱动的 UI 消息系统

本次提交的核心目标是利用 GameplayTag 系统，实现"捡起道具 → 在 HUD 上显示对应消息"的完整数据链路：

```
GameplayEffect（带 Asset Tags）
    ↓ 应用到 ASC
AuraAbilitySystemComponent::OnEffectApplied
    ↓ 广播 EffectAssetTags 委托（携带 FGameplayTagContainer）
OverlayWidgetController（Lambda 绑定）
    ↓ 遍历标签，MatchesTag("Message") 过滤
    ↓ GetDataTableRowByTag 查找数据表行
    ↓ 广播 MessageWidgetRowDelegate
WBP_Overlay（蓝图绑定）
    ↓ 创建 EffectMessage Widget，设置图像和文本
    ↓ 播放动画 → 延迟销毁
```

同时修复了属性钳制在 `PreAttributeChange` 中不生效的 Bug。

---

## 二、GameplayTag 基础概念（7.1~7.3）

### 2.1 什么是 GameplayTag

GameplayTag 本质上是 **FName** 类型的层级化标签，与 GameplayTagManager 注册。它在 GAS 中至关重要，几乎所有 API 都使用 GameplayTag。

```
核心特点：
✅ 层级化结构（父.子.孙）
✅ 存储在 FGameplayTagContainer 中（比 TArray 更优化）
✅ 支持标签映射计数（TagMapCount）—— 同一标签可叠加
✅ 可实现 IGameplayTagAssetInterface 接口
✅ 用于：输入识别、能力属性、伤害类型、增益/减益消息等
```

### 2.2 GameplayTag 的层级结构

```
Attributes                    ← 根层级
├── Vital                     ← 子层级
│   ├── Health                ← 具体标签
│   ├── MaxHealth
│   ├── Mana
│   └── MaxMana
└── Primary                   ← 另一个子层级
    ├── Strength
    ├── Intelligence
    ├── Resilience
    └── Vigor

Message                       ← 消息层级
├── HealthPotion
├── HealthCrystal
├── ManaPotion
└── ManaCrystal
```

### 2.3 创建 GameplayTag 的三种方式

| 方式 | 操作 | 存储位置 |
|------|------|----------|
| **项目设置编辑器** | 编辑 → 项目设置 → GameplayTags → 添加新标签 | `Config/DefaultGameplayTags.ini` |
| **INI 文件直接编辑** | 编辑 `DefaultGameplayTags.ini` 中的 `+GameplayTagList` | 同上 |
| **数据表（DataTable）** | 创建 DataTable，行结构选 `GameplayTagTableRow`，在项目设置中添加到 `GameplayTagTableList` | Content 中的 DataTable 资产 |

**本次提交使用的标签**（来自 `DefaultGameplayTags.ini`）：

```ini
+GameplayTagList=(Tag="Attributes.Vital.Health",DevComment="role's live")
+GameplayTagList=(Tag="Attributes.Vital.Mana",DevComment="A resource used to cast spells")
+GameplayTagList=(Tag="Attributes.Vital.MaxHealth",DevComment="")
+GameplayTagList=(Tag="Attributes.Vital.MaxMana",DevComment="")
+GameplayTagList=(Tag="Message.HealthCrystal",DevComment="")
+GameplayTagList=(Tag="Message.HealthPotion",DevComment="")
+GameplayTagList=(Tag="Message.ManaCrystal",DevComment="")
+GameplayTagList=(Tag="Message.ManaPotion",DevComment="")
```

数据表方式添加的标签：

```ini
+GameplayTagTableList=/Game/BluePrint/AbilitySystem/GamePlayTags/DT_PrimaryAttributes.DT_PrimaryAttributes
```

### 2.4 MatchesTag 的匹配规则

```cpp
// MatchesTag：子标签匹配父标签返回 true，反之返回 false
FGameplayTag Tag = FGameplayTag::RequestGameplayTag(FName("Message.HealthPotion"));
FGameplayTag MessageTag = FGameplayTag::RequestGameplayTag(FName("Message"));

Tag.MatchesTag(MessageTag);        // ✅ true  — "Message.HealthPotion" 匹配 "Message"
MessageTag.MatchesTag(Tag);        // ❌ false — "Message" 不匹配 "Message.HealthPotion"
```

> **核心规则**：`A.B.C.MatchesTag(A.B)` → true；`A.B.MatchesTag(A.B.C)` → false。更具体的标签可以匹配更宽泛的父标签。

---

## 三、GameplayEffect 中的标签类型（7.5）

### 3.1 资产标签（Asset Tags）vs 授予标签（Granted Tags）

| 标签类型 | 行为 | 适用场景 |
|----------|------|----------|
| **Asset Tags（资产标签）** | GE "拥有"这些标签，但**不会授予**给目标 ASC | 用于标识 GE 本身，通过 `GetAllAssetTags()` 获取 |
| **Granted Tags（授予标签）** | GE 应用时，这些标签**会被授予**给目标 ASC | 持续时间效果期间，ASC 临时获得该标签 |
| **Combined Tags** | = 继承标签 + 添加标签 - 移除标签 | 查看最终生效的标签集合 |

```
⚠️ 关键区别：
  - Asset Tags  → Spec.GetAllAssetTags()    → 即时/持续效果均可获取
  - Granted Tags → 仅持续时间效果有效，即时效果不会授予标签
```

### 3.2 标签叠加 vs 效果叠加

```
场景：GE 设置 StackingType = AggregateByTarget, StackLimitCount = 1
  → 多次应用同一个 GE（堆叠效果），标签只会计数 1 次
  → 因为多个效果实例被合并为一个堆叠

场景：GE 设置 StackingType = None
  → 每次应用都是独立的效果实例
  → 标签计数 = 效果实例数（标签会叠加）
```

### 3.3 即时效果与标签授予的局限性

```
❌ 即时效果（Instant）：
  - Granted Tags 不会生效（没有持续时间，标签无法"持续"）
  - 但 Asset Tags 仍然可以通过 GetAllAssetTags() 获取

✅ 解决方案：
  - 使用 Asset Tags 标识所有类型的 GE（即时 + 持续）
  - 通过 ASC 的 OnGameplayEffectAppliedDelegateToSelf 委托捕获所有 GE 应用
```

---

## 四、AbilitySystemComponent 委托系统（7.5~7.6）

### 4.1 ASC 中的关键委托概览

| 委托名称 | 触发时机 | 包含即时效果 | 包含持续效果 |
|----------|----------|:---:|:---:|
| `OnGameplayEffectAppliedDelegateToSelf` | GE 应用到自身 | ✅ | ✅ |
| `OnGameplayEffectAppliedDelegateToTarget` | GE 应用到目标 | ✅ | ✅ |
| `OnActiveGameplayEffectAddedDelegateToSelf` | 活跃 GE 被添加 | ❌ | ✅ |
| `OnPeriodicGameplayEffectExecuteDelegateOnSelf` | 周期性 GE 执行 | ❌ | ✅ |
| `OnAnyGameplayEffectRemovedDelegate()` | 任何 GE 被移除 | — | ✅ |

> **推荐使用 `OnGameplayEffectAppliedDelegateToSelf`**：它同时覆盖即时和持续效果，是最通用的选择。

### 4.2 OnGameplayEffectAppliedDelegateToSelf

```cpp
// 声明在 AbilitySystemComponent.h 中
DECLARE_MULTICAST_DELEGATE_ThreeParams(
    FOnGameplayEffectAppliedDelegate,
    UAbilitySystemComponent*,
    const FGameplayEffectSpec&,
    FActiveGameplayEffectHandle
);

// 注释：在服务器上被调用，每当一个 GE 被应用到自身
// 这包括即时和基于持续时间的 GameplayEffect
```

### 4.3 复制策略对委托的影响

| 复制模式 | GE 复制到谁 | GameplayCues 复制 | GameplayTags 复制 |
|----------|------------|:---:|:---:|
| **Full** | 所有客户端 | ✅ | ✅ |
| **Mixed**（玩家） | 仅拥有者客户端 | ✅ | ✅ |
| **Minimal**（敌人） | 不复制 GE | ✅ | ✅ |

> ⚠️ `OnGameplayEffectAppliedDelegateToSelf` 在**服务器**上被调用。在客户端可能不会被触发，取决于复制策略。

---

## 五、AuraAbilitySystemComponent 自定义委托（7.7~7.8）

### 5.1 类重命名：My_AuraAbilitySystemComponent → AuraAbilitySystemComponent

```cpp
// 旧文件（已删除）
// Source/Aura/Public/AbilitySystem/My_AuraAbilitySystemComponent.h
// Source/Aura/Private/AbilitySystem/My_AuraAbilitySystemComponent.cpp

// 新类名：UAuraAbilitySystemComponent
// 使用 CoreRedirects 保证兼容性：
+ClassRedirects=(OldName="/Script/Aura.My_AuraAbilitySystemComponent",NewName="/Script/Aura.AuraAbilitySystemComponent")
```

所有引用点同步更新：
- `AuraEnemy.cpp`：`CreateDefaultSubobject<UAuraAbilitySystemComponent>`
- `AuraPlayerState.cpp`：同上
- `AuraCharacter.cpp`：`Cast<UAuraAbilitySystemComponent>(...)`

### 5.2 声明 EffectAssetTags 多播委托

```cpp
// AuraAbilitySystemComponent.h 中
DECLARE_MULTICAST_DELEGATE_OneParam(
    FEffectAssetTags, 
    const FGameplayTagContainer& /* AssetTags */
);

UCLASS()
class AURA_API UAuraAbilitySystemComponent : public UAbilitySystemComponent
{
    GENERATED_BODY()
    
public:
    FEffectAssetTags EffectAssetTags;  // 公开，供 WidgetController 绑定
};
```

> **为什么不用 Dynamic 委托**：WidgetController 是 C++ 类，不需要蓝图绑定，使用普通多播委托更轻量。

### 5.3 在 OnEffectApplied 中广播资产标签

```cpp
// AuraAbilitySystemComponent.cpp — 绑定到 ASC 的 OnGameplayEffectAppliedDelegateToSelf
void UAuraAbilitySystemComponent::OnEffectApplied(
    UAbilitySystemComponent* ASC, 
    const FGameplayEffectSpec& Spec, 
    FActiveGameplayEffectHandle Handle)
{
    FGameplayTagContainer TagContainer;
    Spec.GetAllAssetTags(TagContainer);  // 获取 GE 的所有 Asset Tags
    EffectAssetTags.Broadcast(TagContainer); // 广播给所有绑定者
}
```

### 5.4 AbilityActorInfoSet 初始化回调

```cpp
// 在 AuraAbilitySystemComponent 中新增方法
void UAuraAbilitySystemComponent::AbilityActorInfoSet()
{
    // 当 AbilityActorInfo 设置完成后调用
    // 在这里绑定 ASC 的委托
    OnGameplayEffectAppliedDelegateToSelf.AddUObject(
        this, &UAuraAbilitySystemComponent::OnEffectApplied);
}
```

调用链：
```
AuraCharacter::InitAbilityActorInfo()
  → ASC->InitAbilityActorInfo(PlayerState, this)
  → Cast<UAuraAbilitySystemComponent>(ASC)->AbilityActorInfoSet()

AuraEnemy::InitAbilityActorInfo()
  → ASC->InitAbilityActorInfo(this, this)
  → Cast<UAuraAbilitySystemComponent>(ASC)->AbilityActorInfoSet()
```

### 5.5 InitAbilityActorInfo 虚函数重构

```
重构前：
  - AAuraCharacterBase 没有 InitAbilityActorInfo
  - AAuraCharacter 和 AAuraEnemy 各自独立实现

重构后：
  - AAuraCharacterBase 声明 virtual void InitAbilityActorInfo()
  - AAuraCharacter::InitAbilityActorInfo() 添加 override
  - AAuraEnemy::InitAbilityActorInfo() 添加 override
  - AAuraEnemy::BeginPlay() 中调用 InitAbilityActorInfo()
```

---

## 六、OverlayWidgetController 重构（7.8~7.11）

### 6.1 用 Lambda 替换回调函数（7.8 & 7.14）

**重构前**（需要4个成员回调函数）：

```cpp
// .h 中声明
void HealthChanged(const FOnAttributeChangeData& Data) const;
void MaxHealthChanged(const FOnAttributeChangeData& Data) const;
void ManaChanged(const FOnAttributeChangeData& Data) const;
void MaxManaChanged(const FOnAttributeChangeData& Data) const;

// .cpp 中绑定
AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
    AuraAttributeSet->GetHealthAttribute())
    .AddUObject(this, &UOverlayWidgetController::HealthChanged);
```

**重构后**（使用 Lambda，一行搞定）：

```cpp
AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
    AuraAttributeSet->GetHealthAttribute()).AddLambda(
        [this](const FOnAttributeChangeData& Data)
        {
            OnHealthChange.Broadcast(Data.NewValue);
        }
    );
```

> **Lambda 语法回顾**：`[捕获](参数){函数体}`
> - `[this]` 捕获 this 指针，使 Lambda 内可访问成员变量/函数
> - 适合简单回调，避免创建大量成员函数

### 6.2 统一委托签名：FOnAttributeChangedSignature

```cpp
// 重构前：4 个不同的委托类型
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthChangeSignature, float, NewHealth);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMaxHealthChangeSignature, float, NewMaxHealth);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnManaChangeSignature, float, NewMana);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMaxManaChangeSignature, float, NewMaxMana);

// 重构后：统一为 1 个
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnAttributeChangedSignature, float, NewValue);

// 4 个成员变量共用同一个签名
FOnAttributeChangedSignature OnHealthChange;
FOnAttributeChangedSignature OnMaxHealthChange;
FOnAttributeChangedSignature OnManaChange;
FOnAttributeChangedSignature OnMaxManaChange;
```

### 6.3 FUIWidgetRow 结构体与数据表（7.10~7.11）

```cpp
USTRUCT(BlueprintType)
struct FUIWidgetRow : public FTableRowBase
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    FGameplayTag MessageTag = FGameplayTag();  // 用于匹配的消息标签
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    FText Message = FText();                   // 显示的文本
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSubclassOf<class UAuraUserWidget> MessageWidget; // 对应 Widget 类
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    UTexture2D* Image = nullptr;               // 图标
};
```

数据表 `MessageWidgetDataTable` 的行结构为 `FUIWidgetRow`，每一行对应一种消息类型。

### 6.4 绑定 EffectAssetTags 委托

```cpp
// OverlayWidgetController::BindCallbacksToDependencies() 中
Cast<UAuraAbilitySystemComponent>(AbilitySystemComponent)->EffectAssetTags.AddLambda(
    [this](const FGameplayTagContainer& AssetTags)
    {
        for (const FGameplayTag& Tag : AssetTags)
        {
            FGameplayTag MessageTag = FGameplayTag::RequestGameplayTag(FName("Message"));
            if (Tag.MatchesTag(MessageTag))
            {
                const FUIWidgetRow* Row = GetDataTableRowByTag<FUIWidgetRow>(
                    MessageWidgetDataTable, Tag);
                MessageWidgetRowDelegate.Broadcast(*Row);
            }
        }
    }		
);
```

### 6.5 GetDataTableRowByTag 模板函数

```cpp
template<typename T>
T* UOverlayWidgetController::GetDataTableRowByTag(UDataTable* DataTable, const FGameplayTag& Tag)
{
    return DataTable->FindRow<T>(Tag.GetTagName(), TEXT(""));
}
```

> `FindRow<T>()` 通过 RowName（此处为 Tag 名称）在数据表中查找对应行。

### 6.6 MessageWidgetRowDelegate 广播

```cpp
// 新增委托声明
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FMessageWidgetRowSignature, FUIWidgetRow, Row);

// 成员变量
UPROPERTY(BlueprintAssignable, Category = "GAS|Messages")
FMessageWidgetRowSignature MessageWidgetRowDelegate;
```

蓝图侧（WBP_Overlay）绑定该委托 → 创建 EffectMessage Widget → 设置图像和文本 → 添加到 Viewport → 播放动画 → 延迟销毁。

### 6.7 MatchesTag 过滤非消息标签

```cpp
// 只有标签层级以 "Message" 为根的标签才会触发消息显示
FGameplayTag MessageTag = FGameplayTag::RequestGameplayTag(FName("Message"));
if (Tag.MatchesTag(MessageTag))
{
    // 在数据表中查找并广播
}
```

这样确保 `Attributes.Vital.Health` 等非消息标签不会触发 UI 消息。

---

## 七、属性钳制 Bug 修复（7.16）

### 7.1 Bug 复现：PreAttributeChange 中 Clamp 的局限性

```
复现步骤：
1. 捡起多个 HealthCrystal → 血量超过 100（Clamp 在 PreAttributeChange 中生效）
2. 再捡一个 → 实际内部值超过 100
3. 走进火区域 → 血量不会下降！

原因：PreAttributeChange 中的 NewValue = Clamp(NewValue, 0, MaxHealth)
      只是修改了"从 Modifier 查询返回的值"，没有真正设置属性值
```

### 7.2 根本原因：NewValue 是临时值

```cpp
// ❌ PreAttributeChange 中的 Clamp 只是修改返回值
void UAuraAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    // NewValue 是引用，修改它只会影响 Modifier 查询的返回值
    // 不会永久改变 Attribute 的真实值
    NewValue = FMath::Clamp(NewValue, 0.f, GetMaxHealth());
    // ⚠️ 下次其他 GE 查询 Modifier 时，会重新计算，值可能再次越界
}
```

### 7.3 解决方案：PostGameplayEffectExecute 中 SetHealth/SetMana

```cpp
void UAuraAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);
    
    FEffectProperties Props;
    SetEffectProperties(Data, Props);
    
    // ✅ 在效果执行完成后，真正设置属性值
    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        SetHealth(FMath::Clamp(GetHealth(), 0.f, GetMaxHealth()));
    }
    
    if (Data.EvaluatedData.Attribute == GetManaAttribute())
    {
        SetMana(FMath::Clamp(GetMana(), 0.f, GetMaxMana()));
    }
}
```

> **核心理解**：
> - `PreAttributeChange`：修改 Modifier 查询的**临时返回值**，适合做临时钳制
> - `PostGameplayEffectExecute`：在效果完全执行后，**真正设置**属性值，适合做最终钳制
> - 两者配合使用：Pre 做第一道防线，Post 做最终保障

---

## 八、新增/修改文件清单

| 文件 | 操作 | 说明 |
|------|:---:|------|
| `Config/DefaultEngine.ini` | 修改 | 添加 CoreRedirects（类重命名兼容） |
| `Config/DefaultGameplayTags.ini` | 修改 | 添加 Vital 属性标签、Message 标签、数据表引用 |
| `Source/Aura/Public/AbilitySystem/My_AuraAbilitySystemComponent.h` | **删除** | 重命名为 AuraAbilitySystemComponent |
| `Source/Aura/Private/AbilitySystem/My_AuraAbilitySystemComponent.cpp` | **删除** | 同上 |
| `Source/Aura/Public/AbilitySystem/AuraAbilitySystemComponent.h` | 新增 | 添加 EffectAssetTags 委托、AbilityActorInfoSet、OnEffectApplied |
| `Source/Aura/Private/AbilitySystem/AuraAbilitySystemComponent.cpp` | 新增 | 实现上述方法 |
| `Source/Aura/Public/Character/AuraCharacterBase.h` | 修改 | 添加 `virtual void InitAbilityActorInfo()` |
| `Source/Aura/Private/Character/AuraCharacterBase.cpp` | 修改 | 添加空的 InitAbilityActorInfo 实现 |
| `Source/Aura/Public/Character/AuraCharacter.h` | 修改 | InitAbilityActorInfo 添加 override |
| `Source/Aura/Private/Character/AuraCharacter.cpp` | 修改 | 调用 AbilityActorInfoSet，引用新类名 |
| `Source/Aura/Public/Character/AuraEnemy.h` | 修改 | InitAbilityActorInfo 添加 override |
| `Source/Aura/Private/Character/AuraEnemy.cpp` | 修改 | BeginPlay 调用 InitAbilityActorInfo，引用新类名 |
| `Source/Aura/Private/Player/AuraPlayerState.cpp` | 修改 | 引用新类名 UAuraAbilitySystemComponent |
| `Source/Aura/Public/UI/WidgetController/OverlayWidgetController.h` | 修改 | 添加 FUIWidgetRow 结构体、统一委托签名、模板函数、MessageWidgetRowDelegate |
| `Source/Aura/Private/UI/WidgetController/OverlayWidgetController.cpp` | 修改 | Lambda 替换回调、绑定 EffectAssetTags、实现消息广播逻辑 |
| `Source/Aura/Private/AbilitySystem/AuraAttributeSet.cpp` | 修改 | PostGameplayEffectExecute 中添加 Health/Mana 钳制 |

---

## 九、知识点总结

| 知识点 | 要点 |
|--------|------|
| **GameplayTag** | 层级化 FName 标签，存储在 FGameplayTagContainer 中，支持 MatchesTag 部分匹配 |
| **Asset Tags vs Granted Tags** | Asset Tags 标识 GE 本身（即时+持续都可用），Granted Tags 授予给 ASC（仅持续效果） |
| **MatchesTag 规则** | 子标签匹配父标签 → true；父标签匹配子标签 → false |
| **ASC 委托选择** | `OnGameplayEffectAppliedDelegateToSelf` 覆盖即时+持续效果，最通用 |
| **多播委托 vs 动态多播委托** | C++ 绑定用普通多播，蓝图绑定用 Dynamic 多播 |
| **Lambda 表达式** | `[捕获](参数){函数体}`，适合简单回调，避免创建大量成员函数 |
| **统一委托签名** | 功能相同的委托应共用同一签名类型，减少代码冗余 |
| **数据表驱动 UI** | 通过 DataTable + FTableRowBase 结构体，用 GameplayTag 作为 RowName 查找 |
| **PreAttributeChange Clamp 局限** | NewValue 是临时返回值，不会永久改变属性真实值 |
| **PostGameplayEffectExecute 钳制** | 效果执行完成后用 SetHealth/SetMana 真正设置值，做最终钳制 |
| **CoreRedirects** | 类重命名时在 DefaultEngine.ini 中添加重定向，保证已有资产兼容 |
| **虚函数重构** | 将 InitAbilityActorInfo 提取到基类 AAuraCharacterBase，子类 override |

# UE5 学习笔记 - 第二十次提交（第14章：自定义 EffectContext、伤害类型抗性与多人修复）

> Commit `56d0481`：实现伤害、暴击、格挡可视化，新增抗性属性并修复多人游戏问题  
> 日期：2026-08-13  
> 对应视频：14.1~14.14  
> 提交规模：24 个文件，新增 519 行、删除 33 行

---

## 目录

- [一、本章解决了什么问题](#section-1)
- [二、整体数据流](#section-2)
- [三、为什么需要自定义 GameplayEffectContext](#section-3)
- [四、FAuraGameplayEffectContext 的实现](#section-4)
- [五、NetSerialize 与 RepBits](#section-5)
- [六、让 GAS 创建自定义 Context](#section-6)
- [七、格挡与暴击信息如何贯穿整条链路](#section-7)
- [八、伤害飘字可视化](#section-8)
- [九、伤害类型与抗性体系](#section-9)
- [十、ExecCalc_Damage 的新版计算流程](#section-10)
- [十一、多人模式中的 Authority 与 RPC 修复](#section-11)
- [十二、专题：Dedicated Server 火球提前爆炸问题](#section-12)
- [十三、投射物 Owner、Instigator 与 EffectCauser](#section-13)
- [十四、本次提交中的其他修复](#section-14)
- [十五、提交文件清单](#section-15)
- [十六、源码校验与当前边界](#section-16)
- [十七、知识点总结](#section-17)

---

<a id="section-1"></a>
## 一、本章解决了什么问题

上一章已经能算出格挡、护甲、暴击和最终伤害，但存在三个明显缺口：

1. `ExecCalc_Damage` 中的 `bBlocked`、`bCriticalHit` 是局部变量，离开函数后信息就丢失，飘字只能显示数字；
2. 所有技能仍使用一个通用 `Damage` Tag，无法区分火焰、闪电、奥术和物理伤害，也无法应用对应抗性；
3. 单机正常不代表多人正常，客户端访问 GameMode、错误选择 PlayerController、投射物本地 Overlap 等问题会在 Listen Server 或 Dedicated Server 测试中暴露。

本章最终形成三条闭环：

```text
格挡/暴击结果
  -> 自定义 EffectContext
  -> 网络序列化
  -> AttributeSet
  -> Client RPC
  -> 不同样式的伤害飘字

多类型基础伤害
  -> Damage.<Type> SetByCaller
  -> DamageType 到 Resistance Tag 映射
  -> 捕获目标对应抗性
  -> 分类型减伤后求和
  -> 原有格挡/护甲/暴击流程

多人测试
  -> 区分服务端权威逻辑和客户端表现逻辑
  -> 只在服务器初始化属性/授予能力
  -> RPC 发给真正的伤害来源客户端
  -> 投射物过滤自身并防止重复表现
```

---

<a id="section-2"></a>
## 二、整体数据流

```text
UAuraProjectileSpell
  |
  |-- DamageTypes: TMap<DamageTag, FScalableFloat>
  |-- 为每种伤害写入 SetByCaller
  |-- MakeEffectContext() 实际分配 FAuraGameplayEffectContext
  v
GE_Damage Spec
  |
  v
ExecCalc_Damage（服务器）
  |
  |-- 遍历 DamageTypesToResistance
  |-- 读取每种 SetByCaller 伤害
  |-- 捕获目标对应抗性并分别减伤
  |-- 合计后计算格挡、护甲和暴击
  |-- 把 bBlocked / bCriticalHit 写回 Context
  `-- 输出 IncomingDamage
          |
          v
AuraAttributeSet::PostGameplayEffectExecute（服务器）
  |
  |-- 扣血并触发受击/死亡
  |-- 从 Context 读取格挡/暴击
  |-- 找到 SourceCharacter->Controller
  `-- ShowDamageNumber Client RPC
          |
          v
伤害来源玩家的本地 PlayerController
  -> DamageTextComponent::SetDamageText(Damage, Blocked, Critical)
  -> 蓝图按普通/格挡/暴击/格挡暴击选择文字与颜色
```

这里的设计重点是：**最终伤害走元属性，结算结果的附加信息走 EffectContext。**

---

<a id="section-3"></a>
## 三、为什么需要自定义 GameplayEffectContext

原生 `FGameplayEffectContext` 已能保存：

- Instigator 与 EffectCauser；
- Ability CDO、能力实例与等级；
- SourceObject；
- 相关 Actor 数组；
- HitResult 与 WorldOrigin。

但它没有项目专属的：

```cpp
bool bIsBlockedHit;
bool bIsCriticalHit;
```

这两个值属于“本次伤害事件的元数据”：

- 它们不是长期角色属性；
- 不适合用额外 Attribute 保存；
- 不只是伤害公式的输入，而是计算后产生的结果；
- AttributeSet、飘字和未来 GameplayCue 都可能需要读取；
- 多人游戏中还要随 Context 到达正确机器。

因此最合适的方案是继承原生 Context，而不是创建与 GAS 平行的全局变量或临时单例。

关于原生 Context、Handle、智能指针和 `TWeakObjectPtr` 的源码基础，已经单独整理在 C++ 专题：`GameplayEffectContext源码-智能指针与WeakPtr详解.md`。

---

<a id="section-4"></a>
## 四、FAuraGameplayEffectContext 的实现

### 4.1 自定义结构体

```cpp
USTRUCT(BlueprintType)
struct FAuraGameplayEffectContext : public FGameplayEffectContext
{
    GENERATED_BODY()

public:
    bool IsCriticalHit() const { return bIsCriticalHit; }
    bool IsBlockedHit() const { return bIsBlockedHit; }

    void SetIsCriticalHit(bool bIn) { bIsCriticalHit = bIn; }
    void SetIsBlockedHit(bool bIn) { bIsBlockedHit = bIn; }

protected:
    UPROPERTY()
    bool bIsBlockedHit = false;

    UPROPERTY()
    bool bIsCriticalHit = false;
};
```

使用 Getter/Setter 而不是暴露字段，可以保持封装，并为以后增加规则、日志或兼容处理留出入口。

### 4.2 GetScriptStruct

```cpp
virtual UScriptStruct* GetScriptStruct() const override
{
    return StaticStruct();
}
```

基类指针实际指向子类对象时，GAS 的反射、复制和序列化系统必须知道真实类型是 `FAuraGameplayEffectContext`。如果仍返回基类 ScriptStruct，自定义字段可能被当作不存在。

### 4.3 Duplicate

```cpp
virtual FAuraGameplayEffectContext* Duplicate() const override
{
    FAuraGameplayEffectContext* NewContext = new FAuraGameplayEffectContext();
    *NewContext = *this;

    if (GetHitResult())
    {
        NewContext->AddHitResult(*GetHitResult(), true);
    }
    return NewContext;
}
```

普通赋值会复制两个布尔值和弱对象引用，但 `HitResult` 是 `TSharedPtr<FHitResult>`，直接复制会让新旧 Context 共享同一个命中结果。因此这里对 HitResult 做深拷贝，使 Duplicate 得到真正可独立修改的上下文。

### 4.4 TStructOpsTypeTraits

```cpp
template<>
struct TStructOpsTypeTraits<FAuraGameplayEffectContext>
    : TStructOpsTypeTraitsBase2<FAuraGameplayEffectContext>
{
    enum
    {
        WithNetSerializer = true,
        WithCopy = true
    };
};
```

它是在告诉 UE：

- 这个结构体有自定义 `NetSerialize`；
- 允许使用复制操作。

仅实现同名函数还不够，Traits 决定反射系统是否调用它。

---

<a id="section-5"></a>
## 五、NetSerialize 与 RepBits

### 5.1 为什么需要网络序列化

服务器与客户端不共享内存地址。Context 中的 Actor 引用、HitResult、WorldOrigin 和自定义布尔值必须被编码成网络数据，再由接收端还原。

```cpp
virtual bool NetSerialize(
    FArchive& Ar,
    UPackageMap* Map,
    bool& bOutSuccess) override;
```

三个参数的作用：

| 参数 | 作用 |
|---|---|
| `FArchive& Ar` | 保存时写入、加载时读取的序列化数据流 |
| `UPackageMap* Map` | 将 UObject 网络引用转换为网络标识并在另一端还原 |
| `bOutSuccess` | 告诉调用者序列化是否成功 |

### 5.2 RepBits 的思想

Context 中很多字段是可选的。先用位掩码说明哪些字段存在，接收方只读取对应数据：

```text
bit 0 -> Instigator
bit 1 -> EffectCauser
bit 2 -> AbilityCDO
bit 3 -> SourceObject
bit 4 -> Actors
bit 5 -> HitResult
bit 6 -> WorldOrigin
bit 7 -> bIsBlockedHit
bit 8 -> bIsCriticalHit
```

保存时：

```cpp
if (bIsBlockedHit)
{
    RepBits |= 1 << 7;
}
```

网络中只发送 9 个有效位：

```cpp
Ar.SerializeBits(&RepBits, 9);
```

加载时根据位恢复字段。布尔值本身可直接由“该位是否存在”表达，标准实现通常不需要再额外序列化一份 bool。

### 5.3 为什么加载后重新 AddInstigator

```cpp
if (Ar.IsLoading())
{
    AddInstigator(Instigator.Get(), EffectCauser.Get());
}
```

`InstigatorAbilitySystemComponent` 是根据 Instigator 推导出来的非复制缓存。接收端还原 Instigator 和 EffectCauser 后，需要重新调用 `AddInstigator` 初始化 ASC 引用。

### 5.4 当前源码的实现提醒

当前代码在 bit 7/8 已表示 true 后，还执行：

```cpp
Ar << bIsBlockedHit;
Ar << bIsCriticalHit;
```

只要保存端和加载端完全对称，它仍能工作，但会多发送布尔值，且与“RepBit 本身就是值”的常见写法重复。更精简的加载逻辑是：

```cpp
bIsBlockedHit = (RepBits & (1 << 7)) != 0;
bIsCriticalHit = (RepBits & (1 << 8)) != 0;
```

这是当前实现可继续优化的地方，不影响本章主流程理解。

---

<a id="section-6"></a>
## 六、让 GAS 创建自定义 Context

仅声明子类不会自动生效。`ASC->MakeEffectContext()` 默认仍会分配原生类型，因此还要替换全局分配入口。

### 6.1 自定义 AbilitySystemGlobals

```cpp
UCLASS()
class AURA_API UAuraAbilitySystemGlobals : public UAbilitySystemGlobals
{
    GENERATED_BODY()

public:
    virtual FGameplayEffectContext* AllocGameplayEffectContext() const override;
};
```

```cpp
FGameplayEffectContext*
UAuraAbilitySystemGlobals::AllocGameplayEffectContext() const
{
    return new FAuraGameplayEffectContext();
}
```

函数返回基类指针，但实际堆对象是 Aura 子类，这是 C++ 多态。

### 6.2 在 DefaultGame.ini 中注册

```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
+AbilitySystemGlobalsClassName="/Script/Aura.AuraAbilitySystemGlobals"
```

完整调用链变为：

```text
ASC::MakeEffectContext
  -> 当前项目的 UAuraAbilitySystemGlobals
  -> AllocGameplayEffectContext
  -> new FAuraGameplayEffectContext
  -> FGameplayEffectContextHandle
```

如果漏掉配置，后续把基类 Context 强转成 Aura Context 会产生类型错误风险。

---

<a id="section-7"></a>
## 七、格挡与暴击信息如何贯穿整条链路

### 7.1 ExecCalc 写入

执行计算从拥有的 Spec 取得 ContextHandle：

```cpp
FGameplayEffectContextHandle EffectContextHandle = Spec.GetContext();
```

判定格挡/暴击后，通过蓝图库 Setter 写回：

```cpp
UAuraAbilitySystemLibrary::SetIsBlockedHit(
    EffectContextHandle, bBlocked);

UAuraAbilitySystemLibrary::SetIsCriticalHit(
    EffectContextHandle, bCriticalHit);
```

### 7.2 蓝图库的类型桥梁

```cpp
FAuraGameplayEffectContext* AuraContext =
    static_cast<FAuraGameplayEffectContext*>(EffectContextHandle.Get());
```

Handle 对外只暴露基类指针，蓝图库集中完成向子类的转换，并提供 BlueprintPure/BlueprintCallable 接口。这样其他业务代码不必到处重复 Cast。

这里使用 `static_cast` 的前提是项目已正确注册自定义 `AbilitySystemGlobals`，保证所有 Context 都确实是 Aura 子类。

### 7.3 AttributeSet 读取

`IncomingDamage` 结算时读取：

```cpp
const bool bBlocked =
    UAuraAbilitySystemLibrary::IsBlockedHit(Props.EffectContextHandle);

const bool bCriticalHit =
    UAuraAbilitySystemLibrary::IsCriticalHit(Props.EffectContextHandle);
```

随后把最终伤害和两个状态一起交给飘字系统。

---

<a id="section-8"></a>
## 八、伤害飘字可视化

### 8.1 C++ 参数扩展

```text
AttributeSet::ShowFloatingText
  (Damage, bBlockedHit, bCriticalHit)

PlayerController::ShowDamageNumber [Client RPC]
  (Damage, TargetCharacter, bBlockedHit, bCriticalHit)

DamageTextComponent::SetDamageText [BlueprintImplementableEvent]
  (Damage, bBlockedHit, bCriticalHit)
```

### 8.2 四种显示组合

| 格挡 | 暴击 | 表现建议 |
|---:|---:|---|
| false | false | 普通颜色，只显示伤害数字 |
| true | false | 显示 Blocked/格挡，伤害已减半 |
| false | true | 暴击颜色与 Critical 提示 |
| true | true | 同时显示格挡与暴击；计算顺序决定最终数值 |

本章把表现选择留给 Widget 蓝图：C++ 只提供权威结果，蓝图负责颜色、文本和动画。这保持了逻辑层与表现层分离。

---

<a id="section-9"></a>
## 九、伤害类型与抗性体系

### 9.1 四种伤害 Tag

```text
Damage.Fire
Damage.Lighting
Damage.Arcane
Damage.Physical
```

### 9.2 四种抗性 Tag 与 Attribute

```text
Attributes.Resistance.Fire      -> ResistanceFire
Attributes.Resistance.Lighting  -> ResistanceLighting
Attributes.Resistance.Arcane    -> ResistanceArcane
Attributes.Resistance.Physical  -> ResistancePhysical
```

四项抗性都是复制的次级属性，完整实现了：

- `FGameplayAttributeData`；
- `ATTRIBUTE_ACCESSORS`；
- `ReplicatedUsing`；
- `OnRep_...`；
- `DOREPLIFETIME_CONDITION_NOTIFY`；
- `TagsToAttribute` 映射。

### 9.3 DamageType 到 Resistance 的映射

```cpp
TMap<FGameplayTag, FGameplayTag> DamageTypesToResistance;
```

```text
Damage.Fire      -> Attributes.Resistance.Fire
Damage.Lighting  -> Attributes.Resistance.Lighting
Damage.Arcane    -> Attributes.Resistance.Arcane
Damage.Physical  -> Attributes.Resistance.Physical
```

为什么用 Map，而不是大量 if/else：

- 每种伤害类型与对应抗性是明确的一对一关系；
- ExecCalc 可以统一遍历；
- 增加新类型时主要扩展数据映射和捕获定义；
- 避免技能、属性集、计算类各自硬编码同一套对应关系。

### 9.4 DamageGameplayAbility 基类

伤害相关配置从通用 `UAuraGameplayAbility` 下移到新基类：

```cpp
UCLASS()
class UAuraDamageGameplayAbility : public UAuraGameplayAbility
{
protected:
    TSubclassOf<UGameplayEffect> DamageEffectClass;
    TMap<FGameplayTag, FScalableFloat> DamageTypes;
};
```

继承关系：

```text
UAuraGameplayAbility
  -> UAuraDamageGameplayAbility
       -> UAuraProjectileSpell
```

这样非伤害技能不再携带 Damage 配置，而所有伤害技能可复用多伤害类型结构。

### 9.5 技能写入多种 SetByCaller

```cpp
for (auto& Pair : DamageTypes)
{
    const float ScaledDamage =
        Pair.Value.GetValueAtLevel(GetAbilityLevel());

    AssignTagSetByCallerMagnitude(
        SpecHandle, Pair.Key, ScaledDamage);
}
```

这也修复了上一提交中 `Damage.GetValueAtLevel(20)` 写死等级的问题，现在真正使用 `GetAbilityLevel()`。

---

<a id="section-10"></a>
## 十、ExecCalc_Damage 的新版计算流程

### 10.1 捕获四项目标抗性

```text
ResistanceFireDef      -> Target, non-snapshot
ResistanceLightingDef  -> Target, non-snapshot
ResistanceArcaneDef    -> Target, non-snapshot
ResistancePhysicalDef  -> Target, non-snapshot
```

`AuraDamageStatics` 增加 `TagsToCaptureDef`，把抗性 Tag 映射到捕获定义。

### 10.2 分类型减伤

核心思想：每种伤害先单独应用自己的抗性，再求和。

```text
TotalDamage = 0

for each (DamageType -> ResistanceTag):
    TypeDamage = Spec.GetSetByCallerMagnitude(DamageType)
    Resistance = Capture(Target, ResistanceTag)
    Resistance = Clamp(Resistance, 0, 100)
    TypeDamage *= (100 - Resistance) / 100
    TotalDamage += TypeDamage
```

例如：

```text
火焰伤害 100，火抗 25%      -> 75
物理伤害 40，物理抗 50%     -> 20
抗性处理后的总伤害           -> 95
```

然后总伤害继续进入：

```text
格挡 -> 护甲穿透/护甲减伤 -> 暴击 -> IncomingDamage
```

### 10.3 计算顺序的含义

当前顺序意味着：

1. 元素/物理抗性先处理每一类基础伤害；
2. 格挡对合计伤害减半；
3. 护甲继续影响合计伤害；
4. 暴击最后翻倍并加入额外暴击伤害。

这属于游戏规则，而不只是代码实现。以后若希望护甲只影响物理伤害，就需要调整结构，在求和前分别处理物理护甲与元素抗性。

---

<a id="section-11"></a>
## 十一、多人模式中的 Authority 与 RPC 修复

### 11.1 客户端访问 GameMode 导致空指针

`GameMode` 只存在于服务器。原先敌人在每台机器都调用默认属性初始化，客户端通过 `GetGameMode()` 取得 `nullptr`，继续使用 CharacterClassInfo 就可能崩溃。

修复：

```cpp
if (HasAuthority())
{
    GiveStartupAbilities(...);
}

if (HasAuthority())
{
    InitializeDefaultAttributes();
}
```

注意：`InitAbilityActorInfo` 仍应在服务器和客户端执行，因为双方 ASC 都需要正确的 Owner/Avatar 信息；只有权威属性写入和能力授予限制在服务器。

### 11.2 伤害飘字发给了错误玩家

旧代码在服务器上使用：

```cpp
UGameplayStatics::GetPlayerController(SourceCharacter, 0)
```

索引 0 只是服务器视角的第一个 PlayerController，并不一定是造成伤害的玩家。因此客户端攻击时，飘字可能显示在服务器玩家窗口。

修复：

```cpp
AAuraPlayerController* PC =
    Cast<AAuraPlayerController>(Props.SourceCharacter->Controller);
```

然后对这个 Controller 调用 Client RPC。服务器拥有所有玩家的服务端 Controller 实例，RPC 会路由到拥有该 Controller 的客户端。

客户端实现再检查：

```cpp
IsLocalController()
```

保证 Widget 只在正确的本地玩家窗口创建。

### 11.3 为什么 IncomingDamage 不复制仍能显示飘字

`IncomingDamage` 是服务器端元属性，不复制。服务器在 `PostGameplayEffectExecute` 中完成结算，然后主动通过 PlayerController Client RPC 把需要的表现数据发送给伤害来源客户端。

```text
服务器权威元属性
  -> 服务器判断最终结果
  -> 定向 Client RPC
  -> 客户端本地 UI
```

这比复制临时元属性更符合事件型表现需求。

---

<a id="section-12"></a>
## 十二、专题：Dedicated Server 火球提前爆炸问题

这是另一个任务中针对本次提交提出的问题。

### 12.1 现象

使用两个独立客户端连接 Dedicated Server 时：

- 火球刚生成就在角色面前播放爆炸，但不造成伤害；
- 真正命中敌人后不再播放爆炸；
- Listen Server 模式看起来正常。

日志中的关键信息：

```text
OtherActor=BP_Aura_C_0
Owner=BP_AuraPlayerState_C_0
Authority=0
bHit=0 -> bHit=1
```

### 12.2 根本原因

旧生成代码使用：

```cpp
Owner      = GetOwningActorFromActorInfo();
Instigator = Cast<APawn>(GetOwningActorFromActorInfo());
```

在 Aura 架构中 OwningActor 是 PlayerState：

```text
OwnerActor  = PlayerState
AvatarActor = AuraCharacter
```

因此：

- 火球实际与 `BP_Aura_C_0` 发生出生重叠；
- `GetOwner()` 却是 `BP_AuraPlayerState_C_0`；
- `OtherActor == GetOwner()` 无法过滤施法角色。

与此同时，`DamageEffectSpecHandle` 只在服务器创建并保存到投射物，客户端副本中通常无效。因此客户端也不能依靠：

```cpp
SpecHandle.Data->GetContext().GetEffectCauser()
```

识别施法者。

客户端于是把“与自己角色的出生重叠”当作第一次命中，播放特效并把本地 `bHit` 设为 true。后续真正碰到敌人时，`bHit` 已经阻止再次播放表现。

服务器的伤害仍可能正常，因为它有 Authority 和有效 Spec；错误主要污染客户端表现状态。

### 12.3 为什么 Listen Server 不容易暴露

Listen Server 主机玩家同时是服务器：

- 拥有 Authority；
- 拥有有效的 DamageEffectSpecHandle；
- 能通过 Context 做更多过滤。

而 Dedicated Server 的两个玩家窗口都只是纯客户端，都会暴露“客户端 Spec 无效但仍自行处理 Overlap 表现”的问题。

### 12.4 本次提交采用的修复

生成投射物时改用 Avatar：

```cpp
Owner      = GetAvatarActorFromActorInfo();
Instigator = Cast<APawn>(GetAvatarActorFromActorInfo());
```

Overlap 先使用所有机器都有的复制身份过滤：

```cpp
if (OtherActor == GetOwner() || OtherActor == GetInstigator())
{
    return;
}
```

只有 Spec 有效时才额外检查 EffectCauser：

```cpp
if (DamageEffectSpecHandle.Data.IsValid() &&
    DamageEffectSpecHandle.Data->GetContext().GetEffectCauser() == OtherActor)
{
    return;
}
```

这正是另一个任务给出的最小修复方向，也与最终提交一致。

### 12.5 更严格的长期方案

当前方案仍让每个客户端根据本地 Overlap 决定播放命中特效。网络位置误差可能令不同客户端产生不同 Overlap 顺序。

更权威的设计是：

```text
服务器检测真正命中
  -> 应用伤害
  -> 服务器确认 Impact
  -> Multicast / replicated event / GameplayCue
  -> 各客户端只播放服务器确认的命中特效
```

当前最小修复足以适配现有结构；当投射物、延迟与预测进一步复杂时，再升级到服务器确认表现更稳妥。

---

<a id="section-13"></a>
## 十三、投射物 Owner、Instigator 与 EffectCauser

这三个概念容易混淆：

| 概念 | 所属系统 | 本章用途 |
|---|---|---|
| Actor Owner | Actor 网络/所有权关系 | 标识投射物属于哪个 Actor，参与复制与 RPC 语义 |
| Actor Instigator | 伤害/行为发起 Pawn | 标识哪个 Pawn 发起行为，适合过滤施法者 |
| EffectContext EffectCauser | GAS 效果来源元数据 | 标识实际造成该 GameplayEffect 的 Actor |

它们可以相同，也可以不同。角色持武器发射投射物时，理想语义可能是：

```text
Projectile Owner       = 施法角色或能力所属 Actor
Projectile Instigator  = 施法 Pawn
GE Instigator          = ASC Owner（Aura 玩家为 PlayerState）
GE EffectCauser        = 角色、武器或投射物，取决于创建 Context 的时机和显式设置
GE SourceObject        = 项目约定的武器/投射物/角色
```

关键不是强行让所有字段相同，而是明确每个字段的消费者和网络可用性。

---

<a id="section-14"></a>
## 十四、本次提交中的其他修复

### 14.1 防止重复命中特效与音效

客户端可能收到多个 Overlap 或在 `OnOverlap` 与 `Destroyed` 两处播放表现。现在使用：

```cpp
if (!bHit)
{
    PlayImpactSound();
    SpawnImpactEffect();
}
```

确保一次投射物只播放一次 Impact 表现。

### 14.2 LoopingSoundComponent 空指针保护

投射物可能在循环音效组件完成创建前被销毁，直接 `Stop()` 会崩溃：

```cpp
if (LoopingSoundComponent)
{
    LoopingSoundComponent->Stop();
}
```

### 14.3 保留 Pitch 瞄准目标

删除：

```cpp
Rotator.Pitch = 0.f;
```

火球现在使用从 Socket 到目标位置的完整旋转，避免发射点较高时水平飞过敌人头顶。

### 14.4 自动寻路空路径保护

```cpp
if (NavPath->PathPoints.IsEmpty()) return;
```

避免对空数组访问最后一个元素。

### 14.5 测试等级恢复

PlayerState 默认等级从上一章临时测试用的 20 恢复为 1。

### 14.6 函数重定向

`IsBlockHit` 重命名为 `IsBlockedHit` 后，在 `DefaultEngine.ini` 增加 Core Redirect，避免已有蓝图引用丢失。

---

<a id="section-15"></a>
## 十五、提交文件清单

### 15.1 新增文件

| 文件 | 作用 |
|---|---|
| `AuraAbilityTypes.h/.cpp` | 自定义 EffectContext、深拷贝、Traits 与网络序列化 |
| `AuraAbilitySystemGlobals.h/.cpp` | 覆盖 Context 分配入口 |
| `AuraDamageGameplayAbility.h/.cpp` | 提取伤害 GE 与多类型伤害配置 |

### 15.2 主要修改文件

| 文件 | 主要变更 |
|---|---|
| `DefaultGame.ini` | 注册 AuraAbilitySystemGlobals |
| `DefaultEngine.ini` | 增加 IsBlockedHit 函数重定向 |
| `AuraAbilitySystemLibrary.h/.cpp` | 自定义 Context 的格挡/暴击 Getter 与 Setter |
| `ExecCalc_Damage.cpp` | 写入格挡暴击；加入四种抗性捕获和分类型减伤 |
| `AuraGameplayTags.h/.cpp` | 四类伤害、四类抗性及对应映射 |
| `AuraAttributeSet.h/.cpp` | 四项复制抗性属性；读取 Context 并定向发送飘字 |
| `AuraProjectileSpell.h/.cpp` | 改继承伤害能力；写入多类 SetByCaller；修正 Owner/Instigator 和瞄准 |
| `AuraProjectile.cpp` | 自身过滤、Spec 有效性、重复表现和音效空指针保护 |
| `AuraEnemy.cpp` | 只在 Authority 初始化属性和授予能力 |
| `AuraPlayerController.h/.cpp` | RPC 增加格挡/暴击参数并限制本地显示 |
| `DamageTextComponent.h` | 蓝图事件增加格挡/暴击输入 |
| `AuraPlayerState.h` | 默认等级恢复为 1 |

---

<a id="section-16"></a>
## 十六、源码校验与当前边界

### 16.1 已完成的闭环

- 自定义 Context 已由项目全局分配器实际创建；
- 格挡/暴击可从服务器 ExecCalc 传到 AttributeSet 与客户端飘字；
- 四种伤害与四种抗性通过 GameplayTag 映射；
- 技能可以配置多种按等级缩放的伤害；
- 客户端不再初始化权威属性或授予启动能力；
- 飘字 RPC 被发送给真正造成伤害的玩家；
- Dedicated Server 下的投射物自身重叠已有 Owner/Instigator 过滤；
- 投射物命中特效和音效增加一次性保护。

### 16.2 当前值得继续改进的部分

1. `Lighting` 应为更常见的 `Lightning`。目前 Tag、属性和代码内部保持一致，因此能运行，但越晚重命名资产迁移成本越高；
2. 当前 `NetSerialize` 对格挡/暴击同时使用存在位和 bool 数据，可精简为直接由 RepBits 恢复；
3. 蓝图库对 Context 使用 `static_cast`，依赖全局配置绝对正确；可在开发构建中先校验 `GetScriptStruct()`，提高错误诊断能力；
4. `GetCharacterClassInfo` 仍可增加更完整的空指针保护，即使调用方原则上只在服务器执行；
5. 当前护甲作用于所有抗性处理后的总伤害，是否应只影响物理伤害需要由游戏规则决定；
6. 投射物 Impact 表现仍主要由各客户端本地 Overlap 决定，严格网络一致性可改为服务器确认事件；
7. Context 演示中加入 SourceObject、Actors 和手工 HitResult 并非都被业务使用，应在理解后删除冗余数据；
8. `DamageTypesToResistance` 与捕获定义必须同步增加，当前用 `checkf` 及早暴露漏配是合理的，但资产/发行版本需要明确错误策略。

---

<a id="section-17"></a>
## 十七、知识点总结

### 17.1 核心职责

| 组件 | 本章职责 |
|---|---|
| 自定义 EffectContext | 携带一次伤害的格挡/暴击结果 |
| NetSerialize | 让自定义 Context 在网络中正确传递 |
| AbilitySystemGlobals | 保证 MakeEffectContext 分配 Aura 子类 |
| Damage GameplayAbility | 配置多种带等级曲线的伤害输入 |
| GameplayTag Map | 建立伤害类型到抗性的统一关系 |
| ExecCalc | 分类型抗性减伤并完成最终伤害计算 |
| AttributeSet | 消费伤害并读取 Context 的附加结果 |
| PlayerController Client RPC | 把飘字只发给正确的本地玩家 |
| Projectile | 服务端伤害、客户端表现与自身过滤 |

### 17.2 多人编程的三个判断问题

写每段 GAS/Actor 逻辑前都问：

```text
1. 这段逻辑应该由谁决定？
   -> 伤害、属性、能力授予：服务器

2. 哪些机器需要看到结果？
   -> 所有人、拥有者客户端，还是仅服务器？

3. 我依赖的数据在那台机器上是否真的存在？
   -> GameMode 只在服务器；未复制的 SpecHandle 客户端无效；
      PlayerController 只在服务器和所属客户端存在。
```

### 17.3 一句话总结

**本次提交把伤害系统从“算出一个数字”推进为可网络传递、可按类型抵抗、可区分格挡暴击表现的完整事件管线，同时通过 Dedicated Server 测试明确了服务器权威数据、Actor 复制身份和客户端本地表现之间的边界。**

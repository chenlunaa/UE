# UE5 学习笔记 - 第十九次提交（第13章：伤害计算管线 - IncomingDamage 元属性与 ExecCalc）

> Commit `b4b2ef3`：实现 Attribute 对伤害的影响机制  
> 日期：2026-08-11  
> 对应视频：13.1~13.17  
> 提交规模：21 个 C++ 文件，新增 412 行、删除 10 行

---

## 目录

- [一、本章目标](#section-1)
- [二、完整伤害流程](#section-2)
- [三、IncomingDamage 元属性](#section-3)
- [四、SetByCaller：由技能传入基础伤害](#section-4)
- [五、受击、死亡、溶解与伤害飘字](#section-5)
- [六、重点：MMC 与 ExecCalc 的区别](#section-6)
- [七、ExecCalc_Damage 的基本流程](#section-7)
- [八、格挡、护甲与暴击公式](#section-8)
- [九、属性捕获的实现方式](#section-9)
- [十、数据驱动与公共能力](#section-10)
- [十一、网络职责](#section-11)
- [十二、提交文件清单](#section-12)
- [十三、源码校验与当前边界](#section-13)
- [十四、知识点总结](#section-14)

---

<a id="section-1"></a>
## 一、本章目标

上一章的伤害 GE 直接修改 `Health`，适合验证投射物命中，却无法在扣血前统一处理格挡、护甲、护甲穿透和暴击。本章把伤害拆成四层：

1. **技能层决定原始伤害**：技能使用 `FScalableFloat Damage`，创建 Spec 时通过 `SetByCaller` 写入数值；
2. **计算层决定最终伤害**：`UExecCalc_Damage` 捕获攻击者与目标的战斗属性，结合等级曲线完成计算；
3. **结算层处理结果**：计算结果先写入 `IncomingDamage` 元属性，`PostGameplayEffectExecute` 再扣减生命值；
4. **表现层响应结果**：非致命伤害触发受击能力，致命伤害触发布娃娃和溶解，同时在攻击者客户端显示伤害数字。

这里最关键的设计变化是：**伤害计算不再直接等于修改生命值**。计算和结算之间增加 `IncomingDamage` 这一层后，所有伤害都能走同一个出口。

---

<a id="section-2"></a>
## 二、完整伤害流程

```text
UAuraProjectileSpell::SpawnProjectile
  |
  | 1. Damage.GetValueAtLevel(...)
  | 2. AssignTagSetByCallerMagnitude(Spec, DamageTag, BaseDamage)
  v
Projectile 保存 DamageEffectSpecHandle
  |
  | 命中目标后，把 GE_Damage Spec 应用到目标 ASC
  v
GE_Damage
  |
  | Executions -> ExecCalc_Damage
  v
UExecCalc_Damage::Execute_Implementation
  |
  |-- 读取 SetByCaller 的基础 Damage
  |-- 捕获 Target.BlockChance
  |-- 捕获 Target.Armor
  |-- 捕获 Source.ArmorPenetration
  |-- 按攻击者/目标等级读取伤害系数曲线
  |-- 计算格挡和有效护甲
  |-- 捕获暴击三属性并判定暴击
  `-- 输出 Additive(IncomingDamage, FinalDamage)
          |
          v
UAuraAttributeSet::PostGameplayEffectExecute
  |
  |-- 读取并立即清零 IncomingDamage
  |-- Health = Clamp(Health - Damage, 0, MaxHealth)
  |-- 致命：CombatInterface::Die()
  |-- 非致命：按 Effects.HitReact Tag 激活受击能力
  `-- ShowFloatingText -> Client RPC -> DamageTextComponent
```

职责边界可以概括为：

| 层 | 负责什么 | 不负责什么 |
|---|---|---|
| GameplayAbility | 决定技能基础伤害并制作 Spec | 不直接扣血 |
| GameplayEffect | 声明采用哪个 Execution | 不承载复杂公式 |
| ExecCalc | 把基础伤害和双方属性变成最终伤害 | 不处理死亡和 UI |
| AttributeSet | 消费最终伤害并改变生命值 | 不关心该伤害来自哪个具体技能 |
| Character / Controller / Widget | 受击、死亡、溶解、飘字 | 不参与数值公式 |

---

<a id="section-3"></a>
## 三、IncomingDamage 元属性

### 3.1 为什么不直接修改 Health

如果每个 GE 都直接减少 `Health`，伤害的中间信息会在属性修改后丢失，并且每种技能容易各写一套结算逻辑。`IncomingDamage` 是一个临时中转属性：

```cpp
UPROPERTY(BlueprintReadOnly, Category="Meta Attributes")
FGameplayAttributeData IncomingDamage;
ATTRIBUTE_ACCESSORS(UAuraAttributeSet, IncomingDamage);
```

它不是角色需要长期保存的状态，而是一次 GE 执行产生的“待结算伤害”。因此本次实现没有为它添加 `ReplicatedUsing`，也没有注册到 `GetLifetimeReplicatedProps`。

### 3.2 结算过程

```cpp
if (Data.EvaluatedData.Attribute == GetIncomingDamageAttribute())
{
    const float LocalIncomingDamage = GetIncomingDamage();
    SetIncomingDamage(0.f);

    if (LocalIncomingDamage > 0.f)
    {
        const float NewHealth = GetHealth() - LocalIncomingDamage;
        SetHealth(FMath::Clamp(NewHealth, 0.f, GetMaxHealth()));

        const bool bFatal = NewHealth <= 0.f;
        // 致命时死亡；否则触发 HitReact；最后显示伤害数字
    }
}
```

先缓存、再清零非常重要。若不清零，下一次伤害会和旧的 `IncomingDamage` 累积，元属性就从“事件载荷”错误地变成了持久状态。

### 3.3 为什么放在 PostGameplayEffectExecute

`PostGameplayEffectExecute` 在 Instant GE 完成属性修改后执行，适合集中处理最终结果：

- Clamp `Health`；
- 判断致命伤害；
- 触发只应发生一次的受击/死亡逻辑；
- 启动伤害反馈。

它比监听 `Health` 的普通变化更适合做伤害结算，因为此处仍能确定本次变更来自 `IncomingDamage`。

---

<a id="section-4"></a>
## 四、SetByCaller：由技能传入基础伤害

### 4.1 Damage 原生 GameplayTag

本次提交新增 `Damage` Tag，用它作为 SetByCaller Map 的键：

```cpp
GameplayTags.Damage = UGameplayTagsManager::Get().AddNativeGameplayTag(
    FName("Damage"), FString("Damage"));
```

Tag 的价值不是显示名称，而是让“写入方”和“读取方”共享一个类型安全、可集中管理的键。

### 4.2 技能侧写入

所有 Aura 技能基类新增：

```cpp
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Damage")
FScalableFloat Damage;
```

投射物法术创建 GE Spec 后写入伤害：

```cpp
UAbilitySystemBlueprintLibrary::AssignTagSetByCallerMagnitude(
    SpecHandle,
    FAuraGameplayTags::Get().Damage,
    Damage.GetValueAtLevel(20));
```

`FScalableFloat` 允许蓝图直接填固定数值，也允许绑定曲线表，使技能伤害可以随技能等级成长。

### 4.3 ExecCalc 侧读取

```cpp
float Damage = Spec.GetSetByCallerMagnitude(
    FAuraGameplayTags::Get().Damage);
```

于是 GE 只负责定义“如何计算”，具体基础伤害由创建 Spec 的技能决定。同一个 `GE_Damage` 和 `ExecCalc_Damage` 可以被多个技能复用。

---

<a id="section-5"></a>
## 五、受击、死亡、溶解与伤害飘字

### 5.1 受击反应

本次新增原生 Tag：`Effects.HitReact`。非致命伤害发生时，`AttributeSet` 不直接播放 Montage，而是请求目标 ASC 激活带该 Tag 的能力：

```cpp
FGameplayTagContainer TagContainer;
TagContainer.AddTag(FAuraGameplayTags::Get().Effects_HitReact);
Props.TargetASC->TryActivateAbilitiesByTag(TagContainer);
```

受击动画通过 `ICombatInterface::GetHitReactMontage()` 获取。敌人监听 `Effects.HitReact` 的 Tag 数量：Tag 存在时令 `MaxWalkSpeed = 0`，移除后恢复 `BaseWalkSpeed`。这样受击状态由 GAS Tag 驱动，移动逻辑不用依赖具体能力类。

### 5.2 公共启动能力

`UCharacterClassInfo` 新增 `CommonAbilities`。敌人初始化 ASC 后调用：

```text
AAuraEnemy::BeginPlay
  -> InitAbilityActorInfo
  -> UAuraAbilitySystemLibrary::GiveStartupAbilities
  -> 遍历 CharacterClassInfo.CommonAbilities
  -> ASC->GiveAbility(...)
```

受击能力可作为所有职业共享的启动能力放入 DataAsset，而不必在每个敌人蓝图中重复配置。

### 5.3 死亡与溶解

致命伤害通过 `ICombatInterface::Die()` 进入角色统一死亡接口：

1. `AAuraEnemy::Die` 设置剩余寿命，之后调用基类；
2. 基类先让武器脱离角色；
3. `MulticastHandleDeath` 在各端启用武器和角色网格体物理模拟；
4. 关闭胶囊体碰撞，形成布娃娃效果；
5. 创建角色/武器动态材质实例；
6. 调用蓝图事件驱动 Dissolve 时间线；
7. `LifeSpan` 到期后销毁敌人。

使用 `NetMulticast, Reliable` 是为了让服务端确认的死亡表现同步到所有客户端。

### 5.4 伤害飘字

`AttributeSet` 只把“伤害值 + 目标角色”交给攻击者的 PlayerController：

```text
AttributeSet::ShowFloatingText
  -> Source PlayerController::ShowDamageNumber (Client, Reliable)
  -> NewObject<UDamageTextComponent>
  -> 挂到目标后立刻以 KeepWorldTransform 分离
  -> 蓝图事件 SetDamageText(Damage)
  -> Widget 动画播放并自行结束
```

先挂载再分离，使组件得到目标的世界位置，但不会继续跟随目标移动。`SourceCharacter != TargetCharacter` 的判断避免自伤时显示当前这套飘字。

---

<a id="section-6"></a>
## 六、重点：MMC 与 ExecCalc 的区别

MMC 指 `UGameplayModMagnitudeCalculation`，本次自定义计算类指 `UGameplayEffectExecutionCalculation`（项目中为 `UExecCalc_Damage`）。二者都会捕获属性、读取 Spec 和 Tag，但解决的问题不同。

| 对比项 | MMC | ExecCalc / Execution Calculation |
|---|---|---|
| 核心问题 | “这个 Modifier 的 Magnitude 是多少？” | “这次执行要读取什么，并向哪些属性输出什么？” |
| 返回方式 | `CalculateBaseMagnitude_Implementation` 返回一个 `float` | `Execute_Implementation` 向 `OutExecutionOutput` 添加零到多个 Modifier |
| 输出范围 | 一次调用为一个 Modifier 计算一个幅值 | 一次执行可以输出多个属性修改 |
| 典型用途 | MaxHealth、MaxMana 等单项派生值 | 伤害、治疗、吸血等多输入、多阶段结算 |
| 属性捕获 | 支持 Source / Target 捕获 | 支持 Source / Target 捕获 |
| 非属性数据 | 可通过 Spec Context、SourceObject、Tag 等读取 | 同样可读取，还能组合更复杂的执行参数与输出 |
| 持续重算 | 适合 Infinite GE 的 Modifier 随依赖变化重新评估 | 只在 Instant 或 Periodic 执行时计算，不是持续依赖关系 |
| 预测 | 可用于普通 Modifier 的幅值计算 | **不支持客户端预测，执行计算只在服务器运行** |
| GE 配置入口 | Modifier -> Custom Calculation Class | Executions -> Calculation Class |
| 当前项目示例 | `MMC_Max_Health`、`MMC_MaxMana` | `ExecCalc_Damage` |

### 6.1 MMC 的思考方式

MMC 是一个“数值函数”。例如最大生命值：

```text
MaxHealth = 80 + 2.5 * Vigor + 10 * CharacterLevel
```

它最终只需要回答一个问题：`MaxHealth` 这个 Modifier 的 Magnitude 是多少。GE 决定将该返回值用 Add、Multiply 或 Override 应用到哪个属性。

### 6.2 ExecCalc 的思考方式

ExecCalc 是一次“结算事务”。本次伤害需要同时读取：

- 来自 Spec 的基础伤害；
- 目标的格挡率、护甲、暴击抗性；
- 来源的护甲穿透、暴击率、暴击伤害；
- 来源和目标等级；
- DataAsset 中的三条等级系数曲线；
- Source / Target 聚合 GameplayTag。

计算结束后，它主动构造输出：

```cpp
const FGameplayModifierEvaluatedData EvaluationData(
    UAuraAttributeSet::GetIncomingDamageAttribute(),
    EGameplayModOp::Additive,
    Damage);
OutExecutionOutput.AddOutputModifier(EvaluationData);
```

当前实现只输出 `IncomingDamage` 一个属性，但 ExecCalc 本身允许一次输出多个 Modifier。这种“多输入 + 分支/随机判定 + 可多输出”的问题正是 Execution Calculation 的适用场景。

### 6.3 为什么本次伤害不继续用 MMC

理论上可以把部分伤害公式写进 MMC，让它返回一个最终 Magnitude，但会有三个问题：

1. 伤害不是某个长期派生属性，而是一次独立事件；
2. 格挡和暴击包含随机分支，且未来可能同时输出伤害、状态和其他元属性；
3. 公式需要同时协调多项 Source/Target 属性与等级曲线，放在单个 Modifier 的“幅值函数”里会模糊职责。

因此本章采用：**MMC 负责单个 Modifier 的值，ExecCalc 负责一次完整战斗结算**。

### 6.4 ExecCalc 的限制

- 不支持预测，应由服务端产生权威结果；
- 只适用于 Instant 或 Periodic GE；
- 捕获属性时不会替你重新执行 `PreAttributeChange` 中的 Clamp，计算类内仍应做必要的非负限制；
- 输出的是属性修改，不应在计算类里直接播放动画、创建 Widget 或销毁 Actor。

---

<a id="section-7"></a>
## 七、ExecCalc_Damage 的基本流程

### 7.1 声明所有捕获项

`AuraDamageStatics` 通过 GAS 宏生成属性捕获定义：

```cpp
DECLARE_ATTRIBUTE_CAPTUREDEF(Armor);
DECLARE_ATTRIBUTE_CAPTUREDEF(ArmorPenetration);
DECLARE_ATTRIBUTE_CAPTUREDEF(BlockChance);
DECLARE_ATTRIBUTE_CAPTUREDEF(CriticalHitChance);
DECLARE_ATTRIBUTE_CAPTUREDEF(CriticalHitResistance);
DECLARE_ATTRIBUTE_CAPTUREDEF(CriticalHitDamage);
```

构造函数明确每个属性来自 Source 还是 Target：

| 捕获属性 | 来源 | 原因 |
|---|---|---|
| Armor | Target | 防御者护甲 |
| BlockChance | Target | 防御者格挡概率 |
| CriticalHitResistance | Target | 防御者暴击抗性 |
| ArmorPenetration | Source | 攻击者护甲穿透 |
| CriticalHitChance | Source | 攻击者暴击概率 |
| CriticalHitDamage | Source | 攻击者额外暴击伤害 |

六项当前都设置 `bSnapshot = false`，即执行时读取当前值，而不是创建 Spec 时冻结数值。

### 7.2 注册相关属性

`UExecCalc_Damage` 构造函数把所有捕获定义加入 `RelevantAttributesToCapture`。只声明宏而不注册，执行时无法成功计算捕获值。

### 7.3 获取执行上下文

执行时从 `ExecutionParams` 取得 Source ASC、Target ASC 和 Avatar，再从 `Spec` 取得双方聚合 Tag：

```cpp
FAggregatorEvaluateParameters EvaluationParameters;
EvaluationParameters.SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
EvaluationParameters.TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();
```

Tag 会参与 GameplayEffect 属性聚合器的条件评估。即使当前公式未手写判断 Tag，也应把它们传给属性捕获计算。

### 7.4 读取输入并依次计算

```text
基础 Damage（SetByCaller）
  -> 格挡判定
  -> 护甲穿透与有效护甲减伤
  -> 暴击率与暴击抗性判定
  -> 保证最终伤害 >= 0
```

计算顺序属于战斗规则的一部分。当前代码先格挡、再护甲、最后暴击，因此一次攻击可以同时格挡并暴击：减半后的伤害再翻倍，并加上额外暴击伤害。

### 7.5 输出到元属性

最终结果不直接写 `Health`，而是输出到 `IncomingDamage`，让 `AttributeSet` 继续完成扣血和后续反应。这保证了“数值计算”和“状态改变”解耦。

---

<a id="section-8"></a>
## 八、格挡、护甲与暴击公式

### 8.1 格挡

```text
bBlocked = Random(1, 100) <= TargetBlockChance
Damage   = bBlocked ? Damage * 0.5 : Damage
```

`BlockChance` 按百分比解释。当前版本只改变伤害值，尚未把 `bBlocked` 写入 EffectContext，所以伤害飘字还不能显示“格挡”。

### 8.2 护甲穿透

先按攻击者等级从 `DamageCalculationCoefficients` 查 `ArmorPenetration` 曲线：

```text
ArmorPenetrationCoefficient = ArmorPenetrationCurve(SourceLevel)

EffectiveArmor = TargetArmor
               * (100 - SourceArmorPenetration * ArmorPenetrationCoefficient)
               / 100
```

然后按目标等级查 `EffectiveArmor` 曲线，将有效护甲转成减伤：

```text
EffectiveArmorCoefficient = EffectiveArmorCurve(TargetLevel)

Damage = Damage
       * (100 - EffectiveArmor * EffectiveArmorCoefficient)
       / 100
```

攻击者等级控制“每点护甲穿透有多有效”，目标等级控制“每点有效护甲能减多少伤害”。两者都由曲线表调参，避免把平衡系数硬编码进 C++。

### 8.3 暴击

先按目标等级取得 `CriticalHitResistance` 系数：

```text
CriticalHitResistanceCoefficient = Curve(TargetLevel)

EffectiveCriticalHitChance = SourceCriticalHitChance
                           - TargetCriticalHitResistance
                           * CriticalHitResistanceCoefficient

bCriticalHit = Random(1, 100) <= EffectiveCriticalHitChance
Damage       = bCriticalHit ? 2 * Damage + SourceCriticalHitDamage : Damage
```

最后执行：

```cpp
Damage = FMath::Max(Damage, 0.f);
```

这样即使配置产生了超高护甲或暴击抗性，也不会输出负伤害。

---

<a id="section-9"></a>
## 九、属性捕获的实现方式

MMC 和 ExecCalc 使用相同的 GAS 捕获概念，但代码组织略有不同。

### 9.1 MMC 中的写法

MMC 通常把捕获定义直接作为类成员：

```cpp
VigorDef.AttributeToCapture = UAuraAttributeSet::GetVigorAttribute();
VigorDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
VigorDef.bSnapshot = false;
RelevantAttributesToCapture.Add(VigorDef);
```

计算时调用：

```cpp
GetCapturedAttributeMagnitude(VigorDef, Spec, EvaluationParameters, Vigor);
```

### 9.2 ExecCalc 中的写法

伤害计算项较多，因此使用 `DECLARE_ATTRIBUTE_CAPTUREDEF` / `DEFINE_ATTRIBUTE_CAPTUREDEF` 宏，并放进函数内静态对象：

```cpp
static const AuraDamageStatics& DamageStatics()
{
    static AuraDamageStatics DStatics;
    return DStatics;
}
```

这不是 ExecCalc 的强制要求，而是一种减少样板代码并让捕获定义只初始化一次的组织方式。执行时调用：

```cpp
ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
    DamageStatics().ArmorDef,
    EvaluationParameters,
    TargetArmor);
```

### 9.3 Source、Target 与 Snapshot

两个维度必须分开理解：

| 配置 | 回答的问题 |
|---|---|
| Source / Target | 从攻击者还是防御者捕获属性？ |
| Snapshot true / false | 在 Spec 创建时冻结，还是在执行/评估时读取当前值？ |

本章全部使用非 Snapshot，因此投射物飞行期间若双方属性变化，命中结算读取的是执行时的当前值。

---

<a id="section-10"></a>
## 十、数据驱动与公共能力

`UCharacterClassInfo` 在上一章保存职业属性 GE，本章继续承担全局战斗配置：

```cpp
UPROPERTY(EditDefaultsOnly, Category="Common Class Defaults")
TArray<TSubclassOf<UGameplayAbility>> CommonAbilities;

UPROPERTY(EditDefaultsOnly, Category="Common Class Defaults|Damage")
TObjectPtr<UCurveTable> DamageCalculationCoefficients;
```

`UAuraAbilitySystemLibrary::GetCharacterClassInfo` 将通过 GameMode 取 DataAsset 的过程集中起来，属性初始化、启动能力授予和伤害公式都复用这一入口。

曲线表当前包含三条关键曲线：

| 曲线名 | 输入等级 | 用途 |
|---|---|---|
| `ArmorPenetration` | Source Level | 缩放攻击者护甲穿透效率 |
| `EffectiveArmor` | Target Level | 缩放目标有效护甲减伤效率 |
| `CriticalHitResistance` | Target Level | 缩放目标暴击抗性效率 |

这使“算法结构”留在 C++，“等级平衡参数”留在曲线资产中。

---

<a id="section-11"></a>
## 十一、网络职责

```text
服务器
  -> 应用伤害 GE
  -> 运行 ExecCalc_Damage（权威随机数与最终伤害）
  -> 修改 IncomingDamage / Health
  -> 判定受击或死亡
  -> 死亡表现通过 NetMulticast 发给所有客户端
  -> 飘字通过 Client RPC 只发给伤害来源玩家

客户端
  -> 接收复制后的 Health 等持久属性
  -> 播放多人可见的死亡表现
  -> 伤害来源客户端创建本地 DamageTextComponent
```

关键点：ExecCalc 不支持预测。格挡和暴击含随机数，如果每个客户端各算一次会产生不同结果，因此只能以服务器结果为准。

---

<a id="section-12"></a>
## 十二、提交文件清单

提交 `b4b2ef3` 共涉及 21 个文件。

### 12.1 新增文件

| 文件 | 作用 |
|---|---|
| `Private/AbilitySystem/ExecCalc/ExecCalc_Damage.cpp` | 捕获双方属性，完成格挡、护甲和暴击计算并输出 IncomingDamage |
| `Public/AbilitySystem/ExecCalc/ExecCalc_Damage.h` | 声明 `UGameplayEffectExecutionCalculation` 子类 |
| `Private/UI/Widget/DamageTextComponent.cpp` | 伤害文本组件实现文件 |
| `Public/UI/Widget/DamageTextComponent.h` | 声明 WidgetComponent 和蓝图事件 `SetDamageText` |

### 12.2 修改文件

| 文件 | 主要变更 |
|---|---|
| `AuraProjectileSpell.cpp` | 创建伤害 Spec 时用 Damage Tag 写入 SetByCaller Magnitude |
| `AuraGameplayAbility.h` | 新增可按等级缩放的 `FScalableFloat Damage` |
| `AuraGameplayTags.h/.cpp` | 新增 `Damage` 与 `Effects.HitReact` 原生 Tag |
| `AuraAttributeSet.h/.cpp` | 新增 IncomingDamage 元属性，集中处理扣血、死亡、受击和飘字 |
| `AuraAbilitySystemLibrary.h/.cpp` | 新增公共能力授予与 CharacterClassInfo 获取函数 |
| `CharacterClassInfo.h` | 新增 `CommonAbilities` 与 `DamageCalculationCoefficients` |
| `CombatInterface.h` | 新增受击 Montage 获取函数与纯虚 `Die` 接口 |
| `AuraCharacterBase.h/.cpp` | 实现受击 Montage、死亡多播、布娃娃与溶解入口 |
| `AuraEnemy.h/.cpp` | 授予公共能力，监听 HitReact Tag，限制移动并设置死亡寿命 |
| `AuraPlayerController.h/.cpp` | 新增只发给所属客户端的伤害数字 RPC |
| `AuraPlayerState.h` | 把默认等级临时改为 20，便于测试等级曲线 |

### 12.3 蓝图与资产侧配套

这些资源不出现在本次 Git 的 C++ 文件统计中，但完整流程需要相应配置：

- `GE_Damage`：移除原来的 IncomingDamage Modifier，在 Executions 中选择 `ExecCalc_Damage`；
- `DA_CharacterClassInfo`：配置公共受击能力和伤害系数曲线表；
- 受击 GameplayAbility：授予/持有 `Effects.HitReact` Tag，播放接口返回的 Montage；
- 敌人蓝图：配置 HitReact Montage、溶解材质和时间线；
- PlayerController 蓝图：配置 DamageTextComponentClass；
- DamageText Widget：实现 `SetDamageText` 并播放飘字动画。

---

<a id="section-13"></a>
## 十三、源码校验与当前边界

### 13.1 已完成的闭环

- 基础伤害已经从 GE 硬编码值迁移到技能 `FScalableFloat + SetByCaller`；
- 最终伤害已通过 `ExecCalc_Damage` 受六项战斗属性影响；
- 伤害通过非复制元属性进入统一结算；
- 非致命受击、致命死亡、布娃娃、溶解和本地飘字已接入；
- 伤害公式的三个等级系数已经改为曲线表驱动；
- 公共能力可由 CharacterClassInfo 集中授予敌人。

### 13.2 当前仍是测试性或待加强的部分

1. `Damage.GetValueAtLevel(20)` 仍写死为 20，应改成 `Damage.GetValueAtLevel(GetAbilityLevel())`，否则技能等级不会真正控制基础伤害；
2. `AAuraPlayerState::Level` 默认值被临时改为 20，这是曲线测试值，不应视为最终角色等级规则；
3. `GetCharacterClassInfo`、曲线查找和 `ICombatInterface` 转换后缺少完整空指针保护，资产漏配时可能崩溃；
4. `GetSetByCallerMagnitude` 未传入告警控制和缺省值，调用方漏写 Damage Tag 时需要明确诊断策略；
5. 当前只把最终伤害输出到 `IncomingDamage`，没有把 `bBlocked` / `bCriticalHit` 写入自定义 EffectContext，因此飘字无法区分普通、格挡和暴击；
6. 敌人获得公共能力的调用没有显式 Authority 判断。`GiveAbility` 应由服务器执行，当前依赖运行上下文，后续宜明确守卫；
7. 当前公式允许一次攻击既格挡又暴击，这是计算顺序自然产生的规则，需要确认是否符合最终设计；
8. 伤害飘字使用 `GetPlayerController(..., 0)` 查找控制器，更稳妥的实现应从 SourceActor/Controller 关系取得真正的来源控制器，尤其是在多人环境中。

---

<a id="section-14"></a>
## 十四、知识点总结

### 14.1 四个最重要的概念

| 概念 | 一句话理解 |
|---|---|
| SetByCaller | Spec 创建后，由技能用 GameplayTag 动态写入本次效果的输入数值 |
| Meta Attribute | 不长期保存，只把一次计算结果送入统一结算出口的临时属性 |
| MMC | 为 GE 的某一个 Modifier 计算一个 Magnitude |
| ExecCalc | 执行一次权威的复杂结算，并显式输出零到多个属性修改 |

### 14.2 选择规则

```text
只需要算一个 Modifier 的值？
  -> 优先用 Attribute-Based 或 MMC

需要多个 Source/Target 属性、随机分支、分阶段公式或多个输出？
  -> 使用 ExecCalc

需要处理扣血后的死亡、受击、UI 等副作用？
  -> 放在 AttributeSet / Character / Controller，不放进 ExecCalc
```

### 14.3 一句话总结

**本次提交建立了完整的权威伤害管线：技能通过 SetByCaller 提供基础伤害，ExecCalc 结合双方属性和等级曲线算出最终伤害，IncomingDamage 负责把结果交给 AttributeSet 统一扣血，角色与 UI 再分别处理受击、死亡和视觉反馈。**

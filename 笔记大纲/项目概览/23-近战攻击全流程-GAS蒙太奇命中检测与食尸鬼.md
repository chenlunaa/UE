# UE5 学习笔记 - 第二十三次提交（第16章：近战攻击完整闭环）

> Commit `691d67e`：完成了近战敌人的攻击全流程  
> 日期：2026-08-16  
> 对应视频：16.1～16.13  
> 提交规模：18 个源码文件，新增 285 行、删除 50 行  
> 本章边界：近战敌人已经能选取目标、播放攻击蒙太奇、检测命中并通过 GAS 造成伤害；远程弹弓攻击留待下一阶段

本章承接第 15 章的 AI 行为树。上一章的 `BT_Attack` 只绘制红色调试球，本章把这个占位任务接入 Gameplay Ability System，形成从 AI 决策到伤害结算、伤害飘字和多人同步的完整近战攻击流程。

---

## 目录

- [一、本章最终形成的攻击闭环](#section-1)
- [二、用 GameplayTag 连接行为树与 GAS](#section-2)
- [三、按职业数据驱动授予启动能力](#section-3)
- [四、创建近战 GameplayAbility 与攻击蒙太奇](#section-4)
- [五、CombatTarget 与 Motion Warping 面向目标](#section-5)
- [六、Anim Notify 到 Gameplay Event 的攻击时机](#section-6)
- [七、按 Socket 定位攻击命中点](#section-7)
- [八、查询半径内存活角色](#section-8)
- [九、CauseDamage 与 GAS 伤害管线](#section-9)
- [十、伤害飘字与多人测试修复](#section-10)
- [十一、从硬编码长矛到 FTaggedMontage](#section-11)
- [十二、武器、左手和右手 Socket 的统一接口](#section-12)
- [十三、创建食尸鬼敌人的蓝图流程](#section-13)
- [十四、食尸鬼左右手攻击蒙太奇](#section-14)
- [十五、友伤、死亡、溶解与移动优化](#section-15)
- [十六、源码审查与当前边界](#section-16)
- [十七、本次提交文件清单与总结](#section-17)

---

<a id="section-1"></a>
## 一、本章最终形成的攻击闭环

```text
Behavior Tree 判断敌人进入攻击范围
  -> BT_Attack 读取 TargetToFollow
  -> EnemyInterface::SetCombatTarget
  -> ASC::TryActivateAbilitiesByTag(Abilities.Attack)
  -> GA_MeleeAttack 激活
  -> 从角色的 AttackMontages 随机选择 FTaggedMontage
  -> UpdateFacingTarget + Motion Warping
  -> PlayMontageAndWait
  -> 蒙太奇 Notify 发送 Montage.Attack.* Gameplay Event
  -> Wait Gameplay Event 收到精确标签
  -> GetCombatSocketLocation(MontageTag)
  -> GetLivePlayerWithinRadius
  -> IsNotFriend 过滤友方
  -> CauseDamage(TargetActor)
  -> GE_Damage + SetByCaller Damage.Physical
  -> ExecCalc_Damage 处理抗性、格挡、护甲与暴击
  -> AttributeSet 扣除生命并显示伤害飘字
```

这一结构的关键是把职责拆开：

| 层 | 负责内容 |
|---|---|
| Behavior Tree | 决定何时攻击、攻击哪个目标 |
| GameplayTag | 用抽象标签激活对应攻击能力 |
| GameplayAbility | 管理蒙太奇、事件等待、命中检测和技能结束 |
| Character / Interface | 提供战斗目标、攻击蒙太奇和 Socket 位置 |
| AbilitySystemLibrary | 提供范围查询和阵营判断等通用算法 |
| GameplayEffect / ExecCalc | 负责最终权威伤害计算 |

---

<a id="section-2"></a>
## 二、用 GameplayTag 连接行为树与 GAS

### 2.1 新增攻击能力标签

原生 Tag 新增：

```cpp
GameplayTags.Abilities_Attack =
    UGameplayTagsManager::Get().AddNativeGameplayTag(
        FName("Abilities.Attack"),
        FString("Attack Ability Tag"));
```

这个标签不是具体技能名，而是“攻击能力”这个语义类别。近战、远程和法术攻击都可以携带它，行为树无需知道敌人的具体能力类。

### 2.2 BT_Attack 蓝图操作

在 `BT_Attack` 蓝图任务中：

1. 删除上一章的 `Draw Debug Sphere`；
2. 从 `Controlled Pawn` 调用 `Get Ability System Component`；
3. 新建变量 `AttackTag`，类型 `Gameplay Tag`，默认值设为 `Abilities.Attack`；
4. 从 `AttackTag` 创建 `Gameplay Tag Container`；
5. 调用 `Try Activate Abilities by Tag`；
6. 无论是否成功，都调用 `Finish Execute`，避免行为树卡住。

```text
Receive Execute AI
  -> Controlled Pawn
  -> Get Ability System Component
  -> Make Gameplay Tag Container(Abilities.Attack)
  -> Try Activate Abilities by Tag
  -> Finish Execute
```

行为树现在只发出“攻击”意图。战士拥有近战能力就激活近战，游侠以后拥有远程能力就激活远程，任务本身不需要 `Cast To Goblin` 或分支判断职业。

---

<a id="section-3"></a>
## 三、按职业数据驱动授予启动能力

### 3.1 CharacterClassInfo 扩展

`FCharacterClassDefaultInfo` 新增：

```cpp
UPROPERTY(EditDefaultsOnly, Category="Class Defaults")
TArray<TSubclassOf<UGameplayAbility>> StartUpAbilities;
```

每个职业现在除了主属性 GE，还能配置自己的启动技能。数据资源中的 Warrior 可配置 `GA_MeleeAttack`，Ranger 和 Elementalist 后续配置各自攻击能力。

### 3.2 GiveStartupAbilities

函数签名增加 `ECharacterClass`：

```cpp
void UAuraAbilitySystemLibrary::GiveStartupAbilities(
    const UObject* WorldContextObject,
    UAbilitySystemComponent* ASC,
    ECharacterClass CharacterClass);
```

授予顺序：

```text
GetCharacterClassInfo
  -> Give CommonAbilities（固定等级 1）
  -> GetClassDefaultInfo(CharacterClass)
  -> 遍历 StartUpAbilities
  -> 从 ASC Avatar 获取 CombatInterface
  -> 使用角色等级创建 FGameplayAbilitySpec
  -> ASC::GiveAbility
```

角色专属能力按敌人等级授予，因此同一 `GA_MeleeAttack` 在 1 级和 40 级敌人身上能读取不同的 `GetAbilityLevel()`，再从 `FScalableFloat` 曲线得到不同伤害。

`AAuraEnemy::BeginPlay` 仍只在 `HasAuthority()` 时调用授予函数，避免客户端重复授予权威能力。

---

<a id="section-4"></a>
## 四、创建近战 GameplayAbility 与攻击蒙太奇

### 4.1 C++ 能力类

新增 `UAuraMeleeAttack`，继承 `UAuraDamageGameplayAbility`：

```text
UAuraGameplayAbility
  -> UAuraDamageGameplayAbility
       -> UAuraMeleeAttack
```

当前 C++ 子类本身为空，主要作用是提供清晰的类型层级，并让蓝图能力继承伤害配置和 `CauseDamage`。

### 4.2 GA_MeleeAttack 蓝图

1. 基于 `UAuraMeleeAttack` 创建 `GA_MeleeAttack`；
2. Ability Tags 添加 `Abilities.Attack`；
3. Instancing Policy 设为 `Instanced Per Actor`；
4. Damage Effect Class 设为 `GE_Damage`；
5. Damage Types 添加 `Damage.Physical`；
6. `FScalableFloat` 绑定近战伤害曲线。

示例伤害曲线：

| 等级 | 物理伤害 |
|---:|---:|
| 1 | 5 |
| 2 | 7.5 |
| 40 | 50 |

### 4.3 第一版长矛攻击蒙太奇

从哥布林长矛攻击动画创建 `AM_Attack_GoblinSpear`。在 `GA_MeleeAttack` 中使用 `Play Montage and Wait`，最初可以先硬编码该蒙太奇验证系统：

```text
Activate Ability
  -> Play Montage and Wait(AM_Attack_GoblinSpear)
  -> Completed -> End Ability
  -> Interrupted/Cancelled -> End Ability
```

这是第一阶段的“先跑通再泛化”。后续再改为从角色接口获取随机蒙太奇。

---

<a id="section-5"></a>
## 五、CombatTarget 与 Motion Warping 面向目标

### 5.1 为什么需要 CombatTarget

行为树知道 `TargetToFollow` 是 Aura，但 GameplayAbility 只知道自己的 Avatar。如果能力要在攻击开始时让敌人转向目标，就需要把行为树目标传递给敌人本体。

`EnemyInterface` 新增：

```cpp
UFUNCTION(BlueprintCallable, BlueprintNativeEvent)
void SetCombatTarget(AActor* InCombatTarget);

UFUNCTION(BlueprintCallable, BlueprintNativeEvent)
AActor* GetCombatTarget() const;
```

`AAuraEnemy` 保存：

```cpp
UPROPERTY(BlueprintReadWrite, Category="Combat")
TObjectPtr<AActor> CombatTarget;
```

### 5.2 BT_Attack 设置目标

蓝图任务新增 `CombatTargetSelector`，类型 `Blackboard Key Selector`，打开眼睛图标后，在行为树近战和远程 Attack 节点上绑定 `TargetToFollow`。

```text
CombatTargetSelector
  -> Get Blackboard Value as Actor
  -> Is Valid?
      true  -> Set Combat Target(Controlled Pawn, Target)
             -> Try Activate Abilities by Tag
      false -> Finish Execute(false)
```

设置目标必须发生在能力激活之前，否则 `GA_MeleeAttack` 激活时读取到的仍是空指针或旧目标。

### 5.3 Motion Warping 蓝图配置

在 `BP_EnemyBase`：

1. 添加 `Motion Warping Component`；
2. 实现 CombatInterface 的 `Update Facing Target` 事件；
3. 调用 `Add or Update Warp Target from Location`；
4. Warp Target Name 统一使用 `FacingTarget`；
5. Location 使用传入的 Target Vector。

在攻击蒙太奇：

1. 添加 `Motion Warping` Notify State；
2. 覆盖攻击前摇到突刺/挥击开始的窗口；
3. Warp Target Name 同样填 `FacingTarget`；
4. 关闭 Warp Translation，只启用旋转；
5. Rotation Type 设为 `Facing`；
6. 确认源动画启用了 Root Motion。

能力激活后先从 EnemyInterface 获取 `CombatTarget`，取其 Actor Location，调用 `Update Facing Target`，然后播放蒙太奇。

当前 Warp Target 只在攻击开始时更新一次。玩家在攻击窗口中快速移动时，敌人不会每帧继续修正朝向；若要更强追踪，需要计时器或持续更新 Warp Target。

---

<a id="section-6"></a>
## 六、Anim Notify 到 Gameplay Event 的攻击时机

攻击不能在蒙太奇开始时立即造成伤害。真正命中时机应由动画中武器到达目标位置的那一帧决定。

### 6.1 蒙太奇操作

1. 在攻击蒙太奇新增 `Event` Notify Track；
2. 在长矛最大伸展位置添加自定义 `AnimNotify_MontageEvent`；
3. 最初使用 `Event.Montage.Attack.Melee`，泛化后改为 `Montage.Attack.Weapon`；
4. Notify 触发时对 Montage Owner 调用 `Send Gameplay Event to Actor`。

### 6.2 能力监听

在 `GA_MeleeAttack`：

```text
Activate Ability
  |-- Play Montage and Wait
  `-- Wait Gameplay Event(MontageTag, Only Exact Match)
        -> Event Received
        -> 执行 Socket 查询和伤害
        -> End Ability
```

`Only Match Exact` 可以避免父标签或相邻攻击事件误触发当前命中逻辑。蒙太奇的 Completed、Interrupted 和 Cancelled 也要结束能力，尤其是受击反应打断攻击时，否则攻击能力可能永远保持激活。

---

<a id="section-7"></a>
## 七、按 Socket 定位攻击命中点

### 7.1 CombatInterface 改为 BlueprintNativeEvent

```cpp
UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
FVector GetCombatSocketLocation(const FGameplayTag& MontageTag);
```

因为它是 `BlueprintNativeEvent`，C++ 外部不能再直接调用接口虚函数，而应使用自动生成的执行包装：

```cpp
ICombatInterface::Execute_GetCombatSocketLocation(
    AvatarActor, MontageTag);
```

本章测试中曾因 `AuraProjectileSpell` 仍直接调用接口函数触发引擎检查，调用栈明确提示“不要直接调用 Event function，要调用 Execute_...”。这也是 BlueprintNativeEvent 接口最重要的调用规则之一。

### 7.2 三种 Socket 来源

```cpp
if (MontageTag == Montage.Attack.Weapon && IsValid(Weapon))
    return Weapon->GetSocketLocation(WeaponTipSocketName);

if (MontageTag == Montage.Attack.LeftHand)
    return GetMesh()->GetSocketLocation(LeftHandSocketName);

if (MontageTag == Montage.Attack.RightHand)
    return GetMesh()->GetSocketLocation(RightHandSocketName);
```

角色基类新增 `LeftHandSocketName` 和 `RightHandSocketName`。有武器的哥布林配置 `WeaponTipSocketName = TipSocket`；无武器的食尸鬼在骨骼网格的左右腕骨上创建 `LeftHandSocket`、`RightHandSocket`。

使用 Debug Sphere 验证时，球体应出现在长矛尖端或对应拳头，而不是角色原点。若位置错误，先检查蓝图 Class Defaults 中 Socket Name 是否填写，而不是先怀疑范围查询。

---

<a id="section-8"></a>
## 八、查询半径内存活角色

### 8.1 CombatInterface 的生命状态

新增接口：

```cpp
UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
bool IsDead() const;

UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
AActor* GetAvatar();
```

`AAuraCharacterBase` 保存 `bDead`，并在 `MulticastHandleDeath` 中设为 true。因为死亡多播在服务器和客户端执行，双方都能通过接口读取一致的本地死亡状态。

### 8.2 GetLivePlayerWithinRadius

```cpp
void UAuraAbilitySystemLibrary::GetLivePlayerWithinRadius(
    const UObject* WorldContextObject,
    TArray<AActor*>& OutOverlappingActors,
    const TArray<AActor*>& ActorToIgnore,
    float Radius,
    const FVector& SphereOrigin)
```

实现步骤：

```text
FCollisionQueryParams
  -> AddIgnoredActors（通常忽略技能拥有者）
  -> World->OverlapMultiByObjectType
       Object Type = AllDynamicObjects
       Shape = Sphere(Radius)
  -> 遍历 FOverlapResult
  -> Actor Implements UCombatInterface
  -> Execute_IsDead(Actor) == false
  -> AddUnique 到输出数组
```

判断顺序很重要：必须先确认对象实现 `CombatInterface`，再调用 `Execute_IsDead`。本章测试场景中的 `BP_TestActor` 不实现接口，原逻辑直接调用 Execute 导致检查失败；利用 `&&` 的短路求值即可避免：

```cpp
if (Actor->Implements<UCombatInterface>() &&
    !ICombatInterface::Execute_IsDead(Actor))
{
    OutOverlappingActors.AddUnique(Actor);
}
```

在蓝图中把 Avatar 接成 Ignore Actors，把 Socket Location 作为 Sphere Origin，半径约 45；对输出数组 `ForEachLoop`，逐个判断阵营并造成伤害。

---

<a id="section-9"></a>
## 九、CauseDamage 与 GAS 伤害管线

### 9.1 通用伤害函数

`UAuraDamageGameplayAbility` 新增蓝图可调用函数：

```cpp
void UAuraDamageGameplayAbility::CauseDamage(AActor* TargetActor)
{
    FGameplayEffectSpecHandle Spec =
        MakeOutgoingGameplayEffectSpec(DamageEffectClass, 1.f);

    for (const auto& Pair : DamageTypes)
    {
        const float Damage =
            Pair.Value.GetValueAtLevel(GetAbilityLevel());

        UAbilitySystemBlueprintLibrary::
            AssignTagSetByCallerMagnitude(
                Spec, Pair.Key, Damage);
    }

    GetAbilitySystemComponentFromActorInfo()
        ->ApplyGameplayEffectSpecToTarget(
            *Spec.Data.Get(),
            UAbilitySystemBlueprintLibrary::
                GetAbilitySystemComponent(TargetActor));
}
```

它把近战和投射物共有的“根据能力等级计算 DamageTypes，然后写入 SetByCaller”流程集中起来。蓝图只需调用 `CauseDamage(Array Element)`。

### 9.2 为什么 GE_Damage 没有 Modifier 也能扣血

`GE_Damage` 使用 `ExecCalc_Damage`。DamageTypes 中的 `Damage.Physical` 作为 SetByCaller 输入，执行计算依次处理：

```text
物理伤害曲线值
  -> 目标 Physical Resistance
  -> 格挡
  -> 护甲穿透与有效护甲
  -> 暴击
  -> IncomingDamage
  -> PostGameplayEffectExecute
  -> Health 扣除 / HitReact / Death
```

因此蓝图不用手动减 Health，也不应绕过 ExecCalc。复用同一个 GE 可以保证近战攻击和投射物遵循相同的抗性、护甲、格挡、暴击和飘字规则。

---

<a id="section-10"></a>
## 十、伤害飘字与多人测试修复

### 10.1 敌人伤害玩家时的飘字归属

原逻辑只尝试从 SourceCharacter 取得 `AAuraPlayerController`。玩家攻击敌人时成立，但敌人没有 PlayerController，因此敌人攻击玩家时不会显示飘字。

现在先尝试 Source Controller；失败时再尝试 Target Controller：

```text
Source 是玩家
  -> 向攻击者客户端显示目标受到的伤害

Source 是敌人、Target 是玩家
  -> 向受击玩家客户端显示自己受到的伤害
```

第一个分支成功后立即 `return`，避免同一伤害重复显示。

### 10.2 HitReact 黑板空指针

Listen Server 两人测试发现：HitReact GameplayTag 回调会在服务器和客户端都触发，但客户端不存在权威 `AuraAIController`。旧代码直接访问 Blackboard，导致无效内存访问。

修复：

```cpp
if (AuraAIController &&
    AuraAIController->GetBlackboardComponent())
{
    SetValueAsBool("HitReacting", bHitReacting);
}
```

随后分别测试：

- 单人 Listen Server；
- 两名玩家 Listen Server；
- Dedicated Server + 两个纯客户端；
- 敌人受到 HitReact、玩家被近战伤害、伤害飘字与生命球变化。

这次问题再次说明：GameplayTag/ASC 状态可能在客户端存在，但 AIController 和 BehaviorTree 只在服务器权威侧运行。

---

<a id="section-11"></a>
## 十一、从硬编码长矛到 FTaggedMontage

### 11.1 为什么只保存 UAnimMontage 不够

食尸鬼有左手和右手攻击，哥布林有武器攻击。选择哪段蒙太奇，决定了命中检测应使用哪个 Socket，同时也决定能力要等待哪个 Gameplay Event。

因此新增：

```cpp
USTRUCT(BlueprintType)
struct FTaggedMontage
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    UAnimMontage* Montage = nullptr;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FGameplayTag MontageTag;
};
```

角色基类保存：

```cpp
UPROPERTY(EditAnywhere, Category="Combat")
TArray<FTaggedMontage> AttackMontages;
```

CombatInterface 提供 `GetAttackMontages()`，能力不再依赖某个具体敌人类。

### 11.2 GA_MeleeAttack 的随机选择

蓝图流程：

```text
Get Attack Montages(Avatar)
  -> Length > 0
  -> Random Integer in Range(0, Length - 1)
  -> Get
  -> Promote to Variable: TaggedMontage
  -> Break TaggedMontage
       Montage    -> Play Montage and Wait
       MontageTag -> Wait Gameplay Event
                  -> Get Combat Socket Location
```

必须把随机结果提升为变量。`Random Integer in Range` 是 Blueprint Pure 节点，每次执行链读取它都可能重新计算；如果 Montage 和 MontageTag 分别从原纯节点读取，可能播放左手动画却监听右手事件，能力就永远等不到正确 Notify。

同时必须在读取索引前检查 `Length > 0`。本章食尸鬼初建时 AttackMontages 为空，直接 `Get(0)` 产生蓝图运行时数组越界。

---

<a id="section-12"></a>
## 十二、武器、左手和右手 Socket 的统一接口

新增原生 Tag：

```text
Montage.Attack.Weapon
Montage.Attack.LeftHand
Montage.Attack.RightHand
```

它们同时承担三种职责：

1. `FTaggedMontage` 中标识这段蒙太奇属于哪个攻击部位；
2. Anim Notify 发送对应 Gameplay Event；
3. `GetCombatSocketLocation(MontageTag)` 选择 Weapon、LeftHand 或 RightHand Socket。

```text
FTaggedMontage
  Montage = AM_Ghoul_Attack_L
  MontageTag = Montage.Attack.LeftHand
                  |
                  |-- Wait Gameplay Event 监听该 Tag
                  |-- Anim Notify 发送该 Tag
                  `-- Socket 查询选择 LeftHandSocketName
```

课程采用三个明确 `if`。若以后出现尾巴、头部、盾牌或多把武器，可以把 `MontageTag -> SocketName/SocketSource` 改成数据表或 Map，减少硬编码分支。

---

<a id="section-13"></a>
## 十三、创建食尸鬼敌人的蓝图流程

### 13.1 角色与动画蓝图

1. 基于 `BP_EnemyBase` 创建 `BP_Ghoul`；
2. 基于 `ABP_Enemy`，选择 Ghoul Skeleton 创建 `ABP_Ghoul`；
3. 为 BP 设置 Ghoul Skeletal Mesh 并调整网格相对胶囊的位置；
4. Weapon 保持空；
5. Character Class 保持 Warrior，以获得 `GA_MeleeAttack`；
6. 设置 LeftHandSocketName 和 RightHandSocketName；
7. 创建并设置 `AM_HitReact_Ghoul`；
8. 确认继承的 BehaviorTree 与 AIController 配置。

### 13.2 Idle/Walk Blend Space

1. 创建 `BS_Ghoul_IdleWalk`；
2. 水平轴命名 Speed，范围 0～100；
3. 0 放 Idle，100 放 Walk；
4. 在 `ABP_Ghoul` 的资产覆盖中替换基础 Idle/Walk/Run；
5. Sample Weight Speed 约设为 4，平滑停止与起步；
6. 按体型降低 BaseWalkSpeed，并调整 CharacterMovement Rotation Rate。

首次测试应先验证“生成、追踪、受击、HitReact”是否正常，再继续配置攻击蒙太奇，避免同时排查 AI、动画和伤害三套系统。

---

<a id="section-14"></a>
## 十四、食尸鬼左右手攻击蒙太奇

分别从左右手攻击动画创建：

- `AM_Ghoul_Attack_L`
- `AM_Ghoul_Attack_R`

每个蒙太奇都需要：

1. 启用源动画 Root Motion；
2. 添加 Motion Warping Notify State；
3. Warp Target Name = `FacingTarget`；
4. 只 Warp Rotation，Rotation Type = Facing；
5. 在挥击命中帧添加 Montage Event Notify；
6. 左手动画发送 `Montage.Attack.LeftHand`；
7. 右手动画发送 `Montage.Attack.RightHand`。

然后在 `BP_Ghoul.AttackMontages` 配置两个 `FTaggedMontage` 元素。注意资源原始命名可能把左右手标反，不能只看文件名；应在蒙太奇预览中确认实际挥动的手，再绑定 Tag 和 Socket。

若 `Play Montage and Wait` 配置为能力结束时自动 Stop Montage，可能在 Notify 后立即截断动作。本章取消该设置，让攻击动画自然完成；但在 HitReact 能力中配置 `Cancel Abilities with Tag = Abilities.Attack`，使受击仍能中断正在进行的攻击。

---

<a id="section-15"></a>
## 十五、友伤、死亡、溶解与移动优化

### 15.1 阵营过滤

`IsNotFriend` 使用 Actor Tag，而不是 GAS GameplayTag：

```cpp
Player + Player -> friend -> false
Enemy  + Enemy  -> friend -> false
Player + Enemy  -> not friend -> true
Enemy  + Player -> not friend -> true
```

在 `GA_MeleeAttack` 的范围结果循环中：

```text
IsNotFriend(Avatar, OverlapActor)
  -> true  -> CauseDamage
  -> false -> 跳过
```

这样同一个近战能力也能用于玩家宠物或玩家近战，不必写成“只伤害 Player”。

### 15.2 降低敌人生命用于闭环测试

敌人最大生命由 Vitality、Level 和 MMC 公式共同决定。课程临时把敌方次级属性/最大生命结果乘以约 0.25，使敌人更容易死亡，以便验证死亡、碰撞关闭、溶解和 LifeSpan。这里应理解为测试与平衡参数，不是固定设计结论。

### 15.3 食尸鬼溶解材质

1. 从已有 Dissolve 材质复制溶解节点组；
2. Ghoul 材质 Blend Mode 改为 Masked；
3. 把溶解输出接到 Opacity Mask；
4. 创建 `MI_Ghoul_Dissolve`；
5. 设置初始 Dissolve 参数为约 -2，保证存活时完整显示；
6. 在 `BP_Ghoul` 配置 Dissolve Material Instance；
7. 击杀后确认时间线更新标量并完全消失。

### 15.4 RVO 避障的取舍

CharacterMovement 的 `Use RVO Avoidance` 可让服务器 AI 互相绕开，减少拥堵；但没有侧移动画时，角色可能面向目标横向滑动，视觉效果不自然。本章展示后选择关闭。若以后启用，应配合 Strafe 动画、朝向策略和网络表现一起设计。

---

<a id="section-16"></a>
## 十六、源码审查与当前边界

### 16.1 GetLivePlayerWithinRadius 命名不准确

函数实际返回“所有实现 CombatInterface 且未死亡的动态 Actor”，既可能是玩家，也可能是敌人。更准确的名称是 `GetLiveActorsWithinRadius` 或 `GetLiveCombatantsWithinRadius`。

### 16.2 CauseDamage 需要有效性检查

当前代码直接解引用 `DamageSpecHandle.Data`，并假设 TargetActor 有 ASC。后续建议检查：

```cpp
if (!IsValid(TargetActor) ||
    !DamageSpecHandle.IsValid() ||
    !IsValid(TargetASC))
{
    return;
}
```

还可把 Spec Level 从固定 1 改为 `GetAbilityLevel()`，让 Effect 本身的等级语义与能力等级保持一致。当前 DamageTypes 已使用能力等级取曲线值，因此实际伤害缩放仍能工作。

### 16.3 IsNotFriend 的边界

当前函数没有空指针保护。两个 Actor 若都没有 `Player`/`Enemy` Tag，会返回“不是朋友”；一个对象若错误地同时拥有两种 Tag，结果也可能不符合预期。长期方案可以使用 Team ID、Generic Team Agent Interface 或项目专用阵营组件。

### 16.4 HitReact 的临时日志

`HitReactTagChange` 中仍有 `UE_LOG(LogTemp, Warning, ...)`。它适合多人排查 Authority 和回调次数，完成验证后应移入项目日志类别、降低级别或删除，避免战斗中大量刷日志。

### 16.5 AttackMontages 空数组

蓝图已经在使用前检查 Length，但更稳妥的 C++/蓝图 API 可以直接提供 `GetRandomAttackMontage`，在内部处理空数组、随机索引和结果有效性，避免每个能力重复实现。

### 16.6 动画资产不在本次源码差异中

本次 Git 提交记录的是 18 个 C++ 文件。蒙太奇、Anim Notify、Motion Warping、行为树节点、食尸鬼蓝图、材质实例和测试地图属于 Content 资产；当前仓库配置未把这些资产作为本次源码差异列出。复现功能时不能只复制 C++，还要同步对应编辑器配置。

### 16.7 当前功能边界

- 近战命中仍是单帧球形 Overlap，不是连续武器 Sweep；
- 快速挥动可能在两帧之间穿过目标，复杂武器可改为上一帧到当前帧的 Sphere/Capsule Trace；
- CombatTarget 在攻击开始时采样，攻击过程中不持续追踪；
- `GetAllDynamicObjects` 查询范围较宽，后续可用专用 Object Channel 降低无关重叠；
- 远程弹弓、弓箭形变和投射物攻击尚未实现。

---

<a id="section-17"></a>
## 十七、本次提交文件清单与总结

### 17.1 新增文件

| 文件 | 作用 |
|---|---|
| `AuraMeleeAttack.h/.cpp` | 近战攻击 C++ GameplayAbility 类型，继承通用伤害能力 |

### 17.2 主要修改文件

| 文件 | 主要变更 |
|---|---|
| `AuraDamageGameplayAbility.h/.cpp` | 新增 `CauseDamage`，创建 GE Spec、写入各类 SetByCaller 并应用到目标 |
| `AuraAbilitySystemLibrary.h/.cpp` | 职业启动能力、存活角色范围查询、阵营判断 |
| `CharacterClassInfo.h` | 每个职业新增 `StartUpAbilities` 数组 |
| `AuraGameplayTags.h/.cpp` | 攻击能力 Tag 和武器/左右手 Montage Tag |
| `CombatInterface.h` | `FTaggedMontage`、Socket 查询、死亡、Avatar、攻击蒙太奇接口 |
| `EnemyInterface.h` | CombatTarget Getter/Setter |
| `AuraCharacterBase.h/.cpp` | 攻击蒙太奇、三种 Socket、死亡状态与接口实现 |
| `AuraEnemy.h/.cpp` | CombatTarget、职业能力授予、HitReact 黑板空指针保护 |
| `AuraProjectileSpell.cpp` | 改用 `Execute_GetCombatSocketLocation` 并传入 Weapon Montage Tag |
| `AuraAttributeSet.cpp` | 敌人伤害玩家时，改由 Target PlayerController 显示飘字 |

### 17.3 一句话总结

**第 16 章把 AI 的“攻击意图”真正接入 GAS：行为树只用 `Abilities.Attack` 标签激活能力，角色通过 `FTaggedMontage` 提供动画与攻击部位，Motion Warping 负责朝向，Anim Notify 决定命中帧，Socket 球形查询确定目标，最后统一通过 `GE_Damage + ExecCalc_Damage` 完成权威伤害；同一套结构已经同时支持长矛武器和食尸鬼左右手攻击。**


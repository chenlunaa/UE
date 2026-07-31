# UE5 学习笔记 — 第十五次提交（第10章：GameplayAbility 与 Tag 驱动输入）

> 📦 Commit `555c4d9`：绑定了技能 Tag 的按下、持续、松开逻辑系统  
> 📦 Commit `921de1f`：补充提交  
> 📅 日期：2026-07-29  
> 🎬 对应视频：10.1~10.6（GameplayAbility 基础、技能授予、数据驱动输入）

---

## 目录

- [一、整体目标：从“拥有技能”到“输入识别”](#section-3)
- [二、GameplayAbility 基础概念（10.1）](#section-5)
- [三、创建并授予角色初始技能（10.2）](#section-9)
- [四、GameplayAbility 关键配置（10.3）](#section-17)
- [五、GameplayTag 驱动的输入配置（10.4）](#section-26)
- [六、自定义 AuraInputComponent（10.5）](#section-32)
- [七、绑定按下、持续与松开回调（10.6）](#section-39)
- [八、完整调用链与数据流](#section-45)
- [九、新增与修改文件清单](#section-50)
- [十、源码校验与注意事项](#section-54)
- [十一、知识点总结](#section-60)

---

<a id="section-3"></a>
## 一、整体目标：从“拥有技能”到“输入识别”

本章开始正式进入 **GameplayAbility（游戏玩法能力）** 系统。目标分为两步：

1. 让角色在服务器端获得一组初始技能，并验证技能能够激活和结束。
2. 建立一套数据驱动的输入层，将增强输入的 `UInputAction` 转换为统一的 `FGameplayTag`，再把按下、持续、松开三个阶段传给玩家控制器。

当前提交完成到“输入事件已经携带正确 Tag 到达 PlayerController”为止，还没有真正根据 Tag 激活 ASC 中的技能。

```text
物理按键（鼠标左键、数字键等）
    ↓ Input Mapping Context
UInputAction
    ↓ UAuraInputConfig：InputAction ↔ InputTag
UAuraInputComponent::BindAbilityActions
    ↓ Started / Triggered / Completed
AAuraPlayerController
    ├── AbilityInputTagPressed(InputTag)
    ├── AbilityInputTagHeld(InputTag)
    └── AbilityInputTagReleased(InputTag)
    ↓（下一阶段）
UAuraAbilitySystemComponent 根据 InputTag 查找并激活技能
```

### 为什么不直接把按键绑定到某个技能

如果输入层直接引用具体技能类，就会产生紧耦合：

- 更换键位时需要修改技能或控制器代码。
- 同一个技能很难动态换到另一技能槽。
- 键盘、手柄等不同输入设备难以共用技能逻辑。
- UI 技能栏与实际输入之间缺少统一标识。

加入 `InputTag` 后，输入和技能只认同一个“语义标签”。键位、输入动作和技能可以分别配置。

---

<a id="section-5"></a>
## 二、GameplayAbility 基础概念（10.1）

### 2.1 GameplayAbility 是什么

`UGameplayAbility` 用来描述角色能够执行的一个动作或技能，例如：

- 普通攻击
- 火球术
- 冲刺
- 格挡
- 使用药水

它不是直接放在场景中的 Actor，而是一种由 **Ability System Component（ASC）管理的能力类**。

一个能力通常负责：

- 判断能否激活。
- 执行技能逻辑、动画和特效。
- 创建并等待 AbilityTask。
- 应用 GameplayEffect。
- 消耗资源并进入冷却。
- 在成功、取消或失败时结束能力。

### 2.2 AbilityTask

GameplayAbility 经常需要等待异步事件，例如等待蒙太奇结束、等待目标数据或等待输入释放。GAS 使用 `UAbilityTask` 表示这些异步步骤。

```text
ActivateAbility
    ↓
创建 AbilityTask
    ↓
等待动画 / 输入 / 目标数据
    ↓
回调继续执行技能
    ↓
EndAbility
```

AbilityTask 的重要价值是能够与 GameplayAbility 的生命周期配合：能力被取消或结束时，相关任务也可以被统一清理。

### 2.3 能力必须先被授予 ASC

创建了 GameplayAbility 类，并不代表角色自动拥有它。必须先由 ASC 获得一个 `FGameplayAbilitySpec`：

```cpp
FGameplayAbilitySpec AbilitySpec(AbilityClass, 1);
AbilitySystemComponent->GiveAbility(AbilitySpec);
```

`FGameplayAbilitySpec` 是能力类进入 ASC 后的运行时描述，包含：

- 能力类
- 能力等级
- 输入状态
- 动态标签
- 激活实例与句柄等运行时信息

通常由服务器授予能力，再将 AbilitySpec 复制给拥有该角色的客户端。客户端不能自行决定永久获得一个权威技能。

---

<a id="section-9"></a>
## 三、创建并授予角色初始技能（10.2）

### 3.1 创建项目能力基类

新增 `UAuraGameplayAbility`，继承 `UGameplayAbility`：

```cpp
UCLASS()
class AURA_API UAuraGameplayAbility : public UGameplayAbility
{
    GENERATED_BODY()
};
```

当前类还是空壳，但建立项目自己的能力基类很重要。后续所有 Aura 技能都可以在这里共享：

- 输入标签
- 技能类型标签
- 伤害配置
- 公共激活和结束逻辑
- 项目统一的辅助函数

### 3.2 在角色基类保存初始技能

`AAuraCharacterBase` 增加初始技能数组：

```cpp
private:
    UPROPERTY(EditAnywhere, Category = "Attributes")
    TArray<TSubclassOf<UGameplayAbility>> StartupAbilities;
```

这里保存的是 `TSubclassOf<UGameplayAbility>`，即“能力类”，而不是已经创建的能力对象。

使用类引用的原因：

- 蓝图可以选择 `GA_FireBolt` 等能力蓝图类。
- ASC 在授予时负责创建和管理 AbilitySpec。
- 角色默认配置不持有运行时技能实例。

> 当前源码把分类写成了 `Attributes`，从语义上更适合改为 `Abilities`。

### 3.3 由 AuraASC 统一授予

`UAuraAbilitySystemComponent` 新增：

```cpp
void UAuraAbilitySystemComponent::AddCharacterAbilities(
    const TArray<TSubclassOf<UGameplayAbility>>& StartupAbilities)
{
    for (TSubclassOf<UGameplayAbility> AbilityClass : StartupAbilities)
    {
        FGameplayAbilitySpec AbilitySpec(AbilityClass, 1);
        GiveAbilityAndActivateOnce(AbilitySpec);
    }
}
```

职责被分成两层：

```text
AAuraCharacterBase
    └── 持有角色应该拥有哪些 StartupAbilities

UAuraAbilitySystemComponent
    └── 将能力类包装为 AbilitySpec 并授予 ASC
```

这样角色负责“配置”，ASC 负责“能力管理”。

### 3.4 只允许服务器授予技能

角色基类中的入口：

```cpp
void AAuraCharacterBase::AddCharacterAbilities()
{
    if (!HasAuthority()) return;

    UAuraAbilitySystemComponent* AuraASC =
        CastChecked<UAuraAbilitySystemComponent>(AbilitySystemComponent);

    AuraASC->AddCharacterAbilities(StartupAbilities);
}
```

`HasAuthority()` 保证只有服务器执行授予操作。原因是：

- 能力所有权属于权威游戏状态。
- 如果客户端也授予，会产生重复 AbilitySpec 或预测与服务器不一致。
- 服务器授予后，ASC 会把需要的信息复制给拥有客户端。

### 3.5 调用时机

玩家角色在服务器端的 `PossessedBy()` 中：

```cpp
void AAuraCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);

    InitAbilityActorInfo();
    AddCharacterAbilities();
}
```

必须先执行 `InitAbilityActorInfo()`，让 ASC 知道 OwnerActor 和 AvatarActor，随后再授予或激活能力。

```text
服务器 PossessedBy
    ↓
InitAbilityActorInfo
    ↓ ASC 建立 Owner / Avatar 上下文
AddCharacterAbilities
    ↓
遍历 StartupAbilities
    ↓
构造 FGameplayAbilitySpec
    ↓
授予（并按当前实现立即激活）
```

### 3.6 GiveAbility 与 GiveAbilityAndActivateOnce

| API | 行为 | 适用场景 |
|---|---|---|
| `GiveAbility` | 授予后保留在 ASC，等待以后激活 | 普通攻击、主动技能、常驻技能 |
| `GiveAbilityAndActivateOnce` | 授予后立即激活一次，结束后移除 | 一次性能力、临时触发逻辑 |

本次源码使用 `GiveAbilityAndActivateOnce` 来立即验证测试技能，因此测试蓝图会在角色初始化时触发 `ActivateAbility`。

如果目标是让 `StartupAbilities` 成为角色后续可通过输入反复使用的技能，应改用 `GiveAbility`。否则能力结束后会从 ASC 中移除，后面的输入系统将找不到它。

### 3.7 能力必须显式结束

GameplayAbility 激活后不会自动结束。蓝图测试流程为：

```text
Event ActivateAbility
    ↓ Print：测试技能激活
Delay 5 秒
    ↓
End Ability
    ↓ Event OnEndAbility / 后续结束逻辑
```

如果遗漏 `EndAbility`：

- 能力会一直处于 Active 状态。
- 冷却、阻挡标签或互斥能力可能无法正确恢复。
- AbilityTask 和其他资源可能无法清理。

---

<a id="section-17"></a>
## 四、GameplayAbility 关键配置（10.3）

### 4.1 Ability Tags：能力自身标签

用于描述“这是什么能力”，例如：

```text
Abilities.Fire.FireBolt
Abilities.Attack.Melee
Abilities.Status.Passive
```

其他能力、效果或 ASC 可以通过这些标签识别和筛选能力。

### 4.2 Cancel Abilities with Tag

当前能力激活时，取消带有指定标签的其他能力。

示例：翻滚激活时取消蓄力攻击。

```text
GA_Roll 激活
    └── CancelAbilitiesWithTag = Abilities.Attack.Charging
```

### 4.3 Block Abilities with Tag

当前能力激活期间，阻止带有指定标签的其他能力激活。

示例：眩晕状态能力运行时，阻止所有主动技能。

### 4.4 Activation Owned Tags

能力激活期间临时赋予拥有者的标签。能力结束后由系统移除。

```text
GA_Block 激活
    ↓
角色获得 State.Blocking
    ↓
其他系统检测该标签
    ↓
GA_Block 结束，标签移除
```

### 4.5 激活条件标签

| 配置 | 含义 |
|---|---|
| Activation Required Tags | ASC 必须拥有全部标签，能力才能激活 |
| Activation Blocked Tags | ASC 拥有任一标签时，能力不能激活 |
| Source Required Tags | 技能源对象必须拥有全部标签 |
| Source Blocked Tags | 技能源对象拥有任一标签时阻止激活 |
| Target Required Tags | 目标必须拥有全部标签 |
| Target Blocked Tags | 目标拥有任一标签时阻止激活 |

这些配置把大量条件判断从硬编码 `if` 转换成声明式标签规则。

### 4.6 Instancing Policy

| 策略 | 含义 | 特点 |
|---|---|---|
| Non-Instanced | 不为每次激活创建能力实例 | 性能好，但不能安全保存每次激活状态，功能限制多 |
| Instanced Per Actor | 每个拥有者共享一个能力实例 | 常用；可以保存成员状态，但重复激活前需重置状态 |
| Instanced Per Execution | 每次激活创建新实例 | 状态完全独立，开销更高 |

需要使用 AbilityTask、绑定委托或保存运行时状态时，通常选择实例化策略。

### 4.7 Net Execution Policy

| 策略 | 执行位置 |
|---|---|
| Local Predicted | 客户端先预测执行，服务器校验 |
| Local Only | 只在本地执行 |
| Server Initiated | 服务器发起，并同步到客户端 |
| Server Only | 只在服务器执行 |

动作 RPG 常用 `Local Predicted` 降低按键后的体感延迟，但伤害、资源消耗等权威结果仍由服务器确认。

### 4.8 Cost 与 Cooldown

- **Cost GameplayEffect**：激活时消耗法力、耐力等资源。
- **Cooldown GameplayEffect**：激活后添加冷却标签并维持一段时间。

通常先通过 `CanActivateAbility` 检查条件，再在技能中提交：

```text
输入请求激活
    ↓
检查标签、Cost、Cooldown
    ↓
CommitAbility
    ├── 应用 Cost
    └── 应用 Cooldown
    ↓
执行技能主体
```

---

<a id="section-26"></a>
## 五、GameplayTag 驱动的输入配置（10.4）

### 5.1 输入映射的三层结构

本章输入系统包含三个相互独立的层级：

| 层级 | 示例 | 负责内容 |
|---|---|---|
| 物理输入 | 鼠标左键、数字键 1 | 玩家实际按下的设备按键 |
| InputAction | `IA_LMB`、`IA_1` | Enhanced Input 的逻辑动作 |
| InputTag | `InputTag.LMB`、`InputTag.Num1` | GAS 与技能栏使用的语义标识 |

`Input Mapping Context` 负责“物理键 → InputAction”，`UAuraInputConfig` 负责“InputAction → InputTag”。

### 5.2 FAuraInputAction

```cpp
USTRUCT(BlueprintType)
struct FAuraInputAction
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly)
    const UInputAction* InputAction = nullptr;

    UPROPERTY(EditDefaultsOnly)
    FGameplayTag InputTag = FGameplayTag();
};
```

一个结构体元素就是一条映射关系：

```text
IA_LMB  ↔ InputTag.LMB
IA_RMB  ↔ InputTag.RMB
IA_1    ↔ InputTag.Num1
```

### 5.3 UAuraInputConfig 数据资产

```cpp
UCLASS()
class AURA_API UAuraInputConfig : public UDataAsset
{
    GENERATED_BODY()

public:
    const UInputAction* FindAbilityInputActionForTag(
        const FGameplayTag& InputTag,
        bool bLogNotFound = false) const;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TArray<FAuraInputAction> AbilityInputActions;
};
```

使用 DataAsset 的好处：

- 输入映射可在编辑器中配置。
- 更换键位或技能槽不需要重新编译 C++。
- 可以为不同角色、设备或游戏模式创建不同配置。
- InputAction 与 GAS 技能系统之间不直接依赖。

### 5.4 根据 Tag 反查 InputAction

```cpp
const UInputAction* UAuraInputConfig::FindAbilityInputActionForTag(
    const FGameplayTag& InputTag,
    bool bLogNotFound) const
{
    for (const FAuraInputAction& Action : AbilityInputActions)
    {
        if (Action.InputAction && Action.InputTag == InputTag)
        {
            return Action.InputAction;
        }
    }

    if (bLogNotFound)
    {
        UE_LOG(LogTemp, Error,
            TEXT("Can't find AbilityInputAction for InputTag [%s], on InputConfig [%s]"),
            *InputTag.ToString(), *GetNameSafe(this));
    }

    return nullptr;
}
```

查询前先检查 `Action.InputAction`，避免返回空输入动作。找不到时是否记录日志由调用方决定。

### 5.5 原生输入标签

`FAuraGameplayTags` 新增：

```cpp
FGameplayTag InputTag_LMB;
FGameplayTag InputTag_RMB;
FGameplayTag InputTag_Num1;
FGameplayTag InputTag_Num2;
FGameplayTag InputTag_Num3;
FGameplayTag InputTag_Num4;
```

预期注册出的标签：

| C++ 成员 | GameplayTag |
|---|---|
| `InputTag_LMB` | `InputTag.LMB` |
| `InputTag_RMB` | `InputTag.RMB` |
| `InputTag_Num1` | `InputTag.Num1` |
| `InputTag_Num2` | `InputTag.Num2` |
| `InputTag_Num3` | `InputTag.Num3` |
| `InputTag_Num4` | `InputTag.Num4` |

---

<a id="section-32"></a>
## 六、自定义 AuraInputComponent（10.5）

### 6.1 为什么继承 UEnhancedInputComponent

原本 `AAuraPlayerController::SetupInputComponent()` 直接逐个调用 `BindAction`。技能输入增加后，如果仍在控制器中逐项绑定，会出现大量重复代码。

因此创建：

```cpp
UCLASS()
class AURA_API UAuraInputComponent : public UEnhancedInputComponent
{
    GENERATED_BODY()

public:
    template<class UserClass,
             typename PressedFuncType,
             typename ReleasedFuncType,
             typename HeldFuncType>
    void BindAbilityActions(
        const UAuraInputConfig* InputConfig,
        UserClass* Object,
        PressedFuncType PressedFunc,
        ReleasedFuncType ReleasedFunc,
        HeldFuncType HeldFunc);
};
```

输入组件负责“如何批量绑定”，PlayerController 只负责“收到输入后做什么”。

### 6.2 为什么 BindAbilityActions 是模板函数

`BindAction` 需要接收回调所属对象及成员函数指针。不同调用者的类型和函数指针类型可能不同，所以使用模板参数：

- `UserClass`：回调函数所属对象类型。
- `PressedFuncType`：按下回调类型。
- `ReleasedFuncType`：松开回调类型。
- `HeldFuncType`：持续回调类型。

这样输入组件不必依赖 `AAuraPlayerController`，以后其他类型也能复用该绑定函数。

### 6.3 模板实现必须放在头文件

C++ 模板在使用点实例化。编译调用代码时必须能看到完整实现，因此 `BindAbilityActions` 的实现写在 `.h` 中，而不是 `.cpp` 中。

如果只在 `.cpp` 中定义，其他翻译单元使用模板时通常会出现链接错误，除非显式实例化所有需要的类型组合。

### 6.4 遍历并绑定三种输入状态

```cpp
for (const FAuraInputAction& Action : InputConfig->AbilityInputActions)
{
    if (Action.InputAction && Action.InputTag.IsValid())
    {
        if (PressedFunc)
        {
            BindAction(Action.InputAction, ETriggerEvent::Started,
                Object, PressedFunc, Action.InputTag);
        }

        if (ReleasedFunc)
        {
            BindAction(Action.InputAction, ETriggerEvent::Completed,
                Object, ReleasedFunc, Action.InputTag);
        }

        if (HeldFunc)
        {
            BindAction(Action.InputAction, ETriggerEvent::Triggered,
                Object, HeldFunc, Action.InputTag);
        }
    }
}
```

### 6.5 ETriggerEvent 对应关系

| TriggerEvent | 本章语义 | 触发特点 |
|---|---|---|
| `Started` | Pressed | 输入刚开始时触发一次 |
| `Triggered` | Held | 满足触发条件期间持续触发，通常每帧调用 |
| `Completed` | Released | 已触发的输入结束时调用 |

要注意 `Triggered` 不等于只触发一次的“长按成功”。它的实际行为还受 InputAction 中 Trigger 配置影响。

### 6.6 额外参数 Action.InputTag

```cpp
BindAction(Action.InputAction, ETriggerEvent::Started,
    Object, PressedFunc, Action.InputTag);
```

最后的 `Action.InputTag` 会作为额外参数传给回调：

```cpp
void AbilityInputTagPressed(FGameplayTag InputTag);
```

因此所有技能输入都能绑定到同一组函数，再通过传入的 Tag 区分具体输入，无需为 `IA_1`、`IA_2`、鼠标左键分别创建回调。

---

<a id="section-39"></a>
## 七、绑定按下、持续与松开回调（10.6）

### 7.1 PlayerController 保存 InputConfig

```cpp
UPROPERTY(EditDefaultsOnly, Category="Input")
TObjectPtr<UAuraInputConfig> InputConfig;
```

在 PlayerController 蓝图默认值中，需要把创建好的 `UAuraInputConfig` DataAsset 赋给该属性。否则 `BindAbilityActions` 中的 `check(InputConfig)` 会直接中止程序。

### 7.2 三个输入回调

```cpp
void AbilityInputTagPressed(FGameplayTag InputTag);
void AbilityInputTagReleased(FGameplayTag InputTag);
void AbilityInputTagHeld(FGameplayTag InputTag);
```

当前实现使用屏幕调试信息验证绑定：

```cpp
void AAuraPlayerController::AbilityInputTagPressed(FGameplayTag InputTag)
{
    GEngine->AddOnScreenDebugMessage(
        1, 3.f, FColor::Red, *InputTag.ToString());
}
```

另外两个函数分别用蓝色和绿色打印，确认：

- 按下时收到了 `Started`。
- 按住期间收到了 `Triggered`。
- 松开时收到了 `Completed`。
- 回调参数是 DataAsset 中与 InputAction 配对的正确标签。

### 7.3 SetupInputComponent

```cpp
void AAuraPlayerController::SetupInputComponent()
{
    Super::SetupInputComponent();

    UAuraInputComponent* AuraInputComponent =
        CastChecked<UAuraInputComponent>(InputComponent);

    AuraInputComponent->BindAction(
        MoveAction, ETriggerEvent::Triggered,
        this, &AAuraPlayerController::Move);

    AuraInputComponent->BindAbilityActions(
        InputConfig,
        this,
        &ThisClass::AbilityInputTagPressed,
        &ThisClass::AbilityInputTagReleased,
        &ThisClass::AbilityInputTagHeld);
}
```

`CastChecked` 表示项目明确要求默认 InputComponent 必须是 `UAuraInputComponent`。如果配置不正确，应尽早崩溃并暴露问题，而不是静默跳过技能输入。

### 7.4 设置默认输入组件类

`DefaultInput.ini` 修改为：

```ini
DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
DefaultInputComponentClass=/Script/Aura.AuraInputComponent
```

这一步让 PlayerController 创建出的 `InputComponent` 实际类型变成 `UAuraInputComponent`，否则 `CastChecked` 会失败。

### 7.5 编辑器资产配置

除了 C++ 代码，还需要完成以下蓝图/资产配置：

1. 创建技能用的一维 `InputAction`，例如 `IA_LMB`、`IA_1`。
2. 在 `Input Mapping Context` 中把物理按键映射到 InputAction。
3. 创建 `UAuraInputConfig` DataAsset。
4. 在 `AbilityInputActions` 数组中配置 InputAction 与 InputTag。
5. 在 PlayerController 蓝图默认值中指定该 InputConfig。
6. 在项目输入设置中确认默认组件为 `AuraInputComponent`。

任意一层遗漏，都会造成输入链路中断。

---

<a id="section-45"></a>
## 八、完整调用链与数据流

### 8.1 角色技能授予链

```text
服务器占有 AAuraCharacter
    ↓ PossessedBy
AAuraCharacter::InitAbilityActorInfo
    ↓
ASC 建立 OwnerActor / AvatarActor
    ↓ AAuraCharacterBase::AddCharacterAbilities
HasAuthority 检查
    ↓
UAuraAbilitySystemComponent::AddCharacterAbilities
    ↓ 遍历 StartupAbilities
FGameplayAbilitySpec(AbilityClass, Level = 1)
    ↓
GiveAbilityAndActivateOnce
```

### 8.2 输入资产链

```text
物理键 1
    ↓ Input Mapping Context
IA_1
    ↓ UAuraInputConfig
InputTag.Num1
```

### 8.3 运行时绑定链

```text
AAuraPlayerController::SetupInputComponent
    ↓ CastChecked
UAuraInputComponent
    ↓ BindAbilityActions(InputConfig, Controller, 三个函数)
遍历 AbilityInputActions
    ↓
IA_1 Started   → AbilityInputTagPressed(InputTag.Num1)
IA_1 Triggered → AbilityInputTagHeld(InputTag.Num1)
IA_1 Completed → AbilityInputTagReleased(InputTag.Num1)
```

### 8.4 本章完成边界

当前 PlayerController 只打印 Tag，还没有把事件传给 ASC。下一阶段通常会实现：

```cpp
AuraASC->AbilityInputTagPressed(InputTag);
AuraASC->AbilityInputTagHeld(InputTag);
AuraASC->AbilityInputTagReleased(InputTag);
```

ASC 再遍历可激活的 `FGameplayAbilitySpec`，匹配能力的 InputTag，调用 `TryActivateAbility` 或更新 Spec 的输入状态。

---

<a id="section-50"></a>
## 九、新增与修改文件清单

### 9.1 新增文件

| 文件 | 作用 |
|---|---|
| `AuraGameplayAbility.h/.cpp` | 项目 GameplayAbility 基类 |
| `AuraInputConfig.h/.cpp` | InputAction 与 InputTag 的数据资产映射 |
| `AuraInputComponent.h/.cpp` | 批量绑定技能输入的增强输入组件 |

### 9.2 主要修改文件

| 文件 | 修改内容 |
|---|---|
| `AuraAbilitySystemComponent.h/.cpp` | 新增初始技能授予函数 |
| `AuraGameplayTags.h/.cpp` | 声明并注册六个 InputTag |
| `AuraCharacterBase.h/.cpp` | 保存 StartupAbilities，并在服务器调用 AuraASC 授予 |
| `AuraCharacter.cpp` | ASC 初始化后添加角色技能 |
| `AuraPlayerController.h/.cpp` | 保存 InputConfig，绑定三态输入回调 |
| `DefaultInput.ini` | 默认 InputComponent 改为 `AuraInputComponent` |

### 9.3 提交规模

- `555c4d9`：修改 12 个文件，新增 122 行，删除 8 行。
- `921de1f`：补充 6 个新文件，新增 137 行。

第二次“补充提交”补齐了第一次提交中已经被引用、但尚未纳入 Git 的 GameplayAbility、InputConfig 和 InputComponent 文件。

---

<a id="section-54"></a>
## 十、源码校验与注意事项

### 10.1 InputTag 注册存在复制粘贴错误

当前 `AuraGameplayTags.cpp` 中六次注册都赋值给了 `InputTag_LMB`：

```cpp
GameplayTags.InputTag_LMB = AddNativeGameplayTag("InputTag.LMB");
GameplayTags.InputTag_LMB = AddNativeGameplayTag("InputTag.RMB");
GameplayTags.InputTag_LMB = AddNativeGameplayTag("InputTag.Num1");
// ...
```

这会导致：

- `InputTag_LMB` 最终只保存最后一次注册的 `InputTag.Num4`。
- `InputTag_RMB`、`InputTag_Num1`～`Num4` 成员仍然是无效标签。
- 标签本身可能已经注册进 GameplayTagsManager，但 C++ 成员映射全部错误。

正确写法应分别赋值：

```cpp
GameplayTags.InputTag_LMB  = AddNativeGameplayTag("InputTag.LMB");
GameplayTags.InputTag_RMB  = AddNativeGameplayTag("InputTag.RMB");
GameplayTags.InputTag_Num1 = AddNativeGameplayTag("InputTag.Num1");
GameplayTags.InputTag_Num2 = AddNativeGameplayTag("InputTag.Num2");
GameplayTags.InputTag_Num3 = AddNativeGameplayTag("InputTag.Num3");
GameplayTags.InputTag_Num4 = AddNativeGameplayTag("InputTag.Num4");
```

### 10.2 初始技能当前会立即激活并移除

源码选择了：

```cpp
GiveAbilityAndActivateOnce(AbilitySpec);
```

这适合本章测试“技能激活/结束”，但与后续通过输入反复激活技能的目标不同。正式的初始主动技能通常应使用：

```cpp
GiveAbility(AbilitySpec);
```

### 10.3 StartupAbilities 的编辑器分类

当前属性为：

```cpp
UPROPERTY(EditAnywhere, Category = "Attributes")
TArray<TSubclassOf<UGameplayAbility>> StartupAbilities;
```

建议分类改为 `Abilities`，避免在角色蓝图中与属性初始化 GE 混在一起。

### 10.4 Held 回调频率

`ETriggerEvent::Triggered` 可能每帧触发。未来在 `AbilityInputTagHeld` 中查找并激活能力时，需要避免不必要的重复遍历或重复激活。

常见策略是：

- 未激活时尝试 `TryActivateAbility`。
- 已激活时只更新 `InputPressed` 状态或发送复制事件。
- 需要持续施法的能力自行监听输入保持与释放。

### 10.5 服务器与客户端职责

- 技能授予：服务器权威执行。
- 本地输入：拥有客户端最先收到。
- 技能激活：按能力的 Net Execution Policy 预测或请求服务器执行。
- 最终伤害、资源与状态：服务器确认。

不要因为输入发生在客户端，就让客户端绕过 ASC 的网络预测与服务器校验直接修改权威状态。

---

<a id="section-60"></a>
## 十一、知识点总结

### 核心类职责

| 类 | 职责 |
|---|---|
| `UAuraGameplayAbility` | Aura 项目的能力基类 |
| `UAuraAbilitySystemComponent` | 授予和管理 GameplayAbility |
| `AAuraCharacterBase` | 配置角色初始技能列表 |
| `UAuraInputConfig` | 保存 InputAction ↔ InputTag 映射 |
| `UAuraInputComponent` | 批量绑定输入动作与三态回调 |
| `AAuraPlayerController` | 接收玩家输入 Tag，并决定下一步行为 |

### 核心设计模式

1. **服务器权威授予**：`HasAuthority()` 保证能力所有权一致。
2. **类配置与实例管理分离**：Character 保存能力类，ASC 管理 AbilitySpec 和实例。
3. **数据驱动输入**：DataAsset 连接 InputAction 与 GameplayTag。
4. **语义解耦**：输入层和技能层通过 Tag 通信，不直接引用彼此。
5. **模板复用**：自定义 InputComponent 接受不同对象及成员函数类型。
6. **三态输入**：Started、Triggered、Completed 分别表达按下、持续和松开。

### 一句话总结

> 本章先让服务器把 GameplayAbility 授予角色，再用 `UAuraInputConfig` 将 Enhanced Input 转换成 `InputTag`，最终通过 `UAuraInputComponent` 把同一套按下、持续、松开回调批量绑定到 PlayerController，为下一步由 ASC 根据 Tag 激活技能建立完整输入通道。

# 第 21 章：Overlay 技能栏——ASC 能力信息通信与多人同步

> 笔记类型：项目章节记录｜第 21 章

> 本章目标：重构主 HUD，加入技能球与经验条，并打通 `AbilitySystemComponent → OverlayWidgetController → SpellGlobe` 的能力信息通信。最终效果是：技能图标不再硬编码到某个槽位，而是根据 `AbilitySpec` 中的 `InputTag` 自动显示；多人客户端也能在能力复制完成后正确初始化 HUD。

---

## 目录

- [1. 本章完成了什么](#section-1)
- [2. 新 Overlay 的组件化结构](#section-2)
- [5. AbilityInfo：静态 UI 数据资产](#section-5)
- [7. OverlayWidgetController：ASC 与 UMG 的中间层](#section-7)
- [9. 多人客户端为什么没有技能图标](#section-9)

<a id="section-1"></a>
## 1. 本章完成了什么

本章的工作可以分成三层：

1. **表现层**：制作 `WBP_HealthManaSpells`、`WBP_SpellGlobe` 与 `WBP_XPBar`，重新组织 Overlay。
2. **数据层**：创建 `UAbilityInfo` DataAsset，用 `AbilityTag` 保存图标和背景材质等静态展示数据。
3. **通信层**：ASC 遍历运行时 `FGameplayAbilitySpec`，WidgetController 合并静态与动态信息并广播，技能球按自己的 `InputTag` 接收对应数据。

完整思路如下：

```text
GameplayAbility 类 / CDO
    └─ AbilityTags：这是什么技能

FGameplayAbilitySpec（ASC 中的运行时实例数据）
    └─ DynamicSpecSourceTags：这个技能当前绑在哪个输入上

DA_AbilityInfo（静态 UI 配置）
    └─ AbilityTag → Icon + BackgroundMaterial

OverlayWidgetController
    └─ 用 AbilityTag 查 DataAsset，再补入 InputTag，广播 FAuraAbilityInfo

WBP_SpellGlobe
    └─ 只接收与自身 InputTag 精确匹配的信息并刷新图片
```

这套结构的关键价值是**能力身份、输入绑定和 UI 资源彼此解耦**。

---

<a id="section-2"></a>
## 2. 新 Overlay 的组件化结构

### 2.1 `WBP_HealthManaSpells`

主 HUD 不再直接把所有图片、进度条和技能槽堆进 `WBP_Overlay`，而是创建组合组件 `WBP_HealthManaSpells`：

```text
WBP_HealthManaSpells
└─ Base Horizontal Box
   ├─ Health Box（Fill 权重约 0.2）
   │  └─ WBP_HealthGlobe
   ├─ Center Box（Fill 权重约 0.6）
   │  ├─ Offensive Box
   │  │  ├─ 标题文本
   │  │  ├─ 输入提示：LMB、RMB、1、2、3、4
   │  │  └─ 6 个 WBP_SpellGlobe
   │  └─ Passive Box
   │     ├─ 被动技能标题
   │     └─ 2 个被动技能球
   └─ Mana Box（Fill 权重约 0.2）
      └─ WBP_ManaGlobe
```

Horizontal Box / Vertical Box 中的 **Fill 不是固定像素宽度**，而是对父容器剩余空间的权重分配。例如三段权重为 `0.2 / 0.6 / 0.2` 时，中间区域大约占剩余空间的 60%。因此窗口尺寸变化时，它比硬编码 Position 更稳定。

组件化的好处：

- `WBP_Overlay` 只负责摆放顶层模块；
- `WBP_HealthManaSpells` 负责内部布局与向子控件传递 WidgetController；
- Health、Mana、SpellGlobe 各自负责自己的显示和委托绑定；
- 后续修改技能栏，不必碰血量球内部逻辑。

### 2.2 将组合组件装入 `WBP_Overlay`

蓝图操作要点：

1. 删除 Overlay 中旧的独立 HealthGlobe 和 ManaGlobe。
2. 添加 `WBP_HealthManaSpells`，锚点设为底部中央。
3. Alignment X 设为 `0.5`，Position X 设为 `0`，使组件以自身中心对齐屏幕中央。
4. 勾选 **Is Variable**，以便事件图访问。
5. 在 Overlay 的 `WidgetControllerSet` 中，对它调用 `SetWidgetController`。
6. 在 `WBP_HealthManaSpells` 的 `WidgetControllerSet` 中继续把同一个控制器传给 HealthGlobe、ManaGlobe 和所有 SpellGlobe。

控制器传递形成一棵树：

```text
WBP_Overlay
└─ SetWidgetController(OverlayWC)
   └─ WBP_HealthManaSpells
      ├─ HealthGlobe.SetWidgetController(OverlayWC)
      ├─ ManaGlobe.SetWidgetController(OverlayWC)
      └─ 每个 SpellGlobe.SetWidgetController(OverlayWC)
```

这里并不是为每个子 Widget 创建一个 Controller。它们共享同一个 `OverlayWidgetController`，但分别订阅自己关心的委托。

---

<a id="section-3"></a>
## 3. `WBP_SpellGlobe` 的蓝图设计

### 3.1 组件层级

```text
SizeBox Root
└─ Overlay Root
   ├─ Background Image（材质背景）
   ├─ Spell Icon Image（技能图标）
   ├─ Ring Image（外圈）
   ├─ Glass Image（玻璃高光）
   └─ Text_Cooldown（冷却秒数）
```

建议暴露的设计变量：

- `BoxWidth`、`BoxHeight`
- `RingBrush`
- `SpellIconBrush`
- `GlassPadding`
- 背景 Brush 或 Material
- `InputTag`

在 `PreConstruct` 中调用 `UpdateBoxSize`、`UpdateRingBrush`、`UpdateGlassPadding`、`UpdateSpellIconBrush`，能够在设计器阶段实时预览样式。

### 3.2 PreConstruct 和 Construct 的区别

- `PreConstruct`：编辑器预览时也可能执行，适合尺寸、Brush、Padding 等纯表现设置。
- `Construct`：Widget 真正构造时执行，可能因 Widget 被重新加入界面而多次调用。
- `WidgetControllerSet`：项目自定义事件，控制器真正设置完成后触发，适合绑定 Controller 委托。

不要在 `PreConstruct` 中假定 ASC、PlayerState 或 WidgetController 一定存在，因为设计器预览阶段通常没有完整游戏环境。

### 3.3 空技能槽

空槽不能简单把 Brush 重置成默认值，否则 Image 常显示为白色方块。课程采用透明 Brush：

```text
ClearGlobe
├─ SpellIcon.SetBrush(TransparentBrush)
└─ Background.SetBrush(TransparentBrush)
```

外圈和玻璃仍保留，因此玩家能看出这是一个可装备槽位。

另一个函数负责显示已装备技能：

```text
SetIconAndBackground(IconBrush, BackgroundBrush)
├─ SpellIcon.SetBrush(IconBrush)
└─ Background.SetBrush(BackgroundBrush)
```

### 3.4 冷却显示

本章先准备冷却 UI，而不是完成完整冷却系统：

- `Text_Cooldown` 显示剩余秒数；
- `SetBackgroundTint(float Tint)` 把同一个 `Tint` 写入 RGB，Alpha 保持 1；
- 冷却期间可降低背景亮度并显示数字；
- 冷却结束恢复 Tint 并隐藏文本。

Slate Brush 是 UMG 对“如何绘制图片”的描述，它可以引用 Texture，也可以引用 UI Material。图标使用纹理，带效果的底图可使用材质实例。

---

<a id="section-4"></a>
## 4. 新增经验条 `WBP_XPBar`

经验条由 Overlay、Frame Image 和 ProgressBar 组成：

```text
WBP_XPBar（约 880 × 50）
└─ Overlay
   ├─ XP Frame
   └─ ProgressBar
      ├─ 半透明深色背景
      └─ XP 填充图
```

将它作为独立 Widget 放入 `WBP_Overlay`，而不是塞进 HealthManaSpells，便于单独调整层级、缩放和位置。要注意 Widget 的绘制顺序：经验条在技能球前面或后面会产生不同遮挡效果；测试时还要把 Health/Mana 百分比拉到 0，确认底层没有意外露出的图形。

本节同时发现两类关卡问题：

- 投射物穿过地面块：为相应网格设置 Projectile 通道的 Block/Overlap，并按需求启用 Generate Overlap Events；
- 隔墙仍能鼠标高亮敌人：当前光标 Trace 没有把遮挡墙作为终止条件，需要后续统一碰撞通道与 Trace 策略。

---

<a id="section-5"></a>
## 5. AbilityInfo：静态 UI 数据资产

### 5.1 数据结构

```cpp
USTRUCT(BlueprintType)
struct FAuraAbilityInfo
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FGameplayTag AbilityTag;

    UPROPERTY(BlueprintReadOnly)
    FGameplayTag InputTag;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<const UTexture2D> Icon = nullptr;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<const UMaterialInterface> BackgroundMaterial = nullptr;
};
```

`UAbilityInfo : UDataAsset` 保存 `TArray<FAuraAbilityInfo>`，并提供：

```cpp
FAuraAbilityInfo FindAbilityInfoForTag(
    const FGameplayTag& AbilityTag,
    bool bLogNotFound = true) const;
```

它遍历数组，按 `AbilityTag` 精确查找；未找到时用自定义 `LogAura` 输出错误并返回空结构体。

### 5.2 为什么 InputTag 不在 DataAsset 中编辑

字段虽然属于最终广播的 `FAuraAbilityInfo`，但它没有 `EditDefaultsOnly`：

- Icon、BackgroundMaterial 是能力长期稳定的美术配置，适合存 DataAsset；
- InputTag 是运行时绑定，可能由初始配置、换键、技能装备系统改变；
- 因此 WidgetController 查到静态 Info 后，再从 `AbilitySpec` 写入 `Info.InputTag`。

DataAsset 负责“这个技能长什么样”，AbilitySpec 负责“它现在处于什么运行状态”。

### 5.3 `AbilityTag` 与 `InputTag` 不可混用

| 标签 | 回答的问题 | 数据来源 | 示例 |
|---|---|---|---|
| AbilityTag | 这是什么能力？ | Ability CDO 的 `AbilityTags` | `Abilities.Fire.FireBolt` |
| InputTag | 这个 Spec 现在绑在哪个输入？ | `AbilitySpec` 动态标签 | `InputTag.LMB` |

一个能力换到其他按键后，AbilityTag 不变，InputTag 改变。一个输入槽换上其他能力时，InputTag 可不变，AbilityTag 改变。

---

<a id="section-6"></a>
## 6. ASC 中的运行时能力信息

### 6.1 授予启动能力

```cpp
FGameplayAbilitySpec AbilitySpec(AbilityClass, 1);
AbilitySpec.GetDynamicSpecSourceTags().AddTag(
    AuraAbility->StartupInputTag);
GiveAbility(AbilitySpec);
```

`GiveAbility` 将 Spec 加入 ASC 的 `ActivatableAbilities`。此时能力只是被授予，并不等于已经激活。

所有启动能力处理完后：

```cpp
bStartupAbilities = true;
AbilitiesGivenDelegate.Broadcast(this);
```

布尔值表示“初始化事实已经发生”，委托表示“刚刚发生了这个事件”。两者配合解决订阅先后顺序不确定的问题。

### 6.2 为什么既要 bool 又要 Delegate

存在两种时序：

```text
情况 A：WidgetController 先绑定 → ASC 后授予 → 委托通知成功
情况 B：ASC 先授予并广播 → WidgetController 后绑定 → 已错过广播
```

因此绑定时先检查状态：

```cpp
if (AuraASC->bStartupAbilities)
{
    OnInitializeStartupAbilities(AuraASC);
}
else
{
    AuraASC->AbilitiesGivenDelegate.AddUObject(
        this,
        &UOverlayWidgetController::OnInitializeStartupAbilities);
}
```

这是一种常见的 **“状态 + 事件”** 初始化模式：状态处理迟到的观察者，事件处理提前就位的观察者。

### 6.3 `ForEachAbility` 与 Scope Lock

ASC 将遍历能力列表的细节封装起来：

```cpp
void UAuraAbilitySystemComponent::ForEachAbility(
    const FForEachAbility& Delegate)
{
    FScopedAbilityListLock ActiveScopedLock(*this);

    for (const FGameplayAbilitySpec& Spec : GetActivatableAbilities())
    {
        Delegate.ExecuteIfBound(Spec);
    }
}
```

这样 WidgetController 不需要直接操作 ASC 内部容器。`FScopedAbilityListLock` 在作用域内保护能力列表：如果回调间接导致授予/移除能力，相关修改会被安全延后，避免遍历时容器改变而使引用或迭代器失效。

### 6.4 从 Spec 提取 AbilityTag

```cpp
for (const FGameplayTag& Tag : AbilitySpec.Ability.Get()->AbilityTags)
{
    if (Tag.MatchesTag(
        FGameplayTag::RequestGameplayTag(FName("Abilities"))))
    {
        return Tag;
    }
}
```

`AbilitySpec.Ability` 指向能力对象，通常可视作 Ability 类的 CDO，用于读取类级别配置。`MatchesTag(Abilities)` 表示只接受 `Abilities` 层级下的标签。

### 6.5 从 Spec 提取 InputTag

```cpp
for (const FGameplayTag& Tag : AbilitySpec.GetDynamicSpecSourceTags())
{
    if (Tag.MatchesTag(
        FGameplayTag::RequestGameplayTag(FName("InputTag"))))
    {
        return Tag;
    }
}
```

InputTag 存在动态 Spec 标签中，因此将来可在运行时移除旧标签、加入新标签，实现技能换槽或换键。两个 Helper 都可声明为 `static`，因为结果只取决于传入的 Spec，不需要访问某个 ASC 实例的成员状态。

> `MatchesTag` 用来判断父子标签关系，并不代表这里需要 `FGameplayTagContainer`。代码是在多个 Tag 中寻找属于指定根节点的单个 Tag；只有需要把一组标签作为整体传递、求交集或批量查询时，Container 才更合适。

---

<a id="section-7"></a>
## 7. OverlayWidgetController：ASC 与 UMG 的中间层

### 7.1 为什么不让 Widget 直接读 ASC

Widget 直接 Cast Player、PlayerState、ASC 虽然能暂时工作，但会产生大量依赖：

- 每个 Widget 都要理解 GAS；
- 测试和复用困难；
- 多人时机判断散落在蓝图各处；
- UI 既负责显示又负责查询业务数据。

WidgetController 负责把 GAS 数据转换成 UI 需要的结构，再通过 BlueprintAssignable 委托广播。Widget 只需监听和显示。

### 7.2 广播结构体

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(
    FAbilityInfoSignature,
    const FAuraAbilityInfo&,
    Info);

UPROPERTY(BlueprintAssignable, Category="GAS|Messages")
FAbilityInfoSignature AbilityInfoDelegate;
```

使用动态多播委托，是因为多个 SpellGlobe 都要在蓝图中绑定同一个广播。

### 7.3 合并并广播数据

```cpp
void UOverlayWidgetController::OnInitializeStartupAbilities(
    UAuraAbilitySystemComponent* AuraASC)
{
    if (!AuraASC->bStartupAbilities) return;

    FForEachAbility BroadcastDelegate;
    BroadcastDelegate.BindLambda(
        [this, AuraASC](const FGameplayAbilitySpec& Spec)
        {
            FAuraAbilityInfo Info =
                AbilityInfo->FindAbilityInfoForTag(
                    AuraASC->GetAbilityTagFromSpec(Spec));

            Info.InputTag = AuraASC->GetInputTagFromSpec(Spec);
            AbilityInfoDelegate.Broadcast(Info);
        });

    AuraASC->ForEachAbility(BroadcastDelegate);
}
```

注意它不是把 AbilitySpec 直接暴露给蓝图，而是只发送 UI 真正需要的轻量信息。这降低了 UI 对 GAS 内部类型的耦合。

---

<a id="section-8"></a>
## 8. SpellGlobe 如何接收信息

每个 `WBP_SpellGlobe` 预先拥有自己的 `InputTag`。在 `WidgetControllerSet` 中：

```text
WidgetControllerSet
└─ Cast 为 BP_OverlayWidgetController
   ├─ 保存 Controller 引用
   └─ Assign AbilityInfoDelegate
      └─ 收到 FAuraAbilityInfo
         ├─ Info.InputTag == Self.InputTag（精确匹配）？
         ├─ Icon → Make SlateBrush → SetIcon
         └─ BackgroundMaterial → Make SlateBrush → SetBackground
```

父组件为各个主动槽设置：

```text
LMB Globe → InputTag.LMB
RMB Globe → InputTag.RMB
1 Globe   → InputTag.1
2 Globe   → InputTag.2
3 Globe   → InputTag.3
4 Globe   → InputTag.4
```

每个 Globe 都会收到所有能力广播，但只处理与自己标签精确匹配的一条。这相当于一个基于 GameplayTag 的本地消息路由。

### 8.1 必须保证的初始化顺序

```text
1. 给每个 Globe 设置 InputTag
2. 给每个 Globe 设置 WidgetController
3. WidgetControllerSet 触发并绑定 AbilityInfoDelegate
4. WidgetController 广播 AbilityInfo
5. Globe 比较 InputTag 并刷新
```

如果先设置 Controller，广播可能发生在 InputTag 仍为空时；如果没有设置 Controller，`WidgetControllerSet` 根本不会执行，Globe 也就不会绑定委托。

---

<a id="section-9"></a>
## 9. 多人客户端为什么没有技能图标

### 9.1 问题根源

服务器调用 `GiveAbility` 并执行：

```cpp
AbilitiesGivenDelegate.Broadcast(this);
```

普通 C++ 委托不是网络消息。服务器上的广播不会自动传到客户端。客户端虽然随后复制到了 `ActivatableAbilities`，但它没有执行服务器本地的那次委托，所以本地 HUD 不知道何时刷新。

### 9.2 `OnRep_ActivateAbilities`

```cpp
void UAuraAbilitySystemComponent::OnRep_ActivateAbilities()
{
    Super::OnRep_ActivateAbilities();

    if (!bStartupAbilities)
    {
        bStartupAbilities = true;
        AbilitiesGivenDelegate.Broadcast(this);
    }
}
```

客户端完整时序：

```text
服务器 GiveAbility
  → ActivatableAbilities 网络复制
  → 客户端 ASC::OnRep_ActivateAbilities
  → 客户端本地广播 AbilitiesGivenDelegate
  → 客户端 OverlayWidgetController 遍历本地 AbilitySpec
  → 客户端本地 SpellGlobe 刷新图标
```

要点：

- `OnRep_ActivateAbilities` 是复制通知，不是再次授予能力；
- 必须调用 `Super::OnRep_ActivateAbilities()`；
- Widget 本身不需要复制，它是每台客户端本地创建的表现对象；
- 本地委托只负责告诉本地 UI：“复制数据现在可以用了”；
- bool 防止后续能力列表复制时重复执行启动初始化。

---

<a id="section-10"></a>
## 10. 本章 Bug 与排查方法

### Bug 1：所有技能槽都是空的

可能原因：`GA_FireBolt` 的 Class Defaults 没有配置 `Abilities.*` 标签。

结果链：

```text
GetAbilityTagFromSpec 返回空 Tag
→ AbilityInfo DataAsset 查询失败
→ 得到空 Info
→ 图标与背景为空
```

修复：在 Ability 类默认设置中配置正确的 AbilityTag，并确认 DataAsset 中存在同一标签的条目。

### Bug 2：某个 Globe 永远收不到数据

可能原因：父 Widget 没对它调用 `SetWidgetController`，或该控件未勾选 Is Variable。

排查顺序：

1. `WBP_Overlay` 是否给 `WBP_HealthManaSpells` 设置 Controller；
2. 父组件是否继续给所有子 Globe 设置 Controller；
3. Globe 的 `WidgetControllerSet` 是否触发；
4. 是否成功 Cast 为 OverlayWidgetController；
5. 是否绑定 `AbilityInfoDelegate`。

### Bug 3：只有主机有图标，客户端没有

原因：只在服务器 `AddCharacterAbilities` 之后广播了普通委托。

修复：客户端在 `OnRep_ActivateAbilities` 中广播自己的本地初始化事件。

### Bug 4：图标显示在错误槽位

排查：

- AbilitySpec 的动态标签中是否只有一个 `InputTag.*`；
- 每个 Globe 自己的 InputTag 是否配置正确；
- 比较是否使用 Exact Match；
- 是否在设置 Controller 前完成 InputTag 分配。

### Bug 5：自定义日志类别重定义

`AuraLogChannels.h` 被多个文件包含时发生重复声明。修复是在头文件顶部添加：

```cpp
#pragma once
```

日志类别应在 `.h` 中 `DECLARE_LOG_CATEGORY_EXTERN`，在一个 `.cpp` 中只 `DEFINE_LOG_CATEGORY` 一次。

---

<a id="section-11"></a>
## 11. 为什么这套设计天然支持技能换槽

UI 不知道 FireBolt 必须在 LMB。它只执行：

```text
从 Spec 读取 InputTag
→ 将 InputTag 随 AbilityInfo 广播
→ 匹配该 InputTag 的 Globe 显示图标
```

因此把 FireBolt 的输入标签由 LMB 改成 RMB，重新广播后图标就会移动到 RMB 槽。未来实现装备/换键时，核心流程可以是：

```text
找到 AbilitySpec
→ 移除旧 InputTag
→ 添加新 InputTag
→ 标记 Spec 需要复制
→ 清空旧槽并重新广播 AbilityInfo
```

动态换槽还要处理：目标槽已有技能、客户端预测/服务器权威、旧槽清空、Spec Dirty 和持久化配置，但本章的数据结构已经为这些功能打好了基础。

---

<a id="section-12"></a>
## 12. 完整通信流程

```text
[服务器初始化角色]
StartupAbilities
  → AddCharacterAbilities
  → 创建 FGameplayAbilitySpec
  → 写入 StartupInputTag 到 DynamicSpecSourceTags
  → GiveAbility
  → bStartupAbilities = true
  → AbilitiesGivenDelegate.Broadcast

[服务端/监听服务器本地 UI]
OverlayWidgetController
  → OnInitializeStartupAbilities
  → ASC.ForEachAbility
  → GetAbilityTagFromSpec
  → DA_AbilityInfo.FindAbilityInfoForTag
  → GetInputTagFromSpec
  → AbilityInfoDelegate.Broadcast
  → 匹配 InputTag 的 SpellGlobe 更新

[远程客户端]
ActivatableAbilities 复制
  → OnRep_ActivateAbilities
  → 本地 bStartupAbilities = true
  → 本地 AbilitiesGivenDelegate.Broadcast
  → 本地 WidgetController 与 SpellGlobe 执行同一刷新流程
```

---

<a id="section-13"></a>
## 13. 测试清单

### 单人 / Standalone

- Health、Mana 球更新正常；
- XP 条位置与层级正确；
- FireBolt 图标显示在预期输入槽；
- 未装备槽只显示 Ring/Glass，不出现白块；
- 修改 StartupInputTag 后，图标移动到新槽；
- DataAsset 缺失条目时能看到清晰的 `LogAura` 错误。

### 多人 PIE

- Listen Server 的图标正常；
- Client 1、Client 2 都能在能力复制后显示图标；
- 客户端没有再次调用 GiveAbility；
- `OnRep_ActivateAbilities` 中执行了 Super；
- 重复复制不会反复初始化启动技能；
- 每个客户端只更新自己的本地 HUD。

### 蓝图初始化

- 所有需要访问的子 Widget 已勾选 Is Variable；
- InputTag 在 SetWidgetController 之前设置；
- 所有 SpellGlobe 都收到相同 OverlayWidgetController；
- 每个 Globe 只处理精确匹配的 InputTag；
- Construct/WidgetControllerSet 多次执行时，不会意外重复绑定委托。

---

<a id="section-14"></a>
## 14. 核心理解

本章最重要的不是几个蓝图节点，而是三种数据职责：

- **GameplayAbility / AbilityTag**：定义能力身份；
- **GameplayAbilitySpec / InputTag**：保存角色当前获得的能力及运行时输入绑定；
- **AbilityInfo DataAsset**：保存与逻辑无关的 UI 展示资源。

ASC 是运行时事实来源，DataAsset 是静态表现配置，WidgetController 是二者的适配器，SpellGlobe 是纯显示终端。多人环境中，服务器复制的是能力状态，不复制 Widget，也不会复制普通委托；客户端必须在复制通知到达后，用本地委托唤醒本地 UI。

掌握这条链路后，后续的技能装备、换键、等级、冷却、法力消耗和禁用状态，都可以继续沿用同一模式扩展，而不用让每个 Widget 直接理解整个 GAS。

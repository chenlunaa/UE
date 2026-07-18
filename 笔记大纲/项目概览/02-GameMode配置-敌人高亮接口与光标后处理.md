# UE5 学习笔记 — 第三次提交

> 📦 Commit `6dede93`：GameMode 配置 + 敌人高光标记 + 完善运动系统  
> 📅 日期：2026-06-02  
> 🎬 对应视频：2.10 GameMode / 2.11 高亮接口 / 2.12 光标追踪 / 2.13 后处理高亮

---

## 目录

- [UE5 学习笔记 — 第三次提交](#ue5-学习笔记--第三次提交)
  - [目录](#目录)
  - [一、GameMode 游戏模式配置](#一gamemode-游戏模式配置)
    - [1.1 为什么需要 GameMode？](#11-为什么需要-gamemode)
    - [1.2 C++ 基类创建](#12-c-基类创建)
    - [1.3 蓝图配置流程](#13-蓝图配置流程)
  - [二、角色移动系统 + 动画修复](#二角色移动系统--动画修复)
    - [2.1 弹簧臂与相机（蓝图添加）](#21-弹簧臂与相机蓝图添加)
    - [2.2 AuraCharacter C++ 构造函数](#22-auracharacter-c-构造函数)
    - [2.3 动画蓝图修复：空闲 / 跑步状态分离](#23-动画蓝图修复空闲--跑步状态分离)
  - [三、Custom Depth 自定义深度配置](#三custom-depth-自定义深度配置)
    - [3.1 启用 Custom Depth-Stencil Pass](#31-启用-custom-depth-stencil-pass)
    - [3.2 自定义深度宏定义](#32-自定义深度宏定义)
  - [四、敌人高光标记系统（核心）](#四敌人高光标记系统核心)
    - [4.1 为什么用接口而不是直接 Cast？](#41-为什么用接口而不是直接-cast)
    - [4.2 EnemyInterface 接口定义](#42-enemyinterface-接口定义)
    - [4.3 AAuraEnemy 实现接口](#43-aauraenemy-实现接口)
  - [五、鼠标光标追踪系统](#五鼠标光标追踪系统)
    - [5.1 PlayerTick 每帧追踪](#51-playertick-每帧追踪)
    - [5.2 CursorTrace 核心逻辑](#52-cursortrace-核心逻辑)
    - [5.3 五状态机详解](#53-五状态机详解)
  - [六、后处理高亮效果配置](#六后处理高亮效果配置)
    - [6.1 后处理体积（Post Process Volume）](#61-后处理体积post-process-volume)
    - [6.2 PP\_Highlight 材质工作原理](#62-pp_highlight-材质工作原理)
    - [6.3 完整高亮流程](#63-完整高亮流程)
  - [七、新增 / 修改文件清单](#七新增--修改文件清单)
  - [八、知识点总结](#八知识点总结)

---

## 一、GameMode 游戏模式配置

### 1.1 为什么需要 GameMode？

游戏中有三个核心类需要关联：**玩家控制器**、**Pawn（角色）**、**GameMode**。GameMode 充当"粘合剂"——在世界设置中指定 GameMode 后，它会自动决定：
- 使用哪个玩家控制器类
- 使用哪个默认 Pawn 类
- 其他游戏规则类（HUD、GameState 等）

### 1.2 C++ 基类创建

在 `Source/Aura/Public/Game/` 下创建 `AAuraGameModeBase`，继承自 `AGameModeBase`（比 `AGameMode` 更轻量，适合不需要完整多人游戏框架的项目）。

```cpp
// AuraGameModeBase.h
class AURA_API AAuraGameModeBase : public AGameModeBase
{
    GENERATED_BODY()
};
```

### 1.3 蓝图配置流程

1. 在 `Content/Blueprints/Game/` 下创建蓝图类 `BP_AuraGameMode`，父类选 `AuraGameModeBase`
2. 打开蓝图，设置：
   - **Player Controller Class** → `BP_AuraPlayerController`
   - **Default Pawn Class** → `BP_AuraCharacter`
3. 在世界设置（World Settings）中，将 **GameMode Override** 设为 `BP_AuraGameMode`
4. 在关卡中放置一个 **Player Start** 来确定出生点

> ⚠️ 此时还没有相机和弹簧臂，需要下一步添加。

---

## 二、角色移动系统 + 动画修复

### 2.1 弹簧臂与相机（蓝图添加）

由于相机和弹簧臂不需要在 C++ 中做复杂逻辑，直接在 `BP_AuraCharacter` 蓝图中添加即可——运行时性能没有区别。

| 组件 | 配置 |
|------|------|
| **Spring Arm** | 挂载到 CapsuleComponent（根组件），旋转约 -45°（俯视角度），Target Arm Length ≈ 750 |
| **Camera** | 挂载到 Spring Arm 下方 |

**Spring Arm 关键设置：**
- `bUsePawnControlRotation` → **false**（固定相机，不跟随控制器旋转）
- `bInheritPitch / bInheritYaw / bInheritRoll` → **全部 false**
- 可选：`bEnableCameraLag` → true，增加相机延迟带来更平滑的跟拍效果

### 2.2 AuraCharacter C++ 构造函数

```cpp
AAuraCharacter::AAuraCharacter()
{
    GetCharacterMovement()->bOrientRotationToMovement = true;
    GetCharacterMovement()->RotationRate = FRotator(0.f, 400.f, 0.f);

    GetCharacterMovement()->bConstrainToPlane = true;
    GetCharacterMovement()->bSnapToPlaneAtStart = true;

    bUseControllerRotationPitch = false;
    bUseControllerRotationRoll = false;
    bUseControllerRotationYaw = false;
}
```

| 属性 | 值 | 作用 |
|------|-----|------|
| `bOrientRotationToMovement` | `true` | 角色自动朝向移动方向旋转 |
| `RotationRate` | `(0, 400, 0)` | 仅 Yaw 轴每秒旋转 400°，Pitch/Roll 不旋转 |
| `bConstrainToPlane` | `true` | 约束在平面上移动（俯视角游戏标配） |
| `bSnapToPlaneAtStart` | `true` | 开始时自动吸附到平面 |
| `bUseControllerRotation*` | `false` × 3 | 禁止控制器旋转影响角色朝向 |

> 🎯 **设计意图**：俯视角 RPG 移动方案——角色在平面上移动，自动面向移动方向，相机固定俯拍。

### 2.3 动画蓝图修复：空闲 / 跑步状态分离

**问题**：角色停止移动后，头部会轻微前后摆动。

**原因**：Idle/Walk/Run 混合空间（Blend Space）一直处于激活状态，即使速度为 0 也会在 Idle 和 Walk 之间插值。

**解决方案**——在动画蓝图中引入状态机：

```
[入口] → Idle 状态（仅播放 Idle 动画，Loop）
                ↓ ShouldMove == true
           Run 状态（使用 Blend Space）
                ↓ ShouldMove == false
           Idle 状态
```

- 创建 `ShouldMove` 布尔变量
- 每帧判断：`GroundSpeed > 3` → `ShouldMove = true`，否则 `false`
- Idle → Run：条件为 `ShouldMove == true`
- Run → Idle：条件为 `ShouldMove == false`
- Idle 动画记得勾选 **Loop Animation**

---

## 三、Custom Depth 自定义深度配置

### 3.1 启用 Custom Depth-Stencil Pass

在 `Config/DefaultEngine.ini` 中添加：

```ini
r.CustomDepth=3
```

> 也可以在编辑器 **Project Settings → Rendering → Custom Depth-Stencil Pass** 中设置为 **"Enabled with Stencil"**。

- `r.CustomDepth=3` 表示启用 **Custom Depth + Stencil Pass**，允许渲染自定义深度值和模板值。
- 这是实现**物体高亮/描边**效果的前提——没有它，后处理材质无法读取 Custom Depth 信息。

### 3.2 自定义深度宏定义

在 `Source/Aura/Aura.h` 中定义：

```cpp
#define CUSTOM_DEPTH_RED 250
```

- Custom Depth Stencil Value 范围 0-255，不同值对应后处理材质中的不同描边颜色。
- 本项目使用的后处理材质 `PP_Highlight` 将 **250 映射为红色轮廓**。
- 本次只保留红色（250），删除了之前的蓝色（251）和棕色（252）。

---

## 四、敌人高光标记系统（核心）

### 4.1 为什么用接口而不是直接 Cast？

**直觉做法**：在 `CursorTrace` 中 `Cast<AAuraEnemy>(HitActor)`，如果是敌人就高亮。

**这种做法的两个问题：**

| 问题 | 说明 |
|------|------|
| **类型耦合** | `AuraPlayerController` 会依赖 `AAuraEnemy`，破坏了代码的灵活性 |
| **扩展性差** | 如果将来要对门、桶、NPC 也做高亮，需要逐个 Cast 或修改类继承 |

**接口方案的优势：**

```
PlayerController 只需要知道：
  "这个 Actor 实现了 IEnemyInterface 吗？"
  → 是 → 调用 HighlightActor() / UnHighlightActor()
  → 否 → 忽略

每个 Actor 自己决定如何响应高亮：
  - 敌人 → 红色描边
  - 门   → 绿色描边
  - 桶   → 黄色描边
  ...
```

> 💡 尽管接口名叫 `EnemyInterface`，但实际上它更接近 `HighlightInterface` 或 `TargetInterface`。课程作者也承认命名不太准确，但为了教程一致性保留了原名。

### 4.2 EnemyInterface 接口定义

**头文件** `Source/Aura/Public/Interaction/EnemyInterface.h`：

```cpp
class AURA_API IEnemyInterface
{
    GENERATED_BODY()
public:
    virtual void HighlightActor() = 0;   // 纯虚函数：高亮
    virtual void UnHighlightActor() = 0; // 纯虚函数：取消高亮
};
```

- `= 0` 表示**纯虚函数**（pure virtual），使该类成为**抽象类**。
- 任何继承 `IEnemyInterface` 的类**必须**重写这两个函数，否则编译报错：`cannot instantiate abstract class`。

### 4.3 AAuraEnemy 实现接口

**头文件修改** `Source/Aura/Public/Character/AuraEnemy.h`：

```cpp
class AURA_API AAuraEnemy : public AAuraCharacterBase, public IEnemyInterface
{
    GENERATED_BODY()
public:
    AAuraEnemy();
    virtual void HighlightActor() override;
    virtual void UnHighlightActor() override;
};
```

**实现** `Source/Aura/Private/Character/AuraEnemy.cpp`：

```cpp
AAuraEnemy::AAuraEnemy()
{
    // 让敌人 Mesh 阻挡 Visibility 通道，使鼠标射线能命中
    GetMesh()->SetCollisionResponseToChannel(ECC_Visibility, ECR_Block);
}

void AAuraEnemy::HighlightActor()
{
    GetMesh()->SetRenderCustomDepth(true);
    GetMesh()->SetCustomDepthStencilValue(CUSTOM_DEPTH_RED);
    Weapon->SetRenderCustomDepth(true);
    Weapon->SetCustomDepthStencilValue(CUSTOM_DEPTH_RED);
}

void AAuraEnemy::UnHighlightActor()
{
    GetMesh()->SetRenderCustomDepth(false);
    Weapon->SetRenderCustomDepth(false);
}
```

| 关键点 | 说明 |
|--------|------|
| `SetCollisionResponseToChannel(ECC_Visibility, ECR_Block)` | 在 C++ 构造函数中统一设置碰撞，避免逐个蓝图手动配置 |
| `SetRenderCustomDepth(true)` | 开启该组件的自定义深度渲染 |
| `SetCustomDepthStencilValue(250)` | 设为 250 → 后处理材质渲染红色描边 |
| `Weapon` 也参与高亮 | 武器与身体同时高亮/取消，保证视觉统一 |

---

## 五、鼠标光标追踪系统

### 5.1 PlayerTick 每帧追踪

```cpp
void AAuraPlayerController::PlayerTick(float DeltaTime)
{
    Super::PlayerTick(DeltaTime);
    CursorTrace();
}
```

- 在 `PlayerTick` 中每帧调用 `CursorTrace()`。
- 这是一个轻量操作（一次射线检测 + 几次指针比较），每帧执行不会有性能问题，而且能保证即时响应。

### 5.2 CursorTrace 核心逻辑

```cpp
void AAuraPlayerController::CursorTrace()
{
    FHitResult CursorHit;
    GetHitResultUnderCursor(ECC_Visibility, false, CursorHit);
    if (!CursorHit.bBlockingHit) return;

    LastActor = ThisActor;                           // 保存上一帧
    ThisActor = Cast<IEnemyInterface>(CursorHit.GetActor());  // 当前帧

    // 状态机判断...
}
```

**关键 API：**
- `GetHitResultUnderCursor(Channel, bTraceComplex, HitResult)` — 从鼠标位置向场景发射射线
- `ECC_Visibility` — 使用可见性碰撞通道（敌人已在构造函数中设为 Block）
- `Cast<IEnemyInterface>()` — 如果 Actor 实现了接口则返回有效指针，否则返回 `nullptr`

**成员变量：**
```cpp
IEnemyInterface* LastActor;  // 上一帧鼠标下的接口 Actor
IEnemyInterface* ThisActor;  // 当前帧鼠标下的接口 Actor
```

### 5.3 五状态机详解

每帧根据 `LastActor` 和 `ThisActor` 的值判断当前状态：

| 状态 | LastActor | ThisActor | 含义 | 处理 |
|------|-----------|-----------|------|------|
| **A** | `nullptr` | `nullptr` | 没瞄准任何可交互物体 | 什么都不做 |
| **B** | `nullptr` | 有效 | 刚刚进入敌人（首次悬停） | 🔴 高亮 `ThisActor` |
| **C** | 有效 | `nullptr` | 刚刚离开敌人 | ⚪ 取消高亮 `LastActor` |
| **D** | 有效（≠） | 有效（≠） | 从一个敌人切换到另一个 | ⚪ 取消高亮 `LastActor` + 🔴 高亮 `ThisActor` |
| **E** | 有效（＝） | 有效（＝） | 持续瞄准同一个敌人 | 什么都不做 |

```cpp
// 伪代码实现
if (LastActor == nullptr)
{
    if (ThisActor != nullptr)
        ThisActor->HighlightActor();   // 情况 B
    // else: 情况 A — 什么都不做
}
else  // LastActor 有效
{
    if (ThisActor == nullptr)
        LastActor->UnHighlightActor(); // 情况 C
    else if (LastActor != ThisActor)
    {
        LastActor->UnHighlightActor(); // 情况 D
        ThisActor->HighlightActor();
    }
    // else: 情况 E — 什么都不做
}
```

> 🎯 **为什么用状态机？** 避免每帧重复调用 `HighlightActor()` / `UnHighlightActor()`，只在状态**变化**时触发。虽然 `SetRenderCustomDepth(true)` 本身是廉价操作，但良好的设计习惯能避免未来扩展时的隐患。

---

## 六、后处理高亮效果配置

### 6.1 后处理体积（Post Process Volume）

1. 在关卡中拖入 **Post Process Volume**
2. 勾选 **Infinite Extent (Unbound)** → 影响整个关卡
3. 在 **Post Process Materials** 数组中添加 `PP_Highlight` 材质

### 6.2 PP_Highlight 材质工作原理

- 该材质读取场景中每个像素的 **Custom Depth Stencil Value**
- 当值为 **250** 时 → 渲染**红色轮廓**
- 轮廓粗细可通过材质中的参数调整（默认 1.6）

### 6.3 完整高亮流程

```
鼠标悬停在敌人上
  → CursorTrace 检测到 ThisActor 实现了 IEnemyInterface
  → 调用 HighlightActor()
    → GetMesh()->SetRenderCustomDepth(true)
    → GetMesh()->SetCustomDepthStencilValue(250)
  → 后处理体积的 PP_Highlight 材质读取到 Stencil=250
  → 渲染红色轮廓 ✨
```

---

## 七、新增 / 修改文件清单

| 文件 | 类型 | 说明 |
|------|------|------|
| `Source/Aura/Public/Game/AuraGameModeBase.h` | 新增 | GameMode 基类声明 |
| `Source/Aura/Private/Game/AuraGameModeBase.cpp` | 新增 | GameMode 基类实现（空） |
| `Source/Aura/Public/Interaction/EnemyInterface.h` | 新增 | 敌人高亮接口声明 |
| `Source/Aura/Private/Interaction/EnemyInterface.cpp` | 新增 | 接口实现（空，纯虚函数无需实现） |
| `Source/Aura/Public/Character/AuraEnemy.h` | 修改 | 继承 `IEnemyInterface`，添加高亮函数声明和构造函数 |
| `Source/Aura/Private/Character/AuraEnemy.cpp` | 修改 | 实现高亮/取消高亮 + 碰撞通道设置 |
| `Source/Aura/Private/Character/AuraCharacter.cpp` | 修改 | 添加构造函数，配置移动参数 |
| `Source/Aura/Aura.h` | 修改 | 添加 `CUSTOM_DEPTH_RED` 宏 |
| `Config/DefaultEngine.ini` | 修改 | 添加 `r.CustomDepth=3` |
| `Content/Blueprints/Game/BP_AuraGameMode` | 蓝图新增 | 配置 PlayerController 和 Default Pawn |
| `Content/Blueprints/Character/BP_AuraCharacter` | 蓝图修改 | 添加 Spring Arm + Camera 组件 |
| 动画蓝图 `ABP_Enemy` | 蓝图修改 | 添加 Idle/Run 状态机分离 |

---

## 八、知识点总结

| # | 知识点 | 要点 |
|---|--------|------|
| 1 | **GameMode** | 连接 PlayerController 和 Pawn 的桥梁，在世界设置中指定 |
| 2 | **Spring Arm + Camera** | 俯视角游戏标配：固定旋转、约束到平面、可选相机延迟 |
| 3 | **Custom Depth-Stencil** | UE5 物体描边的标准方案：开启 Stencil Pass → 设置 Stencil Value → 后处理材质读取 |
| 4 | **UInterface + 纯虚函数** | 跨类多态的推荐方式，解耦调用方和实现方 |
| 5 | **Cast vs Interface** | 直接 Cast 导致类型耦合；Interface 让代码更灵活、更易扩展 |
| 6 | **PlayerTick + CursorTrace** | 俯视角游戏鼠标交互的标准模式，每帧射线检测 |
| 7 | **五状态机** | 用 LastActor/ThisActor 双指针追踪鼠标进出，避免重复操作 |
| 8 | **碰撞通道** | `ECC_Visibility` + `ECR_Block` 让鼠标射线能命中敌人 Mesh |
| 9 | **动画状态机** | Idle 和 Run 分离，避免 Blend Space 在速度为 0 时产生抖动 |
| 10 | **C++ vs 蓝图** | 碰撞设置、移动参数在 C++ 构造函数中统一配置；相机等纯表现层用蓝图更快 |

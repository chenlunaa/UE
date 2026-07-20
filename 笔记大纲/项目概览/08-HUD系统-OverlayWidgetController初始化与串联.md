# UE5 学习笔记 — 第八次提交

> 📦 Commit `63fa93e`：Overlay Widget Controller 初始化配置（HUD 系统 + 参数结构体 + 控制器连接）  
> 📅 日期：2026-06-10  
> 🎬 对应视频：5.5 创建 AuraHUD / 5.6 WidgetController 初始化与串联

---

## 目录

- [UE5 学习笔记 — 第八次提交](#ue5-学习笔记--第八次提交)
  - [目录](#目录)
  - [一、整体目标：从关卡蓝图迁移到 HUD 系统](#一整体目标从关卡蓝图迁移到-hud-系统)
  - [二、AAuraHUD — 自定义 HUD 类（5.5 核心）](#二aaurahud--自定义-hud-类55-核心)
    - [2.1 为什么需要自定义 HUD](#21-为什么需要自定义-hud)
    - [2.2 AAuraHUD 成员变量设计](#22-aaurahud-成员变量设计)
    - [2.3 蓝图侧的配置流程](#23-蓝图侧的配置流程)
  - [三、FWidgetControllerParams — 四关键变量参数结构体（5.6 核心）](#三fwidgetcontrollerparams--四关键变量参数结构体56-核心)
    - [3.1 设计动机：一键初始化](#31-设计动机一键初始化)
    - [3.2 结构体完整代码解析](#32-结构体完整代码解析)
    - [3.3 SetWidgetControllerParams — 基类赋值函数](#33-setwidgetcontrollerparams--基类赋值函数)
  - [四、UOverlayWidgetController — 覆盖层专用控制器](#四uoverlaywidgetcontroller--覆盖层专用控制器)
    - [4.1 继承关系](#41-继承关系)
    - [4.2 为什么用子类而非直接用基类](#42-为什么用子类而非直接用基类)
  - [五、AAuraHUD 的核心函数详解](#五aaurahud-的核心函数详解)
    - [5.1 GetOverlayWidgetController — 单例式获取器](#51-getoverlaywidgetcontroller--单例式获取器)
    - [5.2 InitOverlay — 总装函数](#52-initoverlay--总装函数)
  - [六、调用链路：从角色到 HUD 的完整串联](#六调用链路从角色到-hud-的完整串联)
    - [6.1 InitAbilityActorInfo 中的新增代码](#61-initabilityactorinfo-中的新增代码)
    - [6.2 为什么选择 InitAbilityActorInfo 作为调用点](#62-为什么选择-initabilityactorinfo-作为调用点)
    - [6.3 完整调用流程图](#63-完整调用流程图)
  - [七、多人游戏注意事项（重要！）](#七多人游戏注意事项重要)
    - [7.1 玩家控制器的有效性](#71-玩家控制器的有效性)
    - [7.2 check 与 if 的选择原则](#72-check-与-if-的选择原则)
    - [7.3 AuraPlayerController::BeginPlay 的修复](#73-auraplayercontrollerbeginplay-的修复)
  - [八、新增/修改文件清单](#八新增修改文件清单)
  - [九、知识点总结](#九知识点总结)

---

## 一、整体目标：从关卡蓝图迁移到 HUD 系统

在上一阶段，Overlay Widget（健康球 + 法力球）是在**关卡蓝图（Level Blueprint）**中手动创建并添加到视口的。这只是临时测试手段。

**本次提交的核心目标**：

```
关卡蓝图（临时） ──迁移──▶ AAuraHUD（正式系统）
                         │
                         ├─ 创建 Overlay Widget
                         ├─ 创建 OverlayWidgetController
                         ├─ 将 Controller 绑定到 Widget
                         └─ 添加到视口
```

> ⚠️ **关键理念**：不要在关卡蓝图中放置游戏逻辑代码。HUD 是 UE 引擎专门为"屏幕 UI 管理"设计的类。

---

## 二、AAuraHUD — 自定义 HUD 类（5.5 核心）

### 2.1 为什么需要自定义 HUD

UE 引擎的 GameMode 有默认类配置：

| 默认类                     | 用途                  |
| ----------------------- | ------------------- |
| Default Pawn Class      | 默认角色                |
| Player Controller Class | 玩家控制器               |
| **HUD Class**           | **屏幕 UI 管理** ← 本次使用 |

`AHUD` 是 UE 内置的基类，专门负责：

- 将 Widget 绘制到屏幕上
- 管理 HUD 相关的生命周期
- 提供 `GetWorld()` 等上下文

### 2.2 AAuraHUD 成员变量设计

```cpp
// 文件：Source/Aura/Public/UI/HUD/AuraHUD.h

UCLASS()
class AURA_API AAuraHUD : public AHUD
{
    GENERATED_BODY()

public:
    // ★ 创建后的小部件实例（运行时）
    UPROPERTY()
    TObjectPtr<UAuraUserWidget> OverlayWidget;

    // ★ 获取或创建 OverlayWidgetController（单例模式）
    UOverlayWidgetController* GetOverlayWidgetController(
        const FWidgetControllerParams& WCParams);

    // ★ 总装函数：创建 Widget + Controller + 绑定 + 添加到视口
    void InitOverlay(APlayerController* PC, APlayerState* PS,
                     UAbilitySystemComponent* ASC, UAttributeSet* AS);

private:
    // ★ 要创建的 Widget 蓝图类（在蓝图中配置）
    UPROPERTY(EditAnywhere)
    TSubclassOf<UAuraUserWidget> OverlayWidgetClass;

    // ★ 已创建的 WidgetController 实例（单例缓存）
    UPROPERTY()
    TObjectPtr<UOverlayWidgetController> OverlayWidgetController;

    // ★ 要创建的 WidgetController 蓝图类（在蓝图中配置）
    UPROPERTY(EditAnywhere)
    TSubclassOf<UOverlayWidgetController> OverlayWidgetControllerClass;
};
```

**设计要点**：

| 变量                             | 类型                                      | 用途                     |
| ------------------------------ | --------------------------------------- | ---------------------- |
| `OverlayWidgetClass`           | `TSubclassOf<UAuraUserWidget>`          | 蓝图配置：用哪个 Widget 蓝图     |
| `OverlayWidgetControllerClass` | `TSubclassOf<UOverlayWidgetController>` | 蓝图配置：用哪个 Controller 蓝图 |
| `OverlayWidget`                | `TObjectPtr<UAuraUserWidget>`           | 运行时实例缓存                |
| `OverlayWidgetController`      | `TObjectPtr<UOverlayWidgetController>`  | 运行时单例缓存                |

> 💡 `TSubclassOf` 用于在蓝图中选择类，`TObjectPtr` 用于持有运行时实例。两个 `EditAnywhere` 变量让美术/策划在蓝图编辑器中即可切换不同的 Widget 和 Controller。

### 2.3 蓝图侧的配置流程

1. 创建蓝图类 `BP_AuraHUD`，父类为 `AAuraHUD`
2. 在 `BP_AuraHUD` 中设置：
   - `OverlayWidgetClass` → `WBP_Overlay`
   - `OverlayWidgetControllerClass` → `OverlayWidgetController`（C++ 类或蓝图子类）
3. 在 `BP_AuraGameMode` 中将 `HUD Class` 设为 `BP_AuraHUD`

---

## 三、FWidgetControllerParams — 四关键变量参数结构体（5.6 核心）

### 3.1 设计动机：一键初始化

WidgetController 需要四个关键变量才能工作：

```
PlayerController → 访问 HUD、输入等
PlayerState     → 获取玩家数据
ASC             → 能力系统组件
AttributeSet    → 属性数据
```

**问题**：每次创建 WidgetController 都要分别设置这四个变量，容易遗漏。

**解决方案**：创建一个结构体 `FWidgetControllerParams`，将四个变量打包，一次传递即可完成初始化。

### 3.2 结构体完整代码解析

```cpp
// 文件：Source/Aura/Public/UI/WidgetController/AuraWidgetController.h

USTRUCT(BlueprintType)           // ★ 可在蓝图中使用
struct FWidgetControllerParams
{
    GENERATED_BODY()

    // 默认构造函数（必须有，UE 反射系统要求）
    FWidgetControllerParams() {}

    // ★ 核心：带参数的构造函数，使用成员初始化列表
    FWidgetControllerParams(
        APlayerController* PC,
        APlayerState* PS,
        UAbilitySystemComponent* ASC,
        UAttributeSet* AS
    ) : PlayerController(PC),
        PlayerState(PS),
        AbilitySystemponent(ASC),
        AttributeSet(AS) {}

    // ★ 四个关键变量，全部初始化为 nullptr
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<APlayerController> PlayerController = nullptr;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<APlayerState> PlayerState = nullptr;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<UAbilitySystemComponent> AbilitySystemponent = nullptr;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<UAttributeSet> AttributeSet = nullptr;
};
```

**设计要点**：

| 特性                       | 说明                        |
| ------------------------ | ------------------------- |
| `USTRUCT(BlueprintType)` | 结构体可在蓝图中作为变量类型使用          |
| 默认构造函数                   | UE 反射系统要求，必须有             |
| 带参构造函数                   | 使用**成员初始化列表**（冒号语法），高效初始化 |
| `= nullptr`              | 所有指针初始化为空，避免编译器警告         |
| `BlueprintReadWrite`     | 蓝图可读写，方便调试和配置             |

### 3.3 SetWidgetControllerParams — 基类赋值函数

```cpp
// 文件：Source/Aura/Private/UI/WidgetController/AuraWidgetController.cpp

void UAuraWidgetController::SetWidgetControllerParams(
    const FWidgetControllerParams& WCParams)
{
    PlayerController = WCParams.PlayerController;
    PlayerState = WCParams.PlayerState;
    AbilitySystemComponent = WCParams.AbilitySystemponent;
    AttributeSet = WCParams.AttributeSet;
}
```

- 标记为 `UFUNCTION(BlueprintCallable)`，蓝图也可调用
- 参数为 `const &`（常量引用），避免拷贝
- 一步到位设置所有四个关键变量

---

## 四、UOverlayWidgetController — 覆盖层专用控制器

### 4.1 继承关系

```
UObject
  └─ UAuraWidgetController        ← 基类（持有四个关键变量）
       └─ UOverlayWidgetController ← 覆盖层专用子类（本次新增）
```

```cpp
// 文件：Source/Aura/Public/UI/WidgetController/OverlayWidgetController.h

UCLASS()
class AURA_API UOverlayWidgetController : public UAuraWidgetController
{
    GENERATED_BODY()
    // 目前为空壳，后续将添加覆盖层特有的逻辑
};
```

### 4.2 为什么用子类而非直接用基类

| 原因       | 说明                                                  |
| -------- | --------------------------------------------------- |
| **扩展性**  | 覆盖层未来需要特有的功能（如属性监听、广播），放在子类中不污染基类                   |
| **类型安全** | `AAuraHUD` 明确持有 `UOverlayWidgetController*`，编译时类型检查 |
| **蓝图支持** | 可以为覆盖层创建专门的蓝图子类，在蓝图中配置覆盖层特有的数据                      |

---

## 五、AAuraHUD 的核心函数详解

### 5.1 GetOverlayWidgetController — 单例式获取器

```cpp
UOverlayWidgetController* AAuraHUD::GetOverlayWidgetController(
    const FWidgetControllerParams& WCParams)
{
    if (OverlayWidgetController == nullptr)
    {
        // ★ 使用 NewObject 创建 UObject（不能用 new）
        OverlayWidgetController = NewObject<UOverlayWidgetController>(
            this,                           // Outer（所有者）
            OverlayWidgetControllerClass);  // 要创建的类

        // ★ 设置四个关键变量
        OverlayWidgetController->SetWidgetControllerParams(WCParams);

        return OverlayWidgetController;
    }
    return OverlayWidgetController;  // 已存在，直接返回
}
```

**单例模式要点**：

```
第一次调用 → OverlayWidgetController == nullptr → NewObject 创建 → 设置参数 → 返回
后续调用   → OverlayWidgetController != nullptr → 直接返回（不重复创建）
```

> ⚠️ **为什么用 `NewObject` 而不是 `new`**：UE 的 UObject 系统需要 GC（垃圾回收）管理，`NewObject` 会正确注册到 UE 的对象系统中。`new` 创建的 UObject 不会被 GC 追踪，可能导致内存泄漏或崩溃。

### 5.2 InitOverlay — 总装函数

```cpp
void AAuraHUD::InitOverlay(
    APlayerController* PC,
    APlayerState* PS,
    UAbilitySystemComponent* ASC,
    UAttributeSet* AS)
{
    // ★ 第一步：checkf 断言 —— 确保蓝图配置完整
    checkf(OverlayWidgetClass,
        TEXT("Overlay Widget Class uninitialized, please fill out BP_AuraHUD"));
    checkf(OverlayWidgetControllerClass,
        TEXT("Overlay Widget Controller Class uninitialized, please fill out BP_AuraHUD"));

    // ★ 第二步：创建 Widget
    UUserWidget* Widget = CreateWidget<UUserWidget>(GetWorld(), OverlayWidgetClass);
    OverlayWidget = Cast<UAuraUserWidget>(Widget);

    // ★ 第三步：构建参数结构体
    const FWidgetControllerParams WidgetControllerParams(PC, PS, ASC, AS);

    // ★ 第四步：获取或创建 WidgetController
    UOverlayWidgetController* WidgetController =
        GetOverlayWidgetController(WidgetControllerParams);

    // ★ 第五步：将 Controller 绑定到 Widget
    OverlayWidget->SetWidgetController(WidgetController);

    // ★ 第六步：添加到视口（显示在屏幕上）
    Widget->AddToViewport();
}
```

**执行顺序图解**：

```
InitOverlay(PC, PS, ASC, AS)
    │
    ├─ ① checkf → 确保蓝图配置了 WidgetClass 和 ControllerClass
    │
    ├─ ② CreateWidget → 实例化 Overlay Widget
    │
    ├─ ③ FWidgetControllerParams(PC, PS, ASC, AS) → 打包四关键变量
    │
    ├─ ④ GetOverlayWidgetController → 单例获取/创建 Controller
    │       └─ NewObject + SetWidgetControllerParams
    │
    ├─ ⑤ SetWidgetController → Widget 持有 Controller 引用
    │
    └─ ⑥ AddToViewport → 显示到屏幕
```

> 💡 `checkf` vs `check`：`checkf` 在断言失败时输出格式化的错误消息，方便调试。这里用它提示开发者"请在 BP_AuraHUD 中填写配置"。

---

## 六、调用链路：从角色到 HUD 的完整串联

### 6.1 InitAbilityActorInfo 中的新增代码

```cpp
// 文件：Source/Aura/Private/Character/AuraCharacter.cpp

void AAuraCharacter::InitAbilityActorInfo()
{
    AAuraPlayerState* AuraPlayerState = GetPlayerState<AAuraPlayerState>();
    check(AuraPlayerState);
    AuraPlayerState->GetAbilitySystemComponent()->InitAbilityActorInfo(
        AuraPlayerState, this);
    AbilitySystemComponent = AuraPlayerState->GetAbilitySystemComponent();
    AttributeSet = AuraPlayerState->GetAttributeSet();

    // ★★★ 新增：初始化 Overlay HUD ★★★
    if (AAuraPlayerController* AuraPlayerController =
            Cast<AAuraPlayerController>(GetController()))
    {
        if (AAuraHUD* AuraHUD =
                Cast<AAuraHUD>(AuraPlayerController->GetHUD()))
        {
            AuraHUD->InitOverlay(
                AuraPlayerController,    // PC
                AuraPlayerState,         // PS
                AbilitySystemComponent,  // ASC
                AttributeSet);           // AS
        }
    }
}
```

### 6.2 为什么选择 InitAbilityActorInfo 作为调用点

| 条件               | 说明                                                                     |
| ---------------- | ---------------------------------------------------------------------- |
| **四个关键变量都已就绪**   | `AuraPlayerController`、`AuraPlayerState`、`ASC`、`AttributeSet` 都在此函数中获取 |
| **服务器 + 客户端都覆盖** | `PossessedBy`（服务器）和 `OnRep_PlayerState`（客户端）都会调用此函数                    |
| **时机正确**         | Controller 已分配、PlayerState 已复制、ASC 已初始化                                |

### 6.3 完整调用流程图

```
游戏启动
  │
  ├─ 服务器
  │   └─ AAuraCharacter::PossessedBy(NewController)
  │       └─ InitAbilityActorInfo()
  │           ├─ 初始化 ASC、AttributeSet
  │           └─ Cast<AAuraPlayerController> → Cast<AAuraHUD>
  │               └─ AuraHUD->InitOverlay(PC, PS, ASC, AS)
  │                   ├─ CreateWidget (OverlayWidgetClass)
  │                   ├─ NewObject (OverlayWidgetControllerClass)
  │                   ├─ SetWidgetControllerParams
  │                   ├─ SetWidgetController
  │                   └─ AddToViewport
  │
  └─ 客户端
      └─ AAuraCharacter::OnRep_PlayerState()
          └─ InitAbilityActorInfo()
              └─ （同上）
```

---

## 七、多人游戏注意事项（重要！）

### 7.1 玩家控制器的有效性

这是本次提交中**最重要的多人游戏知识点**：

| 场景                | PlayerController 是否有效 |
| ----------------- | --------------------- |
| 服务器上的所有角色         | ✅ 有效（服务器持有所有玩家的控制器）   |
| 客户端上**本地控制**的角色   | ✅ 有效                  |
| 客户端上**其他玩家**的角色副本 | ❌ **nullptr**         |

> 🔑 **核心规则**：在多人游戏中，每个客户端只能访问自己本地玩家的 PlayerController。其他玩家的角色副本在该客户端上 `GetController()` 返回 `nullptr`。

### 7.2 check 与 if 的选择原则

```cpp
// ✅ 正确：使用 if 检查（因为可能为 null）
if (AAuraPlayerController* AuraPlayerController =
        Cast<AAuraPlayerController>(GetController()))
{
    if (AAuraHUD* AuraHUD = Cast<AAuraHUD>(AuraPlayerController->GetHUD()))
    {
        AuraHUD->InitOverlay(...);
    }
}

// ❌ 错误：使用 check 断言（会导致非本地角色崩溃）
// check(AuraPlayerController);  // 客户端上其他玩家的角色会在这里崩溃！
```

| 判断标准                   | 使用 `check`                              | 使用 `if`                                           |
| ---------------------- | --------------------------------------- | ------------------------------------------------- |
| 变量**永远不应**为 null       | ✅                                       | ❌                                                 |
| 变量**在某些合法情况下**可能为 null | ❌                                       | ✅                                                 |
| 示例                     | `check(AuraPlayerState)` — PS 在所有机器上都存在 | `if (AuraPlayerController)` — PC 在客户端非本地角色上为 null |

### 7.3 AuraPlayerController::BeginPlay 的修复

同样的原则也适用于 `BeginPlay` 中的输入子系统：

```cpp
// 修复前（会崩溃）
UEnhancedInputLocalPlayerSubsystem* Subsystem = ...;
check(Subsystem);  // ❌ 非本地控制客户端上 Subsystem 为 null
Subsystem->AddMappingContext(AuraContext, 0);

// 修复后（安全）
UEnhancedInputLocalPlayerSubsystem* Subsystem = ...;
if (Subsystem)  // ✅ 只在有效时执行
{
    Subsystem->AddMappingContext(AuraContext, 0);
}
```

---

## 八、新增/修改文件清单

| 文件                                                        | 操作     | 说明                                                                |
| --------------------------------------------------------- | ------ | ----------------------------------------------------------------- |
| `Public/UI/HUD/AuraHUD.h`                                 | **新增** | 自定义 HUD 类声明                                                       |
| `Private/UI/HUD/AuraHUD.cpp`                              | **新增** | HUD 实现：InitOverlay + GetOverlayWidgetController                   |
| `Public/UI/WidgetController/OverlayWidgetController.h`    | **新增** | 覆盖层专用 WidgetController 子类                                         |
| `Private/UI/WidgetController/OverlayWidgetController.cpp` | **新增** | 覆盖层控制器实现（目前为空壳）                                                   |
| `Public/UI/WidgetController/AuraWidgetController.h`       | **修改** | 新增 `FWidgetControllerParams` 结构体 + `SetWidgetControllerParams` 函数 |
| `Private/UI/WidgetController/AuraWidgetController.cpp`    | **修改** | 实现 `SetWidgetControllerParams`                                    |
| `Private/Character/AuraCharacter.cpp`                     | **修改** | 在 `InitAbilityActorInfo` 末尾调用 `InitOverlay`                       |
| `Private/Player/AuraPlayerController.cpp`                 | **修改** | 将 `check(Subsystem)` 改为 `if (Subsystem)`                          |

---

## 九、知识点总结

| 序号  | 知识点                         | 说明                                                            |
| --- | --------------------------- | ------------------------------------------------------------- |
| 1   | **HUD 系统**                  | `AHUD` 是 UE 管理屏幕 UI 的专用类，通过 GameMode 配置                       |
| 2   | **TSubclassOf**             | 模板类，用于在蓝图中选择 UClass 类型（编译时类型安全）                               |
| 3   | **NewObject**               | 创建 UObject 的正确方式（不能用 `new`），需要指定 Outer                        |
| 4   | **FWidgetControllerParams** | 参数结构体模式，将多个初始化参数打包，简化函数签名                                     |
| 5   | **成员初始化列表**                 | C++ 构造函数中 `: var(value)` 的语法，比在函数体内赋值更高效                      |
| 6   | **checkf**                  | 带格式化消息的断言，失败时输出自定义错误信息                                        |
| 7   | **单例模式**                    | `GetOverlayWidgetController` 确保全局只有一个 OverlayWidgetController |
| 8   | **多人游戏 PC 有效性**             | 客户端上非本地控制角色的 PlayerController 为 `nullptr`                     |
| 9   | **check vs if**             | 永远不应为 null → `check`；可能合法为 null → `if`                        |
| 10  | **MVC 串联**                  | HUD 作为"组装工厂"，将 Widget（View）和 WidgetController（Controller）绑定   |

---

> 📝 **架构小结**：本次提交完成了 UI 系统的"骨架搭建"。`AAuraHUD` 作为中央工厂，负责创建 Widget 和 Controller、绑定二者、并添加到屏幕。`FWidgetControllerParams` 优雅地解决了多参数传递问题。`InitAbilityActorInfo` 成为关键的"串联点"，在服务器和客户端都能正确触发 UI 初始化。

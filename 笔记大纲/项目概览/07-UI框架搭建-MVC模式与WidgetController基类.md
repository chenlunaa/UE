# UE5 学习笔记 — 第七次提交

> 📦 Commit `bbf88b2`：新增 UI 绘制（MVC 架构 + Widget 基类 + WidgetController）  
> 📅 日期：2026-06-08  
> 🎬 对应视频：5.1 MVC 架构设计 / 5.2 创建 Widget 和 WidgetController 基类 / 5.3 GlobeProgressBar 蓝图制作 / 5.4 健康球与法力球

---

## 目录

- [一、MVC 架构设计（5.1 核心）](#section-3)
  - [1.1 为什么需要 MVC](#section-4)
  - [1.2 三层职责划分](#section-5)
  - [1.3 单向依赖关系](#section-6)
  - [1.4 本项目的 MVC 实现方案](#section-7)
- [二、C++ 基类创建（5.2 核心）](#section-8)
  - [2.1 UAuraUserWidget — 自定义 Widget 基类](#section-9)
  - [2.2 UAuraWidgetController — 自定义控制器基类](#section-10)
  - [2.3 SetWidgetController 的联动机制](#section-11)
- [三、GlobeProgressBar 蓝图制作详解（5.3 核心）](#section-12)
  - [3.1 蓝图继承架构](#section-13)
  - [3.2 蓝图控件层级结构](#section-14)
  - [3.3 蓝图事件图流程](#section-15)
  - [3.4 蓝图变量与分类](#section-18)
- [四、蓝图流程全景图](#section-19)
  - [4.1 Event Pre Construct 完整执行流程](#section-20)
  - [4.2 蓝图节点详解](#section-21)
- [五、健康球与法力球子类（5.4）](#section-25)
  - [5.1 创建健康球子蓝图](#section-26)
  - [5.2 创建法力球子蓝图](#section-27)
  - [5.3 覆盖层（Overlay）组装](#section-28)
- [六、新增文件清单](#section-29)
- [七、知识点总结](#section-30)

---

<a id="section-3"></a>
## 一、MVC 架构设计（5.1 核心）

<a id="section-4"></a>
### 1.1 为什么需要 MVC

在游戏中，UI 需要显示大量数据：生命值、法力值、等级、能力图标等。这些数据分散在属性集、玩家状态、ASC 等多个类中。

**错误做法**：Widget 直接深入游戏代码获取各种指针和引用。

**问题**：

- 随着游戏规模增长，依赖关系变得混乱
- 硬编码依赖导致系统僵硬、难以维护
- Widget 既管数据获取又管视觉显示，职责不清

<a id="section-5"></a>
### 1.2 三层职责划分

```
┌─────────────────────────────────────────────────────┐
│                     MVC 架构                         │
├──────────────┬──────────────────┬───────────────────┤
│   Model(模型) │ Controller(控制器) │   View(视图)     │
├──────────────┼──────────────────┼───────────────────┤
│ 游戏数据      │ 获取数据+广播     │ 视觉显示          │
│ 属性集/ASC   │ 处理计算+算法     │ 生命条/图标       │
│ 玩家状态      │ 转发按钮点击      │ 所有Widget        │
│ 能力数据      │ 模型↔视图中介    │ 玩家看到的一切     │
└──────────────┴──────────────────┴───────────────────┘
```

- **Model（模型）**：底层数据 —— 属性集、ASC、玩家状态等
- **Controller（控制器）**：中介层 —— 从模型获取数据，广播给视图；处理视图传来的按钮点击，修改模型
- **View（视图）**：视觉层 —— 接收控制器广播的数据，专注于"如何显示"

> ⚠️ 这里的 "Controller" 不是 UE 引擎的 `APlayerController`，而是一个独立的 **WidgetController** 类。

<a id="section-6"></a>
### 1.3 单向依赖关系

```
View ──依赖──▶ Controller ──依赖──▶ Model
```

- **Model 不关心**谁在显示它的数据
- **Controller 不关心**有哪些 Widget 在接收广播
- **View 只知道**自己的 Controller 是谁

**好处**：

- 可以更换 Controller 而无需改 Model
- 可以更换 Widget 而无需改 Controller
- 高度模块化，易于扩展和维护

<a id="section-7"></a>
### 1.4 本项目的 MVC 实现方案

| 层级         | UE 中的实现                                                                          |
| ---------- | -------------------------------------------------------------------------------- |
| Model      | `UAuraAttributeSet`、`UAbilitySystemComponent`、`APlayerState`、`APlayerController` |
| Controller | `UAuraWidgetController`（自定义 UObject）                                             |
| View       | `UAuraUserWidget` 及其蓝图子类                                                         |

---

<a id="section-8"></a>
## 二、C++ 基类创建（5.2 核心）

<a id="section-9"></a>
### 2.1 UAuraUserWidget — 自定义 Widget 基类

**文件**：`Source/Aura/Public/UI/Widget/AuraUserWidget.h`

```cpp
UCLASS()
class AURA_API UAuraUserWidget : public UUserWidget
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintCallable)
    void SetWidgetController(UObject* InWidgetController);

    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<UObject> WidgetController;  // 存储控制器指针

protected:
    UFUNCTION(BlueprintImplementableEvent)
    void WidgetControllerSet();  // 蓝图可实现事件
};
```

**关键设计点**：

| 成员                      | 说明                                              |
| ----------------------- | ----------------------------------------------- |
| `WidgetController`      | `UObject*` 类型，通用性强，任何 UObject 都可以作为控制器          |
| `BlueprintReadOnly`     | 蓝图只能读取，不能直接设置（通过 SetWidgetController 设置）        |
| `WidgetControllerSet()` | `BlueprintImplementableEvent`，在蓝图中实现，控制器设置后自动触发 |

**SetWidgetController 实现**（`.cpp`）：

```cpp
void UAuraUserWidget::SetWidgetController(UObject* InWidgetController)
{
    WidgetController = InWidgetController;
    WidgetControllerSet();  // 触发蓝图事件
}
```

<a id="section-10"></a>
### 2.2 UAuraWidgetController — 自定义控制器基类

**文件**：`Source/Aura/Public/UI/WidgetController/AuraWidgetController.h`

```cpp
UCLASS()
class AURA_API UAuraWidgetController : public UObject
{
    GENERATED_BODY()
protected:
    UPROPERTY(BlueprintReadOnly, Category="WidgetController")
    TObjectPtr<APlayerController> PlayerController;

    UPROPERTY(BlueprintReadOnly, Category="WidgetController")
    TObjectPtr<APlayerState> PlayerState;

    UPROPERTY(BlueprintReadOnly, Category="WidgetController")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    UPROPERTY(BlueprintReadOnly, Category="WidgetController")
    TObjectPtr<UAttributeSet> AttributeSet;
};
```

**四个核心变量**：WidgetController 通过这 4 个变量从 Model 层获取数据，然后广播给 View 层。

| 变量                       | 作用                 |
| ------------------------ | ------------------ |
| `PlayerController`       | 获取玩家输入相关数据         |
| `PlayerState`            | 获取玩家状态数据（等级、经验等）   |
| `AbilitySystemComponent` | 获取能力相关数据、监听 ASC 委托 |
| `AttributeSet`           | 获取属性值（生命、法力等）      |

> 这 4 个变量都是 `protected` + `BlueprintReadOnly`，子类可以访问但不能在蓝图中直接修改。

<a id="section-11"></a>
### 2.3 SetWidgetController 的联动机制

```
蓝图调用 SetWidgetController(控制器对象)
        │
        ▼
C++ SetWidgetController() 执行
        │
        ├── WidgetController = InWidgetController  // 存储引用
        │
        └── WidgetControllerSet()  // 触发蓝图事件
                │
                ▼
        蓝图中实现 WidgetControllerSet 事件
        （在这里初始化 UI、绑定数据等）
```

---

<a id="section-12"></a>
## 三、GlobeProgressBar 蓝图制作详解（5.3 核心）

<a id="section-13"></a>
### 3.1 蓝图继承架构

```
UAuraUserWidget (C++ 基类)
    │
    └── WBP_GlobeProgressBar (蓝图基类，球体进度条模板)
            │
            ├── WBP_HealthGlobe (健康球，红色)
            │
            └── WBP_ManaGlobe (法力球，蓝色)
```

<a id="section-14"></a>
### 3.2 蓝图控件层级结构

```
Canvas Panel (仅在 Overlay 层使用)
  └── SizeBox "SizeBox_Root"          ← 控制整体尺寸
        └── Overlay "Overlay_Root"     ← 堆叠所有层
              ├── Image "Image_Background"   ← 环形边框背景
              ├── ProgressBar "ProgressBar_Globe"  ← 球体填充（进度条）
              └── Image "Image_Glass"        ← 玻璃反光效果
```

**各控件作用**：

| 控件                | 类型          | 作用                                    |
| ----------------- | ----------- | ------------------------------------- |
| SizeBox_Root      | Size Box    | 控制整体宽高，通过变量 `BoxWidth`/`BoxHeight` 驱动 |
| Overlay_Root      | Overlay     | 将背景环、进度球、玻璃效果叠在一起                     |
| Image_Background  | Image       | 显示环形边框纹理（如 `GlobeRing`）               |
| ProgressBar_Globe | ProgressBar | 核心！填充方向从下到上，材质为健康球/法力球                |
| Image_Glass       | Image       | 空球体玻璃材质，制造 3D 玻璃球反光效果                 |

<a id="section-15"></a>
### 3.3 蓝图事件图流程

#### 3.3.1 Event Pre Construct（预构建事件）

> 类似于 Actor 的 Construction Script，在编辑器中修改变量时触发。

```
Event Pre Construct
        │
        ├──▶ Update Box Size        ← 设置 SizeBox 的宽高
        │
        ├──▶ Update Background Brush ← 设置背景环形图片
        │
        ├──▶ Update Globe Image     ← 设置进度条的填充材质
        │
        ├──▶ Update Globe Padding   ← 设置进度条的边距
        │
        ├──▶ Update Glass Brush     ← 设置玻璃反光材质
        │
        └──▶ Update Glass Padding   ← 设置玻璃的边距
```

#### 3.3.2 各 Update 函数详解

**① Update Box Size**

```
SizeBox_Root (引用)
    ├── Set Width Override  ← BoxWidth 变量（默认 250）
    └── Set Height Override ← BoxHeight 变量（默认 250）
```

**② Update Background Brush**

```
Image_Background (引用)
    └── Set Brush  ← BackgroundBrush 变量（SlateBrush 类型，默认 GlobeRing 纹理）
```

**③ Update Globe Image（核心）**

```
ProgressBar_Globe (引用)
    └── Set Style  ← 构建 ProgressBarStyle 结构
            ├── BackgroundImage
            │     └── SlateBrush 结构
            │           └── TintColor (Alpha=0)  ← 背景完全透明
            │
            └── FillImage
                  └── ProgressBarFillImage 变量（默认 MI_HealthGlobe 材质）
```

**关键**：进度条背景设为 Alpha=0 透明，这样只看到填充部分和环形边框。

**④ Update Globe Padding**

```
ProgressBar_Globe (引用)
    └── Slot as Overlay Slot
          └── Set Padding
                └── Make Margin (Left/Top/Right/Bottom = GlobePadding 变量，默认 10)
```

**⑤ Update Glass Brush**

```
Image_Glass (引用)
    └── Set Brush  ← GlassBrush 变量（默认 MI_EmptyGlobe 材质）
```

**⑥ Update Glass Padding**

```
Image_Glass (引用)
    └── Slot as Overlay Slot
          └── Set Padding
                └── Make Margin (全部 = GlobePadding 变量，默认 10)
```

<a id="section-18"></a>
### 3.4 蓝图变量与分类

所有可被子类覆盖的变量放在 **Globe Properties** 分类中：

| 变量名                  | 类型         | 默认值            | 说明             |
| -------------------- | ---------- | -------------- | -------------- |
| BoxWidth             | Float      | 250            | SizeBox 宽度（像素） |
| BoxHeight            | Float      | 250            | SizeBox 高度（像素） |
| BackgroundBrush      | SlateBrush | GlobeRing      | 环形边框纹理         |
| ProgressBarFillImage | SlateBrush | MI_HealthGlobe | 进度条填充材质        |
| GlobePadding         | Float      | 10             | 进度条和玻璃的边距      |
| GlassBrush           | SlateBrush | MI_EmptyGlobe  | 玻璃反光材质         |

---

<a id="section-19"></a>
## 四、蓝图流程全景图

<a id="section-20"></a>
### 4.1 Event Pre Construct 完整执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Event Pre Construct                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐    ┌──────────────────┐                    │
│  │ Update Box Size  │    │ 读取 BoxWidth     │                    │
│  │                  │◀───│ 读取 BoxHeight    │                    │
│  │ SizeBox_Root     │    └──────────────────┘                    │
│  │ .SetWidthOverride│                                           │
│  │ .SetHeightOverride│                                          │
│  └────────┬────────┘                                            │
│           │                                                      │
│  ┌────────▼────────┐    ┌──────────────────┐                    │
│  │Update Background│    │ BackgroundBrush   │                    │
│  │     Brush       │◀───│ (SlateBrush变量)  │                    │
│  │ Image_Background│    └──────────────────┘                    │
│  │ .SetBrush()     │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│  ┌────────▼────────┐    ┌──────────────────┐                    │
│  │ Update Globe    │    │ ProgressBarFill  │                    │
│  │     Image       │◀───│ Image (SlateBrush)│                   │
│  │ ProgressBar_Globe│   └──────────────────┘                    │
│  │ .SetStyle()     │                                            │
│  │  ├ Background:  │    背景 Alpha=0（透明）                     │
│  │  └ FillImage:   │    使用 ProgressBarFillImage 变量           │
│  └────────┬────────┘                                            │
│           │                                                      │
│  ┌────────▼────────┐    ┌──────────────────┐                    │
│  │ Update Globe    │    │ GlobePadding     │                    │
│  │    Padding      │◀───│ (Float变量,=10)  │                    │
│  │ ProgressBar_Globe│   └──────────────────┘                    │
│  │ Slot→SetPadding │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│  ┌────────▼────────┐    ┌──────────────────┐                    │
│  │ Update Glass    │    │ GlassBrush       │                    │
│  │    Brush        │◀───│ (SlateBrush变量)  │                    │
│  │ Image_Glass     │    └──────────────────┘                    │
│  │ .SetBrush()     │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│  ┌────────▼────────┐    ┌──────────────────┐                    │
│  │ Update Glass    │    │ GlobePadding     │                    │
│  │    Padding      │◀───│ (复用同一个变量)  │                    │
│  │ Image_Glass     │    └──────────────────┘                    │
│  │ Slot→SetPadding │                                            │
│  └─────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

<a id="section-21"></a>
### 4.2 蓝图节点详解

#### 关键节点 1：Set Style（进度条样式设置）

```
┌──────────────────────────────────────────┐
│           Set Style (ProgressBar)         │
├──────────────────────────────────────────┤
│  Style                                    │
│   ├── BackgroundImage                     │
│   │     └── Make SlateBrush               │
│   │           └── TintColor               │
│   │                 └── Make SlateColor   │
│   │                       └── Alpha = 0  │
│   │                                       │
│   └── FillImage                           │
│         └── ProgressBarFillImage 变量     │
│              (MI_HealthGlobe / MI_ManaGlobe)│
└──────────────────────────────────────────┘
```

#### 关键节点 2：Slot as Overlay Slot → Set Padding

```
ProgressBar_Globe (或 Image_Glass)
    └── 拖出 "Slot"
          └── 选择 "Slot as Overlay Slot"
                └── 拖出 "Set Padding"
                      └── Make Margin 结构
                            ├── Left   = GlobePadding
                            ├── Top    = GlobePadding
                            ├── Right  = GlobePadding
                            └── Bottom = GlobePadding
```

#### 关键节点 3：ProgressBar 绘制模式设置

在 Designer 中设置 `ProgressBar_Globe` 的属性：

| 属性                      | 值             | 说明              |
| ----------------------- | ------------- | --------------- |
| Fill Type               | Bottom to Top | 从下往上填充          |
| Fill Image → Draw As    | Image         | 以图像方式绘制（而非 Box） |
| Fill Color and Opacity  | 白色 (1,1,1,1)  | 保持材质原始颜色        |
| Background Image → Tint | Alpha=0       | 背景完全透明          |

---

<a id="section-25"></a>
## 五、健康球与法力球子类（5.4）

<a id="section-26"></a>
### 5.1 创建健康球子蓝图

**路径**：`Content/Blueprints/UI/ProgressBar/WBP_HealthGlobe`

**父类**：`WBP_GlobeProgressBar`

**操作步骤**：

1. 右键 → 新建 Widget Blueprint → 选择 `WBP_GlobeProgressBar` 作为父类
2. 打开蓝图 → 点击齿轮图标 ⚙ → **Show Inherited Variables**（显示继承变量）
3. 展开 **Globe Properties** 分类
4. 修改 `ProgressBarFillImage` → 选择 `MI_HealthGlobe`（红色健康球材质）
5. 编译 → Designer 自动更新显示

> 修改继承变量会自动触发父类的 Event Pre Construct，所以 Designer 中立即看到变化。

<a id="section-27"></a>
### 5.2 创建法力球子蓝图

**路径**：`Content/Blueprints/UI/ProgressBar/WBP_ManaGlobe`

**父类**：`WBP_GlobeProgressBar`

**操作步骤**：

1. 同上创建，父类选 `WBP_GlobeProgressBar`
2. `ProgressBarFillImage` 默认已是 `MI_ManaGlobe`（蓝色），无需修改

<a id="section-28"></a>
### 5.3 覆盖层（Overlay）组装

**路径**：`Content/Blueprints/UI/Overlay/WBP_Overlay`

**父类**：`UAuraUserWidget`（C++ 基类）

**控件层级**：

```
WBP_Overlay
  └── Canvas Panel                    ← 唯一使用 Canvas Panel 的地方
        ├── WBP_HealthGlobe           ← 锚点：底部中间
        │     Position: 底部中间偏左
        │
        └── WBP_ManaGlobe             ← 锚点：底部中间
              Position: 底部中间偏右（与 HealthGlobe 同 Y 坐标）
```

**为什么 Overlay 用 Canvas Panel？**

- Canvas Panel 允许手动拖动定位
- 只有一个 Overlay 层使用，性能影响可忽略
- 子 Widget（GlobeProgressBar 内部）使用更高效的 SizeBox + Overlay

**临时测试**（关卡蓝图中）：

```
Event BeginPlay
    ├── Create Widget (Class: WBP_Overlay)
    └── Add to Viewport
          └── 返回值连接
```

---

<a id="section-29"></a>
## 六、新增文件清单

| 文件                                                                 | 说明                     |
| ------------------------------------------------------------------ | ---------------------- |
| `Source/Aura/Public/UI/Widget/AuraUserWidget.h`                    | 自定义 Widget 基类头文件       |
| `Source/Aura/Private/UI/Widget/AuraUserWidget.cpp`                 | 自定义 Widget 基类实现        |
| `Source/Aura/Public/UI/WidgetController/AuraWidgetController.h`    | WidgetController 基类头文件 |
| `Source/Aura/Private/UI/WidgetController/AuraWidgetController.cpp` | WidgetController 基类实现  |
| `Content/Blueprints/UI/ProgressBar/WBP_GlobeProgressBar.uasset`    | 球体进度条基类蓝图              |
| `Content/Blueprints/UI/ProgressBar/WBP_HealthGlobe.uasset`         | 健康球蓝图                  |
| `Content/Blueprints/UI/ProgressBar/WBP_ManaGlobe.uasset`           | 法力球蓝图                  |
| `Content/Blueprints/UI/Overlay/WBP_Overlay.uasset`                 | 覆盖层蓝图                  |

---

<a id="section-30"></a>
## 七、知识点总结

| 知识点                                    | 说明                                               |
| -------------------------------------- | ------------------------------------------------ |
| **MVC 架构**                             | Model(数据) → Controller(中介) → View(显示)，单向依赖       |
| **WidgetController**                   | 不是 APlayerController，是自定义 UObject，负责数据获取和广播      |
| **BlueprintImplementableEvent**        | C++ 声明，蓝图中实现；`WidgetControllerSet()` 在控制器设置后自动触发 |
| **Event Pre Construct**                | Widget 蓝图的"构建脚本"，修改变量时自动触发，用于初始化视觉               |
| **SizeBox + Overlay**                  | 比 Canvas Panel 更高效的布局方式，子 Widget 内部推荐使用          |
| **ProgressBar Draw As**                | 设为 Image（而非 Box）才能正确显示球体材质                       |
| **Slot as Overlay Slot → Set Padding** | 在 Overlay 中设置子控件边距的标准方式                          |
| **Show Inherited Variables**           | 蓝图齿轮菜单中的选项，显示父类变量以便覆盖                            |
| **SlateBrush 变量**                      | 用于存储纹理/材质引用，可在子蓝图中覆盖                             |
| **Make Margin 结构**                     | 用于设置 Padding，包含 Left/Top/Right/Bottom 四个值        |

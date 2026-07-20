# UE5 学习笔记 — 属性菜单（AttributeMenu）专题

> 🎬 对应视频：第9章 9.1 ~ 9.10  
> 📅 日期：2026-07-18  
> 🏷️ 专题：蓝图操作 — 属性菜单 UI 系统

---

## 目录

- [UE5 学习笔记 — 属性菜单（AttributeMenu）专题](#ue5-学习笔记--属性菜单attributemenu专题)
  - [目录](#目录)
  - [一、整体设计与规划（9.1）](#一整体设计与规划91)
    - [1.1 属性菜单的视觉布局](#11-属性菜单的视觉布局)
    - [1.2 组件拆分策略](#12-组件拆分策略)
    - [1.3 主要属性 vs 次要属性的设计差异](#13-主要属性-vs-次要属性的设计差异)
  - [二、WBP\_FramedValue — 带边框数值组件（9.2）](#二wbp_framedvalue--带边框数值组件92)
    - [2.1 组件结构](#21-组件结构)
    - [2.2 蓝图变量与函数](#22-蓝图变量与函数)
    - [2.3 材质与样式设置](#23-材质与样式设置)
  - [三、WBP\_TextValueRow — 文本值行组件（9.3）](#三wbp_textvaluerow--文本值行组件93)
    - [3.1 组件结构](#31-组件结构)
    - [3.2 水平布局与间隔器](#32-水平布局与间隔器)
    - [3.3 Named Slot — 为子类化预留扩展点](#33-named-slot--为子类化预留扩展点)
  - [四、WBP\_TextValueButtonRow — 文本值按钮行组件（9.4）](#四wbp_textvaluebuttonrow--文本值按钮行组件94)
    - [4.1 继承 WBP\_TextValueRow](#41-继承-wbp_textvaluerow)
    - [4.2 按钮叠加层设计](#42-按钮叠加层设计)
    - [4.3 按钮四种状态样式](#43-按钮四种状态样式)
  - [五、WBP\_AttributeMenu — 属性菜单主面板（9.5）](#五wbp_attributemenu--属性菜单主面板95)
    - [5.1 Wrap Box 布局系统](#51-wrap-box-布局系统)
    - [5.2 标题与副标题](#52-标题与副标题)
    - [5.3 主要属性区域（带按钮行）](#53-主要属性区域带按钮行)
    - [5.4 次要属性区域（Scroll Box 滚动列表）](#54-次要属性区域scroll-box-滚动列表)
    - [5.5 背景与边框美化](#55-背景与边框美化)
  - [六、WBP\_Button — 通用按钮基类（9.6）](#六wbp_button--通用按钮基类96)
    - [6.1 为什么需要按钮基类](#61-为什么需要按钮基类)
    - [6.2 尺寸框 + 覆盖层架构](#62-尺寸框--覆盖层架构)
    - [6.3 参数化设计：画笔 / 字体 / 文本](#63-参数化设计画笔--字体--文本)
    - [6.4 暴露属性的眼睛图标机制](#64-暴露属性的眼睛图标机制)
  - [七、WBP\_WideButton — 宽按钮子类（9.7）](#七wbp_widebutton--宽按钮子类97)
    - [7.1 继承 WBP\_Button 并覆盖默认值](#71-继承-wbp_button-并覆盖默认值)
    - [7.2 替换宽按钮纹理资源](#72-替换宽按钮纹理资源)
    - [7.3 添加到 Overlay 覆盖层](#73-添加到-overlay-覆盖层)
  - [八、打开属性菜单（9.8）](#八打开属性菜单98)
    - [8.1 按钮点击事件绑定](#81-按钮点击事件绑定)
    - [8.2 创建 Widget 并添加到视口](#82-创建-widget-并添加到视口)
    - [8.3 视口定位与内边距调整](#83-视口定位与内边距调整)
    - [8.4 打开时禁用按钮防止重复打开](#84-打开时禁用按钮防止重复打开)
  - [九、关闭属性菜单与事件分发器（9.9）](#九关闭属性菜单与事件分发器99)
    - [9.1 关闭按钮的点击绑定](#91-关闭按钮的点击绑定)
    - [9.2 RemoveFromParent 销毁 Widget](#92-removefromparent-销毁-widget)
    - [9.3 Event Dispatcher 跨蓝图通信](#93-event-dispatcher-跨蓝图通信)
    - [9.4 循环依赖问题与解耦方案](#94-循环依赖问题与解耦方案)
    - [9.5 创建/销毁 vs 显示/隐藏 的取舍](#95-创建销毁-vs-显示隐藏-的取舍)
  - [十、数据架构设计 — 属性信息传递系统（9.10）](#十数据架构设计--属性信息传递系统910)
    - [10.1 蛮力方法 vs 通用委托方法](#101-蛮力方法-vs-通用委托方法)
    - [10.2 基于 GameplayTag 的属性标识方案](#102-基于-gameplaytag-的属性标识方案)
    - [10.3 AttributeInfo DataAsset 设计](#103-attributeinfo-dataasset-设计)
    - [10.4 完整数据流架构图](#104-完整数据流架构图)
    - [10.5 后续待实现步骤](#105-后续待实现步骤)

---

## 一、整体设计与规划（9.1）

### 1.1 属性菜单的视觉布局

属性菜单的整体布局从上到下为：

```
┌──────────────────────────────┐
│         ATTRIBUTES           │  ← 标题
├──────────────────────────────┤
│   Attribute Points: [N]      │  ← 剩余属性点（文本值行）
├──────────────────────────────┤
│      PRIMARY ATTRIBUTES      │  ← 副标题
│   Strength     [10] [+]      │  ← 文本值按钮行（带+按钮）
│   Intelligence  [8] [+]      │
│   Resilience    [7] [+]      │
│   Vigor         [9] [+]      │
├──────────────────────────────┤
│     SECONDARY ATTRIBUTES     │  ← 副标题
│   ┌────────────────────┐     │
│   │ Max Health    [100] │     │  ← 滚动框内
│   │ Max Mana      [80]  │     │     文本值行（无按钮）
│   │ Armor         [25]  │     │
│   │ Crit Chance   [5%]  │     │
│   │ ... (可滚动)  ▐███▌ │     │  ← 滚动条
│   └────────────────────┘     │
├──────────────────────────────┤
│   Health Bar  ████████░░     │  ← 生命值进度条（后续）
│   Mana Bar    ██████░░░░     │  ← 法力值进度条（后续）
│                          [X] │  ← 关闭按钮
└──────────────────────────────┘
```

### 1.2 组件拆分策略

采用**自底向上**的组件化设计，从最小粒度开始构建：

| 层级 | 组件名 | 说明 |
|------|--------|------|
| L1 原子组件 | `WBP_FramedValue` | 带边框的数值显示框 |
| L2 行组件 | `WBP_TextValueRow` | 文本 + FramedValue 的行 |
| L2 行组件 | `WBP_TextValueButtonRow` | 继承 TextValueRow，末尾多一个按钮 |
| L3 面板组件 | `WBP_AttributeMenu` | 组装所有行组件的主面板 |
| 通用组件 | `WBP_Button` / `WBP_WideButton` | 可复用的按钮基类/子类 |

```
WBP_FramedValue ──被包含──▶ WBP_TextValueRow ──被继承──▶ WBP_TextValueButtonRow
                                 │                              │
                                 │                              │
                                 ▼                              ▼
                          WBP_AttributeMenu ◀── 组装 ── WBP_Button/WBP_WideButton
```

### 1.3 主要属性 vs 次要属性的设计差异

| | 主要属性 (Primary) | 次要属性 (Secondary) |
|---|---|---|
| **行类型** | `TextValueButtonRow`（有 + 按钮） | `TextValueRow`（无按钮） |
| **数量** | 4 个（力量/智力/韧性/活力） | 约 10 个（衍生属性） |
| **可否加点** | ✅ 玩家可用属性点提升 | ❌ 由主要属性衍生计算 |
| **显示方式** | 直接排列 | Scroll Box 内滚动显示 |

> **设计理念**：主要属性驱动所有次要属性。玩家升级获得属性点，只能加在主要属性上，次要属性通过 MMC 等机制自动重新计算。

---

## 二、WBP_FramedValue — 带边框数值组件（9.2）

### 2.1 组件结构

这是整个属性菜单的最小原子组件，负责在一个带边框的框内显示数值。

```
SizeBox_Root (尺寸框，控制整体大小)
└── Overlay_Root (覆盖层，元素堆叠)
    ├── Image_Background (背景图 — 流动UI材质)
    ├── Image_Border (边框图 — 绘制为Border)
    └── TextBlock_Value (数值文本 — 居中显示)
```

**层级关系说明**：
- **SizeBox**：最外层，通过 `WidthOverride` / `HeightOverride` 控制组件尺寸
- **Overlay**：让背景、边框、文本三层叠加显示
- **Image_Background**：使用 `MI_FlowingUIBackground` 流动云层材质
- **Image_Border**：使用 `Border_One` 纹理，**绘制方式设为 Border**（而非 Image），通过 Margin 控制边框粗细
- **TextBlock_Value**：显示数值如 `99`，字体 Pirata One，字号约 17，描边 1px

### 2.2 蓝图变量与函数

**变量（分类：Frame Properties）**：

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `BoxWidth` | Float | 80 | 尺寸框宽度 |
| `BoxHeight` | Float | 45 | 尺寸框高度 |
| `BackgroundBrush` | Brush | MI_FlowingUIBackground | 背景画笔（可被子类覆盖） |

**函数**：

| 函数名 | 触发时机 | 功能 |
|--------|----------|------|
| `UpdateFrameSize()` | Event Pre Construct | 将 BoxWidth/BoxHeight 应用到 SizeBox 的 Override |
| `UpdateBackgroundBrush()` | Event Pre Construct | 将 BackgroundBrush 应用到 Image_Background |

**Event Pre Construct 流程**：

```
Event Pre Construct
    ├─ UpdateFrameSize()      → SetWidthOverride(BoxWidth) + SetHeightOverride(BoxHeight)
    └─ UpdateBackgroundBrush() → SetBrush(BackgroundBrush)
```

### 2.3 材质与样式设置

- **流动UI背景材质** (`MI_FlowingUIBackground`)：
  - 可调参数：X/Y 速度（云层平移速度）、云层暗度、云层颜色
  - 可完全去饱和或改为任意色调（蓝/红/黑等）
- **边框绘制**：选择 `Border_One` PNG → Draw As → **Border** → Margin 设为 `0.5` 四面统一
- **边框自适应**：当 SizeBox 尺寸改变时，Border 会自动拉伸而不会明显变形

---

## 三、WBP_TextValueRow — 文本值行组件（9.3）

### 3.1 组件结构

```
SizeBox_Root (尺寸框)
└── HorizontalBox (水平盒子，从左到右排列)
    ├── TextBlock (属性名称，左对齐)
    ├── Spacer (间隔器)
    ├── WBP_FramedValue (带边框数值)
    └── NamedSlot (命名槽位，为子类预留扩展点)
```

### 3.2 水平布局与间隔器

| 元素 | 对齐方式 | 说明 |
|------|----------|------|
| TextBlock | 水平左对齐 + 垂直居中 | 显示属性名称，如 "Strength" |
| Spacer | 尺寸 X=40~50 | 在文本和数值之间制造间距 |
| WBP_FramedValue | 水平右对齐 + 垂直居中 | 显示属性数值 |
| NamedSlot | 紧随其后 | 子类可在此添加按钮等额外控件 |

**文本样式**：
- 字体：Pirata One，大小约 32
- 字间距：约 1.76（Letter Spacing）
- 描边：1px 黑色轮廓

**变量（分类：Row Properties）**：

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `BoxWidth` | Float | 720 | 行宽度 |
| `BoxHeight` | Float | 60 | 行高度 |

> 默认宽度 720 经过多次调整确定，需容纳最长属性名（如 "Crit Resistance"）+ 间隔 + 数值框。

### 3.3 Named Slot — 为子类化预留扩展点

**Named Slot（命名槽位）** 是 UMG 中实现组件继承扩展的关键机制：

- 在父类 `WBP_TextValueRow` 中放置一个 Named Slot
- 子类 `WBP_TextValueButtonRow` 继承后，可以在这个槽位中添加新控件（如按钮）
- 父类的其他控件被子类**继承但不可直接修改**（需通过暴露函数来操作）

```
父类: WBP_TextValueRow
    └── NamedSlot (空槽位)
            │
            │ 继承
            ▼
子类: WBP_TextValueButtonRow
    └── NamedSlot
        └── Overlay (按钮 + 边框)  ← 子类在此添加内容
```

---

## 四、WBP_TextValueButtonRow — 文本值按钮行组件（9.4）

### 4.1 继承 WBP_TextValueRow

创建时选择父类为 `WBP_TextValueRow`（而非默认的 UserWidget），继承其所有控件和逻辑。

### 4.2 按钮叠加层设计

在父类的 Named Slot 中添加：

```
NamedSlot
└── Overlay
    ├── Image_Border (按钮边框 — 40×40)
    ├── Button (按钮本体)
    │   └── TextBlock (加号 "+")
```

- **Image_Border**：使用 `ButtonBorder` 纹理，Draw As = Image，40×40 尺寸
- **Button**：同样 40×40，居中叠加在边框之上
- **加号文本**：居中显示，字体 Rubato 或 Amaranth，描边 1px

### 4.3 按钮四种状态样式

按钮控件需要为四种状态分别设置样式，且都改为 **Draw As = Image**（不使用默认的圆角框）：

| 状态 | 纹理资源 | 说明 |
|------|----------|------|
| **Normal** | `Button` (红色) | 默认状态 |
| **Hovered** | `Button_Highlighted` | 悬停时更亮 |
| **Pressed** | `Button_Pressed` | 按下时稍暗 |
| **Disabled** | `Button_Disabled` (灰色) | 禁用状态 |

> 设置方法：选中 Button → Style → 展开各状态 → Image 选择对应纹理 → Draw As 改为 Image。

---

## 五、WBP_AttributeMenu — 属性菜单主面板（9.5）

### 5.1 Wrap Box 布局系统

**Wrap Box** 是实现从上到下自动排列的关键控件：

```
Wrap Box 的行为规则：
┌──────────────────────────────────────┐
│  [元素A]  [元素B]                     │  ← 同一行
│  [元素C——太宽放不下——]               │  ← 自动换到下一行
│  [元素D]                              │  ← 新行
└──────────────────────────────────────┘
```

- 当元素宽度超过 Wrap Box 可用宽度时，自动换到新行
- **强制换行技巧**：将元素（如标题 TextBlock）宽度设得比 Wrap Box 还大（如 1000），它就会独占一行
- **内边距**：Wrap Box 四边 Padding 设为 25，避免内容贴边

### 5.2 标题与副标题

| 元素 | 字体 | 大小 | 字间距 | 对齐 |
|------|------|------|--------|------|
| 标题 "ATTRIBUTES" | Pirata One | 36 | ~400 | 居中（Fill + Center） |
| 副标题 "PRIMARY ATTRIBUTES" | Pirata One | 20 | ~800 | 居中 |
| 副标题 "SECONDARY ATTRIBUTES" | Pirata One | 20 | ~800 | 居中 |

> 标题通过设置 Fill 空白 + 水平居中实现居中效果。

### 5.3 主要属性区域（带按钮行）

使用 `WBP_TextValueButtonRow` × 4，每个之间用 Spacer（高度约 5~15）分隔：

```
Attribute Points Row  (WBP_TextValueRow, 无按钮)
Spacer
"PRIMARY ATTRIBUTES"  (标题)
Spacer
Strength Row          (WBP_TextValueButtonRow)
Spacer
Intelligence Row      (WBP_TextValueButtonRow)
Spacer
Resilience Row        (WBP_TextValueButtonRow)
Spacer
Vigor Row             (WBP_TextValueButtonRow)
```

> 属性点行放在主要属性上方，用于显示剩余可分配点数。

### 5.4 次要属性区域（Scroll Box 滚动列表）

```
Spacer
"SECONDARY ATTRIBUTES" (标题)
Spacer
SizeBox_Scroll (固定高度，如 ~300px)
└── ScrollBox
    ├── WBP_TextValueRow (Max Health)
    ├── WBP_TextValueRow (Max Mana)
    ├── WBP_TextValueRow (Armor)
    ├── ... (约 10 个次要属性)
    └── WBP_TextValueRow (Crit Resistance)
```

**Scroll Box 关键配置**：
- 外层必须用 **SizeBox** 限制高度，否则 ScrollBox 会无限延伸
- SizeBox 设置固定 Height Override（如 ~300px）
- 当内容超出 SizeBox 高度时，右侧自动出现滚动条

### 5.5 背景与边框美化

```
Overlay_Root
├── Image_Background (流动UI材质，内边距 5~25)
├── Image_Border (边框大图，Draw As = Border，Margin = 0.5)
└── WrapBox (所有内容)
```

- **Image_Background**：使用 `MI_FlowingUIBackground`，Padding 设为 25 避免内容超出边框圆角
- **Image_Border**：使用 `Border_Large` 纹理，Draw As = Border，Margin 0.5×4
- 整体 SizeBox 尺寸约：**805 × 960**

---

## 六、WBP_Button — 通用按钮基类（9.6）

### 6.1 为什么需要按钮基类

属性菜单中多处需要按钮：关闭按钮、加号按钮、属性菜单入口按钮。与其每次都重新搭建按钮结构，不如创建一个**可参数化的通用按钮基类**，通过继承和覆盖默认值来复用。

### 6.2 尺寸框 + 覆盖层架构

```
SizeBox_Root (40×40 默认)
└── Overlay_Root
    ├── Image_Border (按钮边框)
    ├── Button (按钮本体，Fill 填充)
    └── TextBlock (按钮文字，居中)
```

### 6.3 参数化设计：画笔 / 字体 / 文本

**变量清单（分类：Button Properties）**：

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `BoxWidth` | Float | 40 | 按钮宽度 |
| `BoxHeight` | Float | 40 | 按钮高度 |
| `BorderBrush` | Brush | ButtonBorder | 边框画笔 |
| `ButtonNormalBrush` | Brush | Button (红色) | 正常状态画笔 |
| `ButtonHoveredBrush` | Brush | Button_Highlighted | 悬停状态画笔 |
| `ButtonPressedBrush` | Brush | Button_Pressed | 按下状态画笔 |
| `ButtonDisabledBrush` | Brush | Button_Disabled | 禁用状态画笔 |
| `ButtonText` | Text | "X" | 按钮文本 |
| `FontFamily` | Font Family | Roboto | 字体家族 |
| `FontSize` | Int | 16 | 字体大小 |
| `OutlineSize` | Int | 1 | 描边大小 |
| `LetterSpacing` | Float | 0 | 字间距 |

**函数**：

| 函数名 | 功能 |
|--------|------|
| `UpdateBoxSize()` | 应用 BoxWidth/BoxHeight 到 SizeBox |
| `UpdateBorderBrush()` | 应用 BorderBrush 到 Image_Border |
| `UpdateButtonBrushes()` | 应用四种状态画笔到 Button Style |
| `UpdateText()` | 应用 ButtonText + 字体设置到 TextBlock |

**Event Pre Construct 流程**：

```
Event Pre Construct
    ├─ UpdateBoxSize()
    ├─ UpdateBorderBrush()
    ├─ UpdateButtonBrushes()
    └─ UpdateText()
```

### 6.4 暴露属性的眼睛图标机制

在变量面板中点击变量旁的**眼睛图标（Expose）**，使变量在子类或实例的 Details 面板中可编辑：

- 编译后，在任意使用 WBP_Button 的地方，Details 面板会出现 **Button Properties** 分类
- 可以直接修改按钮文本（如 "+"）、字体大小、画笔等

> ⚠️ 操作前务必**保存所有文件**，防止编辑器崩溃。

---

## 七、WBP_WideButton — 宽按钮子类（9.7）

### 7.1 继承 WBP_Button 并覆盖默认值

创建时选择父类为 `WBP_Button`，然后在子类中覆盖默认变量值：

| 变量 | 父类默认值 | 子类覆盖值 |
|------|-----------|-----------|
| `BoxWidth` | 40 | 200 |
| `BoxHeight` | 40 | 60~70 |
| `ButtonText` | "X" | "ATTRIBUTES" |
| `FontFamily` | Roboto | Pirata One |
| `FontSize` | 16 | 22 |
| `LetterSpacing` | 0 | 200 |

### 7.2 替换宽按钮纹理资源

宽按钮纹理自带边框，因此：
- **BorderBrush**：设为 Clear（透明度拖到 0），移除额外边框
- **四种状态画笔**：全部替换为宽按钮版本
  - Normal → `WideButton_Red`
  - Hovered → `WideButton_Red_Highlighted`
  - Pressed → `WideButton_Red_Pressed`
  - Disabled → `WideButton_Grey`

> 资源路径：`Assets/UI/Button_Red/WideButton_Red/`

### 7.3 添加到 Overlay 覆盖层

在 `WBP_Overlay` 中拖入 `WBP_WideButton`，设置按钮文本为 "ATTRIBUTES"，作为打开属性菜单的入口按钮。

**预览测试**：点击 Play 可在视口中看到按钮，悬停/点击/禁用状态均正常切换。

---

## 八、打开属性菜单（9.8）

### 8.1 按钮点击事件绑定

在 `WBP_Overlay` 的 **Event Construct** 中绑定：

```
Event Construct
    │
    ├─ 获取 AttributeMenuButton
    │   └─ 获取 Button (内部按钮控件)
    │       └─ Assign OnClicked → 自定义事件 "AttributeMenuButtonClicked"
    │
    └─ AttributeMenuButtonClicked (自定义事件)
        ├─ 禁用按钮 (Set Enabled = false)
        └─ 创建并显示属性菜单
```

**关键点**：`WBP_WideButton` 继承自 `WBP_Button`，其内部按钮控件名为 `Button`。需要通过 `Get Button` 节点获取内部按钮后才能绑定点击事件。

### 8.2 创建 Widget 并添加到视口

```
Create Widget
    ├─ Class: WBP_AttributeMenu
    └─ Owning Player: Get Player Controller (Index 0)
        │
        ▼
    Add to Viewport
        │
        ▼
    Set Viewport Position (X=50, Y=50)
```

> `Get Player Controller (Index 0)` 是 GameplayStatics 函数，获取本地玩家控制器。

### 8.3 视口定位与内边距调整

如果直接 Add to Viewport，Widget 会**填满整个视口**。解决方案：

- 在 `WBP_AttributeMenu` 中，将 `SizeBox_Root` **包裹一层 Overlay**
- 外层 Overlay 作为根，保持 Widget 在视口中的正确尺寸

```
修复前：SizeBox_Root → 填满视口
修复后：Overlay (新根) → SizeBox_Root → 保持 805×960 尺寸
```

视口位置设为 (50, 50) 可产生偏移效果，避免贴边。

### 8.4 打开时禁用按钮防止重复打开

点击打开后立即调用 `Set Enabled = false`，防止玩家在菜单已打开时再次点击。

---

## 九、关闭属性菜单与事件分发器（9.9）

### 9.1 关闭按钮的点击绑定

在 `WBP_AttributeMenu` 的 Event Construct 中：

```
Event Construct
    │
    └─ 获取 CloseButton
        └─ 获取 Button (内部按钮控件)
            └─ Assign OnClicked → Remove from Parent
```

### 9.2 RemoveFromParent 销毁 Widget

`Remove from Parent` 节点将 Widget 从视口中移除并销毁。

### 9.3 Event Dispatcher 跨蓝图通信

**问题**：属性菜单关闭后，Overlay 中的 "ATTRIBUTES" 按钮仍然是禁用状态，无法再次打开。

**解决**：使用 **Event Dispatcher（事件分发器）**——蓝图层面的委托机制。

在 `WBP_AttributeMenu` 中：

```
1. 创建 Event Dispatcher：AttributeMenuClosed

2. 在 Widget 被销毁时广播：
   Event Destruct
       └─ Call AttributeMenuClosed
```

在 `WBP_Overlay` 中：

```
3. 创建 Widget 后订阅：
   Create Widget → 获取 AttributeMenu
       └─ Bind Event to AttributeMenuClosed
           └─ 重新启用按钮 (Set Enabled = true)
```

### 9.4 循环依赖问题与解耦方案

| 方案 | 描述 | 问题 |
|------|------|------|
| ❌ 直接引用 | Overlay 持有 AttributeMenu 引用，AttributeMenu 也持有 Overlay 引用 | 循环依赖，紧密耦合 |
| ✅ Event Dispatcher | AttributeMenu 只广播事件，不关心谁在监听 | 解耦，AttributeMenu 独立于 Overlay |

> **设计原则**：AttributeMenu 不依赖 Overlay。它只负责在关闭时广播 `AttributeMenuClosed` 事件，由外部（Overlay）自行决定如何响应。

### 9.5 创建/销毁 vs 显示/隐藏 的取舍

| 方案 | 优点 | 缺点 |
|------|------|------|
| **创建/销毁**（本课程采用） | 不占用内存、不响应后台回调 | 每次打开有微小创建开销 |
| **显示/隐藏** | 切换更快 | Widget 一直在内存中，可能响应不需要的回调 |

> 两种方式都可以，动态创建 Widget 性能开销不大。本课程选择创建/销毁方式。

---

## 十、数据架构设计 — 属性信息传递系统（9.10）

### 10.1 蛮力方法 vs 通用委托方法

**蛮力方法（不推荐）**：

```
ASC 广播 Strength 变化
    → WidgetController 接收
    → WidgetController 广播 OnStrengthChanged 委托
    → Strength Row Widget 绑定并更新

ASC 广播 Intelligence 变化
    → WidgetController 接收
    → WidgetController 广播 OnIntelligenceChanged 委托
    → Intelligence Row Widget 绑定并更新

... 每个属性都要重复此模式
```

**问题**：每添加一个新属性，就需要：
1. 在 WidgetController 中订阅 ASC 委托
2. 声明新的广播委托
3. 在 Widget 端绑定新委托

维护和扩展极其困难(不符合开闭原则)。

**通用委托方法（推荐）**：

```
ASC 广播任意属性变化
    → WidgetController 接收
    → 确定属性关联的 GameplayTag
    → 从 DataAsset 查询属性信息结构体
    → 广播统一的 "AttributeChanged" 委托（携带结构体）
    → 所有行 Widget 接收
    → 各行检查结构体中的 Tag 是否匹配自身 Tag
    → 匹配则更新显示
```

### 10.2 基于 GameplayTag 的属性标识方案

每个属性关联一个唯一的 **GameplayTag**：

| 属性 | GameplayTag |
|------|-------------|
| Strength | `Attributes.Primary.Strength` |
| Max Health | `Attributes.Secondary.MaxHealth` |
| Armor | `Attributes.Secondary.Armor` |
| ... | ... |

**为什么用 GameplayTag**：
- 层级结构清晰（`Attributes.Primary.X` vs `Attributes.Secondary.X`）
- 可在 C++ 和蓝图中统一使用
- 支持 `MatchesTag()` 等层级匹配查询

### 10.3 AttributeInfo DataAsset 设计

创建一个 **DataAsset** 类 `UAuraAttributeInfo`：

- 存储一个 **TMap `<FGameplayTag, FAuraAttributeInfo>`** 的映射表
- 提供函数：`FindAttributeInfoForTag(FGameplayTag Tag)` → 返回 `FAuraAttributeInfo` 结构体

**FAuraAttributeInfo 结构体**包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `AttributeTag` | FGameplayTag | 属性对应的标签 |
| `AttributeName` | FText | 显示名称（如 "Strength"） |
| `AttributeDescription` | FText | 悬停提示描述 |
| `AttributeValue` | Float | 当前数值 |

### 10.4 完整数据流架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      MODEL (数据层)                          │
│  ┌──────────────┐    ┌──────────────────┐                   │
│  │ AttributeSet  │    │ AttributeInfo    │                   │
│  │ (属性数值)    │    │ DataAsset        │                   │
│  │              │    │ (Tag→名称/描述)  │                   │
│  └──────┬───────┘    └────────┬─────────┘                   │
│         │ 属性变化委托         │ 查询                         │
├─────────┼─────────────────────┼─────────────────────────────┤
│         ▼                     ▼                             │
│  ┌──────────────────────────────────────────┐               │
│  │    CONTROLLER (AttributeMenuWidgetController)            │
│  │                                                          │
│  │  1. 订阅 ASC 的属性变化委托                               │
│  │  2. 确定变化属性对应的 GameplayTag                        │
│  │  3. 从 DataAsset 查询 FAuraAttributeInfo                 │
│  │  4. 广播 AttributeChanged(FAuraAttributeInfo)            │
│  └──────────────────────┬───────────────────┘               │
├─────────────────────────┼───────────────────────────────────┤
│                         ▼                                   │
│  ┌──────────────────────────────────────────┐               │
│  │         VIEW (Widget 层)                  │               │
│  │                                           │               │
│  │  WBP_AttributeMenu                        │               │
│  │    ├─ WBP_TextValueButtonRow (Tag: Strength)              │
│  │    │    └─ 接收广播 → 匹配 Tag → 更新显示                 │
│  │    ├─ WBP_TextValueButtonRow (Tag: Intelligence)          │
│  │    ├─ WBP_TextValueRow (Tag: MaxHealth)                  │
│  │    └─ WBP_TextValueRow (Tag: Armor)                      │
│  └──────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

**信息流步骤**：

```
① AttributeSet 中某属性值变化
② ASC 广播属性变化委托
③ WidgetController 回调接收 → 确定对应的 GameplayTag
④ WidgetController 调用 DataAsset.FindAttributeInfoForTag(Tag)
⑤ 获得 FAuraAttributeInfo 结构体
⑥ WidgetController 广播统一的 AttributeChanged 委托
⑦ 所有行 Widget 收到广播
⑧ 各行检查结构体中的 Tag == 自身 Tag？
⑨ 匹配的行从结构体提取数值/名称，更新 UI
```

### 10.5 后续待实现步骤

根据 9.10 的规划，后续需要完成：

| 步骤 | 内容 | 说明 |
|------|------|------|
| ① | 创建次要属性的 GameplayTag | 在 GameplayTagManager 中定义 |
| ② | 创建集中式 GameplayTag 变量类 | C++ 中统一管理 Tag 引用，避免到处 `RequestGameplayTag(FName)` |
| ③ | 创建 `FAuraAttributeInfo` 结构体 | 包含 Tag、Name、Description、Value 等字段 |
| ④ | 创建 `UAuraAttributeInfo` DataAsset | TMap 存储 + 按 Tag 查询函数 |
| ⑤ | 填充 DataAsset | 为每个属性创建对应的 Info 条目 |
| ⑥ | 创建 `UAttributeMenuWidgetController` | 继承 `UAuraWidgetController`，整合 ASC 委托订阅 + DataAsset 查询 + 广播 |
| ⑦ | 行 Widget 绑定通用委托 | 各行分配 Tag，接收广播后自行匹配更新 |

---

> 📝 **总结**：本章完成了属性菜单的完整 UI 搭建——从最底层的 `FramedValue` 原子组件，到中层的行组件（通过 Named Slot 实现继承扩展），再到顶层面板的 Wrap Box + Scroll Box 布局，以及通用按钮基类的参数化设计。最后通过 Event Dispatcher 实现了打开/关闭的解耦通信，并规划了基于 GameplayTag + DataAsset 的数据驱动架构，为后续属性值动态绑定打下基础。

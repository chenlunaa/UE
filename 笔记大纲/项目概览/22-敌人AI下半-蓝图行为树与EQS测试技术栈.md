# UE5 学习笔记 - 第二十二次提交（第15章下半：蓝图行为树、EQS 与 AI 测试技术栈）

> Commit `306cf95`：15章下半，实现AILogicTree的基本逻辑  
> 日期：2026-08-15  
> 对应视频：15.8～15.14  
> 本篇重点：蓝图操作、行为树分支落地、环境查询系统（EQS）以及可重复的 AI 测试方法

本篇承接[第 15 章前半笔记](</D:/学习/UE/笔记大纲/项目概览/21-敌人AI前半-行为树黑板与最近目标追踪.md>)。前半已经完成 `AIController`、`Blackboard`、最近目标 Service 和基础移动；下半把这些“框架节点”真正组合成近战和远程行为，并通过独立测试地图验证导航、视线和障碍物场景。

---

## 目录

- [一、下半章完成了什么](#section-1)
- [二、蓝图任务的 C++ 基类与执行协议](#section-2)
- [三、创建 BT_Attack 蓝图任务](#section-3)
- [四、近战攻击者：攻击后绕目标换位](#section-4)
- [五、BT_T_BypassTarget 蓝图任务的详细连线](#section-5)
- [六、时间限制与行为树防卡死](#section-6)
- [七、EQS 的思想与运行管线](#section-7)
- [八、EQS 测试资源与独立测试地图](#section-8)
- [九、EQS 生成器：从候选物品网格开始](#section-9)
- [十、Trace 测试：筛选可看见玩家的位置](#section-10)
- [十一、距离测试与评分因子](#section-11)
- [十二、远程攻击分支的蓝图操作](#section-12)
- [十三、完整 AI 行为树流程](#section-13)
- [十四、本章 AI 测试技术栈](#section-14)
- [十五、实际测试中发现的问题与定位方法](#section-15)
- [十六、本次提交文件与资产边界](#section-16)
- [十七、总结](#section-17)

---

<a id="section-1"></a>
## 一、下半章完成了什么

本阶段的“攻击”仍然是红色 Debug Sphere，而不是最终的 GameplayAbility 或投射物伤害。已经完成的是攻击决策和位置选择：

```text
近战敌人
  -> 进入近战范围
  -> BT_Attack（绘制调试球体）
  -> Wait 1 秒 ± 0.5 秒
  -> BT_T_BypassTarget 在目标周围找导航点
  -> Move To（移动位置向量）
  -> 回到攻击分支

远程敌人
  -> Run EQS Query
  -> 生成附近候选点
  -> Trace 过滤无法看见 Aura 的点
  -> Distance 对剩余点评分
  -> 将最佳点写入 Blackboard
  -> Move To 射击位置
  -> BT_Attack
  -> Wait 后再次评估
```

所以这次提交的关键成果不是“敌人已经会造成伤害”，而是让 AI 能根据敌人类型和环境选择战术位置：近战单位不再站在一个点反复攻击，远程单位也不再隔着墙对玩家开火。

---

<a id="section-2"></a>
## 二、蓝图任务的 C++ 基类与执行协议

### 2.1 为什么先建一个 C++ 基类

在内容浏览器中可以直接创建“行为树任务蓝图”，但课程仍先创建一个 C++ 类 `UBTTask_Attack`，基类选择 `BTTask_BlueprintBase`：

```cpp
UCLASS()
class AURA_API UBTTask_Attack : public UBTTask_BlueprintBase
{
    GENERATED_BODY()

    virtual EBTNodeResult::Type ExecuteTask(
        UBehaviorTreeComponent& OwnerComp,
        uint8* NodeMemory) override;
};
```

当前实现只转发父类：

```cpp
EBTNodeResult::Type UBTTask_Attack::ExecuteTask(
    UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    return Super::ExecuteTask(OwnerComp, NodeMemory);
}
```

这样做的意义是保留一个可扩展的 C++ 入口，同时把简单的行为放到蓝图中。当前攻击只需要取 Pawn、画调试球和结束任务，用蓝图完成不会带来有意义的性能收益；复杂算法、可复用底层能力或需要严格类型检查的部分再下沉到 C++。

### 2.2 `ExecuteTask` 与 `Receive Execute AI`

`BTTask_BlueprintBase` 的 C++ `ExecuteTask` 会把执行转发给蓝图事件 `Receive Execute AI`。事件通常可以取得：

- `Owner Controller`：执行行为树的 AIController；
- `Controlled Pawn`：当前被 Controller 占有的敌人；
- `Owner Comp`：行为树组件；
- `Node Memory`：需要持续保存状态时使用的节点内存。

蓝图任务必须明确调用 `Finish Execute`（Success 或 Fail）。如果只执行打印、画球等逻辑而不结束，行为树会一直停在该 Task，后续 Sequence 节点永远不会运行。

```text
Receive Execute AI
  -> 完成本任务的逻辑
  -> Finish Execute (Success/Fail)
```

`Receive Abort AI` 则用于任务被 Decorator 或更高层逻辑中止时的清理工作，例如停止计时器、取消异步动作或恢复状态。

---

<a id="section-3"></a>
## 三、创建 BT_Attack 蓝图任务

### 3.1 内容浏览器操作

1. 在 `Content/Blueprints/AI/Tasks`（本章中也使用 `AI` 子目录）右键；
2. 选择 `Blueprint Class`；
3. 搜索并选择 `BTTask_Attack`；
4. 命名为 `BT_Attack`，与 C++ 类区分；
5. 打开蓝图，点击 `Class Defaults` 旁的眼睛图标（若有变量需要暴露），编译并保存。

行为树的“新建任务”菜单中会同时出现 C++ 任务和蓝图任务。要注意：

- `BTTask_Attack` 是 C++ 类名；
- `BT_Attack` 是基于它创建的蓝图任务；
- 若选错 C++ 版本，可能看不到蓝图中新增的公开变量和调试节点。

### 3.2 蓝图节点连接

在事件图中实现：

```text
Receive Execute AI
  -> Controlled Pawn
  -> Get Actor Location
  -> Draw Debug Sphere
       Center = Pawn Location
       Radius = 40
       Color = Red
       Duration = 3（调试阶段）
  -> Finish Execute
       Success = true
```

`Controlled Pawn` 是敌人本体；`Owner Controller` 是 `AuraAIController`。测试时把 `Get Object Name(Controlled Pawn)` 接到 `Print String`，可确认任务确实由预期的敌人执行，而不是误取了 Controller。

### 3.3 蓝图调试信息的生命周期

屏幕刷屏通常不是 AI 重复执行错误，而是任务被 Sequence 反复成功触发。调试完成后应删除 `Print String`，或把 `Draw Debug Sphere` 的持续时间降到 0.5 秒；否则大量输出会掩盖真正的状态变化。

---

<a id="section-4"></a>
## 四、近战攻击者：攻击后绕目标换位

### 4.1 新增 Blackboard 键

在 Blackboard 中新增：

| 键名 | 类型 | 用途 |
|---|---|---|
| `MoveToLocation` | Vector | 保存目标周围的新导航位置 |

它与 `TargetToFollow` 不同：后者是一个 Actor/Object，指向 Aura；前者是一个 Vector，表示本次要移动到的具体点。

### 4.2 行为树 Sequence

近战分支可以连接为：

```text
Sequence（近战攻击）
  -> BT_Attack
  -> Wait（1 秒，随机误差 ±0.5 秒）
  -> BT_T_BypassTarget
  -> Move To（Blackboard Key = MoveToLocation）
```

`Wait` 的随机误差让多个敌人不会完全同帧攻击，减少“所有敌人同时动作”的机械感。移动到新位置成功后，Sequence 完成，Selector 会重新评估距离与攻击条件。

### 4.3 动画混合的小修正

近战敌人在“移动分支结束—攻击分支开始”之间可能出现轻微抖动。课程检查 `ABP_GoblinSpear` 的 Idle/Walk/Run Blend Space，把 Blend Space 的权重速度调整为约 4，使样本权重在约四分之一秒内完成变化，减少停止和启动时的生硬感。这个修正属于动画表现层，不改变行为树的状态机。

---

<a id="section-5"></a>
## 五、BT_T_BypassTarget 蓝图任务的详细连线

这是本章最值得复用的蓝图任务。它不负责攻击，只负责在目标附近找到一个可导航的新位置，并把位置写入 Blackboard。

### 5.1 创建与暴露变量

1. 右键 `Blueprint Class`；
2. 选择 `BTTask_BlueprintBase`；
3. 命名 `BT_T_BypassTarget`；
4. 新建变量 `NewLocation`，类型为 `Blackboard Key Selector`；
5. 新建变量 `Target`，类型也为 `Blackboard Key Selector`；
6. 新建变量 `Radius`，类型 Float，默认值 300；
7. 打开变量旁的眼睛图标，使它们在行为树 Task 详情面板可编辑；
8. 编译后回到行为树，分别绑定：
   - `NewLocation` -> `MoveToLocation`（Vector）；
   - `Target` -> `TargetToFollow`（Object/Actor）；
   - `Radius` -> 需要的随机搜索半径。

没有编译或没有打开眼睛图标时，变量不会出现在行为树节点的 Details 面板，这是蓝图操作中很常见的“节点上找不到属性”原因。

### 5.2 读取目标并检查有效性

```text
Receive Execute AI
  -> Target（Blackboard Key Selector）
  -> Get Blackboard Value as Object/Actor
  -> Is Valid
```

`TargetToFollow` 的基类是角色，因此可以使用 `Get Blackboard Value as Character`，再调用 `Get Actor Location`。不能把 Object 键直接当 Vector 使用，必须先取出对象，再取对象位置。

### 5.3 获取导航半径内随机位置

```text
Get Random Reachable Point in Radius
  Origin = Target Character -> Get Actor Location
  Radius = Radius（默认 300）
  Random Location -> Set Blackboard Value as Vector(NewLocation)
  Return Value -> Finish Execute
```

该节点和普通“在圆内随机一个 Vector”不同：它会结合导航系统寻找可到达点，避免随机位置落在导航网格孔洞或不可行走几何体内部。`Return Value` 失败时应直接 `Finish Execute(false)`，成功时写入 Vector 后 `Finish Execute(true)`。

### 5.4 行为树中的后续 Move To

把 `Move To` 的 Blackboard Key 设为 `MoveToLocation`，Acceptance Radius 约 20。流程变成：先围绕 Aura 生成一个点，再让敌人真正寻路到这个点，而不是继续把 Aura 当作移动目标。

---

<a id="section-6"></a>
## 六、时间限制与行为树防卡死

敌人可能被其他敌人、Aura 或墙体挡住。导航网格对动态角色不一定会自动挖出可绕行的洞，因此 `Move To(MoveToLocation)` 不能无限等待。

操作步骤：

1. 右键 `Move To` 节点；
2. 添加 `Time Limit` Decorator；
3. 时间设置约 2 秒；
4. `Observer Aborts` 设为 `Self` 或选择节点自身自动中止；
5. 超时后让 Sequence 失败，Selector 重新寻找下一次攻击位置。

```text
Move To(MoveToLocation)
  + Time Limit(2s, Abort Self)
```

这不是“让寻路更快”，而是给任务增加失败出口，防止 AI 卡在一个永远无法完成的 Task。测试中把敌人放在石柱或 Aura 附近，观察 2 秒后是否退出并重新攻击，是验证该保护的直接方法。

---

<a id="section-7"></a>
## 七、EQS 的思想与运行管线

远程敌人不能只根据“距离小于 600”就开火：墙后的位置距离可能很近，却完全看不到 Aura。EQS（Environment Query System，环境查询系统）把选点拆成三步：

```text
Generator 生成候选 Items
  -> Tests 对候选点过滤或评分
  -> Query 选择最佳 Item
  -> 写入 Blackboard
  -> 行为树继续 Move To / Attack
```

### 7.1 Item、Context、Test

| 概念 | 含义 |
|---|---|
| Item | Generator 生成的候选对象或位置 |
| Generator | 生成点阵、圆形点、角色集合等候选项 |
| Context | 测试参照对象，如 Query Pawn 或玩家 Aura |
| Test | 对每个 Item 做 Trace、Distance 等过滤/评分 |
| Score | 归一化后的优先级，通常在 0～1 范围 |

Trace 是布尔性质更强的测试：点能否无遮挡命中目标。Distance 通常用于连续评分：候选点离某个 Context 越近或越远，分数越高。

---

<a id="section-8"></a>
## 八、EQS 测试资源与独立测试地图

### 8.1 内容浏览器整理

当 AI 资产增多时，建议在 `Blueprints/AI` 下建立：

```text
AI/
  Behavior/       行为树与黑板
  Tasks/          BT_Attack、BT_T_BypassTarget
  Services/       BTService_FindNearestPlayer
  AIControllers/  AuraAIController 相关资产
  EQS/            EQ 查询、Context、测试资源
```

文件夹整理不会改变运行逻辑，但能降低在“新建任务”菜单中选错同名资产的概率。

### 8.2 EQS Testing Pawn

1. 在 EQS 文件夹创建 `Blueprint Class`；
2. 搜索 `EQS Testing Pawn`；
3. 命名 `BP_EQSTestingPawn`；
4. 在场景中放置它；
5. 在 Details 中把 Query Template 设为 `EQ_FindRangedAttackPosition`。

Testing Pawn 会把 Generator 生成的点和测试分数绘制出来，是理解 EQS 的可视化探针，不是最终游戏中的 AI 角色。

### 8.3 独立 EQS 测试地图

通过 `File -> New Level -> Basic` 新建地图，保存为 `EQS_TestMap`。地图只保留：

- 一个 EQS Testing Pawn；
- 一个 Aura/目标角色；
- 可控数量的墙、柱子等障碍；
- `NavMeshBoundsVolume`。

独立地图的意义是减少其他敌人、技能和 UI 对观察结果的干扰，并能在同一布局下反复比较 Generator 半径、Trace 通道和评分因子。

---

<a id="section-9"></a>
## 九、EQS 生成器：从候选物品网格开始

在 EQ 查询图中从 Root 拖出 `Points: Grid`（路径网格/点阵生成器）：

- `Grid Half Size = 1000`：生成区域横纵各约 2000 Unreal Units；
- `Space Between = 100`：点间距 100；
- 调大间距会减少候选点和测试成本；
- 调大半径会覆盖更大的战术空间，但候选数量会快速增长。

也可以用 `Points: Circle` 观察圆形候选点：调整 Radius、Spacing、Arc Angle 就能得到完整圆或扇形。Generator 只负责“提出候选”，不保证点可走、能看见目标或一定是最佳点。

选择路径相关 Generator 的好处是它会尊重导航系统，最终候选点不会随意落在 NavMesh 外。但它仍可能受到碰撞、导航代理半径和地图更新状态影响，所以必须配合测试地图验证。

---

<a id="section-10"></a>
## 十、Trace 测试：筛选可看见玩家的位置

### 10.1 创建测试

在 Generator 上右键 `Add Test -> Trace`。在 Details 面板设置：

- Trace Channel：默认 Visibility；
- Trace From：候选 Item；
- Trace To：Context；
- Context：从默认 Query 改为自定义 `EQS_PlayerContext`；
- `Bool Match`：决定命中结果是保留还是过滤。

### 10.2 创建 EQS_PlayerContext

1. 在 EQS 文件夹右键创建 `Blueprint Class`；
2. 选择 `EnvQueryContext_BlueprintBase`；
3. 命名 `EQS_PlayerContext`；
4. 重写 `Provide Actors Set`；
5. 使用 `Get All Actors Of Class (BP_AuraCharacter)`；
6. 将返回的 Aura Character 数组接到 Actors 输出；
7. 编译保存，并在 Trace Test 的 Context 属性中选择它。

Context 的职责是回答“Trace 要追踪谁”。如果用 Query Pawn 作为目标，测试角色本身可能没有可追踪组件，所有点都会得到 0 分；自定义 Context 把目标明确为玩家角色后，测试才有游戏意义。

### 10.3 Bool Match 与调试颜色

Testing Pawn 的颜色反映测试状态：

- 能追踪到玩家、没有障碍的点：通过测试；
- 被墙挡住的点：失败并被过滤；
- `Bool Match` 反向时，颜色和保留集合也会反转。

要测试结果是否正确，可以在测试 Pawn 与 Aura 中间放一面墙，再移动其中一个。墙一侧的点应该被过滤，绕过墙角后重新出现。测试时要注意“移动 Actor 后等待 EQS 刷新”，有些调试图形只有在查询重新执行后才更新。

### 10.4 碰撞与导航的关系

如果墙体没有 Simple Collision，Trace 可能从网格内部穿过；如果墙体没有参与 Visibility 通道，Trace 也不会把它当作遮挡物。检查 Static Mesh Editor 的 `Show -> Simple Collision`，必要时在 Collision 中添加 Box Simplified Collision。

同时放置并调整 `NavMeshBoundsVolume`，按 `P` 显示绿色导航网格。只有位于可导航区域的点才适合被 AI 走到；Trace 解决“看不看得见”，NavMesh 解决“走不走得到”，两者是不同层面的测试。

---

<a id="section-11"></a>
## 十一、距离测试与评分因子

Trace 过滤掉不可见点后，再右键添加 `Distance` Test：

1. Context 选择 `EQS_PlayerContext` 或 Query；
2. Test Purpose 选择 `Score`，不要只选择 Filter；
3. 对剩余候选点计算距离；
4. 使用 `Scoring Factor` 调整“近还是远”更优。

距离通常会归一化为 0～1。若评分因子为 `1`，距离更远的点得分更高；若要选择离 AI 自己最近、但仍然能看见玩家的点，将目标设为 Query Context，并把 Scoring Factor 设为 `-1`：

```text
可见性 Trace：过滤掉被墙挡住的点
距离 Distance：对剩余点评分
Scoring Factor = -1：越靠近 Query Pawn，得分越高
最终 Item：最近的可见点
```

这正是远程敌人需要的战术位置：不必贴到 Aura 身边，同时也不必跑到地图另一端；只要找到最近的无遮挡射击点即可。如果墙挡住直线路径，最高分会转移到墙角附近，AI 先绕过去再攻击。

---

<a id="section-12"></a>
## 十二、远程攻击分支的蓝图操作

在 Behavior Tree 的 `RangedAttacker` Sequence 中：

```text
Sequence（远程）
  -> Run EQS Query
       Query Template = EQ_FindRangedAttackPosition
       Run Mode = Single Best Item
       Blackboard Key = MoveToLocation
  -> Move To
       Blackboard Key = MoveToLocation
  -> BT_Attack（蓝图版本）
  -> Wait（1 秒 ± 0.5 秒）
```

### 12.1 Run EQS Query 的关键设置

从 Sequence 拖出 Task，选择 `Run EQS Query`，展开 `EQS Request`：

- `Query Template` 选择 `EQ_FindRangedAttackPosition`；
- `Blackboard Key` 选择 `MoveToLocation`；
- 查询完成后，最佳 Vector Item 会写入该键；
- `Move To` 随后只读取这个 Vector，不需要知道 EQS 内部如何生成和评分。

节点名称建议改为“计算射击位置”，下一个 Move To 改为“进入射击位置”，让行为树阅读者一眼看到“计算”和“执行”是两个阶段。

### 12.2 常见蓝图选择错误

行为树中可能同时出现 `BTTask_Attack`（C++）和 `BT_Attack`（蓝图）。本阶段测试发现如果误用 C++ 任务，蓝图里绘制的红色球体不会出现，容易误判为行为树没有执行。应确认节点标题和资产路径，远程与近战都使用已经接入调试表现的 `BT_Attack` 蓝图版本。

---

<a id="section-13"></a>
## 十三、完整 AI 行为树流程

```text
Root
`-- Selector
    |-- RangedAttacker Sequence
    |   |-- Target 存在 / 未 HitReacting
    |   |-- 距离满足远程条件
    |   |-- 计算射击位置（Run EQS Query）
    |   |-- 进入射击位置（Move To MoveToLocation）
    |   |-- BT_Attack
    |   `-- Wait 1s ± 0.5s
    |
    |-- Melee Sequence
    |   |-- RangedAttacker == false
    |   |-- 目标存在且距离满足近战条件
    |   |-- BT_Attack
    |   |-- Wait 1s ± 0.5s
    |   |-- 寻找目标周围新位置（BT_T_BypassTarget）
    |   `-- Move To MoveToLocation（Time Limit 2s）
    |
    `-- Approach Sequence
        |-- Target 存在
        `-- Move To TargetToFollow
```

分支的决策数据仍来自前半章的 Blackboard：`TargetToFollow`、`DistanceToTarget`、`HitReacting`、`RangedAttacker`。下半章新增的 `MoveToLocation` 只表示“本次选出的战术位置”，不要把它和目标 Actor 混用。

---

<a id="section-14"></a>
## 十四、本章 AI 测试技术栈

这里的“测试”不是单一按钮，而是一套从资产级到运行时的组合工具。

### 14.1 资产和编辑器级

| 工具/技术 | 用途 | 观察点 |
|---|---|---|
| Behavior Tree Editor | 查看当前执行分支、Decorator、Task 和 Service | 哪条分支变绿、哪个 Task 卡住、Abort 是否发生 |
| Blackboard Editor | 检查键类型与运行时值 | Object、Float、Bool、Vector 是否与节点匹配 |
| EQS Editor | 调整 Generator/Test/Context 参数 | 候选点数量、过滤结果、评分方向 |
| Class Defaults/变量眼睛 | 暴露蓝图 Task 参数 | 行为树节点 Details 是否出现 `Radius`、Key Selector |
| Compile/Save | 同步蓝图生成类与资产引用 | 新变量、事件和查询模板是否真正生效 |

### 14.2 EQS 专用可视化

`EQS Testing Pawn` 是 EQS 的运行时探针：它显示每个 Item 的位置、颜色和分数，适合验证 Generator 与 Tests，而不是验证完整战斗。建议固定一套测试布局，然后一次只改一个参数：

```text
先验证 Grid 半径/间距
  -> 再验证 Trace Context 与 Visibility
  -> 再验证 Bool Match 过滤方向
  -> 最后验证 Distance Scoring Factor
```

这样可以区分“没有生成点”“点被过滤”“点存在但评分低”“行为树没有消费结果”四类问题。

### 14.3 导航测试

| 操作 | 说明 |
|---|---|
| 放置 `NavMeshBoundsVolume` | 决定导航烘焙范围 |
| 按 `P` | 显示绿色可行走 NavMesh |
| 调整 Agent Radius/碰撞 | 影响狭窄通道是否可达 |
| `Get Random Reachable Point in Radius` | 验证随机位置是否真正可走 |
| Move To Acceptance Radius | 验证到达判定是否过近或过远 |
| Time Limit Decorator | 验证卡路时能否退出任务 |

导航与 Trace 必须分别测试：绿色不代表能看见玩家，Trace 通过也不代表角色能到达。

### 14.4 运行时调试

- `Print String`：确认 `Controlled Pawn`、Controller、黑板值和任务是否触发；
- `Draw Debug Sphere`：用红球表示攻击 Task 的执行时刻和敌人当前位置；
- Behavior Tree Runtime Debugger：观察当前运行节点和黑板变化；
- `P` 导航显示：确认 AI 不是在 NavMesh 外寻路；
- 屏幕日志/Output Log：记录重复执行、资产加载和异常；
- Play In Editor：验证从生成、Possess、Service Tick 到分支切换的完整链路。

调试输出要有生命周期。打印最近目标的临时信息在定位完成后应移除；攻击球体可缩短为 0.5 秒，否则输出量会妨碍观察时序。

### 14.5 C++ 编译与崩溃定位

当测试墙体后出现 `PathPoints` 数组越界或 `PlayerController` 相关错误时，调试流程是：

1. 记录错误调用栈和代码行；
2. 回到对应 C++ 访问点；
3. 在使用 `PathPoints.Last()` 前检查 `PathPoints.Num() > 0`；
4. 用 `Ctrl+Shift+B` 编译；
5. 必要时清理中间文件和二进制，重新生成项目文件；
6. 用 Debug 模式再次运行，而不是直接相信上一次的热重载结果。

这套流程把“蓝图行为异常”与“底层 C++ 空数组访问”区分开。蓝图测试触发了边缘条件，但修复仍应落在负责访问数据的 C++ 代码中。

### 14.6 碰撞通道测试

墙体需要分别验证：

- 是否有 Simple Collision；
- 是否阻挡 Visibility Trace；
- 是否阻挡 NavMesh；
- 是否影响点击移动。

课程中为了测试远程 AI，墙需要阻挡 Visibility；但如果同一通道也被点击移动使用，玩家短按点击墙体可能无法得到预期路径。因此长期方案是为点击移动和 AI 视线设计不同的碰撞通道，并在项目设置与静态网格预设中明确配置。

---

<a id="section-15"></a>
## 十五、实际测试中发现的问题与定位方法

### 15.1 远程敌人仍然直冲玩家

排查顺序：

1. 确认 `RangedAttacker` 黑板键确实为 true；
2. 在行为树中看远程 Sequence 是否变绿；
3. 确认 `Run EQS Query` 的模板是 `EQ_FindRangedAttackPosition`；
4. 确认输出 Blackboard Key 是 `MoveToLocation`；
5. 确认后续 `Move To` 使用同一个 Vector 键；
6. 检查是否误放了 C++ `BTTask_Attack` 而不是蓝图 `BT_Attack`；
7. 用 EQS Testing Pawn 单独确认查询能返回可见点。

本次测试实际发现过“使用了 C++ 攻击节点”的资产选择问题；节点逻辑看似执行，但没有蓝图红球，导致行为判断被误导。

### 15.2 所有 EQS 点都为红色或 0 分

可能原因：

- Context 仍是 Query，而 Query Pawn 没有可追踪组件；
- `EQS_PlayerContext` 没有返回 `BP_AuraCharacter`；
- Aura 没有放入测试地图；
- Aura 与测试 Pawn 高度重合，Trace 命中地面；
- Visibility 通道没有被障碍物阻挡或墙没有 Simple Collision；
- `Bool Match` 方向与预期相反。

解决方法是先只放 Testing Pawn 和 Aura，确认无遮挡点能通过，再加入墙和导航障碍。

### 15.3 AI 被墙或其他角色卡住

近战绕行使用 `Get Random Reachable Point in Radius` 后仍可能没有可用路径。Time Limit 不是修复寻路算法，而是保证行为树能够失败并重新尝试。需要同时检查：

- NavMesh 是否覆盖目标区域；
- 动态角色是否造成通道拥堵；
- 随机半径是否过大，导致点落在远处；
- Move To 的 Acceptance Radius 是否过小；
- Time Limit 是否过短，造成频繁重试。

### 15.4 修改未保留与 Debug/Development 模式

崩溃或编辑器关闭后，关卡中删除的敌人、墙体和碰撞设置可能没有保存。每次测试前先保存地图和资产；修改 C++ 后重新编译，再用 Debug 或 Development 配置运行，避免热重载状态与磁盘资产不一致。

---

<a id="section-16"></a>
## 十六、本次提交文件与资产边界

Git 提交 `306cf95` 实际包含 3 个源码文件：

| 文件 | 作用 |
|---|---|
| `Source/Aura/Public/AI/BTTask_Attack.h` | 声明继承 `BTTask_BlueprintBase` 的 C++ 攻击任务基类 |
| `Source/Aura/Private/AI/BTTask_Attack.cpp` | 重写 `ExecuteTask` 并转发父类执行蓝图事件 |
| `Source/Aura/Private/AI/BTService_FindNearestPlayer.cpp` | 删除最近目标 Service 中的屏幕调试打印 |

15.8～15.14 的大量工作发生在 UE 编辑器的蓝图、Behavior Tree、Blackboard、EQS、测试地图和关卡资产中。当前仓库状态中这些内容未以本次源码提交的形式列出，因此整理笔记时应区分：

- C++ 提交提供“可被蓝图继承的任务基类”和 Service 清理；
- 蓝图资产完成行为树的实际连接；
- EQS 测试地图与调试摆放属于编辑器资产/关卡配置；
- 最终真正攻击、伤害和 GameplayAbility 激活尚未在本章完成。

---

<a id="section-17"></a>
## 十七、总结

### 17.1 本阶段最重要的蓝图经验

1. Blueprint Task 必须走 `Receive Execute AI -> Finish Execute` 协议；
2. `Blackboard Key Selector` 让一个任务可以复用到不同黑板键；
3. Vector 位置键和 Actor 目标键要分开，不能混用；
4. `Compile + 打开眼睛图标 + 回到行为树绑定` 是公开 Task 参数的完整流程；
5. Decorator 的 Abort 设置决定条件变化时行为树是否立即切换；
6. EQS 的 Generator 只生成候选，Context 决定参照目标，Test 决定过滤和评分；
7. 远程射击位置必须同时满足“可导航”和“可见目标”。

### 17.2 本阶段最重要的测试经验

```text
独立地图隔离变量
  -> EQS Testing Pawn 看候选点和分数
  -> NavMesh/P 检查可达区域
  -> Trace/Collision 检查视线障碍
  -> Behavior Tree Debugger 检查分支和黑板
  -> Debug Sphere/Print 检查任务时序
  -> Debug 编译与调用栈定位 C++ 边界问题
```

### 17.3 一句话总结

**第 15 章下半把行为树从“会追踪目标”推进为“会按近战/远程职业选择战术位置”：近战通过蓝图任务在目标周围随机换位，远程通过 EQS 生成、过滤和评分候选点；同时用独立测试地图、EQS Testing Pawn、NavMesh、Trace、运行时调试和 C++ 调用栈组成一套可重复的 AI 测试技术栈。**


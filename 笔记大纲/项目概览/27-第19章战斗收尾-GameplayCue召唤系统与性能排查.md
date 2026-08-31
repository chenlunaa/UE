# 第 19 章：战斗收尾——GameplayCue、召唤系统与性能排查

> 本章围绕“让战斗像一个完整游戏”展开：补齐脚步、挥砍、受击、死亡和投射物音效；用 GameplayCue 将服务器执行的命中特效同步到客户端；创建恶魔敌人和萨满召唤能力；增加召唤数量控制、攻击类型分流、投射物抛射角参数；最后通过时间轴缓动、日志、帧率和粒子生命周期排查重复生成与内存泄漏。

---

## 1. 本章完成的功能地图

~~~text
战斗表现
├── 动画 Notify：脚步、挥砍、受击、死亡音效
├── MetaSound/复合音效：随机音量与音高
├── 目标血效：CombatInterface -> Niagara System
├── GameplayCue：把服务器命中的装饰效果同步到客户端
└── 手臂拖尾、投射物命中与落地效果

敌人扩展
├── 恶魔近战/远程敌人
├── 萨满召唤恶魔
├── Spawn Location 随机分布
├── 异步/分帧生成
├── 召唤后记录 MinionCount
└── 低于阈值时重新召唤

工程质量
├── 死亡后停止 AI
├── Pitch Override 参数化
├── Spec SetByCaller 缺失值警告修复
├── Niagara 粒子生命周期修复
└── 日志、断点和 FPS 压力测试
~~~

本章主线是：表现效果不能破坏网络权威；蓝图流程不能依赖隐含状态；大量敌人和粒子必须有生命周期与数量预算。

## 2. 音效：动画事件还是玩法事件

与动作时间轴强相关的声音应放在 Animation 或 Montage Notify 中：

- 脚步声：脚掌触地帧；
- 挥砍/破空声：武器达到挥动速度的帧；
- 受击声：Hit React 动画发生的帧；
- 施法发射声：火球从法杖尖端发出的帧。

操作模式：

~~~text
打开 Animation/Montage
  -> 新建 Sound Notify Track
  -> Add Notify -> Play Sound
  -> 选择 Sound 或 MetaSound
  -> 将 Notify 移到正确动作帧
~~~

命中冲击声不能简单放在攻击动画中，因为挥空时不应播放。应由命中逻辑或 GameplayCue 触发：

~~~text
攻击动画挥空     -> 不播放命中声
攻击动画命中目标 -> 播放命中声并生成血效
~~~

复合音效可以把多个样本放入数组，通过音量和音高随机化减少重复感；大量敌人同时移动时应制作更安静的脚步变体。

## 3. 目标血效：CombatInterface 返回 Niagara

不同敌人可能有不同血液颜色，因此血效从被击目标获取，而不是攻击能力硬编码。

角色基类增加：

~~~cpp
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
TObjectPtr<UNiagaraSystem> BloodEffect;
~~~

CombatInterface 增加蓝图原生事件：

~~~cpp
UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
UNiagaraSystem* GetBloodEffect();
~~~

基类实现直接返回 BloodEffect。头文件只前向声明 UNiagaraSystem，完整 Niagara 头文件放在 CPP 中，减少编译依赖。

近战能力的蓝图流程：

~~~text
For Each Overlapping Target
  -> 检查是否友军
  -> 非友军：应用 Damage GE
  -> 通过 CombatInterface 获取目标 BloodEffect
  -> 在 Combat Socket Location 生成 Niagara
~~~

如果一次攻击命中多个目标，可以生成多份血效；冲击音效通常只播放一次。用 HasHitTarget 布尔变量记录是否命中，循环结束后统一播放 Tagged Montage 中的 ImpactSound。

将 CombatSocketLocation 提升为变量，音效和 Niagara 共用同一位置，避免重复调用接口和杂乱线路。

## 4. GameplayCue：同步服务器命中的装饰效果

AI 敌人的 ASC 通常没有真实玩家拥有。服务器 Ability 中直接执行 Play Sound 或 Spawn Niagara，只会发生在服务器端，客户端看不到。

### 4.1 创建 GameplayCue

1. 在敌人 Ability 目录新建 Cues 文件夹；
2. 创建 Blueprint Class；
3. 选择 GameplayCueNotify_Static；
4. 命名为 GC_MeleeImpact；
5. 重写 OnExecute；
6. 从 GameplayCueParameters 读取 Location、Normal、Magnitude 和 EffectContext；
7. 在该位置播放冲击声并生成 Niagara。

静态 Cue 不需要为每个效果创建 Actor，适合一次性命中、爆炸、飘字等表现。

### 4.2 注册与执行

在 Project Settings 的 Gameplay Tags 中添加：

~~~text
GameplayCue.MeleeImpact
~~~

近战 Ability 中：

~~~text
循环结束
  -> HasHitTarget == true
  -> 创建 GameplayCueParameters
  -> Parameters.Location = CombatSocketLocation
  -> Execute GameplayCue with Owner
  -> GameplayCue Tag = GameplayCue.MeleeImpact
~~~

Location 不会凭空出现，必须手动设置。Cue 负责本地表现，伤害仍由服务器 Damage GE/ExecCalc 完成。

## 5. 恶魔敌人和远程 Ability 复用

恶魔继续从 Enemy Base 创建子蓝图，复用 Mesh、AnimBP、攻击蒙太奇、Motion Warping、Tagged Montage、Hit React、死亡和溶解流程。

远程攻击应采用：

~~~text
GA_RangedAttack：通用生成流程
子蓝图：ProjectileClass、DamageType、Curve、SocketTag
~~~

否则父 Ability 中硬编码的弹弓石会被所有远程敌人继承。

## 6. UAuraSummonAbility：召唤位置算法

源码增加 UAuraSummonAbility，关键配置为：

~~~cpp
int32 NumMinions = 5;
TArray<TSubclassOf<APawn>> MinionClasses;
float MinSpawnDistance = 75.f;
float MaxSpawnDistance = 300.f;
float SpawnSpread = 90.f;
~~~

GetSpawnLocations 首先取得召唤者位置和 ForwardVector，然后把 SpawnSpread 分割为 NumMinions 个角度步进，在前方扇形中随机选取距离。

~~~cpp
const float DeltaSpread = SpawnSpread / NumMinions;
const FVector RightOfSpread =
    ForwardVector.RotateAngleAxis(SpawnSpread / 2.f, FVector::UpVector);
~~~

对每个方向：

~~~cpp
FVector ChosenSpawnLocation =
    Location + Direction * FMath::FRandRange(
        MinSpawnDistance, MaxSpawnDistance);
~~~

再从候选点上方 400 到下方 400 做 Visibility LineTrace。命中地面后使用 Hit.ImpactPoint。生产代码还应补充 NavMesh、胶囊重叠和小兵间距检查。

GetRandomMinionClass 从 MinionClasses 随机返回一个 TSubclassOf<APawn>。必须先检查数组非空，否则可能出现 RandRange(0, -1) 和越界。

## 7. 召唤蓝图流程

~~~text
Get Spawn Locations
  -> For Each Location
  -> Draw Debug Sphere（调试阶段）
  -> Spawn GroundSummon Niagara
  -> Play Summon Montage and Wait
  -> Wait Gameplay Event(Summon)
  -> Get Random Minion Class
  -> Spawn Actor from Class
  -> Increment Minion Count
~~~

召唤蒙太奇中放置 Montage Event Notify，Ability 等待该 Tag 后才生成小兵。开启 Root Motion 可避免萨满在施法期间滑动。

Ground Summon Niagara 的生命周期要和召唤延迟匹配。调试球体和调试打印在验证完成后应删除或放进 Debug 开关。

## 8. 小兵数量和行为树分流

召唤 Ability 应使用：

~~~text
Abilities.Summon
~~~

而攻击 Ability 使用 Abilities.Attack，避免通用 Attack Task 把召唤当作攻击。

CombatInterface 增加：

~~~cpp
int32 GetMinionCount();
void IncrementMinionCount(int32 Amount);
~~~

生成小兵后加 1；小兵销毁后减 1。可在 Spawn Actor 后绑定 OnDestroyed，否则计数会只增不减。

Elementalist 专用攻击 Task 增加：

~~~text
SummonTag = Abilities.Summon
AttackTag = Abilities.Attack
MinionSpawnThreshold = 2
~~~

决策流程：

~~~text
GetMinionCount()
  -> Count < Threshold ?
      True  -> TryActivateAbilitiesByTag(SummonTag)
      False -> TryActivateAbilitiesByTag(AttackTag)
~~~

## 9. Pitch Override 参数化

远程 Ability 可将投射物生成函数改成：

~~~cpp
void SpawnProjectile(
    const FVector& ProjectileTargetLocation,
    const FGameplayTag& SocketTag,
    bool bOverridePitch = false,
    float PitchOverride = 0.f);
~~~

实现中先由目标方向计算 Rotation，只有 bOverridePitch 为 true 时才覆盖 Pitch。

萨满火球通常关闭覆盖，弹弓石块可以设置正的 Pitch。不要把角度写死在所有远程敌人的父类中，应把变量暴露给子 Ability。

## 10. 源码阅读痕迹与正式逻辑

本章源码中有几类内容是为了学习和排错，不是最终功能：

### 10.1 讲解注释

例如对 OnComponentBeginOverlap.Broadcast 参数的注释，只是在说明委托签名，不会执行。

### 10.2 临时 UE_LOG

~~~cpp
UE_LOG(LogTemp, Warning, TEXT("Spawned %s"), *GetName());
~~~

用于确认投射物是否重复生成。问题解决后应删除或改为受控 Verbose 日志。

### 10.3 断点和 Draw Debug Sphere

它们用于观察 Overlap 目标、Gameplay Event 次数和召唤点分布，验证结束后应关闭。

### 10.4 SetByCaller 缺失值

ExecCalc 遍历所有 DamageType 时，某个 GE 没有设置某种伤害 Tag 可能是合法情况，应默认按 0 处理，不要把可选缺失打印成错误。

## 11. 多人和投射物重要函数

SpawnActorDeferred 先生成投射物，再在 FinishSpawning 前写入 DamageEffectSpec。

MakeEffectContext 记录 Ability、Projectile、Actors 和 HitResult，使 ExecCalc 能追踪来源。

AssignTagSetByCallerMagnitude 把每种伤害写入 Spec；Ability 写入的 Tag 必须与 ExecCalc 读取的 Tag 完全一致。

OnSphereOverride 的推荐保护顺序：

~~~text
OtherActor 有效？
  -> 是否 Owner / Instigator？
  -> DamageEffectSpecHandle.Data 有效？
  -> 是否重复命中？
  -> 是否敌对？
  -> 客户端播放表现
  -> 服务器应用 Damage GE 并 Destroy
~~~

客户端负责表现，服务器负责应用伤害和销毁投射物；bHit 用于避免命中回调和 Destroyed 回调重复播放效果。

## 12. 性能排查和内存泄漏

正常负载通常是 FPS 降低后稳定在新的基准线；内存泄漏则是在相同操作持续执行时 FPS 不断下降、对象不断增加。

### 12.1 重复 Gameplay Event

如果一次攻击收到多个事件，可能重复生成投射物。应检查 Ability 是否重复激活、Montage 是否多次播放、Event 是否重复发送，并将 Wait Gameplay Event 设置为 Only Trigger Once。

### 12.2 Niagara 粒子生命周期

Niagara Emitter 中出现无限符号时，应检查 Particle State 的 Kill Particles When Lifetime Has Elapsed。持续生成的命中特效如果不销毁，会造成对象累积。

压力测试要区分：

~~~text
更多敌人/更多粒子
  -> FPS 下降但稳定：正常负载

停止生成后仍持续下降
  -> 对象或粒子没有结束：疑似泄漏
~~~

### 12.3 调试工具链

- FPS 显示：观察是否持续下降；
- Output Log：统计 Spawn 和错误；
- C++ 断点：检查 Overlap、Event 和 Spec；
- Gameplay Debugger：观察 AI 与 Blackboard；
- UE Insights：定位 Tick、生成和销毁开销；
- 单个敌人与多个敌人的对照测试。

## 13. 蓝图、C++ 与 GAS 的职责边界

| 任务 | 蓝图 | C++/GAS |
|---|---|---|
| 敌人外观 | Mesh、AnimBP、材质、音效 | 基类组件和接口 |
| 动作节奏 | Montage、Notify、Motion Warping | Ability 等待事件 |
| 召唤配置 | 小兵类数组、数量、距离、Spread | 位置算法 |
| 召唤决策 | BT Task、Decorator、阈值 | Interface 读写计数 |
| 命中特效 | GameplayCue 蓝图 | GAS 网络调度 |
| 伤害数值 | Curve/DataAsset 参数 | ExecCalc 统一结算 |
| 性能排查 | 日志、调试节点、Niagara 属性 | 生命周期与权限安全 |

蓝图负责资产组合和流程组合，C++ 负责类型安全、网络权威和通用算法。

## 14. 回归测试清单

- [ ] 脚步和攻击音效与动画帧一致；
- [ ] 命中冲击音效只播放一次；
- [ ] GameplayCue 在客户端正确显示并使用传入位置；
- [ ] 不同敌人可使用不同血效；
- [ ] 召唤点位于地面且没有明显重叠；
- [ ] 萨满召唤时播放动画，完成事件后再生成；
- [ ] 小兵生成和销毁都会更新 MinionCount；
- [ ] 小兵不足时召唤，足够时攻击；
- [ ] 火球和石块使用不同 Pitch 设置；
- [ ] 死亡后敌人停止行为树和攻击；
- [ ] 一次 Montage Event 只生成一个投射物；
- [ ] 长时间压力测试 FPS 稳定；
- [ ] Niagara 粒子在生命周期结束后销毁；
- [ ] 临时日志、断点和调试球体已移除或关闭。

## 15. 本章总结

~~~text
动画 Notify 提供准确的声音节奏
  -> CombatInterface 提供目标专属血效
  -> GameplayCue 把服务器命中反馈传播到客户端
  -> SummonAbility 用数据驱动生成位置和小兵类型
  -> Blackboard/Ability Tag 控制召唤与攻击分工
  -> Pitch Override 让不同投射物拥有不同弹道
  -> 日志和 FPS 测试发现重复事件与粒子泄漏
~~~

最重要的工程原则：

1. 命中结果由服务器决定，装饰表现通过 GameplayCue 同步；
2. 通用流程放父类，敌人差异放子蓝图、Tag、DataAsset 和可配置变量；
3. 任何持续生成的 Actor、Ability、Event 和 Niagara 粒子都必须拥有明确的结束与销毁路径。

掌握这些原则后，继续增加治疗萨满、冰霜法师、召唤 Boss 或更多投射物时，主要是添加资产和配置，而不是复制整套系统。

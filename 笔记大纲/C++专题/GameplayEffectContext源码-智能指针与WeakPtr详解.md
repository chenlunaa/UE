# GameplayEffectContext 源码解析：它能做什么，以及智能指针与 WeakPtr

> 对应字幕：第 14 章 `14.1`  
> 主要源码：`GameplayEffectTypes.h/.cpp` 中的 `FGameplayEffectContext` 与 `FGameplayEffectContextHandle`  
> 项目示例：`UAuraProjectileSpell::SpawnProjectile`

---

## 目录

- [一、先说结论：EffectContext 是什么](#section-1)
- [二、Context、Handle 与 Spec 的关系](#section-2)
- [三、MakeEffectContext 的创建流程](#section-3)
- [四、FGameplayEffectContext 能保存什么](#section-4)
- [五、项目中的实际使用流程](#section-5)
- [六、为什么要使用指针](#section-6)
- [七、智能指针的核心：所有权与生命周期](#section-7)
- [八、TSharedPtr：共享拥有普通 C++ 对象](#section-8)
- [九、TWeakPtr：观察普通 C++ 对象](#section-9)
- [十、TWeakObjectPtr：观察 UObject](#section-10)
- [十一、TWeakPtr 与 TWeakObjectPtr 的区别](#section-11)
- [十二、EffectContext 为什么同时使用 SharedPtr 和 WeakObjectPtr](#section-12)
- [十三、Get、IsValid、Pin 和解引用](#section-13)
- [十四、CDO、能力实例与 const_cast](#section-14)
- [十五、复制、网络序列化和 Duplicate](#section-15)
- [十六、源码中的其他 C++ 语法](#section-16)
- [十七、什么时候应该往 Context 中放数据](#section-17)
- [十八、原生 Context 的限制](#section-18)
- [十九、常见误区](#section-19)
- [二十、速查表与记忆方法](#section-20)

---

<a id="section-1"></a>
## 一、先说结论：EffectContext 是什么

`FGameplayEffectContext` 可以理解为一张随 GameplayEffect 一起传递的**事件说明单**。

GameplayEffect 本身主要描述：

- 修改哪个属性；
- 使用什么计算方式；
- 持续多长时间；
- 授予哪些 GameplayTag。

但实际结算时还经常需要知道：

- 谁发起了效果；
- 谁或什么东西实际造成了效果；
- 效果来自哪个 GameplayAbility；
- 技能等级是多少；
- 来源对象是什么；
- 命中了世界中的什么位置；
- 还涉及了哪些 Actor。

这些“本次效果的背景信息”就由 `FGameplayEffectContext` 携带。

```text
GameplayEffect
  = 效果规则

GameplayEffectSpec
  = 已经实例化、准备应用的效果参数

GameplayEffectContext
  = 这次效果发生时的来源、技能、命中等背景信息
```

例如同一个 `GE_Damage` 可以被火球、冰箭和陷阱复用。GE 规则相同，但每次应用时 Context 中的来源技能、施法者、投射物和命中位置可以完全不同。

---

<a id="section-2"></a>
## 二、Context、Handle 与 Spec 的关系

### 2.1 三个类型各自负责什么

| 类型 | 职责 | 类比 |
|---|---|---|
| `FGameplayEffectContext` | 真正保存上下文数据 | 包裹里的内容 |
| `FGameplayEffectContextHandle` | 安全持有和传递 Context | 包裹的取件凭证 |
| `FGameplayEffectSpec` | 保存本次即将应用的 GE 完整实例数据 | 已填写好的效果订单 |

基本创建过程：

```cpp
FGameplayEffectContextHandle ContextHandle = SourceASC->MakeEffectContext();

FGameplayEffectSpecHandle SpecHandle = SourceASC->MakeOutgoingSpec(
    DamageEffectClass,
    GetAbilityLevel(),
    ContextHandle);
```

数据关系可以简化为：

```text
FGameplayEffectSpecHandle
  -> FGameplayEffectSpec
       -> FGameplayEffectContextHandle
            -> TSharedPtr<FGameplayEffectContext>
                 -> 真正的上下文对象
```

### 2.2 为什么平时操作 Handle，而不是直接操作 Context

Handle 提供了一组转发函数：

```cpp
ContextHandle.AddSourceObject(SourceObject);
ContextHandle.SetAbility(Ability);
ContextHandle.AddActors(Actors);
ContextHandle.AddHitResult(HitResult);
```

它内部找到真正的 Context，再调用 Context 的对应函数。这样业务代码不用管理 Context 的内存，也不用直接处理共享指针。

Handle 还有两个重要价值：

1. 多份 Spec 或临时变量可以安全共享同一份 Context；
2. Handle 可以统一处理有效性检查、复制和网络序列化。

---

<a id="section-3"></a>
## 三、MakeEffectContext 的创建流程

### 3.1 表面调用

项目中通过 ASC 创建 Context：

```cpp
FGameplayEffectContextHandle EffectContextHandle =
    SourceASC->MakeEffectContext();
```

### 3.2 引擎内部的主要流程

```text
UAbilitySystemComponent::MakeEffectContext
  -> UAbilitySystemGlobals::AllocGameplayEffectContext
  -> new FGameplayEffectContext
  -> 用原始指针构造 FGameplayEffectContextHandle
  -> Handle 将它放进 TSharedPtr
  -> Context.AddInstigator(OwnerActor, AvatarActor)
  -> 返回 Handle
```

因此 `MakeEffectContext()` 不只是分配一个空结构体。ASC 的 `AbilityActorInfo` 有效时，它还会自动写入：

- `Instigator = OwnerActor`；
- `EffectCauser = AvatarActor`；
- `InstigatorAbilitySystemComponent`；
- Instigator 和 EffectCauser 是否允许被网络复制的标志。

### 3.3 Aura 中为什么 Instigator 是 PlayerState

Aura 把玩家 ASC 放在 `AAuraPlayerState` 中，所以初始化 ActorInfo 时通常是：

```text
OwnerActor  = PlayerState
AvatarActor = AuraCharacter
```

所以调用 `MakeEffectContext()` 后：

```text
Instigator  = PlayerState
EffectCauser = AuraCharacter
```

这不是引擎随便选择的，而是 ASC 的 Owner/Avatar 架构自然产生的结果。

### 3.4 Instigator 与 EffectCauser 的区别

| 字段 | 含义 | 示例 |
|---|---|---|
| Instigator | 对效果负有逻辑责任、通常拥有 ASC 的 Actor | PlayerState、角色 |
| EffectCauser | 实际在世界中造成效果的 Actor | 角色、武器、投射物、陷阱 |

例如角色发射火球：

```text
Instigator   = 玩家 PlayerState 或角色
EffectCauser = 火球投射物（如果后续显式改成投射物）
```

当前 Aura 的 `MakeEffectContext()` 默认把 AvatarActor 作为 EffectCauser，所以最初是 AuraCharacter。

---

<a id="section-4"></a>
## 四、FGameplayEffectContext 能保存什么

下面是 14.1 中重点分析的字段。字段声明的精确修饰符可能随 UE 版本变化，但职责基本一致。

| 数据 | 常见类型 | 用途 |
|---|---|---|
| Instigator | `TWeakObjectPtr<AActor>` | 逻辑发起者，通常是 ASC Owner |
| EffectCauser | `TWeakObjectPtr<AActor>` | 实际造成效果的物理 Actor |
| AbilityCDO | `TWeakObjectPtr<UGameplayAbility>` | 负责本次效果的能力类默认对象 |
| AbilityInstanceNotReplicated | `TWeakObjectPtr<UGameplayAbility>` | 本地能力实例，不参与网络复制 |
| AbilityLevel | `int32` | 创建上下文时关联的技能等级 |
| SourceObject | `TWeakObjectPtr<UObject>` | 业务自定义的来源对象 |
| InstigatorAbilitySystemComponent | `TWeakObjectPtr<UAbilitySystemComponent>` | 发起者的 ASC |
| Actors | `TArray<TWeakObjectPtr<AActor>>` | 与本次效果有关的其他 Actor |
| HitResult | `TSharedPtr<FHitResult>` | 可选的命中/Trace 信息 |
| WorldOrigin | `FVector` | 效果发生或追踪开始的世界位置 |
| 位字段标志 | `uint8 : 1` | 是否有 Origin、是否复制某些引用等 |

### 4.1 为什么很多字段默认是空的

Context 是一个通用数据容器，不同 GameplayEffect 需要的信息不同：

- 属性初始化 GE 不需要 HitResult；
- 火球命中可能需要 HitResult；
- MMC 可能只需要 SourceObject；
- 某些效果需要额外关联一组 Actors；
- 有些 GE 根本不来自 GameplayAbility。

因此引擎只自动设置能从 ASC ActorInfo 确定的信息，其他字段由创建 Spec 的业务代码按需填写。

**Context 不需要被“全部填满”。只写入下游真正需要的数据。**

### 4.2 SourceObject 到底应该是什么

`SourceObject` 没有唯一固定含义，它是留给项目使用的通用 UObject 引用。例如可以是：

- 造成效果的角色；
- 投射物；
- 武器；
- 装备实例；
- 技能生成的数据对象。

Aura 的 `MMC_Max_Health` / `MMC_MaxMana` 会这样使用它：

```cpp
ICombatInterface* CombatInterface = Cast<ICombatInterface>(
    Spec.GetContext().GetSourceObject());

const int32 PlayerLevel = CombatInterface->GetPlayerLevel();
```

所以属性初始化时将 AvatarActor 放入 SourceObject，是为了让 MMC 能通过战斗接口获取等级。

这说明 SourceObject 的关键规则是：

> 写入方和读取方必须对“这里保存什么”达成一致。

---

<a id="section-5"></a>
## 五、项目中的实际使用流程

14.1 对应的 `AuraProjectileSpell.cpp` 中，现在有如下演示代码：

```cpp
FGameplayEffectContextHandle EffectContextHandle =
    SourceASC->MakeEffectContext();

EffectContextHandle.SetAbility(this);
EffectContextHandle.AddSourceObject(Projectile);

TArray<TWeakObjectPtr<AActor>> Actors;
Actors.Add(Projectile);
EffectContextHandle.AddActors(Actors);

FHitResult HitResult;
HitResult.Location = ProjectileTargetLocation;
EffectContextHandle.AddHitResult(HitResult);

const FGameplayEffectSpecHandle SpecHandle = SourceASC->MakeOutgoingSpec(
    DamageEffectClass,
    GetAbilityLevel(),
    EffectContextHandle);
```

### 5.1 每行代码做了什么

#### `MakeEffectContext()`

创建 Context，并自动设置 Instigator、EffectCauser 和发起者 ASC。

#### `SetAbility(this)`

把当前 GameplayAbility 与 Context 关联。引擎会记录：

- 能力 CDO；
- 当前能力实例（不复制）；
- 当前能力等级。

#### `AddSourceObject(Projectile)`

把投射物作为项目自定义来源对象保存。后续可以通过：

```cpp
UObject* SourceObject = ContextHandle.GetSourceObject();
```

取回它。

#### `AddActors(Actors)`

把一组相关 Actor 作为弱引用放入 Context。适合范围攻击、链式技能或需要记录多个参与者的效果。

#### `AddHitResult(HitResult)`

复制一份 `FHitResult` 到堆上，由 `TSharedPtr` 管理。后续可以取得命中位置、法线、命中 Actor、Trace Start 等信息。

### 5.2 这些演示字段是否都必须设置

不必须。字幕也明确说明这里有些数据只是为了观察 Context 能保存什么。

例如：

- SourceObject 已经是 Projectile；
- Actors 数组又只放了同一个 Projectile；
- HitResult 只手动填写了 Location，并不是真正 Trace 得到的完整命中结果。

真实项目应根据后续消费者的需求设置，避免重复和语义不清。

---

<a id="section-6"></a>
## 六、为什么要使用指针

### 6.1 对象值与指针的区别

直接保存对象值：

```cpp
FHitResult HitResult;
```

意味着这块变量本身包含完整 `FHitResult` 数据。

保存指针：

```cpp
FHitResult* HitResultPointer;
```

变量只保存另一个内存位置的地址。

### 6.2 Context 使用指针的原因

1. **字段是可选的**：没有命中信息时，指针可以为空；
2. **避免无条件占用较大空间**：只有需要时才分配 `FHitResult`；
3. **允许多个 Handle 共享同一份 Context**；
4. **Actor/UObject 由 UE 管理，Context 只需要引用它们，不应该拥有它们**；
5. **支持多态**：Handle 可以指向 `FGameplayEffectContext` 的自定义子类实例。

### 6.3 裸指针的问题

```cpp
FGameplayEffectContext* Context = new FGameplayEffectContext();
```

仅使用裸指针会产生两个问题：

- 谁负责 `delete Context`？
- Context 已被删除后，其他地方是否还拿着悬空地址？

智能指针的主要价值就是明确所有权并自动处理生命周期。

---

<a id="section-7"></a>
## 七、智能指针的核心：所有权与生命周期

理解智能指针时，不要先背函数；先回答一个问题：

> 这个指针是否拥有对象？它是否有权让对象继续存活？

### 7.1 强引用/拥有型指针

拥有型指针表示：“只要我还存在，对象就应该存在。”

`TSharedPtr` 是典型例子。它通过引用计数记录还有多少个共享拥有者。

### 7.2 弱引用/观察型指针

弱引用表示：“我想在对象仍然存在时访问它，但我不负责让它活着。”

`TWeakPtr` 和 `TWeakObjectPtr` 都属于观察型引用，但它们面对的是两套不同的对象管理系统。

### 7.3 生命周期示意

```text
拥有者 A ----强引用----> 对象
拥有者 B ----强引用----> 对象
观察者 C ----弱引用----> 对象

A 释放：对象仍活着，因为 B 还拥有它
B 释放：最后一个强引用消失，对象被销毁
C 仍存在：但弱引用已经失效，不能继续访问对象
```

弱引用不会延长对象生命周期，这正是“弱”的含义。

---

<a id="section-8"></a>
## 八、TSharedPtr：共享拥有普通 C++ 对象

### 8.1 基本行为

`TSharedPtr<T>` 用于普通 C++ 对象或结构体，内部有共享引用计数。

```cpp
TSharedPtr<FHitResult> SharedHit = MakeShared<FHitResult>();
```

也可以让第二个共享指针引用同一个对象：

```cpp
TSharedPtr<FHitResult> Another = SharedHit;
```

此时：

```text
共享引用计数 = 2
```

当两个 `TSharedPtr` 都离开作用域或被 `Reset()` 后，引用计数归零，`FHitResult` 自动销毁。

### 8.2 ContextHandle 内部为什么用它

概念上，Handle 内部类似：

```cpp
TSharedPtr<FGameplayEffectContext> Data;
```

复制 Handle 时，不需要复制整个 Context：

```cpp
FGameplayEffectContextHandle A = SourceASC->MakeEffectContext();
FGameplayEffectContextHandle B = A;
```

`A` 和 `B` 的 `TSharedPtr` 都拥有同一份 Context。只要至少一个 Handle 仍存在，Context 就不会被删除。

### 8.3 HitResult 为什么也用 TSharedPtr

`FHitResult` 是普通结构体，不是 UObject。Context 中的命中数据是可选的，所以只有调用 `AddHitResult` 时才创建：

```cpp
HitResult = MakeShared<FHitResult>(InHitResult);
```

好处：

- 没有命中信息时可以是空指针；
- 有数据时自动管理内存；
- Context 被复制/共享时，命中信息也能安全共享。

### 8.4 TSharedPtr 不应该管理 UObject

不要写：

```cpp
// 错误方向：不要用 TSharedPtr 管理 UObject 生命周期
TSharedPtr<AActor> Actor;
```

UObject/Actor 由 UE 垃圾回收与世界生命周期管理，不使用普通 C++ 共享引用计数管理。

---

<a id="section-9"></a>
## 九、TWeakPtr：观察普通 C++ 对象

### 9.1 TWeakPtr 配合 TSharedPtr 使用

`TWeakPtr<T>` 面向由 `TSharedPtr<T>` 管理的普通 C++ 对象。

```cpp
TSharedPtr<FMyData> SharedData = MakeShared<FMyData>();
TWeakPtr<FMyData> WeakData = SharedData;
```

WeakData 不增加共享引用计数。

### 9.2 为什么不能直接解引用

对象可能已经被最后一个 `TSharedPtr` 删除，所以使用前要 `Pin()`：

```cpp
if (TSharedPtr<FMyData> Pinned = WeakData.Pin())
{
    Pinned->DoSomething();
}
```

`Pin()` 尝试临时取得一个强引用：

- 对象还活着：返回有效 `TSharedPtr`；
- 对象已销毁：返回空 `TSharedPtr`。

### 9.3 常见用途

- 避免两个 `TSharedPtr` 互相引用造成循环；
- 缓存或观察一个对象，但不负责其生命周期；
- 异步回调中检查普通 C++ 对象是否仍存在。

---

<a id="section-10"></a>
## 十、TWeakObjectPtr：观察 UObject

### 10.1 14.1 源码真正使用的类型

EffectContext 中的 Actor、Ability、ASC 等都是 UObject 派生对象，因此使用的是：

```cpp
TWeakObjectPtr<AActor>
TWeakObjectPtr<UGameplayAbility>
TWeakObjectPtr<UAbilitySystemComponent>
```

不是普通的 `TWeakPtr<AActor>`。

### 10.2 它如何判断 UObject 是否还有效

`TWeakObjectPtr` 不通过 `TSharedPtr` 的引用计数工作。它记录 UObject 在 UE 全局对象系统中的弱身份信息。

当 Actor 被销毁或 UObject 被垃圾回收后：

- `TWeakObjectPtr` 不会阻止销毁；
- `IsValid()` 会返回 false；
- `Get()` 会返回 `nullptr`。

```cpp
TWeakObjectPtr<AActor> WeakActor = SomeActor;

if (WeakActor.IsValid())
{
    AActor* Actor = WeakActor.Get();
    Actor->DoSomething();
}
```

### 10.3 为什么 Context 必须弱引用 UObject

`FGameplayEffectContext` 是普通 C++ 结构体，并不一定作为 `UPROPERTY` 存放在一个可被垃圾回收扫描的 UObject 中。

如果它保存无法被 GC 识别的普通 UObject 裸指针：

```cpp
AActor* Actor;
```

Actor 被世界销毁后，Context 中可能仍留下一个悬空地址。访问这个地址会产生崩溃或未定义行为。

`TWeakObjectPtr` 可以在不拥有 Actor 的情况下检测它是否已经失效，适合 Context 这种“记录事件关系，但不应该延长 Actor 生命周期”的结构。

### 10.4 为什么不应该让一次历史效果阻止 Actor 销毁

假设火球 Context 保存了目标 Actor：

```text
火球命中 -> Context 记录目标
目标死亡 -> Actor 应该被销毁
```

Context 只是一次事件记录，不应该因为它仍被某个 Spec 保存，就让死亡 Actor 永远不能回收。因此用弱引用更符合业务语义。

---

<a id="section-11"></a>
## 十一、TWeakPtr 与 TWeakObjectPtr 的区别

这是本节最需要分清的地方。

| 对比项 | `TWeakPtr<T>` | `TWeakObjectPtr<T>` |
|---|---|---|
| 面向对象 | 普通 C++ 对象/结构体 | `UObject` 及其派生类 |
| 配套拥有者 | `TSharedPtr<T>` | UE UObject/GC/Actor 生命周期系统 |
| 是否增加强引用 | 否 | 否 |
| 对象如何死亡 | 最后一个 SharedPtr 消失 | GC 回收、Actor Destroy、引擎生命周期 |
| 取得可用对象 | `Pin()` 得到 `TSharedPtr` | `Get()` 得到 UObject 裸指针 |
| 检查有效 | `IsValid()` / `Pin()` | `IsValid()` |
| Context 中用途 | 原生类型一般未直接用它 | Instigator、EffectCauser、Ability、ASC、Actors |

记忆方法：

```text
普通 C++ 对象：TSharedPtr <-> TWeakPtr
UObject 对象：  UPROPERTY/TObjectPtr 或引擎拥有 <-> TWeakObjectPtr
```

### 11.1 字幕中“弱指针不参与引用计数”的准确理解

对于 `TWeakPtr`：

- 它确实不增加 `TSharedPtr` 的强引用计数；
- 最后一个 SharedPtr 消失后，对象立即被删除。

对于 `TWeakObjectPtr`：

- UObject 并不是靠 `TSharedPtr` 引用计数管理的；
- WeakObjectPtr 不形成 GC 强引用，不阻止 UObject 被回收。

两者结果都叫“不拥有对象”，但内部机制不同，不能把它们当成同一种模板替换使用。

---

<a id="section-12"></a>
## 十二、EffectContext 为什么同时使用 SharedPtr 和 WeakObjectPtr

乍看之下可能矛盾：Context 自己由共享指针拥有，Context 内部又使用弱指针。

实际上它们管理的是不同层次：

```text
FGameplayEffectContextHandle
  --TSharedPtr 强拥有--> FGameplayEffectContext（普通 C++ 结构体）

FGameplayEffectContext
  --TWeakObjectPtr 弱观察--> AActor / UObject / Ability / ASC
  --TSharedPtr 强拥有------> FHitResult（普通 C++ 结构体，可选数据副本）
```

### 12.1 各层所有权

| 对象 | 谁拥有它 | 为什么 |
|---|---|---|
| Context | 一个或多个 Handle 共享拥有 | Handle 活着时 Context 必须活着 |
| HitResult 副本 | Context 共享拥有 | 这是 Context 自己按需创建的数据 |
| Actor/UObject | 世界、GC 或其他 UE 系统拥有 | Context 只观察，不能决定其生死 |

这就是智能指针设计的核心：**不是所有指针都应该拥有它指向的对象。**

---

<a id="section-13"></a>
## 十三、Get、IsValid、Pin 和解引用

### 13.1 TSharedPtr

```cpp
TSharedPtr<FHitResult> Hit;

if (Hit.IsValid())
{
    Hit->Location;
    FHitResult* Raw = Hit.Get();
}
```

- `IsValid()`：是否持有对象；
- `Get()`：取得普通裸指针，但不转移所有权；
- `->`：像普通指针一样访问成员；
- `Reset()`：释放当前 SharedPtr 的拥有权。

### 13.2 TWeakPtr

```cpp
TWeakPtr<FMyData> Weak;

if (TSharedPtr<FMyData> Strong = Weak.Pin())
{
    Strong->Use();
}
```

不要先 `IsValid()`，过一段时间再使用原对象；更稳妥的方式是直接 `Pin()`，让返回的 SharedPtr 在当前代码块内保证对象存活。

### 13.3 TWeakObjectPtr

```cpp
TWeakObjectPtr<AActor> WeakActor;

if (AActor* Actor = WeakActor.Get())
{
    Actor->GetActorLocation();
}
```

也可以：

```cpp
if (WeakActor.IsValid())
{
    WeakActor->GetActorLocation();
}
```

但把 `Get()` 结果保存在局部变量中通常更清晰。

### 13.4 Handle Getter 为什么返回裸指针

例如：

```cpp
AActor* Instigator = ContextHandle.GetOriginalInstigator();
```

内部通常是对 `TWeakObjectPtr` 调用 `Get()`。返回裸指针并不表示调用方取得所有权，只是得到一个临时访问入口。

使用前仍要考虑：

```cpp
if (IsValid(Instigator))
{
    // 安全使用
}
```

---

<a id="section-14"></a>
## 十四、CDO、能力实例与 const_cast

### 14.1 什么是 CDO

CDO 是 Class Default Object，类默认对象。每个 UObject 类都有一份由引擎创建的默认对象，保存该类的默认属性。

概念上：

```cpp
UGameplayAbility* AbilityCDO = Ability->GetClass()->GetDefaultObject<UGameplayAbility>();
```

CDO 不是玩家当前正在运行的那一个能力实例，而是“这个能力类的默认模板”。

### 14.2 为什么同时保存 AbilityCDO 和 AbilityInstance

| 字段 | 作用 |
|---|---|
| AbilityCDO | 无论能力是否实例化，都能识别其类和读取默认配置；可用于需要复制的身份信息 |
| AbilityInstanceNotReplicated | 本机运行时的具体能力实例，可访问实例状态，但不能直接依赖它进行跨网络传输 |
| AbilityLevel | 单独保存本次能力等级 |

GameplayAbility 有不同实例化策略，所以 Context 不能只依赖某一个能力实例。

### 14.3 SetAbility 做了什么

`SetAbility(this)` 的主要结果：

```text
当前 Ability
  -> 保存本地 Ability Instance 弱引用
  -> 取得该类的 CDO 并保存
  -> 读取 Ability Level 并保存
```

### 14.4 const_cast 是什么

引擎函数可能接收：

```cpp
const UGameplayAbility* InAbility
```

但内部弱指针字段需要非 const UObject 类型，于是源码可能使用：

```cpp
const_cast<UGameplayAbility*>(InAbility)
```

`const_cast` 只改变编译器眼中的 const 限定，不会复制对象。

重要原则：

- 移除 const 后如果修改一个本来真正是 const 的对象，会产生未定义行为；
- 这里只是把地址保存为弱引用，并非鼓励业务代码随意移除 const；
- 日常代码应优先保持 const 正确性，不要因为编译错误就直接加 `const_cast`。

---

<a id="section-15"></a>
## 十五、复制、网络序列化和 Duplicate

### 15.1 Context 为什么需要 NetSerialize

GameplayEffect 可能从服务器传到客户端，普通内存指针地址在另一台机器上没有意义：

```text
服务器地址 0x12345678 != 客户端中的同一个 Actor
```

因此 Context 必须把允许传输的字段按网络规则序列化，再由接收端还原引用或数据。`NetSerialize` 决定哪些字段如何写入网络数据流。

### 15.2 bReplicate... 位字段

Context 会判断 Instigator、EffectCauser、SourceObject 是否具备网络复制条件，并用位字段记录是否发送。

概念写法：

```cpp
uint8 bReplicateInstigator : 1;
uint8 bReplicateEffectCauser : 1;
uint8 bReplicateSourceObject : 1;
uint8 bHasWorldOrigin : 1;
```

`: 1` 表示每个标志只占一个 bit。多个标志可以紧凑地放在一个或少数字节中。

### 15.3 为什么不能认为 Context 所有字段都会复制

- `AbilityInstanceNotReplicated` 名称已经说明不会复制；
- Actors、SourceObject 等是否传输受具体序列化规则与对象网络能力影响；
- 本地临时 UObject 不一定拥有网络 GUID；
- 弱指针有效不等于它能跨网络复制。

### 15.4 Duplicate 为什么重要

ContextHandle 复制通常只复制共享指针，所以两个 Handle 仍指向同一份 Context。

```text
Handle A ----\
              -> 同一 Context
Handle B ----/
```

如果需要一份互不影响的 Context，就要调用 `Duplicate()` 创建深拷贝。

自定义 Context 时尤其需要正确重写 `Duplicate()`：

- 复制自定义字段；
- 如果包含 `HitResult` 这类指针数据，要确认是否需要深拷贝；
- 避免修改副本时影响原始 Context。

---

<a id="section-16"></a>
## 十六、源码中的其他 C++ 语法

### 16.1 虚函数为什么重要

`FGameplayEffectContext` 中存在虚函数，例如获取 ScriptStruct、复制、网络序列化相关入口。这说明它被设计为可以继承扩展。

Aura 下一步可以创建：

```cpp
struct FAuraGameplayEffectContext : public FGameplayEffectContext
{
    bool bIsBlockedHit = false;
    bool bIsCriticalHit = false;
};
```

这样伤害执行计算可以把格挡/暴击写入 Context，AttributeSet 和伤害飘字再读取。

### 16.2 protected 字段

Context 字段大多是 `protected`，表示：

- 外部代码不能随意直接修改；
- Context 自身和子类可以访问；
- 外部应优先使用 `Add...`、`Set...`、`Get...` 函数。

这保证设置字段时能同时维护复制标志等关联状态。

例如不要只改 `EffectCauser` 而忘记 `bReplicateEffectCauser`；调用 `SetEffectCauser()` 会替你保持二者一致。

### 16.3 `const` 引用参数

```cpp
void AddHitResult(const FHitResult& InHitResult);
```

- `&`：不复制一份参数后再进入函数；
- `const`：函数承诺不修改调用方传入的 HitResult；
- 函数内部可以选择复制一份到自己的 SharedPtr 中。

### 16.4 可选布尔参数

```cpp
AddHitResult(HitResult, true);
AddActors(Actors, true);
```

类似 `bReset` 的参数决定添加前是否清除已有数据。默认 `false` 时通常表示追加或保留已有内容。

---

<a id="section-17"></a>
## 十七、什么时候应该往 Context 中放数据

### 17.1 适合放入的数据

同时满足以下特征时适合放入 Context：

- 属于**本次 GameplayEffect 应用**；
- 下游计算或表现阶段需要读取；
- 不是角色长期状态；
- 需要随 Spec 一起传递；
- 有明确的网络序列化策略。

例如：

- 是否格挡；
- 是否暴击；
- 命中位置和法线；
- 伤害来源武器；
- 径向伤害中心；
- 击退方向。

### 17.2 不适合放入的数据

- 玩家永久等级、经验等长期状态：应放在 PlayerState/AttributeSet；
- 全局职业平衡曲线：应放在 DataAsset/CurveTable；
- 仅创建 Spec 时需要的基础伤害：可用 SetByCaller；
- 大量无关 Actor 引用：会增加复杂度和网络成本；
- 只为方便调试、没有消费者的数据。

### 17.3 Context 与 SetByCaller 的选择

| 需求 | 更适合 |
|---|---|
| 传递一个由 Tag 标识的数值输入 | SetByCaller |
| 记录来源、命中或一次结算的元数据 | EffectContext |
| 保存持久属性 | AttributeSet |
| 保存可共享的平衡配置 | DataAsset / CurveTable |

---

<a id="section-18"></a>
## 十八、原生 Context 的限制

Aura 的 `ExecCalc_Damage` 已经计算出：

```cpp
const bool bBlocked = ...;
const bool bCriticalHit = ...;
```

但是原生 `FGameplayEffectContext` 没有对应字段。因此 `AttributeSet` 最终只收到伤害数值，无法知道：

- 本次攻击是否被格挡；
- 本次攻击是否暴击；
- 是否同时格挡并暴击。

这会限制伤害飘字的样式和后续 GameplayCue。

解决方向不是另建一个与 GAS 平行的全局变量，而是继承 Context：

```text
FGameplayEffectContext
  -> FAuraGameplayEffectContext
       -> bIsBlockedHit
       -> bIsCriticalHit
```

然后还需要配套完成：

1. 自定义结构体和 Getter/Setter；
2. 重写 `GetScriptStruct()`；
3. 重写 `Duplicate()`；
4. 实现自定义 `NetSerialize()`；
5. 在 `UAbilitySystemGlobals` 子类中重写 Context 分配函数；
6. 在配置中启用自定义 AbilitySystemGlobals；
7. 提供安全的 Context 类型转换辅助函数。

这些属于 14.1 之后的内容，但 14.1 阅读源码的目的就是理解为什么必须完成这一整套扩展。

---

<a id="section-19"></a>
## 十九、常见误区

### 误区 1：EffectContext 就是 GameplayEffect

错误。GE 是规则，Context 是本次规则执行时的背景信息。

### 误区 2：Context 中每个字段都必须设置

错误。只设置后续真正会消费的数据。

### 误区 3：WeakPtr 会让对象继续存活

错误。弱引用的定义就是不拥有对象、不延长生命周期。

### 误区 4：TWeakPtr 和 TWeakObjectPtr 是同一个东西

错误。前者配合 `TSharedPtr` 管理普通 C++ 对象，后者观察 UObject。

### 误区 5：TSharedPtr 可以管理 Actor

错误。Actor/UObject 由 UE 系统管理，不应交给普通共享指针删除。

### 误区 6：Get() 返回裸指针，所以智能指针失去作用

错误。`Get()` 只是临时取得访问地址，不转移所有权。智能指针仍负责生命周期或失效检测。

### 误区 7：弱对象指针有效就一定能网络复制

错误。本机对象有效与对象是否有网络身份是两件事。

### 误区 8：复制 Handle 会复制一份独立 Context

通常错误。Handle 内是共享指针，普通复制后仍共享同一 Context；需要独立副本时使用 `Duplicate()`。

### 误区 9：SourceObject 与 EffectCauser 永远相同

错误。EffectCauser 有明确的“实际造成效果者”语义；SourceObject 是项目自定义语义，可能是武器、投射物、角色或其他 UObject。

### 误区 10：使用 const_cast 只是为了消除编译错误

错误。`const_cast` 必须有清晰且安全的原因，普通业务代码不应滥用。

---

<a id="section-20"></a>
## 二十、速查表与记忆方法

### 20.1 指针选择速查

| 情况 | 常用类型 |
|---|---|
| 普通 C++ 对象，单独拥有 | `TUniquePtr<T>` |
| 普通 C++ 对象，多个地方共同拥有 | `TSharedPtr<T>` |
| 观察 SharedPtr 对象但不拥有 | `TWeakPtr<T>` |
| UObject 强引用，且希望 GC 跟踪 | `UPROPERTY() TObjectPtr<T>` |
| 观察 UObject 但不阻止销毁/回收 | `TWeakObjectPtr<T>` |
| 临时访问 UObject、不拥有 | `T*`，并检查有效性 |

### 20.2 Context 字段速查

```text
谁负责这次效果？       -> Instigator
谁在世界中实际造成？   -> EffectCauser
来自哪个技能类？       -> AbilityCDO
本地技能实例是什么？   -> AbilityInstanceNotReplicated
技能等级是多少？       -> AbilityLevel
项目自定义来源是什么？ -> SourceObject
还涉及哪些 Actor？     -> Actors
命中了哪里？           -> HitResult / WorldOrigin
```

### 20.3 最重要的三句话

1. **EffectContext 是随 GameplayEffectSpec 传递的一次性背景数据包。**
2. **Handle 用 `TSharedPtr` 拥有 Context；Context 用 `TWeakObjectPtr` 观察 UObject。**
3. **`TWeakPtr` 面向 SharedPtr 管理的普通对象，`TWeakObjectPtr` 面向 UE 管理的 UObject，二者不可混用。**

### 20.4 从创建到消费的完整流程

```text
Source ASC::MakeEffectContext
  -> 分配 Context
  -> 自动写入 Instigator / EffectCauser / Instigator ASC
  -> 业务按需 SetAbility / AddSourceObject / AddHitResult
  -> MakeOutgoingSpec(ContextHandle)
  -> GE Spec 携带 Context
  -> GE 应用到 Target ASC
  -> ExecCalc / AttributeSet 从 Spec.GetContext() 读取信息
  -> 根据伤害结果触发扣血、受击、死亡和飘字
```

一句话总结：

> `FGameplayEffectContext` 不是用来计算伤害的类，而是让伤害计算与后续结算知道“这次效果究竟是怎么来的”；智能指针则保证这份上下文可以安全共享，同时不会错误地控制 Actor 与 UObject 的生命周期。

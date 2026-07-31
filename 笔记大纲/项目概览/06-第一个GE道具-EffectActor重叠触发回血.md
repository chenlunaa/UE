# UE5 学习笔记 — 第六次提交

> 📦 Commit `f712823`：增加了一个回血道具（EffectActor）  
> 📅 日期：2026-06-08  
> 🎬 对应视频：4.4 通过重叠改变属性

---

## 目录

- [一、效果演员（AuraEffectActor）的设计思路](#section-3)
- [二、C++ 类创建与组件配置](#section-4)
  - [2.1 头文件定义](#section-5)
  - [2.2 构造函数：静态网格 + 球体碰撞](#section-6)
- [三、重叠回调绑定机制](#section-7)
  - [3.1 委托签名的查找方法](#section-8)
  - [3.2 BeginPlay 中绑定委托](#section-9)
- [四、通过 IAbilitySystemInterface 访问属性集](#section-10)
  - [4.1 接口转换获取 ASC](#section-11)
  - [4.2 获取指定类型的属性集](#section-12)
- [五、const_cast 的临时方案与局限性](#section-13)
  - [5.1 为什么要用 const_cast](#section-14)
  - [5.2 当前方案的三大局限](#section-15)
  - [5.3 TODO 标记：未来改用 GameplayEffect](#section-16)
- [六、蓝图化：创建生命药水](#section-17)
- [七、控制台验证属性变化](#section-18)
- [八、新增文件清单](#section-19)
- [九、知识点总结](#section-20)

---

<a id="section-3"></a>
## 一、效果演员（AuraEffectActor）的设计思路

在游戏中经常需要**可拾取的对象**来影响玩家属性（如血瓶、魔瓶）。本节创建一个通用的 `AAuraEffectActor` 类：

- 拥有一个**球体碰撞体积**用于检测重叠
- 拥有一个**静态网格**作为视觉表现
- 重叠时改变重叠者的属性值

> ⚠️ 改变属性的**正确方式**是通过 GameplayEffect，但本节先用直接修改的方式演示，目的是学习**如何访问其他 Actor 的 ASC 和属性集**。

---

<a id="section-4"></a>
## 二、C++ 类创建与组件配置

<a id="section-5"></a>
### 2.1 头文件定义

```cpp
// AuraEffectActor.h
class USphereComponent;

UCLASS()
class AURA_API AAuraEffectActor : public AActor
{
    GENERATED_BODY()
    
public:	
    AAuraEffectActor();

    UFUNCTION()
    virtual void OnOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
        UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFramSweep, 
        const FHitResult& SweepResult);

    UFUNCTION()
    virtual void EndOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
        UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

protected:
    virtual void BeginPlay() override;

private:
    UPROPERTY(VisibleAnywhere)
    TObjectPtr<USphereComponent> Sphere;

    UPROPERTY(VisibleAnywhere)
    TObjectPtr<UStaticMeshComponent> Mesh;
};
```

<a id="section-6"></a>
### 2.2 构造函数：静态网格 + 球体碰撞

```cpp
AAuraEffectActor::AAuraEffectActor()
{
    PrimaryActorTick.bCanEverTick = false;  // 不需要每帧 Tick

    Mesh = CreateDefaultSubobject<UStaticMeshComponent>("Mesh");
    SetRootComponent(Mesh);                  // 网格作为根组件

    Sphere = CreateDefaultSubobject<USphereComponent>("Sphere");
    Sphere->SetupAttachment(GetRootComponent());  // 球体附着在根上
}
```

**组件层级**：
```
Root (UStaticMeshComponent "Mesh")
  └── USphereComponent "Sphere"  ← 用于碰撞检测
```

---

<a id="section-7"></a>
## 三、重叠回调绑定机制

<a id="section-8"></a>
### 3.1 委托签名的查找方法

在 UE 中，要绑定一个函数到组件的重叠委托，必须知道**正确的函数签名**。查找方法：

1. 右键 `OnComponentBeginOverlap` → **Go to Definition**
2. 找到委托声明中的宏，例如 `FComponentBeginOverlapSignature`
3. 该宏展开后即为所需的参数列表：

```cpp
// OnComponentBeginOverlap 的回调签名
void(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
     UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, 
     bool bFromSweep, const FHitResult& SweepResult)
```

<a id="section-9"></a>
### 3.2 BeginPlay 中绑定委托

```cpp
void AAuraEffectActor::BeginPlay()
{
    Super::BeginPlay();
    
    // 绑定重叠开始事件
    Sphere->OnComponentBeginOverlap.AddDynamic(this, &AAuraEffectActor::OnOverlap);
    // 绑定重叠结束事件
    Sphere->OnComponentEndOverlap.AddDynamic(this, &AAuraEffectActor::EndOverlap);
}
```

- `AddDynamic` 用于绑定 UFUNCTION 到**动态多播委托**
- 第一个参数是对象指针，第二个是成员函数指针

---

<a id="section-10"></a>
## 四、通过 IAbilitySystemInterface 访问属性集

<a id="section-11"></a>
### 4.1 接口转换获取 ASC

```cpp
void AAuraEffectActor::OnOverlap(UPrimitiveComponent* OverlappedComponent, 
    AActor* OtherActor, ...)
{
    // 尝试将重叠的 Actor 转换为 IAbilitySystemInterface
    if (IAbilitySystemInterface* ASCInterface = Cast<IAbilitySystemInterface>(OtherActor))
    {
        // 通过接口获取 ASC，再获取属性集
        const UAuraAttributeSet* AuraAttributeSet = Cast<UAuraAttributeSet>(
            ASCInterface->GetAbilitySystemComponent()
                ->GetAttributeSet(UAuraAttributeSet::StaticClass())
        );
        // ...
    }
}
```

**关键流程**：

```
OtherActor  →  Cast<IAbilitySystemInterface>  →  GetAbilitySystemComponent()
                                                      ↓
                                              GetAttributeSet(类型)
                                                      ↓
                                              UAuraAttributeSet*
```

<a id="section-12"></a>
### 4.2 获取指定类型的属性集

```cpp
GetAttributeSet(UAuraAttributeSet::StaticClass())
```

- `GetAttributeSet` 是模板函数，需要传入 `TSubclassOf<UAttributeSet>` 类型参数
- `StaticClass()` 返回 UClass 指针，用于在 ASC 中查找对应的属性集实例
- 返回的是 **const 指针**，这是 GAS 的设计保护——属性集不应被外部直接修改

---

<a id="section-13"></a>
## 五、const_cast 的临时方案与局限性

<a id="section-14"></a>
### 5.1 为什么要用 const_cast

`GetAttributeSet` 返回 `const UAuraAttributeSet*`，但我们需要调用 `SetHealth()` 来修改值：

```cpp
// ❌ 直接调用会报错：const 指针不能调用非 const 成员函数
AuraAttributeSet->SetHealth(AuraAttributeSet->GetHealth() + 25.f);

// ✅ 临时方案：const_cast 去掉 const 限定
UAuraAttributeSet* MutableAuraAttributeSet = const_cast<UAuraAttributeSet*>(AuraAttributeSet);
MutableAuraAttributeSet->SetHealth(AuraAttributeSet->GetHealth() + 25.f);
Destroy();  // 拾取后销毁
```

> ⚠️ **const_cast 是 C++ 中应该尽量避免的操作**，它破坏了 GAS 的封装保护。

<a id="section-15"></a>
### 5.2 当前方案的三大局限

| 局限 | 说明 |
|------|------|
| **类型耦合** | 必须硬编码 `UAuraAttributeSet`，无法用于其他属性集 |
| **属性耦合** | 只能改 Health，改其他属性需要写新代码 |
| **破坏封装** | const_cast 绕过了 GAS 的保护机制 |
| **不可复用** | 每种药水都需要单独写逻辑 |

<a id="section-16"></a>
### 5.3 TODO 标记：未来改用 GameplayEffect

```cpp
// TODO: 更改这个以应用 GameplayEffect
// 目前使用 const_cast 作为 hack
```

正确的方式是：EffectActor 持有一个 `TSubclassOf<UGameplayEffect>` 变量，重叠时调用 `ASC->ApplyGameplayEffectToSelf()`。这样：

- 不需要知道属性集的类型
- 不需要 const_cast
- 一个 EffectActor 可以通过配置应用任意效果

---

<a id="section-17"></a>
## 六、蓝图化：创建生命药水

1. 在 `Content/Blueprints/Actor/` 下创建蓝图类 `BP_HealthPotion`
2. 父类选择 `AuraEffectActor`
3. 设置 `Mesh` 的静态网格为药水瓶资产
4. 调整缩放（Scale = 0.2），增大球体半径以匹配视觉大小

---

<a id="section-18"></a>
## 七、控制台验证属性变化

运行游戏后，按 `~` 打开控制台，输入：

```
showdebug abilitysystem
```

走到药水旁边拾取后，观察 Health 值从 100 变为 125，验证成功。

---

<a id="section-19"></a>
## 八、新增文件清单

| 文件 | 说明 |
|------|------|
| `Source/Aura/Public/Actor/AuraEffectActor.h` | 效果演员头文件 |
| `Source/Aura/Private/Actor/AuraEffectActor.cpp` | 效果演员实现 |
| `Content/Blueprints/Actor/BP_HealthPotion.uasset` | 生命药水蓝图 |

---

<a id="section-20"></a>
## 九、知识点总结

| 知识点 | 说明 |
|--------|------|
| **委托签名查找** | 右键 → Go to Definition → 找宏定义，可获取精确的回调参数列表 |
| **AddDynamic** | 将 UFUNCTION 绑定到动态多播委托 |
| **IAbilitySystemInterface** | 通过 Cast 获取任意 Actor 的 ASC，无需知道具体类型 |
| **GetAttributeSet\<T\>** | 从 ASC 中按类型查找属性集，返回 const 指针 |
| **const_cast** | 临时绕过 const 保护，不推荐；后续用 GameplayEffect 替代 |
| **showdebug abilitysystem** | 控制台命令，实时查看 ASC 和属性值 |

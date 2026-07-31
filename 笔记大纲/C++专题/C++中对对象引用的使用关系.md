# C++ 成员访问符号详解：`->`、`.`、`::`

## 快速对照表

| 符号   | 左边是什么            | 访问方式 | 用途             |
| ---- | ---------------- | ---- | -------------- |
| `->` | 指针 `T*`          | 间接访问 | 通过指针访问对象的成员    |
| `.`  | 实例 `T` / 引用 `T&` | 直接访问 | 通过对象本身访问成员     |
| `::` | 类名 / 命名空间        | 静态访问 | 访问静态成员、枚举、嵌套类型 |

---

## 1. `->` 指针访问

**适用场景：左边是指针类型（`T*`）**

```cpp
// 声明指针
AAuraCharacter* Character = GetCharacter();

// 用 -> 访问成员函数和变量
Character->GetHealth();
Character->AbilitySystemComponent;

// this 也是指针
this->GetPlayerState<AAuraPlayerState>();
```

### UE 常见示例

```cpp
// PlayerState 是指针
AAuraPlayerState* PS = GetPlayerState<AAuraPlayerState>();
PS->GetPlayerName();
PS->SetPlayerName("Hero");

// AbilitySystemComponent 是指针
AbilitySystemComponent->InitAbilityActorInfo(this, this);
AbilitySystemComponent->GiveAbility(SomeAbilitySpec);

// World 是指针
UWorld* World = GetWorld();
World->SpawnActor<AActor>(ActorClass, Location, Rotation);
```

### 本质

```cpp
// 以下两者等价
pointer->Member;
(*pointer).Member;  // 先解引用得到对象，再用 . 访问
```

---

## 2. `.` 对象/引用访问

**适用场景：左边是对象实例（`T`）或引用（`T&`）**

```cpp
// 栈上对象
AAuraCharacter Character;
Character.GetHealth();

// 引用
AAuraCharacter& CharacterRef = *GetCharacter();
CharacterRef.GetHealth();

// 结构体实例
FVector Location = GetActorLocation();
Location.X = 100.0f;
Location.Y = 200.0f;

// 返回值是对象（不是指针）
FString Name = GetName();
Name.Len();
```

### UE 常见示例

```cpp
// FGameplayTag 是结构体
FGameplayTag Tag = FGameplayTag::RequestGameplayTag(FName("State.Dead"));
Tag.IsValid();

// FGameplayAbilitySpec
FGameplayAbilitySpec AbilitySpec(AbilityClass, Level);
AbilitySpec.Handle;

// 解引用后转 .
AAuraPlayerState* PS = GetPlayerState<AAuraPlayerState>();
(*PS).GetPlayerName();  // 等价于 PS->GetPlayerName()
```

---

## 3. `::` 类/命名空间访问

**适用场景：左边是类名、结构体名或命名空间**

### 3.1 访问静态成员函数/变量

```cpp
// 获取类的 UClass 对象
UClass* CharacterClass = AAuraCharacter::StaticClass();

// 数学工具函数
float ClampedValue = FMath::Clamp(Health, 0.0f, MaxHealth);
float RandomValue = FMath::RandRange(0, 100);

// 日志
UE_LOG(LogTemp, Warning, TEXT("Health: %f"), Health);
```

### 3.2 访问枚举

```cpp
// GameplayCue 事件类型
EGameplayCueEvent::OnActive;
EGameplayCueEvent::WhileActive;
EGameplayCueEvent::Removed;
EGameplayCueEvent::Executed;

// 碰撞通道
ECC_Pawn;
ECC_WorldStatic;
ECC_GameTraceChannel1;
```

### 3.3 访问嵌套类型

```cpp
// UAbilitySystemComponent 内部定义的结构体
UAbilitySystemComponent::FAbilitySpec;
UAbilitySystemComponent::FGameplayAbilitySpecHandle;

// TMap/TArray 等容器
TArray<AActor*>::ElementType;
```

### 3.4 访问命名空间

```cpp
// UE 命名空间
::IsValid(Object);  // 全局命名空间
FMath::Sin(Angle);
UKismetSystemLibrary::PrintString(this, TEXT("Hello"));
```

---

## UE 中的综合示例

```cpp
void AAuraCharacter::InitAbilityActorInfo()
{
    // :: 访问类的静态函数
    UClass* PlayerStateClass = AAuraPlayerState::StaticClass();

    // -> 指针访问
    AAuraPlayerState* AuraPlayerState = GetPlayerState<AAuraPlayerState>();

    // -> 继续链式访问
    AuraPlayerState->GetAbilitySystemComponent();

    // . 访问返回的结构体
    FGameplayTag Tag = FGameplayTag::RequestGameplayTag(FName("State.Dead"));
    bool bIsValid = Tag.IsValid();

    // :: 访问枚举
    if (CueEventType == EGameplayCueEvent::OnActive)
    {
        // -> 指针访问
        AbilitySystemComponent->InitAbilityActorInfo(AuraPlayerState, this);
    }
}
```

---

## 常见错误

```cpp
// ❌ 错误：实例用 ->
AAuraCharacter Character;
Character->GetHealth();  // 编译错误！

// ✅ 正确：实例用 .
Character.GetHealth();

// ---

// ❌ 错误：指针用 .
AAuraCharacter* Character = GetCharacter();
Character.GetHealth();  // 编译错误！

// ✅ 正确：指针用 ->
Character->GetHealth();

// ---

// ❌ 错误：类名用 -> 或 .
AAuraCharacter::GetHealth();  // 错误！非静态成员

// ✅ 正确：类名用 :: 访问静态成员
AAuraCharacter::StaticClass();
```

---

## 一句话记忆

> **指针用 `->`，实例用 `.`，类/命名空间用 `::`**

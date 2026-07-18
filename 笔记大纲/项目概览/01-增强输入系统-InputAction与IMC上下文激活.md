## 一、 增强输入系统的初始化与注入

在 UE5 中，激活一套输入方案需要获取**本地玩家子系统**，并将输入映射上下文（IMC）注入其中。

### 1. 激活 IMC 上下文

```
// 获取本地玩家的增强输入子系统（大管家）
UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetLocalPlayer());
check(Subsystem);

// 将 IMC 注入到输入系统中，参数 0 代表优先级（Priority）
Subsystem->AddMappingContext(AuraContext, 0);
```

- **核心机制**：`ULocalPlayer::GetSubsystem` 是 UE5 特有的子系统获取方式。`AddMappingContext` 负责把我们在编辑器里配好的按键资产（`AuraContext`）正式应用到玩家身上。

### 2. 绑定输入动作到 C++ 函数

```
// 将默认的 InputComponent 转换为增强输入组件
UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(InputComponent); 

// 将 IA_Move 动作与底层的 Move 回调函数进行绑定
EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AAuraPlayerController::Move);
```

- **`CastChecked`**：安全检查式类型转换。如果当前项目没有开启增强输入组件（默认是开启的），转换失败会直接触发崩溃，便于在开发阶段定位错误。

- **`ETriggerEvent::Triggered`**：代表持续触发（如长按 WASD 键），这会让绑定的 `Move` 函数在按键按下期间每帧都被调用。

## 二、 俯视角（RPG/MOBA）鼠标模式配置

为了让游戏像《暗黑破坏神》或 League of Legends 那样显示鼠标，并且点击画面时不会把鼠标锁死隐藏，使用了混合输入模式。

```
bShowMouseCursor = true;                  // 显示鼠标光标
DefaultMouseCursor = EMouseCursor::Default; // 设置鼠标的样式为操作系统默认的箭头样式

FInputModeGameAndUI InputModeData;        // 虚幻三种输入模式之一：Game 和 UI 混合模式
InputModeData.SetLockMouseToViewportBehavior(EMouseLockMode::DoNotLock); // 鼠标不要被锁死在游戏窗口内
InputModeData.SetHideCursorDuringCapture(false);                         // 点击游戏画面时不要隐藏鼠标
SetInputMode(InputModeData);              // 正式应用规则
```

## 三、 基于相机朝向的水平面移动算法（核心数学）

这是全篇最关键的逻辑。传统的 FPS 游戏直接获取相机的向前向量，但在俯视角或 3D 游戏中，相机通常是向下倾斜（俯视）的。如果直接用相机的 Forward 向量，角色就会往地底下钻。

```
// 1. 从输入包裹中提取 Vector2D 数据（X 对应左右，Y 对应前后）
const FVector2D InputAxisVector = InputActionValue.Get<FVector2D>();

// 2. 获取当前控制器的旋转角度（包含 Pitch俯仰、Yaw偏航、Roll翻滚）
const FRotator Rotation = GetControlRotation();

// 3. 【核心】只保留偏航角（Yaw），将俯仰角和翻滚角强制清零！
const FRotator YawRotation(0.f, Rotation.Yaw, 0.f);

// 4. 通过旋转矩阵，提取出当前朝向在绝对水平面上的“正前”和“正右”单位向量
const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X); // UE中 X 轴代表正前方
const FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);   // UE中 Y 轴代表正右方

// 5. 应用移动输入到 Pawn
if (APawn* ControlledPawn = GetPawn<APawn>()) {
    ControlledPawn->AddMovementInput(ForwardDirection, InputAxisVector.Y); // 键盘 W/S 的输入
    ControlledPawn->AddMovementInput(RightDirection, InputAxisVector.X);   // 键盘 A/D 的输入
}
```

### 💡 为什么必须通过 `FRotationMatrix(YawRotation).GetUnitAxis` 获取方向？

- **消除俯仰角（Pitch）的影响**：将角度过滤为 `(0, Yaw, 0)` 后，角色计算出来的“前方”就永远贴合地面，绝对不会因为镜头朝下看而产生向地下的力。

- **保证方向的绝对正确**：无论关卡中的相机如何旋转、缩放，通过这个旋转矩阵提取出的 `EAxis::X` 永远是**屏幕视角的绝对前方**。当玩家按 W 时，角色就会非常自然地朝着屏幕上方（视角前方）奔跑。

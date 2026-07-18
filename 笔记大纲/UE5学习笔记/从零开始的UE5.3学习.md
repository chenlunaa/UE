# 从零开始的UE5.3学习

## 1.新建一个蓝图类

在内容侧滑菜单中新建一个文件夹BP并且右键空白处新建一个蓝图类命名为test。在蓝图类的事件图表中可以编辑蓝图，其中默认会有三个模块：

                                                    <img src="file:///C:/Users/chenlun/Pictures/Screenshots/屏幕截图%202026-05-10%20103152.png" title="" alt="块" data-align="center">

第一个模块是事件开始运行模块只要开始运行关卡就会自动执行

<img src="file:///C:/Users/chenlun/Pictures/Screenshots/屏幕截图%202026-05-10%20103323.png" title="" alt="1" data-align="center">

第二个模块是重叠开时运行，当有物体与蓝图重叠时开始运行

<img src="file:///C:/Users/chenlun/Pictures/Screenshots/屏幕截图%202026-05-10%20103552.png" title="" alt="1" data-align="center">

第三个是Tick事件是每一帧这个逻辑都会执行

**逻辑框为虚化是因为后面没有衔接代码，在编译保存完蓝图文件后必须要在关卡中把蓝图类拖动到关卡当中才能够执行。将三角箭头拖动后可以创建后续逻辑的函数**

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-19-15-34-image.png" title="" alt="" data-align="center">

在设置好函数后可以根据函数所需要的参数设置变量并且传入进去，<u>要先编译才能在右侧的默认值中修改变量的值</u>

## 2.创建一个可以移动的角色

* 准备工作：
  
  1.需要在内容目录下创建一个文件夹Code这个表明我们的代码逻位置
  
  2.在Code中需要创建一个为Character的文件夹，里面存放我们角色的相关信息，还需要创建一个input文件夹，用来获取我们键盘输入对应角色需要做出的动作。
  
  <img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-19-44-04-image.png" title="" alt="" data-align="center">
  
  3.Character文件夹中需要创建两个蓝图类，一个为蓝图游戏模式基础 -n BP_gamemode，一个为蓝图角色-n BP_character。这是因为关卡的整体运行是基于一种游戏模式的，而BP_character就是我们实现角色移动功能的代码段。在BP_gamemode中只需要修改类中的默认pawn类，把它改成我们所创建的蓝图角色，并且再关卡设置的项目设置中，在<u><地图和模式></u>中把默认游戏模式修改为我们的BP_gamemode，BP_character的编写最后再说。
  
  <img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-19-48-07-image.png" title="" alt="" data-align="center">
  
  <img title="" src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-19-51-26-image.png" alt="" data-align="center" width="466">

        4.input文件就需要我们规定好角色是怎么移动的了，需要创建输入类的输入操作-n         IA_move2和输入映射情景-n IMC_input

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-19-52-20-image.png" title="" alt="" data-align="center">

        在IA_move2中规定移动的方式，这里修改操作中的<u>值类型</u>为Vector2D，其他不用修          改，这样表明人物是以二维坐标移动的，我们计划用WASD来控制人物的移动。

<img title="" src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-19-54-12-image.png" alt="" data-align="center">

        在IMC_input中，我们在映射中规定移动的键盘映射功能，新建映射并且选择为我们刚          才创建的IA_move2，对其新增属性WASD并且规定为键盘录入。其中UE默认只有一个         轴，我们把它作为W的前进轴，所以AD需要换一个轴，因此在S中S的方向与D相反，我          们需要创建一个修改器，把S变成否定，代表取反。在A中我们新增两个修改器，第一个         是拌合输入轴值，用来定义一个新的轴，另一个就是否定用来走向相反方向，D也同理

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-19-58-05-image.png" title="" alt="" data-align="center">

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-20-00-51-image.png" title="" alt="" data-align="center">

* 至此我们就完成了创意一个可以移动的角色的准备工作，只需要在BP_character中增加函数功能来让角色绑定好我们所写的文件就好了。BP_character编写如下：

    首先我们由之前新建蓝图类的情况得知会创建新的3个模块，我们的角色是运行直接创建     的，所以把后两个删除就行，然后右键创建 get player controller这个函数帮我们创建角     色控制系统，下一个是get enhance...这里表明用增强的方式来关联我们刚才的映射。

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-20-08-27-image.png" title="" alt="" data-align="center">

       在创建一个add maping context用来关联上下文，也就是连接Event BeginPlay达到一开        始就创建的目的。连线方式如下： (注意Add中要增加一个资产，就算我们刚才的        IMC_input)

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-20-09-57-image.png" title="" alt="" data-align="center">

        现在我们只是创建了一个角色，却没有把刚才配置的行为逻辑关联起来，我们还需要增         加移动逻辑，新建IA，找到刚才我们写的IA_move2，新建后展开，其中的Action Value         就表示了我们刚才写的WASD的返回二维向量

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-20-14-35-image.png" title="" alt="" data-align="center">

        所以我们需要把这个二维向量分解成XY轴的具体向量来控制移动，需要用到Break          Vector2D来分解，再增加一个Add Movement input来实现我们的移动，这个函数是实         现我们控制轴的移动的，绿色的Scale Value就是我们传入的分解的XY值。

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-10-20-16-19-image.png" title="" alt="" data-align="center">

        由于我们的移动不是固定方向的，是由我们主视角的方向来决定了，所以还要获取我们         主视角的方向并且传给AMI的World Direction需要用到Get Actor Forward Vector，对         于Y轴的移动也是同理，下面是连线图：

![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-10-20-19-22-image.png)

我们完成了角色移动的创建逻辑，角色可以前进后退左右移动了。

## 2.创建一个可以随着鼠标移动的视角

在input文件夹中新增一个输入IA_Look2，与IA_move2相同，修改为vector2d类型，然后在IMC_input中新增一个映射，把IA_Look2选中并且新增修改器否定，因为默认情况鼠标向上移动是下降，向下移动是视角上升，左右移动反而是正常的。所以我们需要把Y轴的反转打开，将索引打开可以勾选作用域。

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-11-20-57-30-image.png" title="" alt="" data-align="center">

基本的添加功能完成了，现在同样跟添加移动功能一样要到BP_character中程序化

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-11-20-58-29-image.png" title="" alt="" data-align="center">

这里右键输入IA_...找到我们刚才写的IA_Look2，，同样把值分为XY两个方向，后续的添加输入功能与移动不一样，因为移动只需要有一个方向，但是旋转是需要轴的，绕XYZ轴进行旋转的效果是不一致的。绕X轴旋转就是左右,Y轴旋转就是上下。在UE中X轴旋转的函数是Add Yaw Input,Y轴的旋转是Add Pitch Input。添加完后我们就实现了视角绕轴旋转的功能，就可以控制人物的运动了。(注释是框选区域后键盘C键)

## 3.创建一个可以旋转的门（单向）

### Step 1：准备门资产与碰撞设置

1. 在素材库 **Fab** 中导入门的相关资产（纹理、材质、网格体、门框等）。

2. **设置碰撞（关键）**：双击打开门的静态网格体资产。在编辑器左上角将“查看模式”改为 **玩家碰撞 (Player Collision)**，检查是否有绿色线框（碰撞体）。如果没有，在右侧面板的 **凸包分解 (Convex Decomposition)** 中点击**应用 (Apply)** 自动创建碰撞。*（如果漏掉这一步，角色在游戏中就能直接穿模穿过门）*。

### Step 2：构建蓝图组件

1. 在 `Code/Door` 文件夹下，新建一个**Actor蓝图类**，命名为 `BP_Door`。

2. 进入蓝图编辑器，在左侧组件面板中点击 **添加 (Add)**，添加两个**静态网格体 (Static Mesh)** 组件，分别命名为 `DoorFrame`（门框）和 `Door`（门扇）。<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-15-17-06-24-image.png" title="" alt="" data-align="center">3. 在右侧细节面板中，为它们指定刚才导入的 Fab 网格体资产，并调整好门扇在门框中的相对位置。<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-15-17-08-32-image.png" title="" alt="" data-align="center">4.**添加触发区域**：继续点击添加 **盒体碰撞 (Box Collision)** 组件。在细节面板中调整其**盒体半高/半径 (Box Extent)**，使其覆盖门前后的交互区域。*（注意：调整范围时要修改 Box Extent 属性，不要直接缩放 Transform 的 Scale，否则会导致碰撞体变形）*。![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-15-17-12-08-image.png)

### Step 3：编写开关门逻辑

1. 选中 Box 组件，在右侧细节面板最下方找到并点击 **On Component Begin Overlap**（组件开始重叠时）和 **On Component End Overlap**（组件结束重叠时）的绿色加号，将它们添加至事件图表。

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-15-17-15-38-image.png" title="" alt="" data-align="center">

        上方的是开始时的主线，下方是结束时的主线。

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-15-17-16-58-image.png" title="" alt="" data-align="center">

2. 在事件图表中右键搜索并添加 **Add Timeline（添加时间轴）** 组件，双击进入时间轴面板：
- 点击左上角 **f** 图标添加一条**浮点型轨道**，命名为 `Opendoor`。

- 将时间轴的 **长度 (Length)** 设置为 `2.0` 秒。

- 右键曲线添加两个关键帧：`(0, 0)` 和 `(1, 90)`。代表开始时旋转为0，1秒时旋转到90度，剩下1秒保持开启，实现延时关门效果。

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-15-17-21-43-image.png" title="" alt="" data-align="center">

3. 回到事件图表，将 **Begin Overlap** 连接到时间轴的 **Play（播放）**，将 **End Overlap** 连接到时间轴的 **Reverse（倒回）**。

4. 拉出时间轴的 **Update（更新）** 执行引脚，连接到 **Set Relative Rotation（设置相对旋转）** 节点，目标指定为 `Door` 组件。

5. 将 `OpenValue` 引脚连接到旋转体的 **Z 轴 (Yaw)**。*（可以右键旋转引脚选择“分割结构体引脚”单独连接 Z 轴）*。

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-15-17-27-32-image.png" title="" alt="" data-align="center">

![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-15-17-28-51-image.png)

## 4.创建一个双向旋转的门

### 核心原理

单向门无论玩家从哪一侧进入都会朝固定方向开。为了实现“人推门永远向前开”的效果，我们需要计算**玩家相对于门的位置关系**。

在视口中跳到顶视图，发现进入门是Y轴，所以我们需要获取门的Y轴坐标以及角色的Y轴坐标来控制门的旋转，调整旋转角度发现门旋转90°时，朝向是Y轴正半轴，所以由人指向门的向量应该是正的，另一边则是反的，现在我们可以去调整蓝图了。

详细步骤

1. **获取位置坐标**：在事件图表中添加 **Get Actor Location** 获取门的世界坐标；添加 **Get Player Pawn** 并拉出 **Get Actor Location** 获取玩家的世界坐标。

2. **计算相对方向向量**：添加减法节点（**Subtract**，类型设为 Vector），用**门的坐标减去玩家的坐标**（向量终点为门，起点为玩家），得到从玩家指向门的向量。

3. 减法运算上方是**被减数**下方是**减数**

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-16-09-42-04-image.png" title="" alt="" data-align="center">

4. **分解标量**：使用 **Break Vector（分解向量）** 将结果拆分为 X、Y、Z。根据我们在顶视图中的观察，这里我们需要判断 Y 轴的数值。

5. **条件分支判断**：使用 **Greater（大于）** 节点判断 `Y > 0` 是否成立，并将返回的布尔值连接到 **Branch（分支）** 节点。

6. **动态改变旋转方向**：
   
   - 新建一个浮点型变量 `Direction`。
   
   - 当 Branch 为 **True** 时，使用 **Set Direction** 设置为 `1.0`；
   
   - 当 Branch 为 **False** 时，设置 `Direction` 为 `-1.0`。

7. **注入时间轴**：在时间轴输出的 `OpenValue` 后面连接一个 **Multiply（乘法）** 节点，将轨道值乘以 `Direction` 变量，然后再传给 `Set Relative Rotation` 的 Z 轴。至此这个门的逻辑就造好了。下面是总的接线：

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-16-09-57-29-image.png" title="" alt="" data-align="center">

上述方法在门处于默认朝向时工作完美。但如果将门在关卡中旋转90度摆放，基于世界坐标的 `Y > 0` 判断就会失效。更完美的做法是使用 **Transform Location（将世界坐标转换为局部坐标）** 节点，将玩家的绝对位置转换为相对于 `BP_Door` 的局部位置，此时只需要判断局部坐标的 X 或 Y 正负即可，该逻辑将不受门自身摆放角度的影响！

## 5.常见的流程控制节点

## 1. Branch（分支）

- **作用**：最基础的条件判断节点（类似于代码中的 `if-else`）。根据传入的布尔值决定程序走向。

- **输入**：
  
  - `Exec`（白色三角）：触发该节点执行。
  
  - `Condition`（布尔型）：判断条件（True 或 False）。

- **输出**：
  
  - `True`（白色三角）：当 Condition 为真时执行此分支。
  
  - `False`（白色三角）：当 Condition 为假时执行此分支。

## 2. Flip Flop（交替开关）

- **作用**：如同一个拉线开关。第一次调用执行 A，第二次调用执行 B，第三次又执行 A，以此类推，在两个状态间反复横跳。

- **输入**：
  
  - `Exec`：触发该节点。

- **输出**：
  
  - `A`（白色三角）：第 1, 3, 5... 次调用时执行。
  
  - `B`（白色三角）：第 2, 4, 6... 次调用时执行。
  
  - `Is A`（布尔型）：输出当前执行的是不是 A 状态（方便后续逻辑判断）。

## 3. Delay（延迟）

- **作用**：让当前的逻辑暂停执行指定的时间，时间到了之后再继续向后执行。

- **输入**：
  
  - `Exec`：触发延迟。
  
  - `Duration`（浮点型）：延迟的时间（单位：秒）。

- **输出**：
  
  - `Completed`（白色三角）：延迟时间结束后触发。

## 4. Do Once（执行一次）

- **作用**：限制其后的逻辑只能执行一次。通常用于防止玩家重复触发某个事件（如只播放一次的过场动画、只能开启一次的宝箱）。

- **输入**：
  
  - `Exec`：尝试触发后面的逻辑。
  
  - `Reset`：重置该节点，使其可以再次被执行一次。
  
  - `Start Closed`（布尔型）：若勾选，默认初始状态是关闭的，必须先调用一次 Reset 激活它，Exec 才能通过。

- **输出**：
  
  - `Completed`（白色三角）：第一次被 Exec 触发时执行。

## 5. Sequence（序列）

- **作用**：按顺序执行一系列逻辑。它**不是**同时执行（并非多线程），而是严格按照 0、1、2 的引脚顺序瞬间执行完毕，主要用于让蓝图线条更加整洁。

- **输入**：
  
  - `Exec`：触发序列开始。

- **输出**：
  
  - `Then 0`（白色三角）：最先执行。
  
  - `Then 1`（白色三角）：Then 0 的逻辑执行完（或者连线延伸完）后立即执行。
  
  - *（可通过“Add pin”按钮无限添加 `Then N`）*

## 6. For Loop（循环）

- **作用**：在指定的索引范围内，重复执行某段逻辑。

- **输入**：
  
  - `Exec`：启动循环。
  
  - `First Index`（整型）：循环开始的索引（通常是 0）。
  
  - `Last Index`（整型）：循环结束的索引（包含该值）。

- **输出**：
  
  - `Loop Body`（白色三角）：每次循环时都会执行的逻辑。
  
  - `Index`（整型）：当前正在执行的循环索引值。
  
  - `Completed`（白色三角）：当整个循环结束后（所有次数跑完）触发。

## 7. Gate（门）

- **作用**：像一扇门一样，可以控制一段逻辑是否能够通过。你可以自由地打开、关闭或在通过时顺便关闭它。

- **输入**：
  
  - `Enter`（白色三角）：尝试通过这扇门。如果门是开的，逻辑就能通过；如果是关的，则被拦截。
  
  - `Open`（白色三角）：把门打开。
  
  - `Close`（白色三角）：把门关闭。
  
  - `Toggle`（白色三角）：切换门的状态（开变关，关变开）。
  
  - `Start Closed`（布尔型）：设置这扇门默认是开启还是关闭状态。

- **输出**：
  
  - `Exit`（白色三角）：当门处于打开状态，且 `Enter` 被触发时，从这里输出执行。

| **节点名称**      | **核心用途** | **一句话记特点**          |
|:-------------:|:--------:|:-------------------:|
| **Branch**    | 条件判断     | 就是代码里的 `if-else`    |
| **Flip Flop** | 交替执行     | 第一次走 A，第二次走 B       |
| **Delay**     | 延时等待     | 暂停指定秒数后再继续          |
| **Do Once**   | 限制次数     | 默认只让过一次，除非被 Reset   |
| **Sequence**  | 理顺线缆     | 0, 1, 2 顺序执行，拒绝面条代码 |
| **For Loop**  | 重复劳动     | 指定次数的循环，自带计数器 Index |
| **Gate**      | 权限控制     | 开门放行，关门拦截           |

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-16-15-36-32-image.png" title="" alt="" data-align="center">

## 6.创建实际人物与移动动画

在内容文件夹中找到我们创建的Code文件夹，打开其中的Character文件夹，找到我们新建的BP_character蓝图类。在视口窗口中打开，正常来说里面只有一个空的胶囊体(如果你选择的是第一人称游戏的话)，如果我们想修改为第三人称游戏，可以在左侧组件中找到网格体来添加人物建模，找到网格体(骨骼网格体组件,代表人物是带动画的,而静态网格体表明建模是静态的,不带动画)，胶囊体代表人物的碰撞范围，在右侧的网格体——资产中找到一个人物模型，这里随便找一个就好。

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-17-15-26-26-image.png" title="" alt="" data-align="center">

如果在这一步中没有找到人物模型也是很正常的，因为我们选择的是第一人称视角游戏，自带的模板中也并没有人物模型模板，顶多只会有一双手，可以通过下面的方式来添加人物模型:

* 点击内容侧滑菜单，点击左上角的添加，选择添加功能或者内容包

    <img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-17-15-30-35-image.png" title="" alt="" data-align="center">

* 在弹出的提示框中选择第三人称游戏，等待添加完成

* 添加完成后会在我们的内容文件夹下新增一个character文件夹，注意这个和我们自己在Code下建立的character文件夹是不一样的，其中包含了我们刚才新建的第三人称游戏的人物动画。

如果添加完第三人称模式后进入到了第三人称关卡，可以搜索FirstPersonMap打开这个第一人称关卡就行。

此时再去添加人物模型就会发现有可以选择的模型了。

我们把模型放到胶囊体中，并且把模型的方向调整为和蓝色箭头一样的方向

1. 此时我们需要给人物添加一个摄像机，用来代表我们的视角，在组件中选择添加，添加弹簧臂组件和摄像机组件，并且把摄像机组件设定为弹簧臂组件的子类，然后在视口中调整弹簧臂和摄像机的相对位置，让他们在一条线上，再调整弹簧臂让摄像机在人物的头部后面。弹簧臂的作用是为了防止摄像机穿进墙体中，弹簧臂起一个伸缩功能。

2. 添加完摄像机后我们要新增一个控制视角的功能，在弹簧臂中打开细节，找到摄像机设置，勾选上使用pawn控制旋转，表明我们可以在游戏中用鼠标来控制旋转。

3. 在上侧的类默认值中搜索使用控制器旋转yaw并取消勾选

现在应该可以使用鼠标控制视野，但是做不到视野朝向哪里就往哪里运动。接下来做这一步

1. 在BP_character中打开事件图标，找到我们之前做过的WASD控制角色移动，当初用来控制移动方向的是获取Actor向前向量，也就是我们上面说的蓝色箭头方向，因此运动方向不会改变。我们需要设置的移动方向是摄像机的朝向。

2. 在组件中拖出Camera拉出搜索——获取世界旋转如果找不到就需要把搜索时候右上方的情景关联给取消掉，再添加getforward——获取向前向量节点，选择分割结构体引脚，把这两者的Z轴连起来

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-17-15-52-15-image.png" title="" alt="" data-align="center">

3. 为什么只连Z轴，是为了控制摄像机在Z轴有移动时，例如俯视或者仰视时Z轴有位移，但是实际移动是只在XY平面上移动的，因此连接Z轴相当于是把Z轴固定了，不考虑Z轴

4. 把输出的向量连到添加移动输入的World Direction中

5. 同样的操作我们添加左右移动，只需要改变getforward为getright——搜索获取向右向量注意获取的是数学下的向量下的获取向右向量，拆分结构体引脚并且连接Z轴，到后面一个的添加移动输入。总体如下：

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-17-15-58-20-image.png" title="" alt="" data-align="center">

6. 现在我们预览后发现，人物不会改变方向后转动，我们在类默认值中搜索将旋转朝向运动后勾选上即可。

现在是制作动画的过程：

在Code文件夹下新建一个Animation文件夹，新建动画——动画蓝图，选择我们刚才人物模型的骨骼，命名为ABP_Man，再新建动画——混合空间，选择同一套骨骼命名为BS_walk，双击进入混合空间。

1. 展开左侧的水平坐标和垂直坐标，水平坐标命名为Speed，最大值设置为600，表示最大速度，网格划分为2，勾选与网格对齐，垂直坐标也选择与网格对齐。

2. 添加动画，在内容中的characters文件夹中找到Mannequins——Animations打开manny其中包含了我们需要的动画有站立，跑动，跳跃等。

3. 回到我们自己创建的混合空间在资产浏览器中搜索我们刚才找到的文件，选择一个跑步的动作拖动到时间轴的最右侧，选择一个走路的姿势拖动到最左侧。按住ctrl可以预览整个的动画。

4. 打开动画蓝图，把刚才创建的混合空间拖到其中，需要输入水平坐标Speed和垂直坐标，垂直坐标没用到，不用输入。

5. 在事件图标中拖出尝试获取Pawn拥有者，搜索getmovement获取移动组件，再搜索get velo获取velocity搜索不到可以关闭上下文关联。

6. 再搜索len选择获取向量长度XY，只考虑速度的大小，把输出的参数保存为speed类型为浮点，接线如下

<img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-17-16-16-20-image.png" title="" alt="" data-align="center">

7. 回到动画图标把speed拖出连到水平坐标中

现在我们创建完动画了，需要加到角色身上，打开BP_character找到类默认值——动画——选择使用动画蓝图，蓝图选择我们刚才创建的ABP_Man。到此就完成移动与站立的动画了，但是动画的衔接不够平滑，我们在混合空间中的水平坐标中更改平滑时间为0.2s，类型为立方即可。



### 7.利用mixamo来获取人物模型和动画

有时候直接从Fab商城获取的人物模型和动画并不是我们想要的，例如你要做跳跃动画或者死亡动画，但是Fab商城中导入的人物模型中是不带这些动画的，因此我们可以选择到https://www.mixamo.com/这个网站中获取免费的人物模型和动画。

1. 打开网址：简单注册Adobe账号后，先在左上角的character处寻找自己需要的人物模型，因为动画是需要绑定人物骨骼的，因此需要选一个人物模型来锁定住动画，选择好自己喜欢的人物模型后下载，选择T型姿势下载即可

        <img src="file:///C:/Users/chenlun/AppData/Roaming/marktext/images/2026-05-23-19-36-26-image.png" title="" alt="" data-align="center">

2. 选择好人物后，后续的动画选择就都会以选择的人物的骨骼为框架，不需要做其他的改动。



### 8. 人物的死亡逻辑

一、

    添加自定义事件：，在事件图标中右键搜索 ..可以看见下图，选择add custom event表示添加自定义事件，自定义事件就相当于是一个子程序，可以在需要触发时进行调用，来实现随时随地调用。

    ![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-23-19-43-52-image.png)

给自定义事件命好名后，开始书写其逻辑，拖出此处的网格体，

![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-23-19-47-13-image.png)

然后右键搜索set simulate  找到设置模拟物理，这一步是模拟人物死亡时的布娃娃效果。

![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-23-19-48-58-image.png)

然后人物在死亡后应该不能有运动模式，此时我们搜索get character movement,和set movement mode，把get movement输入到target中，新的移动方式则选择无来达到我们的效果，左侧的spring arm弹簧臂是防止弹簧臂在死亡时卡在人物与地形之间，set函数名为set do collision test。

![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-23-19-52-39-image.png)

![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-23-19-54-19-image.png)

二、

人物死亡后我们需要在重生点复活，所以还需要写复活时的逻辑

![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-23-19-55-42-image.png)

先新建一个delay延迟2s来进行复活，销毁角色实例，然后重新设置游戏模式get game mode，然后把这个游戏模式通过cast to “自己的游戏模式名”，来设置为当前游戏模式，最后连入一个自定义事件"复活"。这个复活事件是要写在自己的游戏模式当中的，就是一开始新建的用来代替默认的游戏模式，在该模式的事件图标下，我们新建一个变化类型的变量rebirth_location并分隔结构体引脚如下：第一个为位置，第二个为旋转，第三个为缩放

![](C:\Users\chenlun\AppData\Roaming\marktext\images\2026-05-23-20-00-09-image.png)

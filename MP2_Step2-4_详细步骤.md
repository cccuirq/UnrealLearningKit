# MP2 Part 1 —— Health System / Collectibles / Pursuer 超详细步骤（UE 5.4 蓝图）

> 本文档按官方 spec 的 Step 2、3、4 展开。每个"新建蓝图 / 加节点 / 连线"都写到具体引脚级别。
> 建议你从上到下顺序做，**每一步做完先保存（Ctrl+S）再继续**。

---

## 〇、开始前的三条铁律（先记住，能少返工）

1. **跟血量、分数有关的变量和事件，全部放在「玩家蓝图」里**（`BP_LearningKit_PlayerCharacter`）。其它蓝图（血条、血包、收集物、追击者）都只是"引用"玩家，不自己存一份血量。
2. **UI 显示（血条、分数、Game Over 文字）全部放在 Widget 蓝图里**，玩家蓝图只负责"告诉 Widget 更新"。
3. **引用玩家的正确姿势**：其它蓝图里用 `Cast to BP_LearningKit_PlayerCharacter`，Cast 成功后才能调玩家身上的事件。

---

## 一、关键蓝图清单

| 蓝图 / 资源 | 作用 | 状态 |
|---|---|---|
| `BP_LearningKit_PlayerCharacter` | 玩家（在这里加 Health / MaxHealth / Score） | 已存在，直接改 |
| `W_HUD` | 屏幕上的血条 + 分数 | 新建 |
| `W_GameOver` | 游戏结束界面 | 新建 |
| `BP_HealthPack` | 血包（吃一口回血） | 新建 |
| `BP_Collectible` | 收集物（吃掉加分） | 新建 |
| `Pursuer_AIController` | 追击者的 AI 控制器 | 新建 |
| `BP_Pursuer` | 追击者角色（AI Character） | 新建 |

> 建这些资源的位置（文件夹）不影响功能，但建议统一放一个你自己新建的文件夹里，比如 `Content/MP2/`，方便找。

---

## 二、Step 2：健康系统（Health System）

### 2.1 玩家蓝图：血量变量 + 受伤 / 回血事件

打开 `BP_LearningKit_PlayerCharacter`（在 `Content/LearningKit_Games/Blueprints/PlayerCharacter/`）。

**2.1.1 加两个变量**

1. 左侧 **My Blueprint** 面板 → **Variables** 旁边点 **+**。
2. 新建变量，命名 `Health`，类型选 **Float**（默认会显示成 Boolean，点类型下拉改成 Float）。
3. 选中 `Health`，在右侧 Details 里把 **Default Value** 改成 **100.0**。
4. 再点 **+** 新建 `MaxHealth`，类型 **Float**，默认值 **100.0**。
5. 再新建 `IsDead`，类型 **Boolean**，默认值 **false**。它用来防止死亡后重复创建 Game Over 界面或继续处理伤害。

**2.1.2 创建「受伤」自定义事件 LoseHealth**

1. 在 Event Graph 空白处右键 → 搜 **Add Custom Event** → 命名 `LoseHealth`。
2. **选中这个事件节点**，看右侧 **Details 面板**，找到 **Inputs** 栏，点它右边的 **+**。
3. 多出来的一行：把名字改成 `Damage`，类型下拉改成 **Float**。事件节点下方会多出一个 `Damage` 输入引脚。
4. 连线（先看整体文字图，再照着连）：

```
Event LoseHealth (Damage: float)
   │
   ▼
[Health - Damage]  (浮点减法：Health 减 Damage)
   │
   ▼
[Clamp (float)]  Min = 0.0, Max = MaxHealth
   │
   ▼
[Set Health]       ← 把 Clamp 的结果写回 Health
   │
   ▼
[Health <= 0 ?]  (比较：Health <= 0)
   ├─ True  → 接「触发 Game Over」（2.3 再做，先留空）
   └─ False → 接「HUDWidget → UpdateHealth」(2.2 做)
```

具体连法：
- 拖出 `Health` 变量（Get）和 `Damage` 引脚，右键搜 **float - float**（或搜 **Subtract**，选浮点减法）做 `Health - Damage`。
- 减法结果接 **Clamp (float)**（搜 `Clamp (float)`），节点上的 **Min** 填 `0.0`，**Max** 接 `MaxHealth` 变量。
- Clamp 结果接 **Set Health**（搜 `Set Health`）。
- 从 `Set Health` 的 exec 输出再拖出 `Health`（Get），搜 **<=**（LessEqual），另一端接 `0`。
- 这个 `<=` 的结果接 **Branch**（搜 `Branch`），`True` 和 `False` 先各留一条，后面再接。

**2.1.3 创建「回血」自定义事件 GainHealth**

1. 右键 → **Add Custom Event** → 命名 `GainHealth`。
2. 选中节点 → Details 面板 **Inputs** → **+** → 命名 `Amount`，类型 **Float**。
3. 连线：

```
Event GainHealth (Amount: float)
   │
   ▼
[Health + Amount]  (加法)
   │
   ▼
[Clamp (float)]  Min = 0.0, Max = MaxHealth
   │
   ▼
[Set Health]
   │
   ▼
[HUDWidget → UpdateHealth(Health, MaxHealth)]   ← 2.2 做
```

> 这一步先做到「Set Health」即可，最后的 `UpdateHealth` 等 2.2 建好 Widget 后再补。**先把 2.1 全部保存。**

---

### 2.2 血条 UI（W_HUD）

**2.2.1 创建 Widget 蓝图**

1. Content Browser 右键 → **User Interface → Widget Blueprint**，命名 `W_HUD`，双击打开。
2. 左侧 **Palette** 面板里，把 **Progress Bar** 拖到中间画布上，命名 `HealthBar`。
3. （可选，想显示数字再加）拖一个 **Text**，命名 `HealthText`。
4. （Step 3 要用）再拖一个 **Text**，命名 `ScoreText`，先放在右上角。
5. 分别选中 `HealthBar`、`HealthText`、`ScoreText`，确认 Details 顶部的 **Is Variable** 已勾选；否则 Graph 中不能取得它们来调用 `Set Percent` / `Set Text`。

**2.2.2 设置血条位置（锚点）**

1. 选中 `HealthBar`，在右上角 **Details** 面板顶部找到 **Anchors**（锚点格子），点开选**左上角**那一格。
2. 在 Details 里设置：
   - **Position X = 50，Position Y = 50**（离屏幕左上角留点边距）
   - **Size X = 300，Size Y = 30**

**2.2.3 加「更新血条」事件 UpdateHealth**

1. 在这个 Widget 的 **Graph**（W_HUD 自己的图表）里，右键 → **Add Custom Event** → 命名 `UpdateHealth`。
2. 选中节点 → Details **Inputs** → 加两个输入：
   - `CurrentHealth`（Float）
   - `MaxHealth`（Float）
3. 连线：
   - `CurrentHealth` 和 `MaxHealth` 做**除法**（搜 `Divide`，`CurrentHealth ÷ MaxHealth`）。
   - 除法结果 → 从 `HealthBar` 拖出 → 搜 **Set Percent**（把结果接到 Set Percent 的 `In Percent` 引脚）。
   - （可选）再做数字文字：搜 **Format Text**，`Format` 填 `Health: {0} / {1}`，把 `CurrentHealth`、`MaxHealth` 接进去，结果接 `HealthText` 的 **Set Text**。
4. 保存。

**2.2.4 玩家蓝图里创建并显示 HUD**

回到 `BP_LearningKit_PlayerCharacter` 的 Event Graph：

1. 右键搜 **Event BeginPlay**（搜出来的可能是 `Event BlueprintBeginPlay` / `Event ActorBeginPlay`，都一样，点它）。
2. 从 BeginPlay 的 exec 输出拖出 → 搜 **Create Widget**：
   - 节点上的 **Class** 下拉选 **W_HUD**。
3. 从 `Create Widget` 的 **Return Value** 引脚右键 → **Promote to Variable** → 命名 `HUDWidget`（会自动插一个 `Set HUDWidget` 节点）。
4. 从 `Set HUDWidget` 的 exec 输出 → 搜 **Add to Viewport**（Target 接 `Set HUDWidget` 的输出值引脚，也就是那个 Widget 实例）。
5. 从 `Add to Viewport` 的 exec 输出 → 搜 `UpdateHealth`（从 `HUDWidget` 变量拖出调用）：
   - `CurrentHealth` 接 `Health` 变量
   - `MaxHealth` 接 `MaxHealth` 变量

整体文字图：

```
Event BeginPlay
   │
   ▼
[Create Widget]  Class = W_HUD
   │  Return Value ──► Set HUDWidget
   ▼
[Add to Viewport]  Target = HUDWidget
   │
   ▼
[UpdateHealth]  Target = HUDWidget, CurrentHealth = Health, MaxHealth = MaxHealth
```

6. **回填 2.1 的两个事件**：在 `LoseHealth` 和 `GainHealth` 里，`Set Health` 之后，都调用一次 `HUDWidget → UpdateHealth(Health, MaxHealth)`（从 `HUDWidget` 变量拖出搜 `UpdateHealth`，参数同上）。这样每次血变，血条就实时刷新。

---

### 2.3 Game Over 界面（W_GameOver）

**2.3.1 创建 Widget**

1. Content Browser 右键 → **User Interface → Widget Blueprint**，命名 `W_GameOver`，打开。
2. 拖一个 **Text**，内容改成 **Game Over**，字号调大（比如 72），用锚点放中间。
3. 拖一个 **Button**，把 Button 里的文字（Text）改成 **Restart**，放在 Game Over 文字下方。

**2.3.2 Restart 按钮点击重启**

1. 选中 Button，在 Details 面板最下面找到 **Events** → 点 **OnClicked** 右边的 **+**（进入它的点击事件）。
2. 连线：先搜 **Set Game Paused**，Paused 设为 **false**；再接 **Get Current Level Name** → 结果接 **Open Level (by Name)** 的 `Level Name` 输入。
   - （更稳的做法：直接 `Open Level (by Name)`，`Level Name` 里手填你地图的名字，比如 `MyLevel`，不带路径。）

**2.3.3 玩家血量归零时弹 Game Over**

回到玩家蓝图 `LoseHealth` 事件里，`Branch` 的 **True** 分支接：

```
True
 │
 ▼
[Set IsDead] = true
 │
 ▼
[Create Widget]  Class = W_GameOver
 │
 ▼
[Add to Viewport]
 │
 ▼
[Get Player Controller] → [Set Input Mode UI Only]
 │
 ▼
[Get Player Controller] → [Set Show Mouse Cursor]  bShowMouseCursor = true
```

- `Set Input Mode UI Only`：Target 接 `Get Player Controller`（这样鼠标才能点到 Restart 按钮）。
- `Set Input Mode UI Only` 的 **In Widget to Focus** 接 `Create Widget` 的 Return Value（即刚创建的 `W_GameOver`）。
- 在 `Set Show Mouse Cursor` 后加 `Set Game Paused(true)`，让角色和敌人停止活动。
- 在 `LoseHealth` 的最开始加一个 `Branch`：Condition 接 `IsDead`；为 **True** 时直接结束，为 **False** 时才继续扣血。这样不会重复创建 Game Over 界面。

---

### 2.4 血包（BP_HealthPack）

1. Content Browser 右键 → **Blueprint Class → Actor**，命名 `BP_HealthPack`，打开。
2. 组件面板（左上）点 **Add Component**，加：
   - **Static Mesh**：随便选个能当"药包"的 mesh（找不到就用 Sphere 缩一缩，之后能换好看的）。
   - **Sphere Collision**：选中它，在 Details 里：
     - **Shape → Sphere Radius** 调大一点（比 mesh 大一圈，比如 100）。
     - **Collision → Collision Presets** 改成 **OverlapAllDynamic**。
     - 确认 **Generate Overlap Events** 已勾选（选 Overlap 预设一般会自动勾上）。
   - 选中 **Static Mesh**，将 **Collision Presets** 设为 **NoCollision**；由 Sphere Collision 单独负责检测拾取，避免 mesh 自己挡住玩家。
3. 加一个变量 `HealAmount`，类型 **Float**，默认 **25.0**。
4. 在 Event Graph 里，选中 **Sphere Collision** 组件 → 右键 → **Add Event → Add OnComponentBeginOverlap**（生成 `On Component Begin Overlap` 事件）。
5. 连线：

```
On Component Begin Overlap (Other Actor)
   │
   ▼
[Cast to BP_LearningKit_PlayerCharacter]  ← Other Actor 接 Cast 的 Object 输入
   ├─ As BP LearningKit Player Character → [GainHealth]  Amount = HealAmount
   └─ (Cast 的 exec 输出) → 继续往下
   ▼
[Destroy Actor]  (销毁自己)
```

具体：
- `Other Actor` 引脚 → 拖出搜 **Cast to BP_LearningKit_PlayerCharacter**（Class 选你的玩家蓝图类）。
- 从 Cast 的 **As BP LearningKit Player Character** 输出引脚 → 搜 `GainHealth`，`Amount` 接 `HealAmount` 变量。
- Cast 节点的 exec 输出（白色执行线）→ 搜 **Destroy Actor**。

6. 保存，然后把 `BP_HealthPack` 从 Content Browser 拖几个到关卡里。

---

### 2.5 测试健康系统（重要！）

因为追击者要到 Step 4 才有、而且 Part 1 追击者**还不扣血**，你现在没有"自然掉血"的来源。临时做法：

1. 在玩家蓝图里，给 **Input** 加一个测试按键（Project Settings → Input 里加一个 Action，比如叫 `TestDamage`，绑个键 K；或者直接右键搜 **Keyboard Events → K**）。
2. 这个按键事件 → 调 `LoseHealth`，`Damage` 填 **10**。
3. 运行游戏，按 K 看血条下降；吃血包看回血；血扣到 0 看 Game Over 界面弹出、Restart 能不能重开。
4. 验证完把测试按键删掉即可（Part 2 追击者碰撞扣血做好后，就有正式掉血来源了）。

---

## 三、Step 3：收集物 + 分数（Collectibles & Score）

### 3.1 玩家蓝图：加分数变量和加分事件

回到 `BP_LearningKit_PlayerCharacter`：

1. 加变量 `Score`，类型 **Integer**，默认 **0**。
2. 右键 → **Add Custom Event** → 命名 `AddScore`，加输入参数 `Amount`（类型 **Integer**）。
3. 连线：

```
Event AddScore (Amount: int)
   │
   ▼
[Score + Amount]  (整数加法)
   │
   ▼
[Set Score]
   │
   ▼
[HUDWidget → UpdateScore(Score)]   ← 3.2 做
```

### 3.2 W_HUD：加分数显示事件

打开 `W_HUD`：

1. 右键 → **Add Custom Event** → 命名 `UpdateScore`，加输入 `NewScore`（**Integer**）。
2. 连线：`NewScore` → 搜 **Format Text**，`Format` 填 `Score: {0}`，`{0}` 接 `NewScore` → 结果接 `ScoreText` 的 **Set Text**。
   - （不想用 Format Text 的话，最简单：`NewScore` → **To Text (int)** → `ScoreText` 的 **Set Text**。）
3. 保存。回到玩家蓝图 BeginPlay，在 `UpdateHealth` 之后再补一个 `HUDWidget → UpdateScore(Score)`（初始化分数显示为 0）。

### 3.3 收集物蓝图（BP_Collectible）

1. Content Browser 右键 → **Blueprint Class → Actor**，命名 `BP_Collectible`，打开。
2. 加组件：
   - **Static Mesh**：用学习包里一个好看的收集物 mesh（搜索 `coin` / `gem` / 或直接用 Sphere 缩小）。
   - **Sphere Collision**：同血包，**Collision Presets = OverlapAllDynamic**，**Generate Overlap Events** 勾上，半径比 mesh 大一点。
   - 选中 **Static Mesh**，将 **Collision Presets** 设为 **NoCollision**；由 Sphere Collision 单独负责检测拾取。
3. 加变量 `ScoreValue`，类型 **Integer**，默认 **1**。

**3.3.1 让它"浮空 + 旋转"（好看，可选但建议做）**

1. 右键 Event Graph 搜 **Event Tick**。
2. 旋转：从 Tick 拖出 → 搜 **Add Actor Local Rotation**，`Delta Rotation` 的 **Z（Yaw）** 填 `2.0`（或 `90 * Delta Seconds`，让它匀速转）。
3. 上下浮动（可选）：从 Tick → **Get Game Time in Seconds** → **Sin** → 乘一个幅度（比如 20）→ **Set Actor Relative Location** 的 Z。想省事的话，只做旋转也行，浮动不强制。

**3.3.2 碰撞 → 加分 → 销毁**

1. 选中 **Sphere Collision** → 右键 → **Add Event → Add OnComponentBeginOverlap**。
2. 连线：

```
On Component Begin Overlap (Other Actor)
   │
   ▼
[Cast to BP_LearningKit_PlayerCharacter]
   ├─ As ... → [AddScore]  Amount = ScoreValue
   ▼
[Destroy Actor]
```

> 注意：这里 Cast 的目标要和玩家蓝图一致；`AddScore` 是玩家蓝图上的事件。

3. 保存，把 `BP_Collectible` 拖一些到关卡各处（浮空摆放，引导玩家探索）。

---

## 四、Step 4：追击者（Pursuer enemy）

### 4.1 创建 AI Controller

1. Content Browser → 右键 → **Blueprint Class** → 在弹出窗顶部 Class 下拉里展开 **All Classes** → 搜索并选 **AIController** → 命名 `Pursuer_AIController`。
2. 保存。（这个蓝图里不需要写逻辑，就是给追击者"大脑"用的。）

### 4.2 创建 AI Character（BP_Pursuer）

1. 右键 → **Blueprint Class** → **Character** → 命名 `BP_Pursuer`，打开。
2. 组件面板选中 **Mesh**（有的模板叫 Mesh 或 SkeletalMesh），在 Details 里：
   - **Skeletal Mesh** → `SK_EpicCharacter`
   - **Anim Class** → `EpicCharacter_AnimBP`
   （找不到这两个就先用默认，之后能换。）
3. 如果组件里没有 **CapsuleComponent**，就 **Add Component** 加一个 **Capsule Collision**。
4. 让 mesh 套进胶囊里：选中 Mesh，在 Details 里把 **Location** 的 **Z** 改成 **-80**，**Scale** 改成 **0.85**（三个轴都 0.85）。
5. 选中组件面板最顶上的 **BP_Pursuer (self)**（根节点），在 Details 里找到 **AI Controller Class** → 设成 `Pursuer_AIController`。
   - 顺手把 **Auto Possess AI** 设成 **Placed in World or Spawned**（保险起见，确保放进关卡后自动被 AI 控制）。

### 4.3 加 Pawn Sensing（感知玩家）

1. 组件面板 → **Add Component** → 搜 **Pawn Sensing**（PawnSensing），加上。
2. 选中 PawnSensing，在 Details 里设：
   - **Sight Radius** = 1800（中等距离，看到玩家就追）
   - **Peripheral Vision Angle** = 90（正前方 90° 视野，相当于"视线"）
   - **Sensing Interval** = 0.2（多久检测一次，小一点更灵敏）

### 4.4 Roam 随机巡逻逻辑

**先建这些变量：**
- `bIsChasing`（Boolean，默认 false）
- `HomeLocation`（Vector）—— 记住出生点，巡逻围绕它
- `PatrolSpeed`（Float，默认 300）
- `ChaseSpeed`（Float，默认 600）
- `AcquireRange`（Float，默认 1500）—— 在视野内且不超过此距离时开始追击
- `ChaseRange`（Float，默认 2200）—— 超过这个距离就放弃追击并返航；应大于 AcquireRange，避免刚开始追就放弃
- `PlayerRef`（类型选 **Character** 或 **Actor**）

**4.4.1 BeginPlay**

```
Event BeginPlay
   │
   ▼
[Set HomeLocation] = Get Actor Location
   │
   ▼
[Get Character Movement] → [Set Max Walk Speed] = PatrolSpeed
   │
   ▼
[Get Player Character] → [Set PlayerRef]
   │
   ▼
[Roam]  (调用自定义事件)
```

- `Get Actor Location`：搜 "Get Actor Location"。
- `Get Character Movement`：搜它，返回 CharacterMovementComponent → 搜 "Set Max Walk Speed"。
- `Get Player Character`：搜它（返回玩家角色）→ Set PlayerRef。
- 最后调用 `Roam`。

**4.4.2 自定义事件 Roam**

```
Event Roam
   │
   ▼
[Branch]  Condition = bIsChasing ?
   ├─ True → 什么都不做（直接结束，别和追击打架）
   └─ False → 继续往下
         │
         ▼
   [AI MoveTo]  Pawn = Self, Destination = GetRandomReachablePointInRadius(Origin=HomeLocation, Radius=1000)
         │
         ▼ (On Success 和 On Fail 都连到这里)
   [Delay]  Duration = 0.5
         │
         ▼
   [Roam]  (再调自己，形成循环)
```

具体：
- **Branch**：Condition 接 `bIsChasing` 变量。
- **GetRandomReachablePointInRadius**：搜这个节点，`Origin` 接 `HomeLocation`，`Radius` 填 `1000`，它的 **RandomLocation** 输出接 AI MoveTo 的 Destination。
- **AI MoveTo**：搜 "AI MoveTo"；它的 **Pawn** 输入接 `Self`（不要接 `Get Controller`）；`Destination` 接上面那个随机点。
- AI MoveTo 的 **On Success** 和 **On Fail** 两个输出都接同一个 **Delay**（Duration 0.5），Delay 再接回 `Roam`。

> ⚠️ **必须先放 NavMesh**：`GetRandomReachablePointInRadius` 和 `AI MoveTo` 都依赖导航网格。在关卡里放一个 **Nav Mesh Bounds Volume**（Place Actors 搜索 NavMesh 拖进关卡，放大到覆盖整张地图），然后 **Build → Build Paths**（或按 P 看绿色导航区域）。没有 NavMesh，AI 不会动。

### 4.5 追击逻辑（看到玩家 → 追；太远 → 走回出生巡逻区）

**4.5.1 看到玩家时开追**

选中 **PawnSensing** 组件 → 右键 → **Add Event → On See Pawn**（生成 `On See Pawn` 事件）：

```
Event On See Pawn (Pawn)
   │
   ▼
[Branch]  Pawn == PlayerRef AND Get Distance To(PlayerRef) <= AcquireRange AND bIsChasing == false
   ├─ False → 结束
   └─ True → 继续
         │
         ▼
[Set bIsChasing] = true
   │
   ▼
[Get Character Movement] → [Set Max Walk Speed] = ChaseSpeed
   │
   ▼
[Chase]  (调用自定义事件)
```

> `On See Pawn` 已由 PawnSensing 的 Sight Radius 和视野角度保证“在视线中”。上面的 Branch 已额外检查 `AcquireRange`，因此只有“视线内 + 中等距离”的玩家会触发追击；`ChaseRange` 必须大于 `AcquireRange`，才不会刚开始追就放弃。

**4.5.2 自定义事件 Chase（追踪玩家）**

```
Event Chase
   │
   ▼
[AI MoveTo]  Pawn = Self, Target Actor = PlayerRef
```

具体：
- `AI MoveTo` 的 **Pawn** 接 `Self`；**Target Actor** 接 `PlayerRef`。不要把玩家的单次 `Get Actor Location` 接到 Destination；使用 Target Actor 才能持续跟随移动中的玩家。
- `Chase` 只负责发出“追随玩家”的移动请求。距离检查放在下面的定时检查中，避免只有抵达玩家旧位置后才检查距离。

**4.5.3 追击距离定时检查（必须加）**

1. 在 `BP_Pursuer` 新建自定义事件 `CheckChaseDistance`。
2. BeginPlay 中，在调用 `Roam` 后再调用一次 `CheckChaseDistance`，启动下面这个每 0.2 秒自我调用的检查循环。
3. `CheckChaseDistance` 的逻辑：

```
Event CheckChaseDistance
   │
   ▼
[Branch]  bIsChasing
   ├─ False → [Delay 0.2] → [CheckChaseDistance]
   └─ True
       │
       ▼
  [Get Distance To]  Other Actor = PlayerRef
       │
       ▼
  [Distance > ChaseRange] → [Branch]
       ├─ False → [Delay 0.2] → [CheckChaseDistance]
       └─ True  → [Set bIsChasing] = false
                   → [Get Controller] → Stop Movement
                   → [Get Character Movement] → Set Max Walk Speed = PatrolSpeed
                   → [ReturnHome]
                   → [Delay 0.2] → [CheckChaseDistance]
```

> 每一条执行路径最后都回到 `Delay → CheckChaseDistance`。这样不需要处理 Timer 的 Event Delegate 接线，也会稳定地每 0.2 秒检查一次。

### 4.5.4 自定义事件 ReturnHome（先走回出生点，再巡逻）

```
Event ReturnHome
   │
   ▼
[AI MoveTo]  Pawn = Self, Destination = HomeLocation
   │
   ├─ On Success → [Delay 0.5] → [Roam]
   └─ On Fail    → [Delay 0.5] → [Roam]
```

> 这一步不能省略。只把 `bIsChasing` 设回 false 再直接调用 `Roam`，不能保证敌人会先回到出生巡逻区；而 `ReturnHome` 会让它实际走回 `HomeLocation`，符合“enemy moving, not teleportation”的要求。

### 4.6 摆放与测试

1. 把 `BP_Pursuer` 从 Content Browser 拖进关卡（放在玩家出生点附近）。
2. 确认关卡里有 **Nav Mesh Bounds Volume** 且 Build 过路径。
3. 运行游戏：
   - 追击者应该在出生点附近随机走动（Roam）。
   - 你靠近它（进入它正前方 1800 范围内）→ 它加速冲向你。
   - 你跑远（超过 ChaseRange）→ 它减速、实际走回 `HomeLocation`，到达后才恢复随机巡逻。
4. 调参：Sight Radius / ChaseRange / 两个速度按手感调。

> 注：Part 1 追击者**只追不扣血**；追击者碰到玩家扣血（player-enemy collision）是 Part 2 的内容。想提前测试扣血，用 2.5 的测试按键即可。

---

## 五、提交前自查清单

- [ ] 屏幕上有血条，受伤/回血时实时变化
- [ ] 血量归零弹出 Game Over，Restart 能重开
- [ ] 关卡里散落血包，碰到后消失并回血
- [ ] 关卡里散落收集物，碰到后消失并加分，分数显示在 HUD 上
- [ ] 追击者在出生点附近随机巡逻
- [ ] 玩家进入视野（中等距离 + 正前方）追击者加速冲过来
- [ ] 玩家跑远后追击者减速并先走回出生巡逻区（不是瞬移），到达后恢复随机巡逻
- [ ] 关卡里放了 Nav Mesh Bounds Volume 且路径已 Build
- [ ] 录制视频前把所有测试按键、临时节点清理掉

# C4 玩家控制系统 (Player Control System)

> 系统 ID: C4
> 类别：Core
> 优先级：MVP
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**C4 玩家控制系统** 负责玩家角色的移动和位置管理，是玩家与游戏世界交互的基础。

**职责**：
- 玩家移动（360 度自由移动，支持键盘和手柄）
- 移动速度管理（固定速度，可通过进化树升级）
- 场景边界限制（硬边界，玩家不能移出屏幕）
- 玩家 hitbox 位置同步（与碰撞检测系统联动）
- 移动状态输出（供光束系统、收集系统使用）

**设计目标**：
- **响应迅速**：输入到移动的延迟 < 50ms
- **操控精准**：360 度移动，指哪打哪
- **易于上手**：固定速度，无惯性，操作直观
- **东方适配**：小 hitbox 设计，鼓励擦弹操作

---

## 2. 玩家幻想 (Player Fantasy)

玩家控制系统让玩家：
- **灵活穿梭**：在密集弹幕中自由移动，体验"擦弹如舞蹈"的快感
- **精准操控**：指哪打哪，没有延迟和滑动感
- **成长可视**：升级移动速度后明显感觉更灵活
- **安全边界**：屏幕边界清晰可见，不会意外死亡

**东方 Project 适配**：
- 传统东方弹幕游戏使用小 hitbox（角色中心点）
- 本作 hitbox 半径 5px（被弹）/ 12px（擦弹）
- 鼓励玩家在弹幕缝隙中穿梭的高风险高回报玩法

---

## 3. 详细规则 (Detailed Rules)

### 3.1 移动输入处理

```gdscript
# 360 度自由移动
# 键盘：WASD 或方向键，支持斜向（如 W+D = 右上）
# 手柄：左摇杆，360 度模拟输入

# 输入向量合成
move_input = Vector2(
    Input.get_axis("move_left", "move_right"),
    Input.get_axis("move_up", "move_down")
)

# 归一化，防止斜向移动更快
if move_input.length() > 1.0:
    move_input = move_input.normalized()

# 手柄摇杆直接返回归一化向量
```

### 3.2 移动速度配置

```gdscript
# res://assets/data/player/movement_config.tres
class_name PlayerMovementConfig
extends Resource

@export var base_move_speed: float = 200.0  # 像素/秒
@export var max_move_speed: float = 400.0   # 升级后上限
@export var speed_upgrade_steps: Array[float] = [
    220.0,  # 第 1 级升级
    250.0,  # 第 2 级
    280.0,  # 第 3 级
    320.0,  # 第 4 级
    360.0,  # 第 5 级
    400.0,  # 第 6 级（满级）
]

@export_group("边界配置")
@export var play_area_width: float = 480.0   # 游戏区域宽度
@export var play_area_height: float = 320.0  # 游戏区域高度
```

### 3.3 边界限制

```
游戏区域（例如 480x320 像素）
┌─────────────────────────────────┐
│                                 │
│   ┌───────────────────────┐     │
│   │  玩家活动区域          │     │
│   │  (480x320)            │     │
│   │                       │     │
│   │  [玩家] 不能移出此区域 │     │
│   │                       │     │
│   └───────────────────────┘     │
│                                 │
└─────────────────────────────────┘

实现：
player_pos.x = clamp(player_pos.x, boundary_left, boundary_right)
player_pos.y = clamp(player_pos.y, boundary_top, boundary_bottom)
```

### 3.4 玩家状态机

```
┌─────────┐
│  IDLE   │ ← 初始状态，无输入
└────┬────┘
     │
     │ 移动输入
     ↓
┌─────────┐
│ MOVING  │ ← 正在移动
└────┬────┘
     │
     │ 被弹
     ↓
┌─────────┐
│  HIT    │ ← 被弹状态（短暂无敌）
└────┬────┘
     │
     │ 无敌时间结束
     ↓
┌─────────┐
│  IDLE   │ 或继续移动
└─────────┘
```

### 3.5 玩家组件结构

```
Player (CharacterBody2D)
├── Sprite (像素图，雾雨魔理沙)
├── Hitbox (Area2D) - 被弹判定，半径 5px
├── GrazeBox (Area2D) - 擦弹判定，半径 12px
├── CollectRange (Area2D) - 收集范围，半径 50px（可升级）
├── CollisionShape2D (用于物理边界)
└── PlayerController.gd (脚本)
```

### 3.6 移动代码实现

```gdscript
# res://scripts/systems/player_controller.gd
class_name PlayerController
extends CharacterBody2D

signal player_moved(new_position: Vector2)
signal player_hit()
signal speed_upgraded(new_speed: float)

@export var config: PlayerMovementConfig

var _current_speed: float
var _is_invincible: bool = false
var _current_upgrade_level: int = 0

func _ready():
    _current_speed = config.base_move_speed

func _physics_process(delta):
    # 获取输入
    var move_input = _get_move_input()

    # 计算速度
    velocity = move_input * _current_speed

    # 移动并处理边界
    move_and_slide()
    _apply_boundary_limits()

    # 发送位置信号
    emit_signal("player_moved", global_position)

func _get_move_input() -> Vector2:
    var input = Vector2(
        Input.get_axis("move_left", "move_right"),
        Input.get_axis("move_up", "move_down")
    )
    return input.normalized() if input.length() > 1.0 else input

func _apply_boundary_limits():
    global_position.x = clamp(
        global_position.x,
        -config.play_area_width / 2,
        config.play_area_width / 2
    )
    global_position.y = clamp(
        global_position.y,
        -config.play_area_height / 2,
        config.play_area_height / 2
    )

func upgrade_speed():
    if _current_upgrade_level >= config.speed_upgrade_steps.size():
        return false

    _current_speed = config.speed_upgrade_steps[_current_upgrade_level]
    _current_upgrade_level += 1
    emit_signal("speed_upgraded", _current_speed)
    return true

func get_current_speed() -> float:
    return _current_speed

func get_position() -> Vector2:
    return global_position
```

---

## 4. 公式 (Formulas)

### 4.1 移动速度计算

```
# 基础速度
move_speed = base_move_speed  # 200 px/s

# 升级后速度
move_speed = speed_upgrade_steps[upgrade_level]

# 速度增量
speed_delta = speed_upgrade_steps[n] - speed_upgrade_steps[n-1]
# 典型增量：+20 ~ +40 px/s 每级
```

### 4.2 移动位移计算

```
# 每帧位移
delta_position = move_input * move_speed * delta_time

# delta_time = 1/60 ≈ 0.0167s (60 FPS)
# 示例：200 px/s × 0.0167s ≈ 3.34 px/帧
```

### 4.3 边界钳制

```
clamped_x = clamp(x, -play_area_width/2, play_area_width/2)
clamped_y = clamp(y, -play_area_height/2, play_area_height/2)
```

### 4.4 穿越时间计算

```
# 从屏幕一端到另一端的时间
cross_time = play_area_width / move_speed

# 示例：480px / 200px/s = 2.4 秒（基础速度）
# 升级后：480px / 400px/s = 1.2 秒（满级速度）
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **多个方向键同时按下** | 向量合成后归一化，保持速度一致 |
| **输入设备切换** | 键盘→手柄自动切换，无需手动切换 |
| **玩家卡住** | 边界钳制保证不会卡出地图 |
| **被弹时无敌时间** | 移动不受限制，可继续躲避 |
| **手柄摇杆漂移** | 死区 0.15 过滤微小输入 |
| **帧率波动** | 使用 `delta` 时间，保证移动距离与帧率无关 |
| **升级速度后超出边界** | 边界钳制优先，不会移出屏幕 |

---

## 6. 依赖关系 (Dependencies)

<!-- TODO: 依赖的其他系统和被哪些系统依赖 -->

### 6.1 本系统依赖

| 系统 | 依赖原因 |
|------|----------|
| F1 输入系统 | 读取玩家移动输入 |
| C3 碰撞检测系统 | 玩家 hitbox 位置同步 |

### 6.2 依赖本系统

| 系统 | 依赖原因 |
|------|----------|
| C2 时间管理系统 | 玩家存活状态影响计时 |
| G1 魔炮光束系统 | 玩家位置决定光束发射位置 |
| G3 收集系统 | 玩家移动决定收集范围覆盖 |
| P1 UI 系统 | 玩家位置映射到 HUD |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `BASE_MOVE_SPEED` | 200 px/s | 150 - 300 px/s | 基础移动速度 |
| `MAX_MOVE_SPEED` | 400 px/s | 300 - 600 px/s | 升级后最大速度 |
| `PLAY_AREA_WIDTH` | 480 px | 320 - 640 px | 游戏区域宽度 |
| `PLAY_AREA_HEIGHT` | 320 px | 240 - 480 px | 游戏区域高度 |
| `JOY_DEADZONE` | 0.15 | 0.1 - 0.25 | 手柄摇杆死区 |
| `SPEED_UPGRADE_COUNT` | 6 级 | 3 - 10 级 | 速度升级次数 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 360 度自由移动正常工作（键盘 WASD/方向键）
- [ ] 手柄左摇杆移动正常
- [ ] 移动速度为固定值（无惯性）
- [ ] 边界限制生效，玩家不能移出屏幕
- [ ] 移动输入到屏幕响应的延迟 < 50ms
- [ ] 斜向移动速度归一化（不比直线快）
- [ ] 玩家位置信号正确发送（供其他系统使用）
- [ ] 速度升级功能正常（进化树节点）

### 完整版标准

- [ ] 手柄键盘无缝切换
- [ ] 速度升级后移动明显变快
- [ ] 满级速度 (400px/s) 下仍能精确操控
- [ ] 边界有视觉提示（如半透明边框）
- [ ] 被弹后移动不受影响（仅触发惩罚）

---

## 附录 A: 玩家状态机

### 状态转换图

```
┌─────────────────────────────────────────────────────────────┐
│                    Player State Machine                     │
└─────────────────────────────────────────────────────────────┘

     ┌─────────┐
     │  IDLE   │ ← 初始状态，无移动输入
     └────┬────┘
          │
          │ 移动输入 ≠ 0
          ↓
     ┌─────────┐
     │ MOVING  │ ← 正在移动
     └────┬────┘
          │
    ┌─────┴─────┐
    │           │
    │ 输入 = 0   │ 被弹检测
    │           ↓
    │      ┌─────────┐
    │      │  HIT    │ ← 被弹状态（无敌 0.5s）
    │      └────┬────┘
    │           │
    │           │ 无敌时间结束
    └───────────┘
          ↓
     ┌─────────┐
     │  IDLE   │ 或继续移动
     └─────────┘
```

---

## 附录 B: 游戏区域示意

```
┌───────────────────────────────────────────────────┐
│            战斗场景 (例如 640x480)                 │
│                                                   │
│   ┌─────────────────────────────────────────┐     │
│   │                                         │     │
│   │   ┌───────────────────────────────┐     │     │
│   │   │                               │     │     │
│   │   │     玩家活动区域               │     │     │
│   │   │     (480x320 像素)            │     │     │
│   │   │                               │     │     │
│   │   │        [魔理沙]               │     │     │
│   │   │         ● hitbox 5px          │     │     │
│   │   │        ~~~ graze 12px         │     │     │
│   │   │                               │     │     │
│   │   └───────────────────────────────┘     │     │
│   │         ↑                               │     │
│   │         硬边界，玩家不能移出              │     │
│   └─────────────────────────────────────────┘     │
│                                                   │
└───────────────────────────────────────────────────┘
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始版本创建 |

# F1 输入系统 (Input System)

> 系统 ID：F1
> 类别：Foundation
> 优先级：MVP
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**F1 输入系统** 是游戏的基础输入层，负责定义和管理所有玩家输入 action。

**职责**：
- 定义 Godot Input Map 的所有 action 名称和默认绑定
- 封装原始输入信号，提供语义化的输入 API（如 `is_move_left()` 而非 `is_key_pressed()`）
- 支持键盘（WASD/方向键双布局）和手柄（XInput 标准）
- 提供输入上下文切换（战斗/进化树/菜单）

**设计目标**：
- 单一来源：所有 input action 定义集中在 `input_config.gd`
- 易于扩展：新增 action 只需修改配置表
- 支持手柄：Godot 默认支持 XInput，只需配置映射
- 输入缓冲：支持短时输入缓存（用于连击/快速切换）

---

## 2. 玩家幻想 (Player Fantasy)

输入系统是**隐形**的——玩家不应该"感觉到"它的存在，只应该感受到：

- **响应迅速**：按下按键到角色移动/发射的延迟 < 50ms
- **操作准确**：不会有"我明明按了却没反应"的挫败感
- **多设备友好**：键盘玩家和手柄玩家都能舒适游玩
- **无冲突**：同时按多个键（如移动 + 蓄力）不会互相干扰

**东方 Project 适配**：
- 传统东方弹幕游戏使用方向键 + Z/X 布局，本作支持 WASD/方向键双布局
- 蓄力/发射操作应当符合弹幕游戏玩家的习惯

---

## 3. 详细规则 (Detailed Rules)

### 3.1 Input Action 定义

所有 Input Action 定义在 `input_config.gd` 中：

| Action 名称 | 描述 | 键盘 (WASD) | 键盘 (方向键) | 手柄 |
|-------------|------|-------------|---------------|------|
| `move_left` | 向左移动 | A | Left Arrow | Left Stick / D-Pad Left |
| `move_right` | 向右移动 | D | Right Arrow | Left Stick / D-Pad Right |
| `move_up` | 向上移动 | W | Up Arrow | Left Stick / D-Pad Up |
| `move_down` | 向下移动 | S | Down Arrow | Left Stick / D-Pad Down |
| `charge` | 蓄力（按住） | Space | Z | A Button (Xbox) / Cross (PS) |
| `fire` | 发射（释放） | Space (release) | Z (release) | A Button (release) |
| `dash` | 冲刺/闪避 | Shift | X | B Button (Xbox) / Circle (PS) |
| `confirm` | 确认/选择 | Enter / Z | Enter / Z | A Button |
| `cancel` | 取消/返回 | X / Esc | X / Esc | B Button |
| `pause` | 暂停菜单 | P / Esc | P / Esc | Start Button |
| `evolution_tree` | 打开进化树 (战斗中) | E | E | Y Button |

### 3.2 输入上下文

输入系统支持**上下文切换**，不同场景启用不同的 input actions：

| 上下文 | 启用的 Actions | 禁用的 Actions |
|--------|----------------|----------------|
| **战斗** | move_*, charge, fire, dash, evolution_tree | confirm, cancel (菜单用) |
| **进化树** | move_*, confirm, cancel | charge, fire, dash |
| **菜单/对话** | move_*, confirm, cancel, pause | move_*(战斗), charge, fire, dash |
| **暂停** | move_*, confirm, cancel, pause | 所有战斗相关 |

### 3.3 输入响应模式

| 模式 | 描述 | 使用场景 |
|------|------|----------|
| **Hold (按住)** | 按住时持续触发，每帧返回 `true` | 移动、蓄力 |
| **Press (按下)** | 按下瞬间触发一次 | 确认、取消、暂停 |
| **Release (释放)** | 释放瞬间触发一次 | 发射（蓄力后） |
| **Axis (轴向)** | 返回 -1.0 到 1.0 的模拟值 | 手柄摇杆移动 |

### 3.4 输入缓冲

为避免玩家感觉"按键无效"，实现短时输入缓冲：

```
输入缓冲窗口 = 0.1 秒 (100ms)
```

- 如果玩家在 `charge` 按下后 100ms 内松开，视为有效 `fire`
- 如果玩家在 `charge` 按下超过 100ms 后松开，视为蓄力中断（被弹情况）

### 3.5 手柄支持

- **自动检测**：Godot 自动检测手柄连接/断开
- **UI 导航**：手柄连接后，UI 自动切换到手柄导航模式
- **震动反馈**：预留手柄震动 API（被弹、升级时触发）

---

## 4. 公式 (Formulas)

### 4.1 移动输入合成

```
# 键盘移动：归一化方向向量
move_direction = Vector2(
    Input.get_axis("move_left", "move_right"),
    Input.get_axis("move_up", "move_down")
).normalized()

# 手柄移动：直接使用摇杆模拟值（已经是归一化的）
move_direction = Vector2(
    Input.get_joy_axis(0, JOY_AXIS_LEFT_X),
    Input.get_joy_axis(0, JOY_AXIS_LEFT_Y)
)
```

### 4.2 输入缓冲窗口

```
# 蓄力时间计算
charge_time = release_time - press_time

# 有效发射判定
if charge_time < CHARGE_MIN_TIME (0.1s):
    # 视为"点按"，发射基础光束
    fire_type = FIRE_TYPE_QUICK
elif charge_time >= CHARGE_MIN_TIME and not interrupted:
    # 视为"蓄力完成"，发射强化光束
    fire_type = FIRE_TYPE_CHARGED
else:
    # 蓄力中断（被弹），发射弱化光束
    fire_type = FIRE_TYPE_INTERRUPTED
```

### 4.3 死区阈值（手柄）

```
# 摇杆死区，防止漂移
JOY_DEADZONE = 0.15

if abs(axis_value) < JOY_DEADZONE:
    axis_value = 0.0
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **多个按键同时按下** | 所有按键都响应，通过 `get_axis()` 合成方向 |
| **手柄中途断开** | 暂停游戏，提示玩家重新连接；如无法恢复，返回主菜单 |
| **手柄中途连接** | 下一帧自动检测，切换到手柄输入模式（无需重启） |
| **按键冲突** | Godot 的 Input 系统自动处理，每个 action 独立判定 |
| **输入设备切换** | 记录最后一次使用的设备类型，UI 高亮提示相应按钮 |
| **Alt-Tab 切出** | Godot `NOTIFICATION_WM_OUT_FOCUS` 时暂停输入响应 |
| **长按卡键** | 操作系统级别的卡键过滤，Godot 自动处理 |

---

## 6. 依赖关系 (Dependencies)

<!-- TODO: 依赖的其他系统和被哪些系统依赖 -->

### 6.1 本系统依赖

| 系统 | 依赖原因 |
|------|----------|
| *无* | Foundation 层，无前置依赖 |

### 6.2 依赖本系统

| 系统 | 依赖原因 |
|------|----------|
| C4 玩家控制系统 | 需要读取玩家移动和射击输入 |
| C1 场景管理系统 | 需要读取返回/确认输入 |
| G4 进化树系统 | 需要读取节点选择输入 |
| P1 UI 系统 | 需要读取菜单导航输入 |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `INPUT_BUFFER_WINDOW` | 0.1s (100ms) | 0.05s - 0.2s | 输入缓冲窗口大小，影响"点按"判定 |
| `CHARGE_MIN_TIME` | 0.1s (100ms) | 0.05s - 0.3s | 最小蓄力时间，小于此值视为点按 |
| `JOY_DEADZONE` | 0.15 | 0.1 - 0.25 | 手柄摇杆死区，防止漂移 |
| `MOVE_REPEAT_DELAY` | 0.05s (50ms) | 0.03s - 0.1s | 键盘按住时的重复触发间隔 |
| `MAX_SIMULTANEOUS_INPUTS` | 4 | 2 - 6 | 同时响应的最大按键数 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 所有 11 个 Input Actions 定义在 `input_config.gd` 中
- [ ] WASD + 空格布局可正常移动和蓄力/发射
- [ ] 方向键 + Z 布局可正常移动和蓄力/发射
- [ ] 手柄（XInput 标准）可正常移动和蓄力/发射
- [ ] 输入响应延迟 < 50ms（从按键到角色移动）
- [ ] 战斗场景和进化树场景的输入上下文切换正确
- [ ] 同时按多个键（移动 + 蓄力）不会冲突
- [ ] 手柄摇杆死区生效，无漂移现象
- [ ] 暂停/确认/取消功能正常

### 完整版标准

- [ ] 手柄震动 API 预留（被弹、升级时触发）
- [ ] 输入缓冲系统正常工作
- [ ] 手柄中途连接/断开处理正确
- [ ] 可重新绑定按键（配置文件中）

---

## 附录 A: Godot 输入映射配置

### project.godot Input 配置

```gdscript
# 自动生成的 Input Map 配置
# 位置：res://input_config.gd

class_name InputConfig

# Action 名称常量
const ACTION_MOVE_LEFT = "move_left"
const ACTION_MOVE_RIGHT = "move_right"
const ACTION_MOVE_UP = "move_up"
const ACTION_MOVE_DOWN = "move_down"
const ACTION_CHARGE = "charge"
const ACTION_FIRE = "fire"
const ACTION_DASH = "dash"
const ACTION_CONFIRM = "confirm"
const ACTION_CANCEL = "cancel"
const ACTION_PAUSE = "pause"
const ACTION_EVOLUTION_TREE = "evolution_tree"

# 输入配置表
const INPUT_CONFIG = {
    ACTION_MOVE_LEFT: {
        "keyboard_wasd": KEY_A,
        "keyboard_arrows": KEY_LEFT,
        "joy_axis": {"axis": JOY_AXIS_LEFT_X, "value": -1},
        "joy_dpad": JOY_DPAD_LEFT,
    },
    ACTION_MOVE_RIGHT: {
        "keyboard_wasd": KEY_D,
        "keyboard_arrows": KEY_RIGHT,
        "joy_axis": {"axis": JOY_AXIS_LEFT_X, "value": 1},
        "joy_dpad": JOY_DPAD_RIGHT,
    },
    # ... 其他 action 配置
}

# 死区设置
const JOY_DEADZONE = 0.15

# 输入缓冲窗口（秒）
const INPUT_BUFFER_WINDOW = 0.1

# 最小蓄力时间（秒）
const CHARGE_MIN_TIME = 0.1
```

### Input Map 初始化代码

```gdscript
# 在游戏启动时自动配置 Input Map
static func setup_input_map():
    for action in INPUT_CONFIG.keys():
        if not InputMap.has_action(action):
            InputMap.add_action(action)

        # 添加键盘绑定（WASD 布局）
        var key_wasd = InputEventKey.new()
        key_wasd.keycode = INPUT_CONFIG[action]["keyboard_wasd"]
        InputMap.action_add_event(action, key_wasd)

        # 添加键盘绑定（方向键布局）
        var key_arrows = InputEventKey.new()
        key_arrows.keycode = INPUT_CONFIG[action]["keyboard_arrows"]
        InputMap.action_add_event(action, key_arrows)

        # 添加手柄绑定
        # ... (axis 和 dpad 配置)
```

---

## 附录 B: 输入 API 封装

```gdscript
# 玩家可使用的输入 API
# 位置：res://scripts/systems/input_handler.gd

class_name InputHandler

# 移动输入（返回归一化向量）
static func get_move_direction() -> Vector2:
    return Vector2(
        Input.get_axis(ACTION_MOVE_LEFT, ACTION_MOVE_RIGHT),
        Input.get_axis(ACTION_MOVE_UP, ACTION_MOVE_DOWN)
    ).normalized()

# 蓄力输入（返回是否按住）
static func is_charging() -> bool:
    return Input.is_action_pressed(ACTION_CHARGE)

# 发射输入（返回释放瞬间）
static func is_fire_released() -> bool:
    return Input.is_action_just_released(ACTION_FIRE)

# 确认输入
static func is_confirm_pressed() -> bool:
    return Input.is_action_just_pressed(ACTION_CONFIRM)

# 取消输入
static func is_cancel_pressed() -> bool:
    return Input.is_action_just_pressed(ACTION_CANCEL)
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始骨架创建 |

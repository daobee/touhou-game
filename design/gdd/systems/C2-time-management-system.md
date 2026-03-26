# C2 时间管理系统 (Time Management System)

> 系统 ID: C2
> 类别：Core
> 优先级：MVP
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**C2 时间管理系统** 负责战斗场景的时间管理，包括计时、时长升级和时间结束处理。

**职责**：
- 战斗计时（初始 10 秒，可升级至最长 30 秒）
- 进度条显示（不显示具体数字，用视觉反馈）
- 时间升级逻辑（进化树节点：前几级 +1 秒，后几级 +2 秒）
- 时间结束提示（玩家可选择返回进化树或继续战斗）
- 暂停/恢复支持（暂停菜单时停止计时）

**设计目标**：
- **清晰反馈**：进度条让玩家直观了解剩余时间
- **灵活控制**：时间结束后玩家自主决定返回或继续
- **可预测升级**：进化树节点明确显示延长多少秒

---

## 2. 玩家幻想 (Player Fantasy)

时间管理系统让玩家：
- **紧张感**：进度条减少带来时间压力，促使玩家高效收集
- **成长感**：每次升级延长战斗时间，能收集更多资源
- **掌控感**：时间结束后可自主选择返回或继续，不被强制打断

**视觉反馈**：
- 进度条颜色变化：绿色（充足）→ 黄色（中等）→ 红色（紧张）
- 最后 3 秒闪烁提示
- 时间结束时弹出选择对话框

---

## 3. 详细规则 (Detailed Rules)

### 3.1 战斗时长配置

```gdscript
# res://assets/data/combat/time_config.tres
class_name CombatTimeConfig
extends Resource

# 基础时长
@export var base_duration: float = 10.0  # 初始 10 秒

# 最大时长
@export var max_duration: float = 30.0  # 升级后最长 30 秒

# 升级增量配置（分段式）
@export var upgrade_durations: Array[float] = [
    1.0,  # 第 1 级 +1 秒
    1.0,  # 第 2 级 +1 秒
    1.0,  # 第 3 级 +1 秒
    2.0,  # 第 4 级 +2 秒
    2.0,  # 第 5 级 +2 秒
    2.0,  # 第 6 级 +2 秒
    2.0,  # 第 7 级 +2 秒
    2.0,  # 第 8 级 +2 秒
]
# 总计：3×1 + 5×2 = 13 秒增量，10 + 13 = 23 秒（不超过 30 秒上限）
```

### 3.2 进度条显示规则

```
进度条填充比例 = 当前时间 / 最大时间

颜色渐变:
- 70%-100%: 绿色 (#4CAF50)
- 30%-70%:  黄色 (#FFC107)
- 0%-30%:   红色 (#F44336)

闪烁阈值：< 3 秒时，进度条边框闪烁（2Hz 频率）
```

### 3.3 时间结束流程

```
1. 计时器归零
2. 暂停战斗（敌人停止攻击，弹幕停止发射）
3. 显示选择对话框：
   ┌────────────────────────────┐
   │  战斗时间已结束！           │
   │                            │
   │  [返回进化树]  [继续战斗]  │
   └────────────────────────────┘

4. 玩家选择：
   - "返回进化树"：触发场景切换
   - "继续战斗"：重置计时器（当前时长），战斗恢复
```

### 3.4 时间管理状态机

```
[Idle] ← 初始状态
   │
   │ 战斗开始
   ↓
[Running] ←──┐
   │         │ 暂停
   │ 计时    │
   ↓         │
[Paused] ────┘ 恢复
   │
   │ 时间归零
   ↓
[Ended]
   │
   │ 玩家选择
   ├─→ [Idle] (返回进化树)
   └─→ [Running] (继续战斗，重置计时)
```

### 3.5 时间升级逻辑

```gdscript
# 计算当前升级可增加的时间
func get_upgrade_duration(upgrade_level: int) -> float:
    if upgrade_level >= upgrade_durations.size():
        return 0.0  # 已满级
    return upgrade_durations[upgrade_level]

# 计算升级后的总时长
func get_duration_after_upgrade(current_duration: float, upgrade_level: int) -> float:
    var added_time = get_upgrade_duration(upgrade_level)
    return min(current_duration + added_time, max_duration)
```

### 3.6 暂停/恢复逻辑

```gdscript
# 暂停时
func pause():
    if state == State.RUNNING:
        state = State.PAUSED
        _pause_timer()  # Godot 计时器暂停

# 恢复时
func resume():
    if state == State.PAUSED:
        state = State.RUNNING
        _resume_timer()
```

---

## 4. 公式 (Formulas)

### 4.1 战斗时长计算

```
当前最大时长 = base_duration + Σ(已升级的增量)
            = 10 + (第 1 级 + 第 2 级 + ... + 第 n 级)

示例:
- 0 级升级：10 秒
- 3 级升级：10 + 1 + 1 + 1 = 13 秒
- 8 级升级：10 + 1 + 1 + 1 + 2 + 2 + 2 + 2 + 2 = 23 秒
```

### 4.2 进度条颜色插值

```
if time_ratio >= 0.7:
    color = GREEN
elif time_ratio >= 0.3:
    color = lerp(YELLOW, GREEN, (time_ratio - 0.3) / 0.4)
else:
    color = lerp(RED, YELLOW, time_ratio / 0.3)
```

### 4.3 继续战斗重置

```
继续战斗后时长 = 当前最大时长（不变）
重置后时间 = 当前最大时长（满状态重新开始）
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **暂停菜单打开** | 计时器暂停，进度条冻结 |
| **游戏切出后台** | 自动暂停，恢复后继续计时 |
| **时间结束瞬间玩家死亡** | 优先触发时间结束流程，忽略死亡 |
| **继续战斗后立刻再次结束** | 允许，但第 2 次结束后自动返回（防无限刷） |
| **升级后时长超过 30 秒** | clamp 到 max_duration (30 秒) |
| **进度条 UI 未加载** | 使用日志记录，不影响计时逻辑 |
| **场景切换中断** | 计时器清理，防止内存泄漏 |

---

## 6. 依赖关系 (Dependencies)

<!-- TODO: 依赖的其他系统和被哪些系统依赖 -->

### 6.1 本系统依赖

| 系统 | 依赖原因 |
|------|----------|
| C1 场景管理系统 | 通知场景切换（时间结束返回进化树） |
| F2 数据配置系统 | 读取时间配置（基础时长、升级增量） |

### 6.2 依赖本系统

| 系统 | 依赖原因 |
|------|----------|
| G5 战斗系统 | 获取战斗剩余时间、判断是否结束 |
| P1 UI 系统 | HUD 显示倒计时/计时器 |
| G8 敌人升级系统 | 根据战斗时长调整敌人强度 |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `BASE_DURATION` | 10.0s | 5.0s - 15.0s | 初始战斗时长 |
| `MAX_DURATION` | 30.0s | 20.0s - 60.0s | 升级后最大时长 |
| `UPGRADE_INCREMENT_EARLY` | 1.0s | 0.5s - 2.0s | 前几级升级增量 |
| `UPGRADE_INCREMENT_LATE` | 2.0s | 1.0s - 3.0s | 后几级升级增量 |
| `COLOR_YELLOW_THRESHOLD` | 0.7 | 0.5 - 0.8 | 进度条变黄阈值 |
| `COLOR_RED_THRESHOLD` | 0.3 | 0.2 - 0.4 | 进度条变红阈值 |
| `BLINK_THRESHOLD` | 3.0s | 2.0s - 5.0s | 闪烁提示阈值 |
| `BLINK_FREQUENCY` | 2.0Hz | 1.0Hz - 4.0Hz | 闪烁频率 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 战斗计时器正确启动和运行
- [ ] 基础时长 10 秒配置正确
- [ ] 进度条 UI 显示当前时间比例
- [ ] 进度条颜色随时间变化（绿→黄→红）
- [ ] 时间结束后显示选择对话框
- [ ] "返回进化树"选项触发场景切换
- [ ] "继续战斗"选项重置计时器并恢复战斗
- [ ] 暂停菜单打开时计时器暂停
- [ ] 进化树升级可增加战斗时长（+1 秒/+2 秒）

### 完整版标准

- [ ] 升级增量配置为数组，支持自定义每级增量
- [ ] 进度条 < 3 秒时闪烁提示
- [ ] 颜色渐变平滑过渡（非突变）
- [ ] 继续战斗后时长重置为当前最大值
- [ ] 最大时长限制 30 秒生效（超出部分截断）
- [ ] 场景切换时计时器正确清理

---

## 附录 A: 状态机图

### 时间管理状态机

```
┌───────────────────────────────────────────────────────────────┐
│                       TimeManager                             │
└───────────────────────────────────────────────────────────────┘

State Transition Diagram:

     ┌─────────┐
     │  IDLE   │ ← 初始状态，计时器未启动
     └────┬────┘
          │
          │ 战斗开始信号
          ↓
     ┌─────────┐
     │ RUNNING │ ← 计时中
     └────┬────┘
          │
    ┌─────┴─────┐
    │           │
    │ 暂停信号   │ 时间归零信号
    ↓           ↓
┌─────────┐  ┌─────────┐
│ PAUSED  │  │ ENDED   │ ← 时间结束，等待玩家选择
└────┬────┘  └────┬────┘
     │           │
     │ 恢复信号  ├──────┬──────────┐
     │           │      │          │
     └───────────┘      ↓          ↓
                   ┌────────┐  ┌──────────┐
                   │  IDLE  │  │ RUNNING  │
                   │ (返回) │  │ (继续)   │
                   └────────┘  └──────────┘
```

---

## 附录 B: TimerManager 实现

```gdscript
# res://scripts/systems/time_manager.gd
class_name TimeManager
extends Node

signal time_started(duration: float)
signal time_updated(elapsed: float, remaining: float)
signal time_ended()
signal time_paused()
signal time_resumed()
signal time_continue_requested()
signal time_return_requested()

enum State { IDLE, RUNNING, PAUSED, ENDED }

@export var config: CombatTimeConfig

var _state: State = State.IDLE
var _current_duration: float = 10.0
var _elapsed_time: float = 0.0
var _upgrade_level: int = 0

var _timer: Timer
var _is_dialog_showing: bool = false

func _ready():
    _timer = Timer.new()
    _timer.timeout.connect(_on_timer_timeout)
    add_child(_timer)

## 开始战斗计时
func start_combat():
    _state = State.RUNNING
    _elapsed_time = 0.0
    _timer.start(0.01)  # 10ms 精度
    emit_signal("time_started", _current_duration)

## 升级战斗时长
func upgrade_duration():
    if _upgrade_level >= config.upgrade_durations.size():
        return false

    var added_time = config.upgrade_durations[_upgrade_level]
    _current_duration = min(_current_duration + added_time, config.max_duration)
    _upgrade_level += 1
    return true

## 暂停计时
func pause():
    if _state == State.RUNNING:
        _state = State.PAUSED
        _timer.stop()
        emit_signal("time_paused")

## 恢复计时
func resume():
    if _state == State.PAUSED:
        _state = State.RUNNING
        _timer.start(0.01)
        emit_signal("time_resumed")

## 玩家选择继续战斗
func continue_combat():
    _state = State.RUNNING
    _elapsed_time = 0.0
    _is_dialog_showing = false
    emit_signal("time_continue_requested")

## 玩家选择返回进化树
func return_to_evolution_tree():
    _state = State.IDLE
    _is_dialog_showing = false
    emit_signal("time_return_requested")

func _on_timer_timeout():
    if _state != State.RUNNING:
        return

    _elapsed_time += 0.01
    emit_signal("time_updated", _elapsed_time, _current_duration - _elapsed_time)

    if _elapsed_time >= _current_duration:
        _on_time_ended()

func _on_time_ended():
    _state = State.ENDED
    _timer.stop()
    _is_dialog_showing = true
    emit_signal("time_ended")

func get_elapsed_time() -> float:
    return _elapsed_time

func get_remaining_time() -> float:
    return max(0.0, _current_duration - _elapsed_time)

func get_time_ratio() -> float:
    return 1.0 - (_elapsed_time / _current_duration)

func get_current_duration() -> float:
    return _current_duration
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始版本创建 |

# C1 场景管理系统 (Scene Management System)

> 系统 ID：C1
> 类别：Core
> 优先级：MVP
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**C1 场景管理系统** 负责游戏所有场景的加载、切换、过渡和状态管理。

**场景类型**：
- **主菜单场景** (MainMenu) — 游戏标题、开始游戏、选项、退出
- **战斗场景** (Combat) — 10-30 秒战斗循环，魔理沙发射魔炮
- **进化树场景** (EvolutionTree) — 网状 UI，消耗资源点亮节点
- **剧情场景** (Story) — 对话展示，剧情事件触发

**职责**：
- 管理场景加载（同步/异步）
- 控制场景切换流程（过渡动画）
- 保存和恢复场景状态
- 提供场景间通信机制（信号/事件总线）

**设计目标**：
- **流畅切换**：过渡动画让场景切换不突兀
- **状态保持**：返回进化树时，已点亮的节点保持状态
- **低耦合**：场景之间通过事件通信，不直接引用
- **易于扩展**：新增场景类型只需注册，不修改核心逻辑

---

## 2. 玩家幻想 (Player Fantasy)

场景管理系统让玩家：
- **沉浸体验**：流畅的过渡动画，不会感到"跳戏"
- **无缝循环**：战斗→进化树→战斗的循环流畅自然
- **进度安心**：返回进化树时，之前的进度完好保存
- **快速加载**：场景切换等待时间短（目标 < 2 秒）

**东方 Project 适配**：
- 剧情场景使用东方风格的对话框和立绘（如有）
- 战斗场景和进化树场景的美术风格统一

---

## 3. 详细规则 (Detailed Rules)

### 3.1 场景架构图

```
游戏根节点 (Window)
└── SceneManager (单例，持久化)
    └── 当前场景容器 (MarginContainer)
        ├── 主菜单场景 (MainMenu.tscn)
        ├── 战斗场景 (CombatScene.tscn)
        ├── 进化树场景 (EvolutionTreeScene.tscn)
        └── 剧情场景 (StoryScene.tscn)

过渡动画层 (ColorRect + AnimationPlayer)
    └── 淡入淡出/滑动效果
```

### 3.2 场景状态机

```
[启动] → 主菜单
   │
   ├─→ [开始游戏] ─→ 战斗场景 ←──┐
   │         │                   │
   │         └─→ 战斗结束        │
   │                  │          │
   │                  ↓          │
   │         进化树场景 ─────────┘
   │
   └─→ [退出] ─→ 退出游戏
```

### 3.3 场景配置

```gdscript
# res://assets/data/scenes/scene_config.tres
class_name SceneConfig
extends Resource

enum SceneType {
    MAIN_MENU,
    COMBAT,
    EVOLUTION_TREE,
    STORY
}

@export var scene_id: String
@export var scene_type: SceneType
@export var scene_path: String  # e.g., "res://scenes/combat_scene.tscn"
@export var auto_load: bool = false  # 是否预加载
@export var persistent: bool = true  # 是否保持状态
```

### 3.4 场景切换流程

```gdscript
# SceneManager.gd
func change_scene(new_scene_id: String, transition: String = "fade"):
    # 1. 播放过渡动画（淡出）
    await play_transition_out(transition)

    # 2. 保存当前场景状态
    _save_current_scene_state()

    # 3. 卸载旧场景（如非持久化）
    if not _current_scene_config.persistent:
        _unload_current_scene()

    # 4. 加载新场景
    await _load_scene(new_scene_id)

    # 5. 恢复新场景状态（如有）
    _restore_scene_state(new_scene_id)

    # 6. 播放过渡动画（淡入）
    await play_transition_in(transition)
```

### 3.5 过渡动画类型

| 类型 | 描述 | 使用场景 |
|------|------|----------|
| `fade` | 淡入淡出（黑屏） | 战斗↔进化树，主菜单→战斗 |
| `slide_left` | 从左侧滑入 | 战斗开始 |
| `slide_right` | 从右侧滑入 | 返回进化树 |
| `none` | 无过渡，硬切 | 调试模式，快速测试 |

### 3.6 场景状态保存

```gdscript
# 场景状态数据结构
class SceneState:
    var scene_id: String
    var timestamp: int
    var data: Dictionary  # 场景特定数据

# 进化树场景状态示例
{
    "scene_id": "evolution_tree",
    "unlocked_nodes": ["beam_width_1", "beam_damage_1"],
    "current_path": "damage_branch",
    "resource_counts": {"main": 150, "aux": 5}
}

# 战斗场景状态示例
{
    "scene_id": "combat",
    "battle_number": 3,  # 第几次战斗
    "current_duration": 15.5,  # 当前战斗时长
    "collected_resources": 25
}
```

### 3.7 场景间通信

```gdscript
# 使用信号进行场景间通信
signal scene_changed(from_scene: String, to_scene: String)
signal scene_request_exit(scene_id: String)
signal scene_request_transition(target_scene: String)

# 示例：战斗场景请求返回进化树
func on_combat_time_up():
    emit_signal("scene_request_transition", "evolution_tree")
```

---

## 4. 公式 (Formulas)

### 4.1 过渡动画时长

```
标准过渡时长 = 0.3s (300ms)
快速过渡时长 = 0.15s (150ms)  # 调试模式
慢速过渡时长 = 0.5s (500ms)  # 重要场景切换（如首次进入战斗）

# 总切换时间 = 淡出 + 加载 + 淡入
目标总时间 < 2.0s
```

### 4.2 场景加载超时

```
加载超时阈值 = 5.0s
如果加载时间 > 阈值：显示加载提示或跳过过渡动画
```

### 4.3 状态保存频率

```
# 进化树场景：每次点亮节点后保存
# 战斗场景：战斗结束后保存
# 主菜单：无需保存
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **场景加载失败** | 回退到主菜单，显示错误信息（场景路径错误/资源缺失） |
| **过渡动画中请求切换** | 忽略新请求，等待当前过渡完成；或加入队列 |
| **场景状态损坏** | 使用默认状态启动，记录警告日志 |
| **内存不足** | 强制卸载非持久化场景，释放资源 |
| **快速连续切换** | 加入切换队列，按顺序处理；或冷却时间（0.5s 内只响应一次） |
| **存档丢失** | 首次启动使用默认状态（主菜单） |
| **场景间循环依赖** | 通过 SceneManager 中介，避免场景直接互相引用 |

---

## 6. 依赖关系 (Dependencies)

<!-- TODO: 依赖的其他系统和被哪些系统依赖 -->

### 6.1 本系统依赖

| 系统 | 依赖原因 |
|------|----------|
| F1 输入系统 | 读取场景切换输入（确认/取消/暂停） |
| F2 数据配置系统 | 读取场景配置数据 |

### 6.2 依赖本系统

| 系统 | 依赖原因 |
|------|------|
| C2 时间管理系统 | 战斗场景启动计时器 |
| G5 战斗系统 | 战斗场景的管理框架 |
| G4 进化树系统 | 进化树场景的管理框架 |
| P1 UI 系统 | 场景过渡 UI 显示 |
| M1 剧情系统 | 剧情场景的触发和切换 |
| M2 存档系统 | 场景状态保存/加载 |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `TRANSITION_DURATION` | 0.3s | 0.1s - 1.0s | 过渡动画时长 |
| `SCENE_LOAD_TIMEOUT` | 5.0s | 2.0s - 10.0s | 场景加载超时阈值 |
| `MAX_SCENE_HISTORY` | 5 | 2 - 10 | 保存的场景历史数量 |
| `PERSISTENT_SCENE_MEMORY` | 100MB | 50MB - 500MB | 持久化场景内存上限 |
| `TRANSITION_COOLDOWN` | 0.5s | 0.2s - 1.0s | 场景切换冷却时间 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 4 个场景类型定义完成（主菜单、战斗、进化树、剧情）
- [ ] SceneManager 单例实现，场景切换功能正常
- [ ] 淡入淡出过渡动画正常工作
- [ ] 主菜单→战斗→进化树→战斗循环流畅
- [ ] 进化树场景状态保存（节点点亮状态）
- [ ] 战斗场景状态保存（战斗次数、资源）
- [ ] 场景间通信通过信号实现，无直接引用
- [ ] 场景加载失败时有错误提示

### 完整版标准

- [ ] 所有过渡动画类型实现（fade, slide_left, slide_right）
- [ ] 场景状态序列化到文件，重启游戏后恢复
- [ ] 场景加载时间 < 2 秒（目标平台测试）
- [ ] 内存占用在预算内（持久化场景 < 100MB）
- [ ] 调试模式支持快速切换（无过渡）
- [ ] 场景切换冷却机制防止误操作

---

## 附录 A: 场景架构图

### 场景树结构

```
Window (游戏窗口)
├── SceneManager (Node, 单例，自动启动)
│   ├── TransitionLayer (ColorRect + AnimationPlayer)
│   │   └── 用于淡入淡出/滑动效果
│   └── SceneContainer (MarginContainer)
│       └── [当前场景实例]
│           ├── MainMenuScene (主菜单)
│           ├── CombatScene (战斗)
│           ├── EvolutionTreeScene (进化树)
│           └── StoryScene (剧情)
└── AudioListener2D (全局音频监听器)
```

### SceneManager 状态机

```
┌──────────────┐
│  MAIN_MENU   │
│   (初始)     │
└──────┬───────┘
       │ 开始游戏
       ↓
┌──────────────┐     战斗结束      ┌──────────────────┐
│    COMBAT    │ ───────────────→ │  EVOLUTION_TREE  │
│  (10-30 秒)   │                  │   (玩家主动)      │
└──────▲───────┘                  └────────┬─────────┘
       │                                   │
       │         继续战斗                   │
       └───────────────────────────────────┘
```

### 场景切换时序图

```
战斗场景          SceneManager        进化树场景
   │                   │                   │
   │  请求切换          │                   │
   │ ─────────────────→│                   │
   │                   │  保存战斗状态      │
   │                   │  淡出动画          │
   │                   │ ──────→ [黑屏]    │
   │                   │  卸载战斗场景      │
   │                   │  加载进化树场景    │
   │                   │ ──────→ [实例化]  │
   │                   │  恢复状态          │
   │                   │  淡入动画          │
   │                   │ ──────→ [显示]    │
   │  切换完成          │                   │
   │ ←─────────────────│                   │
   ▼                   ▼                   ▼
```

---

## 附录 B: SceneManager 实现

```gdscript
# res://scripts/systems/scene_manager.gd
class_name SceneManager
extends Node

## 场景切换信号
signal scene_changed(from_scene: String, to_scene: String)
signal scene_change_requested(target_scene: String)
signal scene_transition_started()
signal scene_transition_completed()

## 场景枚举
enum SceneType {
    MAIN_MENU,
    COMBAT,
    EVOLUTION_TREE,
    STORY
}

## 场景配置
const SCENE_CONFIG = {
    "main_menu": {
        "path": "res://scenes/main_menu.tscn",
        "persistent": false,
        "transition": "fade"
    },
    "combat": {
        "path": "res://scenes/combat_scene.tscn",
        "persistent": true,
        "transition": "slide_left"
    },
    "evolution_tree": {
        "path": "res://scenes/evolution_tree_scene.tscn",
        "persistent": true,
        "transition": "slide_right"
    },
    "story": {
        "path": "res://scenes/story_scene.tscn",
        "persistent": false,
        "transition": "fade"
    }
}

## 过渡动画时长
const TRANSITION_DURATION = 0.3
const TRANSITION_COOLDOWN = 0.5

var _current_scene_id: String = ""
var _scene_states: Dictionary = {}
var _is_transitioning: bool = false
var _transition_cooldown_timer: float = 0.0

## 切换到目标场景
func change_scene(target_scene_id: String, force: bool = false) -> void:
    if _is_transitioning and not force:
        push_warning("Scene transition in progress, ignoring request")
        return

    if target_scene_id == _current_scene_id and not force:
        return  # 已经是当前场景

    _is_transitioning = true
    emit_signal("scene_transition_started")

    var from_scene = _current_scene_id

    # 保存当前场景状态
    if _current_scene_id.is_not_empty():
        _save_scene_state(_current_scene_id)

    # 播放淡出动画
    await _play_transition_out(SCENE_CONFIG[target_scene_id].transition)

    # 卸载旧场景（如非持久化）
    if not SCENE_CONFIG[_current_scene_id].persistent:
        _unload_scene(_current_scene_id)

    # 加载新场景
    await _load_scene(target_scene_id)

    # 恢复新场景状态
    _restore_scene_state(target_scene_id)

    # 播放淡入动画
    await _play_transition_in(SCENE_CONFIG[target_scene_id].transition)

    _current_scene_id = target_scene_id
    _is_transitioning = false
    _transition_cooldown_timer = TRANSITION_COOLDOWN

    emit_signal("scene_changed", from_scene, target_scene_id)
    emit_signal("scene_transition_completed")

## 保存场景状态
func _save_scene_state(scene_id: String):
    var state = null
    match scene_id:
        "evolution_tree":
            state = _save_evolution_tree_state()
        "combat":
            state = _save_combat_state()

    if state:
        _scene_states[scene_id] = state

## 恢复场景状态
func _restore_scene_state(scene_id: String):
    if scene_id in _scene_states:
        var state = _scene_states[scene_id]
        match scene_id:
            "evolution_tree":
                _restore_evolution_tree_state(state)
            "combat":
                _restore_combat_state(state)
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始版本创建 |

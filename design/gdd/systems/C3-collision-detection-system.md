# C3 碰撞检测系统 (Collision Detection System)

> 系统 ID: C3
> 类别：Core
> 优先级：MVP
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**C3 碰撞检测系统** 负责游戏中所有碰撞检测和判定，是战斗系统的核心基础。

**职责**：
- 玩家 - 弹幕碰撞检测（被弹判定）
- 光束 - 敌人碰撞检测（伤害判定）
- 玩家 - 掉落物碰撞检测（收集判定）
- 擦弹判定（双层 hitbox 机制）
- 碰撞性能优化（空间分区、距离剔除、分组检测）

**碰撞类型**：
| 类型 | 检测对象 | 结果 |
|------|----------|------|
| **被弹检测** | 玩家 hitbox ↔ 弹幕 hitbox | 触发被弹惩罚 |
| **擦弹检测** | 玩家外层 hitbox ↔ 弹幕 hitbox | 触发擦弹加速 |
| **伤害检测** | 光束 hitbox ↔ 敌人 hitbox | 触发伤害计算 |
| **收集检测** | 玩家收集范围 ↔ 掉落物 | 触发资源收集 |

**设计目标**：
- **精确判定**：双层 hitbox 实现擦弹/被弹分离判定
- **高性能**：支持满屏弹幕（目标 500+ 子弹）仍保持 60FPS
- **Godot 原生**：使用 Area2D 信号机制，代码简洁

---

## 2. 玩家幻想 (Player Fantasy)

碰撞检测系统让玩家：
- **精准操作**：擦弹成功时感受到"刚刚好"的快感
- **公平判定**：被弹时心服口服（不是判定欺诈）
- **流畅体验**：满屏弹幕也不卡顿，操作响应及时

**东方 Project 适配**：
- 传统东方弹幕游戏的 hitbox 很小（通常只有角色中心的像素点）
- 本作采用双层 hitbox：
  - 内层：被弹判定（小，约 4-6px 半径）
  - 外层：擦弹判定（中，约 10-15px 半径）
- 视觉反馈：被弹时红色闪光，擦弹时金色闪光

---

## 3. 详细规则 (Detailed Rules)

### 3.1 碰撞层配置 (Godot Collision Layers)

```gdscript
# 碰撞层定义 (bitmask)
enum CollisionLayer {
    LAYER_NONE       = 0b00000,
    LAYER_PLAYER     = 0b00001,  # 1: 玩家本体
    LAYER_PLAYER_HIT = 0b00010,  # 2: 玩家被弹 hitbox（内层）
    LAYER_PLAYER_GRAZE = 0b00100, # 3: 玩家擦弹 hitbox（外层）
    LAYER_BULLET     = 0b01000,  # 4: 敌人弹幕
    LAYER_BEAM       = 0b10000,  # 5: 玩家光束
    LAYER_ENEMY      = 0b100000, # 6: 敌人本体
    LAYER_DROP       = 0b1000000 # 7: 掉落物
}

# 碰撞掩码配置
# player_hitbox: 只与弹幕碰撞
collision_layer = LAYER_PLAYER_HIT
collision_mask = LAYER_BULLET

# player_graze_box: 只与弹幕碰撞（擦弹判定）
collision_layer = LAYER_PLAYER_GRAZE
collision_mask = LAYER_BULLET

# bullet: 与玩家两个 hitbox 都碰撞
collision_layer = LAYER_BULLET
collision_mask = LAYER_PLAYER_HIT | LAYER_PLAYER_GRAZE

# beam: 与敌人碰撞
collision_layer = LAYER_BEAM
collision_mask = LAYER_ENEMY

# enemy: 与光束碰撞
collision_layer = LAYER_ENEMY
collision_mask = LAYER_BEAM

# drop: 与玩家收集范围碰撞
collision_layer = LAYER_DROP
collision_mask = LAYER_PLAYER  # 玩家有一个收集范围 area
```

### 3.2 双层 Hitbox 结构

```
玩家角色 (CharacterBody2D)
├── Sprite (像素图，雾雨魔理沙)
├── PlayerHitbox (Area2D) - 被弹判定
│   └── CollisionShape2D (半径 5px 圆形)
├── PlayerGrazeBox (Area2D) - 擦弹判定
│   └── CollisionShape2D (半径 12px 圆形)
└── CollectRange (Area2D) - 收集范围
    └── CollisionShape2D (半径 50px 圆形，可升级扩大)
```

### 3.3 碰撞检测流程

```
1. 弹幕进入玩家附近
       ↓
2. 首先接触外层 hitbox (GrazeBox)
       ↓
   ┌──┴──┐
   │ 继续接近？ │
   └──┬──┘
    YES │         NO (离开范围)
        ↓
3. 接触内层 hitbox (被弹)
       ↓
4. 触发被弹信号 → 蓄力中断 → 光束提前发射
```

### 3.4 Area2D 信号处理

```gdscript
# 玩家被弹 hitbox
func _on_player_hitbox_area_entered(area):
    if area.is_in_group("enemy_bullets"):
        emit_signal("player_hit", area.bullet_data)

# 玩家擦弹 hitbox
func _on_player_graze_box_area_entered(area):
    if area.is_in_group("enemy_bullets"):
        if not area.has_been_grazed:  # 防止重复触发
            emit_signal("graze_success", area.bullet_data)
            area.has_been_grazed = true

# 玩家收集范围
func _on_collect_range_area_entered(area):
    if area.is_in_group("drops"):
        emit_signal("resource_collected", area.drop_data)

# 光束 hitbox
func _on_beam_area_entered(area):
    if area.is_in_group("enemies"):
        emit_signal("beam_hit_enemy", area.enemy_data, beam_power)
```

### 3.5 性能优化

```gdscript
# 空间分区：使用 Godot 的 Area2D 自动管理
# Godot 内部使用空间哈希表优化碰撞检测

# 距离剔除：远处的弹幕不参与检测
func should_check_collision(bullet_pos: Vector2, player_pos: Vector2) -> bool:
    return bullet_pos.distance_to(player_pos) < MAX_CHECK_DISTANCE
    # MAX_CHECK_DISTANCE = 500px (可配置)

# 分组检测：弹幕按类型分组，批量处理
enum BulletGroup {
    GROUP_BASIC,    # 普通子弹
    GROUP_LASER,    # 激光
    GROUP_BOMB,     # 炸弹
}

# 同组子弹共享碰撞逻辑，减少重复计算
```

### 3.6 命中特效触发

```gdscript
# 碰撞后触发特效
func _on_beam_hit_enemy(enemy, power):
    # 生成命中粒子
    spawn_hit_effect(enemy.position, power)
    # 生成伤害数字
    spawn_damage_number(enemy.position, calculate_damage(power))
```

---

## 4. 公式 (Formulas)

### 4.1 碰撞判定公式

```
# 距离检测（备用方案，用于验证 Area2D 结果）
distance = bullet_pos.distance_to(player_pos)

# 擦弹判定
is_graze = distance <= graze_radius  # 12px

# 被弹判定
is_hit = distance <= hit_radius  # 5px
```

### 4.2 擦弹奖励公式

```
# 每个擦弹减少蓄力时间
graze_charge_bonus = base_graze_bonus * (1 + graze_count * 0.05)
# base_graze_bonus = 0.02s (2%)
# 上限：最多少 50% 蓄力时间
```

### 4.3 性能预算

```
# 目标：60 FPS (16.6ms/frame)
# 碰撞检测预算：< 2ms/frame

# 预期弹幕数量：
# - 普通敌人：50-100 发
# - 精英敌人：200-300 发
# - Boss 阶段（后期）：500+ 发

# 使用 Godot Area2D 自动优化：
# - 空间哈希表 O(1) 平均复杂度
# - 只检测屏幕内 + 边缘的子弹
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **同时被多发射弹击中** | 只触发一次被弹（首次命中），其余忽略 |
| **擦弹和被弹同时触发** | 优先被弹（内层 hitbox 优先级高） |
| **子弹穿透玩家** | 每发子弹只触发一次碰撞（标记已处理） |
| **玩家无敌时间** | 被弹后有 0.5s 无敌，期间跳过碰撞检测 |
| **光束同时击中多个敌人** | 每个敌人独立触发伤害（支持穿透升级） |
| **掉落物重叠** | 每个掉落物独立检测，支持批量收集 |
| **Area2D 信号丢失** | 每帧手动验证一次距离作为 backup |
| **性能骤降** | 动态降低弹幕数量或简化碰撞检测 |

---

## 6. 依赖关系 (Dependencies)

<!-- TODO: 依赖的其他系统和被哪些系统依赖 -->

### 6.1 本系统依赖

| 系统 | 依赖原因 |
|------|----------|
| F1 输入系统 | 玩家移动输入影响碰撞检测位置 |
| C4 玩家控制系统 | 获取玩家当前位置和 hitbox |

### 6.2 依赖本系统

| 系统 | 依赖原因 |
|------|----------|
| C7 伤害计算系统 | 光束击中敌人触发伤害计算 |
| G6 擦弹系统 | 擦弹判定依赖碰撞检测结果 |
| G7 被弹系统 | 被弹判定依赖碰撞检测结果 |
| G3 收集系统 | 掉落物收集判定 |
| P3 特效系统 | 碰撞命中特效触发 |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `HITBOX_RADIUS` | 5.0px | 3.0px - 8.0px | 被弹判定大小（越小越容易） |
| `GRAZEBOX_RADIUS` | 12.0px | 8.0px - 20.0px | 擦弹判定范围（越大越容易擦弹） |
| `COLLECT_RANGE_RADIUS` | 50.0px | 30.0px - 200.0px | 基础收集范围 |
| `INVINCIBILITY_DURATION` | 0.5s | 0.3s - 1.0s | 被弹后无敌时间 |
| `MAX_CHECK_DISTANCE` | 500.0px | 300.0px - 800.0px | 碰撞检测距离上限 |
| `GRAZE_BONUS_BASE` | 0.02s | 0.01s - 0.05s | 基础擦弹加速 |
| `GRAZE_BONUS_CAP` | 0.5 (50%) | 0.3 - 0.7 | 擦弹加速上限 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 玩家双层 hitbox 正确配置（内层 5px，外层 12px）
- [ ] 被弹检测触发正确（玩家 hitbox ↔ 弹幕）
- [ ] 擦弹检测触发正确（玩家 grazebox ↔ 弹幕）
- [ ] 光束 - 敌人伤害检测正常
- [ ] 掉落物收集检测正常
- [ ] Area2D 信号连接正确
- [ ] 碰撞层/掩码配置正确，无误触发
- [ ] 无敌时间机制生效

### 完整版标准

- [ ] 距离剔除生效（> 500px 子弹跳过检测）
- [ ] 分组检测实现（同组子弹共享逻辑）
- [ ] 满屏 300+ 弹幕时仍保持 60 FPS
- [ ] 命中特效正确触发
- [ ] 同时被多发子弹击中时只触发一次被弹
- [ ] 擦弹奖励公式正确计算

---

## 附录 A: 碰撞层配置

### Godot Collision Layers/Masks 详细配置

```gdscript
# res://scripts/systems/collision_config.gd
class_name CollisionConfig

# 碰撞层定义 (bitmask, 最多 32 层)
const LAYER_NONE        = 0b000000000
const LAYER_PLAYER      = 0b000000001  # Bit 0: 玩家本体
const LAYER_PLAYER_HIT  = 0b000000010  # Bit 1: 玩家被弹 hitbox
const LAYER_PLAYER_GRAZE= 0b000000100  # Bit 2: 玩家擦弹 hitbox
const LAYER_PLAYER_COLLECT=0b000001000 # Bit 3: 玩家收集范围
const LAYER_BULLET      = 0b000010000  # Bit 4: 敌人弹幕
const LAYER_BEAM        = 0b000100000  # Bit 5: 玩家光束
const LAYER_ENEMY       = 0b001000000  # Bit 6: 敌人本体
const LAYER_DROP        = 0b010000000  # Bit 7: 掉落物

# 碰撞掩码配置表
const COLLISION_CONFIG = {
    "player_hitbox": {
        "layer": LAYER_PLAYER_HIT,
        "mask": LAYER_BULLET,  # 只与子弹碰撞
    },
    "player_graze_box": {
        "layer": LAYER_PLAYER_GRAZE,
        "mask": LAYER_BULLET,  # 只与子弹碰撞
    },
    "player_collect_range": {
        "layer": LAYER_PLAYER_COLLECT,
        "mask": LAYER_DROP,  # 只与掉落物碰撞
    },
    "enemy_bullet": {
        "layer": LAYER_BULLET,
        "mask": LAYER_PLAYER_HIT | LAYER_PLAYER_GRAZE,  # 与玩家两个 hitbox 都碰撞
    },
    "player_beam": {
        "layer": LAYER_BEAM,
        "mask": LAYER_ENEMY,  # 只与敌人碰撞
    },
    "enemy": {
        "layer": LAYER_ENEMY,
        "mask": LAYER_BEAM,  # 只与光束碰撞
    },
    "drop": {
        "layer": LAYER_DROP,
        "mask": LAYER_PLAYER_COLLECT,  # 只与玩家收集范围碰撞
    },
}
```

---

## 附录 B: CollisionDetector 实现

```gdscript
# res://scripts/systems/collision_detector.gd
class_name CollisionDetector
extends Node

## 碰撞信号
signal player_hit(bullet_data: Dictionary)
signal graze_success(bullet_data: Dictionary)
signal beam_hit_enemy(enemy_data: Dictionary, beam_power: float)
signal resource_collected(drop_data: Dictionary)

## 配置参数
@export var hitbox_radius: float = 5.0
@export var grazebox_radius: float = 12.0
@export var collect_range_radius: float = 50.0
@export var invincibility_duration: float = 0.5
@export var max_check_distance: float = 500.0

var _is_invincible: bool = false
var _grazed_bullets: Array = []  # 已处理的擦弹

func _ready():
    _setup_collision_layers()

## 设置碰撞层
func _setup_collision_layers():
    # 通过 CollisionConfig 配置所有碰撞层
    for obj in CollisionConfig.COLLISION_CONFIG:
        var config = CollisionConfig.COLLISION_CONFIG[obj]
        # 应用到对应的 Area2D 节点

## 开始无敌时间
func start_invincibility():
    _is_invincible = true
    await get_tree().create_timer(invincibility_duration).timeout
    _is_invincible = false

## 检测擦弹去重
func _on_graze_detected(bullet):
    if bullet in _grazed_bullets:
        return  # 已处理，跳过

    _grazed_bullets.append(bullet)
    emit_signal("graze_success", bullet.data)

    # 清理已远离的子弹记录
    _cleanup_grazed_bullets()

func _cleanup_grazed_bullets():
    # 移除距离玩家过远的子弹记录
    _grazed_bullets = _grazed_bullets.filter(
        func(b): b.global_position.distance_to(get_player_position()) < 200
    )

func get_player_position() -> Vector2:
    return get_node("../Player").global_position

func is_invincible() -> bool:
    return _is_invincible
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始版本创建 |

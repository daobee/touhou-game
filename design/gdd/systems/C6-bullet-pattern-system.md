# C6 弹幕系统 (Bullet Pattern System)

> 系统 ID: C6
> 类别：Core
> 优先级：MVP
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**C6 弹幕系统** 负责敌人弹幕的发射、移动模式、性能优化，是东方 Project 弹幕射击的核心特色。

**职责**：
- 弹幕类型定义（普通子弹、激光、炸弹等）
- 弹幕发射模式（直线、瞄准、环形、螺旋、散射等）
- 弹幕性能优化（GPU 粒子，目标 500+ 子弹 60FPS）
- 弹幕 hitbox 管理（与碰撞检测系统联动）
- 弹幕视觉配置（速度、大小、颜色、生命周期）

**设计目标**：
- **东方风格**：参考传统东方弹幕，但强度适中，以限制玩家移动为主
- **高性能**：使用 GPU 粒子渲染，支持满屏弹幕不卡顿
- **易于配置**：策划可通过配置文件定义新弹幕模式
- **可扩展**：新增弹幕类型无需修改核心逻辑

---

## 2. 玩家幻想 (Player Fantasy)

弹幕系统让玩家：
- **紧张刺激**：满屏弹幕中穿梭，感受"一发极限"的快感
- **策略移动**：弹幕略微限制移动路线，需要规划走位
- **视觉震撼**：华丽的弹幕图案，配合东方风格的音效
- **成就感**：无伤通过弹幕密集区，展现技术

**东方 Project 适配**：
- 弹幕密度低于原作，但保持视觉风格
- 弹幕速度适中，给玩家反应时间
- 弹幕图案多样（环形、螺旋、扇形等）
- 擦弹时有金色闪光反馈

---

## 3. 详细规则 (Detailed Rules)

### 3.1 弹幕类型

```gdscript
# res://scripts/data/resources/bullet_config.gd
class_name BulletConfig
extends Resource

enum BulletType {
    NORMAL,     # 普通子弹（圆形，直径 8-12px）
    LASER,      # 激光（长条形，宽 4px，长 40-60px）
    BOMB,       # 炸弹（大圆形，直径 20-30px，爆炸效果）
    PELLET,     # 霰弹（小圆形，直径 4-6px，密集发射）
}

@export var bullet_id: String
@export var bullet_type: BulletType
@export var sprite: Texture2D
@export var color: Color
@export var size: Vector2
@export var speed: float
@export var damage: int  # 用于伤害计算
@export var lifetime: float  # 存活时间（秒）
```

### 3.2 弹幕发射模式

```gdscript
# res://scripts/data/resources/bullet_pattern_config.gd
class_name BulletPatternConfig
extends Resource

enum PatternType {
    STRAIGHT,       # 直线射击（固定方向）
    AIMED,          # 瞄准玩家（追踪）
    RING,           # 环形（360 度均匀发射）
    SPIRAL,         # 螺旋形（旋转发射）
    FAN,            # 扇形（角度范围内散射）
    RANDOM,         # 随机散射
}

@export var pattern_id: String
@export var pattern_type: PatternType

# 直线/扇形参数
@export var fire_direction: float = 0.0  # 弧度，0=向下
@export var fire_angle_range: float = 0.0  # 扇形角度（弧度）

# 环形参数
@export var ring_bullet_count: int = 12  # 环形子弹数量

# 螺旋参数
@export var spiral_rotation_speed: float = 2.0  # 旋转速度（弧度/秒）
@export var spiral_bullet_count: int = 2  # 螺旋臂数量

# 散射参数
@export var scatter_count: int = 5  # 散射子弹数
@export var scatter_spread: float = 0.5  # 散射角度范围
```

### 3.3 弹幕发射器

```gdscript
# res://scripts/systems/bullet_spawner.gd
class_name BulletSpawner
extends Node2D

@export var config: BulletPatternConfig
@export var bullet_config: BulletConfig

var _fire_timer: Timer
var _fire_interval: float = 1.0
var _is_active: bool = true

func _ready():
    _fire_timer = Timer.new()
    _fire_timer.wait_time = _fire_interval
    _fire_timer.timeout.connect(_fire_pattern)
    add_child(_fire_timer)
    _fire_timer.start()

func _fire_pattern():
    if not _is_active:
        return

    match config.pattern_type:
        PatternType.STRAIGHT:
            _fire_straight()
        PatternType.AIMED:
            _fire_aimed()
        PatternType.RING:
            _fire_ring()
        PatternType.SPIRAL:
            _fire_spiral()
        PatternType.FAN:
            _fire_fan()
        PatternType.RANDOM:
            _fire_random()

func _fire_straight():
    var direction = Vector2.RIGHT.rotated(config.fire_direction)
    _spawn_bullet(global_position, direction)

func _fire_aimed():
    var player_pos = get_node_or_null("../Player").global_position
    if player_pos:
        var direction = (player_pos - global_position).normalized()
        _spawn_bullet(global_position, direction)

func _fire_ring():
    for i in range(config.ring_bullet_count):
        var angle = TAU / config.ring_bullet_count * i
        var direction = Vector2.RIGHT.rotated(angle)
        _spawn_bullet(global_position, direction)

func _fire_spiral():
    # 螺旋形：每次发射旋转一定角度
    var base_angle = _fire_timer.times_called() * config.spiral_rotation_speed * 0.1
    for i in range(config.spiral_bullet_count):
        var angle = base_angle + TAU / config.spiral_bullet_count * i
        var direction = Vector2.RIGHT.rotated(angle)
        _spawn_bullet(global_position, direction)

func _fire_fan():
    var start_angle = config.fire_direction - config.fire_angle_range / 2
    for i in range(config.scatter_count):
        var angle = start_angle + config.fire_angle_range / (config.scatter_count - 1) * i
        var direction = Vector2.RIGHT.rotated(angle)
        _spawn_bullet(global_position, direction)

func _fire_random():
    for i in range(config.scatter_count):
        var angle = randf_range(-config.scatter_spread, config.scatter_spread)
        var direction = Vector2.RIGHT.rotated(angle)
        _spawn_bullet(global_position, direction)

func _spawn_bullet(pos: Vector2, dir: Vector2):
    # 由 BulletManager 统一管理
    BulletManager.spawn_bullet(bullet_config, pos, dir)
```

### 3.4 弹幕管理器（GPU 粒子优化）

```gdscript
# res://scripts/systems/bullet_manager.gd
class_name BulletManager
extends Node2D

## 使用 GPU 粒子渲染弹幕
@export var bullet_particle: GPUParticles2D
@export var particle_texture: Texture2D

## 活跃弹幕列表（用于逻辑更新）
var _active_bullets: Array[Dictionary] = []

## 对象池（预分配内存）
var _bullet_pool: Array[Dictionary] = []
const MAX_BULLETS = 1000

func _ready():
    # 预分配对象池
    for i in range(MAX_BULLETS):
        _bullet_pool.append({
            "active": false,
            "position": Vector2.ZERO,
            "direction": Vector2.ZERO,
            "speed": 0.0,
            "lifetime": 0.0,
            "age": 0.0,
        })

func spawn_bullet(config: BulletConfig, pos: Vector2, dir: Vector2):
    # 从对象池获取空闲子弹
    for bullet in _bullet_pool:
        if not bullet.active:
            bullet.active = true
            bullet.position = pos
            bullet.direction = dir.normalized()
            bullet.speed = config.speed
            bullet.lifetime = config.lifetime
            bullet.age = 0.0
            bullet.config = config
            _active_bullets.append(bullet)
            break

func _process(delta):
    # 更新所有活跃子弹
    for i in range(_active_bullets.size() - 1, -1, -1):
        var bullet = _active_bullets[i]
        bullet.age += delta
        bullet.position += bullet.direction * bullet.speed * delta

        # 检查是否超时
        if bullet.age >= bullet.lifetime:
            _despawn_bullet(i)
            continue

        # 更新 GPU 粒子位置
        _update_gpu_particle(i, bullet)

func _despawn_bullet(index: int):
    var bullet = _active_bullets[index]
    bullet.active = false
    _active_bullets.remove_at(index)

    # 归还对象池
    _bullet_pool.append(bullet)

func _update_gpu_particle(index: int, bullet: Dictionary):
    # 更新 GPU 粒子系统的位置数据
    # 具体实现依赖 Godot 的粒子材质参数
    pass
```

### 3.5 弹幕配置示例

```gdscript
# res://assets/data/enemies/bullets/fairy_straight.tres
class_name BulletConfig
extends Resource

bullet_id = "fairy_bullet_01"
bullet_type = BulletType.NORMAL
sprite = preload("res://assets/sprites/bullets/bullet_red_8px.png")
color = Color(1.0, 0.3, 0.3)
size = Vector2(8, 8)
speed = 200.0  # px/s
damage = 1
lifetime = 5.0  # 秒

# 弹幕模式配置
pattern_id = "fairy_straight"
pattern_type = PatternType.STRAIGHT
fire_direction = PI / 2  # 向下
fire_interval = 1.0  # 秒
```

---

## 4. 公式 (Formulas)

### 4.1 弹幕运动公式

```
# 直线运动
position = start_position + direction * speed * time

# 螺旋运动（旋转 + 直线）
angle = base_angle + rotation_speed * time
direction = Vector2.RIGHT.rotated(angle)
position = start_position + direction * speed * time

# 环形运动
angle = TAU / bullet_count * index
direction = Vector2.RIGHT.rotated(angle)
position = center_position + direction * radius
```

### 4.2 弹幕密度计算

```
# 扇形弹幕密度
density = bullet_count / fire_angle_range

# 环形弹幕密度（每 10 度子弹数）
density_per_10deg = ring_bullet_count / (360 / 10)

# 东方风格推荐密度
# 简单：2-3 发/10 度
# 中等：4-6 发/10 度
# 困难：7-10 发/10 度（本作不推荐）
```

### 4.3 弹幕速度分类

```
# 慢速弹幕：100-150 px/s（玩家容易躲避）
# 中速弹幕：150-250 px/s（需要一定反应）
# 快速弹幕：250-400 px/s（高难度）

# 本作推荐：以慢速 - 中速为主，限制移动而非杀伤
```

### 4.4 弹幕存活时间

```
# 根据战斗区域大小计算
lifetime = max(
    play_area_diagonal / speed,  # 飞越对角线的时间
    3.0  # 最小存活时间
)

# 示例：480x320 区域，速度 200px/s
# diagonal = sqrt(480² + 320²) ≈ 577px
# lifetime = 577 / 200 ≈ 2.9s → 取 3.0s
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **弹幕超出屏幕** | 存活时间到期后自动销毁，或距离玩家>800px 时销毁 |
| **对象池耗尽** | 新子弹无法生成，记录警告日志（不应发生，MAX_BULLETS=1000 应足够） |
| **GPU 粒子同步失败** | 回退到 CPU 渲染（Sprite 节点），保证功能正常 |
| **敌人死亡时弹幕仍在飞行** | 弹幕继续存在，直到超时或飞出区域 |
| **多个敌人同时发射** | 每个敌人独立 BulletSpawner，BulletManager 统一管理 |
| **弹幕速度为 0** | 输入验证，拒绝 speed<=0 的配置 |
| **循环引用配置** | 加载时检测（BulletConfig 引用 PatternConfig，反之则报错） |

---

## 6. 依赖关系 (Dependencies)

<!-- TODO: 依赖的其他系统和被哪些系统依赖 -->

### 6.1 本系统依赖

| 系统 | 依赖原因 |
|------|----------|
| F2 数据配置系统 | 读取弹幕配置数据 |
| C4 玩家控制系统 | 获取玩家位置用于瞄准 |
| C7 伤害计算系统 | 被弹时触发惩罚 |

### 6.2 依赖本系统

| 系统 | 依赖原因 |
|------|----------|
| C3 碰撞检测系统 | 弹幕 hitbox 用于碰撞检测 |
| G6 擦弹系统 | 擦弹判定依赖弹幕位置 |
| P3 特效系统 | 弹幕特效、爆炸特效 |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `BULLET_SPEED_SLOW` | 150 px/s | 100-200 px/s | 慢速弹幕速度 |
| `BULLET_SPEED_MEDIUM` | 250 px/s | 200-350 px/s | 中速弹幕速度 |
| `BULLET_SPEED_FAST` | 400 px/s | 300-500 px/s | 快速弹幕速度 |
| `RING_BULLET_COUNT` | 12 | 6-36 | 环形弹幕数量 |
| `SPIRAL_ROTATION_SPEED` | 2.0 rad/s | 1.0-5.0 rad/s | 螺旋旋转速度 |
| `FAN_SPREAD_ANGLE` | 45° (π/4) | 15°-90° | 扇形散射角度 |
| `FIRE_INTERVAL` | 1.0s | 0.3-3.0s | 发射间隔 |
| `BULLET_LIFETIME` | 3.0s | 2.0-10.0s | 子弹存活时间 |
| `MAX_BULLETS` | 1000 | 500-2000 | 对象池容量 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 子弹类型定义完成（NORMAL, LASER, BOMB）
- [ ] 直线射击模式正常工作
- [ ] 瞄准玩家射击模式正常工作
- [ ] 环形射击模式正常工作
- [ ] 扇形/散射射击模式正常工作
- [ ] BulletSpawner 按配置发射弹幕
- [ ] BulletManager 对象池正常工作
- [ ] 子弹超时自动销毁
- [ ] 子弹 hitbox 正确配置（与 C3 碰撞检测联动）
- [ ] GPU 粒子渲染正常工作

### 完整版标准

- [ ] 螺旋射击模式正常工作
- [ ] 随机散射模式正常工作
- [ ] 500+ 子弹同时存在时保持 60 FPS
- [ ] 弹幕配置文件外部化（可策划配置）
- [ ] 弹幕颜色/大小/速度可自定义
- [ ] 敌人死亡后弹幕继续飞行直到销毁
- [ ] 弹幕视觉反馈完整（拖尾、闪光等）

---

## 附录 A: 弹幕 Pattern 配置

### 配置示例：妖精 straight 射击

```gdscript
# res://assets/data/enemies/bullets/fairy_straight.tres
class_name BulletConfig
extends Resource

bullet_id = "fairy_bullet_01"
bullet_type = BulletType.NORMAL
sprite = preload("res://assets/sprites/bullets/bullet_red_8px.png")
color = Color(1.0, 0.3, 0.3)
size = Vector2(8, 8)
speed = 180.0  # px/s (慢速，易于躲避)
damage = 1
lifetime = 4.0  # 秒

# 弹幕模式配置
pattern_id = "fairy_straight"
pattern_type = PatternType.STRAIGHT
fire_direction = PI / 2  # 向下（弧度）
fire_interval = 1.2  # 秒
```

### 配置示例：妖精环形射击

```gdscript
# res://assets/data/enemies/bullets/fairy_ring.tres
class_name BulletConfig
extends Resource

bullet_id = "fairy_bullet_02"
bullet_type = BulletType.NORMAL
sprite = preload("res://assets/sprites/bullets/bullet_blue_6px.png")
color = Color(0.3, 0.5, 1.0)
size = Vector2(6, 6)
speed = 150.0  # px/s
damage = 1
lifetime = 3.0  # 秒

# 弹幕模式配置
pattern_id = "fairy_ring"
pattern_type = PatternType.RING
ring_bullet_count = 12  # 12 发环形
fire_interval = 2.0  # 秒
```

### 配置示例：精英敌人螺旋射击

```gdscript
# res://assets/data/enemies/bullets/elite_spiral.tres
class_name BulletConfig
extends Resource

bullet_id = "elite_bullet_01"
bullet_type = BulletType.NORMAL
sprite = preload("res://assets/sprites/bullets/bullet_purple_10px.png")
color = Color(0.8, 0.3, 1.0)
size = Vector2(10, 10)
speed = 200.0  # px/s
damage = 1
lifetime = 5.0  # 秒

# 弹幕模式配置
pattern_id = "elite_spiral"
pattern_type = PatternType.SPIRAL
spiral_rotation_speed = 3.0  # rad/s
spiral_bullet_count = 2  # 双螺旋
fire_interval = 0.5  # 秒 (快速连续)
```

### 配置示例：扇形散射

```gdscript
# res://assets/data/enemies/bullets/fairy_fan.tres
class_name BulletConfig
extends Resource

bullet_id = "fairy_bullet_03"
bullet_type = BulletType.PELLET
sprite = preload("res://assets/sprites/bullets/bullet_yellow_4px.png")
color = Color(1.0, 0.8, 0.3)
size = Vector2(4, 4)
speed = 220.0  # px/s
damage = 1
lifetime = 2.5  # 秒

# 弹幕模式配置
pattern_id = "fairy_fan"
pattern_type = PatternType.FAN
fire_direction = PI / 2  # 向下
fire_angle_range = PI / 3  # 60 度扇形
scatter_count = 5  # 5 发
fire_interval = 1.5  # 秒
```

---

## 附录 B: 弹幕 Pattern 可视化

```
Pattern 类型图示:

STRAIGHT (直线):
  ↓
  ↓
  ↓

RING (环形，12 发):
      ↑
   ↖     ↗
  ←   ⊙   →
   ↙     ↘
      ↓

SPIRAL (双螺旋):
  ↗ ↖
   ⊙
  ↙ ↘

FAN (扇形，5 发):
  ↙ ↓ ↘
   ⊙

RANDOM (随机散射):
 ↙ ↘ ↓ ↖ ↗ (随机角度)
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始版本创建 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始骨架创建 |

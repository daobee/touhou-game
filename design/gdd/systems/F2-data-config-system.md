# F2 数据配置系统 (Data Configuration System)

> 系统 ID：F2
> 类别：Foundation
> 优先级：MVP
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**F2 数据配置系统** 是游戏的数据驱动核心，负责管理所有可配置的游戏数据。

**职责**：
- 定义 Godot Resource 类结构（`.tres` 文件）用于类型安全的数据存储
- 提供配置加载器，在游戏启动时加载所有配置
- 支持开发时热重载，修改配置后无需重启游戏
- 为其他系统提供统一的配置读取 API

**设计目标**：
- **类型安全**：Godot Resource 在编辑器中显示类型错误
- **易於调整**：策划/开发者可直接在 Godot 编辑器中修改数值
- **热重载**：开发时修改配置后立即生效（通过 `FileSystem.changed` 信号）
- **集中管理**：所有配置存放在 `assets/data/` 目录，结构清晰

---

## 2. 玩家幻想 (Player Fantasy)

数据配置系统是**幕后**系统，玩家不会直接感受到它的存在，但会影响：

- **平衡性**：通过外部配置，可快速迭代调整游戏平衡（敌人血量、伤害、掉落率）
- **Mod 支持**：未来可开放配置文件，允许玩家自定义数值
- **多语言**：文本配置与代码分离，支持本地化

**开发者体验**：
- 在 Godot 编辑器中直接修改数值，无需改代码
- 热重载功能让测试迭代更快（修改 → 切换窗口 → 立即生效）

---

## 3. 详细规则 (Detailed Rules)

### 3.1 配置文件目录结构

```
assets/data/
├── enemies/              # 敌人配置
│   ├── basic_enemy.tres
│   ├── fast_enemy.tres
│   └── elite_enemy.tres
├── evolution_tree/       # 进化树配置
│   ├── nodes/            # 节点配置
│   │   ├── beam_width_node.tres
│   │   ├── beam_damage_node.tres
│   │   └── auto_fire_node.tres
│   └── paths/            # 进化树路径配置
├── drops/                # 掉落配置
│   ├── basic_drop_table.tres
│   └── elite_drop_table.tres
├── combat/               # 战斗公式配置
│   ├── damage_formulas.tres
│   └── penalty_config.tres
└── global/               # 全局配置
    └── game_constants.tres
```

### 3.2 Resource 类结构

```gdscript
# res://scripts/data/resources/enemy_config.gd
class_name EnemyConfig
extends Resource

@export_group("基础属性")
@export var enemy_name: String
@export var enemy_id: String
@export var max_health: float = 100.0
@export var move_speed: float = 50.0

@export_group("弹幕配置")
@export var bullet_pattern: BulletPattern
@export var bullet_speed: float = 200.0
@export var fire_interval: float = 1.0

@export_group("掉落配置")
@export var drop_table: DropTableConfig
@export var main_resource_drop: int = 1
@export var aux_resource_drop_chance: float = 0.1

@export_group("行为配置")
@export var ai_behavior: AIBehaviorType
@export var aggro_range: float = 300.0
```

### 3.3 配置加载器

```gdscript
# res://scripts/systems/config_loader.gd
class_name ConfigLoader

# 单例实例
static var instance: ConfigLoader = null

# 配置缓存
var _enemy_configs: Dictionary[String, EnemyConfig] = {}
var _evolution_node_configs: Dictionary[String, EvolutionNodeConfig] = {}
var _drop_tables: Dictionary[String, DropTableConfig] = {}

static func load_all_configs():
    instance = ConfigLoader.new()
    instance._load_enemy_configs()
    instance._load_evolution_configs()
    instance._load_drop_configs()

static func get_enemy(enemy_id: String) -> EnemyConfig:
    return instance._enemy_configs.get(enemy_id)

static func get_evolution_node(node_id: String) -> EvolutionNodeConfig:
    return instance._evolution_node_configs.get(node_id)
```

### 3.4 热重载机制

```gdscript
# 开发时热重载
func _setup_hot_reload():
    var file_system = EditorFileSystem.new()
    file_system.filesystem_changed.connect(self._on_filesystem_changed)

func _on_filesystem_changed():
    # 检测配置文件变化并重新加载
    _reload_modified_configs()
```

### 3.5 配置验证

- 游戏启动时验证所有配置的完整性
- 检查引用是否存在（如敌人引用的掉落表）
- 检查数值是否在合理范围内（如血量 > 0）

---

## 4. 公式 (Formulas)

### 4.1 配置加载优先级

```
1. 加载全局配置 (game_constants.tres)
2. 加载掉落配置 (drop_tables)
3. 加载敌人配置 (enemies) — 依赖掉落表
4. 加载进化树配置 (evolution_tree)
5. 加载战斗公式配置 (combat)
```

### 4.2 配置验证公式

```gdscript
# 敌人血量验证
assert(enemy.max_health > 0, "敌人血量必须 > 0")
assert(enemy.bullet_speed > 0, "弹幕速度必须 > 0")
assert(enemy.fire_interval > 0.01, "射击间隔不能为 0")

# 掉落概率验证
assert(drop_table.total_probability <= 1.0, "掉落概率总和不能超过 100%")

# 进化树节点验证
assert(node.cost >= 0, "节点消耗不能为负")
assert(node.effect_value != 0, "节点效果不能为空")
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **配置文件丢失** | 启动时报错，显示缺失的文件路径；使用默认值填充 |
| **配置格式错误** | Godot 编辑器会提示，运行时使用 `ResourceLoader.load()` 失败检测 |
| **循环引用** | 加载时检测（A 配置引用 B，B 配置引用 A）并报错 |
| **热重载失败** | 回退到上一次成功加载的配置，显示错误日志 |
| **配置值越界** | 验证时 clamp 到合理范围，显示警告日志 |
| **引用不存在** | 加载时验证所有 `@export` 的 Resource 路径是否有效 |

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
| C7 伤害计算系统 | 读取敌人血量、伤害参数 |
| C8 掉落物系统 | 读取掉落表配置 |
| G2 敌人系统 | 读取敌人类型、弹幕 pattern 配置 |
| G4 进化树系统 | 读取节点配置、升级效果 |
| G8 敌人升级系统 | 读取升级倍率配置 |
| M3 成就系统 | 读取成就配置 |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `CONFIG_LOAD_TIMEOUT` | 5.0s | 2.0s - 10.0s | 配置加载超时时间 |
| `HOT_RELOAD_ENABLED` | true (开发) / false (发布) | 布尔 | 是否启用热重载 |
| `HOT_RELOAD_INTERVAL` | 0.5s | 0.2s - 2.0s | 热重载检测间隔 |
| `VALIDATION_STRICT_MODE` | true (开发) / false (发布) | 布尔 | 是否严格验证配置 |
| `DEFAULT_FALLBACK_ENABLED` | true | 布尔 | 配置缺失时是否使用默认值 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 所有 Resource 类定义完成（EnemyConfig, EvolutionNodeConfig, DropTableConfig 等）
- [ ] 配置加载器实现，可加载 `.tres` 文件
- [ ] 敌人配置可加载并通过 ID 查询
- [ ] 进化树节点配置可加载
- [ ] 掉落表配置可加载
- [ ] 配置验证逻辑实现（血量>0、概率<=1 等）
- [ ] 配置缺失时有默认值或错误提示

### 完整版标准

- [ ] 热重载功能在开发模式下工作
- [ ] Godot 编辑器中可直接创建和修改配置
- [ ] 所有战斗公式参数外部化到配置
- [ ] 配置文件完整测试（所有字段都有测试数据）
- [ ] 发布构建时禁用热重载以减少包体大小

---

## 附录 A: 配置文件结构

### 敌人配置示例

```gdscript
# res://assets/data/enemies/basic_enemy.tres
class_name EnemyConfig
extends Resource

# 基础属性
@export var enemy_name: String = "杂兵妖精"
@export var enemy_id: String = "enemy_basic_fairy"
@export var max_health: float = 50.0
@export var move_speed: float = 80.0

# 弹幕配置
@export var bullet_pattern: BulletPattern = BulletPattern.SPREAD
@export var bullet_speed: float = 150.0
@export var fire_interval: float = 1.5

# 掉落配置
@export var main_resource_drop: int = 1
@export var aux_resource_drop_chance: float = 0.0

# 行为配置
@export var ai_behavior: AIBehaviorType = AIBehaviorType.PATROL
@export var aggro_range: float = 400.0
```

### 进化树节点配置示例

```gdscript
# res://assets/data/evolution_tree/nodes/beam_width_1.tres
class_name EvolutionNodeConfig
extends Resource

# 基础属性
@export var node_id: String = "beam_width_1"
@export var node_name: String = "魔炮宽度 +1"
@export var node_description: String = "光束宽度增加 20%"
@export var node_type: EvolutionNodeType = EvolutionNodeType.PASSIVE

# 消耗配置
@export var cost: int = 10
@export var cost_type: ResourceType = ResourceType.MAIN

# 前置节点
@export var prerequisites: Array[String] = []  # 空表示起始节点

# 效果配置
@export var effect_type: EffectType = EffectType.BEAM_WIDTH
@export var effect_value: float = 0.2  # +20%
@export var effect_description: String = "光束宽度增加 {value}%"
```

### 掉落表配置示例

```gdscript
# res://assets/data/drops/basic_drop_table.tres
class_name DropTableConfig
extends Resource

@export var drop_table_id: String = "drop_table_basic"

@export_group("主要资源掉落")
@export var main_resource_min: int = 1
@export var main_resource_max: int = 3

@export_group("辅助资源掉落")
@export var aux_resource_enabled: bool = false
@export var aux_resource_chance: float = 0.0
@export var aux_resource_count: int = 1

@export_group("特殊掉落")
@export var special_drop_chance: float = 0.0
@export var special_drop_id: String = ""
```

### 战斗公式配置示例

```gdscript
# res://assets/data/combat/damage_formulas.tres
class_name CombatFormulaConfig
extends Resource

# 光束伤害公式参数
@export_group("光束基础伤害")
@export var beam_base_damage: float = 10.0
@export var beam_width_multiplier: float = 1.5
@export var beam_charge_multiplier: float = 2.0

# 被弹惩罚参数
@export_group("被弹惩罚")
@export var interrupted_damage_multiplier: float = 0.5
@export var interrupted_beam_width_multiplier: float = 0.5
@export var interrupted_duration_multiplier: float = 0.5

# 敌人血量升级倍率
@export_group("敌人升级")
@export var enemy_health_upgrade_multiplier: float = 1.2  # +20% 每级
@export var enemy_bullet_density_upgrade: float = 1.1  # +10% 每级
@export var enemy_drop_bonus_multiplier: float = 1.25  # +25% 每级
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始版本创建 |

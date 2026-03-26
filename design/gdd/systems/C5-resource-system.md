# C5 资源系统 (Resource System)

> 系统 ID: C5
> 类别：Core
> 优先级：MVP
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**C5 资源系统** 负责游戏中所有资源的管理，包括资源的获取、存储、消耗和统计。

**资源类型**：
- **主要资源** (Main Resource) — 红色，普通敌人掉落，用于点亮普通进化节点
- **辅助资源** (Aux Resource) — 蓝色/紫色等，精英敌人掉落，用于解锁精英节点/阶段节点
- **特殊货币** (Special Currency) — 金色，成就/里程碑奖励，用于解锁新角色/皮肤等（可选）

**职责**：
- 资源数据结构定义（Godot Resource）
- 资源获取（从掉落物系统接收）
- 资源存储（无限上限）
- 资源消耗（进化树节点点亮）
- 资源统计（成就系统查询）

**设计目标**：
- **简单清晰**：主要资源=普通升级，辅助资源=关键升级
- **无上限设计**：玩家不用担心溢出，放心收集
- **易于扩展**：新增资源类型只需添加配置

---

## 2. 玩家幻想 (Player Fantasy)

资源系统让玩家：
- **收集快感**：看到资源数字增长，正反馈强烈
- **无压力收集**：无上限设计，不用担心溢出浪费
- **目标清晰**：主要资源=变强，辅助资源=解锁新能力
- **成长可视**：资源消耗转化为进化树进度，看得见变强

**视觉反馈**：
- 主要资源：红色光点，小体型
- 辅助资源：蓝色/紫色光点，中等体型，带微光
- 特殊货币：金色光点，大体型，带粒子特效

---

## 3. 详细规则 (Detailed Rules)

### 3.1 资源数据结构

```gdscript
# res://scripts/data/resources/resource_config.gd
class_name ResourceConfig
extends Resource

enum ResourceType {
    MAIN,       # 主要资源（红色，普通敌人掉落）
    AUX_BLUE,   # 辅助资源 - 蓝（精英节点）
    AUX_PURPLE, # 辅助资源 - 紫（阶段节点）
    SPECIAL,    # 特殊货币（金色，成就奖励）
}

@export var resource_id: String
@export var resource_type: ResourceType
@export var resource_name: String
@export var description: String
@export var icon: Texture2D
@export var color: Color
```

### 3.2 资源管理器

```gdscript
# res://scripts/systems/resource_manager.gd
class_name ResourceManager
extends Node

signal resource_changed(resource_type: ResourceType, new_amount: int)
signal resource_added(resource_type: ResourceType, amount: int)
signal resource_spent(resource_type: ResourceType, amount: int)

## 资源存储（无限上限）
var _resources: Dictionary = {
    ResourceType.MAIN: 0,
    ResourceType.AUX_BLUE: 0,
    ResourceType.AUX_PURPLE: 0,
    ResourceType.SPECIAL: 0,
}

## 添加资源
func add_resource(type: ResourceType, amount: int) -> void:
    if amount < 0:
        push_warning("Cannot add negative resource")
        return

    _resources[type] += amount
    emit_signal("resource_added", type, amount)
    emit_signal("resource_changed", type, _resources[type])

## 消耗资源
func spend_resource(type: ResourceType, amount: int) -> bool:
    if amount < 0:
        push_warning("Cannot spend negative resource")
        return false

    if _resources[type] < amount:
        return false  # 资源不足

    _resources[type] -= amount
    emit_signal("resource_spent", type, amount)
    emit_signal("resource_changed", type, _resources[type])
    return true

## 查询资源
func get_resource(type: ResourceType) -> int:
    return _resources.get(type, 0)

## 是否有足够资源
func has_enough(type: ResourceType, amount: int) -> bool:
    return _resources.get(type, 0) >= amount

## 重置资源（用于调试/重启）
func reset_resources():
    for key in _resources:
        _resources[key] = 0
```

### 3.3 资源获取流程

```
战斗场景
   │
   ↓ 击败敌人
   ↓
掉落物系统生成资源掉落
   │
   ↓ 玩家收集
   ↓
收集系统触发 signal
   │
   ↓
ResourceManager.add_resource()
   │
   ├──→ 更新 UI 显示
   ├──→ 通知成就系统（统计）
   └──→ 保存进度（存档系统）
```

### 3.4 资源消耗流程

```
进化树场景
   │
   ↓ 玩家点击节点
   ↓
检查资源是否足够
   │
   ├─→ 足够：spend_resource() → 点亮节点
   │
   └─→ 不足：显示提示"资源不足"
```

### 3.5 资源掉落配置

```gdscript
# res://assets/data/drops/resource_drop_config.tres
class_name ResourceDropConfig
extends Resource

@export_group("主要资源掉落")
@export var main_resource_base_drop: int = 1      # 基础掉落
@export var main_resource_elite_bonus: int = 2    # 精英敌人 bonus

@export_group("辅助资源掉落")
@export var aux_blue_drop_chance: float = 1.0     # 100% 掉落（精英敌人）
@export var aux_purple_drop_chance: float = 0.5   # 50% 掉落（阶段 Boss）

@export_group("特殊货币掉落")
@export var special_drop_chance: float = 0.0      # 0%（成就获取）
```

---

## 4. 公式 (Formulas)

### 4.1 资源掉落计算

```
# 普通敌人
main_resource_drop = base_drop × enemy_upgrade_bonus
# base_drop = 1-3
# enemy_upgrade_bonus = 1.0 + (upgrade_level × 0.25)

# 精英敌人
main_resource_drop = base_drop × 2  # 双倍主要资源
aux_blue_drop = 1                   # 100% 掉落辅助资源（蓝）

# 阶段 Boss
aux_purple_drop = 1                 # 100% 掉落辅助资源（紫）
```

### 4.2 进化树节点消耗

```
# 普通节点
cost = base_cost × node_tier
# base_cost = 10
# tier_1 = 10, tier_2 = 20, tier_3 = 30, ...

# 精英节点
cost = aux_resource_count
# 通常 = 1-3 个辅助资源

# 阶段节点
cost = aux_purple_count
# 通常 = 1 个紫色辅助资源
```

### 4.3 资源统计

```
# 单场战斗收集
battle_total = Σ(resource_drops)

# 累计收集（成就）
lifetime_total = Σ(all_battle_totals)

# 累计消耗（成就）
lifetime_spent = Σ(all_node_costs)
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **资源不足时尝试点亮节点** | 显示提示"资源不足"，不执行消耗 |
| **同时收集多个资源** | 逐个添加，每个触发一次 signal |
| **资源显示 UI 未加载** | 缓存更新，UI 加载后刷新 |
| **存档损坏导致资源丢失** | 从进化树状态反推恢复 |
| **负数资源（作弊/漏洞）** | 输入验证，拒绝负数操作 |
| **资源溢出（超过 int 范围）** | 使用 int64，理论上不会溢出 |
| **消耗时并发修改** | 加锁或队列处理，防止竞态 |

---

## 6. 依赖关系 (Dependencies)

<!-- TODO: 依赖的其他系统和被哪些系统依赖 -->

### 6.1 本系统依赖

| 系统 | 依赖原因 |
|------|----------|
| F2 数据配置系统 | 读取资源配置、掉落配置 |
| C8 掉落物系统 | 获取掉落的资源 |

### 6.2 依赖本系统

| 系统 | 依赖原因 |
|------|----------|
| G4 进化树系统 | 消耗资源点亮节点 |
| G8 敌人升级系统 | 消耗资源升级敌人 |
| M3 成就系统 | 统计资源收集成就 |
| P1 UI 系统 | 显示资源数量 |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `MAIN_RESOURCE_BASE_DROP` | 1-3 | 1-5 | 普通敌人基础掉落 |
| `MAIN_RESOURCE_ELITE_BONUS` | ×2 | ×1.5 - ×3 | 精英敌人主要资源倍率 |
| `AUX_BLUE_DROP_CHANCE` | 100% | 50% - 100% | 蓝色辅助资源掉落率 |
| `AUX_PURPLE_DROP_CHANCE` | 50% | 25% - 100% | 紫色辅助资源掉落率 |
| `NODE_BASE_COST` | 10 | 5 - 20 | 普通节点基础消耗 |
| `NODE_COST_SCALING` | +10/级 | +5 - +20/级 | 节点消耗递增 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 主要资源数据结构定义完成
- [ ] 辅助资源数据结构定义完成
- [ ] ResourceManager 单例实现
- [ ] add_resource() 功能正常
- [ ] spend_resource() 功能正常
- [ ] 资源不足时返回 false
- [ ] 资源变化信号正确触发
- [ ] UI 订阅信号并更新显示
- [ ] 进化树节点消耗资源逻辑正常

### 完整版标准

- [ ] 多种辅助资源支持（蓝、紫）
- [ ] 资源掉落配置外部化（DataConfig）
- [ ] 累计统计功能（成就系统查询）
- [ ] 资源获取/消耗日志（调试用）
- [ ] 存档系统正确保存/加载资源
- [ ] 无上限设计验证（大数值测试）

---

## 附录 A: 资源配置示例

### 主要资源配置

```gdscript
# res://assets/data/resources/main_resource.tres
class_name ResourceConfig
extends Resource

resource_id = "main_resource"
resource_type = ResourceType.MAIN
resource_name = "灵力"
description = "击败敌人获得的灵力，用于点亮普通进化节点"
icon = preload("res://assets/sprites/resources/main_resource.png")
color = Color(1.0, 0.3, 0.3)  # 红色
```

### 辅助资源配置（蓝）

```gdscript
# res://assets/data/resources/aux_blue_resource.tres
class_name ResourceConfig
extends Resource

resource_id = "aux_blue"
resource_type = ResourceType.AUX_BLUE
resource_name = "精灵石"
description = "精英敌人掉落的精灵石，用于解锁精英节点"
icon = preload("res://assets/sprites/resources/aux_blue.png")
color = Color(0.3, 0.5, 1.0)  # 蓝色
```

### 辅助资源配置（紫）

```gdscript
# res://assets/data/resources/aux_purple_resource.tres
class_name ResourceConfig
extends Resource

resource_id = "aux_purple"
resource_type = ResourceType.AUX_PURPLE
resource_name = "境界玉"
description = "阶段 Boss 掉落的境界玉，用于解锁阶段节点"
icon = preload("res://assets/sprites/resources/aux_purple.png")
color = Color(0.8, 0.3, 1.0)  # 紫色
```

---

## 附录 B: ResourceManager 完整实现

```gdscript
# res://scripts/systems/resource_manager.gd
class_name ResourceManager
extends Node

## 资源信号
signal resource_changed(resource_type: ResourceType, new_amount: int)
signal resource_added(resource_type: ResourceType, amount: int)
signal resource_spent(resource_type: ResourceType, amount: int)
signal resource_insufficient(resource_type: ResourceType, required: int, current: int)

## 资源类型枚举
enum ResourceType {
    MAIN,
    AUX_BLUE,
    AUX_PURPLE,
    SPECIAL,
}

## 资源存储（无限上限）
var _resources: Dictionary = {
    ResourceType.MAIN: 0,
    ResourceType.AUX_BLUE: 0,
    ResourceType.AUX_PURPLE: 0,
    ResourceType.SPECIAL: 0,
}

## 累计统计（成就用）
var _lifetime_stats: Dictionary = {
    ResourceType.MAIN: {"gained": 0, "spent": 0},
    ResourceType.AUX_BLUE: {"gained": 0, "spent": 0},
    ResourceType.AUX_PURPLE: {"gained": 0, "spent": 0},
    ResourceType.SPECIAL: {"gained": 0, "spent": 0},
}

func _ready():
    # 单例模式
    if has_node("/root/ResourceManager"):
        queue_free()
        return
    name = "ResourceManager"
    add_to_group("systems")

## 添加资源
func add_resource(type: ResourceType, amount: int) -> void:
    if amount < 0:
        push_warning("Cannot add negative resource: %s" % amount)
        return

    _resources[type] += amount
    _lifetime_stats[type]["gained"] += amount

    emit_signal("resource_added", type, amount)
    emit_signal("resource_changed", type, _resources[type])

    # 调试日志
    if OS.is_debug_build():
        print("[Resource] Added %d %s, total: %d" % [amount, type, _resources[type]])

## 消耗资源
func spend_resource(type: ResourceType, amount: int) -> bool:
    if amount < 0:
        push_warning("Cannot spend negative resource: %s" % amount)
        return false

    if not has_enough(type, amount):
        emit_signal("resource_insufficient", type, amount, _resources[type])
        return false

    _resources[type] -= amount
    _lifetime_stats[type]["spent"] += amount

    emit_signal("resource_spent", type, amount)
    emit_signal("resource_changed", type, _resources[type])

    # 调试日志
    if OS.is_debug_build():
        print("[Resource] Spent %d %s, remaining: %d" % [amount, type, _resources[type]])

    return true

## 查询当前资源
func get_resource(type: ResourceType) -> int:
    return _resources.get(type, 0)

## 查询累计统计
func get_lifetime_stats(type: ResourceType) -> Dictionary:
    return _lifetime_stats.get(type, {"gained": 0, "spent": 0})

## 是否有足够资源
func has_enough(type: ResourceType, amount: int) -> bool:
    return _resources.get(type, 0) >= amount

## 重置所有资源（调试用）
func reset_resources():
    for key in _resources:
        _resources[key] = 0
    emit_signal("resource_changed", -1, 0)  # -1 表示全部重置

## 保存进度
func save_progress() -> Dictionary:
    return {
        "resources": _resources,
        "lifetime_stats": _lifetime_stats,
    }

## 加载进度
func load_progress(data: Dictionary):
    if "resources" in data:
        _resources = data["resources"]
    if "lifetime_stats" in data:
        _lifetime_stats = data["lifetime_stats"]

    # 通知 UI 刷新
    for type in _resources:
        emit_signal("resource_changed", type, _resources[type])
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始版本创建 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始骨架创建 |

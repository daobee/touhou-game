# F3 本地化系统 (Localization System)

> 系统 ID：F3
> 类别：Foundation
> 优先级：Alpha
> 版本：0.1 (骨架)
> 最后更新：2026-03-26
> 状态：设计中

---

## 1. 概述 (Overview)

**F3 本地化系统** 负责游戏的多语言支持，让玩家可以选择自己偏好的语言进行游戏。

**支持语言**：
- 简体中文 (zh_CN)
- 日本語 (ja_JP)
- English (en_US)

**职责**：
- 使用 Godot GetText 系统 (`.po`/`.mo` 文件) 管理翻译
- 提供运行时语言切换功能（立即生效，无需重启）
- 管理多语言字体（Noto Sans CJK 为默认，支持扩展其他字体）
- 为 UI 系统和剧情系统提供翻译文本查询 API

**设计目标**：
- **易于翻译**：翻译人员可直接编辑 `.po` 文件
- **运行时切换**：玩家可随时切换语言，所有文本立即更新
- **字体灵活**：支持为不同语言配置不同字体
- **自动化**：使用 Godot 内置工具提取翻译文本

---

## 2. 玩家幻想 (Player Fantasy)

本地化系统让玩家：
- **母语体验**：用自己最熟悉的语言享受游戏
- **自由选择**：随时切换语言（如日语配音 + 中文字幕）
- **无违和感**：字体与游戏美术风格统一

**东方 Project 粉丝向适配**：
- 日语是原作语言，日语玩家应感受到原汁原味
- 中文翻译应保留东方 Project 的中文圈常用译名（如"博丽灵梦"、"雾雨魔理沙"）
- 英语翻译应保留部分日文术语（如"Danmaku"、"Spell Card"）

---

## 3. 详细规则 (Detailed Rules)

### 3.1 翻译文件目录结构

```
assets/locale/
├── translations/         # 翻译文件
│   ├── messages_en.po    # English
│   ├── messages_zh.po    # 简体中文
│   └── messages_ja.po    # 日本語
├── fonts/                # 字体文件
│   ├── NotoSansCJK-Regular.ttf
│   ├── NotoSansCJK-Bold.ttf
│   └── [自定义字体]/
└── locale_config.tres    # 本地化配置
```

### 3.2 语言配置

```gdscript
# res://assets/locale/locale_config.tres
class_name LocaleConfig
extends Resource

const SUPPORTED_LOCALES = ["en_US", "zh_CN", "ja_JP"]
const DEFAULT_LOCALE = "zh_CN"

# 字体配置（按语言）
const FONT_CONFIG = {
    "en_US": "res://assets/locale/fonts/NotoSans-Regular.ttf",
    "zh_CN": "res://assets/locale/fonts/NotoSansCJK-Regular.ttf",
    "ja_JP": "res://assets/locale/fonts/NotoSansCJK-Regular.ttf",
}

# 字体大小（不同语言可能需要不同字号）
const FONT_SIZE_CONFIG = {
    "en_US": {"default": 16, "small": 12, "large": 24},
    "zh_CN": {"default": 18, "small": 14, "large": 26},
    "ja_JP": {"default": 18, "small": 14, "large": 26},
}
```

### 3.3 翻译文本提取

```bash
# 使用 Godot 工具提取可翻译文本
godot --do-export-translations .

# 生成 messages.pot 模板文件
# 然后翻译为 messages_en.po, messages_zh.po, messages_ja.po
```

### 3.4 代码中使用翻译

```gdscript
# 使用 tr() 函数获取翻译文本
label.text = tr("ui_main_menu_start")  # 根据当前语言返回对应文本

# 带参数的翻译
tr("enemy_defeated").format({"enemy_name": enemy.name})

# 复数形式翻译
tr_n("item_collected", "items_collected", count)
```

### 3.5 语言切换流程

```gdscript
# LocalizationManager.gd
func set_locale(locale: String) -> bool:
    if locale not in SUPPORTED_LOCALES:
        push_warning("Unsupported locale: %s" % locale)
        return false

    TranslationServer.set_locale(locale)
    _update_fonts(locale)
    _notify_ui_refresh()  # 通知所有 UI 刷新文本
    return true

func _update_fonts(locale: String):
    var font_path = FONT_CONFIG[locale]
    var font = load(font_path)
    # 通知所有 Label 节点切换字体
```

### 3.6 翻译键命名规范

```
<场景>_<元素>_<描述>

示例:
ui_main_menu_start       # 主菜单 - 开始游戏
ui_hud_health           # HUD - 血量显示
enemy_goblin_name       # 敌人 - 哥布林 - 名称
item_potion_name        # 物品 - 药水 - 名称
story_chapter1_title    # 剧情 - 第 1 章 - 标题
```

---

## 4. 公式 (Formulas)

### 4.1 文本长度适配

```
# 不同语言的文本长度差异系数（相对于中文）
LENGTH_FACTOR = {
    "zh_CN": 1.0,   # 中文最紧凑
    "ja_JP": 1.1,   # 日文稍长
    "en_US": 1.5,   # 英文可能长 50%
}

# UI 元素宽度适配
element_width = base_width * LENGTH_FACTOR[locale]
```

### 4.2 字体回退机制

```
# 如果指定字体缺少某些字符，使用回退字体
if not font.has_glyph(char):
    font = fallback_font  # Noto Sans CJK 作为通用回退
```

---

## 5. 边缘情况 (Edge Cases)

| 情况 | 处理方式 |
|------|----------|
| **翻译文本缺失** | 使用默认语言（中文）显示，记录警告日志 |
| **字体加载失败** | 回退到 Godot 默认字体，记录错误日志 |
| **翻译中包含特殊字符** | 使用 Unicode 转义，`.po` 文件原生支持 UTF-8 |
| **运行时切换语言后 UI 未更新** | 发送 `locale_changed` 信号，UI 订阅后刷新 |
| **文本过长超出 UI 边界** | 自动缩小字号或换行；标题可截断加省略号 |
| **复数形式处理** | 使用 `tr_n()` 支持复数翻译（英文有单复数，中日文无） |
| **变量插值** | 使用 `.format()` 方法，翻译文本中包含 `{var_name}` 占位符 |
| ** RTL 语言** | 当前不支持（无需配置） |

---

## 6. 依赖关系 (Dependencies)

<!-- TODO: 依赖的其他系统和被哪些系统依赖 -->

### 6.1 本系统依赖

| 系统 | 依赖原因 |
|------|----------|
| F2 数据配置系统 | 读取本地化配置数据 |
| C1 场景管理系统 | 场景切换时可能需要切换语言资源 |

### 6.2 依赖本系统

| 系统 | 依赖原因 |
|------|----------|
| P1 UI 系统 | 所有 UI 文本需要多语言支持 |
| M1 剧情系统 | 剧情文本需要多语言支持 |

---

## 7. 调优旋钮 (Tuning Knobs)

| 参数 | 默认值 | 可调范围 | 影响 |
|------|--------|----------|------|
| `DEFAULT_LOCALE` | "zh_CN" | 支持的语言之一 | 游戏首次启动的默认语言 |
| `AUTO_DETECT_LOCALE` | true | 布尔 | 是否根据系统语言自动选择 |
| `FONT_CACHE_SIZE` | 5 | 3 - 10 | 预加载的字体数量 |
| `UI_REFRESH_DELAY` | 0.05s | 0.02s - 0.2s | 语言切换后 UI 刷新延迟 |
| `FALLBACK_LOCALE` | "en_US" | 支持的语言之一 | 翻译缺失时的回退语言 |

---

## 8. 接受标准 (Acceptance Criteria)

### MVP 标准

- [ ] 3 种语言配置正确（zh_CN, ja_JP, en_US）
- [ ] `.po` 翻译文件结构正确，可被 Godot 加载
- [ ] 主菜单和 HUD 的所有文本可翻译
- [ ] 语言切换功能正常工作（无需重启游戏）
- [ ] Noto Sans CJK 字体正确加载并应用于中日文
- [ ] 英文使用无衬线字体（Noto Sans）
- [ ] 翻译缺失时回退到默认语言

### 完整版标准

- [ ] 所有 UI 文本 100% 可翻译
- [ ] 所有剧情文本 100% 可翻译
- [ ] 字体可配置（支持后续替换为更符合美术风格的字体）
- [ ] 文本长度适配（英文不溢出 UI 边界）
- [ ] 复数形式翻译正确（英文单复数，中日文无）
- [ ] 变量插值翻译正确（如"获得 {count} 个资源"）
- [ ] 语言选择保存在配置文件中，下次启动记住

---

## 附录 A: 翻译文件结构

### messages.po 文件示例

```po
# messages_zh.po - 简体中文翻译
msgid ""
msgstr ""
"Project-Id-Version: Touhou Blaster Master\n"
"Language: zh_CN\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Plural-Forms: nplurals=1; plural=0;\n"

# 主菜单
msgid "ui_main_menu_title"
msgstr "东方魔炮使"

msgid "ui_main_menu_start"
msgstr "开始游戏"

msgid "ui_main_menu_continue"
msgstr "继续"

msgid "ui_main_menu_options"
msgstr "选项"

msgid "ui_main_menu_quit"
msgstr "退出"

# HUD
msgid "ui_hud_time"
msgstr "时间：{time}s"

msgid "ui_hud_resources"
msgstr "资源：{count}"

msgid "ui_hud_beam_power"
msgstr "魔炮威力：{power}%"

# 进化树
msgid "ui_evolution_node_beam_width"
msgstr "魔炮宽度 +{level}"

msgid "ui_evolution_node_beam_damage"
msgstr "魔炮伤害 +{level}"

msgid "ui_evolution_node_auto_fire"
msgstr "自动发射"

# 敌人名称
msgid "enemy_fairy_basic_name"
msgstr "妖精"

msgid "enemy_fairy_elite_name"
msgstr "妖精队长"

# 剧情
msgid "story_chapter1_title"
msgstr "第一章：异变的预感"

msgid "story_chapter1_intro"
msgstr "幻想乡出现了异常的魔力波动..."
```

### messages_en.po 示例

```po
# messages_en.po - English Translation
msgid ""
msgstr ""
"Project-Id-Version: Touhou Blaster Master\n"
"Language: en_US\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Plural-Forms: nplurals=2; plural=(n != 1);\n"

# 主菜单
msgid "ui_main_menu_title"
msgstr "Touhou Blaster Master"

msgid "ui_main_menu_start"
msgstr "Start Game"

msgid "ui_main_menu_continue"
msgstr "Continue"

msgid "ui_main_menu_options"
msgstr "Options"

msgid "ui_main_menu_quit"
msgstr "Quit"

# HUD
msgid "ui_hud_time"
msgstr "Time: {time}s"

msgid "ui_hud_resources"
msgstr "Resources: {count}"

# 复数示例
msgid "enemy_defeated"
msgid_plural "enemies_defeated"
msgstr[0] "1 enemy defeated"
msgstr[1] "{count} enemies defeated"
```

---

## 附录 B: LocalizationManager 实现

```gdscript
# res://scripts/systems/localization_manager.gd
class_name LocalizationManager
extends Node

signal locale_changed(new_locale: String)

const CONFIG_PATH = "res://assets/locale/locale_config.tres"

var _current_locale: String = "zh_CN"
var _available_locales: Array[String] = ["en_US", "zh_CN", "ja_JP"]

func _ready():
    # 加载配置
    var config = load(CONFIG_PATH)
    _available_locales = config.SUPPORTED_LOCALES

    # 自动检测或加载保存的语言
    _load_saved_locale()

func set_locale(locale: String) -> bool:
    if locale not in _available_locales:
        push_warning("Locale not supported: %s" % locale)
        return false

    TranslationServer.set_locale(locale)
    _current_locale = locale
    _update_fonts(locale)

    # 发送信号，通知 UI 刷新
    emit_signal("locale_changed", locale)

    # 保存偏好
    _save_locale(locale)

    return true

func get_current_locale() -> String:
    return _current_locale

func get_available_locales() -> Array[String]:
    return _available_locales

func _update_fonts(locale: String):
    var config = load(CONFIG_PATH)
    var font_path = config.FONT_CONFIG[locale]
    var font = load(font_path) as Font

    # 设置全局默认字体
    var theme = Theme.new()
    theme.default_font = font
    # 应用到所有 UI
    get_tree().root.theme = theme

func _save_locale(locale: String):
    var config = ConfigFile.new()
    config.set_value("locale", "language", locale)
    config.save("user://settings.cfg")

func _load_saved_locale():
    var config = ConfigFile.new()
    var err = config.load("user://settings.cfg")
    if err == OK:
        var saved_locale = config.get_value("locale", "language", "zh_CN")
        set_locale(saved_locale)
```

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 0.1 | 2026-03-26 | 初始版本创建 |

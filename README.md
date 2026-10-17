# 简单放置破坏榜 (SimplePlacedMinedScoreboard)

服务端 Sidebar 的 **破坏总和榜 + 放置总和榜**，一个 [MCDReforged](https://github.com/Fallen-Breath/MCDReforged) 单文件插件。

- **插件命令**：`!!spms`（旧命令 `!!dsb` 已不再注册）
- **内部 id / 数据包命名空间**：`sidebar_boards`（保持不变，避免世界里的计分项与索引重来）
- **依赖**：MCDReforged `>= 2.1.0`
- **目标版本**：Minecraft 1.20.4（数据包模式内置 1.20.4 注册表；`stacked` 布局需要 1.20.3+）

---

## 功能特性

| | 说明 |
|---|---|
| 双榜同屏 | 侧边栏同时显示「破坏总和榜」和「放置总和榜」，各自上方一行「总破坏 / 总放置」合计 |
| 真·实时 | 数据包模式由服务端**每 tick** 统计、排序、刷新，不依赖 RCON，也不必等统计文件落盘 |
| 历史导入 | 首次启动只导入一次 `world/stats/*.json` 的历史累计值，不会从 0 开始 |
| bot 过滤 | 默认排除所有以 `bot_` 开头的玩家名 |
| 两种布局 | `stacked`（双榜同屏，默认） / `rotate`（两榜轮播） |
| 千分位 | 可配置分隔符，由数据包函数宏在原版侧拼出 `1,234,567` |
| 模组方块 | 提供 `datapack_registry_file` / `extra_blocks` / `extra_items` 配置项 |

---

## 安装

1. 确认服务端已安装 [MCDReforged 2.1.0+](https://mcdreforged.com/)。
2. 把 `simpleplacedminedscoreboard.py` 放进 MCDR 的 `plugins/` 目录：

   ```
   MCDReforged/
   └── plugins/
       └── simpleplacedminedscoreboard.py
   ```

3. 重载插件（或在控制台执行 `!!MCDR plugin load simpleplacedminedscoreboard`）。
4. 插件首次加载会在 `config/simpleplacedminedscoreboard/config.json` 生成默认配置，`auto_show` 为 `true` 时会自动开榜。

> **注意**：文件必须是 `.py` 单文件形式，不要改文件名里的内部 id。插件的 `PLUGIN_METADATA['id']` 是 `sidebar_boards`，改 id 会让 `config/`、数据包索引、计分项槽位全部重来，世界里的 `dsb_*` / `dsbm*` / `dsbu*` 计分项和历史分数会变成孤儿。

---

## 命令

游戏内或控制台输入 `!!spms`：

| 命令 | 说明 | 权限 |
|---|---|---|
| `!!spms` | 帮助 | 无 |
| `!!spms status` | 查看榜单与运行状态 | 无 |
| `!!spms debug` | 诊断路径 / 文件数 / RCON / 每个玩家的口径 | 无 |
| `!!spms dp` | 数据包模式诊断（模式、pack_format、生成的文件） | 无 |
| `!!spms dp rebuild` | 重建实时数据包（保留已累计分数） | 等级 2 |
| `!!spms show` | 显示侧边栏 | 等级 2 |
| `!!spms hide` | 隐藏侧边栏（不删分数） | 等级 2 |
| `!!spms refresh` | 立即重画 | 等级 2 |

---

## 数据口径

### 破坏总和榜

```
sum(minecraft:mined 的所有项)
```

与参考插件 StatsHelper 的 `#all` 求和逻辑一致。

### 放置总和榜

```
目标集合 = 该玩家自己的 used.keys() ∩ mined.keys()
初始值   = used 中这些目标项的计数总和
实时值   = 初始值 + 逐 tick 按 used 新增数量继续累计
```

关键设计：快照保存的是**整份 `used`**，不只是交集内的项。所以某个方块从「不在交集」变成「在交集」时，它的上次值仍是真实历史值，增量为 0，**不会补计历史数量**；只有它真的又被放置了才会累加。

如果玩家 JSON 里已有现成的「放置总和」字段，可用 `placed_precomputed_path` 指定（点号分隔的 JSON 路径），优先读它。

---

## 两种实时方式

### `realtime_mode = "datapack"`（默认）

插件在 `<世界目录>/datapacks/sidebar_boards/` 生成一个数据包，用原版计分板准则项（`minecraft.mined:*` / `minecraft.used:*`）逐 tick 统计，并在函数里完成排序、合计、渲染。

- **不需要 RCON**，也不需要周期性 `save-all`
- 内置 Minecraft 1.20.4 完整方块/物品注册表（压缩存放，运行时无外部依赖）
- 首次启动会 `save-all flush` 并等待 `Saved the game`，然后只导入一次历史值
- 其他 MC 版本请配置 `datapack_registry_file` 提供完整注册表

### `realtime_mode = "poll"`

插件自己轮询 `world/stats/*.json`。原版统计只在保存时落盘，所以要「实时」就得周期性 `save-all`（`save_all_interval_seconds`，默认 20 秒）。延迟是分钟级到秒级，适合不想动数据包的场景。

---

## 侧边栏是怎么画的

原版**一个 display slot 只能挂一个计分项**，`sidebar.team.<color>` 只是「按队伍颜色区分可见性」，并不是把多个榜单叠起来。所以「同时显示两个榜单」的做法是：

1. 建一个 `dummy` 计分项（`dsb_rank`）；
2. 用**假玩家名**占行，`scoreboard players set` 控制名次顺序（分数越高越靠上）；
3. 每一行显示的文本用 `scoreboard players display name` 写入 JSON 文本组件；
4. `scoreboard objectives modify <obj> numberformat blank` 把分数列藏掉（需要 MC 1.20.3+）。

在 1.20.4 中由数据包每 tick 汇总增量并用函数宏（`with storage`）更新双榜；`stats` JSON 仅首次导入历史值，不作为实时数据源。

布局自上而下：

```
放置/破坏榜          <- 计分项标题
总破坏  12,345,678
破坏总和榜
  #1 Alice  1,234
  #2 Bob    1,100
  ...
总放置  9,876,543
放置总和榜
  #1 Alice  987
  ...
```

侧边栏上限 15 行：合计行各 1 行 + 榜单标题各 1 行，所以每榜最多 `(15-4)//2 = 5` 行。

---

## 配置

首次运行自动生成 `config/simpleplacedminedscoreboard/config.json`：

### 服务端 / 世界路径

| 键 | 默认 | 说明 |
|---|---|---|
| `server_path` | `""` | 留空 = 用 MCDR 的 `working_directory` |
| `world_folder` | `""` | 留空 = 自动从 `server.properties` 的 `level-name` 读 |

### 逐 tick 刷新

| 键 | 默认 | 说明 |
|---|---|---|
| `tick_interval_ms` | `50` | 一次「tick」的间隔，50ms ≈ 原版 20 TPS |
| `scan_interval_ticks` | `1` | 每多少 tick 扫一次文件 mtime |
| `save_all_interval_seconds` | `20` | 主动 `save-all` 间隔；`0` = 不主动保存（延迟变分钟级） |

### 实时方式

| 键 | 默认 | 说明 |
|---|---|---|
| `realtime_mode` | `"datapack"` | `datapack`（服务端逐 tick） / `poll`（读统计文件） |
| `minecraft_version` | `"1.20.4"` | 无法从服务端取到版本时用它 |
| `datapack_registry_file` | `""` | 可选：包含 block/item 的完整注册表 JSON，支持模组方块 |
| `datapack_extra_blocks` | `[]` | 额外方块 ID，如 `["mymod:ruby_block"]` |
| `datapack_extra_items` | `[]` | 额外物品 ID |
| `datapack_dir` | `""` | 留空 = `<世界目录>/datapacks` |
| `datapack_pack_format` | `0` | `0` = 自动推断 |
| `datapack_exclude_items` | `[]` | 排除某些统计键（跨版本降级 / 移除模组后旧方块 ID 会让准则项建失败） |

### 榜单显示

| 键 | 默认 | 说明 |
|---|---|---|
| `sidebar_layout` | `"stacked"` | `stacked`（双榜同屏） / `rotate`（轮播） |
| `board_top_n` | `6` | 每个榜显示前几名（stacked 下 `2 + 2*N` 需 ≤ 15） |
| `rotate_interval_seconds` | `10` | 仅 `rotate` 布局的轮播间隔 |
| `hide_number_column` | `true` | `stacked` 布局隐藏分数列（需 1.20.3+） |
| `sidebar_title` | `"放置/破坏榜"` | 侧边栏最上方的计分项标题 |
| `break_title` | `"破坏总和榜"` | |
| `place_title` | `"放置总和榜"` | |
| `show_rank_number` | `true` | 行内是否带 `#1 #2` 名次 |
| `show_total_row` | `true` | 是否显示「总破坏 / 总放置」两行 |
| `total_break_label` | `"总破坏"` | |
| `total_place_label` | `"总放置"` | |
| `thousands_separator` | `""` | 千分位分隔符；`","` = 恢复逗号，也支持 `" "` / `"."` |
| `title_color` | `"gold"` | 标题颜色 |
| `name_color` | `"white"` | 玩家名颜色 |
| `value_color` | `"aqua"` | 数值颜色 |

### 玩家过滤

| 键 | 默认 | 说明 |
|---|---|---|
| `bot_name_prefixes` | `["bot_"]` | 排除这些前缀的玩家名 |
| `bot_name_patterns` | `[]` | 额外的正则黑名单（完整匹配，忽略大小写） |

> 这里**刻意不使用**参考插件 `isBot()` 里「名字长度 <4 或 >16 也算 bot」的启发式——那条规则会把 `Bob` / `Ann` 这种正常短名误判掉。

### 其他

| 键 | 默认 | 说明 |
|---|---|---|
| `auto_show` | `true` | 插件加载后自动开榜 |
| `refresh_uuid_interval_seconds` | `30` | uuid/玩家名映射的重扫间隔 |
| `placed_precomputed_path` | `""` | 现成的 placed 总和字段的 JSON 路径 |
| `debug_log` | `false` | 详细日志 |

---

## 常见问题

**侧边栏没出现？**
`!!spms debug` 看统计目录是否存在、识别到多少玩家；`!!spms dp` 看数据包是否加载。MCDR 是先加载插件再启动服务端，插件会等服务端就绪后自动补建（最多等 30 分钟）。

**用了模组方块，榜单数字不动？**
准则项建失败通常是因为注册表里没有那个方块。提供 `datapack_registry_file`（完整注册表 JSON），或把 ID 填进 `datapack_extra_blocks` / `datapack_extra_items`，然后 `!!spms dp rebuild`。

**数值和游戏内 `stat` 不一致？**
数据包模式下实时值由服务端计分板持有，用 `/scoreboard players get <玩家> dsb_brk` / `dsb_plc` 核对。`!!spms status` 里显示的是本地磁盘快照，仅作参考。

**想改回单榜轮播？**
`sidebar_layout` 改成 `"rotate"`，重载插件。

---

## 文件说明

```
simpleplacedminedscoreboard.py   插件本体（单文件，MCDR 直接加载）
README.md                        本文档
LICENSE                          MIT
```

插件会在世界目录下生成数据包：

```
<世界目录>/datapacks/sidebar_boards/
├── pack.mcmeta
└── data/
    ├── minecraft/tags/function/{load,tick}.json
    └── sidebar_boards/functions/
        ├── load.mcfunction / tick.mcfunction / update.mcfunction
        ├── new_player.mcfunction
        ├── event/m{slot}.mcfunction / event/u{slot}.mcfunction
        ├── seed/<slug>.mcfunction
        ├── fmt/*.mcfunction
        └── render/*.mcfunction
```

以及运行期索引 `config/simpleplacedminedscoreboard/datapack_index.json`（方块 ID → 槽位，保证重载后槽位稳定）。

---

## 许可

[MIT](LICENSE)

# TAHAI-Skills · 社区技能库

> 踏海（TAHAI）的插件生成技能库。每个 **skill** 是一包"领域提示词"，在生成流水线的
> **Planner（规划）** 与 **FileGen（逐文件生成）** 两个阶段被注入，让普通的生成模型在某个
> 垂直场景下少踩坑、写出能编译、符合 Paper 最佳实践的代码。
>
> 本仓库面向社区共建：你只需写 Markdown 与一个 `brief.json`，通过踏海的上传入口提交，
> 经维护者人工审批后入库即可被全站调用。

---

## 1. 它在踏海里扮演什么角色

踏海的生成流水线（简化）：

```
需求澄清 → 复杂度分级/路径确认 → Planner 出文件清单(含 generatorType) → 按文件分桶逐个 FileGen → Maven 编译
```

- **skill 不是代码模板**，而是"提示词增强包"。当用户需求命中某个 skill 时，它的内容被拼进
  Planner / FileGen 的 system prompt，相当于给生成模型临时挂载一段领域知识与硬约束。
- 因此一个 skill 的价值 = **把某类需求的"踩坑点 / 约定 / 最简实现范式"固化成提示词**，
  而不是把整段实现代码贴死（那样无法适配千变万化的需求）。

---

## 2. 仓库结构

`master` 分支下，**一个 skill = 一个顶层文件夹**：

```
TAHAI-Skills/
├─ README.md                  ← 本文件（规范）
├─ teleport-spawn/            ← skill A
│  ├─ brief.json
│  └─ functions/
│     ├─ SpawnCommand.md
│     └─ LocationUtil.md
├─ economy-vault/             ← skill B
│  ├─ brief.json
│  └─ functions/
│     ├─ EconomyManager.md
│     └─ BalanceCommand.md
└─ nms-version-bridge/        ← skill C（RAG 类，见 §7，规范待定）
   ├─ brief.json
   └─ knowledge/
```

约定：

- 文件夹名 = skill 唯一 id，用 **kebab-case**（小写连字符），全库唯一。
- `brief.json` 必须位于 skill 根目录。
- 提示词文件统一放 `functions/`（RAG 类知识放 `knowledge/`，见 §7）。
- **目录最多一层,不要文件夹再套文件夹。** 文件多时可在 skill 根下按用途分几个一级文件夹
  （如 `functions/`、`config/`），但每个文件夹内直接放文件,不再嵌套子目录——保持扁平、好审阅。

---

## 3. `brief.json` 规范

记录这个 skill 的身份与"结构"——结构即它提供了哪些 func 提示词、各自属于哪种 FileGen 类型、
彼此的依赖顺序。Planner 读到它就知道命中该 skill 时该规划出哪些文件。

```jsonc
{
  "id": "teleport-spawn",
  "name": "传送到出生点",
  "author": "your-github-id",
  "version": "1.0.0",
  "description": "提供 /spawn 命令把玩家传送到世界出生点，含坐标工具类。",
  "tags": ["command", "teleport"],          // 用于需求命中检索
  "mcVersions": ["1.20", "1.21"],           // 适用 MC 版本（可空=通用）
  "coreTypes": ["PAPER", "SPIGOT"],         // 适用核心（可空=通用）

  // ── 结构：本 skill 提供的提示词文件清单 ──
  "structure": [
    {
      "file": "functions/LocationUtil.md",
      "fileGen": "UtilGen",                 // 见 §5 类型表
      "role": "封装读取世界出生点坐标的静态方法",
      "depends": []
    },
    {
      "file": "functions/SpawnCommand.md",
      "fileGen": "CommandGen",
      "role": "/spawn 命令：将发起者传送到主世界出生点",
      "depends": ["LocationUtil.md"]        // 依赖同 skill 内其它文件，填文件名
    }
  ]
}
```

字段说明：

| 字段 | 必填 | 说明 |
|---|---|---|
| `id` / `name` / `author` / `description` | ✓ | 身份四件套；`id` 与文件夹名一致 |
| `version` | ✓ | 语义化版本，便于审批/回滚 |
| `tags` | ✓ | 命中检索关键词；尽量贴近用户口语 |
| `mcVersions` / `coreTypes` | — | 留空表示通用 |
| `structure[]` | ✓ | 每项对应一个 func.md：`file` / `fileGen` / `role` / `depends` |

`structure` 的字段刻意与踏海 Planner 内部的文件项（`path / role / generatorType / depends`）一一对齐，
入库后可被直接消费，无需二次转换。

---

## 4. `func.md` 规范

每个提示词文件 = **YAML frontmatter 元信息头 + 正文实现细则**。

### 4.1 元信息头（frontmatter）

开头声明两件事：**用哪种 FileGen** + **需要的 Maven 依赖（即"4 行信息"：标准 `<dependency>` 四元素）**。

```markdown
---
fileGen: CommandGen            # 必填：该文件交给哪种生成器，见 §5
deps:                          # 可选：本文件需要的额外 Maven 依赖
  - groupId: net.milkbowl.vault
    artifactId: VaultAPI
    version: "1.7"
    scope: provided
mcVersions: ["1.21"]           # 可选：覆盖 brief 的适用版本
---
```

- `deps` 的每一项就是一段标准 Maven 依赖坐标——**groupId / artifactId / version / scope 四行**。
  踏海生成 `pom.xml`（`FileRelatedGen`）时会把这些依赖收集进去；不需要额外依赖就省略 `deps`。
- `paper-api` / `spigot-api` 这类核心依赖**不用写**，踏海会按 `coreType` + `mcVersion` 自动注入。
  这里只声明你额外引入的第三方（Vault / PlaceholderAPI / WorldGuard 等）。

### 4.2 正文（实现细则）

正文是写给生成模型看的、关于"这个文件该怎么实现"的要点。**用要点而非整段代码**，
聚焦该 FileGen 类型容易出错的地方。例如 `SpawnCommand.md`：

```markdown
## 实现要点
- onCommand 中：sender 必须 instanceof Player，否则提示"仅玩家可用"并 return true。
- 目标坐标通过 LocationUtil.getSpawn(world) 获取（依赖本 skill 的 LocationUtil）。
- 传送用 player.teleport(loc)，成功后 sendMessage 反馈。
- 权限节点 tahai.spawn，未通过 hasPermission 则拒绝。

## 注意
- 不要在本文件注册命令；注册由 Main.onEnable 统一完成。
- 通过 Bukkit.getPluginManager().getPlugin("<projectName>") 取插件实例，禁止强转主类。
```

---

## 5. FileGen 类型速查

`fileGen` 字段必须是下列之一。每种类型踏海都有对应"专项规则"，选对类型能让生成模型自动继承这些约束：

| 类型 | 用途 | 关键约束 |
|---|---|---|
| `CommandGen` | 命令执行器 | 同类 `implements CommandExecutor, TabCompleter`；**不自注册**（Main 统一注册） |
| `ListenerGen` | 事件监听器 | `implements Listener` + 方法加 `@EventHandler`；不自注册。`tag=gui` 时为 GUI 配对（持有类 + 点击监听类） |
| `TaskGen` | 调度任务 | `extends BukkitRunnable`，只写 `run()`；调度由 Main.onEnable 启动 |
| `ManagerGen` | 服务/数据单例 | Main 持有引用；**禁 `static getInstance()`**；构造时载入 YAML，提供 `save()/shutdown()` |
| `ConfigGen` | YAML 资源文件 | `plugin.yml`/`config.yml` 等纯 YAML，**不含 Java** |
| `ConfigClassGen` | 配置包装类 | 包装 `YamlConfiguration`，提供 `reload()/save()/get*()`；默认值放 `config.yml` |
| `ModelGen` | 纯数据 POJO | **禁 import `org.bukkit.*`**；玩家引用用 `UUID` 而非 `Player` |
| `EnumGen` | Java enum | 禁复杂业务逻辑、禁依赖 Bukkit/Paper API |
| `UtilGen` | 静态工具类 | 全 `public static`，无实例字段，不持插件引用 |
| `FileRelatedGen` | 非 Java 项目文件 | `pom.xml` / `.gitignore` / `README.md` 等 |
| `MainGen` | 主类编排 | 按蓝图注册命令/监听/任务/服务，写 `onEnable`/`onDisable` |

> 一个 skill 通常只涉及其中几种。`MainGen` / `FileRelatedGen` 由踏海主流程统筹生成，
> **社区 skill 一般无需提供**，除非你的场景对主类编排/pom 有特殊要求。

---

## 6. 编写铁律（务必遵守）

下游 FileGen 是**低端模型**，skill 提示词的首要职责是把它**导向能满足需求的最简实现**。
违反以下任意一条，审批不予通过：

1. **持久化只用 YAML 配置 / PDC（PersistentDataContainer）。**
   **禁止** SQL / JDBC / 数据库（SQLite/MySQL）/ ORM / D1 等一切数据库与 Web 词汇。
2. **全程 Paper/Bukkit 词汇**，不引入与 MC 插件无关的概念。
3. **外部硬集成**仅限 Vault / PlaceholderAPI / WorldGuard 这类 Paper 生态插件，且**缺失要降级**
   （`plugin.yml` 用 softdepend，运行时判空），不得硬依赖导致加载失败。
4. **不强转插件主类**：取实例一律 `Bukkit.getPluginManager().getPlugin("<projectName>")`，
   返回 `Plugin` 接口；禁止 `(MainClass) getPlugin(...)` / `MainClass.getInstance()`。
5. **不做过度抽象**：不规划与需求无关的基类/接口/工厂；能一个文件解决就别拆。
6. 提示词写**要点**，不贴大段成品代码；把"容易写错的点"讲清楚比给范例更有用。

---

## 7. RAG 类 skill（实验性 · 规范待定）

像 **NMS（net.minecraft.server 跨版本反射/映射）** 这类知识，特点是**体量大、碎片化、版本敏感、
且只有少数文件会用到**——不适合像普通 md 那样整包塞进 prompt。这类适合做成 **RAG（检索增强）** skill。

**两者差异：**

| | 常规 md skill | RAG skill |
|---|---|---|
| 知识形态 | 一小撮约定/范式，写成 Markdown | 海量条目（如各版本类名、方法签名、混淆映射） |
| 注入方式 | **全量**拼进 prompt | **按需检索** top-k 相关片段再注入 |
| 适用 | 模式、最佳实践、API 用法 | 大型参考表、跨版本映射、签名库 |
| 贡献者要写 | 只写 md | 提供结构化知识条目 + 检索键 |
| 基础设施 | 无需额外设施 | 需要 embedding 向量化 + 向量库 + 检索 |

**为什么 NMS 适合 RAG：** 它的知识是"一张大表"，单次需求只命中其中极少数条目；全量注入既超 prompt
预算又稀释重点。RAG 把知识切块 → 向量化存库，运行时拿"当前 func 的上下文"做语义检索，只取最相关的
几条注入，既省 token 又更准。

**渐进式 RAG（推荐的一期做法）：** 与其等向量库，不如把大知识库**按版本/场景人工切成多个小而专的
普通 md skill**——例如 `nms-srgmap-1.21-common` 只收 1.21 最常用的 SRG 映射，`nms-srgmap-1.20-common`
另立一包。每包小到可全量注入，靠 `brief.json` 的 `mcVersions` / `tags` 在命中对应版本/场景时按需挂载。
这等于把"运行时语义检索"换成 **"上传时人工分片 + 命中时按标签选取"**——零基础设施、可控、当下就能用，
对 NMS 这种**天然按版本可切**的知识尤其合适，很可能长期就够。真正的向量 RAG 只在"单包仍然过大、
且需要跨条目语义召回"时才必要。

**现状与建议：** 踏海目前**尚无** embedding / 向量库基础设施，重型 RAG 属于二期。在那之前，
NMS / 大型映射类一律走上面的**人工分片小 skill**。`knowledge/` 目录暂为重型 RAG 占位，
**其格式待踏海向量能力落地后再正式定义**；当下不需要它——分片小 skill 用常规 `functions/` 即可。

> 如果你正打算贡献 NMS / 协议 / 大型映射类 skill，建议先按"版本 + 场景"切片成小 skill；
> 体量实在压不下来再开 issue 讨论向量 RAG 规范。

---

## 8. 贡献流程

> 上传入口正在设计中；下方先约定**审批与入库流程**，入口上线后无缝衔接。

1. 在踏海站内（或本仓库）按上述规范组织好你的 skill 文件夹。
2. 通过上传入口提交 → 自动进入 **merge 队列**（不直接落 `master`）。
3. **维护者人工审批**：核对命名/`brief.json` schema/铁律（§6）/是否与现有 skill 重复。
4. 通过后合并入 `master`，全站生效；不通过会附驳回理由，可改后重提。

提交前自检清单：

- [ ] 文件夹名 = `brief.json.id`，kebab-case，全库唯一。
- [ ] `brief.json` 字段完整，`structure[]` 与 `functions/` 下文件一一对应。
- [ ] 每个 `func.md` 有 `fileGen` 头；需要第三方依赖的写了 `deps`（4 行坐标）。
- [ ] 通读 §6 铁律，确认无数据库/SQL、无主类强转、外部依赖可降级。
- [ ] 提示词聚焦要点，无大段成品代码。

---

## 附：最小可用示例

`teleport-spawn/brief.json` + `teleport-spawn/functions/SpawnCommand.md` 即 §3、§4 中的内容，
两文件加起来即一个合规 skill。复制它改成你的场景，是最快的上手方式。

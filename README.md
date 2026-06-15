# TAHAI-Skills · 社区技能库

> 踏海（TAHAI）的**能力扩展**技能库。每个 **skill** 是一包领域提示词 + 可照抄模板，
> 用来给生成模型**注入它本来不会、或写不好的某项能力**（对接 QQ bot、用某个冷门库、
> NMS 反射……），在生成流水线的 **Planner（规划）** 与 **FileGen（逐文件生成）** 阶段被注入。
>
> 本仓库面向社区共建：你写 Markdown + 一个 `brief.json`，通过踏海上传入口提交，
> 经维护者人工审批后入库，即可供全站用户在生成时**自行挂载**。

---

## 1. 它在踏海里扮演什么角色

踏海的生成流水线（简化）：

```
需求澄清 → 复杂度分级/路径确认 → Planner 出文件清单 → 按文件分桶逐个 FileGen → Maven 编译
```

- **skill 由用户主动挂载，不是自动命中。** `master/` 下所有 skill 会平行展示在选择面板里，
  用户像**安排手牌**一样勾选要用的 skill 并排定顺序。被选中的 skill 内容按该顺序注入
  Planner / FileGen 的 prompt（**靠前者优先**，冲突时先生效）。
- skill 的本质是**能力扩展**：把"模型默认不熟悉的库 / 协议 / API 用法"连同可照抄的骨架，
  固化成提示词挂载上去。它**不是**"某个具体插件的实现分享"——那种模型本来就会做的事
  （如 `/spawn` 传送）不值得做成 skill。

---

## 2. 什么是优秀的 skill

判断一个 skill 值不值得做、好不好，看这 5 条（也是审批的核心标准）：

1. **能力增量，而非常识复述。**
   它应让模型获得**本来做不好的能力**——冷门库、外部协议、NMS、特定平台 API。
   模型本就会写的（基础命令、传送、简单监听）做成 skill 没有价值。
2. **强适用 · 业务中立。**
   覆盖该能力的**通用骨架**，能被该领域**大量不同需求**复用；**绝不写死具体业务玩法**。
   👉 代码量**可以很大**，但必须业务中立：`qqbot` skill 给"怎么连 ws、怎么收发群消息"=好；
   写死"有人发『签到』就发 100 金币"=坏（那是当次需求该生成的，绑死了适用性）。
3. **可直接照抄。**
   给**确定的库坐标**（`groupId:artifactId:version`）和**可粘贴的骨架代码**，
   让低端模型"照着抄就对"。只给抽象描述、不给样板，等于没帮上忙。
4. **明确边界与提问点。**
   部署相关（连接地址 / 端口 / token）、用户偏好（要不要开某配置）一律写明
   **"必须暴露到 config 并提示用户填写 / 必须显式提问"**，**不替用户拍板、不硬编码**。
5. **自带硬约束。**
   沿用 §7 铁律（持久化只 YAML/PDC、不强转主类、外部依赖降级），
   让能力扩展不破坏 Paper 最佳实践。

> 一句话：**好 skill = 给模型一项它本来不会、但业务无关、可照抄落地的能力。**

---

## 3. 仓库结构

`master` 分支下，**一个 skill = 一个顶层文件夹**。文件夹内按用途分**一级**子目录，分两类：

- **生成型目录 `functions/`** — 每个 md → Planner 规划出的**一个待生成文件**，必须标 `fileGen`（见 §6）。
- **能力/资料型目录**（`api/`、`protocol/`、`itemList/`、`outlook/` …自取名）— 注入给模型当
  **参考上下文**（库用法、协议说明、报文模板、外观清单等），**不一一对应生成文件**，不强制 `fileGen`。

```
TAHAI-Skills/
├─ README.md
├─ qqbot-onebot/                 ← 能力型 skill（推荐范式）
│  ├─ brief.json
│  ├─ protocol/
│  │  └─ websocket.md            ← ref：用哪个 ws 库、怎么建连/心跳/重连
│  ├─ api/
│  │  ├─ send-message.md         ← ref：发群/私聊消息的 OneBot 报文模板
│  │  └─ receive-message.md      ← ref：接收消息事件的解析模板
│  └─ functions/
│     ├─ BotConfig.md            ← gen：ws 地址/端口/token 配置类
│     └─ BotConnection.md        ← gen：维护 ws 长连接的服务单例
└─ chest-ui-beautify/            ← 美化箱子 UI（多资料目录示例）
   ├─ brief.json
   ├─ itemList/ …                ← ref：可用装饰物品清单
   ├─ outlook/ …                 ← ref：版式/配色约定
   └─ functions/ …               ← gen：GUI 持有类 + 点击监听
```

约定：

- 文件夹名 = skill 唯一 id，**kebab-case**，全库唯一。
- `brief.json` 必须位于 skill 根目录。
- **目录最多一层，不要文件夹再套文件夹。** 文件多时分几个一级目录即可，每个目录内直接放文件。
- 所有 md（gen + ref）都要在 `brief.json` 的 `structure[]` 里登记，否则审批/注入都看不到它。

---

## 4. `brief.json` 规范

记录身份 + 能力声明 + 结构（提供了哪些文件、生成型还是资料型、依赖顺序）。

```jsonc
{
  "id": "qqbot-onebot",
  "name": "QQ 机器人交互 (OneBot)",
  "author": "your-github-id",
  "version": "1.0.0",
  "description": "让生成的插件通过 OneBot/WebSocket 对接 QQ 机器人：建连 + 收发群消息。",
  "capability": "注入对接 QQ bot 的能力：WebSocket 长连接维护 + OneBot v11 收发报文模板。模型默认不熟这套协议。",
  "tags": ["qqbot", "onebot", "websocket", "群消息"],  // 选择面板里的检索/归类标签（非自动命中）
  "mcVersions": [],                                     // 与 MC 版本无关时留空
  "coreTypes": ["PAPER", "SPIGOT"],

  // ── 结构：本 skill 的所有文件 ──
  "structure": [
    { "kind": "ref", "file": "protocol/websocket.md",    "role": "用哪个 ws 库 + 建连/心跳/重连骨架" },
    { "kind": "ref", "file": "api/send-message.md",      "role": "发送群/私聊消息的 OneBot 报文模板" },
    { "kind": "ref", "file": "api/receive-message.md",   "role": "接收消息事件的解析模板" },

    { "kind": "gen", "file": "functions/BotConfig.md", "fileGen": "ConfigClassGen",
      "role": "读取 ws 地址/端口/access_token 配置", "depends": [] },

    { "kind": "gen", "file": "functions/BotConnection.md", "fileGen": "ManagerGen",
      "role": "维护到 OneBot 的 ws 长连接，暴露发消息/注册回调",
      "depends": ["functions/BotConfig.md", "protocol/websocket.md", "api/send-message.md", "api/receive-message.md"] }
  ]
}
```

字段说明：

| 字段 | 必填 | 说明 |
|---|---|---|
| `id` / `name` / `author` / `description` | ✓ | 身份四件套；`id` 与文件夹名一致 |
| `capability` | ✓ | 一句话讲清"这个 skill 给模型加了什么它原本不具备的能力"——审批和用户选择都看它 |
| `version` | ✓ | 语义化版本，便于审批/回滚 |
| `tags` | ✓ | 选择面板里的检索/归类标签（**不是**自动命中关键词） |
| `mcVersions` / `coreTypes` | — | 留空表示通用 |
| `structure[]` | ✓ | 每项一个文件：`kind`（`gen`/`ref`）+ `file` + `role`；`gen` 还需 `fileGen` 与 `depends` |

- `kind:"gen"` 的项**必须**有 `fileGen`，会被 Planner 当作一个待生成文件；其字段刻意与踏海内部
  文件项（`path / role / generatorType / depends`）对齐，入库即可直接消费。
- `kind:"ref"` 的项是参考资料，**不**生成文件；通过被 `gen` 项的 `depends` 引用来"喂给"对应生成步骤。
- `depends` 填 `structure` 里其它项的 `file` 路径（可含 `ref`），表示"生成此文件时把这些资料一并注入"。

---

## 5. 单个 md 文件规范

每个文件 = **YAML frontmatter + 正文**。

### 5.1 frontmatter

```markdown
---
kind: gen                      # gen=待生成文件 / ref=参考资料（与 brief 一致）
fileGen: ManagerGen            # gen 必填：交给哪种生成器，见 §6；ref 省略
deps:                          # 可选：本文件引入的额外 Maven 依赖（标准 <dependency> 四元素）
  - groupId: org.java-websocket
    artifactId: Java-WebSocket
    version: "1.5.7"
    scope: compile             # 纯 Java 第三方库 → compile + 需 shade 进 jar（见铁律 3b）
---
```

- `deps` 每项就是一段标准 Maven 依赖坐标——**groupId / artifactId / version / scope 四行**，
  踏海生成 `pom.xml` 时统一收集。
- `paper-api` / `spigot-api` **不用写**，踏海按 `coreType`+`mcVersion` 自动注入。这里只声明额外引入的。
- `deps` 可出现在任意 md（资料型 `websocket.md` 声明 ws 库最合适）。

### 5.2 正文

**能力型 skill 鼓励给大量可照抄代码**（这与早期"只写要点"的思路相反——能力扩展靠样板）。
但务必业务中立。例如 `protocol/websocket.md`：

````markdown
## 用哪个库
- WebSocket 客户端用 `org.java-websocket:Java-WebSocket`（轻量、无传递依赖，适合 shade）。
- 它是 compile 依赖，pom 必须配 maven-shade-plugin 把它打进最终 jar。

## 建立连接（可直接照抄骨架）
```java
client = new WebSocketClient(URI.create(wsUrl /* 来自配置，勿硬编码 */)) {
    @Override public void onOpen(ServerHandshake h) { /* 标记已连接 */ }
    @Override public void onMessage(String msg)     { /* 交给 OneBot 事件解析 */ }
    @Override public void onClose(int c, String r, boolean remote) { /* 触发重连 */ }
    @Override public void onError(Exception ex)      { /* 记录日志 */ }
};
client.connect();
```

## 必须显式提问 / 暴露配置（勿写死）
- ws 地址、端口、access_token：部署相关，必须放进 config.yml 由用户填写。
- 心跳/重连间隔可给默认值（30s/5s），但也建议入配。

## 业务中立
- 本 skill 只保证"连得上、收得到、发得出"；收到消息后做什么由用户当次需求决定，不在此写死。
````

---

## 6. FileGen 类型速查（`kind:"gen"` 用）

| 类型 | 用途 | 关键约束 |
|---|---|---|
| `CommandGen` | 命令执行器 | 同类 `implements CommandExecutor, TabCompleter`；**不自注册**（Main 统一注册） |
| `ListenerGen` | 事件监听器 | `implements Listener` + `@EventHandler`；不自注册。`tag=gui` 时为 GUI 配对 |
| `TaskGen` | 调度任务 | `extends BukkitRunnable`，只写 `run()`；调度由 Main.onEnable 启动 |
| `ManagerGen` | 服务/数据单例 | Main 持有引用；**禁 `static getInstance()`**；构造载入 YAML，提供 `save()/shutdown()` |
| `ConfigGen` | YAML 资源文件 | `plugin.yml`/`config.yml` 等纯 YAML，**不含 Java** |
| `ConfigClassGen` | 配置包装类 | 包装 `YamlConfiguration`，`reload()/save()/get*()`；默认值放 `config.yml` |
| `ModelGen` | 纯数据 POJO | **禁 import `org.bukkit.*`**；玩家引用用 `UUID` |
| `EnumGen` | Java enum | 禁复杂业务逻辑、禁依赖 Bukkit/Paper API |
| `UtilGen` | 静态工具类 | 全 `public static`，无实例字段，不持插件引用 |
| `FileRelatedGen` | 非 Java 项目文件 | `pom.xml` / `.gitignore` / `README.md` 等 |
| `MainGen` | 主类编排 | 按蓝图注册命令/监听/任务/服务，写 `onEnable`/`onDisable` |

> `MainGen` / `FileRelatedGen` 由踏海主流程统筹，社区 skill 一般无需提供。

---

## 7. 编写铁律（务必遵守，违反不予过审）

1. **持久化只用 YAML 配置 / PDC。** 禁止 SQL / JDBC / 数据库 / ORM / D1 等数据库与 Web 词汇。
2. **全程 Paper/Bukkit 词汇**，不引入与 MC 插件无关的概念。
3. **依赖分两类，分别处理：**
   - **3a. Bukkit 插件依赖** — 仅限 Vault / PlaceholderAPI / WorldGuard 这类，且**缺失要降级**
     （`plugin.yml` 用 softdepend，运行时判空），不得硬依赖导致加载失败。
   - **3b. 纯 Java 第三方库**（如 `Java-WebSocket`、JSON 库）— **允许引入**，但 `deps` 给**确定坐标**、
     `scope: compile` + 配 **maven-shade** 打进 jar；优先选**轻量、无传递依赖**的库。
4. **不强转插件主类**：取实例一律 `Bukkit.getPluginManager().getPlugin("<projectName>")`（`Plugin` 接口），
   禁止 `(MainClass) getPlugin(...)` / `MainClass.getInstance()`。
5. **业务中立**：只给能力骨架，**不写死具体业务玩法/参数**——那是用户当次需求该生成的。
6. **该问就问、勿硬编码**：连接地址/端口/token、用户偏好等一律暴露到 config 并提示用户。
7. **不做无谓抽象**：不引入与能力无关的基类/接口/工厂。

---

## 8. RAG 类 skill（实验性 · 规范待定）

像 **NMS（跨版本反射/映射）** 这类知识**体量大、碎片化、版本敏感、只有少数文件用到**，
不适合整包塞进 prompt。这类适合 **RAG（检索增强）**。

| | 常规 md skill | RAG skill |
|---|---|---|
| 知识形态 | 一撮约定/模板，写成 md | 海量条目（版本类名、方法签名、混淆映射） |
| 注入方式 | **全量**拼进 prompt | **按需检索** top-k 片段再注入 |
| 贡献者要写 | 写 md | 提供结构化知识条目 + 检索键 |
| 基础设施 | 无需额外设施 | 需要 embedding + 向量库 + 检索 |

**渐进式 RAG（推荐的一期做法）：** 把大知识库**按版本/场景人工切成多个小而专的普通 md skill**——
例如 `nms-srgmap-1.21-common` 只收 1.21 最常用映射，`nms-srgmap-1.20-common` 另立一包。
每包小到可全量注入，靠用户在面板里挑对应版本的那包挂载。这等于把"运行时语义检索"换成
**"上传时人工分片 + 挂载时按需选取"**——零基础设施、可控、当下就能用，对 NMS 这种天然按版本可切的
知识尤其合适，很可能长期就够。真正的向量 RAG 只在"单包仍过大、且需跨条目语义召回"时才必要。

**现状：** 踏海暂无 embedding/向量库，重型 RAG 属二期；在那之前 NMS/大型映射类一律走**人工分片小 skill**。
`knowledge/` 目录暂为重型 RAG 占位，格式待向量能力落地再定。

---

## 9. 贡献流程

> 上传入口设计中；下方先约定审批与入库流程，入口上线后无缝衔接。

1. 按规范组织好 skill 文件夹。
2. 经上传入口提交 → 自动进 **merge 队列**（不直接落 `master`）。
3. **维护者人工审批**：核对命名 / `brief.json` schema / §2 标准 / §7 铁律 / 是否与现有 skill 重复。
4. 通过即合并入 `master`，全站可挂载；不通过附驳回理由，可改后重提。

提交前自检：

- [ ] 文件夹名 = `brief.json.id`，kebab-case，全库唯一。
- [ ] `capability` 一句话说清能力增量；**不是**模型本就会的常识。
- [ ] `structure[]` 登记了所有 md；`gen` 项都有 `fileGen`，`depends` 指向有效 `file`。
- [ ] 代码量大没关系，但**业务中立**、可照抄；第三方库给了确定坐标。
- [ ] 部署/偏好项暴露到 config 或显式提问，无硬编码。
- [ ] 通读 §7：无数据库/SQL、无主类强转、Bukkit 插件依赖可降级、第三方库已 shade。

---

## 附：最快上手

复制 `qqbot-onebot/`（§3 结构 + §4 `brief.json` + §5.2 `websocket.md`）整套，
按你的能力场景替换库坐标与模板即可——它就是一个合规的能力型 skill 范本。

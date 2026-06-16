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

`master` 分支下，**一个 skill = 一个顶层文件夹**（文件夹名 = skill id）。文件夹内按用途分
**自取名的一级语义子目录**（`api/`、`handler/`、`protocol/`…，名字随能力自定），子目录里直接放 md。

> **关键：`gen` / `ref` 是每个「文件」的属性（写在 frontmatter 的 `kind`），不是目录的属性。**
> 同一个子目录里 gen 和 ref 可以共存——例如下面 `api/` 里既有三个 ref「报文片段」，
> 又有一个把它们聚合落盘的 gen 工具类。**不要**按 gen/ref 去切目录，按业务语义切。

下面是随本 README 附带的范本 skill `qqbot-onebot/`（对接 QQ bot）的真实结构：

```
qqbot-onebot/
├─ brief.json
├─ protocol/
│  └─ websocket.md            ← gen(ManagerGen)：OneBotClient 正向长连接 + 自动重连
├─ api/
│  ├─ sendGroupMsg.md         ← ref：send_group_msg 报文片段（sendG 方法体）
│  ├─ sendPrivateMsg.md       ← ref：send_private_msg 报文片段（sendP 方法体）
│  ├─ sendGroupAt.md          ← ref：at 消息段片段（sendGroupAt 方法体）
│  └─ OneBotApi.md            ← gen(UtilGen)：聚合上面三个 ref，落盘成 OneBotApi 工具类
└─ handler/
   ├─ privateMsgHandler.md    ← gen(UtilGen)：私聊指令路由，回复走 OneBotApi.sendP
   ├─ groupMsgHandler.md      ← gen(UtilGen)：群聊指令路由，回复走 OneBotApi.sendG/sendGroupAt
   └─ JsonParser.md           ← gen(UtilGen)：上行 JSON 解析 + 按 message_type 派发到两个 Handler
```

约定：

- 文件夹名 = skill 唯一 id，**kebab-case**，与 `brief.json.id` 一致，全库唯一。
- `brief.json` 必须位于 skill 根目录。
- **目录最多一层，不要文件夹再套文件夹。** 文件多时多分几个一级语义目录，目录内直接放 md。
- 所有 md（gen + ref）都要在 `brief.json` 的 `structure[]` 里登记，否则审批/注入都看不到它。

---

## 4. `brief.json` 规范

记录身份 + 能力声明 + **对宿主项目的全局符号假设** + 结构（哪些文件、gen/ref、依赖顺序）。
下面就是范本 skill 的真实 `brief.json`：

```jsonc
{
  "id": "qqbot-onebot",
  "name": "QQ 机器人交互 (OneBot)",
  "author": "Sudark (superwfox)",
  "version": "1.0.0",
  "description": "让生成的插件通过 OneBot/WebSocket 对接 QQ 机器人：建连 + 收发群/私聊消息 + 消息解析分发。",
  "capability": "注入对接 QQ bot 的能力：WebSocket 长连接维护与自动重连 + OneBot v11 收发报文模板 + 上行消息 JSON 解析分发与回复入口。模型默认不熟这套协议。",
  "tags": ["qqbot", "onebot", "websocket", "群消息", "私聊", "消息解析"],  // 检索/归类标签（非自动命中）
  "mcVersions": [],                                     // 与 MC 版本无关时留空
  "coreTypes": ["PAPER", "SPIGOT"],

  // ── 假定宿主项目已存在的全局符号（由踏海内置的哪种 Gen 提供）──
  // 踏海据此确保这些前置类被一并规划；缺了它们本 skill 接不上线。
  "expectedGlobals": {
    "Main.get()": "MainGen 提供的主类单例（getLogger / Bukkit 任务调度）",
    "ConfigManager.GroupId": "ConfigClassGen 提供的目标 QQ 群号（String）",
    "ConfigManager.WsUrl": "ConfigClassGen 提供的 OneBot ws 地址（String）"
  },

  // ── 结构：本 skill 的所有文件（gen/ref 由 file 自身 frontmatter 的 kind 决定，此处复述一致即可）──
  "structure": [
    { "kind": "ref", "file": "api/sendGroupMsg.md",   "role": "发送群消息的 OneBot 报文模板（send_group_msg）" },
    { "kind": "ref", "file": "api/sendPrivateMsg.md", "role": "发送私聊消息的报文模板（send_private_msg）" },
    { "kind": "ref", "file": "api/sendGroupAt.md",    "role": "群内 @ 某人的消息段（at）报文模板" },

    { "kind": "gen", "file": "api/OneBotApi.md", "fileGen": "UtilGen",
      "role": "对外发消息工具类：聚合 sendG/sendP/sendGroupAt，持 static client 字段",
      "depends": ["api/sendGroupMsg.md", "api/sendPrivateMsg.md", "api/sendGroupAt.md"] },

    { "kind": "gen", "file": "handler/privateMsgHandler.md", "fileGen": "UtilGen",
      "role": "私聊指令路由 PrivateMsgHandler.handle(userQQ,msg)；回复走 OneBotApi.sendP",
      "depends": ["api/OneBotApi.md"] },

    { "kind": "gen", "file": "handler/groupMsgHandler.md", "fileGen": "UtilGen",
      "role": "群消息指令路由 GroupMsgHandler.handle(userQQ,msg)；回复走 OneBotApi.sendG/sendGroupAt",
      "depends": ["api/OneBotApi.md"] },

    { "kind": "gen", "file": "handler/JsonParser.md", "fileGen": "UtilGen",
      "role": "上行消息解析分发 OneBotHandler.MsgDivider：拆段 + 按 message_type 派发",
      "depends": ["handler/privateMsgHandler.md", "handler/groupMsgHandler.md"] },

    { "kind": "gen", "file": "protocol/websocket.md", "fileGen": "ManagerGen",
      "role": "维护 ws 正向长连接 OneBotClient，自动重连，onMessage 交 OneBotHandler 解析",
      "depends": ["handler/JsonParser.md", "api/OneBotApi.md"] }
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
| `expectedGlobals` | 视情况 | 声明本 skill **假定宿主项目已存在**的全局符号（`Main.get()`、`ConfigManager.X`…）及由哪种 Gen 提供。踏海据此把前置类一并纳入规划 |
| `structure[]` | ✓ | 每项一个文件：`kind`（`gen`/`ref`）+ `file` + `role`；`gen` 还需 `fileGen` 与 `depends` |

- `kind:"gen"` 的项**必须**有 `fileGen`，被 Planner 当作一个待生成文件；其字段刻意与踏海内部
  文件项（`path / role / generatorType / depends`）对齐，入库即可直接消费。
- `kind:"ref"` 的项**不**单独生成文件；它是**可复用的代码片段/模板**，通过被某个 `gen` 项 `depends`
  引用，在生成那一步被「拼进去」。范本里 `api/OneBotApi.md`（UtilGen）就 `depends` 三个 ref 方法片段，
  把它们聚合成一个完整 `OneBotApi` 类——这是「ref = 方法片段，gen = 聚合落盘」的典型用法。
- `depends` 填 `structure` 里其它项的 `file` 路径（可含 `ref`），既表达**注入关系**，也给出**生成拓扑顺序**。

---

## 5. 单个 md 文件规范

每个文件 = **YAML frontmatter + 正文**。范本里每个 md 都是这套写法，照抄即可。

### 5.1 frontmatter

可用键（按是否出现取舍，不是每个都必填）：

| 键 | 用于 | 说明 |
|---|---|---|
| `kind` | 全部 | `gen` / `ref`，与 `brief.json` 一致 |
| `fileGen` | gen | 交给哪种生成器，见 §6；ref 省略 |
| `pom: \|` | 可选 | **原样写 Maven `<dependency>` XML 块**（字面量，不是结构化列表）。Paper 自带的库（Gson 等）可注释说明可省 |
| `depends` | 常用 | 本文件依赖的其它 md（相对路径 + 行内注释），生成时一并注入；与 brief 的 `depends` 对齐 |
| `globals` | gen 常用 | 本文件**对外暴露**的全局符号（类名 / 静态方法 / 字段），供别处 `depends`/`uses` 引用 |
| `uses` | 可选 | 反向/弱链接：谁会调用本类、本类又用到谁（说明数据流，不强制注入） |
| `main_wiring: \|` | 需接线的类 | 给出在 `Main.onEnable` / `onDisable` 里实例化、启动、关闭的**确切片段**（Manager 类尤其需要） |
| `role` / `notes` | ref 常用 | 一句职责 + 注意事项 |

范本 `protocol/websocket.md` 的真实 frontmatter（注意 `pom` 是字面 XML 块）：

```yaml
---
kind: gen
fileGen: ManagerGen

pom: |
  <dependency>
      <groupId>org.java-websocket</groupId>
      <artifactId>Java-WebSocket</artifactId>
      <version>1.5.7</version>
  </dependency>

depends:
  - ../handler/JsonParser.md     # 收到消息后调 OneBotHandler.MsgDivider 解析

globals:                          # 本类对外暴露的全局符号
  - Main.get().getOneBotClient()  # ManagerGen 约定：Main 持有 OneBotClient 实例
  - OneBotApi.client              # onOpen 时回写：OneBotApi.client = this

uses:
  - ../api/OneBotApi.md           # 发消息侧通过 OneBotApi.client 复用本连接

main_wiring: |
  // Main.onEnable 末尾：new OneBotClient(URI.create(ConfigManager.WsUrl)).connect();
  // Main.onDisable：    if (oneBotClient != null) oneBotClient.close();
---
```

- `pom` 里只写**额外引入**的库；`paper-api` / `spigot-api` 由踏海按 `coreType`+`mcVersion` 自动注入，**不要写**。
- 纯 Java 第三方库（如 `Java-WebSocket`）走 `compile` + maven-shade 打进 jar（见铁律 3b）；选轻量、无传递依赖的。
- `globals` / `uses` / `expectedGlobals` 共同构成 skill 内外的**符号契约**：谁暴露、谁引用、谁是宿主前置，
  让低端模型生成时不必猜类名/方法名，照符号抄即可。

### 5.2 正文

**能力型 skill 鼓励给大量可照抄代码**（这与早期"只写要点"的思路相反——能力扩展靠样板），但务必业务中立。
范本里每个正文都按同一骨架写，建议沿用：

1. `# 类名 — 一句话职责（GenType）` 标题；
2. 一两段散文：本类在数据流里的位置、谁调它、它调谁；
3. 一个 `> **GenType 约束** / **职责边界** / **禁忌**` 引用块，把红线写死；
4. 一段 ```java``` 代码：
   - **ref** 给**完整可抄的方法体**（如 `sendGroupMsg.md` 的 `send_group_msg` 报文拼装）；
   - **gen 的入口类**（指令路由这类）给 **`// TODO` + 注释掉的示例骨架**，把"做什么"留给用户当次需求，
     保持业务中立（见范本 `handler/groupMsgHandler.md`：只给 `switch(cmd)` 注释样例，不写死任何玩法）。

> 红线示例（取自范本）：UtilGen 全 `public static` 无实例字段；指令 Handler **不**自己 `new WebSocketClient`、
> **不**自己校验群号（上游 `OneBotHandler` 已过滤）；阻塞/耗时操作丢 `BukkitRunnable.runTaskAsynchronously(Main.get())`；
> 发往 QQ 端先 `stripColor` 去掉 `§x` 颜色码。**这些"该问就问、勿硬编码"的边界全部写进引用块，别埋在代码注释里。**

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
   - **3b. 纯 Java 第三方库**（如 `Java-WebSocket`、JSON 库）— **允许引入**，但 `pom` 给**确定坐标**、
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

复制范本 skill `qqbot-onebot/`（§3 结构 + §4 `brief.json` + §5 的 frontmatter & 正文骨架）整套，
按你的能力场景替换库坐标与模板即可——它就是一个合规的能力型 skill 范本。

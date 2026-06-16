---
kind: gen
fileGen: UtilGen

# 该文件由踏海生成为 OneBotApi.java（UtilGen：全 public static，无实例字段）
# 聚合 sendGroupMsg / sendPrivateMsg / sendGroupAt 三个报文模板为完整工具类

depends:
  - sendGroupMsg.md             # sendG 方法实现
  - sendPrivateMsg.md           # sendP 方法实现
  - sendGroupAt.md              # sendGroupAt 方法实现

globals:                         # 本类对外暴露的全局符号（其他 gen 文件可引用）
  - OneBotApi.client            # public static OneBotClient client，由 ManagerGen 注入
  - OneBotApi.sendG(message)
  - OneBotApi.sendP(userQQ, message)
  - OneBotApi.sendGroupAt(userQQ)

uses:
  - ../protocol/websocket.md    # ManagerGen 在建连成功后赋值 OneBotApi.client = this
  - ../handler/groupMsgHandler.md
  - ../handler/privateMsgHandler.md
---

# OneBotApi — 对外发消息静态工具类（UtilGen）

把三个 `kind:"ref"` 报文模板聚合到同一个类里。模型生成时按以下结构落盘：

```java
public class OneBotApi {

    // 由 OneBotClient（ManagerGen）在 onOpen 成功后赋值
    public static OneBotClient client;

    // ===== 来自 sendGroupMsg.md =====
    public static void sendG(String message) { /* 见 sendGroupMsg.md */ }

    // ===== 来自 sendPrivateMsg.md =====
    public static void sendP(String userQQ, String message) { /* 见 sendPrivateMsg.md */ }

    // ===== 来自 sendGroupAt.md =====
    public static void sendGroupAt(String userQQ) { /* 见 sendGroupAt.md */ }

    // 颜色码剥离：MC 端的 §x 颜色码不应发到 QQ 端
    private static String stripColor(String s) {
        return s == null ? "" : s.replaceAll("§[0-9a-fk-or]", "");
    }
}
```

> **生成约束（UtilGen）**：
> - 全 `public static`，禁实例字段（除 `client` 这一个静态门面）
> - 不持插件引用（不 import `Main`），日志走 `Bukkit.getLogger()` 或异常直接抛
> - 不在这里做重试 / 排队 — 链路状态由 `OneBotClient`（ManagerGen）管

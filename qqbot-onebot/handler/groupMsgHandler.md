---
kind: gen
fileGen: UtilGen

# 该文件由踏海生成为 GroupMsgHandler.java（UtilGen：全 public static，无实例字段）
# 仅负责"群消息指令路由 + 通过 OneBotApi 回复"，不持有 WebSocketClient 引用

depends:
  - ../api/OneBotApi.md          # 回复群消息统一走 OneBotApi.sendG / sendGroupAt

globals:                          # 模型生成时可直接引用的全局符号
  - Main.get()                    # 主类单例（MainGen 提供）
  - ConfigManager.GroupId         # 目标 QQ 群号（ConfigClassGen 提供）
  - OneBotApi.sendG(msg)          # 发送群消息
  - OneBotApi.sendGroupAt(userQQ) # 群内 @ 某人（常和 sendG 组合）

uses:                             # 反向链接：谁会调用本类
  - handler/JsonParser.md         # OneBotHandler.MsgDivider 在 message_type == "group" 时派发
---

# GroupMsgHandler — 群消息处理器（UtilGen）

被 `OneBotHandler.MsgDivider` 派发的群消息入口（已通过 `ConfigManager.GroupId` 过滤目标群）。
在 `handle(userQQ, msg)` 里做指令解析，回复时统一调 `OneBotApi.sendG(...)` 或先 `sendGroupAt` 再 `sendG`。

> **职责边界**：仅做"指令路由 + 回复"；网络收发交给 `OneBotClient`，JSON 解析交给 `OneBotHandler`。
> **禁忌**：
> - 不要在这里再校验群号（`OneBotHandler` 已过滤）
> - 不要在这里 `new WebSocketClient(...)` 或直接持有 ws 引用
> - 阻塞 IO / 数据库操作必须丢到 `BukkitRunnable.runTaskAsynchronously(Main.get())`

```java
public class GroupMsgHandler {

    public static void handle(String userQQ, String msg) {
        msg = msg.trim();
        if (msg.isEmpty()) return;

        // TODO: 在此实现群消息指令路由
        // 示例骨架：
        // String[] parts = msg.split("\\s+", 2);
        // String cmd = parts[0].toLowerCase();
        // String args = parts.length > 1 ? parts[1] : "";
        //
        // switch (cmd) {
        //     case "ping" -> {
        //         OneBotApi.sendGroupAt(userQQ);
        //         OneBotApi.sendG(" pong");
        //     }
        //     case "list" -> /* 在线玩家列表 */ ;
        //     default     -> { /* 静默或回执 */ }
        // }
    }
}
```

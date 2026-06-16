---
kind: gen
fileGen: UtilGen

# 该文件由踏海生成为 PrivateMsgHandler.java（UtilGen：全 public static，无实例字段）
# 仅负责"私聊指令路由 + 通过 OneBotApi 回复"，不持有 WebSocketClient 引用

depends:
  - ../api/OneBotApi.md          # 回复私聊统一走 OneBotApi.sendP

globals:                          # 模型生成时可直接引用的全局符号
  - Main.get()                    # 主类单例（MainGen 提供）
  - ConfigManager.*               # 配置项（ConfigClassGen 提供）
  - OneBotApi.sendP(userQQ, msg)  # 发送私聊（UtilGen 提供）

uses:                             # 反向链接：谁会调用本类
  - handler/JsonParser.md         # OneBotHandler.MsgDivider 在 message_type == "private" 时派发
---

# PrivateMsgHandler — 私聊消息处理器（UtilGen）

被 `OneBotHandler.MsgDivider` 派发的私聊消息入口。
在 `handle(userQQ, msg)` 里做指令解析，回复时统一调 `OneBotApi.sendP(userQQ, ...)`。

> **职责边界**：仅做"指令路由 + 回复"；网络收发交给 `OneBotClient`，JSON 解析交给 `OneBotHandler`。
> **禁忌**：
> - 不要在这里 `new WebSocketClient(...)` 或直接持有 ws 引用
> - 不要在这里做长耗时阻塞操作（解析线程会被卡）；耗时逻辑用 `BukkitRunnable.runTaskAsynchronously(Main.get())`

```java
public class PrivateMsgHandler {

    public static void handle(String userQQ, String msg) {
        msg = msg.trim();
        if (msg.isEmpty()) return;

        // TODO: 在此实现私聊指令路由
        // 示例骨架：
        // String[] parts = msg.split("\\s+", 2);
        // String cmd = parts[0].toLowerCase();
        // String args = parts.length > 1 ? parts[1] : "";
        //
        // switch (cmd) {
        //     case "ping"  -> OneBotApi.sendP(userQQ, "pong");
        //     case "bind"  -> /* 绑定逻辑 */ ;
        //     default      -> OneBotApi.sendP(userQQ, "未知指令：" + cmd);
        // }
    }
}
```

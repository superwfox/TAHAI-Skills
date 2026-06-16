# 用法教学（qqbot-onebot · usage.md）

这套能力让插件通过 **OneBot v11 正向 WebSocket** 对接 QQ 机器人。先理解数据流，再按接线落地。

## 数据流

```
OneBotClient(ws 长连接)
   └─ onMessage ─▶ OneBotHandler.MsgDivider(解析消息段 + 按 message_type 分发)
         ├─ "private" ─▶ PrivateMsgHandler.handle(userQQ, msg) ─回复─▶ OneBotApi.sendP
         └─ "group"   ─▶ GroupMsgHandler.handle(userQQ, msg)  ─回复─▶ OneBotApi.sendG / sendGroupAt
发送侧：任何地方调用 OneBotApi.sendG / sendP / sendGroupAt（内部复用 OneBotClient 连接）
```

## 接线（写进 Main）

- `onEnable` 末尾：`this.oneBotClient = new OneBotClient(URI.create(ConfigManager.WsUrl)); this.oneBotClient.connect();`
- `onDisable`：`if (oneBotClient != null) oneBotClient.close();`
- 连接断了由 `OneBotClient.retry()` 自动 5 秒重连，**不需要**额外保活任务。

## 必做

- `config.yml` 暴露：OneBot 服务端 ws 地址（WsUrl）、目标 QQ 群号（GroupId）、access_token（如有）。由 `ConfigManager`（ConfigClassGen）读出。
- 指令逻辑只写在 `PrivateMsgHandler.handle` / `GroupMsgHandler.handle` 里；网络收发交给 `OneBotClient`，解析交给 `OneBotHandler`。

## 该问就问（缺这些信息时，在 config 暴露并提示用户填，不要替用户臆测）

- OneBot 服务端 ws 地址与端口、access_token（如有鉴权）、要对接的目标 QQ 群号。

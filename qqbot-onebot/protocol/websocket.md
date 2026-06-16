---
kind: gen
fileGen: ManagerGen

# 该文件由踏海生成为 OneBotClient.java（ManagerGen：Main 持有引用，禁 static getInstance）
# 维护到 OneBot 的正向 WebSocket 长连接 + 自动重连，收到消息交 OneBotHandler 解析

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
  // Main.onEnable 中：
  //   this.oneBotClient = new OneBotClient(URI.create(ConfigManager.WsUrl));
  //   this.oneBotClient.connect();
  // Main.onDisable 中：
  //   if (oneBotClient != null) oneBotClient.close();
---

# OneBotClient — 正向 WebSocket 客户端（ManagerGen）

继承 `org.java_websocket.client.WebSocketClient`，由 `Main.onEnable` 实例化并保留引用。
`onOpen` 时把自身赋给 `OneBotApi.client`，`onMessage` 时把原文丢给 `OneBotHandler.MsgDivider` 解析。
`onClose` 走 `retry()` 5 秒后重连。

> **ManagerGen 约束**：
> - **禁** `static getInstance()` —— Main 持有唯一引用，外部通过 `Main.get().getOneBotClient()` 访问
> - 构造期不做阻塞 IO（`connect()` 由 Main 在 `onEnable` 末尾调用）
> - 提供 `shutdown()` 等价物 —— 这里就是父类 `close()`
> - 自动重连用独立线程，避免阻塞 ws 回调线程

```java
public class OneBotClient extends WebSocketClient {

    public OneBotClient(URI serverUri) {
        super(serverUri);
    }

    @Override
    public void onOpen(ServerHandshake serverHandshake) {
        OneBotApi.client = this;
        Main.get().getLogger().info("§f OneBotWebsocket 连接成功");
    }

    @Override
    public void onMessage(String s) {
        OneBotHandler.MsgDivider(s);
    }

    @Override
    public void onClose(int i, String s, boolean b) {
        retry();
    }

    @Override
    public void onError(Exception e) {
        // 由上层日志拦截；这里保持静默避免刷屏
    }

    private void retry() {
        if (this.isOpen()) return;
        new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException ignored) {
            }
            this.reconnect();
        }, "OneBot-Reconnect").start();
    }
}
```

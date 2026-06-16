# 禁止事项（qqbot-onebot · deny.md）

下面每一条都是硬约束，违反任意一条都视为**错误实现**，生成时必须严格遵守：

- **不要自己实现心跳 / ping-pong。** OneBot 正向 WebSocket 的心跳由协议端（go-cqhttp / NapCat / Lagrange 等）维护；客户端只需保持连接 + 断线重连（已在 `OneBotClient.retry()` 里处理）。**禁止**新建 `HeartbeatTask`、禁止定时发 ping 帧、禁止为"保活"再起一个 `BukkitRunnable`。
- **全程只有一个 WebSocket 连接。** 只有 `OneBotClient` 这一个连接；发消息一律复用 `OneBotApi.client`。**禁止**在 Handler / Command / Listener 里 `new WebSocketClient(...)` 或再建第二套连接。
- **不要硬编码部署参数。** ws 地址、目标群号、access_token 一律从 `ConfigManager`（config.yml）读取；缺省值也走配置。禁止把它们写死在 Java 里。
- **不要在 ws 回调线程里阻塞。** `onMessage` / `handle` 内禁止做长耗时或阻塞操作（网络、文件、sleep）；耗时逻辑丢 `BukkitRunnable.runTaskAsynchronously(Main.get())`。
- **持久化只用 YAML / PDC。** 禁止 SQL / JDBC / 数据库。
- **发往 QQ 端前先 `stripColor`。** 不要把 Minecraft 的 `§x` 颜色码发到 QQ。

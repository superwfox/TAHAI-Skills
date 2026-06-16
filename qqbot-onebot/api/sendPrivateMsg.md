---
kind: ref

# 这是 OneBotApi 类中 sendP 方法的报文模板片段
# 由 api/OneBotApi.md（UtilGen）聚合后生成实际 Java 类

role: 发送私聊消息的 OneBot 报文模板（action = send_private_msg）

globals:
  - OneBotApi.client            # 已建连的 OneBotClient 静态字段

notes:
  - user_id 在 OneBot 协议里可以是 long 或 string；这里直接传 String 也兼容
  - 同样使用 stripColor 剥离 §x 颜色码
---

```java
public class OneBotApi {

    public static void sendP(String userQQ, String message) {
        if (client == null || !client.isOpen()) return;

        JsonObject params = new JsonObject();
        params.addProperty("user_id", userQQ);
        params.addProperty("message", stripColor(message));
        params.addProperty("auto_escape", false);

        JsonObject request = new JsonObject();
        request.addProperty("action", "send_private_msg");
        request.add("params", params);

        try {
            client.send(request.toString());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

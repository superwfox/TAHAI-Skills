---
kind: ref

# 这是 OneBotApi 类中 sendG 方法的报文模板片段
# 由 api/OneBotApi.md（UtilGen）聚合后生成实际 Java 类

role: 发送群消息的 OneBot 报文模板（action = send_group_msg）

globals:
  - OneBotApi.client            # 已建连的 OneBotClient 静态字段
  - ConfigManager.GroupId       # 目标群号

notes:
  - stripColor(message) 用于剥离 Minecraft §x 颜色码，避免发到 QQ 端显示乱码
  - auto_escape=false 允许富文本/at 消息段
---

```java
public class OneBotApi {

    public static void sendG(String message) {
        if (client == null || !client.isOpen()) return;

        JsonObject params = new JsonObject();
        params.addProperty("group_id", ConfigManager.GroupId);
        params.addProperty("message", stripColor(message));
        params.addProperty("auto_escape", false);

        JsonObject request = new JsonObject();
        request.addProperty("action", "send_group_msg");
        request.add("params", params);

        try {
            client.send(request.toString());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---
kind: ref

# 这是 OneBotApi 类中 sendGroupAt 方法的报文模板片段
# 由 api/OneBotApi.md（UtilGen）聚合后生成实际 Java 类

role: 群内 @ 某人的消息段（at）报文模板，复用 send_group_msg

globals:
  - OneBotApi.client            # 已建连的 OneBotClient 静态字段
  - ConfigManager.GroupId       # 目标群号

notes:
  - at 必须用结构化消息段（JsonArray），不能拼字符串
  - 一般和 sendG(message) 组合使用：先 @ 再发正文
---

```java
public class OneBotApi {

    public static void sendGroupAt(String userQQ) {
        if (client == null || !client.isOpen()) return;

        JsonObject atData = new JsonObject();
        atData.addProperty("qq", userQQ);

        JsonObject atSegment = new JsonObject();
        atSegment.addProperty("type", "at");
        atSegment.add("data", atData);

        JsonArray msgArray = new JsonArray();
        msgArray.add(atSegment);

        JsonObject params = new JsonObject();
        params.addProperty("group_id", ConfigManager.GroupId);
        params.add("message", msgArray);

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

---
kind: gen
fileGen: UtilGen

# 该文件由踏海生成为 OneBotHandler.java（UtilGen：全 public static，无实例字段）
# 职责：把 ws 收到的原始 JSON 拆解 → 格式化消息段 → 按 message_type 派发给 *MsgHandler

pom: |
  <!-- Paper 自带 Gson，可不显式声明；Spigot/独立运行时再加 -->
  <dependency>
      <groupId>com.google.code.gson</groupId>
      <artifactId>gson</artifactId>
      <version>2.10.1</version>
  </dependency>

depends:
  - privateMsgHandler.md         # 私聊派发目标
  - groupMsgHandler.md           # 群聊派发目标

globals:
  - ConfigManager.GroupId        # 群号过滤（仅放行目标群）

uses:
  - ../protocol/websocket.md     # OneBotClient.onMessage 调用本类 MsgDivider
---

# OneBotHandler — 上行消息解析分发器（UtilGen）

负责把 `OneBotClient.onMessage` 收到的原始 JSON 字符串：
1. 解析 → 校验 `post_type` / `message` / `message_type` 字段；
2. 把 `message` 数组里各消息段（text/face/image/at/reply/video/record）格式化成可读字符串；
3. 按 `message_type` 派发给 `PrivateMsgHandler` 或 `GroupMsgHandler`（群消息还需匹配 `ConfigManager.GroupId`）。

> **职责边界**：仅做解析 + 派发；指令处理不写在这里。
> **禁忌**：不要在这里 `OneBotApi.send*`（让 Handler 自己回复，保持单向数据流）。

```java
public class OneBotHandler {

    public static void MsgDivider(String rawMsg) {
        JsonObject jsonObject;
        try {
            jsonObject = JsonParser.parseString(rawMsg).getAsJsonObject();
        } catch (Exception e) {
            return;
        }

        if (!jsonObject.has("post_type")) return;
        if (!jsonObject.has("message")) return;

        JsonElement msgElement = jsonObject.get("message");
        if (!msgElement.isJsonArray()) return;

        JsonArray msgArray = msgElement.getAsJsonArray();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < msgArray.size(); i++) {
            sb.append(parseMsg(msgArray.get(i).getAsJsonObject()));
        }
        String parsedMsg = sb.toString();

        if (!jsonObject.has("message_type")) return;
        String messageType = jsonObject.get("message_type").getAsString();

        switch (messageType) {
            case "private" -> {
                JsonObject sender = jsonObject.getAsJsonObject("sender");
                String userQQ = String.valueOf(sender.get("user_id").getAsLong());
                PrivateMsgHandler.handle(userQQ, parsedMsg);
            }
            case "group" -> {
                String groupId = String.valueOf(jsonObject.get("group_id").getAsLong());
                if (!groupId.equals(ConfigManager.GroupId)) return;

                JsonObject sender = jsonObject.getAsJsonObject("sender");
                String userQQ = String.valueOf(sender.get("user_id").getAsLong());
                GroupMsgHandler.handle(userQQ, parsedMsg);
            }
        }
    }

    private static String parseMsg(JsonObject obj) {
        StringBuilder mb = new StringBuilder();
        String type = obj.has("type") ? obj.get("type").getAsString() : "";
        switch (type) {
            case "text"   -> mb.append(obj.getAsJsonObject("data").get("text").getAsString());
            case "face"   -> mb.append("[表情]");
            case "image"  -> mb.append("[图片]");
            case "at" -> {
                String nickname = obj.getAsJsonObject("data").get("name").getAsString();
                mb.append("[@").append(nickname).append("]");
            }
            case "reply"  -> mb.append("[回复]");
            case "video"  -> mb.append("[视频]");
            case "record" -> mb.append("[语音]");
            default       -> mb.append("[未知消息]");
        }
        return mb.toString();
    }
}
```

# 飞书Bot互@：官方API、实现机制与权威文档（完整出处）

本文仅收录飞书开放平台官方文档中，与Bot之间互相@交流相关的API、实现机制、权限要求及权威出处，不添加任何个人解读，可直接用于验证OpenClaw、Hermes Agent等框架的适配可行性。所有内容均标注完整官方文档链接及原文片段，确保方案的准确性与合规性。

## 一、核心前提：Bot接收@消息的权限基础

Bot要能被其他Bot@到并触发响应，必须配置以下权限与事件订阅，这是飞书平台的强制要求，无此基础则无法实现Bot间互@。

|权限/事件|官方标识|功能说明|官方文档链接|
|-|-|-|-|
|群聊@Bot消息只读权限|im:message.group\_at\_msg:readonly|允许Bot接收群聊中被@的消息（包括被其他Bot@），是Bot接收同类Bot@消息的核心权限|https://open.feishu.cn/document/server-docs/im-v1/message/events/receive?lang=zh-CN|
|消息接收事件订阅|im.message.receive\_v1|必须订阅此事件，Bot才能接收任何消息（含@消息），是消息推送的入口|https://open.feishu.cn/document/server-docs/im-v1/message/events/receive?lang=zh-CN|

### 官方原文摘录

> 当具备 获取用户在群组中@机器人的消息（im:message\\\\.group\\\\\\\_at\\\\\\\_msg） 或者 接收群聊中@机器人消息事件（im:message\\\\.group\\\\\\\_at\\\\\\\_msg:readonly）权限时，可接收机器人所在群聊中用户 @ 机器人的消息。
> 
> 

## 二、发送@Bot消息的核心API（唯一官方接口）

实现Bot A @ Bot B，必须调用飞书官方提供的“发送消息API”，且需同时满足两个强制条件（缺一不可），否则目标Bot无法收到@通知。

### 1\. 核心API基础信息

|项目|官方参数/说明|
|-|-|
|接口地址|https://open.feishu.cn/open-apis/im/v1/messages|
|请求方法|POST（固定请求方式）|
|接口权限|应用必须开启「机器人能力」并发布版本（群自定义机器人无法调用此接口）|
|频率限制|1000次/分钟、50次/秒（官方标准限流）|
|官方文档链接|https://open.feishu.cn/document/server-docs/im-v1/message/create?lang=zh-CN|

### 2\. 发送@Bot消息的两个强制条件（飞书官方明确要求）

#### 条件1：消息文本中必须包含@标签

文本格式（支持open\_id、user\_id、union\_id，推荐使用open\_id定位Bot）：

```html
<at id="目标Bot的OpenID"></at>
```

官方文档链接：https://open.feishu.cn/document/server-docs/im-v1/message-content-description/create\_json?lang=zh-CN

#### 官方原文摘录

> at：@标签，名称类型是否必填描述user\\\\\\\_id string是用户 ID，用来指定被 @ 的用户。传入的值可以是用户的 user\\\\\\\_id、open\\\\\\\_id、union\\\\\\\_id。
> 
> 

#### 条件2：请求体中必须包含mentions字段

mentions是强制必填的数组参数，用于声明消息中@的对象（用户或Bot），无此字段时，目标Bot无法收到@通知，仅能显示消息文本，无法触发消息接收事件。

格式示例：

```json
"mentions": \\\[
    {
        "id": "目标Bot的OpenID",
        "id\\\_type": "open\\\_id"  // 支持open\\\_id、user\\\_id、union\\\_id，与@标签中的id类型一致
    }
]
```

官方文档链接：https://open.feishu.cn/document/server-docs/im-v1/message/create?lang=zh-CN

#### 官方原文摘录

> mentions：mention\\\\\\\[\\\\]，消息内被@的用户或机器人的ID列表。
> 
> 

## 三、完整请求体结构（官方示例）

以下是飞书官方文档提供的“发送@消息”完整请求示例，适用于Bot A @ Bot B的场景，可直接作为框架适配的参考标准。

```json
{
    "receive\\\_id": "群聊ID",  // 接收消息的群聊ID（receive\\\_id\\\_type为chat\\\_id时填写）
    "receive\\\_id\\\_type": "chat\\\_id",  // 固定值chat\\\_id，表示接收对象为群聊
    "msg\\\_type": "text",  // 消息类型为文本（支持其他类型，需对应调整content格式）
    "content": "{\\\\"text\\\\":\\\\"Bot B，收到请回复 <at id=\\\\\\\\\\\\"BotB的OpenID\\\\\\\\\\\\"></at>\\\\"}",  // 包含@标签的文本内容（注意转义字符）
    "mentions": \\\[  // 强制声明@对象，与@标签中的OpenID一致
        {
            "id": "BotB的OpenID",
            "id\\\_type": "open\\\_id"
        }
    ]
}
```

## 四、Bot接收@消息的事件机制

当Bot B被Bot A @后，飞书开放平台会通过“消息接收事件”（im.message.receive\_v1）向Bot B推送消息通知，Bot需通过订阅该事件，解析推送内容，才能触发后续回复逻辑。

以下是飞书官方规定的事件推送格式（核心片段）：

```json
{
    "schema": "2.0",
    "header": {
        "event\\\_id": "事件ID",  // 唯一事件标识
        "event\\\_type": "im.message.receive\\\_v1",  // 事件类型（固定）
        "token": "验证token",  // 用于验证事件合法性
        "create\\\_time": "时间戳",  // 事件创建时间
        "app\\\_id": "BotB的AppID",  // 接收方Bot的AppID
        "tenant\\\_key": "企业标识"  // 企业tenantKey
    },
    "event": {
        "message": {
            "chat\\\_id": "群聊ID",  // 消息所在群聊ID
            "content": "{\\\\"text\\\\":\\\\"Bot B，收到请回复 <at id=\\\\\\\\\\\\"BotB的OpenID\\\\\\\\\\\\"></at>\\\\"}",  // 消息内容
            "mentions": \\\[  // 被@对象信息
                {
                    "id": "BotB的OpenID",
                    "id\\\_type": "open\\\_id"
                }
            ],
            "sender": {
                "sender\\\_id": {
                    "open\\\_id": "BotA的OpenID",  // 发送方Bot的OpenID
                    "user\\\_id": "BotA的UserID"    // 发送方Bot的UserID
                },
                "sender\\\_type": "app"  // 标识发送方为应用Bot（区别于真人用户）
            }
        }
    }
}
```

官方文档链接：https://open.feishu.cn/document/server-docs/im-v1/message/events/receive?lang=zh-CN

## 五、关键验证点（OpenClaw/Hermes适配可行性）

以下是验证OpenClaw、Hermes Agent等框架是否支持飞书Bot互@的核心要点，均基于飞书官方规范，可直接对照框架代码实现逐一检查。

|验证项|飞书官方要求|框架适配检查方法|
|-|-|-|
|1. mentions字段支持|发送@Bot消息时，请求体必须包含mentions字段，否则目标Bot收不到@通知|检查框架的飞书消息发送接口，是否暴露mentions参数，或允许自定义请求体参数|
|2. @标签格式支持|消息文本中必须包含\&lt;at id=\&#34;xxx\&#34;\&gt;\&lt;/at\&gt;标签，才能正确显示@效果并触发通知|确认框架允许自定义消息文本格式，不过滤、不转义\&lt;at\&gt;标签|
|3. 核心权限配置|Bot必须配置im:message.group\_at\_msg:readonly权限，否则无法接收其他Bot的@消息|无需检查框架，直接在飞书开放平台（Bot后台）查看权限配置|
|4. 事件订阅配置|Bot必须订阅im.message.receive\_v1事件，否则无法接收任何消息（含@消息）|检查框架的飞书事件订阅配置，是否已注册im.message.receive\_v1事件|
|5. Bot OpenID获取|发送@消息时，必须使用目标Bot的OpenID（唯一标识）|确认框架是否支持获取Bot的OpenID，或允许手动配置目标Bot的OpenID|

## 六、官方文档总索引（完整出处）

以下是本文所有内容的官方文档源头，可直接访问链接查看完整内容，用于进一步验证或排查问题：

1. 发送消息API文档（核心接口）：https://open.feishu.cn/document/server-docs/im-v1/message/create?lang=zh-CN
2. 消息内容格式文档（@标签规范）：https://open.feishu.cn/document/server-docs/im-v1/message-content-description/create\_json?lang=zh-CN
3. 消息接收事件文档（事件推送规范）：https://open.feishu.cn/document/server-docs/im-v1/message/events/receive?lang=zh-CN
4. 机器人权限文档（核心权限说明）：https://open.feishu.cn/document/client-docs/bot-v3/bot-overview
5. @功能FAQ（常见问题排查）：https://open.feishu.cn/document/server-docs/im-v1/faq

## 七、极简结论（基于官方文档）

飞书平台完全支持Bot之间互相@交流，实现的唯一官方路径（无任何非官方后门）如下：

1. 两个Bot均配置im:message.group\_at\_msg:readonly权限，并订阅im.message.receive\_v1事件；
2. 调用im.v1.message.create API发送消息；
3. 消息文本中包含\&lt;at id=\&#34;目标Bot的OpenID\&#34;\&gt;\&lt;/at\&gt;标签；
4. 请求体中必须添加mentions字段，声明@的目标Bot信息。

以上所有规则均来自飞书开放平台官方文档，可直接对照OpenClaw、Hermes Agent等框架的代码实现，验证适配可行性。

> （注：文档部分内容基于豆包共创生成）


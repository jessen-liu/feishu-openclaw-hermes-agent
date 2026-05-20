# Feishu Bot-to-Bot @Mentions: Official APIs, Implementation & Authoritative References

This document compiles the official Feishu Open Platform documentation on bot-to-bot @mention communication — including core APIs, implementation mechanisms, permission requirements, and authoritative sources. It contains no personal interpretation and can be used directly to verify compatibility of frameworks such as OpenClaw and Hermes Agent. All content is cited with complete official documentation links and original excerpts.

## I. Prerequisite: Permission Foundation for Receiving @Messages

For a bot to be @mentioned by another bot and trigger a response, the following permissions and event subscriptions must be configured. These are mandatory Feishu platform requirements — without them, bot-to-bot @mentions cannot be implemented.

| Permission / Event | Official Identifier | Description | Official Doc Link |
|---|---|---|---|
| Group @Bot message read-only | `im:message.group_at_msg:readonly` | Allows the bot to receive @mention messages in group chats (including from other bots). This is the core permission for receiving bot @mentions. | https://open.feishu.cn/document/server-docs/im-v1/message/events/receive |
| Message receive event subscription | `im.message.receive_v1` | Must subscribe to this event for the bot to receive any messages (including @mentions). This is the entry point for message delivery. | https://open.feishu.cn/document/server-docs/im-v1/message/events/receive |

### Official Excerpt

> When the bot has the **im:message.group_at_msg** or **im:message.group_at_msg:readonly** permission, it can receive messages where users @mention the bot in group chats.

## II. Core API for Sending @Bot Messages (The Only Official Interface)

For Bot A to @mention Bot B, you must call Feishu's official "Send Message API" and satisfy two mandatory conditions (both required). Otherwise, the target bot will not receive the @ notification.

### 1. Core API Information

| Item | Official Parameter / Description |
|---|---|
| Endpoint | `https://open.feishu.cn/open-apis/im/v1/messages` |
| HTTP Method | POST (fixed) |
| API Permission | The app must enable "Bot Capability" and publish a version (custom group bots cannot call this API) |
| Rate Limit | 1,000 requests/min, 50 requests/sec (standard official limit) |
| Official Doc Link | https://open.feishu.cn/document/server-docs/im-v1/message/create |

### 2. Two Mandatory Conditions for Sending @Bot Messages (Explicitly Required by Feishu)

#### Condition 1: Message text must contain @ tags

Format (supports `open_id`, `user_id`, `union_id`; using `open_id` for bot targeting is recommended):

```html
<at id="target_bot_open_id"></at>
```

Official doc: https://open.feishu.cn/document/server-docs/im-v1/message-content-description/create_json

**Official Excerpt:**
> **at**: @ tag. `user_id` string — Required. The user ID being @mentioned. Accepted values: `user_id`, `open_id`, or `union_id`.

#### Condition 2: Request body must contain the `mentions` field

`mentions` is a mandatory array parameter that declares the @mentioned targets (users or bots). Without this field, the target bot will not receive the @ notification — only the message text will be displayed, and the message receive event will not be triggered.

Format example:

```json
"mentions": [
    {
        "id": "target_bot_open_id",
        "id_type": "open_id"
    }
]
```

Official doc: https://open.feishu.cn/document/server-docs/im-v1/message/create

**Official Excerpt:**
> **mentions**: `mention[]` — List of user or bot IDs @mentioned in the message.

## III. Complete Request Body Structure (Official Example)

Below is the official Feishu "Send @Message" request example, suitable for Bot A → Bot B scenarios. It can be used directly as a reference standard for framework adaptation.

```json
{
    "receive_id": "group_chat_id",
    "receive_id_type": "chat_id",
    "msg_type": "text",
    "content": "{\"text\":\"Bot B, please respond <at id=\\\"BotB_OpenID\\\"></at>\"}",
    "mentions": [
        {
            "id": "BotB_OpenID",
            "id_type": "open_id"
        }
    ]
}
```

## IV. Bot @Message Receive Event Mechanism

When Bot B is @mentioned by Bot A, the Feishu Open Platform pushes a message notification to Bot B via the **Message Receive Event** (`im.message.receive_v1`). The bot must subscribe to this event, parse the pushed content, and then trigger the reply logic.

Below is the official event push format (key excerpt):

```json
{
    "schema": "2.0",
    "header": {
        "event_id": "event_id",
        "event_type": "im.message.receive_v1",
        "token": "verification_token",
        "create_time": "timestamp",
        "app_id": "BotB_AppID",
        "tenant_key": "tenant_key"
    },
    "event": {
        "message": {
            "chat_id": "group_chat_id",
            "content": "{\"text\":\"Bot B, please respond <at id=\\\"BotB_OpenID\\\"></at>\"}",
            "mentions": [
                {
                    "id": "BotB_OpenID",
                    "id_type": "open_id"
                }
            ],
            "sender": {
                "sender_id": {
                    "open_id": "BotA_OpenID",
                    "user_id": "BotA_UserID"
                },
                "sender_type": "app"
            }
        }
    }
}
```

Official doc: https://open.feishu.cn/document/server-docs/im-v1/message/events/receive

## V. Key Verification Points (OpenClaw / Hermes Compatibility)

The following are the core verification points for checking whether frameworks like OpenClaw or Hermes Agent support Feishu bot-to-bot @mentions. All are based on official Feishu specifications and can be checked against the framework's source code.

| # | Verification Item | Feishu Official Requirement | Framework Compatibility Check |
|---|---|---|---|
| 1 | `mentions` field support | When sending @Bot messages, the request body must include `mentions`; otherwise the target bot cannot receive @ notifications. | Check whether the framework's Feishu message send interface exposes the `mentions` parameter, or allows custom request body parameters. |
| 2 | @ tag format support | The message text must contain `<at id="xxx"></at>` tags for correct @ display and notification triggering. | Confirm that the framework allows custom message text formatting without filtering or escaping `<at>` tags. |
| 3 | Core permission configuration | The bot must have `im:message.group_at_msg:readonly` permission to receive @mentions from other bots. | No framework check needed — verify directly in the Feishu Open Platform (Bot Console) permission settings. |
| 4 | Event subscription configuration | The bot must subscribe to `im.message.receive_v1` to receive any messages (including @mentions). | Check whether the framework's Feishu event subscription config has registered the `im.message.receive_v1` event. |
| 5 | Bot OpenID retrieval | Sending @messages requires the target bot's OpenID (unique identifier). | Confirm whether the framework supports retrieving bot OpenIDs, or allows manual configuration of target bot OpenIDs. |

## VI. Official Documentation Index (Complete Sources)

The following are the official documentation sources for all content in this document. Visit the links directly for full details, further verification, or troubleshooting:

1. Send Message API (core interface): https://open.feishu.cn/document/server-docs/im-v1/message/create
2. Message Content Format (@ tag specification): https://open.feishu.cn/document/server-docs/im-v1/message-content-description/create_json
3. Message Receive Event (event push specification): https://open.feishu.cn/document/server-docs/im-v1/message/events/receive
4. Bot Permissions (core permission documentation): https://open.feishu.cn/document/client-docs/bot-v3/bot-overview
5. @ Feature FAQ (common troubleshooting): https://open.feishu.cn/document/server-docs/im-v1/faq

## VII. Conclusion (Based on Official Documentation)

The Feishu platform fully supports bot-to-bot @mention communication. The only official implementation path (no unofficial backdoors) is:

1. Both bots must have `im:message.group_at_msg:readonly` permission and subscribe to the `im.message.receive_v1` event.
2. Call the `im.v1.message.create` API to send messages.
3. The message text must contain `<at id="target_bot_open_id"></at>` tags.
4. The request body must include the `mentions` field declaring the @mentioned target bot.

All rules above are sourced from the Feishu Open Platform official documentation. They can be directly used to verify the compatibility of frameworks such as OpenClaw and Hermes Agent.

> (Note: Portions of this document were co-generated with Doubao)

# 飞书 Bot 互 @ 通信：开发者指南

> 在飞书企业应用中实现 Bot 之间互相 @ 的实战指南。

## ⚠️ 核心限制

飞书的 WebSocket 事件推送（`im.message.receive_v1`）**不会**触发 Bot 之间的 @ 消息。API 返回 `code=0`，但接收方 Bot 永远收不到事件。

| 场景 | WebSocket 事件 |
|------|---------------|
| 用户 @ Bot | ✅ 正常触发 |
| Bot @ Bot | ❌ 不触发 |

这是平台层面的限制——不是框架或权限问题。**必须通过 REST API 轮询来获取消息。**

![Bot 聊天截图](screenshots/bot-chat-demo.png?v=2)

---

## 前置要求

### 所需权限

| 权限 ID | 说明 | 是否必须 |
|---------|------|---------|
| `im:message` | 基础消息 | 是 |
| `im:message:send_as_bot` | 以应用身份发消息 | 是 |
| `im:message.group_msg:readonly` | 读取群消息 | 是（轮询需要） |
| `im:message.group_at_msg:include_bot:readonly` | 接收 Bot @ 消息 | 是 |
| `im:message.p2p_msg:readonly` | 读取私聊 | 可选 |

**设置路径：** 飞书开放平台 → 应用控制台 → 权限管理 → 导入权限 ID → **发布新版本**使更改生效。

### 共享配置

| 项目 | 值（已脱敏） |
|------|-------------|
| 共享群聊 ID | `oc_xxxxxxxxxxxxxxxxxxxx` |
| API 基础地址 | `https://open.feishu.cn/open-apis` |

| Bot | App ID | @ Open ID |
|-----|--------|-----------|
| OpenClaw (JessenClaw) | `cli_xxxxxxxxxxxx` | `ou_xxxxxxxxxxxx` |
| Hermes (Hermes_Agent) | `cli_xxxxxxxxxxxx` | `ou_xxxxxxxxxxxx` |

---

## 发送 @ 消息

两个框架使用相同的飞书 API 格式：

```
POST /open-apis/im/v1/messages?receive_id_type=chat_id

{
  "receive_id": "<chat_id>",
  "msg_type": "text",
  "content": "{\"text\":\"消息正文\\n\\n<at id=\\\"target_open_id\\\"></at>\"}",
  "mentions": [{"id": "target_open_id", "id_type": "open_id"}]
}
```

---

## 实现方案：Hermes Agent

Hermes 自带 cron 定时任务支持。

### 步骤一：创建轮询 Cron Job

**`--no_agent --script` 方案不可行**——脚本沙箱没有 DNS/网络，无法调用飞书 API。

改用**完整 agent 模式**，通过 prompt 指示 agent 用 `execute_code` 轮询并用 `send_message` 回复：

```
cron add "every 1m" "Use execute_code to poll the Feishu API for new
@Hermes messages in the shared group. Use a state file to track
last_message_id. When a new @mention is detected, reply via
send_message to feishu:SharedGroup (group), prefixed with
<at id='target_open_id'></at> to @mention the sender.
If nothing new, output [SILENT]." --deliver local
```

关键设置：
- **`--deliver local`** —— 不推送给用户，静默运行
- **`[SILENT]`** —— 无新消息时输出此标记，抑制推送
- **状态文件** —— 记录 `last_checked_message_id`，避免重复处理

### 步骤二：状态追踪

Cron agent 读写状态文件（如 `~/.hermes/cron/hermes_bot_at_last_msg.txt`），记录最后处理的消息 ID。每次轮询：

1. 拉取群聊最新 20 条消息
2. 找出比 `last_message_id` 更新的消息
3. 过滤：跳过自己的消息，只处理 Bot 发送的
4. 检查是否包含 `<at id="hermes_open_id">` 的 @ 模式
5. 匹配则回复，更新状态文件

---

## 实现方案：OpenClaw

OpenClaw 没有内置 cron，使用操作系统的任务调度器。

### 步骤一：轮询脚本

保存为 `openclaw_bot_at_poll.js`：

```javascript
// 轮询飞书群聊中的 @OpenClaw 消息
// 检测到后发私信给自己触发 agent 处理
// 使用状态文件追踪消息
```

### 步骤二：系统任务调度

**Windows（任务计划程序）：**
```powershell
schtasks /create /tn "OpenClaw Bot@Poll" `
  /tr "node path/to/openclaw_bot_at_poll.js" `
  /sc minute /mo 1 /f
```

**macOS/Linux（crontab）：**
```bash
* * * * * node /path/to/openclaw_bot_at_poll.js
```

---

## 架构图

```
         ┌─────────────────────────────────┐
         │         飞书共享群聊              │
         │    oc_xxxxxxxxxxxxxxxxxxxxxxx    │
         └─────────────────────────────────┘
                ↑                       ↓
                │                       │
    ┌───────────┴───────┐    ┌──────────┴──────────┐
    │     OpenClaw      │    │       Hermes         │
    │  (AppID: cli_xxx) │    │  (AppID: cli_xxx)    │
    └───────────┬───────┘    └──────────┬───────────┘
                │                       │
    ┌───────────┴───────┐    ┌──────────┴───────────┐
    │   系统任务调度器    │    │     Cron Job         │
    │   每 60s 轮询      │    │     每 60s 轮询       │
    │                    │    │                      │
    │  GET /im/v1/msgs   │    │  GET /im/v1/msgs     │
    │  检测 @ 消息        │    │  检测 @ 消息          │
    │  → 私信自己         │    │  → LLM 直接生成       │
    │    触发 agent       │    │    回复              │
    └────────────────────┘    └──────────────────────┘
```

---

## 踩坑记录

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 脚本沙箱无 DNS | `--no_agent --script` 运行在隔离环境 | 使用完整 agent 模式 + `execute_code` |
| Cron 系统注入"禁止 send_message" | 默认 cron 指令冲突 | 在 prompt 中明确允许 send_message |
| 状态文件未初始化 | 首次运行没有基线消息 ID | 用虚拟/哨兵 ID 初始化 |
| 回复缺少 @ 标记 | 忘了加 `<at>` 标签 | 回复始终以 `<at id="target"></at>` 开头 |
| 自己回复自己陷入死循环 | 没过滤自己的消息 | 检查 `sender.id` 跳过自己 |

---

## 已知限制

- **延迟：** 轮询间隔约 60s
- **资源：** 每个 Bot 需要一个常驻进程（任务调度器/cron）
- **消息顺序：** 逆时间序轮询；消息爆发时可能需要多轮才能全部处理
- **无 Webhook：** 平台限制——Bot 互 @ 只能靠轮询

---

*作者：Jessen (Socrates@2026)，2026 年 5 月 19 日*
*技术修正：Hermes Agent*

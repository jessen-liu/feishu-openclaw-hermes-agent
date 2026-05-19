# Feishu Bot @ Bot Communication: Developer's Guide

> A practical guide to implementing bot-to-bot @mentions in Feishu enterprise apps.

## ⚠️ Critical Limitation

Feishu's WebSocket event push (`im.message.receive_v1`) does **NOT** fire for bot-to-bot @mentions. The API returns `code=0`, but the receiving bot never gets the event.

| Scenario | WebSocket Event |
|----------|----------------|
| Human @ Bot | ✅ Works |
| Bot @ Bot | ❌ No event |

This is a platform-level limitation — not a framework or permission issue. **Polling via REST API is required.**

---

## Requirements

### Required Permissions

| Permission ID | Description | Required? |
|--------------|-------------|-----------|
| `im:message` | Basic messaging | Yes |
| `im:message:send_as_bot` | Send as app | Yes |
| `im:message.group_msg:readonly` | Read group messages | Yes (polling) |
| `im:message.group_at_msg:include_bot:readonly` | Receive bot @mentions | Yes |
| `im:message.p2p_msg:readonly` | Read DMs | Optional |

**Setup:** Feishu Open Platform → App Console → Permissions → Import Permission IDs → **Publish a new version** for changes to take effect.

### Shared Configuration

| Item | Value (sanitized) |
|------|-------------------|
| Shared Group Chat ID | `oc_xxxxxxxxxxxxxxxxxxxx` |
| API Base URL | `https://open.feishu.cn/open-apis` |

| Bot | App ID | @ Open ID |
|-----|--------|-----------|
| OpenClaw (JessenClaw) | `cli_xxxxxxxxxxxx` | `ou_xxxxxxxxxxxx` |
| Hermes (Hermes_Agent) | `cli_xxxxxxxxxxxx` | `ou_xxxxxxxxxxxx` |

---

## Sending @ Messages

Both frameworks use the same Feishu API format:

```
POST /open-apis/im/v1/messages?receive_id_type=chat_id

{
  "receive_id": "<chat_id>",
  "msg_type": "text",
  "content": "{\"text\":\"message text\\n\\n<at id=\\\"target_open_id\\\"></at>\"}",
  "mentions": [{"id": "target_open_id", "id_type": "open_id"}]
}
```

---

## Implementation: Hermes Agent

Hermes has built-in cron support.

### Step 1: Create a Polling Cron Job

**The `--no_agent --script` approach does NOT work** — the script sandbox lacks DNS/network access and cannot call the Feishu API.

Instead, use **full agent mode** with a prompt that instructs the agent to poll via `execute_code` and reply via `send_message`:

```
cron add "every 1m" "Use execute_code to poll the Feishu API for new
@Hermes messages in the shared group. Use a state file to track
last_message_id. When a new @mention is detected, reply via
send_message to feishu:SharedGroup (group), prefixed with
<at id='target_open_id'></at> to @mention the sender.
If nothing new, output [SILENT]." --deliver local
```

Key settings:
- **`--deliver local`** — suppress delivery to user; cron runs silently
- **`[SILENT]`** — output this when no new messages to suppress delivery
- **State file** — track `last_checked_message_id` to avoid duplicate processing

### Step 2: State Tracking

The cron agent reads/writes a state file (e.g. `~/.hermes/cron/hermes_bot_at_last_msg.txt`) containing the last processed `message_id`. On each tick:

1. Fetch latest 20 messages from the group
2. Find messages newer than `last_message_id`
3. Filter: skip messages from self, only process bot senders
4. Check content for `<at id="hermes_open_id">` patterns
5. Reply if matched, update state file

---

## Implementation: OpenClaw

OpenClaw does not have built-in cron. Use the platform's task scheduler.

### Step 1: Polling Script

Save as `openclaw_bot_at_poll.js`:

```javascript
// Polls Feishu group for @OpenClaw messages
// Sends DM to self to trigger processing when detected
// Uses state file for message tracking
```

### Step 2: Schedule via Platform Task Scheduler

**Windows (Task Scheduler):**
```powershell
schtasks /create /tn "OpenClaw Bot@Poll" `
  /tr "node path/to/openclaw_bot_at_poll.js" `
  /sc minute /mo 1 /f
```

**macOS/Linux (crontab):**
```bash
* * * * * node /path/to/openclaw_bot_at_poll.js
```

---

## Architecture

```
         ┌─────────────────────────────────┐
         │        Shared Feishu Group       │
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
    │  Task Scheduler    │    │    Cron Job          │
    │  Poll every 60s    │    │    Poll every 60s    │
    │                    │    │                      │
    │  GET /im/v1/msgs   │    │  GET /im/v1/msgs     │
    │  Detect @mentions  │    │  Detect @mentions    │
    │  → DM self to      │    │  → LLM generates     │
    │    trigger agent   │    │    reply directly    │
    └────────────────────┘    └──────────────────────┘
```

---

## Pitfalls Encountered

| Issue | Cause | Solution |
|-------|-------|----------|
| Script sandbox has no DNS | `--no_agent --script` runs in isolated env | Use full agent mode with `execute_code` |
| Cron system injects "do NOT use send_message" | Default cron instructions conflict | Override in prompt: mention send_message is required |
| State file not initialized | First run has no baseline message ID | Seed with dummy/sentinel ID |
| Replies missing @mention | Forgot to prepend `<at>` tag | Always prefix reply with `<at id="target"></at>` |
| Self-reply loop | Not filtering own messages | Check `sender.id` and skip self |

---

## Known Limitations

- **Latency:** ~60s delay due to polling interval
- **Resource:** Each bot needs a persistent process (task scheduler / cron)
- **Message ordering:** Reverse chronological polling; bursts may need multiple ticks
- **No webhook:** Platform limitation — polling is the only option for bot-to-bot

---

*Written by Jessen (Socrates@2026), May 19 2026*
*Technical corrections by Hermes Agent*

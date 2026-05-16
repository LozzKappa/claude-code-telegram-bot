# Generic Pattern - Applicable to Any MCP Channel

> **Context:** This section generalizes the file-queue + bash watcher + tmux workaround documented for the Telegram plugin. The same pattern applies to any MCP channel plugin (Discord, Slack, WhatsApp, custom webhooks) where `notifications/claude/channel` is silently dropped by Claude Code.
>
> **Root bug:** Claude Code (confirmed through at least v2.1.139) receives MCP `notifications/claude/channel` notifications on the socket but never surfaces them to the active session. This affects every channel plugin uniformly - it is not plugin-specific.

---

## The Three-Component Pattern

Every working workaround for this bug follows the same structure:

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Component 1 - MCP Server (plugin-specific)                     â”‚
â”‚  Receives inbound message from external channel                  â”‚
â”‚  Writes message to inbox-queue.json (in addition to broken MCP) â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                             â”‚ writes
                             â–¼
                    inbox-queue.json
                             â”‚
                             â”‚ reads every 5s
                             â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Component 2 - Bash Watcher (invariant across all channels)     â”‚
â”‚  Polls the file every 5 seconds                                 â”‚
â”‚  Checks Claude is idle (not processing another message)         â”‚
â”‚  Sends trigger keystroke to Claude TUI via tmux send-keys       â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                             â”‚ tmux send-keys
                             â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Component 3 - Claude TUI (invariant across all channels)       â”‚
â”‚  Receives trigger command ("check and reply to inbox")          â”‚
â”‚  Reads inbox-queue.json                                         â”‚
â”‚  Replies via channel-specific MCP tool (e.g. telegram:reply)   â”‚
â”‚  Clears the queue (writes [])                                   â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

**The MCP notification path still fires** - do not remove it. The workaround adds a parallel delivery path; if Anthropic ships a fix, only the workaround code needs to be removed.

---

## Generic Pseudocode

### Component 1 - MCP Server patch (adapt per channel)

In your channel plugin's `server.ts` (or equivalent), locate the inbound message handler and add the file-write **before** the existing `mcp.notification()` call:

```typescript
// WORKAROUND: write to inbox so bash watcher can deliver to Claude
// Remove this block when notifications/claude/channel is fixed upstream.
import { readFileSync, writeFileSync } from 'fs'
import { join } from 'path'

const INBOX_PATH = join(STATE_DIR, 'inbox-queue.json')

function writeToInbox(message: {
  chat_id: string
  message_id?: string
  user: string
  user_id: string
  ts: string        // ISO 8601
  content: string
  [key: string]: string | undefined  // channel-specific extras
}): void {
  try {
    let queue: typeof message[] = []
    try { queue = JSON.parse(readFileSync(INBOX_PATH, 'utf8')) } catch {}
    queue.push(message)
    writeFileSync(INBOX_PATH, JSON.stringify(queue))
  } catch {}
}

// Call this in your inbound handler, before mcp.notification():
writeToInbox({
  chat_id: incomingMessage.channelId,
  message_id: incomingMessage.id,
  user: incomingMessage.senderName,
  user_id: incomingMessage.senderId,
  ts: new Date().toISOString(),
  content: incomingMessage.text,
})
```

**STATE_DIR** is wherever your plugin stores state. Use a consistent path - the bash watcher must read the same file.

---

### Component 2 - Bash Watcher (copy-paste, only change INBOX and SESSION)

```bash
inbox_watcher() {
  local INBOX="/path/to/channel/state/inbox-queue.json"  # adapt per channel
  local SESSION="claude-bot-session"                      # your tmux session name
  local TRIGGER_CMD="check and reply to inbox"            # must match Claude's CLAUDE.md
  local STALL_TIMEOUT=300
  local last_trigger=0
  local triggered=0

  while tmux has-session -t "$SESSION" 2>/dev/null; do
    sleep 5
    local content
    content=$(cat "$INBOX" 2>/dev/null)
    local inbox_has_msgs=0
    [ "$content" != "[]" ] && [ -n "$content" ] && inbox_has_msgs=1

    if [ "$inbox_has_msgs" -eq 1 ]; then
      local now
      now=$(date +%s)

      # Watchdog: stall detected - restart service
      if [ "$triggered" -eq 1 ] && [ $(( now - last_trigger )) -gt $STALL_TIMEOUT ]; then
        systemctl --user restart YOUR-SERVICE-NAME.service
        return
      fi

      # Fire only when Claude is idle
      if [ "$triggered" -eq 0 ]; then
        local pane
        pane=$(tmux capture-pane -t "$SESSION" -p 2>/dev/null)
        if echo "$pane" | grep -q "â¯\|âµâµ" && \
           ! echo "$pane" | grep -q "esc to interrupt"; then
          tmux send-keys -t "$SESSION" "$TRIGGER_CMD" Enter
          last_trigger=$(date +%s)
          triggered=1
        fi
      fi
    else
      triggered=0
      last_trigger=0
    fi
  done
}

inbox_watcher &
WATCHER_PID=$!
trap 'kill $WATCHER_PID 2>/dev/null' EXIT TERM INT HUP
```

Two lines change per channel: `INBOX` and `SESSION`. Everything else is invariant.

---

### Component 3 - Claude instruction (add to CLAUDE.md)

Claude needs to know what `TRIGGER_CMD` means and where the inbox is. Add this to your agent's `CLAUDE.md`:

```markdown
## Inbox command: "check and reply to inbox"

Read `/path/to/channel/state/inbox-queue.json`.
If it contains messages (non-empty array), process each one:
- Use the channel reply tool (e.g. `discord:reply`, `slack:post_message`) with `chat_id` and `content`
- Handle any channel-specific fields (image_path, attachment_file_id, etc.)
After replying to all messages, write `[]` to the file to clear the queue.
If the file is already `[]` or does not exist, do nothing.
```

---

## What Changes Per Channel

| Component | Telegram | Discord | Slack | Custom webhook |
|-----------|----------|---------|-------|---------------|
| MCP server | `server.ts` (bun, grammY) | Discord.js bot | Slack Bolt app | Any HTTP server |
| State dir | `~/.claude/channels/telegram/` | `~/.claude/channels/discord/` | `~/.claude/channels/slack/` | your choice |
| Inbox file | `inbox-queue.json` | `inbox-queue.json` | `inbox-queue.json` | same name, different path |
| Reply tool | `mcp__telegram__reply` | `mcp__discord__reply` | `mcp__slack__post_message` | your tool name |
| Extra fields | `image_path`, `reply_to` | `guild_id`, `attachment_file_id` | `thread_ts`, `channel` | whatever you need |
| Trigger command | "check and reply to inbox" | same | same | same |

**Inbox JSON schema** is intentionally flexible - the `content` and `chat_id` fields are the only required ones. Add channel-specific fields as needed; Claude will use whatever is present.

---

## What Stays the Same

Regardless of channel:

| Element | Description |
|---------|-------------|
| Inbox file format | `[{ chat_id, user, user_id, ts, content, ...extras }]` |
| File clear signal | `[]` - watcher stops triggering when it sees this |
| Idle detection | `grep "â¯\|âµâµ"` and `grep -v "esc to interrupt"` |
| Delivery mechanism | `tmux send-keys -t SESSION "TRIGGER_CMD" Enter` |
| Token cost at idle | **Zero** - pure bash, no Claude API calls |
| Stall recovery | Watchdog: restart service after STALL_TIMEOUT seconds |
| Removal path | Delete `writeToInbox()` block from MCP server; remove watcher from launch script |

---

## Adapting the Pattern: Step-by-Step

1. **Patch your MCP server** - add `writeToInbox()` before the `mcp.notification()` call. Map your channel's inbound event fields to the standard inbox schema.

2. **Set STATE_DIR** - pick a stable path on disk. Create it before the service starts.

3. **Copy the bash watcher** - change only `INBOX` and `SESSION`. Keep everything else identical.

4. **Update CLAUDE.md** - tell Claude where the inbox is, what the trigger command means, and which reply tool to use.

5. **Test** - write a test message directly to the JSON file, wait 10 seconds, verify Claude replies and clears the queue:
   ```bash
   echo '[{"chat_id":"YOUR_ID","user":"test","user_id":"0","ts":"2026-01-01T00:00:00Z","content":"hello"}]' \
     > /path/to/channel/state/inbox-queue.json
   ```

6. **Monitor** - `tmux capture-pane -t SESSION -p | tail -5` shows Claude's current state. The inbox file should go back to `[]` within 10 seconds of being written.

---

## Why Not Fix MCP Instead?

The `notifications/claude/channel` bug is inside Claude Code's session loop - not in the MCP server or plugin. The MCP server sends a well-formed notification; Claude Code receives it on the socket; it is then silently discarded before reaching the active session. This is confirmed in multiple upstream issues (#36802, #46744, #40729, #44283 and others).

Until Anthropic ships a fix, any solution must work around the delivery layer - which is exactly what this pattern does.

---

*Pattern extracted from the Telegram bot workaround - May 2026. Applies to Claude Code v2.1.80 through at least v2.1.139.*

---

## License

MIT License - Copyright (c) 2026 Khaled I. (LozzKappa)

Permission is hereby granted, free of charge, to any person obtaining a copy of this work to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of it, provided that this copyright notice is preserved in all copies.

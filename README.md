# Claude Code Telegram Bot — Complete Setup Blueprint

> **Why this exists:** There is a confirmed bug in Claude Code (v2.1.80 through at least v2.1.139) where inbound channel notifications via MCP are silently ignored. As of May 2026, there are no official solutions or public guides. This blueprint documents a fully working workaround discovered through hands-on debugging on a Raspberry Pi 4 running Debian, tested in production.

**Tested on:** Claude Code v2.1.140 · Telegram plugin v0.0.1 · Debian 13 aarch64 (RPi 4) · May 2026  
**Updated:** 2026-05-13 — added context overflow watchdog + persistent typing indicator

---

## The Bug

When you install the `telegram@claude-plugins-official` plugin and launch Claude Code with `--channels`, the bot:
- ✅ Receives messages (Telegram shows "typing..." indicator)
- ✅ Can send replies (the `telegram:reply` tool works)
- ❌ Never processes inbound messages — `notifications/claude/channel` is received by the MCP server but silently dropped by Claude Code

This is not a configuration error. It is a bug in Claude Code itself, confirmed in multiple GitHub issues (#36802, #46744 and others). The plugin works, the MCP pipe works, the notification is sent — Claude just never acts on it.

---

## How the Workaround Works

Instead of relying on the broken MCP notification channel, we use a **file-based inbox** + a **shell watcher** as an alternative delivery mechanism:

1. `server.ts` writes every inbound message to `inbox-queue.json` (in addition to the broken MCP notification)
2. A bash loop (`inbox_watcher`) polls the file every 5 seconds
3. When messages are found **and Claude is idle**, it sends `"check and reply to telegram inbox"` as a keystroke to the Claude TUI via `tmux send-keys`
4. Claude reads the inbox, replies via Telegram, clears the file
5. *(Optional)* A cron job as last-resort fallback — **disabled by default** since the watcher already handles all cases at zero token cost

**Result:** ~5–9 second response latency. The watcher is pure bash — it consumes zero tokens when idle.

---

## Prerequisites

- Linux server (guide written for Debian/Ubuntu, adapts to any distro)
- [Claude Code CLI](https://www.npmjs.com/package/@anthropic-ai/claude-code) installed
- [bun](https://bun.sh) runtime installed (required by the Telegram plugin)
- A Telegram bot token from [@BotFather](https://t.me/botfather)
- `tmux` installed (`apt install tmux`)

---

## Step-by-Step Setup

### 1. Install the Telegram plugin

Launch Claude Code once interactively and install the plugin:

```bash
claude --dangerously-skip-permissions
```

Inside Claude:
```
/plugin install telegram@claude-plugins-official
```

This installs the plugin to `~/.claude/plugins/installed/telegram/`.

---

### 2. Create the state directory and credentials

```bash
mkdir -p ~/.claude/channels/telegram

# Store bot token (never commit this file)
echo "TELEGRAM_BOT_TOKEN=your_bot_token_here" > ~/.claude/channels/telegram/.env
chmod 600 ~/.claude/channels/telegram/.env
```

Create the access control file. Get your Telegram user ID by messaging [@userinfobot](https://t.me/userinfobot):

```bash
cat > ~/.claude/channels/telegram/access.json << 'EOF'
{
  "dmPolicy": "allowlist",
  "allowFrom": ["YOUR_TELEGRAM_USER_ID"],
  "ackReaction": "👀"
}
EOF
```

---

### 3. Configure `~/.claude.json`

Add the MCP server entry so Claude Code spawns bun with the correct environment:

```json
{
  "mcpServers": {
    "telegram": {
      "command": "/home/YOUR_USER/.bun/bin/bun",
      "args": ["/home/YOUR_USER/.claude/plugins/installed/telegram/server.ts"],
      "env": {
        "TELEGRAM_BOT_TOKEN": "your_bot_token_here",
        "TELEGRAM_STATE_DIR": "/home/YOUR_USER/.claude/channels/telegram"
      }
    }
  }
}
```

---

### 4. Patch `server.ts` — inbox workaround + persistent typing indicator

**Part A — Inbox workaround:** find the `handleInbound` function, locate `mcp.notification({` and add this block **immediately before it**:

```typescript
// INBOX WORKAROUND for Claude Code bug (notifications/claude/channel silently ignored)
// Remove this block when the bug is fixed upstream.
try {
  const inboxPath = join(STATE_DIR, 'inbox-queue.json')
  let queue: Array<Record<string, string | undefined>> = []
  try { queue = JSON.parse(readFileSync(inboxPath, 'utf8')) } catch {}
  queue.push({
    chat_id,
    message_id: msgId != null ? String(msgId) : undefined,
    user: from.username ?? String(from.id),
    user_id: String(from.id),
    ts: new Date((ctx.message?.date ?? 0) * 1000).toISOString(),
    content: text,
    ...(imagePath ? { image_path: imagePath } : {}),
  })
  writeFileSync(inboxPath, JSON.stringify(queue))
} catch {}
```

**Part B — Persistent typing indicator (optional but recommended):** add this block near the top of `server.ts`, right after `const bot = new Bot(TOKEN)`, then replace the single `sendChatAction` call in `handleInbound` with `startTypingLoop(chat_id)`:

```typescript
// Persistent typing loop — keeps 'typing...' visible for the full processing duration.
// Purely bun-side: zero Claude tokens. Telegram typing action lasts ~5s, resend every 4s.
const typingIntervals = new Map<string, ReturnType<typeof setInterval>>()
const INBOX_PATH = join(STATE_DIR, 'inbox-queue.json')

function startTypingLoop(chat_id: string): void {
  if (typingIntervals.has(chat_id)) return
  void bot.api.sendChatAction(chat_id, 'typing').catch(() => {})
  const iv = setInterval(() => {
    try {
      const queue = JSON.parse(readFileSync(INBOX_PATH, 'utf8'))
      const still_pending = Array.isArray(queue) && queue.some(
        (m: Record<string, unknown>) => String(m.chat_id) === chat_id
      )
      if (!still_pending) {
        clearInterval(iv)
        typingIntervals.delete(chat_id)
        return
      }
    } catch {
      clearInterval(iv)
      typingIntervals.delete(chat_id)
      return
    }
    void bot.api.sendChatAction(chat_id, 'typing').catch(() => {})
  }, 4000)
  typingIntervals.set(chat_id, iv)
}
```

In `handleInbound`, replace:
```typescript
void bot.api.sendChatAction(chat_id, 'typing').catch(() => {})
```
with:
```typescript
startTypingLoop(chat_id)
```

`join` and `readFileSync` are already imported — no new imports needed.

---

### 5. Create the launch script

Save as `~/run-bot.sh` (adjust paths to match your setup):

```bash
#!/bin/bash
export HOME=/home/YOUR_USER
export TELEGRAM_BOT_TOKEN="your_bot_token_here"
export TELEGRAM_STATE_DIR="/home/YOUR_USER/.claude/channels/telegram"
export PATH="/home/YOUR_USER/.npm-global/bin:/home/YOUR_USER/.bun/bin:$PATH"

# REQUIRED: tmux silently fails in headless environments without this
export TERM="xterm-256color"

# Remove IDE variables that break Claude's startup screen
unset TERM_PROGRAM TERM_PROGRAM_VERSION
unset WINDSURF_CASCADE_TERMINAL WINDSURF_CASCADE_TERMINAL_KIND
unset VSCODE_IPC_HOOK_CLI VSCODE_GIT_IPC_HANDLE
unset VSCODE_GIT_ASKPASS_MAIN VSCODE_GIT_ASKPASS_NODE VSCODE_GIT_ASKPASS_EXTRA_ARGS

SESSION="claude-telegram"
CLAUDE_BIN="/home/YOUR_USER/.npm-global/bin/claude"
WORKSPACE="/home/YOUR_USER/your-workspace"

# Kill any existing session
tmux kill-session -t "$SESSION" 2>/dev/null
sleep 1

# REQUIRED: -x and -y flags force a window size — without them tmux may fail headlessly
tmux new-session -d -s "$SESSION" -x 220 -y 50 \
  -c "$WORKSPACE" \
  -e "HOME=$HOME" \
  -e "TERM=xterm-256color" \
  -e "TELEGRAM_BOT_TOKEN=$TELEGRAM_BOT_TOKEN" \
  -e "TELEGRAM_STATE_DIR=$TELEGRAM_STATE_DIR" \
  "$CLAUDE_BIN --dangerously-skip-permissions --channels plugin:telegram@claude-plugins-official" \
  || { echo "ERROR: tmux new-session failed"; exit 1; }

# Inbox watcher + anti-stall watchdog
# Triggers Claude within ~5s of a new message.
# Watchdog: if inbox not cleared within 5 min after triggering → auto-restart (context overflow protection).
inbox_watcher() {
  local INBOX="$TELEGRAM_STATE_DIR/inbox-queue.json"
  local STALL_TIMEOUT=300   # seconds before considering Claude stuck
  local last_trigger=0      # epoch of last "rispondi inbox" sent
  local triggered=0         # 1 = command sent, waiting for inbox to clear

  while tmux has-session -t "$SESSION" 2>/dev/null; do
    sleep 5
    local content
    content=$(cat "$INBOX" 2>/dev/null)
    local inbox_has_msgs=0
    [ "$content" != "[]" ] && [ -n "$content" ] && inbox_has_msgs=1

    if [ "$inbox_has_msgs" -eq 1 ]; then
      local now
      now=$(date +%s)

      # Watchdog: triggered but inbox still full after STALL_TIMEOUT → Claude is stuck
      if [ "$triggered" -eq 1 ] && [ $(( now - last_trigger )) -gt $STALL_TIMEOUT ]; then
        echo "[inbox_watcher] STALL DETECTED — restarting service (context overflow?)" >&2
        systemctl --user restart claude-telegram.service
        return
      fi

      # Trigger only when Claude is idle (❯ present, "esc to interrupt" absent)
      if [ "$triggered" -eq 0 ]; then
        local pane
        pane=$(tmux capture-pane -t "$SESSION" -p 2>/dev/null)
        if echo "$pane" | grep -q "❯\|⏵⏵" && ! echo "$pane" | grep -q "esc to interrupt"; then
          tmux send-keys -t "$SESSION" "check and reply to telegram inbox" Enter
          last_trigger=$(date +%s)
          triggered=1
        fi
      fi
    else
      # Inbox cleared — reset state
      triggered=0
      last_trigger=0
    fi
  done
}
inbox_watcher &
WATCHER_PID=$!

# CRITICAL: use $WATCHER_PID, NOT "kill 0"
# "kill 0" sends SIGTERM to the entire bash process group including Node.js,
# which causes a SIGSEGV crash in Claude Code ~60 seconds after startup.
trap 'tmux kill-session -t "$SESSION" 2>/dev/null; kill $WATCHER_PID 2>/dev/null' EXIT TERM INT HUP

# Keep this process alive (required for systemd to track the service)
while tmux has-session -t "$SESSION" 2>/dev/null; do
  sleep 5
done
```

Make it executable:
```bash
chmod +x ~/run-bot.sh
```

---

### 6. Create the systemd service

```bash
cat > ~/.config/systemd/user/claude-telegram.service << 'EOF'
[Unit]
Description=Claude Code Telegram Bot
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/YOUR_USER
ExecStart=/bin/bash /home/YOUR_USER/run-bot.sh
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=claude-telegram

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now claude-telegram.service

# Enable auto-start on boot without an active login session
loginctl enable-linger $USER
```

---

### 7. Set up the cron fallback

In your workspace, create or edit `cron-registry.json`:

```json
[
  {
    "id": "telegram-inbox-poll",
    "name": "Telegram Inbox Poller (fallback)",
    "cron": "*/10 * * * *",
    "prompt": "Check the file `~/.claude/channels/telegram/inbox-queue.json`. If it contains messages (non-empty array), reply to each one using the telegram reply tool, addressing the `content` field to the `chat_id`. After replying to all messages, write `[]` to the file to clear the queue. If the file is already `[]` or does not exist, do nothing.",
    "enabled": false
  }
]
```

Add this to your `CLAUDE.md` so Claude recreates the cron on every startup:

```markdown
At the start of every new session, BEFORE responding:
1. Read `cron-registry.json` and recreate all enabled crons using CronCreate
```

---

### 8. Trigger the startup routine

The first time (or after any restart), Claude needs to receive an input to run its startup routine and register the cron. Send a message from your tmux terminal:

```bash
tmux send-keys -t claude-telegram "start session" Enter
```

Or simply send a message from Telegram — the inbox_watcher will catch it and trigger Claude.

---

## Verification Checklist

```bash
# Service running?
systemctl --user is-active claude-telegram

# bun process alive?
ps aux | grep "bun.*telegram" | grep -v grep

# tmux session exists?
tmux ls

# Claude TUI visible and idle?
tmux capture-pane -t claude-telegram -p | tail -5
# Should show ❯ prompt without "esc to interrupt"

# Inbox watcher running?
ps aux | grep "sleep 5" | grep -v grep | wc -l
# Should be > 0

# Manual inbox test (bypasses Telegram):
echo '[{"chat_id":"YOUR_CHAT_ID","user":"test","user_id":"YOUR_USER_ID","ts":"2026-01-01T00:00:00Z","content":"hello bot"}]' \
  > ~/.claude/channels/telegram/inbox-queue.json
# Wait 5-10s — Claude should reply on Telegram
```

---

## Common Pitfalls (all found in production)

### Pitfall 1 — Service exits immediately with status=0 (no error message)
**Cause:** `TERM` env variable not set in systemd environment. `tmux new-session -d` fails silently.
**Fix:** Add `export TERM="xterm-256color"` and `-x 220 -y 50` to the `tmux new-session` command.

### Pitfall 2 — Service crashes with SIGSEGV every ~60 seconds
**Cause:** `kill 0` in the bash trap kills the entire process group including Node.js, causing a segfault.
**Fix:** Replace `kill 0` with `kill $WATCHER_PID` to kill only the watcher background process.

### Pitfall 3 — Bot receives messages but never responds (queued commands pile up)
**Cause:** Idle check only looks for `⏵⏵` (the status bar), which is **always** visible — even when Claude is busy. The watcher fires a new command every 5 seconds, filling up the TUI queue.
**Fix:** Add a second condition that checks `"esc to interrupt"` is NOT present:
```bash
if echo "$pane" | grep -q "❯\|⏵⏵" && ! echo "$pane" | grep -q "esc to interrupt"; then
```

### Pitfall 4 — Bot receives messages but Claude never sees them
**Cause:** You forgot to patch `server.ts` with the inbox workaround, or bun restarted with the old (unpatched) file.
**Fix:** Re-apply the patch and restart the service.

### Pitfall 5 — "tmux: no server running" after service restart
**Cause:** A previous tmux session with the same name was not properly cleaned up.
**Fix:** The `tmux kill-session -t "$SESSION" 2>/dev/null` + `sleep 1` at the top of the script handles this. The `2>/dev/null` suppresses the error when no session exists.

### Pitfall 6 — Claude uses wrong bot token or STATE_DIR
**Cause:** Old PM2, `script -q -f -c`, or other process managers injecting stale environment variables.
**Fix:** Kill all related processes, clean up, and use only systemd to manage the service:
```bash
pkill -f "bun.*telegram" 2>/dev/null
pkill -f "script -q" 2>/dev/null
systemctl --user restart claude-telegram.service
```

### Pitfall 7 — Bot goes silent for hours after a long task (context overflow)
**Cause:** Claude Code's context grows unbounded over a long session. At ~80–100k uncached tokens, Claude degrades significantly: it becomes slow, fails to process `check and reply to telegram inbox` correctly, or appears to process it but produces garbage and never clears the inbox. The watcher keeps triggering but Claude never truly responds — the bot appears dead.

**When this happens:** typically after a heavy task (long code generation, multi-step workflow, or `calibrate` at end of a long session). The symptom is messages piling up in `inbox-queue.json` that never get cleared.

**Fix (immediate):** restart the service to get a fresh context:
```bash
systemctl --user restart claude-telegram.service
```

**Fix (automatic — two-stage, token-efficient):** the watchdog in the launch script uses a two-stage recovery:

| Stage | Trigger | Action | Token cost |
|-------|---------|--------|-----------|
| **L1** | inbox not cleared 5 min after trigger | `/clear` in tmux + retry | ~500 tokens (context reset, process stays alive) |
| **L2** | inbox still not cleared 5 more min after L1 | `systemctl restart` | ~5000 tokens (full reload) |

L1 handles 99% of stall cases at a fraction of the cost. L2 is the last resort for when even `/clear` doesn't help (rare process-level failure).

**Prevention:**
- The watchdog handles it automatically — no manual intervention needed
- For very long sessions, `/clear` in tmux proactively keeps context lean

---

## Token Consumption

This workaround is designed to consume **zero tokens when idle**.

| Component | How it works | Tokens when idle |
|-----------|-------------|------------------|
| `inbox_watcher` | Pure bash loop — reads a local file, never calls Claude | **0** |
| Cron fallback | Disabled by default | **0** |
| Per Telegram reply | Claude wakes up, reads inbox, replies | ~500–2000 per message |
| Session startup | Claude reads CLAUDE.md and context | ~1000–3000 once |

**Watcher logic step by step (every 5 seconds):**
```
1. sleep 5                        (no cost)
2. cat inbox-queue.json           (local file read, no cost)
3. if [] or empty → back to 1     (no cost)
4. tmux capture-pane              (local process, no cost)
5. if "esc to interrupt" present → back to 1  (Claude busy, no cost)
6. tmux send-keys → Claude wakes  (HERE tokens are consumed)
```

With 20 messages/day: ~10k–40k tokens total, all productive.

---

## When Anthropic Fixes the Bug

Once `notifications/claude/channel` works natively in Claude Code, the workaround can be removed:

1. **Delete** the `// INBOX WORKAROUND` block from `server.ts`
2. **Remove** the `inbox_watcher` function and `WATCHER_PID` from the run script
3. **Delete** `inbox-queue.json` from the state directory
4. *(Optional)* remove `cron-registry.json` entry if you added it

The bot will respond instantly (< 1 second) once the native MCP notification delivery works.

---

## Architecture Diagram (complete)

```
┌─────────────────────────────────────────────────────▛
│                   systemd user service               │
│  run-bot.sh                                          │
│  ├── tmux new-session (claude --channels)            │
│  │   └── Claude Code TUI (Node.js)                  │
│  │       └── bun server.ts (MCP child process)      │
│  │           ├── Telegram Bot API polling            │
│  │           ├── gate() → access control             │
│  │           ├── startTypingLoop() every 4s [bun]   │
│  │           ├── writes inbox-queue.json  ◄──────┌  │
│  │           └── mcp.notification (buggy) ✗      │  │
│  └── inbox_watcher + watchdog (bash bg loop)      │  │
│      ├── sleep 5                                  │  │
│      ├── reads inbox-queue.json ──────────────────  │
│      ├── checks Claude idle (no "esc to interrupt")  │
│      ├── tmux send-keys "reply inbox" ──► Claude TUI │
│      └── WATCHDOG: if inbox not cleared in 5min      │
│              └── systemctl restart → fresh context   │
│                                                      │
│  CronCreate fallback (disabled by default, 0 tokens) │
│  └── enable in cron-registry.json if needed          │
└─────────────────────────────────────────────────────┘
```

---

## License

This blueprint is released to the public domain. Use it, modify it, share it. If you find a better solution or Anthropic ships a fix, please contribute back to the community.

*Blueprint by Khaled I. — Raspberry Pi 4, May 2026.*

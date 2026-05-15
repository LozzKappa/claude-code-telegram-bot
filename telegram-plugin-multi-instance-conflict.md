# Claude Code Telegram Plugin — Multi-Instance Conflict: Root Cause Analysis

> \*\*Why this exists:\*\* The Telegram plugin (`telegram@claude-plugins-official`) has no locking mechanism to prevent multiple Claude Code instances from polling the same bot token simultaneously. When more than one instance is active — e.g., a dedicated `--channels` session and any other Claude Code session with the plugin loaded — Telegram returns `409 Conflict` errors, messages are silently dropped, and the bot appears to receive messages (shows "typing") but never replies.
>
> \*\*Important:\*\* This blueprint documents a \*\*workaround architecture\*\* — not an upstream fix. The bug still exists in the plugin code. Anyone using `enabledPlugins` the standard way will still hit it. What this document provides is an architecture that avoids all four root causes by design, so the bug can never trigger. The upstream fix (file lock + SIGTERM handler in the plugin) requires changes from Anthropic.

**Tested on:** Claude Code v2.1.140 · Telegram plugin v0.0.1–v0.0.6 · Debian 13 aarch64 (RPi 4) · May 2026  
**Related issues:** [#39808](https://github.com/anthropics/claude-code/issues/39808) · [#36800](https://github.com/anthropics/claude-code/issues/36800) · [#1075](https://github.com/anthropics/claude-plugins-official/issues/1075) · [#1459](https://github.com/anthropics/claude-plugins-official/issues/1459)

\---

## The Bug

The Telegram plugin uses Grammy's `bot.start()` to poll the Telegram Bot API. Every Claude Code instance that loads the plugin starts its own independent polling loop against the same bot token:

* ✅ The first instance polls successfully and receives messages
* ✅ Typing indicators appear in Telegram
* ❌ A second instance starts polling → Telegram returns `409 Conflict`
* ❌ Grammy retries silently → messages consumed by random instance
* ❌ The `--channels` instance may never see the message → no reply

This happens 100% of the time when any of these are true:

* The plugin is set in `\~/.claude/settings.json` under `enabledPlugins`
* A VSCode/Cursor/JetBrains extension with Claude Code is open alongside a `--channels` session
* Claude Code spawns a duplicate plugin process mid-session (confirmed bug [#36800](https://github.com/anthropics/claude-code/issues/36800))
* A previous bun process survived a session crash and is still polling as an orphan

\---

## Root Causes (4 separate issues)

### RC1 — No polling lock

`bot.start()` fires without checking whether another instance already holds the polling connection. Multiple Grammy loops → `409 Conflict` → message loss.

### RC2 — `allowed\_updates` filter persists across crashes

Telegram persists the `allowed\_updates` parameter from previous `getUpdates` calls. A crashed instance can leave a restrictive filter that silently drops `message` updates for all subsequent instances. This looks identical to RC1 from the outside.

### RC3 — Orphan processes after crash

`bun server.ts` has no `SIGHUP`/`SIGTERM` handler. When Claude Code dies, the bun process becomes orphan (ppid=1) and keeps polling. New sessions cannot acquire the polling slot.

### RC4 — `enabledPlugins` loads plugin in every session

Setting `telegram@claude-plugins-official` in global `\~/.claude/settings.json` causes every Claude Code instance — including non-channels IDE extensions and `claude -c` continuations — to spawn the plugin MCP server, compounding RC1 and RC3.

\---

## How the Fix Works

The solution avoids all four root causes by design:

1. **Never use `enabledPlugins`** for the Telegram plugin — use `mcpServers` scoped to specific agent paths in `\~/.claude.json`
2. **One bot token per agent path** — each agent has its own Telegram bot, its own STATE\_DIR, and its own bun process. No two agents share a token → no `409 Conflict` possible
3. **systemd manages lifecycle** — each agent's bun process is owned by its Claude Code session, started and stopped by systemd. No orphans
4. **File-based inbox workaround** ensures delivery even when Grammy's `notifications/claude/channel` is unreliable (see [claude-code-telegram-bot](https://github.com/LozzKappa/claude-code-telegram-bot))

\---

## Prerequisites

* Linux server (guide written for Debian/Ubuntu)
* [Claude Code CLI](https://www.npmjs.com/package/@anthropic-ai/claude-code) installed
* [bun](https://bun.sh) runtime installed
* One Telegram bot token **per agent** from [@BotFather](https://t.me/botfather)
* `tmux` installed

\---

## Step-by-Step Fix

### 1\. Remove `enabledPlugins` entry for Telegram

Edit `\~/.claude/settings.json` and ensure `enabledPlugins` does **not** contain the Telegram plugin:

```json
{
  "enabledPlugins": {}
}
```

If you leave the Telegram plugin in `enabledPlugins`, every Claude Code session — including your IDE extension — will spawn a bot polling process. Remove it entirely.

### 2\. Kill all orphan bun processes

```bash
pkill -f "bun.\*telegram.\*server.ts" 2>/dev/null
sleep 1
# Confirm no telegram bun processes remain
pgrep -a bun | grep telegram || echo "Clean"
```

### 3\. Optional: Patch `server.ts` for graceful shutdown (RC3 interim fix)

Until Anthropic ships the official fix, you can patch the plugin's `server.ts` locally to handle `SIGTERM`/`SIGHUP`. This prevents orphan processes after a crash:

```typescript
// Add at the top of \~/.claude/plugins/installed/telegram/server.ts
process.on("SIGTERM", () => {
  bot.stop(); // Releases the Grammy polling connection
  process.exit(0);
});

process.on("SIGHUP", () => {
  bot.stop();
  process.exit(0);
});
```

> \*\*Note:\*\* This patch will be overwritten if you reinstall or update the plugin. Re-apply after each update until Anthropic adds it natively.

### 4\. Create one STATE\_DIR per agent

Each agent needs its own directory, its own `.env`, and its own `access.json`:

```bash
# Agent 1 (Primary)
mkdir -p \~/.claude/channels/agent1-telegram
echo "TELEGRAM\_BOT\_TOKEN=<AGENT1\_BOT\_TOKEN>" > \~/.claude/channels/agent1-telegram/.env
chmod 600 \~/.claude/channels/agent1-telegram/.env

cat > \~/.claude/channels/agent1-telegram/access.json << 'EOF'
{"dmPolicy": "allowlist", "allowFrom": \["YOUR\_TELEGRAM\_USER\_ID"], "ackReaction": "👀"}
EOF

# Agent 2
mkdir -p \~/.claude/channels/agent2-telegram
echo "TELEGRAM\_BOT\_TOKEN=<AGENT2\_BOT\_TOKEN>" > \~/.claude/channels/agent2-telegram/.env
chmod 600 \~/.claude/channels/agent2-telegram/.env

cat > \~/.claude/channels/agent2-telegram/access.json << 'EOF'
{"dmPolicy": "allowlist", "allowFrom": \["YOUR\_TELEGRAM\_USER\_ID"], "ackReaction": "🔍"}
EOF
```

### 5\. Configure `\~/.claude.json` with per-path `mcpServers`

```json
{
  "mcpServers": {},
  "projects": {
    "/home/YOUR\_USER/project": {
      "mcpServers": {
        "telegram": {
          "command": "/home/YOUR\_USER/.bun/bin/bun",
          "args": \["/home/YOUR\_USER/.claude/plugins/installed/telegram/server.ts"],
          "env": {
            "TELEGRAM\_BOT\_TOKEN": "<AGENT1\_BOT\_TOKEN>",
            "TELEGRAM\_STATE\_DIR": "/home/YOUR\_USER/.claude/channels/agent1-telegram"
          }
        }
      }
    },
    "/home/YOUR\_USER/project/agent2": {
      "mcpServers": {
        "telegram": {
          "command": "/home/YOUR\_USER/.bun/bin/bun",
          "args": \["/home/YOUR\_USER/.claude/plugins/installed/telegram/server.ts"],
          "env": {
            "TELEGRAM\_BOT\_TOKEN": "<AGENT2\_BOT\_TOKEN>",
            "TELEGRAM\_STATE\_DIR": "/home/YOUR\_USER/.claude/channels/agent2-telegram"
          }
        }
      }
    }
  }
}
```

**Key principle:** The global `mcpServers` key at the top level is empty. Each agent path declares its own entry with its own token and STATE\_DIR. Claude Code only loads the entry that matches the CWD of the session.

> \*\*Security note:\*\* `\~/.claude.json` contains bot tokens in plain text. JSON does not support environment variable expansion, so tokens must be written directly. Protect the file:
> ```bash
> chmod 600 \~/.claude.json
> ```

### 6\. Verify no shared tokens

```bash
# Each bun process should show a different token prefix
for pid in $(pgrep -f "bun.\*telegram/server.ts"); do
  token\_prefix=$(cat /proc/$pid/environ 2>/dev/null | tr '\\0' '\\n' | \\
    grep TELEGRAM\_BOT\_TOKEN | cut -d= -f2 | cut -c1-10)
  state\_dir=$(cat /proc/$pid/environ 2>/dev/null | tr '\\0' '\\n' | \\
    grep TELEGRAM\_STATE\_DIR | cut -d= -f2)
  echo "PID $pid → token: ${token\_prefix}... | dir: $state\_dir"
done
# Expected: every PID shows a DIFFERENT token prefix
```

### 7\. Fix `allowed\_updates` filter if messages still drop after fix

If messages are still silently dropped after eliminating duplicate instances, a stale `allowed\_updates` filter from a previous crash may be blocking `message` events. Reset it:

```bash
# Replace TOKEN with your bot token
curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates" \\
  -d '{"limit":0,"timeout":0,"allowed\_updates":\[]}' | python3 -m json.tool
# Expected: {"ok": true, "result": \[]}
```

### 8\. Optional: Automate RC2 + RC3 cleanup via systemd `ExecStartPre`

If you manage the bun process with systemd, add an `ExecStartPre` directive to automatically reset the `allowed\_updates` filter and kill any orphan process before each start:

```ini
\[Service]
EnvironmentFile=/home/YOUR\_USER/.claude/channels/agent1-telegram/.env

# RC2: reset stale allowed\_updates filter before starting
ExecStartPre=/usr/bin/curl -s "https://api.telegram.org/bot${TELEGRAM\_BOT\_TOKEN}/getUpdates" \\
  -d '{"limit":0,"timeout":0,"allowed\_updates":\[]}'

# RC3: kill any orphan bun process from a previous crash
ExecStartPre=/usr/bin/pkill -f "bun.\*telegram.\*server.ts" ; true

ExecStart=/home/YOUR\_USER/.bun/bin/bun /home/YOUR\_USER/.claude/plugins/installed/telegram/server.ts
```

> \*\*Important:\*\* `${TELEGRAM\_BOT\_TOKEN}` expands only if the token is loaded via `EnvironmentFile=`. Without it the variable stays empty and the curl does nothing.

\---

## Verification

```bash
# 1. Confirm only expected bun processes are running
pgrep -a bun | grep telegram
# Expected: one process per agent, each with a different token

# 2. Confirm no 409 errors in logs (if using systemd)
journalctl --user -u your-agent-1 -n 50 --no-pager | grep -i "409\\|conflict\\|error" || echo "Clean"
journalctl --user -u your-agent-2 -n 50 --no-pager | grep -i "409\\|conflict\\|error" || echo "Clean"

# 3. Send a test message to each bot — both should reply independently
# Bot A → replies only from its configured CWD session
# Bot B → replies only from its configured CWD session
```

\---

## Common Pitfalls

### Pitfall 1 — Plugin still in `enabledPlugins` after fix

**Cause:** Forgot to remove the Telegram entry from `\~/.claude/settings.json`.  
**Symptom:** Every `claude -c` continuation and IDE extension session spawns a new bun polling process.  
**Fix:** Set `"enabledPlugins": {}` and restart all Claude Code sessions.

### Pitfall 2 — Orphan bun process from previous crash

**Cause:** A previous session died without killing the bun child. The orphan (ppid=1) keeps polling.  
**Symptom:** Fresh session gets 409 immediately on start.  
**Fix:** `pkill -f "bun.\*telegram.\*server.ts"` before starting any new session.

### Pitfall 3 — Same token in two `mcpServers` entries

**Cause:** Copy-paste error — two agent paths configured with the same bot token.  
**Symptom:** Identical to the original bug: 409 Conflict, messages dropped.  
**Fix:** Each agent **must** have its own bot token from BotFather.

### Pitfall 4 — `allowed\_updates` filter stuck after crash

**Cause:** A previous instance set a restrictive filter via `allowed\_updates`. Telegram persists it server-side.  
**Symptom:** Bot starts cleanly (no 409), typing indicator shows, but messages still not delivered.  
**Fix:** Run the `getUpdates` reset call in Step 7 above, or automate it with the systemd `ExecStartPre` in Step 8.

### Pitfall 5 — IDE extension loads plugin despite fix

**Cause:** Some IDE extensions (VSCode, Cursor) use their own `settings.json` resolution that may still pick up the plugin.  
**Fix:** Add a project-level `.claude/settings.local.json` in any IDE-opened directory:

```json
{"telegram@claude-plugins-official": false}
```

\---

## When Anthropic Ships the Official Fix

The permanent fix requires changes to the plugin or Claude Code itself:

1. File-based lock at `STATE\_DIR/.lock` (O\_CREAT|O\_EXCL) — only one instance polls
2. `bot.start()` `.catch()` handler — logs the 409 and exits cleanly instead of retrying silently
3. `SIGTERM`/`SIGHUP` handler in `server.ts` — stops Grammy gracefully, releases lock
4. `--channels`-only plugin loading — `enabledPlugins` should not start polling unless the instance has `--channels`

Once those land, you can safely use `enabledPlugins` again and the per-path `mcpServers` workaround can be simplified.

\---

## Architecture: Working Multi-Agent Setup

```
\~/.claude.json
├── projects\["/home/YOUR\_USER/project"]          → mcpServers.telegram → token A → STATE\_DIR/agent1-telegram/
│                                                    ↓ spawns bun (only when CWD = /home/YOUR\_USER/project)
│
└── projects\["/home/YOUR\_USER/project/agent2"]  → mcpServers.telegram → token B → STATE\_DIR/agent2-telegram/
                                                    ↓ spawns bun (only when CWD = /home/YOUR\_USER/project/agent2)

Token A ←→ Bot A (@YourFirstBot)   — no shared polling, no 409
Token B ←→ Bot B (@YourSecondBot)  — no shared polling, no 409

\~/.claude/settings.json
└── enabledPlugins: {}   ← EMPTY — plugin never auto-loaded globally
```

\---

## License

MIT License — Copyright (c) 2026 Khaled I. (LozzKappa). Permission is granted to use, copy, modify and distribute this work, provided that this copyright notice is preserved in all copies.


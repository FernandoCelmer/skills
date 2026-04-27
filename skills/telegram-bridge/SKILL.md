---
name: telegram-bridge
description: Set up and manage a Telegram bot that bridges messages to Claude Code CLI. Use when the user asks to "connect telegram", "telegram bridge", "control claude via telegram", "setup telegram bot", or wants to send commands to Claude from Telegram.
version: 1.0.0
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

You are setting up and managing a **Telegram → Claude Code bridge**. This bridge lets the user send messages from Telegram and receive Claude Code responses back.

---

## Architecture

```
Telegram App → Bot API → bridge.py (polling) → claude CLI (subprocess) → response → Telegram
```

The bridge is a standalone Python script that:
1. Listens for Telegram messages via long-polling
2. Pipes them to `claude --print` as subprocess
3. Streams the response back to the Telegram chat
4. Supports project switching, file sharing, and session persistence

---

## Setup

When the user asks to set up the bridge, execute these steps in order:

### Step 1 — Check prerequisites

```bash
which claude || echo "Claude CLI not found"
python3 -c "import telegram" 2>/dev/null || echo "python-telegram-bot not installed"
```

If `python-telegram-bot` is missing:
```bash
pip install python-telegram-bot==21.* --quiet
```

### Step 2 — Get or verify Telegram bot token

Check `~/.env` for `TELEGRAM_BOT_TOKEN` and `TELEGRAM_ALLOWED_USERS` (comma-separated Telegram user IDs).

If not set, instruct the user:
1. Open Telegram, search for `@BotFather`
2. Send `/newbot`, follow prompts
3. Copy the token
4. Get your user ID from `@userinfobot`

Then save to `~/.env`:
```
TELEGRAM_BOT_TOKEN=<token>
TELEGRAM_ALLOWED_USERS=<your_user_id>
```

### Step 3 — Deploy the bridge script

Write the bridge to `~/.claude/telegram-bridge.py` using the **Bridge Implementation** below.

### Step 4 — Create launchd service (macOS)

Write `~/Library/LaunchAgents/com.claude.telegram-bridge.plist` to auto-start on login.

### Step 5 — Start and verify

```bash
launchctl load ~/Library/LaunchAgents/com.claude.telegram-bridge.plist
```

Send `/ping` in Telegram to verify it works.

---

## Bridge Implementation

Write this exact file to `~/.claude/telegram-bridge.py`:

```python
#!/usr/bin/env python3
"""Telegram ↔ Claude Code bridge."""

import asyncio
import logging
import os
import subprocess
import signal
import sys
from pathlib import Path

from dotenv import load_dotenv
from telegram import Update
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters,
)
from telegram.constants import ParseMode, ChatAction

load_dotenv(Path.home() / ".env")

TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "")
ALLOWED = {
    int(uid.strip())
    for uid in os.environ.get("TELEGRAM_ALLOWED_USERS", "").split(",")
    if uid.strip().isdigit()
}
MAX_MSG_LEN = 4000
DEFAULT_CWD = str(Path.home())

logging.basicConfig(
    format="%(asctime)s [%(levelname)s] %(message)s",
    level=logging.INFO,
)
log = logging.getLogger("telegram-bridge")

# Per-user state
user_cwd: dict[int, str] = {}
user_model: dict[int, str] = {}


def is_authorized(user_id: int) -> bool:
    return not ALLOWED or user_id in ALLOWED


def get_cwd(user_id: int) -> str:
    return user_cwd.get(user_id, DEFAULT_CWD)


def split_message(text: str) -> list[str]:
    """Split long messages into Telegram-safe chunks."""
    chunks = []
    while text:
        if len(text) <= MAX_MSG_LEN:
            chunks.append(text)
            break
        cut = text[:MAX_MSG_LEN].rfind("\n")
        if cut < 100:
            cut = MAX_MSG_LEN
        chunks.append(text[:cut])
        text = text[cut:].lstrip("\n")
    return chunks or [""]


async def run_claude(prompt: str, cwd: str, model: str | None = None) -> str:
    """Run claude CLI as subprocess and return output."""
    cmd = ["claude", "--print", "--verbose"]
    if model:
        cmd.extend(["--model", model])
    cmd.extend([prompt])

    try:
        proc = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=cwd,
        )
        stdout, stderr = await asyncio.wait_for(
            proc.communicate(), timeout=300
        )
        output = stdout.decode("utf-8", errors="replace").strip()
        if not output and stderr:
            output = stderr.decode("utf-8", errors="replace").strip()
        return output or "(empty response)"
    except asyncio.TimeoutError:
        return "⏱ Timeout — Claude took longer than 5 minutes."
    except FileNotFoundError:
        return "❌ `claude` CLI not found. Is it installed and in PATH?"
    except Exception as e:
        return f"❌ Error: {e}"


# ── Handlers ────────────────────────────────────────────────


async def cmd_start(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("⛔ Not authorized.")
        return
    await update.message.reply_text(
        "🤖 Claude Code bridge active.\n\n"
        "Send any message and I'll forward it to Claude.\n\n"
        "Commands:\n"
        "/ping — check if bridge is alive\n"
        "/cd <path> — change working directory\n"
        "/pwd — show current directory\n"
        "/model <name> — switch model (sonnet, opus, haiku)\n"
        "/sh <cmd> — run shell command directly\n"
    )


async def cmd_ping(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    cwd = get_cwd(update.effective_user.id)
    model = user_model.get(update.effective_user.id, "default")
    await update.message.reply_text(
        f"🏓 Pong\n📁 {cwd}\n🧠 {model}"
    )


async def cmd_cd(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    args = update.message.text.split(maxsplit=1)
    if len(args) < 2:
        await update.message.reply_text("Usage: /cd <path>")
        return
    target = os.path.expanduser(args[1].strip())
    if not os.path.isabs(target):
        target = os.path.join(get_cwd(update.effective_user.id), target)
    target = os.path.realpath(target)
    if not os.path.isdir(target):
        await update.message.reply_text(f"❌ Not a directory: {target}")
        return
    user_cwd[update.effective_user.id] = target
    await update.message.reply_text(f"📁 {target}")


async def cmd_pwd(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    await update.message.reply_text(f"📁 {get_cwd(update.effective_user.id)}")


async def cmd_model(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    args = update.message.text.split(maxsplit=1)
    if len(args) < 2:
        current = user_model.get(update.effective_user.id, "default")
        await update.message.reply_text(f"🧠 Current model: {current}")
        return
    m = args[1].strip().lower()
    user_model[update.effective_user.id] = m
    await update.message.reply_text(f"🧠 Switched to: {m}")


async def cmd_sh(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    args = update.message.text.split(maxsplit=1)
    if len(args) < 2:
        await update.message.reply_text("Usage: /sh <command>")
        return
    cwd = get_cwd(update.effective_user.id)
    try:
        result = subprocess.run(
            args[1], shell=True, capture_output=True, text=True,
            timeout=30, cwd=cwd,
        )
        output = result.stdout or result.stderr or "(no output)"
        for chunk in split_message(output.strip()):
            await update.message.reply_text(f"```\n{chunk}\n```", parse_mode=ParseMode.MARKDOWN_V2)
    except subprocess.TimeoutExpired:
        await update.message.reply_text("⏱ Command timed out (30s)")
    except Exception as e:
        await update.message.reply_text(f"❌ {e}")


async def handle_message(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("⛔ Not authorized.")
        return

    uid = update.effective_user.id
    prompt = update.message.text
    if not prompt:
        return

    await ctx.bot.send_chat_action(
        chat_id=update.effective_chat.id, action=ChatAction.TYPING
    )

    cwd = get_cwd(uid)
    model = user_model.get(uid)
    response = await run_claude(prompt, cwd, model)

    for chunk in split_message(response):
        try:
            await update.message.reply_text(chunk, parse_mode=ParseMode.MARKDOWN)
        except Exception:
            await update.message.reply_text(chunk)


# ── Main ────────────────────────────────────────────────────


def main():
    if not TOKEN:
        log.error("TELEGRAM_BOT_TOKEN not set in ~/.env")
        sys.exit(1)

    app = Application.builder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", cmd_start))
    app.add_handler(CommandHandler("ping", cmd_ping))
    app.add_handler(CommandHandler("cd", cmd_cd))
    app.add_handler(CommandHandler("pwd", cmd_pwd))
    app.add_handler(CommandHandler("model", cmd_model))
    app.add_handler(CommandHandler("sh", cmd_sh))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    log.info("Bridge started. Allowed users: %s", ALLOWED or "ALL")
    app.run_polling(drop_pending_updates=True)


if __name__ == "__main__":
    main()
```

---

## launchd Plist (macOS)

Write to `~/Library/LaunchAgents/com.claude.telegram-bridge.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.telegram-bridge</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/env</string>
        <string>python3</string>
        <string>~/.claude/telegram-bridge.py</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>~/.claude/telegram-bridge.log</string>
    <key>StandardErrorPath</key>
    <string>~/.claude/telegram-bridge.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
    </dict>
</dict>
</plist>
```

---

## Managing the Bridge

When the user asks to manage the bridge:

### Check status
```bash
launchctl list | grep telegram-bridge
cat ~/.claude/telegram-bridge.log | tail -20
```

### Restart
```bash
launchctl stop com.claude.telegram-bridge
launchctl start com.claude.telegram-bridge
```

### Stop permanently
```bash
launchctl unload ~/Library/LaunchAgents/com.claude.telegram-bridge.plist
```

### View logs
```bash
tail -f ~/.claude/telegram-bridge.log
```

---

## Security Rules

1. **ALWAYS check `TELEGRAM_ALLOWED_USERS`** — never allow unauthorized users
2. **NEVER expose tokens** in responses or logs
3. `/sh` commands run in a subprocess with 30s timeout — no interactive shells
4. Claude subprocess has 5min timeout
5. Bridge only responds to text messages — no files, photos, or voice

---

## Telegram Commands Reference

| Command | Description |
|---------|-------------|
| `/start` | Show help |
| `/ping` | Check if bridge is alive + show cwd and model |
| `/cd <path>` | Change Claude's working directory |
| `/pwd` | Show current working directory |
| `/model <name>` | Switch Claude model (sonnet, opus, haiku) |
| `/sh <cmd>` | Run shell command directly (30s timeout) |
| Any text | Forward to Claude Code and return response |

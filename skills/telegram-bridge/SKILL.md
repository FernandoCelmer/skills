---
name: telegram-bridge
description: Set up and manage a Telegram bot that bridges messages to Claude Code CLI. Use when the user asks to "connect telegram", "telegram bridge", "control claude via telegram", "setup telegram bot", or wants to send commands to Claude from Telegram.
version: 1.3.0
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
"""Telegram <-> Claude Code bridge."""

import asyncio
import logging
import os
import subprocess
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

ESCAPE_CHARS = r'_*[]()~`>#+-=|{}.!'


def escape_mdv2(text: str) -> str:
    return ''.join(f'\{c}' if c in ESCAPE_CHARS else c for c in text)

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

user_cwd: dict[int, str] = {}
user_model: dict[int, str] = {}
user_session: dict[int, str] = {}
user_proc: dict[int, asyncio.subprocess.Process] = {}


def is_authorized(user_id: int) -> bool:
    return not ALLOWED or user_id in ALLOWED


def get_cwd(user_id: int) -> str:
    return user_cwd.get(user_id, DEFAULT_CWD)


def split_message(text: str) -> list[str]:
    chunks = []
    while text:
        if len(text) <= MAX_MSG_LEN:
            chunks.append(text)
            break
        cut = text[:MAX_MSG_LEN].rfind("
")
        if cut < 100:
            cut = MAX_MSG_LEN
        chunks.append(text[:cut])
        text = text[cut:].lstrip("
")
    return chunks or [""]


async def run_claude(prompt: str, cwd: str, model: str | None = None, session_id: str | None = None, send_fn=None, uid: int | None = None) -> tuple[str, str | None]:
    import json as _json

    cmd = [
        "claude", "--dangerously-skip-permissions",
        "-p", prompt,
        "--output-format", "stream-json", "--verbose",
        "--add-dir", str(Path.home()),
        "--append-system-prompt",
        "IMPORTANT: You are running via Telegram. The user cannot interact with prompts. "
        "Before ANY destructive or irreversible action (git push, file delete, deploy, "
        "database changes, PR merge, sending messages to external services), you MUST: "
        "1) Describe what you plan to do. "
        "2) Stop and wait for the user's next message confirming 'ok', 'sim', 'go', 'yes'. "
        "3) Only then execute. "
        "For read-only actions (git status, file reads, searches) execute immediately. "
        "Keep responses short — this is a mobile screen.",
    ]
    if model:
        cmd.extend(["--model", model])
    if session_id:
        cmd.extend(["--resume", session_id])

    sid = session_id
    result_text = ""

    try:
        proc = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=cwd,
        )
        if uid is not None:
            user_proc[uid] = proc

        async def read_stream():
            nonlocal sid, result_text
            while True:
                line = await asyncio.wait_for(proc.stdout.readline(), timeout=300)
                if not line:
                    break
                raw = line.decode("utf-8", errors="replace").strip()
                if not raw:
                    continue
                try:
                    event = _json.loads(raw)
                except _json.JSONDecodeError:
                    continue

                etype = event.get("type", "")

                if etype == "system" and event.get("subtype") == "init":
                    sid = event.get("session_id", sid)

                elif etype == "assistant":
                    msg = event.get("message", {})
                    for block in msg.get("content", []):
                        if block.get("type") == "tool_use":
                            tool = block.get("name", "?")
                            inp = block.get("input", {})
                            preview = ""
                            if tool == "Bash":
                                preview = inp.get("command", "")[:200]
                            elif tool == "Read":
                                preview = inp.get("file_path", "")
                            elif tool == "Edit":
                                preview = inp.get("file_path", "")
                            elif tool == "Write":
                                preview = inp.get("file_path", "")
                            elif tool == "Grep":
                                preview = inp.get("pattern", "")
                            elif tool == "Glob":
                                preview = inp.get("pattern", "")
                            else:
                                preview = str(inp)[:150]
                            if send_fn and preview:
                                await send_fn(f"[{tool}] {preview}")

                elif etype == "tool_result":
                    content = event.get("content", "")
                    if isinstance(content, list):
                        content = "
".join(
                            b.get("text", "") for b in content if b.get("type") == "text"
                        )
                    if send_fn and content:
                        truncated = content[:1500]
                        if len(content) > 1500:
                            truncated += f"
... ({len(content)} chars)"
                        await send_fn(truncated)

                elif etype == "result":
                    sid = event.get("session_id", sid)
                    result_text = event.get("result", "")

        await read_stream()
        await proc.wait()
        return result_text or "(empty response)", sid

    except asyncio.TimeoutError:
        return "Timeout - Claude took longer than 5 minutes.", sid
    except FileNotFoundError:
        return "claude CLI not found. Is it installed and in PATH?", sid
    except Exception as e:
        return f"Error: {e}", sid


async def cmd_start(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("Not authorized.")
        return
    await update.message.reply_text(
        "Claude Code bridge active.

"
        "Send any message and I'll forward it to Claude.
"
        "Session persists between messages.

"
        "Commands:
"
        "/ping - status
"
        "/stop - kill running task
"
        "/new - start new session
"
        "/cd <path> - change directory
"
        "/pwd - show directory
"
        "/model <name> - switch model
"
        "/sh <cmd> - shell command
"
    )


async def cmd_ping(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    cwd = get_cwd(update.effective_user.id)
    model = user_model.get(update.effective_user.id, "default")
    sid = user_session.get(update.effective_user.id, "none")
    await update.message.reply_text(f"Pong
{cwd}
{model}
session: {sid[:8] if sid != 'none' else 'none'}")


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
        await update.message.reply_text(f"Not a directory: {target}")
        return
    user_cwd[update.effective_user.id] = target
    await update.message.reply_text(f"{target}")


async def cmd_pwd(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    await update.message.reply_text(get_cwd(update.effective_user.id))


async def cmd_model(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    args = update.message.text.split(maxsplit=1)
    if len(args) < 2:
        current = user_model.get(update.effective_user.id, "default")
        await update.message.reply_text(f"Current model: {current}")
        return
    m = args[1].strip().lower()
    user_model[update.effective_user.id] = m
    await update.message.reply_text(f"Switched to: {m}")


async def cmd_new(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    uid = update.effective_user.id
    proc = user_proc.pop(uid, None)
    if proc and proc.returncode is None:
        proc.kill()
    user_session.pop(uid, None)
    await update.message.reply_text("New session started.")


async def cmd_stop(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        return
    uid = update.effective_user.id
    proc = user_proc.get(uid)
    if proc and proc.returncode is None:
        proc.kill()
        await update.message.reply_text("Stopped.")
    else:
        await update.message.reply_text("Nothing running.")


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
            await update.message.reply_text(chunk)
    except subprocess.TimeoutExpired:
        await update.message.reply_text("Command timed out (30s)")
    except Exception as e:
        await update.message.reply_text(f"Error: {e}")


async def handle_image(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("Not authorized.")
        return

    uid = update.effective_user.id
    caption = update.message.caption or "Analyze this image."

    if update.message.photo:
        file = await ctx.bot.get_file(update.message.photo[-1].file_id)
    elif update.message.document:
        file = await ctx.bot.get_file(update.message.document.file_id)
    else:
        return

    img_dir = Path.home() / ".claude" / "telegram-images"
    img_dir.mkdir(exist_ok=True)
    ext = Path(file.file_path).suffix or ".jpg"
    local_path = img_dir / f"{uid}_{file.file_unique_id}{ext}"
    await file.download_to_drive(str(local_path))

    await ctx.bot.send_chat_action(chat_id=update.effective_chat.id, action=ChatAction.TYPING)

    prompt = f"I'm sending you an image at {local_path}. Read it with the Read tool. {caption}"
    cwd = get_cwd(uid)
    model = user_model.get(uid)
    sid = user_session.get(uid)

    async def send_update(text):
        for chunk in split_message(text):
            try:
                await update.message.reply_text(f"```
{chunk}
```", parse_mode=ParseMode.MARKDOWN_V2)
            except Exception:
                await update.message.reply_text(chunk)

    response, new_sid = await run_claude(prompt, cwd, model, session_id=sid, send_fn=send_update, uid=uid)
    if new_sid:
        user_session[uid] = new_sid

    if response:
        for chunk in split_message(response):
            try:
                await update.message.reply_text(escape_mdv2(chunk), parse_mode=ParseMode.MARKDOWN_V2)
            except Exception:
                await update.message.reply_text(chunk)


async def handle_message(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("Not authorized.")
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
    sid = user_session.get(uid)

    async def send_update(text):
        for chunk in split_message(text):
            try:
                await update.message.reply_text(f"```
{chunk}
```", parse_mode=ParseMode.MARKDOWN_V2)
            except Exception:
                await update.message.reply_text(chunk)

    response, new_sid = await run_claude(prompt, cwd, model, session_id=sid, send_fn=send_update, uid=uid)
    if new_sid:
        user_session[uid] = new_sid

    if response:
        for chunk in split_message(response):
            try:
                await update.message.reply_text(escape_mdv2(chunk), parse_mode=ParseMode.MARKDOWN_V2)
            except Exception:
                await update.message.reply_text(chunk)


async def main():
    if not TOKEN:
        log.error("TELEGRAM_BOT_TOKEN not set in ~/.env")
        sys.exit(1)

    app = Application.builder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", cmd_start))
    app.add_handler(CommandHandler("ping", cmd_ping))
    app.add_handler(CommandHandler("cd", cmd_cd))
    app.add_handler(CommandHandler("pwd", cmd_pwd))
    app.add_handler(CommandHandler("model", cmd_model))
    app.add_handler(CommandHandler("new", cmd_new))
    app.add_handler(CommandHandler("stop", cmd_stop))
    app.add_handler(CommandHandler("sh", cmd_sh))
    app.add_handler(MessageHandler(filters.PHOTO | filters.Document.IMAGE, handle_image))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    log.info("Bridge started. Allowed users: %s", ALLOWED or "ALL")

    await app.initialize()
    await app.start()
    await app.updater.start_polling(drop_pending_updates=True, poll_interval=1.0)

    try:
        while True:
            await asyncio.sleep(3600)
    except (KeyboardInterrupt, SystemExit):
        pass
    finally:
        await app.updater.stop()
        await app.stop()
        await app.shutdown()


if __name__ == "__main__":
    asyncio.run(main())

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

## Troubleshooting

### macOS Gatekeeper network prompt

macOS may show a "allow network connections" dialog for Python every time the bridge restarts. Fix permanently:

```bash
sudo xattr -rd com.apple.quarantine $(which python3)
```

For pyenv:
```bash
sudo xattr -rd com.apple.quarantine ~/.pyenv/versions/*/bin/python3
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
| `/new` | Start a new session (reset context) |
| `/sh <cmd>` | Run shell command directly (30s timeout) |
| Any text | Forward to Claude Code (session persists) |

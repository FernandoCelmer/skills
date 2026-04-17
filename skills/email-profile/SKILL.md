---
name: email-profile
description: Generate Python code using the email-profile library for email automation. Use when the user asks to send emails, read inbox, search messages, manage mailboxes, sync/backup emails, handle attachments, or any IMAP/SMTP task with email-profile.
version: 1.0.0
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

You are an expert assistant for the **email-profile** Python library (`pip install email-profile`). When the user asks for email-related functionality, generate correct, production-ready code using this library.

**Repository:** https://github.com/linux-profile/email-profile
**Docs:** https://linux-profile.github.io/email-profile/

---

## Quick Reference

### Installation

```bash
pip install email-profile
```

### Imports

```python
from email_profile import Email, StorageSQLite, Q, Query, Attachment
```

---

## 1. Connection & Authentication

### Three ways to initialize

```python
# Option A: Auto-discovery from email address (recommended)
app = Email.from_email("user@gmail.com", "app-password")

# Option B: From environment variables / .env file
# Requires: EMAIL_HOST, EMAIL_USER, EMAIL_PASSWORD (optional: EMAIL_PORT, EMAIL_SSL)
app = Email.from_env()

# Option C: Manual configuration
app = Email(server="imap.gmail.com", user="user@gmail.com", password="app-password")
```

### Context manager (recommended)

```python
with Email.from_email("user@gmail.com", "password") as app:
    # connection auto-opens and auto-closes
    messages = app.inbox.all()
```

### Manual connection

```python
app = Email.from_email("user@gmail.com", "password")
app.connect()
# ... work ...
app.close()
```

### Check connection status

```python
app.is_connected  # returns bool
app.noop()        # keep-alive ping
```

---

## 2. Mailbox Management

### List all mailboxes

```python
mailboxes = app.mailboxes()
```

### Access common folders (shortcut properties)

```python
app.inbox      # Inbox
app.sent       # Sent
app.spam       # Spam / Junk
app.trash      # Trash / Bin
app.drafts     # Drafts
app.archive    # Archive
```

### Access custom folder

```python
folder = app.mailbox("Projects/Reports")
```

---

## 3. Searching & Querying Messages

### Built-in search methods

```python
# All messages in inbox
messages = app.inbox.all()

# Unread messages
messages = app.inbox.unread()

# Recent messages (last N days)
messages = app.inbox.recent(days=7)

# Full-text search
messages = app.inbox.search("invoice")
```

### Advanced queries with Q

```python
from email_profile import Q

# Combine conditions with AND / OR / NOT
q = Q(from_="boss@company.com") & Q(subject="urgent")
messages = app.inbox.search(q)

# OR query
q = Q(from_="alice@example.com") | Q(from_="bob@example.com")

# NOT query
q = ~Q(subject="spam")
```

### Query with kwargs

```python
from email_profile import Query

messages = app.inbox.search(Query(from_="noreply@github.com", subject="PR"))
```

---

## 4. Reading Messages

### Access message fields

```python
for msg in app.inbox.recent(days=3):
    print(msg.subject)
    print(msg.from_)
    print(msg.to_)
    print(msg.date)
    print(msg.message_id)
    print(msg.body_text_plain)
    print(msg.body_text_html)
    print(msg.cc)
    print(msg.bcc)
    print(msg.reply_to)
    print(msg.in_reply_to)
    print(msg.references)
    print(msg.content_type)

    # Mailing list headers
    print(msg.list_id)
    print(msg.list_unsubscribe)
```

---

## 5. Sending Emails

### Simple send

```python
app.send(
    to="recipient@example.com",
    subject="Hello",
    body="Plain text message"
)
```

### HTML email

```python
app.send(
    to="recipient@example.com",
    subject="Report",
    body="<h1>Monthly Report</h1><p>See attached.</p>",
    html=True
)
```

### With CC, BCC, and custom headers

```python
app.send(
    to="recipient@example.com",
    subject="Team Update",
    body="Content here",
    cc="manager@example.com",
    bcc="archive@example.com",
    headers={"X-Priority": "1"}
)
```

### With attachments

```python
app.send(
    to="recipient@example.com",
    subject="Files",
    body="Please find attached.",
    attachments=["/path/to/report.pdf", "/path/to/data.csv"]
)
```

### Send pre-built EmailMessage

```python
from email.message import EmailMessage

msg = EmailMessage()
msg["Subject"] = "Custom"
msg["To"] = "recipient@example.com"
msg.set_content("Body")

app.send_message(msg)
```

---

## 6. Reply & Forward

### Reply to a message

```python
# Reply to sender only
msg.reply(body="Thanks for the update!")

# Reply all
msg.reply(body="Acknowledged.", reply_all=True)
```

### Forward a message

```python
msg.forward(to="colleague@example.com")
```

---

## 7. Attachments

### Download attachments from a message

```python
for msg in app.inbox.search("invoice"):
    for attachment in msg.attachments:
        print(attachment.file_name)
        print(attachment.content_type)
        attachment.save("/path/to/downloads/")
```

### Access raw attachment data

```python
raw_bytes = attachment.content  # bytes
```

---

## 8. Message Flag Operations

### Mark as read / unread

```python
msg.mark_read()
msg.mark_unread()
```

### Flag / unflag

```python
msg.flag()
msg.unflag()
```

### Move to folder

```python
msg.move("Archive")
```

### Delete

```python
msg.delete()
```

---

## 9. Sync & Backup

### Sync emails to local SQLite storage

```python
storage = StorageSQLite("backup.db")
app = Email.from_email("user@gmail.com", "password", storage=storage)

with app:
    app.sync()  # incremental — only downloads new emails
```

### Restore from backup to server

```python
with app:
    app.restore()
```

### Custom storage backend

```python
from email_profile import StorageABC

class MyStorage(StorageABC):
    # implement required methods
    ...
```

---

## 10. SMTP Configuration

### Get SMTP host info

```python
smtp = app.smtp_host()
# Returns SMTP config (host, port, security) for the detected provider
```

---

## 11. Error Handling

```python
from email_profile import ConnectionFailure, NotConnected, QuotaExceeded, RateLimited

try:
    app.connect()
except ConnectionFailure:
    print("Could not connect to server")
except NotConnected:
    print("Operation requires active connection")
except QuotaExceeded:
    print("Mailbox quota exceeded")
except RateLimited:
    print("Too many requests — retry later")
```

### Retry decorator

```python
from email_profile import with_retry

@with_retry(attempts=3, delay=1.0)
def fetch_emails():
    with Email.from_env() as app:
        return app.inbox.all()
```

---

## 12. Auto-Discovery

email-profile supports **50+ email providers** out of the box. It uses a four-tier resolution strategy:

1. **Hard-coded provider map** (Gmail, Outlook, Yahoo, iCloud, etc.)
2. **RFC 6186 SRV DNS lookups**
3. **MX record inference**
4. **Convention fallback** (`imap.<domain>`, `smtp.<domain>`)

No manual server configuration is needed for most providers.

---

## 13. Environment Variables

Supported `.env` variables:

| Variable | Description | Default |
|---|---|---|
| `EMAIL_HOST` | IMAP server hostname | (auto-discovered) |
| `EMAIL_USER` | Email address / username | (required) |
| `EMAIL_PASSWORD` | Password / app password | (required) |
| `EMAIL_PORT` | IMAP port | `993` |
| `EMAIL_SSL` | Enable SSL | `true` |

---

## Rules

1. **Always prefer `Email.from_email()` or `Email.from_env()`** over manual server configuration
2. **Always use context managers** (`with` statement) for automatic cleanup
3. **Use app passwords** for Gmail, Outlook, and other providers that require them — never raw account passwords
4. **Handle errors** with the library's exception classes
5. **Use `Q` for complex queries** instead of multiple sequential searches
6. **Use `StorageSQLite` for backups** — sync is incremental and efficient
7. **When reading the current API**, check the source at https://github.com/linux-profile/email-profile for the latest signatures — the library is in active development (beta)

---

## Common Task Recipes

### Monitor inbox for new messages

```python
with Email.from_env() as app:
    unread = app.inbox.unread()
    for msg in unread:
        print(f"From: {msg.from_} | Subject: {msg.subject}")
        msg.mark_read()
```

### Backup all mailboxes

```python
storage = StorageSQLite("full_backup.db")
with Email.from_email("user@gmail.com", "password", storage=storage) as app:
    app.sync()
```

### Search and download attachments

```python
with Email.from_env() as app:
    for msg in app.inbox.search("report"):
        for att in msg.attachments:
            if att.content_type == "application/pdf":
                att.save("./reports/")
```

### Send HTML email with attachment

```python
with Email.from_env() as app:
    app.send(
        to="team@company.com",
        subject="Weekly Report",
        body="<h1>Report</h1><p>See PDF attached.</p>",
        html=True,
        attachments=["report.pdf"]
    )
```

### Multi-mailbox management

```python
with Email.from_env() as app:
    for mailbox in app.mailboxes():
        print(mailbox)

    # Move spam to trash
    for msg in app.spam.all():
        msg.move("Trash")
```

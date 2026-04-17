---
name: email-profile
description: Generate Python code using the email-profile library for email automation. Use when the user asks to send emails, read inbox, search messages, manage mailboxes, sync/backup emails, handle attachments, or any IMAP/SMTP task with email-profile.
version: 1.1.0
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
from email_profile import Email, StorageSQLite, Q, Query, Message
```

---

## 1. Connection & Authentication

### Three ways to initialize

```python
# Option A: Auto-discovery from email address (recommended)
app = Email.from_email("user@gmail.com", "app-password")

# Option B: From environment variables / .env file
# Requires: EMAIL_SERVER, EMAIL_USERNAME, EMAIL_PASSWORD
app = Email.from_env()

# Option C: Manual configuration
app = Email(server="imap.gmail.com", user="user@gmail.com", password="app-password")

# Option D: Positional shorthand (auto-discovers server from address)
app = Email("user@gmail.com", "app-password")
```

### Context manager (recommended)

```python
with Email.from_email("user@gmail.com", "password") as app:
    # connection auto-opens and auto-closes
    messages = app.inbox.where().list()
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

### Using `.where()` on mailbox

The `.where()` method returns a lazy `Where` object. Call `.messages()`, `.list()`, `.first()`, `.last()`, `.count()`, or `.exists()` to materialize results.

```python
# All messages in inbox
results = app.inbox.where()
messages = results.list()         # materialized list
count = results.count()           # int
first_msg = results.first()       # single Message or None

# Unread messages
messages = app.inbox.where(unseen=True).list()

# Iterate lazily
for msg in app.inbox.where().messages():
    print(msg.subject)
```

### Shortcut methods on Email object

```python
# These return Where objects operating on inbox
app.all()              # all inbox messages
app.unread()           # unread inbox messages
app.recent(days=7)     # messages from last N days
app.search("invoice")  # full-text search in inbox
```

### Fetch modes

```python
# Full message with attachments (default)
for msg in app.inbox.where().messages(mode="full"):
    ...

# Headers + body, no attachments (faster)
for msg in app.inbox.where().messages(mode="text"):
    ...

# Headers only (cheapest)
for msg in app.inbox.where().messages(mode="headers"):
    ...
```

### Advanced queries with Q (static methods)

```python
from email_profile import Q

# Combine conditions with & (AND), | (OR), ~ (NOT)
q = Q.from_("boss@company.com") & Q.subject("urgent")
messages = app.inbox.where(q).list()

# OR query
q = Q.from_("alice@example.com") | Q.from_("bob@example.com")

# NOT query
q = ~Q.subject("spam")

# Other Q methods
Q.unseen()
Q.seen()
Q.since(date(2025, 1, 1))
Q.before(date(2025, 6, 1))
Q.larger(1_000_000)
```

### Query with kwargs

```python
from email_profile import Query

q = Query(subject="PR", unseen=True, since=date(2025, 1, 1))

# Use from_ or from_who for sender filter
q = Query(from_who="noreply@github.com", subject="PR")

# Chaining
q = Query(subject="report").exclude(from_who="spam@example.com").or_(subject="invoice")

messages = app.inbox.where(q).list()
```

---

## 4. Reading Messages

### Access message fields

Message is a Pydantic model with these fields:

```python
for msg in app.inbox.where(unseen=True).messages():
    print(msg.subject)
    print(msg.from_)
    print(msg.to_)
    print(msg.date)
    print(msg.uid)
    print(msg.message_id)
    print(msg.body_text_plain)
    print(msg.body_text_html)
    print(msg.cc)
    print(msg.bcc)
    print(msg.reply_to)
    print(msg.in_reply_to)
    print(msg.references)
    print(msg.content_type)
    print(msg.attachments)
    print(msg.headers)

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
    html="<h1>Monthly Report</h1><p>See attached.</p>"
)
```

### Plain text + HTML (multipart)

```python
app.send(
    to="recipient@example.com",
    subject="Report",
    body="Monthly Report - See attached.",
    html="<h1>Monthly Report</h1><p>See attached.</p>"
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

### Save to sent folder

```python
app.send(
    to="recipient@example.com",
    subject="Hello",
    body="Content",
    save_to_sent=True  # default is True
)
```

---

## 6. Reply & Forward

Reply and forward are methods on the **Email object**, not on Message:

### Reply to a message

```python
# Reply to sender only
app.reply(msg, body="Thanks for the update!")

# Reply all
app.reply(msg, body="Acknowledged.", reply_all=True)
```

### Forward a message

```python
app.forward(msg, to="colleague@example.com", body="FYI")
```

---

## 7. Attachments

### Download attachments from a message

```python
for msg in app.inbox.where(Q.subject("invoice")).messages():
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

Flag operations are methods on the **mailbox**, not on the message. They accept a Message, UID string, or int:

### Mark as seen / unseen

```python
app.inbox.mark_seen(msg)
app.inbox.mark_unseen(msg)
```

### Flag / unflag

```python
app.inbox.flag(msg)
app.inbox.unflag(msg)
```

### Move to folder

```python
app.inbox.move(msg, "Archive")
```

### Copy to folder

```python
app.inbox.copy(msg, "Backup")
```

### Delete

```python
app.inbox.delete(msg)
```

---

## 9. Sync & Backup

### Sync emails to local SQLite storage

```python
app = Email.from_email("user@gmail.com", "password")
app.storage = StorageSQLite("backup.db")

with app:
    result = app.sync()  # incremental — only downloads new emails
    print(f"{result.inserted} new, {result.skipped} skipped")
```

### Restore from backup to server

```python
with app:
    count = app.restore()  # returns int (count of restored messages)
    print(f"Restored {count} messages")
```

### Storage via constructor

```python
storage = StorageSQLite("backup.db")
app = Email(server="imap.gmail.com", user="user@gmail.com", password="pw", storage=storage)
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
from email_profile.retry import with_retry

@with_retry(attempts=3, initial_delay=1.0, max_delay=30.0)
def fetch_emails():
    with Email.from_env() as app:
        return app.inbox.where().list()
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

Supported `.env` variables for `Email.from_env()`:

| Variable | Description | Default |
|---|---|---|
| `EMAIL_SERVER` | IMAP server hostname | (auto-discovered) |
| `EMAIL_USERNAME` | Email address / username | (required) |
| `EMAIL_PASSWORD` | Password / app password | (required) |

Custom env var names can be passed to `from_env()`:

```python
app = Email.from_env(
    server_var="MY_EMAIL_SERVER",
    user_var="MY_EMAIL_USER",
    password_var="MY_EMAIL_PASSWORD"
)
```

---

## Rules

1. **Always prefer `Email.from_email()` or `Email.from_env()`** over manual server configuration
2. **Always use context managers** (`with` statement) for automatic cleanup
3. **Use app passwords** for Gmail, Outlook, and other providers that require them — never raw account passwords
4. **Handle errors** with the library's exception classes
5. **Use `Q` static methods for complex queries** — combine with `&`, `|`, `~` operators
6. **Use `StorageSQLite` for backups** — sync is incremental and efficient
7. **Flag operations go on the mailbox**, not the message — e.g., `app.inbox.mark_seen(msg)`
8. **Reply/forward go on the Email object** — e.g., `app.reply(msg, body="...")`
9. **`.where()` returns a lazy `Where` object** — call `.list()`, `.messages()`, `.first()`, `.count()` to get results
10. **When reading the current API**, check the source at https://github.com/linux-profile/email-profile for the latest signatures — the library is in active development (beta)

---

## Common Task Recipes

### Monitor inbox for new messages

```python
with Email.from_env() as app:
    for msg in app.unread().messages():
        print(f"From: {msg.from_} | Subject: {msg.subject}")
        app.inbox.mark_seen(msg)
```

### Backup all mailboxes

```python
app = Email.from_email("user@gmail.com", "password")
app.storage = StorageSQLite("full_backup.db")

with app:
    result = app.sync()
    print(f"{result.inserted} new, {result.skipped} skipped")
```

### Search and download attachments

```python
with Email.from_env() as app:
    for msg in app.search("report").messages():
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
        html="<h1>Report</h1><p>See PDF attached.</p>",
        attachments=["report.pdf"]
    )
```

### Multi-mailbox management

```python
with Email.from_env() as app:
    for mailbox in app.mailboxes():
        print(mailbox)

    # Move spam to trash
    for msg in app.spam.where().messages():
        app.spam.move(msg, "Trash")
```

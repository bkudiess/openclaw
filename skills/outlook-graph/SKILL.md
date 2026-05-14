---
name: outlook-graph
description: Headless Outlook automation on Windows via the Microsoft Graph CLI (`mgc`) — mail, calendar, and contacts. List/search inbox messages, get message bodies, send mail with attachments, list/create calendar events, list contacts. Use when the user says "send an email to X", "what's on my calendar tomorrow", "search my inbox for ...", "schedule a meeting", "find contact for ..." — and the workflow is server-side / scriptable. Companion to `outlook-com` — pick **outlook-graph** for headless / cloud / cross-machine work and **outlook-com** when you need the desktop Outlook UI, drafts in the running session, or category/rule edits that the COM object exposes.
metadata:
  openclaw:
    emoji: "📧"
    os: ["win32"]
    requires:
      bins: ["pwsh", "mgc"]
    install:
      - id: "mgc-download"
        kind: "download"
        os: ["win32"]
        url: "https://github.com/microsoftgraph/msgraph-cli/releases/download/v1.9.0/msgraph-cli-win-x64-1.9.0.zip"
        archive: "zip"
        extract: true
        targetDir: "mgc"
        bins: ["mgc"]
        label: "Download Microsoft Graph CLI (mgc) for Windows"
---

# Outlook via Microsoft Graph CLI

Use **`mgc`** to drive Outlook **mail, calendar, and contacts** through Microsoft Graph. Works from any shell, no Outlook desktop required, ideal for cron-style scripts and remote sessions.

## When to Use

✅ **USE this skill when:**

- User says "send an email to ..." / "draft a reply to ..."
- User says "what's on my calendar tomorrow" / "schedule a meeting"
- User says "search my inbox for ..." / "list unread mail"
- Workflow needs to run **without** the Outlook desktop client (CI, SSH, scheduled task)
- User wants raw JSON for piping into other tools

## When NOT to Use

❌ **DON'T use this skill when:**

- User needs to interact with the running Outlook **desktop UI** (open compose window, edit a draft visible to them) → `outlook-com`
- User wants Outlook **rules / categories / shared mailbox** edits that aren't first-class in Graph → `outlook-com`
- Account is Gmail/IMAP, not Microsoft 365 → use `himalaya` or the IMAP path of the relevant skill
- The script runs as the user but the user is on **macOS** → consider `imsg` (no Outlook on Apple Mail) or generic IMAP tools

## Setup

`mgc` is the official Microsoft Graph CLI. There is **no winget package**. Install from GitHub releases:

```powershell
$dst = "$env:LOCALAPPDATA\Programs\msgraph-cli"
New-Item -ItemType Directory -Force -Path $dst | Out-Null
$zip = "$env:TEMP\msgraph-cli-win.zip"
Invoke-WebRequest `
  "https://github.com/microsoftgraph/msgraph-cli/releases/download/v1.9.0/msgraph-cli-win-x64-1.9.0.zip" `
  -OutFile $zip
Expand-Archive $zip -DestinationPath $dst -Force
$env:Path = "$dst;$env:Path"
mgc --version
```

### Sign in (interactive, opens a browser)

```powershell
mgc login --scopes "Mail.ReadWrite Mail.Send Calendars.ReadWrite Contacts.Read User.Read"
```

- For send-only flows, scope down to `Mail.Send`.
- Tenant-specific: `--tenant-id <guid|domain>`. Multi-account: re-run `mgc login` per account (only one cached token at a time).
- Token cache: MSAL stores tokens in a DPAPI-encrypted file under `%LOCALAPPDATA%\.IdentityService\` (Windows). `mgc logout` clears it.
- Re-login is required to add scopes — pass the **full** set every time.

## Mail

### List inbox messages

```powershell
mgc users mail-folders messages list --user-id me `
  --mail-folder-id "inbox" `
  --select id,subject,from,receivedDateTime,isRead,hasAttachments `
  --orderby "receivedDateTime desc" --top 25
```

Well-known folder ids: `inbox`, `drafts`, `sentitems`, `deleteditems`, `archive`, `outbox`, `junkemail`, `clutter`. Custom folders need a real id from `mgc users mail-folders list --user-id me`.

### Get a single message (body + headers)

```powershell
mgc users messages get --user-id me --message-id $messageId `
  --select id,subject,from,toRecipients,body,internetMessageHeaders
```

Use `--output-file message.eml` plus `Prefer: outlook.body-content-type="text"` (via `--headers`) to get plain text bodies instead of HTML.

### Search inbox

Graph supports `$search` with KQL-style operators (subject, from, body, hasAttachments, received). Use one — `$search` and `$filter` are mutually exclusive on the same request.

```powershell
# Subject + sender
mgc users messages list --user-id me `
  --search '"subject:invoice AND from:billing@acme.com"' --top 25

# Last 7 days unread, structured filter (no --search)
mgc users mail-folders messages list --user-id me --mail-folder-id "inbox" `
  --filter "isRead eq false and receivedDateTime ge $(((Get-Date).AddDays(-7)).ToString('o'))" `
  --orderby "receivedDateTime desc"
```

### Send mail (destructive — see Safety Rules)

```powershell
$mail = @'
{
  "message": {
    "subject": "Status update",
    "body": { "contentType": "Text", "content": "Shipping on Friday. — me" },
    "toRecipients": [
      { "emailAddress": { "address": "alice@example.com" } }
    ],
    "ccRecipients": [
      { "emailAddress": { "address": "bob@example.com" } }
    ]
  },
  "saveToSentItems": true
}
'@
$mail | Out-File -Encoding utf8 mail.json

mgc users send-mail post --user-id me --body "@mail.json"
```

- `contentType: "HTML"` lets you send rich bodies.
- Inline attachments use `"attachments": [{ "@odata.type": "#microsoft.graph.fileAttachment", "name": "report.pdf", "contentBytes": "<base64>" }]`.
- For attachments **>3 MB**, create a draft, then upload chunks via `createUploadSession` — not a one-liner.

### Reply to a message

```powershell
'{ "comment": "Confirmed — see you then." }' | Out-File -Encoding utf8 reply.json
mgc users messages reply post --user-id me --message-id $messageId --body "@reply.json"
```

`reply-all post` exists too. Both send immediately; use `create-reply` / `create-reply-all` first if you need to inspect or hold the draft.

## Calendar

### Events in a date range

`calendar-view` is the right endpoint for time-bounded queries — expands recurring instances automatically.

```powershell
$start = (Get-Date).ToString('yyyy-MM-ddTHH:mm:ss')
$end   = (Get-Date).AddDays(7).ToString('yyyy-MM-ddTHH:mm:ss')

mgc users calendar-view list --user-id me `
  --start-date-time $start --end-date-time $end `
  --select subject,start,end,location,organizer,isAllDay `
  --orderby "start/dateTime" `
  --headers "Prefer=outlook.timezone=`"Pacific Standard Time`""
```

`events list` is fine for "next 50 future events" but won't expand recurrences.

### Create an event

```powershell
$evt = @'
{
  "subject": "1:1 with Alice",
  "start": { "dateTime": "2026-02-20T10:00:00", "timeZone": "Pacific Standard Time" },
  "end":   { "dateTime": "2026-02-20T10:30:00", "timeZone": "Pacific Standard Time" },
  "attendees": [
    { "emailAddress": { "address": "alice@example.com", "name": "Alice" }, "type": "required" }
  ],
  "body": { "contentType": "Text", "content": "Quarterly plan review." },
  "isOnlineMeeting": true,
  "onlineMeetingProvider": "teamsForBusiness"
}
'@
$evt | Out-File -Encoding utf8 event.json
mgc users events create --user-id me --body "@event.json"
```

### Cancel or delete an event

```powershell
# Organizer-only: cancel sends a notice to attendees
'{ "comment": "Rescheduling — new invite soon." }' | Out-File -Encoding utf8 cancel.json
mgc users events cancel post --user-id me --event-id $eventId --body "@cancel.json"

# Outright delete (no cancellation email)
mgc users events delete --user-id me --event-id $eventId
```

## Contacts

```powershell
# List
mgc users contacts list --user-id me --top 50 `
  --select displayName,emailAddresses,businessPhones,companyName

# Search by name
mgc users contacts list --user-id me `
  --filter "startswith(displayName,'Smith')" --top 25
```

The People API (`mgc users people list`) is often more useful for "who do I email a lot named Bob" — it ranks across mail/calendar/Teams.

## JSON tips

- Default output is JSON; pipe to `ConvertFrom-Json`.
- `--query "value[?isRead==``false``].subject"` for server-ish JMESPath.
- `--all` auto-follows `@odata.nextLink`.

## Safety Rules

1. **`send-mail post`, `messages reply post`, and `messages reply-all post` all send immediately** — no draft staging step. **ALWAYS** echo the resolved To/Cc/subject/body back to the user and require explicit confirmation before invoking any of these.
2. **Never send to addresses the user didn't name explicitly.** When using contacts/people lookup to resolve a name, surface the exact email address before sending.
3. **Avoid mass mailings.** If the `toRecipients` array has more than ~5 entries, pause and confirm intent.
4. **Don't `reply-all` without explicit user direction** — the user may have meant `reply`.
5. **`events delete` skips cancellation notices.** Prefer `events cancel post` when other attendees exist (verify the subcommand shape with `mgc users events cancel -h` on first use — the action verb has moved between mgc releases).
6. **Bulk inbox actions** (mark-all-read, delete-all in a folder) require a typed confirmation from the user — read the count back first:
   ```powershell
   mgc users mail-folders messages list --user-id me --mail-folder-id "junkemail" --count
   ```
7. **Don't echo full message bodies into chat** if subject lines suggest HR/legal/financial content.
8. **Respect throttling**. Graph returns `429` under load; back off the `Retry-After` value.

## Notes

- **HTML vs Text bodies.** Default is HTML; flip with the `Prefer: outlook.body-content-type="text"` request header (`--headers`).
- **Time zones.** Request `Prefer: outlook.timezone="Pacific Standard Time"` to get all `dateTime` fields back in your local TZ; otherwise Graph returns UTC.
- **Drafts.** `messages create` + `--body` makes a draft in `drafts`. `messages send post --user-id me --message-id $id` sends a previously-created draft.
- **Shared mailboxes.** Replace `--user-id me` with `--user-id sharedbox@contoso.com` after delegated permissions are granted.
- **Personal Microsoft accounts** (`outlook.com`) work for mail and calendar but **not** for `Calendars.ReadWrite.Shared` or many `Contacts` extensions.

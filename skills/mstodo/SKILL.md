---
name: mstodo
description: Manage Microsoft To Do tasks on Windows via the Microsoft Graph CLI (`mgc`). List task lists, list/create/update/complete/delete tasks, set due dates and reminders, and turn flagged Outlook emails into tasks. Use when the user says "add a to-do", "what's on my Microsoft To Do list", "mark X done in To Do", "list my Outlook tasks", "remind me to ...", or wants to script their Microsoft 365 task inbox. This is the Windows / Microsoft 365 equivalent of the macOS `apple-reminders` and `things-mac` skills — pick this one when the user is on a work/school account or specifically names "Microsoft To Do", "Outlook Tasks", or "Tasks in Teams".
metadata:
  openclaw:
    emoji: "✅"
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

# Microsoft To Do (via Microsoft Graph CLI)

Use the **Microsoft Graph CLI** (`mgc`) to read and modify Microsoft To Do tasks. To Do is backed by the same `outlookTask` store as Outlook Tasks, so this skill also covers "my Outlook tasks" and "Tasks in Teams".

## When to Use

✅ **USE this skill when:**

- User says "add a to-do", "remind me to ...", "what's on my Microsoft To Do list"
- User says "mark this task done in To Do" / "complete the X task"
- User wants tasks pulled from a Microsoft 365 work or school account
- User says "list my Outlook tasks" or "tasks in Teams"

## When NOT to Use

❌ **DON'T use this skill when:**

- User is clearly on macOS Reminders → use the macOS `apple-reminders` skill
- User wants Things 3 → `things-mac`
- User wants Apple Notes (notes, not tasks) → `apple-notes` or `onenote`
- User wants OneNote pages → `onenote` skill
- User wants Outlook **mail** or **calendar** → `outlook-graph`
- The account is a personal Hotmail/Outlook.com without To Do enabled — degrade gracefully and tell the user

## Setup

`mgc` is the official Microsoft Graph CLI. There is **no winget package** today — install from the GitHub release ZIP:

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

To put `mgc` on PATH permanently, add `$env:LOCALAPPDATA\Programs\msgraph-cli` via *System Properties → Environment Variables → User → Path*.

### Sign in (interactive, **opens a browser**)

```powershell
mgc login --scopes "Tasks.ReadWrite User.Read"
```

- The default `--strategy DeviceCode` prints a code to paste at `https://microsoft.com/devicelogin`. Use `--strategy InteractiveBrowser` to skip the device-code dance on a desktop session.
- Multi-tenant accounts: add `--tenant-id <guid-or-domain>` to pin to a specific tenant. `--client-id <guid>` overrides the default first-party client.
- Token cache: MSAL stores tokens in a DPAPI-encrypted file under `%LOCALAPPDATA%\.IdentityService\` (Windows). `mgc logout` deletes the cache. The cache is **not** in Windows Credential Manager unless WAM is wired up.
- Re-run `mgc login` whenever you need additional scopes — scopes are **not** additive across logins; pass the full set every time.

### Verify the session

```powershell
mgc users get --user-id me --select id,displayName,userPrincipalName
```

## Discovering IDs

To Do tasks live under **task lists**. You need a `--todo-task-list-id` for almost every command. The default list is named "Tasks" and has a stable well-known id `tasks` — but resolve by `displayName` to be safe:

```powershell
# All task lists (default output is JSON)
mgc users todo lists list --user-id me --select id,displayName

# Resolve a list id by name
$listId = (mgc users todo lists list --user-id me `
  --filter "displayName eq 'Groceries'" --select id `
  | ConvertFrom-Json).value[0].id
```

> All `mgc` commands use the canonical form `mgc users <subcommand> --user-id me` — the top-level `me` alias is undocumented across mgc versions, so the `users` form is what the skill commits to.

## Common Workflows

### List tasks in a list

```powershell
mgc users todo lists tasks list `
  --user-id me `
  --todo-task-list-id $listId `
  --select id,title,status,dueDateTime,importance `
  --top 50
```

Filter to active items only:

```powershell
mgc users todo lists tasks list --user-id me --todo-task-list-id $listId `
  --filter "status ne 'completed'" --orderby "createdDateTime desc"
```

### Create a task

`mgc` POSTs whatever JSON you supply via `--body`. Use a heredoc-style here-string in PowerShell:

```powershell
$body = @'
{
  "title": "Buy bagels",
  "importance": "high",
  "body": { "content": "From the place on Main St", "contentType": "text" },
  "dueDateTime": { "dateTime": "2026-02-14T17:00:00", "timeZone": "Pacific Standard Time" },
  "reminderDateTime": { "dateTime": "2026-02-14T15:00:00", "timeZone": "Pacific Standard Time" },
  "isReminderOn": true
}
'@
$body | Out-File -Encoding utf8 task.json
mgc users todo lists tasks create --user-id me `
  --todo-task-list-id $listId --body "@task.json"
```

- `--body "@path"` reads JSON from a file. Inline strings also work but quoting on Windows is painful — prefer files.
- `dueDateTime.timeZone` is an IANA-style **or** Windows time zone name. `Get-TimeZone` lists valid Windows names.

### Mark a task complete

A PATCH that sets `status` to `completed`:

```powershell
'{ "status": "completed" }' | Out-File -Encoding utf8 patch.json
mgc users todo lists tasks patch --user-id me `
  --todo-task-list-id $listId --todo-task-id $taskId `
  --body "@patch.json"
```

Valid status values: `notStarted`, `inProgress`, `completed`, `waitingOnOthers`, `deferred`.

### Set or change a due date

```powershell
$patch = @'
{ "dueDateTime": { "dateTime": "2026-03-01T09:00:00", "timeZone": "Pacific Standard Time" } }
'@
$patch | Out-File -Encoding utf8 patch.json
mgc users todo lists tasks patch --user-id me `
  --todo-task-list-id $listId --todo-task-id $taskId --body "@patch.json"
```

To **clear** a due date, PATCH with `"dueDateTime": null`.

### Delete a task (destructive — see Safety Rules)

```powershell
mgc users todo lists tasks delete --user-id me `
  --todo-task-list-id $listId --todo-task-id $taskId
```

`delete` accepts `--if-match <etag>` for optimistic concurrency — pass the `@odata.etag` from the prior GET if you want delete-only-if-unchanged semantics.

### Create a new task list

```powershell
'{ "displayName": "Errands" }' | Out-File -Encoding utf8 list.json
mgc users todo lists create --user-id me --body "@list.json"
```

### Flagged Outlook emails as tasks

Flagging an email in Outlook automatically creates an entry in the **Flagged Emails** task list. To list them:

```powershell
$flaggedListId = (mgc users todo lists list --user-id me `
  --filter "wellknownListName eq 'flaggedEmails'" --select id `
  | ConvertFrom-Json).value[0].id

mgc users todo lists tasks list --user-id me `
  --todo-task-list-id $flaggedListId `
  --select id,title,status,linkedResources --top 50
```

Each flagged-email task has a `linkedResources` collection with the source `messageId` — feed that into the `outlook-graph` skill to read the email body.

### JSON output and scripting

- `mgc` defaults to **JSON** output already. `ConvertFrom-Json` is the natural Windows partner.
- `--output TABLE` gives a quick human-readable view; `--output RAW_JSON` returns the unflattened Graph response (use this when you need the `@odata.etag` or `@odata.nextLink`).
- `--query` accepts a JMESPath expression evaluated server-side-ish:
  ```powershell
  mgc users todo lists tasks list --user-id me --todo-task-list-id $listId `
    --query "value[?status!='completed'].{title:title,due:dueDateTime.dateTime}"
  ```
- Use `--all` to auto-page through every `@odata.nextLink` instead of one page of `--top`.

## Safety Rules

1. **Always confirm deletes**. `mgc users todo lists tasks delete` is permanent — there is no Recycle Bin. Read the task title back to the user before deleting.
2. **Prefer marking complete over deleting**. `status = completed` keeps history and is reversible.
3. **Never bulk-delete a whole list without explicit user approval**. If asked to "clear my Groceries list", confirm the exact list name and count first:
   ```powershell
   mgc users todo lists tasks list --user-id me --todo-task-list-id $listId --count
   ```
4. **Don't echo task body text into chat** if the user has flagged work-confidential items — Graph tasks can contain meeting notes, customer data, or HR content.
5. **Respect throttling**. Graph returns HTTP `429` with a `Retry-After` header under heavy use. `mgc` surfaces this as a non-zero exit code; back off the suggested seconds before retrying.

## Notes

- **Account scope.** Token + cache are per-Windows-user. A different Windows account = a different `mgc login`.
- **Conditional Access** (CA) policies on work tenants can require MFA on every login, or block device-code flow entirely. If `mgc login` returns `AADSTS50158` / `AADSTS530003`, switch to `--strategy InteractiveBrowser` or have the user complete the CA prompt manually.
- **Tasks API only.** This skill does not cover the legacy `outlookTask` endpoints (deprecated) or Planner (`mgc planner ...`) — those are separate Graph surfaces.
- **Rate limits.** Graph throttles per app + per user; expect ~10k requests / 10 minutes for first-party clients. Use `--all` sparingly.
- **No offline mode.** `mgc` always hits `graph.microsoft.com`; expect failures behind strict egress firewalls. Set `HTTPS_PROXY` if needed.

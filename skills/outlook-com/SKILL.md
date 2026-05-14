---
name: outlook-com
description: Automate the Microsoft Outlook desktop app on Windows via COM (Outlook.Application). Read inbox/sent/drafts/calendar/contacts, search mail with Restrict, send mail, create calendar events, list appointments, and enumerate contacts. Use when the user asks to "read my Outlook inbox", "search Outlook for X", "send mail from Outlook", "add a calendar event", "list today's meetings", "draft an Outlook reply", or any Outlook desktop automation on Windows. Requires Microsoft 365 / Outlook desktop installed and signed in.
metadata:
  openclaw:
    emoji: "📧"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
    install:
      - id: "office"
        kind: "winget"
        os: ["win32"]
        packageId: "Microsoft.Office"
        bins: ["OUTLOOK"]
        label: "Install Microsoft 365 (winget)"
---

# Outlook COM Automation

Drive the **Outlook desktop app** on Windows through its COM automation interface
(`Outlook.Application`). Outlook must be installed and signed in. The first call
launches `OUTLOOK.EXE` in the background; subsequent calls attach to the running
instance.

## When to Use

✅ **USE this skill when:**

- Reading or searching the user's Outlook inbox / sent / drafts / deleted items
- Sending mail through the user's actual Outlook profile (uses their identity, signatures, From account)
- Reading or creating calendar events / appointments on the default calendar
- Enumerating Outlook contacts
- Drafting replies that should appear in the user's Outlook Drafts folder

## When NOT to Use

❌ **DON'T use this skill when:**

- The user is **headless / no Outlook installed** → use Microsoft Graph instead (works without desktop Outlook)
- Outlook is not running and the box has no licensed profile → COM cannot create a session
- The user wants **server-side** automation (services, scheduled tasks under SYSTEM) → COM is per-interactive-user; use Graph
- The mailbox is on a different account than the signed-in Outlook profile → use Graph with explicit auth
- macOS Mail or Apple Calendar → use `apple-mail` / `apple-reminders` style skills

## Setup

- Microsoft 365 / Office (Outlook desktop) installed: `winget install Microsoft.Office`
- Outlook signed in once (interactive) so a default MAPI profile exists
- PowerShell 7 (`pwsh`) recommended; Windows PowerShell 5.1 also works
- **STA threading required** for non-trivial COM sessions — launch pwsh with `-STA`:

  ```powershell
  pwsh -STA -Command "..."
  ```

## Core Pattern

Every Outlook COM session follows this shape. Always release COM objects and call
`$Outlook.Quit()` (or omit Quit if you want to leave a user-visible Outlook running).

```powershell
pwsh -STA -Command @'
$Outlook = $null; $ns = $null
try {
  $Outlook = New-Object -ComObject Outlook.Application
  $ns      = $Outlook.GetNamespace("MAPI")

  # ... do work — for every COM ref you assign to a variable, release it
  # in this finally block (in reverse creation order). Examples below.

} finally {
  if ($ns)      { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($ns)      | Out-Null }
  if ($Outlook) { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($Outlook) | Out-Null }
  $ns = $null; $Outlook = $null
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
}
'@
```

> **Child object release.** The pattern above only releases `$ns` and `$Outlook`. Every additional COM ref you capture (`$folder`, `$mail`, `$items`, `$hits`, `$account`, recipients, attachments) must be released the same way — `Marshal::ReleaseComObject($obj) | Out-Null` + `$obj = $null` — in reverse creation order in the `finally`. Leaked child refs are the most common reason `OUTLOOK.EXE` lingers after `$Outlook.Quit()`.

> Calling `$Outlook.Quit()` only shuts down the COM instance if no user-visible
> Outlook window is open. If the user already has Outlook running, skip `Quit()`.

## Default Folders (numeric constants)

When the Outlook PIA / typelib isn't loaded, `OlDefaultFolders` enum names won't
resolve. Use the numeric values directly with `$ns.GetDefaultFolder(<int>)`:

| Folder       | Const                       | Int |
| ------------ | --------------------------- | --- |
| DeletedItems | olFolderDeletedItems        | 3   |
| Outbox       | olFolderOutbox              | 4   |
| Sent         | olFolderSentMail            | 5   |
| Inbox        | olFolderInbox               | 6   |
| Calendar     | olFolderCalendar            | 9   |
| Contacts     | olFolderContacts            | 10  |
| Journal      | olFolderJournal             | 11  |
| Notes        | olFolderNotes               | 12  |
| Tasks        | olFolderTasks               | 13  |
| Drafts       | olFolderDrafts              | 16  |
| Junk         | olFolderJunk                | 23  |

```powershell
$inbox = $ns.GetDefaultFolder(6)   # Inbox
$cal   = $ns.GetDefaultFolder(9)   # Calendar
```

## Read Recent Mail

Sort descending by `ReceivedTime`, then take the top N. Always sort `Items`
**before** projecting / restricting when you want order.

```powershell
pwsh -STA -Command @'
$Outlook = New-Object -ComObject Outlook.Application
$ns      = $Outlook.GetNamespace("MAPI")
$inbox   = $ns.GetDefaultFolder(6)
$items   = $inbox.Items
$items.Sort("[ReceivedTime]", $true)   # $true = descending

$items | Select-Object -First 10 | ForEach-Object {
  [pscustomobject]@{
    Received = $_.ReceivedTime
    From     = $_.SenderName
    Subject  = $_.Subject
    Unread   = $_.UnRead
  }
} | Format-Table -AutoSize
'@
```

## Search with Restrict

`Restrict` returns a filtered `Items` collection. Pick **one** filter syntax per
call — Outlook rejects strings that mix bracketed `[Field]` shortcuts with a
`@SQL=` DASL clause inside the same expression. To combine clauses across
syntaxes, chain `.Restrict()` calls.

Date literals in bracketed syntax use the user's short date format (`'g'`).

```powershell
# Pure bracketed syntax — last 7 days, unread.
$items  = $inbox.Items
$cutoff = (Get-Date).AddDays(-7).ToString("g")
$step1  = $items.Restrict("[ReceivedTime] >= '$cutoff' AND [UnRead] = True")

# Then a DASL-only Restrict on the result to filter by sender domain.
$step2  = $step1.Restrict('@SQL="urn:schemas:httpmail:fromemail" LIKE ''%@example.com''')
"Hits: $($step2.Count)"
```

Simple subject search:

```powershell
$hits = $inbox.Items.Restrict("[Subject] = 'Weekly Report'")
# or substring (pure DASL):
$hits = $inbox.Items.Restrict('@SQL="urn:schemas:httpmail:subject" LIKE ''%report%''')
```

For very large mailboxes use `Application.AdvancedSearch` instead — it runs a
server-side search and raises `AdvancedSearchComplete`.

## Send Mail

`CreateItem(0)` makes a new `MailItem` (0 = `olMailItem`). Set fields, then call
`.Send()`. Use `.Display()` to open the compose window for human review, or
`.Save()` to drop into Drafts.

```powershell
pwsh -STA -Command @'
$Outlook = New-Object -ComObject Outlook.Application
$mail = $Outlook.CreateItem(0)
$mail.To       = "alice@example.com; bob@example.com"
$mail.CC       = "cc@example.com"
$mail.Subject  = "Hello from Outlook COM"
$mail.HTMLBody = "<p>Hi,</p><p>This is automated.</p>"
# Optional attachment
# $null = $mail.Attachments.Add("C:\path\to\file.pdf")

# Choose ONE:
$mail.Save()        # save to Drafts (safe)
# $mail.Display()   # open compose window for human review (safe)
# $mail.Send()      # SEND IMMEDIATELY (irreversible)
'@
```

### Picking a non-default From account

Direct assignment of `$mail.SendUsingAccount = $account` is a long-standing
COM footgun in PowerShell — Outlook defines that property as `put-by-ref` and
the regular `=` operator frequently silently fails. Use reflection to force
the proper invocation:

```powershell
$account = $Outlook.Session.Accounts | Where-Object { $_.SmtpAddress -eq "alt@example.com" }
[void]$mail.GetType().InvokeMember(
    "SendUsingAccount", 'SetProperty',
    $null, $mail, @($account))
```

## Calendar — List Events in a Range

Calendar Items must be `Sort`ed by `[Start]` **before** setting
`IncludeRecurrences = $true`. Otherwise recurring instances are not expanded
and Restrict returns garbage counts (often `Int32.MaxValue`).

```powershell
pwsh -STA -Command @'
$Outlook = New-Object -ComObject Outlook.Application
$ns      = $Outlook.GetNamespace("MAPI")
$cal     = $ns.GetDefaultFolder(9)
$items   = $cal.Items
$items.Sort("[Start]")
$items.IncludeRecurrences = $true

$start = (Get-Date).Date.ToString("g")
$end   = (Get-Date).Date.AddDays(7).ToString("g")
$range = $items.Restrict("[Start] >= '$start' AND [Start] <= '$end'")

$range | ForEach-Object {
  [pscustomobject]@{
    Start    = $_.Start
    End      = $_.End
    Subject  = $_.Subject
    Location = $_.Location
    Organizer= $_.Organizer
  }
} | Format-Table -AutoSize
'@
```

## Calendar — Create Event

```powershell
$appt = $Outlook.CreateItem(1)   # 1 = olAppointmentItem
$appt.Subject  = "Design review"
$appt.Location = "Teams"
$appt.Start    = [datetime]"2025-01-15 14:00"
$appt.End      = [datetime]"2025-01-15 15:00"
$appt.Body     = "Agenda: ..."
$appt.ReminderSet = $true
$appt.ReminderMinutesBeforeStart = 10
# Optional: invite attendees
$attendee = $appt.Recipients.Add("alice@example.com")
$attendee.Type = 1  # 1 = olRequired
$null = $appt.Recipients.ResolveAll()
$appt.MeetingStatus = 1  # 1 = olMeeting (sends invite). Use 0 for personal appt.
$appt.Save()
# $appt.Send()  # send invite (only if MeetingStatus = 1)
```

## Contacts

```powershell
$contacts = $ns.GetDefaultFolder(10)
$contacts.Items | Select-Object -First 20 | ForEach-Object {
  [pscustomobject]@{
    FullName = $_.FullName
    Email1   = $_.Email1Address
    Phone    = $_.MobileTelephoneNumber
    Company  = $_.CompanyName
  }
} | Format-Table -AutoSize
```

## Safety Rules — MANDATORY for Send

Mail sent through Outlook COM goes from the **user's real mailbox** with the
**user's real identity** and **cannot be recalled** once sent.

1. **Always confirm** the recipient list, subject, and body with the user before
   calling `.Send()`. If confirmation is not possible in context, prefer `.Save()`
   (Drafts) or `.Display()` (compose window) so the user reviews and clicks Send.
2. **Default to Drafts.** Treat `.Send()` as an explicit, one-step-at-a-time
   action — never chain it after `Restrict`/loops without per-item confirmation.
3. **Never send bulk mail** (>1 recipient or >1 message) without a clear, written
   "yes send all N" from the user.
4. **Never auto-reply / auto-forward** based on inbox content. Outlook rules exist
   for that; agents should not re-implement them.
5. **Verify recipients** look right (typo'd domains, external-vs-internal). If
   anything looks off, stop and ask.
6. **Don't add attachments** the user didn't ask for. Confirm file paths exist
   and that the user actually wants them attached.
7. **Calendar invites are mail.** `MeetingStatus = 1` + `Send()` blasts an invite
   to attendees. Same rules apply.
8. **Rate-limit sends.** One send per explicit user confirmation. Never wrap
   `.Send()` in a loop without per-iteration consent — even legitimate batch
   sends should be paced (sleep + per-recipient confirmation) to avoid Outlook
   tripping send-throttle or recipient-rate-limit policies.

## Notes

- **STA only.** `pwsh -STA -Command ...` for anything non-trivial. The default
  PowerShell 7 host is MTA, which breaks many Outlook COM calls (especially modal
  UI like `.Display()` and recipient resolution).
- **Marshal release.** Always release COM refs in a `finally` block and call
  `[GC]::Collect()` once at the end. Leaked refs leave zombie `OUTLOOK.EXE`
  processes that block re-launch.
- **`Sort` before `Restrict`** for `Items` collections where order or recurrence
  expansion matters — especially Calendar with `IncludeRecurrences`.
- **Date literals** in Restrict use the user's short-date format. Build with
  `(Get-Date).ToString("g")`, not ISO 8601.
- **Outlook must be installable / runnable.** A licensed M365 profile is
  required. New-Outlook-only environments (no classic Outlook) lack the COM
  surface and must fall back to Microsoft Graph.
- **No headless service accounts.** COM runs in the interactive desktop session;
  it won't work under Task Scheduler with "Run whether user is logged on or not"
  unless an interactive session is forced.

---
name: scheduled-tasks
description: Schedule recurring or one-shot jobs on Windows from PowerShell using the ScheduledTasks module — the Windows analogue of cron / launchd. Use when the user asks to "run this every day at 9am", "set up a cron job on Windows", "schedule a script at startup", "make this run at login", "list my scheduled tasks", "disable / re-enable / delete that task", or "trigger as a different user". Covers Get-ScheduledTask, New-ScheduledTaskAction, New-ScheduledTaskTrigger (Daily, Weekly, AtLogOn, AtStartup, Once), Register-ScheduledTask, principals (current user vs SYSTEM, run whether logged on or not), and a complete worked example.
metadata:
  openclaw:
    emoji: "⏰"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
---

# Scheduled Tasks (Windows cron parity)

Schedule recurring or one-shot jobs on Windows from PowerShell using the built-in **ScheduledTasks** module. This is the Windows analogue of `cron`, `launchd`, or `systemd timers`. Tasks live in Task Scheduler (`taskschd.msc`) and survive reboots.

## When to Use

✅ **USE this skill when:**

- User asks to "run X every day / week / hour"
- User asks for a Windows cron / launchd equivalent
- User wants a script to run at logon, at startup, or at a specific time
- User wants to list / disable / enable / delete an existing scheduled job
- User asks to run the task as a different user or as SYSTEM

## When NOT to Use

❌ **DON'T use this skill when:**

- User wants something that runs **only** while a specific app is open → just run it in that app
- User wants a fast in-process timer / interval inside a long-running PowerShell session → use `Register-ObjectEvent` on a `Timer` or a `Start-Job` loop
- User wants a one-off "wait 5 minutes and do X" — `Start-Sleep` in a backgrounded job is simpler than registering a task
- User is on macOS / Linux — use `launchd` / `systemd` / `cron`

## Setup

No install required. The `ScheduledTasks` module ships with Windows 8 / Server 2012 and later. To verify:

```powershell
Get-Command -Module ScheduledTasks | Select-Object Name | Format-Wide -Column 3
```

Registering a task does **not** require admin **if** you only schedule under your own user and the principal is the current user. Some operations (running as SYSTEM, registering under `\` root, or `Run with highest privileges`) require an elevated PowerShell.

## Listing Tasks

```powershell
# All tasks (a lot — Windows ships many)
Get-ScheduledTask

# Only tasks you authored / non-Microsoft, under the root path
Get-ScheduledTask -TaskPath '\' | Select-Object TaskName, State

# Search by name
Get-ScheduledTask -TaskName 'Backup*'

# Detail + last run info
Get-ScheduledTask -TaskName 'MyJob' | Get-ScheduledTaskInfo
```

`State` is one of `Ready`, `Running`, `Disabled`. `Get-ScheduledTaskInfo` adds `LastRunTime`, `LastTaskResult` (0 = success), and `NextRunTime`.

## Anatomy of a Task

Every registered task has three required pieces and a couple of optional ones:

| Component   | Cmdlet                      | What it is                                                 |
| ----------- | --------------------------- | ---------------------------------------------------------- |
| Action      | `New-ScheduledTaskAction`   | The program + args to run.                                 |
| Trigger     | `New-ScheduledTaskTrigger`  | When it fires (daily, weekly, AtLogOn, AtStartup, Once).   |
| Principal   | `New-ScheduledTaskPrincipal`| Who it runs as (default: current user).                    |
| Settings    | `New-ScheduledTaskSettingsSet` | Optional: retry, allow on battery, hidden, timeouts.    |
| Description | argument to `Register-ScheduledTask` | Free-text shown in Task Scheduler UI.            |

Compose them with `New-ScheduledTask`, then call `Register-ScheduledTask` to commit.

## Quick Recipes

### Daily at a Specific Time

```powershell
$action  = New-ScheduledTaskAction `
    -Execute 'pwsh.exe' `
    -Argument '-NoProfile -File "C:\Scripts\daily-cleanup.ps1"'

$trigger = New-ScheduledTaskTrigger -Daily -At 9:00am

Register-ScheduledTask `
    -TaskName 'OpenClaw Daily Cleanup' `
    -Action $action -Trigger $trigger `
    -Description 'Daily cleanup at 9am'
```

> Avoid `-ExecutionPolicy Bypass` in the action argument by default — it weakens the user's policy posture for every run. Prefer signing the script, or pinning the policy with `Set-ExecutionPolicy -Scope LocalMachine RemoteSigned`. Only add `-ExecutionPolicy Bypass` to the argument string as a deliberate fallback for unsigned ad-hoc scripts and call it out to the user when you do.

### Every Weekday

```powershell
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday,Tuesday,Wednesday,Thursday,Friday -At 8:30am
```

### Every N Minutes

`New-ScheduledTaskTrigger` does not expose minute-level scheduling directly. Build a `-Once` trigger and let it repeat indefinitely:

```powershell
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
    -RepetitionInterval (New-TimeSpan -Minutes 15) `
    -RepetitionDuration ([TimeSpan]::MaxValue)
```

Older "compose `-Daily` then graft `.Repetition`" recipes are footguns — Task Scheduler frequently rejects the merged XML with "The task XML contains a value which is incorrectly formatted or out of range." Sub-minute intervals are not supported either — minimum repetition is 1 minute.

### At User Logon

```powershell
$trigger = New-ScheduledTaskTrigger -AtLogOn
# Restrict to a specific user:
$trigger = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$env:USERNAME"
```

### At System Startup (requires admin to register)

```powershell
$trigger = New-ScheduledTaskTrigger -AtStartup
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName 'OpenClaw Bootstrap' -Action $action -Trigger $trigger -Principal $principal
```

### One-Shot in the Future

```powershell
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(30)
```

## Running As

Default: the task runs as the **current user**, **interactively**, only when they are logged on.

To change that, build a principal. Prefer **S4U** (service-for-user) or a **gMSA** so no password is ever passed through PowerShell. The password-extraction example is for ad-hoc interactive use only; it can also leak the plaintext into `PSReadLine` history.

```powershell
# Preferred: S4U — runs as the user without storing/passing a password.
# Task fires whether the user is logged on or not, but the principal has no
# network credentials (it cannot reach SMB shares as that user).
$principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" `
    -LogonType S4U -RunLevel Limited

# Preferred for service accounts that need network creds: gMSA (domain).
# $principal = New-ScheduledTaskPrincipal -UserId 'DOMAIN\svc_my_job$' -LogonType Password

# Run as SYSTEM (no UI, no clipboard, no toast notifications, requires elevation).
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -LogonType ServiceAccount -RunLevel Highest

# Highest privileges as current user — note: registering with -RunLevel Highest
# requires the *registering* PowerShell to already be running as admin, even
# when the principal is your own account.
$principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" -RunLevel Highest -LogonType Interactive

# Ad-hoc only — DO NOT script in a long-lived shell. Disable PSReadLine
# history for the session so the plaintext password does not land in
# %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt.
Set-PSReadLineOption -HistorySaveStyle SaveNothing
$cred = Get-Credential   # prompt the user; never inline the password
Register-ScheduledTask -TaskName 'MyJob' -Action $action -Trigger $trigger `
    -User $cred.UserName -Password $cred.GetNetworkCredential().Password
Remove-Variable cred
```

> **Important:** Tasks running as `SYSTEM` cannot show toasts (see `windows-notifications` skill) or interact with the desktop. If the user wants a UI side effect, run as the interactive user.

## Settings That Matter

```powershell
$settings = New-ScheduledTaskSettingsSet `
    -StartWhenAvailable `
    -DontStopOnIdleEnd `
    -AllowStartIfOnBatteries `
    -DontStopIfGoingOnBatteries `
    -ExecutionTimeLimit (New-TimeSpan -Hours 1) `
    -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 5)
```

| Flag                          | Why you usually want it                                              |
| ----------------------------- | -------------------------------------------------------------------- |
| `-StartWhenAvailable`         | If the PC was off at the scheduled time, run as soon as possible.    |
| `-AllowStartIfOnBatteries`    | Otherwise the task is skipped on laptops on battery.                 |
| `-DontStopIfGoingOnBatteries` | Otherwise the task is killed mid-flight when unplugged.              |
| `-ExecutionTimeLimit`         | Default is 72h. Set lower to avoid runaway tasks.                    |
| `-RestartCount` / `-Interval` | Auto-retry on failure.                                               |

Pass with `Register-ScheduledTask -Settings $settings`.

## Disable / Enable / Unregister

```powershell
# Stop firing without losing the definition
Disable-ScheduledTask -TaskName 'OpenClaw Daily Cleanup'

# Resume
Enable-ScheduledTask  -TaskName 'OpenClaw Daily Cleanup'

# Trigger right now, out-of-band
Start-ScheduledTask   -TaskName 'OpenClaw Daily Cleanup'

# Delete the task entirely
Unregister-ScheduledTask -TaskName 'OpenClaw Daily Cleanup' -Confirm:$false
```

## Worked Example — Daily PowerShell Script at 9 AM with Notification

End-to-end: create a script, register a task, verify it, and clean up.

```powershell
# 1. Create the script the task will run
$script = 'C:\Scripts\daily-hello.ps1'
New-Item -ItemType Directory -Path (Split-Path $script -Parent) -Force | Out-Null
@'
Import-Module BurntToast
New-BurntToastNotification -Text 'OpenClaw', "Daily ping at $(Get-Date -Format HH:mm)"
'@ | Set-Content -Path $script -Encoding UTF8

# 2. Build the task
$action  = New-ScheduledTaskAction `
    -Execute 'pwsh.exe' `
    -Argument "-NoProfile -ExecutionPolicy Bypass -File `"$script`""

$trigger = New-ScheduledTaskTrigger -Daily -At 9:00am

$settings = New-ScheduledTaskSettingsSet `
    -StartWhenAvailable `
    -AllowStartIfOnBatteries `
    -DontStopIfGoingOnBatteries `
    -ExecutionTimeLimit (New-TimeSpan -Minutes 5)

# 3. Register (runs as current user, only when logged on by default)
Register-ScheduledTask `
    -TaskName 'OpenClaw Daily Hello' `
    -Action $action -Trigger $trigger -Settings $settings `
    -Description 'Says hello every day at 9am.'

# 4. Verify
Get-ScheduledTask -TaskName 'OpenClaw Daily Hello' | Get-ScheduledTaskInfo

# 5. Run it once now to sanity check
Start-ScheduledTask -TaskName 'OpenClaw Daily Hello'

# 6. Clean up when done
# Unregister-ScheduledTask -TaskName 'OpenClaw Daily Hello' -Confirm:$false
```

## Exporting / Importing Tasks

For backup or to move a task to another machine:

```powershell
# Export to XML (Task Scheduler's native format)
Export-ScheduledTask -TaskName 'OpenClaw Daily Hello' |
    Set-Content -Path 'C:\Backups\openclaw-daily.xml' -Encoding UTF8

# Re-import elsewhere
Register-ScheduledTask `
    -TaskName 'OpenClaw Daily Hello' `
    -Xml (Get-Content 'C:\Backups\openclaw-daily.xml' -Raw)
```

This is the most reliable way to round-trip complex tasks (multi-trigger, principal options, conditions).

## Safety Rules

1. **Always name the task with a recognisable prefix** (e.g. `OpenClaw …`) so the user can find and audit jobs you registered.
2. **Confirm before deleting.** `Unregister-ScheduledTask -Confirm:$false` is silent and irreversible. Default to a confirm prompt unless the user explicitly asked for non-interactive cleanup.
3. **Do not write user passwords to disk.** For "run whether user is logged on or not" prefer `-LogonType S4U` (no password stored, but limited rights) or use `Get-Credential` interactively each time you register.
4. **Sanitize script paths and arguments.** A script path containing spaces must be quoted; an arbitrary user-supplied argument string must not be interpreted by `cmd /c`.
5. **Be careful with `-User SYSTEM` / `-RunLevel Highest`.** These elevate beyond the calling user and silently bypass UAC for that command. Reserve for genuinely system-level work and surface what you are doing to the user first.
6. **Test with `Start-ScheduledTask` before walking away.** A task that registers cleanly can still fail at runtime — bad path, missing module, wrong working directory.

## Notes

- Triggers are AND-ed with conditions (network, idle, AC power). If your task does not fire, check `Get-ScheduledTaskInfo` and the History tab in `taskschd.msc`.
- `LastTaskResult` is an HRESULT. `0` = success, `0x41301` = currently running, `0x41303` = has not run yet. Most other non-zero values translate via `Get-Error` or `win32_error.h`.
- Time zone: triggers use **local time** by default. Pass `-At` a `[datetimeoffset]` to anchor in UTC.
- Tasks created with the `ScheduledTasks` module are stored under `\` by default. The legacy `schtasks.exe` tool can also read/write them — useful for one-off shell scripts where loading PowerShell would be overkill.
- For interactive UI side effects (toasts, GUI windows), the task **must** run as the logged-on interactive user, not as SYSTEM and not with "Run whether user is logged on or not".

---
name: windows-registry
description: Read and write the Windows registry from PowerShell. Use when the user asks to "check a registry key", "set a registry value", "list startup programs", "look at HKLM/HKCU", "find an installed app's uninstall string", "edit App Paths or file associations", or "back up a registry key". Covers Get-ItemProperty, New-ItemProperty, Set-ItemProperty, Remove-Item under HKCU:\ and HKLM:\, the common keys (Run, Uninstall, App Paths, environment variables), and reg.exe as a legacy alternative for export/import.
metadata:
  openclaw:
    emoji: "🗝️"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
---

# Windows Registry

Read and write the Windows registry from PowerShell. The registry is a hierarchical, machine-wide configuration store with two relevant root hives:

- `HKCU:\` (HKEY_CURRENT_USER) — current user; no elevation needed.
- `HKLM:\` (HKEY_LOCAL_MACHINE) — machine-wide; requires an elevated (Run as Administrator) PowerShell.

PowerShell exposes these as PSDrives, so all `Get-Item*`, `Set-Item*`, `New-Item*`, `Remove-Item*` cmdlets work against registry paths.

## When to Use

- ✅ Inspect a key: "what's in `HKCU:\Software\Microsoft\Windows\CurrentVersion\Run`?"
- ✅ List startup programs, installed apps (Uninstall), App Paths, file associations
- ✅ Read/write a single named value under a key the user already named
- ✅ Persist a user environment variable via `HKCU:\Environment`
- ✅ Back up a subtree before editing it (`reg export`)

## When NOT to Use

- ❌ Setting an environment variable for the current shell only → use `$env:NAME = "..."`
- ❌ Installing or uninstalling an app → use winget / MSI / the app's installer
- ❌ Editing group policy → use `gpedit.msc` / `Set-GPRegistryValue`
- ❌ "Cleaning the registry" with a sweep — refuse, this corrupts systems
- ❌ Editing `HKLM:\` from a non-elevated shell — it will silently fail or error

## Setup

PowerShell 5.1 (built into Windows) or PowerShell 7 (`pwsh`) — both work. No install needed; the registry provider is built in.

```powershell
Get-PSDrive -PSProvider Registry
# Name   Root
# ----   ----
# HKCU   HKEY_CURRENT_USER
# HKLM   HKEY_LOCAL_MACHINE
```

## Reading

### Read all values under a key

```powershell
Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run'
```

This returns one object whose properties are the registry values plus a few PowerShell metadata properties (`PSPath`, `PSParentPath`, `PSChildName`, `PSDrive`, `PSProvider`).

### Read a single named value

```powershell
(Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run' -Name 'OneDrive').OneDrive
```

Or, using `Get-ItemPropertyValue` (cleaner):

```powershell
Get-ItemPropertyValue -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run' -Name 'OneDrive'
```

### List subkeys

```powershell
Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall' |
    ForEach-Object { Get-ItemProperty $_.PSPath } |
    Where-Object { $_.DisplayName } |
    Select-Object DisplayName, DisplayVersion, Publisher, UninstallString |
    Sort-Object DisplayName
```

32-bit installs on 64-bit Windows live under the WoW6432Node mirror:

```powershell
Get-ChildItem 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall'
```

### Check if a key or value exists

```powershell
Test-Path 'HKCU:\Software\OpenClaw'

# Value existence (no direct cmdlet — check the property bag):
$null -ne (Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run' -Name 'OneDrive' -ErrorAction SilentlyContinue)
```

## Common Keys

| Path | Purpose |
|------|---------|
| `HKCU:\Software\Microsoft\Windows\CurrentVersion\Run` | Per-user startup programs |
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` | Machine-wide startup programs |
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall` | Installed 64-bit apps |
| `HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall` | Installed 32-bit apps on 64-bit Windows |
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths` | `start <name>` resolver |
| `HKCU:\Environment` | Per-user environment variables (persistent) |
| `HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment` | Machine-wide environment variables |
| `HKLM:\SOFTWARE\Classes` (preferred) or `HKCR:\` (after `New-PSDrive`) | File associations / ProgIDs / shell verbs |
| `HKCU:\Software\Classes` | Per-user file association overrides |

> `HKCR:` is **not** a default registry PSDrive — only `HKCU:` and `HKLM:` are auto-mapped. To use `HKCR:\…` examples, first run `New-PSDrive -Name HKCR -PSProvider Registry -Root HKEY_CLASSES_ROOT` (the drive is process-scoped). Otherwise prefer the underlying `HKLM:\SOFTWARE\Classes` (machine) or `HKCU:\Software\Classes` (per-user) paths, or use the provider-qualified form `Registry::HKEY_CLASSES_ROOT\…`.

## Writing

### Create / set a value

`Set-ItemProperty` creates the value if it doesn't exist OR overwrites it if it does. `New-ItemProperty` only creates and errors if it already exists (use `-Force` to overwrite).

```powershell
# Make sure the key exists first (creates intermediate keys too)
New-Item -Path 'HKCU:\Software\OpenClawDemo' -Force | Out-Null

# String (REG_SZ)
New-ItemProperty -Path 'HKCU:\Software\OpenClawDemo' -Name 'Greeting' -Value 'Hello' -PropertyType String -Force

# DWORD (REG_DWORD)
New-ItemProperty -Path 'HKCU:\Software\OpenClawDemo' -Name 'Count' -Value 42 -PropertyType DWord -Force

# Expandable string (REG_EXPAND_SZ)
New-ItemProperty -Path 'HKCU:\Software\OpenClawDemo' -Name 'Tools' -Value '%USERPROFILE%\tools' -PropertyType ExpandString -Force

# Multi-string (REG_MULTI_SZ)
New-ItemProperty -Path 'HKCU:\Software\OpenClawDemo' -Name 'List' -Value @('a','b','c') -PropertyType MultiString -Force
```

Property types: `String`, `ExpandString`, `Binary`, `DWord`, `MultiString`, `QWord`, `Unknown`.

### Rename / remove a value

```powershell
Rename-ItemProperty -Path 'HKCU:\Software\OpenClawDemo' -Name 'Greeting' -NewName 'Hello'
Remove-ItemProperty -Path 'HKCU:\Software\OpenClawDemo' -Name 'Greeting'
```

### Remove a key (and everything under it)

```powershell
Remove-Item -Path 'HKCU:\Software\OpenClawDemo' -Recurse -Force
```

### Add a startup entry (HKCU is safe; no admin)

```powershell
$exe = 'C:\Tools\MyTool\MyTool.exe'
New-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run' `
    -Name 'MyTool' -Value "`"$exe`"" -PropertyType String -Force
```

### Persist a user environment variable

```powershell
[Environment]::SetEnvironmentVariable('MY_VAR', 'value', 'User')   # preferred — broadcasts WM_SETTINGCHANGE
# OR raw registry write (will NOT be visible to existing processes):
New-ItemProperty -Path 'HKCU:\Environment' -Name 'MY_VAR' -Value 'value' -PropertyType String -Force
```

Prefer `[Environment]::SetEnvironmentVariable` — it triggers the broadcast that lets Explorer / new shells pick up the change.

## Backup & Restore (always do this before HKLM edits)

`reg.exe` is the legacy CLI; it's the simplest way to dump a key subtree to a `.reg` file.

```powershell
# Backup (note: reg.exe uses HKCU\... with a backslash, NOT HKCU:\)
reg export 'HKCU\Software\Microsoft\Windows\CurrentVersion\Run' "$env:USERPROFILE\Desktop\run-backup.reg" /y

# Restore
reg import "$env:USERPROFILE\Desktop\run-backup.reg"
```

Sample exported file header:

```
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\...]
```

## Legacy `reg.exe` Alternatives

Available in every Windows install; useful when PowerShell is unavailable (e.g., `cmd.exe` scripts, remote `psexec`, MSI custom actions).

```cmd
reg query   "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
reg add     "HKCU\Software\OpenClawDemo" /v Greeting /t REG_SZ /d "Hello" /f
reg delete  "HKCU\Software\OpenClawDemo" /v Greeting /f
reg export  "HKCU\Software\OpenClawDemo" demo.reg /y
reg import  demo.reg
```

Note the syntax difference: `reg.exe` uses `HKCU\...` (no colon); PowerShell uses `HKCU:\...`.

## Safety Rules

These are non-negotiable. The registry has no recycle bin — a bad edit can prevent boot, break sign-in, or quietly disable services.

1. **HKLM writes require explicit user confirmation.** Before any `Set-ItemProperty`, `New-ItemProperty`, `Remove-Item*`, or `reg add/delete` under `HKLM:\` (or `HKCR:\`, which writes back to HKLM), state the exact key + value + new data and wait for the user to say yes. Read-only `Get-*` against HKLM is fine.
2. **Always back up before writing.** Run `reg export <key> <path>.reg /y` for the key (or its parent) you're about to modify and tell the user where the `.reg` file is. They can `reg import` to roll back.
3. **Never delete a key without naming it back to the user first.** `Remove-Item -Recurse` is irreversible. Show the path and the subkey/value count first.
4. **Refuse "registry cleaning" / bulk delete requests.** They cause more damage than they fix.
5. **Never write to these paths without an explicit instruction naming the exact path:**
   - Anything under `HKLM:\SYSTEM\CurrentControlSet\Services\` (can disable boot drivers)
   - `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` (sign-in)
   - `HKLM:\SOFTWARE\Microsoft\Cryptography\` (machine identity)
   - `HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\` (kernel/session init)
6. **Quote paths with spaces or special characters** using single quotes in PowerShell: `'HKCU:\Software\My App'`.
7. **Use `-WhatIf` first** on destructive cmdlets if the user wants a dry run: `Remove-ItemProperty ... -WhatIf`.

## Notes

- `HKCR:\` (HKEY_CLASSES_ROOT) is a merged view; writes go to `HKLM:\SOFTWARE\Classes`. To override per-user, write to `HKCU:\Software\Classes` instead.
- `HKU:\` (HKEY_USERS) is NOT mapped as a PSDrive by default. Add it with `New-PSDrive -Name HKU -PSProvider Registry -Root HKEY_USERS` if you need to access another user's hive.
- Some keys are owned by `TrustedInstaller` and cannot be written even from an Administrator shell without taking ownership first — don't try unless the user explicitly asks.
- Group Policy will overwrite manual registry edits at the next refresh interval for policy-managed values. If a value keeps "snapping back", check `gpresult /h report.html`.
- Use single quotes around registry paths; PowerShell otherwise tries to expand `$` and other variables.

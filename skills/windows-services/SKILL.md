---
name: windows-services
description: Inspect and manage Windows services (NT services) from PowerShell. Use when the user asks to "list services", "is X running", "start/stop/restart a service", "change a service's startup type to Automatic/Manual/Disabled", "what depends on this service", "set a service to delayed start", or "configure the service binary / SDDL". Covers Get-Service, Start/Stop/Restart-Service, Set-Service, dependent / required service trees, and sc.exe for advanced operations.
metadata:
  openclaw:
    emoji: "🛎️"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
---

# Windows Services

Manage NT services — long-running background processes installed under the Service Control Manager (SCM). PowerShell exposes them via the `*-Service` cmdlets; `sc.exe` is the canonical low-level CLI for anything the cmdlets can't do.

## When to Use

- ✅ Check service status: "is Windows Search running?"
- ✅ Start / stop / restart a service
- ✅ Change startup type (Automatic, Automatic-Delayed-Start, Manual, Disabled)
- ✅ See what depends on a service (or what it depends on) before stopping it
- ✅ Find services by display name (`Get-Service -DisplayName "*update*"`)
- ✅ Configure a service's binary path, account, or security descriptor (via `sc.exe`)

## When NOT to Use

- ❌ Scheduled tasks → use `Get-ScheduledTask` / `schtasks.exe`
- ❌ Windows Subsystem for Linux units → those are systemd inside WSL; use the `wsl` skill
- ❌ User-mode background apps (Teams, OneDrive) — those are in the Run keys, see `windows-registry`
- ❌ "Disable everything that looks unnecessary" — refuse; many services have non-obvious deps
- ❌ Service installation from arbitrary binaries — confirm the binary with the user first

## Setup

Built into Windows. No install. PowerShell 5.1 (`powershell.exe`) or 7 (`pwsh`) both work.

**Admin rights required for write operations.** Reads (`Get-Service`) work as any user. `Start-Service`, `Stop-Service`, `Restart-Service`, `Set-Service`, `New-Service`, and `Remove-Service` require an elevated shell — check with:

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

If that returns `False`, tell the user to re-launch PowerShell as Administrator before retrying.

## Reading

### Get one service

```powershell
Get-Service -Name Themes
# Status   Name               DisplayName
# ------   ----               -----------
# Running  Themes             Themes
```

### Full detail

```powershell
Get-Service -Name Themes | Select-Object Name, DisplayName, Status, StartType, ServiceType, CanStop, CanPauseAndContinue
```

Useful properties: `Name`, `DisplayName`, `Status` (`Running`, `Stopped`, `Paused`, `StartPending`, `StopPending`), `StartType` (`Automatic`, `Manual`, `Disabled`, `Boot`, `System`).

### Find services

```powershell
Get-Service                                            # all
Get-Service | Where-Object Status -eq 'Running'
Get-Service -DisplayName '*Update*'
Get-Service -Name 'W*'
```

### Dependency tree

```powershell
# What needs this service to be running?
Get-Service -Name RpcSs -DependentServices | Select-Object Name, Status

# What does this service need?
Get-Service -Name BITS -RequiredServices | Select-Object Name, Status

# Recursive view of everything underneath
Get-Service -Name RpcSs -DependentServices -RequiredServices
```

> [!warning] Before stopping a service, **always** list its dependents. Stopping `RpcSs` (RPC) takes down most of Windows.

## Writing (Admin required)

### Start / stop / restart

```powershell
Start-Service   -Name Spooler
Stop-Service    -Name Spooler                # fails if dependents are running
Stop-Service    -Name Spooler -Force         # stop dependents too
Restart-Service -Name Spooler -Force

# Suspend / resume (only for services where CanPauseAndContinue is True)
Suspend-Service -Name <name>
Resume-Service  -Name <name>
```

`-Force` on `Stop-Service` cascades through dependents — confirm with the user before using it.

### Change startup type

```powershell
Set-Service -Name Spooler -StartupType Automatic
Set-Service -Name Spooler -StartupType Manual
Set-Service -Name Spooler -StartupType Disabled
```

Valid values for `Set-Service -StartupType`: `Automatic`, `AutomaticDelayedStart` (PowerShell 7+ only), `Manual`, `Disabled`. `Boot` and `System` are valid `ServiceStartMode` values for kernel drivers but **rejected** by the cmdlet — set those with `sc.exe config <name> start= boot|system` if you genuinely need to.

For PowerShell 5.1 (no `AutomaticDelayedStart` enum value), drop to `sc.exe`:

```powershell
sc.exe config Spooler start= delayed-auto       # mind the space after `start=`
```

### Change display name or description

```powershell
Set-Service -Name MyService -DisplayName "My Service (Production)" -Description "Handles foo and bar"
```

> `-Description` is PowerShell 7+ only. On Windows-default 5.1, use `sc.exe description <name> "<text>"` instead.

### Wait for a service to reach a state

```powershell
(Get-Service Spooler).WaitForStatus('Running', '00:00:30')   # TimeSpan, throws on timeout
```

## Advanced: `sc.exe`

`sc.exe` is the legacy SCM CLI — necessary for anything PowerShell cmdlets don't cover.

```cmd
sc.exe query        WSearch                    # status
sc.exe queryex      WSearch                    # status + PID, flags
sc.exe qc           WSearch                    # config (binary path, account, deps)
sc.exe enumdepend   RpcSs                      # dependents (one-shot, no recursion)
sc.exe sdshow       WSearch                    # security descriptor (SDDL)
sc.exe sdset        WSearch "<SDDL>"           # set SDDL (rarely needed)
sc.exe failure      WSearch reset= 86400 actions= restart/5000/restart/5000/run/0
sc.exe config       MyService binPath= "\"C:\path with spaces\my.exe\" --flag"
sc.exe config       MyService obj= "NT AUTHORITY\NetworkService" password= ""
sc.exe create       MyService binPath= "C:\tools\my.exe" start= auto DisplayName= "My Service"
sc.exe delete       MyService
```

> [!warning] `sc.exe` parameter syntax is unusual: `key= value` with the space **after** the `=`. `key=value` and `key =value` both fail silently or with cryptic errors.

PowerShell equivalents where available:

```powershell
New-Service    -Name MyService -BinaryPathName 'C:\tools\my.exe' -StartupType Automatic -DisplayName "My Service"
Remove-Service -Name MyService                          # PowerShell 6+
```

## Common Services Worth Knowing

| Name | Display Name | Notes |
|------|--------------|-------|
| `RpcSs` | Remote Procedure Call (RPC) | Foundational — do not stop |
| `LSM` | Local Session Manager | Do not stop |
| `Themes` | Themes | Visual styles |
| `WSearch` | Windows Search | Search indexer; see `windows-search` skill |
| `Spooler` | Print Spooler | Printing |
| `BITS` | Background Intelligent Transfer | Used by Windows Update |
| `wuauserv` | Windows Update | |
| `Dnscache` | DNS Client | |
| `WinDefend` | Microsoft Defender Antivirus | Tamper Protection may block changes |
| `MpsSvc` | Windows Defender Firewall | |
| `EventLog` | Windows Event Log | Required by many services |
| `LanmanServer` | Server | SMB file shares |
| `LanmanWorkstation` | Workstation | SMB client |

## Safety Rules

1. **Always check dependents before stopping anything.** `Get-Service -Name <name> -DependentServices` — list them to the user before calling `Stop-Service`.
2. **Refuse `-Force` stop on system-critical services** unless the user names the service and confirms. Critical includes (non-exhaustive): `RpcSs`, `LSM`, `EventLog`, `Dnscache`, `Winmgmt`, `WinDefend`, `MpsSvc`, `lsass` (LSA is a process, not a service, but warn anyway), anything under `HKLM:\SYSTEM\CurrentControlSet\Services\` flagged as a Boot or System driver.
3. **Never `Set-Service -StartupType Disabled` on a critical service** without explicit user confirmation naming the service.
4. **`Stop-Service WSearch`, `Spooler`, `BITS` are usually safe**; `wuauserv` is safe to stop transiently but Windows Update will re-enable it.
5. **Tamper Protection** blocks changes to Defender services (`WinDefend`, `Sense`, `WdNisSvc`) — even from admin. The change will appear to succeed but won't persist. Tell the user to turn off Tamper Protection in Windows Security first if they're sure.
6. **Always run from an elevated shell for writes.** Detect non-admin and tell the user to re-launch as Admin — don't pretend the call succeeded.
7. **Confirm `sc.exe create` / `New-Service` parameters** (binPath, account) before running. A wrong binary path or `obj=` account makes the service un-startable.
8. **Never `sc.exe delete` a service** without backing up its config first: `sc.exe qc <name> > <name>.txt` and `reg export "HKLM\SYSTEM\CurrentControlSet\Services\<name>" <name>.reg /y`.

## Notes

- The Service Control Manager database is cached. After `sc.exe create/delete`, the Services MMC (`services.msc`) may need a refresh (F5).
- Service start failures end up in the Windows Event Log under `System` — `Get-WinEvent -LogName System -MaxEvents 50 | Where-Object ProviderName -eq 'Service Control Manager'`.
- `Get-Service` returns `ServiceController` objects, NOT WMI/CIM. For account, binary path, or description, use `Get-CimInstance -ClassName Win32_Service -Filter "Name='X'"` or `sc.exe qc`.
- Services running as `LocalSystem` (the default for many) have full machine access. Any change to `binPath` is a privilege-escalation primitive — review carefully.
- Recovery actions (what happens on crash) are not exposed by `Set-Service`; use `sc.exe failure` or the Services MMC.

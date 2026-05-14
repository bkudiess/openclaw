---
name: file-explorer
description: Drive Windows Explorer from PowerShell — open folders, reveal a specific file in its parent folder with it selected, pin folders to Quick Access, resolve Known Folder paths (Desktop, Documents, Downloads, AppData, …), and list recently used items. Use when the user says "open this in Explorer", "show me this file in Explorer", "pin this folder to Quick Access", "where is my Downloads folder", "what was I working on recently", or "reveal in Finder" on Windows. Covers explorer.exe /select, Shell.Application COM verbs, Environment.GetFolderPath, and the Recent items folder.
metadata:
  openclaw:
    emoji: "🗂️"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
---

# File Explorer (Windows)

Drive Windows Explorer from PowerShell: open folders, reveal a specific file with it pre-selected, pin items to Quick Access, resolve Known Folder paths (Desktop / Documents / Downloads / AppData / …), and surface recently used files. Equivalent to the macOS Finder "reveal in Finder" / `open` flow.

## When to Use

✅ **USE this skill when:**

- User asks to open a folder in Explorer
- User asks to "show me X in Explorer" / "reveal that file" — needs the parent folder open with the file selected
- User asks to pin a folder to Quick Access
- User asks for the canonical path of Desktop, Documents, Downloads, AppData, etc.
- User asks "what files have I opened recently?"

## When NOT to Use

❌ **DON'T use this skill when:**

- User wants to **read** a file's contents — use `Get-Content` / your normal file tools
- User wants to copy files between locations — use `Copy-Item`
- User wants the file lister to be interactive within the terminal — use `lf`, `nnn`, or `eza`
- User is on macOS — use `open -R <path>` and the Finder skill

## Setup

No install required. Everything is built into Windows: `explorer.exe`, the `Shell.Application` COM object, and the `[Environment]` .NET class.

## Open a Folder in Explorer

```powershell
# By absolute path
explorer.exe 'C:\Users\me\Projects'

# Or via Invoke-Item — same effect, idiomatic PowerShell
Invoke-Item 'C:\Users\me\Projects'
```

Quoting matters: paths with spaces need single or double quotes.

## Reveal a File in Explorer (parent folder + file selected)

This is the Windows analogue of macOS "Reveal in Finder". Use `/select,`:

```powershell
$path = (Resolve-Path -LiteralPath 'C:\Users\me\Documents\report.pdf').Path
# Pass /select, and the path as separate ArgumentList entries — Start-Process
# quotes each element for you and Explorer parses them correctly. Embedding the
# path inside a single `/select,"…"` string causes Explorer to treat the quotes
# as part of the path on some builds and silently open the user's home folder.
Start-Process explorer.exe -ArgumentList '/select,', $path
```

A few gotchas:

- The argument starts with `/select,` and is passed as **a separate token** from the path. No inner quotes around the path.
- Forward slashes in the path also work, but native backslashes are safest.
- If the path does not exist, Explorer silently opens the user's home folder.

Helper that handles resolution for you:

```powershell
function Show-InExplorer {
    param([Parameter(Mandatory)] [string] $Path)
    $full = (Resolve-Path -LiteralPath $Path).Path
    Start-Process explorer.exe -ArgumentList '/select,', $full
}

Show-InExplorer 'C:\Users\me\Documents\report.pdf'
```

## Resolve Known Folder Paths

Always use `[Environment]::GetFolderPath` — do **not** assume `C:\Users\<name>\Documents`. The user may have redirected folders to OneDrive, to a different drive, or to a non-default user profile path.

```powershell
[Environment]::GetFolderPath('Desktop')
# → C:\Users\me\OneDrive\Desktop      (varies by OneDrive setup)

[Environment]::GetFolderPath('MyDocuments')
[Environment]::GetFolderPath('MyPictures')
[Environment]::GetFolderPath('MyMusic')
[Environment]::GetFolderPath('MyVideos')
[Environment]::GetFolderPath('ApplicationData')      # %APPDATA%
[Environment]::GetFolderPath('LocalApplicationData') # %LOCALAPPDATA%
[Environment]::GetFolderPath('UserProfile')          # %USERPROFILE%
[Environment]::GetFolderPath('Recent')               # Recent Items shortcut store
[Environment]::GetFolderPath('Startup')              # Start-up programs
```

Full enum list:

```powershell
[Enum]::GetNames([System.Environment+SpecialFolder])
```

`Downloads` is **not** in the SpecialFolder enum — Microsoft never added it. Use the well-known KnownFolderId via Shell COM (with a registry fallback for redirected setups / Server SKUs where the namespace lookup returns `$null`):

```powershell
# Preferred: Downloads via the shell well-known GUID.
$shell = New-Object -ComObject Shell.Application
try {
  $downloads = $shell.Namespace('shell:Downloads').Self.Path
  if (-not $downloads) { throw "shell:Downloads returned null" }
  $downloads
} catch {
  # Fallback: read the per-user shell-folder redirect from the registry. The
  # GUID is the Downloads KnownFolderId.
  (Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders').'{374DE290-123F-4565-9164-39C4925E467B}'
} finally {
  if ($shell) { [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($shell) }
}
```

## Pin a Folder to Quick Access

Quick Access pinning is exposed as the `pintohome` shell verb on the **folder** item.

```powershell
function Add-ToQuickAccess {
    param([Parameter(Mandatory)] [string] $Path)
    $full = (Resolve-Path -LiteralPath $Path).Path
    $shell = New-Object -ComObject Shell.Application
    try {
        $parent = Split-Path $full -Parent
        $leaf   = Split-Path $full -Leaf
        $item = $shell.Namespace($parent).ParseName($leaf)
        if (-not $item) { throw "Not found: $full" }
        # InvokeVerb('pintohome') is the canonical (locale-independent) verb id.
        $item.InvokeVerb('pintohome')
    } finally {
        if ($shell) { [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($shell) }
    }
}

Add-ToQuickAccess 'C:\Projects\openclaw'
```

Notes on the verb:

- The display name (e.g. `Pin to &Quick access` on en-US, `An "Schnellzugriff" anheften` on de-DE) is localized. Always prefer the canonical id `pintohome` over a `Verbs().Name` match.

## Unpin from Quick Access

```powershell
$shell = New-Object -ComObject Shell.Application
try {
  $item = $shell.Namespace((Split-Path $full -Parent)).ParseName((Split-Path $full -Leaf))
  $item.InvokeVerb('unpinfromhome')
} finally {
  if ($shell) { [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($shell) }
}
```

## List Quick Access Items

Quick Access is exposed via a well-known shell namespace GUID.

```powershell
$shell = New-Object -ComObject Shell.Application
try {
  $qa = $shell.Namespace('shell:::{679f85cb-0220-4080-b29b-5540cc05aab6}')
  $qa.Items() | Select-Object Name, Path
} finally {
  if ($shell) { [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($shell) }
}
```

This returns both user-pinned **and** frequent items. The column index for "Pinned" is locale-dependent on older Windows builds; if `GetDetailsOf($item, 274)` comes back as `$null`, iterate columns with `GetDetailsOf($null, $i)` to find the right index on the target machine.

## List Recently Used Files

Windows tracks recent files as `.lnk` shortcuts in the user's Recent folder.

```powershell
$recent = [Environment]::GetFolderPath('Recent')
Get-ChildItem $recent -Filter *.lnk |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 Name, LastWriteTime
```

To resolve each shortcut to its target:

```powershell
$shell = New-Object -ComObject WScript.Shell
try {
  Get-ChildItem $recent -Filter *.lnk |
      Sort-Object LastWriteTime -Descending |
      Select-Object -First 10 |
      ForEach-Object {
          $lnk = $shell.CreateShortcut($_.FullName)
          [PSCustomObject]@{
              Name       = $_.BaseName
              Target     = $lnk.TargetPath
              LastOpened = $_.LastWriteTime
          }
      }
} finally {
  if ($shell) { [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($shell) }
}
```

The Recent folder is per-user and is what Explorer's "Recent files" section under Quick Access reads from.

## Open a Special Folder in Explorer

Any `shell:` URI (built-ins like `shell:Downloads`, `shell:Desktop`, `shell:AppsFolder`, …) works:

```powershell
explorer.exe shell:Downloads
explorer.exe shell:Desktop
explorer.exe shell:AppsFolder   # the "All apps" Start menu
explorer.exe shell:RecycleBinFolder
```

## Safety Rules

1. **Validate paths before invoking verbs.** A typo'd path silently opens the user's home folder; check existence with `Test-Path -LiteralPath` first and surface a clear error.
2. **Confirm before pin / unpin.** Quick Access is user-visible state — do not mass-pin/unpin without saying what you are about to do.
3. **Treat Recent items as PII.** Document titles, project paths, and file names can leak meaningful information. Do not log or transmit `Recent` contents without the user asking for it.
4. **Do not run shell verbs as `SYSTEM`.** Quick Access pinning, Recent, and `shell:` namespaces are per-user state; running these out of an interactive session pins to the wrong profile or fails silently.

## Notes

- `Shell.Application` is a COM object — it works in PowerShell 7 on Windows and in Windows PowerShell 5.1.
- Quick Access pinning writes to `%APPDATA%\Microsoft\Windows\Recent\AutomaticDestinations\f01b4d95cf55d32a.automaticDestinations-ms`. That file is opaque (jump list format). Do not edit it directly.
- For users with Files Explorer replaced by a third-party app (Files, Directory Opus, Total Commander), `explorer.exe` calls still go to the built-in Explorer — there is no clean way to route to a replacement via the standard APIs.
- The `pintohome` / `unpinfromhome` verbs were added in Windows 10. On Windows 7/8 the equivalent was "Pin to Favorites" — different verb, different namespace, out of scope here.

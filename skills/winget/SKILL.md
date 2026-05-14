---
name: winget
description: Use the Windows Package Manager CLI (`winget`) to search, install, upgrade, list, show, export, and import Windows software. Cover silent installs, source management, pinning, and machine-portable JSON manifests. Use when the user asks to "install X on Windows", "upgrade everything", "list installed apps", "export my installed apps", "find a Windows package for Y", or any other Windows software install/inventory task. Helps other Windows skills bootstrap their dependencies.
metadata:
  openclaw:
    emoji: "📦"
    os: ["win32"]
    requires:
      bins: ["winget"]
---

# winget — Windows Package Manager

`winget` is Microsoft's official CLI package manager for Windows. Use it to
search the Microsoft Store + community manifests, install/upgrade/uninstall
software unattended, and export/import an entire machine's app list as JSON.

## When to Use

✅ **USE this skill when:**

- Installing Windows software from the CLI (bootstrap dev tools, runtimes, apps)
- Upgrading installed packages
- Auditing what is installed on the box
- Producing or restoring a JSON inventory of installed apps
- Locating a package id for a known app name

## When NOT to Use

❌ **DON'T use this skill when:**

- macOS / Linux → use `brew` / `apt` / `dnf`
- Per-user PowerShell modules → use `Install-Module` (PSGallery)
- Container or service installs → use the official installer/MSI directly
- npm / pip / cargo packages → use the language-native tool

## Setup

- Windows 10 1809+ or Windows 11. The CLI ships in the **App Installer** Store
  package. Verify:

  ```powershell
  Get-AppxPackage Microsoft.DesktopAppInstaller |
    Select-Object Name, Version
  winget --version
  ```

  If `winget` is missing, install / update **App Installer** from the Microsoft
  Store (or `Add-AppxPackage` the latest `.msixbundle` from GitHub releases).
- For unattended runs, always pass
  `--accept-source-agreements --accept-package-agreements --silent`.

## Search

```powershell
# Free-text search across name, id, tags, moniker
winget search "Notepad++" --accept-source-agreements

# Restrict to a single source
winget search vscode --source winget

# Narrow by id or moniker
winget search --id Microsoft.PowerToys
winget search --moniker code
```

## Show — package details before installing

```powershell
winget show Microsoft.PowerToys --accept-source-agreements
# Prints publisher, version, license, homepage, description, installer types,
# and silent install switches.
```

## Install

```powershell
# Standard unattended install
winget install --id Microsoft.PowerToys `
  --silent --accept-source-agreements --accept-package-agreements

# Pin to a specific version
winget install --id Microsoft.PowerShell --version 7.4.0 --silent `
  --accept-source-agreements --accept-package-agreements

# Choose installer scope (when the package offers user vs machine)
winget install --id Microsoft.VisualStudioCode --scope user --silent `
  --accept-source-agreements --accept-package-agreements

# Override the installer's silent switches (rare, advanced)
winget install --id Some.Package --override "/VERYSILENT /NORESTART"
```

Useful flags: `--exact` (match id exactly), `--location <path>` (where the app
allows it), `--architecture x64|arm64|x86`, `--locale en-US`.

## Upgrade

```powershell
# What can be upgraded
winget upgrade --include-unknown

# Upgrade a single package
winget upgrade --id Microsoft.PowerToys --silent `
  --accept-source-agreements --accept-package-agreements

# Upgrade everything (the usual "winget update all")
winget upgrade --all --silent `
  --accept-source-agreements --accept-package-agreements
```

`--include-unknown` shows apps whose installed version isn't reported in ARP
(common for portable / store apps).

## List Installed

```powershell
# Everything winget knows about
winget list --accept-source-agreements

# Filter by name / id / source
winget list --name "Visual Studio"
winget list --id Microsoft.PowerShell
winget list --source winget
```

For scripting, pipe through PowerShell parsing rather than depending on column
widths — winget's tabular output is fragile. Use `--source winget` to limit to
packages with stable ids.

## Uninstall

```powershell
winget uninstall --id Microsoft.PowerToys --silent
```

`--purge` removes leftover config; `--preserve` keeps it.

## Export / Import (machine portability)

Export captures installed packages as a JSON manifest you can replay on another
machine.

```powershell
# Export
winget export --output C:\backup\apps.json --accept-source-agreements `
  --include-versions

# Import (on the new box)
winget import --import-file C:\backup\apps.json `
  --accept-source-agreements --accept-package-agreements `
  --ignore-unavailable --ignore-versions
```

`--ignore-unavailable` skips packages winget can no longer find on import
(common after publisher renames). `--ignore-versions` lets you pin only the id.

## Pin / Unpin (block upgrades)

```powershell
# Prevent upgrade of a specific package
winget pin add --id Microsoft.PowerToys

# Pin to a version range
winget pin add --id Microsoft.PowerShell --version 7.4.*

# List pins
winget pin list

# Remove a pin
winget pin remove --id Microsoft.PowerToys
```

## Source Management

```powershell
winget source list
# msstore   https://storeedgefd.dsx.mp.microsoft.com/v9.0
# winget    https://cdn.winget.microsoft.com/cache
# winget-font ...

# Refresh cached manifests
winget source update

# Add a private REST source
winget source add --name corp --arg https://winget.corp.example.com/api --type Microsoft.Rest

# Reset everything to defaults
winget source reset --force
```

## Common Bootstrap One-Liners

```powershell
# Dev essentials
winget install --id Git.Git                 --silent --accept-source-agreements --accept-package-agreements
winget install --id Microsoft.PowerShell    --silent --accept-source-agreements --accept-package-agreements
winget install --id Microsoft.VisualStudioCode --silent --accept-source-agreements --accept-package-agreements
winget install --id Microsoft.WindowsTerminal  --silent --accept-source-agreements --accept-package-agreements

# For other skills in this repo
winget install --id SQLite.SQLite           --silent --accept-source-agreements --accept-package-agreements
winget install --id Microsoft.Office        --silent --accept-source-agreements --accept-package-agreements
```

## Exit Codes

`winget` returns 0 on success. Non-zero codes include:

| Code         | Meaning                                                        |
| ------------ | -------------------------------------------------------------- |
| 0            | Success                                                        |
| 0x8A150010   | No applicable upgrade found                                    |
| 0x8A150011   | Package already installed                                      |
| 0x8A15002B   | Upgrade requires installer override / interactive              |
| 0x8A150049   | Source agreements not accepted                                 |
| (installer)  | Pass-through from the underlying installer (e.g. MSI 1603)     |

Check `$LASTEXITCODE` after each call when scripting.

## Notes

- **`--silent` is best-effort.** Some installers ignore it. If a GUI flashes,
  the publisher's manifest doesn't expose a silent switch — use `--override`
  with the installer's known quiet flags, or accept the GUI.
- **Reboots.** Many installers request a reboot; `winget` does **not** reboot
  automatically. Check `$LASTEXITCODE` (often `3010` from MSI = "reboot
  required").
- **Source agreements.** First-time use prompts to accept the msstore + winget
  source agreements. In automation, always pass
  `--accept-source-agreements --accept-package-agreements`.
- **Machine vs user scope.** `--scope machine` requires elevation. Default is
  whatever the package declares; some publishers force one or the other.
- **App Installer version drift.** New `winget` features (e.g. `winget
  configure`, `winget configuration`) require newer App Installer versions —
  `Get-AppxPackage Microsoft.DesktopAppInstaller | Select Version` to check.
- **JSON output** is available via `winget export` for installed apps. Other
  subcommands don't yet emit structured output reliably — parse text carefully
  or wrap with `ConvertFrom-Csv` after `Out-String` cleanup.
- **Pair this skill** with other Windows skills here (`outlook-com`, `excel-com`,
  `word-com`, `sticky-notes`) to one-shot install their prerequisites.

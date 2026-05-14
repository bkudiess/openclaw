---
name: windows-terminal
description: Automate Windows Terminal (wt.exe) — open profiles, split panes, switch tabs, and edit settings.json. Use when the user asks to "open Windows Terminal", "open a new tab", "split the pane", "launch PowerShell / WSL / cmd in a new tab", "focus tab N", "edit my Windows Terminal settings", or "change my default profile". Covers the wt.exe command-line surface (with the `;` command separator), launching named profiles, working-directory and title flags, and the settings.json location for declarative profile edits.
metadata:
  openclaw:
    emoji: "🪟"
    os: ["win32"]
    requires:
      bins: ["pwsh", "wt"]
    install:
      - id: "winget"
        kind: "winget"
        os: ["win32"]
        packageId: "Microsoft.WindowsTerminal"
        bins: ["wt"]
        label: "Install Windows Terminal (winget)"
---

# Windows Terminal

Drive Windows Terminal — the tabbed multi-shell host — from PowerShell. The CLI is `wt.exe`; configuration is a JSON file.

## When to Use

- ✅ "Open Windows Terminal with my Dev profile"
- ✅ "Split the pane and run `wsl` on the right"
- ✅ "Open three tabs: pwsh, wsl, cmd"
- ✅ Launch a terminal at a specific working directory or with a fixed title
- ✅ Programmatically edit `settings.json` to add a profile or color scheme

## When NOT to Use

- ❌ Sending keystrokes / text to an already-running terminal session — `wt.exe` has no `send-keys`-style verb. Use the `windows-ui` skill (UI Automation / SendInput) instead.
- ❌ Reading output back from a running pane — same; use UI automation, or pipe the inner command's stdout to a file.
- ❌ Driving the old `conhost.exe` (the classic black box) — `wt.exe` only controls Windows Terminal windows.
- ❌ Cross-platform terminal automation — `wt.exe` is Windows-only. For Linux/macOS use `tmux`.

## Setup

Windows 11 ships with Windows Terminal preinstalled. On Windows 10 install it:

```powershell
winget install --id Microsoft.WindowsTerminal
```

Verify the binary:

```powershell
(Get-Command wt.exe).Source
# C:\Users\<you>\AppData\Local\Microsoft\WindowsApps\wt.exe
```

`wt.exe` is a Store app launcher stub — its own `--help` is shown in a Terminal window, not piped to stdout. Reference the official subcommand list below or run `wt -? ;` inside a Terminal session.

## CLI Basics

`wt.exe` parses its arguments into a sequence of **commands** separated by `;`. Because `;` is also a PowerShell statement separator, escape it with a backtick or wrap in single quotes:

```powershell
wt.exe new-tab `; split-pane -V `; split-pane -H
# or
& wt.exe 'new-tab ; split-pane -V ; split-pane -H'
```

From `cmd.exe` no escaping is needed:

```cmd
wt new-tab ; split-pane -V ; split-pane -H
```

### Window targeting

| Flag | Meaning |
|------|---------|
| `-w 0` / `--window 0` | Use the currently focused Terminal window |
| `-w new` | Always make a new window |
| `-w <id>` | A specific window by integer ID |
| `-w <name>` | A specific named window (set via `windowName`) |

Example — open a new tab in the existing window:

```powershell
wt -w 0 new-tab -p "PowerShell"
```

### Common subcommands

| Subcommand | What it does |
|------------|--------------|
| `new-tab` | Open a new tab (default if no subcommand given) |
| `split-pane` | Split the active pane (`-H` horizontal, `-V` vertical; defaults to vertical when neither flag is given) |
| `focus-tab` | Move focus by index (`-t <n>`) or relative (`--next`, `--previous`) |
| `move-focus` | Move focus between panes (positional: `up`, `down`, `left`, `right`, `first`, `last`, `previous`) |
| `swap-pane` | Swap the active pane with the pane in the given direction |
| `focus-pane` | Focus a pane by `-t <n>` |

### Common options (most subcommands accept)

| Flag | Meaning |
|------|---------|
| `-p "<profile>"` / `--profile` | Launch with a named profile (see settings.json) |
| `-d <path>` / `--startingDirectory` | Working directory for the new pane |
| `--title "<text>"` | Initial tab/pane title |
| `--tabColor "#RRGGBB"` | Tab strip color |
| `--colorScheme "<name>"` | Override color scheme |
| `-s <n>` / `--size <n>` | For `split-pane`: size of the new pane as a fraction (e.g., `0.3`) |
| `<command...>` | Trailing positional becomes the command to run in the new pane |

## Examples

### Open a tab with a specific profile

```powershell
wt.exe -w 0 new-tab -p "PowerShell"
wt.exe -w 0 new-tab -p "Ubuntu" -d "\\wsl.localhost\Ubuntu\home\me"
```

The profile name must match the `name` field in `settings.json` exactly.

### Three tabs at once (new window)

```powershell
wt.exe new-tab -p "PowerShell" `; new-tab -p "Command Prompt" `; new-tab -p "Ubuntu"
```

### Tab with a vertical split

```powershell
wt.exe new-tab -p "PowerShell" `; split-pane -V -p "Ubuntu"
```

### Run a one-off command in a new pane and exit when done

The trailing positional becomes the inner command:

```powershell
wt.exe -w 0 new-tab -p "PowerShell" pwsh.exe -NoExit -Command "Get-Process | Sort CPU -Desc | Select -First 10"
```

`-NoExit` keeps the pane open after the command finishes. Drop it to have the pane close.

### Focus tab 2 of the current window

```powershell
wt.exe -w 0 focus-tab -t 1     # 0-indexed: -t 1 == second tab
```

### Make a dev layout: editor on left, two stacked terminals on right

```powershell
wt.exe new-tab -p "PowerShell" `
    `; split-pane -V -s 0.5 -p "Ubuntu" `
    `; move-focus right `
    `; split-pane -H -s 0.5 -p "PowerShell"
```

## settings.json

Profiles, color schemes, key bindings, and global defaults live in JSON. Path:

```
%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json
```

(For the unpackaged / Preview / Canary builds the package name changes — list with `Get-ChildItem "$env:LOCALAPPDATA\Packages" -Filter "Microsoft.WindowsTerminal*"`.)

Open the right file from PowerShell:

```powershell
$pkg = (Get-ChildItem "$env:LOCALAPPDATA\Packages" -Filter "Microsoft.WindowsTerminal_*" -Directory | Select-Object -First 1).Name
$settings = "$env:LOCALAPPDATA\Packages\$pkg\LocalState\settings.json"
notepad $settings
```

Or have Terminal open its own settings UI: `Ctrl+,` inside a Terminal window, or:

```powershell
$pkg = (Get-ChildItem "$env:LOCALAPPDATA\Packages" -Filter "Microsoft.WindowsTerminal_*" -Directory | Select-Object -First 1).Name
wt.exe -w 0 -p "PowerShell" pwsh.exe -NoExit -Command "Start-Process notepad '$env:LOCALAPPDATA\Packages\$pkg\LocalState\settings.json'"
```

### Minimal profile shape

```jsonc
{
  "name": "My PowerShell 7",
  "commandline": "pwsh.exe -NoLogo",
  "startingDirectory": "%USERPROFILE%\\Projects",
  "icon": "ms-appx:///ProfileIcons/{61c54bbd-c2c6-5271-96e7-009a87ff44bf}.png",
  "colorScheme": "Campbell Powershell",
  "fontFace": "Cascadia Code",
  "fontSize": 11
}
```

Add it under `profiles.list`. The `name` is what `wt -p "<name>"` matches.

### Edit programmatically (PowerShell)

Always back up first:

```powershell
$pkg = (Get-ChildItem "$env:LOCALAPPDATA\Packages" -Filter "Microsoft.WindowsTerminal_*" -Directory | Select-Object -First 1).Name
$path = "$env:LOCALAPPDATA\Packages\$pkg\LocalState\settings.json"

Copy-Item $path "$path.bak"
$json = Get-Content $path -Raw | ConvertFrom-Json
$newProfile = [pscustomobject]@{
    name              = "My pwsh"
    commandline       = "pwsh.exe -NoLogo"
    startingDirectory = "%USERPROFILE%"
}
$json.profiles.list += $newProfile
# Write without a UTF-8 BOM — Windows Terminal tolerates the BOM but other
# tooling that round-trips this file may not. -Encoding utf8NoBOM is PS 7+.
$serialized = $json | ConvertTo-Json -Depth 32
if ($PSVersionTable.PSVersion.Major -ge 7) {
  $serialized | Set-Content $path -Encoding utf8NoBOM
} else {
  [System.IO.File]::WriteAllText($path, $serialized, (New-Object System.Text.UTF8Encoding $false))
}
```

> [!warning] Windows Terminal's `settings.json` is **JSON with comments** (jsonc). `ConvertFrom-Json` strips comments. If the user has custom `//` comments in their settings, warn them they'll be lost — or do a targeted text-only edit instead.

## Sending Text to a Running Session (advanced)

`wt.exe` itself does not expose a "send keys" subcommand. Options, in order of preference:

1. **Spawn a fresh pane that runs the command you want** (see "Run a one-off command in a new pane" above). This is the canonical pattern — use it unless the user specifically needs to interact with the already-running shell.
2. **Use the `windows-ui` skill** (UI Automation / SendInput) to focus the Terminal window and inject keystrokes. Slow, brittle if focus shifts, but it works for chat-app-style "type this into my terminal" requests.
3. **Drive a real multiplexer inside the pane** — start `tmux` in WSL or `zellij`, and send commands via their CLIs. Far more reliable than UI injection.

Prefer (1). Reach for (2) only when the user explicitly wants to interact with an existing pane and accepts UI-automation caveats.

## Notes

- `wt.exe` always exits immediately after dispatching its commands to the Terminal window. It is not a tail — don't `Wait-Process` on it.
- The `;` separator binds tighter than PowerShell parsing; always escape with backtick or single-quote the whole argument string.
- The released CLI binary is always `wt.exe` — both the stable and Preview channels. Preview installs in a separate package family (`Microsoft.WindowsTerminalPreview_8wekyb3d8bbwe`) and so has its own `settings.json`; discover yours with `Get-ChildItem "$env:LOCALAPPDATA\Packages" -Filter "Microsoft.WindowsTerminal*"`. `wtd.exe` only exists when you build the Terminal from source (it's the dev-build binary), not in any shipped install.
- `wt.exe new-tab` with no profile uses the default profile from `settings.json` (`defaults.defaultProfile`).
- Working directories starting with `\\wsl.localhost\...` are valid for WSL profiles and resolve to a Linux path inside the pane.
- The Microsoft Store name is "Windows Terminal" — `winget install Microsoft.WindowsTerminal` is the supported install path. Don't grab unofficial binaries.
- Tab and pane indices are 0-based with `wt.exe`; the UI shows 1-based tab numbers — off-by-one is a common confusion.
- Settings UI rebuilds `settings.json` on save and may reorder keys; if you machine-edit, expect cosmetic churn next time the user saves through the UI.

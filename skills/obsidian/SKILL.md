---
name: obsidian
description: Work with Obsidian vaults (plain Markdown notes on disk) on macOS, Linux, and Windows. Locate the active vault from Obsidian's own config, read or edit notes, and automate via the `obsidian-cli` companion CLI (binary `notesmd-cli` on Windows/Linux v0.3.x). Use when the user says "open my Obsidian vault", "search my notes", "create a note in Obsidian", "where is my Obsidian config", or wants to script vault edits without launching the desktop app. Obsidian homepage at help.obsidian.md.
metadata:
  openclaw:
    emoji: "💎"
    os: ["darwin", "linux", "win32"]
    requires:
      anyBins: ["obsidian-cli", "notesmd-cli"]
    install:
      - id: "brew"
        kind: "brew"
        os: ["darwin"]
        formula: "yakitrak/yakitrak/obsidian-cli"
        bins: ["obsidian-cli"]
        label: "Install obsidian-cli (brew)"
      - id: "go"
        kind: "go"
        os: ["darwin", "linux", "win32"]
        module: "github.com/Yakitrak/notesmd-cli@latest"
        bins: ["notesmd-cli"]
        label: "Install obsidian-cli / notesmd-cli (go)"
      - id: "download-windows"
        kind: "download"
        os: ["win32"]
        url: "https://github.com/yakitrak/obsidian-cli/releases/download/v0.3.6/notesmd-cli_0.3.6_windows_amd64.tar.gz"
        archive: "tar.gz"
        extract: true
        bins: ["notesmd-cli"]
        label: "Download obsidian-cli (Windows x64)"
      - id: "download-linux"
        kind: "download"
        os: ["linux"]
        url: "https://github.com/yakitrak/obsidian-cli/releases/download/v0.3.6/notesmd-cli_0.3.6_linux_amd64.tar.gz"
        archive: "tar.gz"
        extract: true
        bins: ["notesmd-cli"]
        label: "Download obsidian-cli (Linux x64)"
---

# Obsidian

Obsidian vault = a normal folder on disk. Works the same on every OS the desktop app supports (macOS, Windows, Linux, iOS, Android) because the file format is plain Markdown.

Vault structure (typical):

- Notes: `*.md` (plain text Markdown; edit with any editor)
- Config: `.obsidian/` (workspace + plugin settings; usually don't touch from scripts)
- Canvases: `*.canvas` (JSON)
- Attachments: whatever folder you chose in Obsidian settings (images/PDFs/etc.)

## Find the active vault(s)

Obsidian desktop tracks vaults in a per-user JSON file. Path differs per OS:

| OS | Path to `obsidian.json` |
|---|---|
| macOS | `~/Library/Application Support/obsidian/obsidian.json` |
| Linux | `~/.config/obsidian/obsidian.json` (or `$XDG_CONFIG_HOME/obsidian/obsidian.json`) |
| Windows | `%APPDATA%\obsidian\obsidian.json` |

The schema is the same on every OS: a JSON object with a `vaults` map keyed by id, each entry carrying `path` and `open: true` for the currently-active vault.

Fast "what vault is active / where are the notes?":

- With a default registered (CLI v0.3.x): `obsidian-cli list-vaults` (older v0.2.x: `obsidian-cli print-default --path-only`)
- Otherwise read the JSON directly. Examples:
  ```bash
  # macOS / Linux
  jq -r '.vaults | to_entries[] | select(.value.open) | .value.path' \
    "${XDG_CONFIG_HOME:-$HOME/.config}/obsidian/obsidian.json" 2>/dev/null \
    || jq -r '.vaults | to_entries[] | select(.value.open) | .value.path' \
       "$HOME/Library/Application Support/obsidian/obsidian.json"
  ```
  ```powershell
  # Windows (PowerShell)
  (Get-Content "$env:APPDATA\obsidian\obsidian.json" -Raw | ConvertFrom-Json).vaults.PSObject.Properties |
    Where-Object { $_.Value.open } | ForEach-Object { $_.Value.path }
  ```

Notes:

- Multiple vaults are common (iCloud vs `~/Documents`, work/personal, OneDrive on Windows, etc.). Don't guess — read config.
- Avoid hardcoding vault paths in scripts; prefer reading the config file or invoking `obsidian-cli`.

## obsidian-cli quick start

The CLI is a Go binary that drives Obsidian via its `obsidian://` URI handler and reads/writes vault files directly. Versions and binary names:

| Channel | Binary | Notes |
|---|---|---|
| Homebrew (`brew install yakitrak/yakitrak/obsidian-cli`) | `obsidian-cli` | macOS only. **The tap formula may still track the v0.2.x line** while upstream has moved to v0.3.x — confirm with `brew info` before assuming the commands below work as-is; if the brew install is older, fall back to `go install` or the GitHub release tarball for v0.3.x. |
| `go install github.com/Yakitrak/notesmd-cli@latest` | `notesmd-cli` | macOS / Linux / Windows. The module path was renamed upstream from `github.com/yakitrak/obsidian-cli` to `github.com/Yakitrak/notesmd-cli`; the old path errors with `module declares its path as: github.com/Yakitrak/notesmd-cli but was required as: github.com/yakitrak/obsidian-cli`. |
| GitHub release tarball (`notesmd-cli_*_<os>_<arch>.tar.gz`) | `notesmd-cli` | macOS / Linux / Windows. **The upstream binary was renamed to `notesmd-cli` in v0.3.x**; pre-built releases ship under the new name. Both names accept the same commands. |

> **Heads-up on the v0.2.x → v0.3.x rename.** The vault-management commands changed: `set-default` / `print-default` were replaced by `add-vault` / `list-vaults` / `remove-vault`. The examples below show the v0.3.x form. If you're on an older brew install, run `--help` to confirm the verbs your build supports.

Register a vault (once per machine, v0.3.x):

```
obsidian-cli add-vault "/path/to/MyVault"   # macOS / Linux
notesmd-cli  add-vault "C:\Users\me\Documents\MyVault"  # Windows (note the binary name)
obsidian-cli list-vaults
```

Common commands (same on every OS — only the binary name differs):

- Create: `obsidian-cli create "Folder/New note" --content "..." --open`
  Requires the Obsidian URI handler (`obsidian://...`) — Obsidian must be installed. Avoid creating notes under hidden dot-folders via the URI; Obsidian may refuse.
- Search: `obsidian-cli search "query"` (fuzzy note-name search, opens the match in Obsidian)
- Print contents: `obsidian-cli print "Folder/Note"`
- Move / rename (safe refactor — updates `[[wikilinks]]` and Markdown links): `obsidian-cli move "old/path/note" "new/path/note"`
- Delete: `obsidian-cli delete "path/note"`
- Daily note: `obsidian-cli daily`
- Frontmatter: `obsidian-cli frontmatter <subcommand>`

Prefer direct edits when appropriate: open the `.md` file and change it; Obsidian picks up changes from the filesystem automatically. The CLI is most valuable for **link-aware moves**, **URI-driven create/open**, and **batch automation** where shelling Obsidian itself would be flaky.

## Notes

- The vault folder itself is fully portable across OSes (it's just Markdown + `.obsidian/`). A vault synced via iCloud / Dropbox / OneDrive can be opened by Obsidian on any platform.
- `.obsidian/workspace.json` is per-machine UI state (split layouts, open files). Don't sync it across machines unless you want layout to match — most users gitignore it.
- On Windows, prefer paths under the user profile (`%USERPROFILE%\Documents\...`) for vault folders so they survive reinstalls and roam with the user. Vaults on system folders or behind ACL'd paths often fail to register.

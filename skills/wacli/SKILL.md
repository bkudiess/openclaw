---
name: wacli
description: Send third-party WhatsApp messages, sync history, or search WhatsApp archives via `wacli` on macOS, Linux, and Windows — not normal active chats with the user. Use when the user explicitly asks to message someone else on WhatsApp, search past WhatsApp threads, or backfill chat history; do not reach for `wacli` when the user is just chatting with you on WhatsApp (OpenClaw already routes that automatically). Tool docs at wacli.sh.
metadata:
  openclaw:
    emoji: "📱"
    os: ["darwin", "linux", "win32"]
    requires:
      bins: ["wacli"]
    install:
      - id: "brew"
        kind: "brew"
        os: ["darwin", "linux"]
        formula: "steipete/tap/wacli"
        bins: ["wacli"]
        label: "Install wacli (brew)"
      - id: "go"
        kind: "go"
        os: ["darwin", "linux", "win32"]
        module: "github.com/openclaw/wacli/cmd/wacli@latest"
        bins: ["wacli"]
        label: "Install wacli (go install)"
      - id: "download-windows"
        kind: "download"
        os: ["win32"]
        url: "https://github.com/openclaw/wacli/releases/download/v0.8.1/wacli-windows-amd64.zip"
        archive: "zip"
        extract: true
        bins: ["wacli"]
        label: "Download wacli (Windows x64)"
      - id: "download-linux"
        kind: "download"
        os: ["linux"]
        url: "https://github.com/openclaw/wacli/releases/download/v0.8.1/wacli-linux-amd64.tar.gz"
        archive: "tar.gz"
        extract: true
        bins: ["wacli"]
        label: "Download wacli (Linux x64)"
---

# wacli

Use `wacli` only when the user explicitly asks you to message someone else on WhatsApp or when they ask to sync/search WhatsApp history.
Do NOT use `wacli` for normal user chats; OpenClaw routes WhatsApp conversations automatically.
If the user is chatting with you on WhatsApp, you should not reach for this tool unless they ask you to contact a third party.

## OS notes

| OS | Recommended install | Store dir |
|---|---|---|
| macOS | `brew install steipete/tap/wacli` | `~/.wacli` |
| Linux | `brew install steipete/tap/wacli` (or download prebuilt tarball) | `~/.local/state/wacli` (XDG state dir; legacy `~/.wacli` still works if it already exists) |
| Windows | Download prebuilt zip from the releases page (preferred — no toolchain) **or** `go install github.com/openclaw/wacli/cmd/wacli@latest` (needs Go + a C compiler for cgo, e.g. mingw-w64 via MSYS2) | `%USERPROFILE%\.wacli` |

The `wacli` binary is the same on every OS — flags and subcommands match.

> **Upstream module path moved.** Old documentation pointed at `github.com/steipete/wacli/cmd/wacli`; the canonical module is now `github.com/openclaw/wacli/cmd/wacli`. The old path fails `go install` with "module declares its path as: github.com/openclaw/wacli but was required as: github.com/steipete/wacli".

## Safety

- Require explicit recipient + message text.
- Confirm recipient + message before sending.
- If anything is ambiguous, ask a clarifying question.

## Auth + sync

- `wacli auth` (QR login + initial sync)
- `wacli sync --follow` (continuous sync)
- `wacli doctor`

## Find chats + messages

- `wacli chats list --limit 20 --query "name or number"`
- `wacli messages search "query" --limit 20 --chat <jid>`
- `wacli messages search "invoice" --after 2025-01-01 --before 2025-12-31`

## History backfill

- `wacli history backfill --chat <jid> --requests 2 --count 50`

## Send

- Text: `wacli send text --to "+14155551212" --message "Hello! Are you free at 3pm?"`
- Group: `wacli send text --to "1234567890-123456789@g.us" --message "Running 5 min late."`
- File: `wacli send file --to "+14155551212" --file /path/agenda.pdf --caption "Agenda"`

## Notes

- Store dir defaults: `~/.wacli` on macOS, `~/.local/state/wacli` on Linux (XDG state dir; legacy `~/.wacli` is still honored if the directory pre-exists), `%USERPROFILE%\.wacli` on Windows. Override with `--store DIR` or `WACLI_STORE_DIR`.
- Use `--json` for machine-readable output when parsing.
- Backfill requires your phone online; results are best-effort.
- WhatsApp CLI is not needed for routine user chats; it's for messaging other people.
- JIDs: direct chats look like `<number>@s.whatsapp.net`; groups look like `<id>@g.us` (use `wacli chats list` to find).
- Windows ARM64 hosts: the prebuilt `wacli-windows-amd64.zip` runs under Windows-on-ARM x64 emulation. Native ARM64 builds aren't published yet; `go install` on Windows needs a working cgo toolchain (`go-sqlite3` is a cgo dependency) — install mingw-w64 first (e.g. via MSYS2) or use the prebuilt zip.

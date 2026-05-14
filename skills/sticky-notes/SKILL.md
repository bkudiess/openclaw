---
name: sticky-notes
description: Read Microsoft Sticky Notes (the UWP app) on Windows by querying its local SQLite database. List notes, search note text, dump by color, and view timestamps. Use when the user asks to "read my sticky notes", "find a sticky note about X", "list my notes from Sticky Notes", or "show recent sticky notes". Read-only — writing notes requires the UWP API and is out of scope.
metadata:
  openclaw:
    emoji: "🗒️"
    os: ["win32"]
    requires:
      bins: ["pwsh", "sqlite3"]
    install:
      - id: "sqlite"
        kind: "winget"
        os: ["win32"]
        packageId: "SQLite.SQLite"
        bins: ["sqlite3"]
        label: "Install SQLite CLI (winget)"
---

# Sticky Notes (read-only)

Microsoft Sticky Notes is the built-in UWP note app on Windows. Its data lives
in a local SQLite database. This skill reads that database directly.

> **Read-only.** Writing or deleting notes requires the UWP `Microsoft.MicrosoftStickyNotes`
> app's own API surface (or driving its UI). Modifying the SQLite file while the
> app is running can corrupt state. **Do not write to it.**

## When to Use

✅ **USE this skill when:**

- User asks to read, list, search, or summarize their Sticky Notes
- Exporting Sticky Notes content to text/markdown
- Finding the note that mentions "X"

## When NOT to Use

❌ **DON'T use this skill when:**

- The user wants to **create or edit** a sticky note → launch the UWP app
  (`start ms-stickynotes:`) and let the user type, or use Microsoft Graph if
  they have a OneNote/Outlook-Notes alternative
- The user wants notes from OneNote, Notion, Obsidian → wrong app
- Sticky Notes isn't installed on this box (no package folder) → tell the user
  and stop

## Setup

- **Sticky Notes app**: ships with Windows 11; if missing,
  `Get-AppxPackage Microsoft.MicrosoftStickyNotes` returns nothing. Install
  from the Microsoft Store.
- **sqlite3 CLI**: `winget install SQLite.SQLite` (provides `sqlite3.exe`).
- The DB lives at:

  ```text
  %LOCALAPPDATA%\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
  ```

  The publisher hash `8wekyb3d8bbwe` is Microsoft's; use a wildcard for safety.

## Locate the Database

```powershell
$pkg = Get-ChildItem $env:LOCALAPPDATA\Packages -Filter 'Microsoft.MicrosoftStickyNotes_*' |
       Select-Object -First 1
if (-not $pkg) {
  Write-Error "Microsoft Sticky Notes is not installed."
  return
}
$db = Join-Path $pkg.FullName "LocalState\plum.sqlite"
if (-not (Test-Path $db)) {
  Write-Error "Sticky Notes DB not found at $db"
  return
}
"DB: $db ($((Get-Item $db).Length) bytes)"
```

## Read Safely (copy first)

The DB is usually open by `StickyNotes.exe`. Always copy it to a temp path and
query the copy — never query the live file (risk of locks, partial reads).

Prefer the SQLite online backup API — it is atomic against a hot DB and avoids
the torn-snapshot risk that plain file copies have:

```powershell
$tmp = Join-Path $env:TEMP ("plum-{0}.sqlite" -f [guid]::NewGuid())
sqlite3 $db ".backup '$tmp'"
```

If `sqlite3.exe` isn't on PATH, fall back to a manual sidecar copy (best-effort —
the snapshot can be torn if Sticky Notes writes mid-copy; retry on
`"database is locked"` / `"file is not a database"`):

```powershell
$tmp = Join-Path $env:TEMP ("plum-{0}.sqlite" -f [guid]::NewGuid())
Copy-Item $db $tmp -Force
# Also copy the WAL / SHM sidecar files so the copy sees latest committed state.
# Sidecar names must match the main file (so plum.sqlite-wal next to plum.sqlite,
# which lands as ${tmp}-wal next to ${tmp}).
foreach ($suffix in @("-wal", "-shm")) {
  $sidePath = "$db$suffix"
  if (Test-Path $sidePath) { Copy-Item $sidePath "$tmp$suffix" -Force }
}
```

## Inspect Schema

The schema has changed over Sticky Notes versions. Discover the actual tables
before querying — don't hard-code names.

```powershell
# List tables
sqlite3 $tmp ".tables"

# Show schema for the main note table (often `Note` or `Notes`)
sqlite3 $tmp ".schema"
```

Typical layout in modern builds:

- `Note` — one row per note. Useful columns: `Id`, `Text` (or `PlainText`),
  `Theme` (color), `IsDeleted` / `IsVisible`, `UpdatedAtTime`, `CreatedAtTime`.
- Older versions used different column names; always introspect first.

## List Notes

```powershell
# Replace column names with what `.schema Note` shows
sqlite3 -separator "`t" $tmp @"
SELECT
  Id,
  COALESCE(Theme, '') AS Color,
  datetime(UpdatedAtTime / 10000000 - 11644473600, 'unixepoch', 'localtime') AS Updated,
  substr(replace(replace(Text, char(10), ' '), char(13), ' '), 1, 80) AS Preview
FROM Note
WHERE IsDeleted = 0
ORDER BY UpdatedAtTime DESC;
"@
```

> Sticky Notes stores timestamps as **Windows FILETIME** (100-ns ticks since
> 1601-01-01 UTC). Convert with `time/10000000 - 11644473600`. If a column is
> stored as ISO-8601 text instead (newer builds sometimes do), use it directly.

## Search Note Text

```powershell
$query = "design review"
sqlite3 -separator "`t" $tmp @"
SELECT
  Id,
  datetime(UpdatedAtTime / 10000000 - 11644473600, 'unixepoch', 'localtime') AS Updated,
  Text
FROM Note
WHERE IsDeleted = 0 AND Text LIKE '%$($query -replace "'", "''")%'
ORDER BY UpdatedAtTime DESC;
"@
```

## Group by Color

```powershell
sqlite3 -separator "`t" $tmp @"
SELECT Theme AS Color, COUNT(*) AS NoteCount
FROM Note
WHERE IsDeleted = 0
GROUP BY Theme
ORDER BY NoteCount DESC;
"@
```

## Powershell-only (System.Data.SQLite) alternative

If `sqlite3.exe` is unavailable, use the `PSSQLite` module — pure PowerShell
wrapper around System.Data.SQLite:

```powershell
Install-Module PSSQLite -Scope CurrentUser -Force
Import-Module PSSQLite

$rows = Invoke-SqliteQuery -DataSource $tmp -Query @"
SELECT Id, Text, UpdatedAtTime
FROM Note
WHERE IsDeleted = 0
ORDER BY UpdatedAtTime DESC
"@
$rows | Format-Table -AutoSize
```

## Notes

- **Schema is not stable.** Microsoft has reshaped the DB across releases
  (`Note` vs `Notes`, `Text` vs `PlainText`, FILETIME vs ISO-8601). Always run
  `.schema` first; do **not** assume column names.
- **Empty DB is normal** on a freshly-installed system or one that's never used
  Sticky Notes — `.tables` may return nothing. Tell the user the app exists but
  has no data.
- **Don't query the live file.** Copy to TEMP first. Also copy the `-wal` and
  `-shm` sidecar files if present so the snapshot reflects the latest committed
  state.
- **Read-only.** Writing to `plum.sqlite` while the app is open will corrupt
  state. To create a note from automation, drive the UI (e.g. via
  `start ms-stickynotes:` then SendKeys / UI Automation) instead.
- **Sync state.** If the user has Sticky Notes synced to their Microsoft
  account, the canonical store is the cloud (Outlook Notes). Local SQLite is
  the latest synced snapshot for that device only.
- **Account folders.** Some builds keep per-account subfolders under
  `LocalState`. Enumerate before assuming a single DB:
  `Get-ChildItem $pkg\LocalState -Recurse -Filter plum.sqlite`.

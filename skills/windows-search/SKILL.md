---
name: windows-search
description: Query the Windows Search index from PowerShell using its OLE DB provider. Use when the user asks to "find files by content", "search for text in my documents", "use Windows Search programmatically", "query the search index", or "search by file property" (kind, size, author, date modified). Covers the Search.CollatorDSO OLE DB provider, the SystemIndex catalog, SQL syntax against SystemIndex, common system properties, and how to verify / start the Windows Search service.
metadata:
  openclaw:
    emoji: "🔎"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
---

# Windows Search

Run SQL queries against the local Windows Search index from PowerShell via the built-in `Search.CollatorDSO` OLE DB provider. Faster than walking the filesystem because Windows Search has already indexed file names, contents, and ~hundreds of properties (`System.*`).

## When to Use

- ✅ "Find PDFs containing 'invoice' under my Documents folder"
- ✅ "Files modified in the last 7 days" (no recursive directory walk)
- ✅ Search by file property: kind, author, size, dimensions, music artist
- ✅ Programmatic equivalent of typing in the Start menu / File Explorer search box
- ✅ Any time `Get-ChildItem -Recurse` would be too slow

## When NOT to Use

- ❌ Searching paths Windows Search doesn't index (e.g., a network share or a folder explicitly excluded). Use `Get-ChildItem` + `Select-String` instead.
- ❌ Searching inside a single file — use `Select-String` / `Get-Content`.
- ❌ Searching Outlook mail / Calendar — that's a separate catalog; query via Outlook automation.
- ❌ Indexing new locations — that's a Control Panel setting (`control.exe srchadmin.dll`), not a query.

## Setup

The OLE DB provider ships with Windows. Verify the indexer service is running:

```powershell
Get-Service -Name WSearch | Select-Object Name, Status, StartType
# Name     Status  StartType
# ----     ------  ---------
# WSearch  Running Automatic
```

If `Status` is `Stopped` or `StartType` is `Disabled`, the queries below will fail with `Provider cannot be found` or return zero rows. Start it (admin shell):

```powershell
Set-Service -Name WSearch -StartupType Automatic
Start-Service -Name WSearch
```

> [!info] On some hardened / Server Core installs Windows Search is uninstalled outright. Check with `Get-WindowsOptionalFeature -Online -FeatureName SearchEngine-Client-Package`. If `State` is `Disabled`, the OLE DB provider is unavailable.

## Quickstart

```powershell
$query = @"
SELECT TOP 10 System.ItemName, System.ItemPathDisplay
FROM SystemIndex
WHERE SCOPE='file:C:\Users\$env:USERNAME'
  AND System.ItemName LIKE '%.txt'
"@

$conn = New-Object System.Data.OleDb.OleDbConnection "Provider=Search.CollatorDSO;Extended Properties='Application=Windows';"
$conn.Open()
$cmd = $conn.CreateCommand()
$cmd.CommandText = $query
$reader = $cmd.ExecuteReader()
while ($reader.Read()) {
    [pscustomobject]@{
        Name = $reader[0]
        Path = $reader[1]
    }
}
$reader.Close()
$conn.Close()
```

## Connection String

Always exactly this:

```
Provider=Search.CollatorDSO;Extended Properties='Application=Windows';
```

The `Extended Properties='Application=Windows'` part is mandatory — it tells the provider to use the user-visible catalog (`SystemIndex`).

## SQL Dialect

Windows Search SQL is a restricted ISO/IEC 9075 subset. Tables: only `SystemIndex`. Columns are property names from the [Windows Property System](https://learn.microsoft.com/en-us/windows/win32/properties/props) (prefixed `System.*`).

Key operators:

- `LIKE 'pattern%'` — wildcard on text properties (works on `System.ItemName`, paths)
- `CONTAINS(column, '"phrase" OR word')` — full-text contains, supports boolean operators
- `FREETEXT(column, 'natural language query')` — relevance-ranked
- `=`, `!=`, `<`, `>`, `<=`, `>=` — for numbers and dates
- `SCOPE='file:<path>'` — restrict to a folder (recursive)
- `DIRECTORY='file:<path>'` — restrict to immediate children only (non-recursive)

### Useful columns

| Property | Type | Meaning |
|----------|------|---------|
| `System.ItemName` | string | File name with extension |
| `System.ItemNameDisplay` | string | Display name |
| `System.ItemPathDisplay` | string | Full path |
| `System.ItemUrl` | string | URL form (`file:C:/...`) |
| `System.ItemType` | string | `.txt`, `.docx`, ... |
| `System.Kind` | array | `document`, `picture`, `music`, `video`, `email`, `folder`, ... |
| `System.Size` | int64 | Bytes |
| `System.DateModified` | datetime | Last-write time |
| `System.DateCreated` | datetime | Creation time |
| `System.Author` | array | Author(s) |
| `System.Title` | string | Document title |
| `System.Music.Artist` | array | |
| `System.Image.Dimensions` | string | `1920 x 1080` |

## Examples

### Find text inside files

```powershell
$query = @"
SELECT System.ItemPathDisplay
FROM SystemIndex
WHERE SCOPE='file:C:\Users\$env:USERNAME\Documents'
  AND CONTAINS('"quarterly report"')
"@
```

`CONTAINS` without a column name searches the full-text content + all text properties.

### Find by kind + date

```sql
SELECT System.ItemPathDisplay, System.DateModified
FROM SystemIndex
WHERE SCOPE='file:C:\Users\$env:USERNAME'
  AND System.Kind = SOME ARRAY['picture']
  AND System.DateModified > '2025-01-01'
ORDER BY System.DateModified DESC
```

(Always build SCOPE with `$env:USERNAME` from PowerShell so the path is real for the calling user — a literal `C:\Users\you` returns zero rows.)

### Find recent large videos

```sql
SELECT TOP 20 System.ItemPathDisplay, System.Size
FROM SystemIndex
WHERE System.Kind = SOME ARRAY['video']
  AND System.Size > 524288000          -- 500 MB
ORDER BY System.DateModified DESC
```

### Free-text relevance search

```sql
SELECT TOP 25 System.ItemPathDisplay, RANK
FROM SystemIndex
WHERE FREETEXT('annual review goals')
ORDER BY RANK DESC
```

`RANK` is a pseudo-column returned only with `FREETEXT` / `CONTAINS` queries.

### Reusable helper function

```powershell
function Invoke-WindowsSearch {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Sql
    )
    $conn = New-Object System.Data.OleDb.OleDbConnection "Provider=Search.CollatorDSO;Extended Properties='Application=Windows';"
    try {
        $conn.Open()
        $cmd = $conn.CreateCommand()
        $cmd.CommandText = $Sql
        $reader = $cmd.ExecuteReader()
        $cols = 0..($reader.FieldCount-1) | ForEach-Object { $reader.GetName($_) }
        while ($reader.Read()) {
            $row = [ordered]@{}
            foreach ($i in 0..($reader.FieldCount-1)) { $row[$cols[$i]] = $reader[$i] }
            [pscustomobject]$row
        }
        $reader.Close()
    }
    finally {
        $conn.Close()
    }
}

Invoke-WindowsSearch "SELECT TOP 5 System.ItemPathDisplay FROM SystemIndex WHERE System.ItemName LIKE 'README%'"
```

## Path Syntax

`SCOPE` / `DIRECTORY` expect a `file:` URL with backslashes (yes, backslashes — not forward slashes):

```sql
SCOPE='file:C:\Users\you\Documents'
```

In PowerShell single-quoted strings or here-strings, backslashes are literal — no escaping needed. In a PowerShell double-quoted string, `$` still expands, but `\` doesn't need escaping:

```powershell
$sql = "... SCOPE='file:C:\Users\$env:USERNAME\Documents' ..."
```

## Notes

- Indexed locations are configured in Control Panel → Indexing Options (`control.exe srchadmin.dll`). By default that's the user profile, Start menu, and a few app stores — NOT the whole drive. Files outside that scope return zero rows.
- The first query after a service start may be slow (the provider cold-starts). Subsequent queries are fast.
- Error `0x80040E14` usually means a syntax error — check column names match `System.*` exactly (case-insensitive but spelling must be right).
- Error `0x80040154` ("Class not registered") means the indexer is disabled or the SearchEngine-Client-Package optional feature is uninstalled.
- The provider name `Search.CollatorDSO` is for 64-bit PowerShell. 32-bit (`powershell.exe -ExecutionPolicy ... -Sta` from a 32-bit host) needs `Search.CollatorDSO.1`; this rarely matters today.
- Results respect the calling user's NTFS permissions — items they can't read are filtered out.
- For very large result sets, paginate with `TOP N` and incremental `WHERE System.DateModified > @last`; the provider has no cursors.

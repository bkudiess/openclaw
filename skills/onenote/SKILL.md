---
name: onenote
description: Read and write OneNote notebooks, sections, and pages on Windows via the Microsoft Graph CLI (`mgc`). List notebooks/sections/pages, get a page's HTML content, create new pages, update/append to pages, and search across all notes. Use when the user says "find my notes about X", "add a note to OneNote", "show me my OneNote notebooks", "paste this into my Project Y section", or wants headless/cloud OneNote access. This is the Windows equivalent of the macOS `apple-notes` skill — prefer this for Microsoft 365 work/school content; for local-only OneNote 2016 notebooks, COM `OneNote.Application` is documented below.
metadata:
  openclaw:
    emoji: "📓"
    os: ["win32"]
    requires:
      bins: ["pwsh", "mgc"]
    install:
      - id: "mgc-download"
        kind: "download"
        os: ["win32"]
        url: "https://github.com/microsoftgraph/msgraph-cli/releases/download/v1.9.0/msgraph-cli-win-x64-1.9.0.zip"
        archive: "zip"
        extract: true
        targetDir: "mgc"
        bins: ["mgc"]
        label: "Download Microsoft Graph CLI (mgc) for Windows"
---

# OneNote (via Microsoft Graph CLI)

Use the **Microsoft Graph CLI** (`mgc`) to manage OneNote notebooks stored in Microsoft 365 / OneDrive (work, school, or personal Microsoft account). The OneNote Graph endpoint exposes Notebooks → Sections → Pages, where each page's content is HTML.

## When to Use

✅ **USE this skill when:**

- User says "show me my OneNote notebooks" / "list sections in X"
- User says "find my notes about ..." (search across all pages)
- User wants to **create** a page or **append** to an existing page
- User wants to read a page's content programmatically
- Notes live in OneDrive / SharePoint (work/school account)

## When NOT to Use

❌ **DON'T use this skill when:**

- User is on macOS Notes → `apple-notes`
- User wants Obsidian Markdown → `obsidian`
- User wants Bear Notes → `bear-notes`
- User wants tasks/reminders → `mstodo`
- The notebook is a **local-only** OneNote 2016 file (`.one` on disk, no OneDrive sync) — Graph cannot see it; use the COM fallback at the bottom of this file
- The user wants to attach a large media file (>20 MB) — Graph has size limits on the `multipart/form-data` page upload; chunked uploads via OneDrive `createUploadSession` are a separate workflow

## Setup

`mgc` is the official Microsoft Graph CLI. There is **no winget package**. Install from the GitHub release ZIP:

```powershell
$dst = "$env:LOCALAPPDATA\Programs\msgraph-cli"
New-Item -ItemType Directory -Force -Path $dst | Out-Null
$zip = "$env:TEMP\msgraph-cli-win.zip"
Invoke-WebRequest `
  "https://github.com/microsoftgraph/msgraph-cli/releases/download/v1.9.0/msgraph-cli-win-x64-1.9.0.zip" `
  -OutFile $zip
Expand-Archive $zip -DestinationPath $dst -Force
$env:Path = "$dst;$env:Path"
mgc --version
```

### Sign in

```powershell
mgc login --scopes "Notes.ReadWrite User.Read"
```

For read-only workflows use `Notes.Read`. For shared notebooks across the tenant add `Notes.Read.All` (admin consent required). Token cache: MSAL stores tokens in a DPAPI-encrypted file under `%LOCALAPPDATA%\.IdentityService\`; `mgc logout` clears it. Re-login is required to add new scopes — pass the full set each time. Multi-tenant or multi-account: pin a tenant with `--tenant-id <guid|domain>`; only one cached account at a time.

## Discovering IDs

Every page operation needs a `--onenote-page-id`. Every page creation needs an `--onenote-section-id`. Resolve top-down:

```powershell
# Notebooks
mgc users onenote notebooks list --user-id me `
  --select id,displayName,isDefault --top 25

# Sections inside a notebook
mgc users onenote notebooks sections list --user-id me `
  --notebook-id $notebookId --select id,displayName

# Pages inside a section
mgc users onenote sections pages list --user-id me `
  --onenote-section-id $sectionId --select id,title,lastModifiedDateTime --top 50

# Or all pages across the user, newest first
mgc users onenote pages list --user-id me `
  --orderby "lastModifiedDateTime desc" --top 25
```

## Common Workflows

### Read a page's HTML content

```powershell
mgc users onenote pages content get --user-id me `
  --onenote-page-id $pageId --output-file page.html
```

- Content is **HTML**, not Markdown. Strip with `HtmlAgilityPack` or a quick regex for plaintext extraction.
- The HTML contains `data-id` attributes on each element — keep these if you plan to PATCH the page later (see "Update a page").

### Search pages

```powershell
mgc users onenote pages list --user-id me `
  --search "quarterly review" --select id,title,parentNotebook,parentSection `
  --top 25
```

`--search` runs server-side full-text search across title + body. Use `--filter` for structured queries (e.g. `--filter "contains(title,'standup')"`).

### Create a new page

Pages are created by POSTing **HTML** to a section. The HTML must include a `<title>` element — that becomes the page title.

```powershell
$html = @'
<!DOCTYPE html>
<html>
  <head>
    <title>Sprint 42 notes</title>
    <meta name="created" content="2026-02-14T10:00:00-08:00" />
  </head>
  <body>
    <h1>Sprint 42 standup</h1>
    <p>Attendees: Alice, Bob, Carol</p>
    <ul>
      <li>Shipped feature X</li>
      <li>Blocked on review for Y</li>
    </ul>
  </body>
</html>
'@
$html | Out-File -Encoding utf8 page.html

mgc users onenote sections pages create --user-id me `
  --onenote-section-id $sectionId `
  --body "@page.html" `
  --headers "Content-Type=text/html"
```

> **Important:** unlike most Graph endpoints, this one accepts `text/html` (or `application/xhtml+xml`) directly — set the `Content-Type` header explicitly. To embed images and attachments, switch to a `multipart/form-data` body; see the Graph reference for `image:1`-style multipart parts.

### Append to an existing page

PATCHing OneNote pages is unusual: the body is a **JSON array** of commands, each targeting an element by `data-id`. Use `action: append` to add to the end:

```powershell
$cmds = @'
[
  {
    "target": "body",
    "action": "append",
    "position": "after",
    "content": "<p>One more note: deployed at 3pm PT.</p>"
  }
]
'@
$cmds | Out-File -Encoding utf8 patch.json

mgc users onenote pages patch --user-id me --onenote-page-id $pageId `
  --body "@patch.json" --headers "Content-Type=application/json"
```

Other useful `action` values: `replace`, `insert`, `prepend`, `delete`. `target` is either `body`, `title`, or a `data-id` you read from the page HTML.

### Delete a page (destructive)

```powershell
mgc users onenote pages delete --user-id me --onenote-page-id $pageId
```

Deleted pages go to **OneNote → Recycle Bin** in the notebook for ~60 days, but treat as permanent for safety messaging.

### Create a section in a notebook

```powershell
'{ "displayName": "Customer X" }' | Out-File -Encoding utf8 section.json
mgc users onenote notebooks sections create --user-id me `
  --notebook-id $notebookId --body "@section.json"
```

### JSON-friendly piping

`mgc` defaults to JSON output. Useful pipelines:

```powershell
# Title -> ID map
$pages = mgc users onenote pages list --user-id me --top 100 `
  --select id,title | ConvertFrom-Json
$pages.value | ForEach-Object { "{0}`t{1}" -f $_.id, $_.title }

# JMESPath at the wire
mgc users onenote notebooks list --user-id me `
  --query "value[?isDefault].{id:id,name:displayName}"
```

## COM fallback (local-only OneNote notebooks)

If the user is on the **classic OneNote desktop app** (the one branded simply "OneNote", bundled with current Microsoft 365 Apps) with notebooks that are not synced to OneDrive, Graph can't see them — drop down to COM. The COM `OneNote.Application` automation surface is only exposed by this classic OneNote, not by the retired UWP "OneNote for Windows 10".

```powershell
$on = New-Object -ComObject OneNote.Application
[xml]$hierarchy = $null
# 0 = hsNotebooks, 4 = hsPages
$on.GetHierarchy("", 4, [ref]$hierarchy)
$hierarchy.Notebooks.Notebook |
  ForEach-Object { "{0}`t{1}" -f $_.ID, $_.name }
```

Then `GetPageContent($pageId, [ref]$xml)` to read, `UpdatePageContent($xml)` to write. The COM XML schema is documented at `learn.microsoft.com/office/client-developer/onenote/onenote-developer-reference`.

Prefer Graph (`mgc`) whenever the notebook is in OneDrive — it works from headless sessions, scales across machines, and uses standard auth.

## Safety Rules

1. **Confirm before deleting pages or sections.** Recycle Bin retention is finite and notebook owners may not check it.
2. **Read titles back to the user** before bulk operations ("found 12 pages titled 'Sprint *' — delete all?").
3. **Don't paste OneNote content into chat unredacted** — pages often contain customer names, credentials in code blocks, or meeting notes marked confidential.
4. **Throttle yourself.** Graph returns `429 Too Many Requests` after ~10 page creations/sec. Honor `Retry-After`.

## Notes

- **HTML, not Markdown.** Every page is HTML on the wire. When generating content from Markdown, convert with `pandoc -f markdown -t html` first.
- **Time zones.** `<meta name="created">` accepts ISO-8601 with offset.
- **Image embedding.** Inline base64 images work for small ones; for >2 MB use multipart with `image:N` parts.
- **Personal vs work accounts.** A personal Microsoft account uses `https://www.onenote.com/api/v1.0` historically — `mgc` papers over this and uses Graph v1.0 for both, but some endpoints (e.g. `siteCollections`) are work/school only.
- **Cycle:** read → modify → write requires keeping the `data-id` attributes from the GET response, otherwise PATCH targets won't resolve.

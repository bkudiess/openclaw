---
name: word-com
description: Automate the Microsoft Word desktop app on Windows via COM (Word.Application). Open documents, read text and paragraphs, find/replace, insert at bookmarks, accept or reject tracked changes, run mail merge, and export to PDF or other formats. Use when the user asks to "read this docx", "find/replace in document X", "fill bookmarks", "export Word to PDF", "accept all tracked changes", or any Word desktop automation. Requires Microsoft 365 / Word desktop installed.
metadata:
  openclaw:
    emoji: "📝"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
    install:
      - id: "office"
        kind: "winget"
        os: ["win32"]
        packageId: "Microsoft.Office"
        bins: ["WINWORD"]
        label: "Install Microsoft 365 (winget)"
---

# Word COM Automation

Drive the **Word desktop app** on Windows through its COM interface
(`Word.Application`). Word must be installed and licensed. Use this skill for
operations that benefit from Word's own engine: format-preserving find/replace,
tracked-changes acceptance, real `.doc` (legacy binary) reads, mail merge, and
PDF export with native Word fidelity.

## When to Use

✅ **USE this skill when:**

- Reading or editing existing `.doc` / `.docx` documents while preserving formatting
- Word-quality find & replace (with wildcards, formatting, MatchCase, etc.)
- Inserting/replacing content at bookmarks
- Accepting or rejecting tracked changes
- Mail merge (Word data sources, .mdb/.xlsx/.csv)
- Exporting `.docx` → `.pdf` with Word's renderer (handles complex tables, equations)

## When NOT to Use

❌ **DON'T use this skill when:**

- Pure programmatic generation with no Word features needed → use `python-docx`
  or similar; faster, no Word install
- Headless server with no Word license → COM cannot run
- macOS → Word for Mac supports AppleScript; this skill is Windows-only
- PDF-only output from scratch (no Word features) → use a PDF library

## Setup

- Microsoft 365 / Office (Word desktop) installed: `winget install Microsoft.Office`
- PowerShell 7 (`pwsh`) recommended; **STA threading** preferred for any UI / dialog interaction.
- Headless run: set `$Word.Visible = $false` and `$Word.DisplayAlerts = 0`.

## Core Pattern

```powershell
pwsh -STA -Command @'
$Word = $null; $doc = $null; $find = $null; $range = $null
try {
  $Word = New-Object -ComObject Word.Application
  $Word.Visible       = $false
  $Word.DisplayAlerts = 0      # wdAlertsNone

  # ... work — assign $doc / $find / $range as you go ...

} finally {
  foreach ($name in 'range','find','doc') {
    $val = Get-Variable -Name $name -ValueOnly -ErrorAction SilentlyContinue
    if ($val) {
      if ($name -eq 'doc') { try { $val.Close($false) } catch {} }
      [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($val)
      Set-Variable $name $null
    }
  }
  if ($Word) {
    $Word.Quit()
    [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($Word)
  }
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
}
'@
```

## Open a Document

```powershell
# Read-write
$doc = $Word.Documents.Open("C:\path\to\doc.docx")

# Read-only (FileName, ConfirmConversions, ReadOnly)
$doc = $Word.Documents.Open("C:\path\to\doc.docx", $false, $true)

# New empty document
$doc = $Word.Documents.Add()
```

## Read Full Text

```powershell
$txt = $doc.Content.Text   # all text, including paragraph marks
$txt | Set-Content out.txt -Encoding UTF8
```

## Iterate Paragraphs

```powershell
$i = 0
foreach ($p in $doc.Paragraphs) {
  $i++
  $line = $p.Range.Text.TrimEnd("`r","`n","`a")
  if ($line) { "{0,3}: {1}" -f $i, $line }
}
```

## Find & Replace (the reliable PowerShell pattern)

Word's `Find.Execute` takes **11 positional args** and PowerShell does **not**
fill defaults reliably. Set Find properties explicitly, then pass all 11
parameters by `[ref]`. The 11th parameter is `Replace` (2 = `wdReplaceAll`).

```powershell
$find = $doc.Content.Find
$find.ClearFormatting()
$find.Replacement.ClearFormatting()
$find.Text             = "PLACEHOLDER"
$find.Replacement.Text = "REPLACED"
$find.Forward          = $true
$find.Wrap             = 1            # wdFindContinue
$find.Format           = $false
$find.MatchCase        = $false
$find.MatchWholeWord   = $false
$find.MatchWildcards   = $false
$find.MatchSoundsLike  = $false
$find.MatchAllWordForms= $false

# Order: FindText, MatchCase, MatchWholeWord, MatchWildcards, MatchSoundsLike,
#        MatchAllWordForms, Forward, Wrap, Format, ReplaceWith, Replace
# Bind every [ref] arg to a real variable — [ref] of a string literal is fragile
# across PowerShell versions and will surface as cryptic InvalidArgument errors.
$findText    = "PLACEHOLDER"
$matchCase   = $false; $wholeWord = $false; $wildcards   = $false
$soundsLike  = $false; $allForms  = $false; $forward     = $true
$wrap        = 1     # wdFindContinue
$format      = $false
$replaceText = "REPLACED"
$replaceMode = 2     # wdReplaceAll

$ok = $find.Execute(
  [ref]$findText, [ref]$matchCase, [ref]$wholeWord, [ref]$wildcards, [ref]$soundsLike,
  [ref]$allForms, [ref]$forward, [ref]$wrap, [ref]$format, [ref]$replaceText, [ref]$replaceMode)

"Find/Replace ran: $ok"
```

Tip: a property-only `$find.Execute()` (no args) finds the **first** match for
the cursor's range. Use that for "find next" navigation.

## Insert at a Bookmark

```powershell
if ($doc.Bookmarks.Exists("Customer")) {
  $bm    = $doc.Bookmarks.Item("Customer")
  $range = $bm.Range
  $range.Text = "Acme Corp"
  # Re-create the bookmark (Range.Text overwrites and consumes the bookmark)
  $null = $doc.Bookmarks.Add("Customer", $range)
}
```

## Tracked Changes — Accept / Reject

```powershell
# Show / hide tracking UI (does not change the data)
$doc.TrackRevisions = $false

# Accept everything
$doc.AcceptAllRevisions()

# Reject everything
$doc.RejectAllRevisions()

# Iterate and accept/reject individually
foreach ($rev in @($doc.Revisions)) {
  if ($rev.Author -eq "Alice") { $rev.Accept() } else { $rev.Reject() }
}
```

## Export to PDF

```powershell
# ExportAsFixedFormat(OutputFileName, ExportFormat=17 for PDF)
$doc.ExportAsFixedFormat("C:\out\final.pdf", 17)
```

Other useful values:

- `17` = `wdExportFormatPDF`
- `18` = `wdExportFormatXPS`

## Save Formats

```powershell
$doc.SaveAs2("C:\out\file.docx", 16)   # 16 = wdFormatDocumentDefault (.docx)
$doc.SaveAs2("C:\out\file.doc",   0)   #  0 = wdFormatDocument         (.doc, legacy)
$doc.SaveAs2("C:\out\file.txt",   2)   #  2 = wdFormatText             (plain UTF-16)
$doc.SaveAs2("C:\out\file.pdf",  17)   # 17 = wdFormatPDF              (since Word 2010)
$doc.SaveAs2("C:\out\file.rtf",   6)   #  6 = wdFormatRTF
```

Prefer `SaveAs2` over the older `SaveAs` — in PowerShell, `SaveAs` often errors
with *"Cannot convert the value … of type 'psobject' to type 'Object'"* because
positional args are not boxed correctly.

## Mail Merge (basics)

```powershell
$doc = $Word.Documents.Open("C:\templates\letter.docx")
$mm  = $doc.MailMerge
$mm.OpenDataSource("C:\data\recipients.xlsx",
  -1, $false, $true, $true, $false, "", "", $false, "", "", "",
  "SELECT * FROM ``Sheet1$``")
$mm.Destination     = 0     # wdSendToNewDocument
$mm.SuppressBlankLines = $true
$mm.Execute($false)
# New doc is now active; save it
$Word.ActiveDocument.SaveAs2("C:\out\merged.docx", 16)
$Word.ActiveDocument.Close($false)
$doc.Close($false)
```

Common `Destination` values: `0` = new document, `1` = printer, `2` = email.

## Complete Worked Example

```powershell
pwsh -STA -Command @'
$Word = $null
try {
  $Word = New-Object -ComObject Word.Application
  $Word.Visible       = $false
  $Word.DisplayAlerts = 0

  # Create + save
  $doc = $Word.Documents.Add()
  $sel = $Word.Selection
  $sel.TypeText("Hello from OpenClaw Word skill.")
  $sel.TypeParagraph()
  $sel.TypeText("PLACEHOLDER")

  $path = Join-Path $env:TEMP "demo.docx"
  $doc.SaveAs2($path, 16)
  $doc.Close()

  # Reopen + replace
  $doc = $Word.Documents.Open($path)
  $f = $doc.Content.Find
  $f.ClearFormatting(); $f.Replacement.ClearFormatting()
  $f.Text = "PLACEHOLDER"; $f.Replacement.Text = "REPLACED"
  $f.Forward = $true; $f.Wrap = 1
  $findText = "PLACEHOLDER"; $replaceText = "REPLACED"
  $matchCase = $false; $wholeWord = $false; $wildcards = $false
  $soundsLike = $false; $allForms = $false; $forward = $true
  $wrap = 1; $format = $false; $replaceMode = 2
  $null = $f.Execute([ref]$findText, [ref]$matchCase, [ref]$wholeWord, [ref]$wildcards,
                     [ref]$soundsLike, [ref]$allForms, [ref]$forward, [ref]$wrap,
                     [ref]$format, [ref]$replaceText, [ref]$replaceMode)

  # Export PDF
  $doc.ExportAsFixedFormat([System.IO.Path]::ChangeExtension($path,".pdf"), 17)
  $doc.Close($true)

} finally {
  foreach ($name in 'f','doc') {
    $val = Get-Variable -Name $name -ValueOnly -ErrorAction SilentlyContinue
    if ($val) {
      if ($name -eq 'doc') { try { $val.Close($false) } catch {} }
      [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($val)
      Set-Variable $name $null
    }
  }
  if ($Word) {
    $Word.Quit()
    [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($Word)
  }
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
}
'@
```

## Safety Rules

1. **Confirm the target path** before opening read-write. Default to read-only
   when only reading.
2. **Copy before mutating** a user document; save changes under a new name and
   let the user diff.
3. **Find/Replace with `wdReplaceAll`** is global and unbounded — confirm the
   replacement text won't match unintended occurrences. Prefer narrower scopes
   (`MatchWholeWord = $true`, `MatchCase = $true`) when possible.
4. **Tracked-changes acceptance is irreversible** without the original. Save a
   copy first, then `AcceptAllRevisions()`.
5. **Macro-enabled docs (`.docm`)** can run code on open. Treat unknown docs as
   untrusted.
6. **Mail merge** can blast email or printer jobs. Default `Destination = 0`
   (new document) and let the user review before sending/printing.

## Notes

- **`SaveAs2`, not `SaveAs`.** `SaveAs` is buggy from PowerShell.
- **STA required** for dialogs, mail merge, and many `Selection`-based ops.
- **Word's `Selection`** is global state. In long scripts prefer `Range` objects
  (`$doc.Content`, `$doc.Paragraphs.Item(n).Range`) to avoid surprises.
- **Headless** = `Visible = $false` + `DisplayAlerts = 0`. Word still spins up a
  full process; expect ~1–2 s startup.
- **Zombie processes:** if `WINWORD.EXE` lingers, you missed `ReleaseComObject` on a
  document, range, or `Find` object. The fix is always to release the missing
  refs — **never** `Stop-Process WINWORD` from a script. The user may have a real
  Word document open (the COM instance is shared) and killing the process can
  destroy unsaved work.
- **`.doc` (legacy)** support is full; Word handles the conversion transparently.
- For **format-agnostic text extraction** of many docs, the COM path is slow.
  Consider `pandoc` or `docx2txt` for batch jobs; use this skill for fidelity.

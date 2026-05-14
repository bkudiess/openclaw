---
name: excel-com
description: Automate the Microsoft Excel desktop app on Windows via COM (Excel.Application). Open workbooks, read and write cell ranges, evaluate formulas, run VBA macros, format cells, export to CSV/PDF, and save in headless mode. Use when the user asks to "read this xlsx", "update cells in workbook X", "run macro Y", "evaluate this formula", "format a sheet", or any Excel desktop automation. Companion to the file-only `xlsx` skill — prefer `xlsx` when the user does not need Excel itself; prefer this when macros, named-range evaluation, or live recalculation are needed.
metadata:
  openclaw:
    emoji: "📊"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
    install:
      - id: "office"
        kind: "winget"
        os: ["win32"]
        packageId: "Microsoft.Office"
        bins: ["EXCEL"]
        label: "Install Microsoft 365 (winget)"
---

# Excel COM Automation

Drive the **Excel desktop app** on Windows through its COM automation interface
(`Excel.Application`). Excel must be installed and licensed. This is the
companion to the file-only `xlsx` skill — use `xlsx` for pure read/write of
`.xlsx` files; use this when you need:

- Macro execution (`$Excel.Run`)
- Live formula evaluation (`$Excel.Evaluate`)
- Live recalculation of dependent cells
- Native Excel features (PivotTables, conditional formatting, charts)
- Format-preserving edits to user-authored workbooks

## When to Use

✅ **USE this skill when:**

- Running VBA macros stored in the workbook
- Evaluating Excel formulas with the real engine (e.g. `XLOOKUP`, `LET`, custom UDFs)
- Reading/writing cells where formulas should recalc as you go
- Exporting a workbook to PDF or formatted CSV
- Modifying an existing workbook while preserving styles/charts/pivots

## When NOT to Use

❌ **DON'T use this skill when:**

- The user just wants to read/write a plain `.xlsx` with no macros or formula
  evaluation → use the `xlsx` skill (faster, no Excel install required)
- Running on a headless server with no Excel license → use openpyxl / xlsx skill
- The workbook lives in OneDrive / SharePoint and needs to be edited via the
  cloud → use Microsoft Graph Excel API
- macOS → Excel for Mac has AppleScript; this skill is Windows-only

## Setup

- Microsoft 365 / Office (Excel desktop) installed: `winget install Microsoft.Office`
- PowerShell 7 (`pwsh`) recommended; **STA threading** preferred:
  `pwsh -STA -Command "..."`
- Headless run: set `$Excel.Visible = $false` and `$Excel.DisplayAlerts = $false`
  to suppress UI and overwrite prompts.

## Core Pattern

Always quit Excel and release **every** COM ref you captured — leaked
Workbook / Worksheet / Range refs leave zombie `EXCEL.EXE` after `$Excel.Quit()`.

```powershell
pwsh -STA -Command @'
$Excel = $null; $wb = $null; $sheet = $null; $range = $null
try {
  $Excel = New-Object -ComObject Excel.Application
  $Excel.Visible        = $false   # headless
  $Excel.DisplayAlerts  = $false   # suppress save/overwrite prompts
  $Excel.ScreenUpdating = $false   # faster for bulk writes

  # ... work — assign $wb / $sheet / $range as you go ...

} finally {
  # Release in reverse creation order, then GC twice (RCWs use a 2-pass cleanup).
  foreach ($name in 'range','sheet','wb') {
    $v = Get-Variable -Name $name -ValueOnly -ErrorAction SilentlyContinue
    if ($v) { [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($v); Set-Variable $name $null }
  }
  if ($Excel) {
    $Excel.Quit()
    [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($Excel)
    $Excel = $null
  }
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
}
'@
```

## Open a Workbook

```powershell
# Read-write
$wb = $Excel.Workbooks.Open("C:\path\to\file.xlsx")

# Read-only (Open signature: Filename, UpdateLinks, ReadOnly, ...)
$wb = $Excel.Workbooks.Open("C:\path\to\file.xlsx", 0, $true)

# New empty workbook
$wb = $Excel.Workbooks.Add()
```

## Enumerate Sheets

```powershell
foreach ($sheet in $wb.Worksheets) {
  "{0}: '{1}' usedRange={2}" -f $sheet.Index, $sheet.Name, $sheet.UsedRange.Address
}

# Get by name or index
$sheet = $wb.Worksheets.Item("Data")
$sheet = $wb.Worksheets.Item(1)
```

## Read a Range

`Value2` is the fastest way to read a range. It returns a 2-D `object[,]`
that Excel hands back **1-based** (lower bound = 1, upper bound = row/col
count). Always iterate with `GetLowerBound(0)`..`GetUpperBound(0)` — using
`GetLength(0)` as the upper bound is off-by-one and silently skips the last row.
A single cell returns a scalar; a range returns the 2-D array.

```powershell
$sheet = $wb.Worksheets.Item(1)
$data  = $sheet.Range("A1:B10").Value2

# Iterate using GetLowerBound/GetUpperBound — works regardless of base.
for ($r = $data.GetLowerBound(0); $r -le $data.GetUpperBound(0); $r++) {
  for ($c = $data.GetLowerBound(1); $c -le $data.GetUpperBound(1); $c++) {
    "[$r,$c] = $($data[$r,$c])"
  }
}
```

### Worked example: read range into `pscustomobject[]`

```powershell
function Read-ExcelRange {
  param([Parameter(Mandatory)] $Sheet, [Parameter(Mandatory)][string] $Range)
  $v = $Sheet.Range($Range).Value2
  if ($null -eq $v) { return @() }
  if ($v -isnot [array]) { return @([pscustomobject]@{ Col1 = $v }) }
  $rLo = $v.GetLowerBound(0); $rHi = $v.GetUpperBound(0)
  $cLo = $v.GetLowerBound(1); $cHi = $v.GetUpperBound(1)
  # Treat first row as headers
  $headers = $cLo..$cHi | ForEach-Object { [string]$v[$rLo, $_] }
  for ($r = $rLo + 1; $r -le $rHi; $r++) {
    $obj = [ordered]@{}
    for ($c = $cLo; $c -le $cHi; $c++) { $obj[$headers[$c - $cLo]] = $v[$r, $c] }
    [pscustomobject]$obj
  }
}

$rows = Read-ExcelRange -Sheet $sheet -Range "A1:C100"
$rows | Format-Table -AutoSize
```

## Write Cells

Setting `.Value2` on a range with a same-shape 2-D array is the **fast path**
(one COM hop). Looping cell-by-cell is 10–100× slower.

```powershell
# Single cell
$sheet.Cells.Item(1,1).Value2 = "Name"
$sheet.Range("A2").Value2     = "Alice"

# Bulk write — single COM call. Excel accepts a 0-based object[,] when assigning
# to .Value2 (it copies cell-by-cell), so the PowerShell-allocated buffer below
# works even though *read* arrays come back 1-based. The Range must match the
# array shape exactly.
$rows = 100; $cols = 3
$buf = New-Object 'object[,]' $rows, $cols
for ($r = 0; $r -lt $rows; $r++) {
  $buf[$r,0] = "row$r"; $buf[$r,1] = $r; $buf[$r,2] = $r * 2
}
$sheet.Range("A1").Resize($rows, $cols).Value2 = $buf

# Formula
$sheet.Range("D1").Formula = "=SUM(B:B)"
```

## Run a Macro

```powershell
# Macro that takes no args
$Excel.Run("MyMacro")

# Macro with args
$result = $Excel.Run("ProcessSheet", "Inputs", 42)
"Macro returned: $result"
```

The workbook containing the macro must be open. If macros are blocked by policy,
the user must allow them in **Trust Center → Macro Settings** for the location.

## Evaluate Formulas

`$Excel.Evaluate(...)` runs a formula in the active workbook's context. Useful
for ad-hoc calculations without writing to a cell.

```powershell
$Excel.Evaluate("=SUM(1,2,3,4,5)")               # 15
$Excel.Evaluate("=VLOOKUP(""Alice"",A:B,2,0)")   # value next to "Alice"
$Excel.Evaluate("=TEXT(NOW(),""yyyy-mm-dd"")")   # today's date as string
```

## Save and Close

```powershell
# Save in place
$wb.Save()

# Save As — second arg is FileFormat enum
$wb.SaveAs("C:\out\report.xlsx", 51)   # 51 = xlOpenXMLWorkbook  (.xlsx)
$wb.SaveAs("C:\out\macro.xlsm",  52)   # 52 = xlOpenXMLWorkbookMacroEnabled
$wb.SaveAs("C:\out\report.csv",   6)   #  6 = xlCSV
$wb.SaveAs("C:\out\report.txt",  42)   # 42 = xlUnicodeText (tab-delimited UTF-16)

# Close without saving
$wb.Close($false)

# Close with save
$wb.Close($true)
```

### Export to PDF

```powershell
# 0 = xlTypePDF, 1 = xlTypeXPS
$wb.ExportAsFixedFormat(0, "C:\out\report.pdf")
```

## Format Cells

```powershell
$r = $sheet.Range("A1:D1")
$r.Font.Bold = $true
$r.Interior.Color = 0x00B050        # green (BGR)
$r.HorizontalAlignment = -4108      # xlCenter
$r.NumberFormat = "0.00%"
$r.Borders.LineStyle = 1             # xlContinuous
$r.ColumnWidth = 18

# Autofit
$sheet.UsedRange.EntireColumn.AutoFit() | Out-Null
```

## Complete Worked Example

Create a workbook, write data, save, reopen read-only, read back:

```powershell
pwsh -STA -Command @'
$Excel = $null; $wb = $null; $wb2 = $null
try {
  $Excel = New-Object -ComObject Excel.Application
  $Excel.Visible       = $false
  $Excel.DisplayAlerts = $false

  $wb = $Excel.Workbooks.Add()
  $sh = $wb.Worksheets.Item(1)
  $sh.Name = "Test"
  $sh.Cells.Item(1,1).Value2 = "Name"
  $sh.Cells.Item(1,2).Value2 = "Score"
  $sh.Cells.Item(2,1).Value2 = "Alice"; $sh.Cells.Item(2,2).Value2 = 90
  $sh.Cells.Item(3,1).Value2 = "Bob";   $sh.Cells.Item(3,2).Value2 = 85
  $sh.Cells.Item(4,1).Value2 = "Total"
  $sh.Cells.Item(4,2).Formula = "=SUM(B2:B3)"

  $path = Join-Path $env:TEMP "demo.xlsx"
  if (Test-Path $path) {
    # Don't silently overwrite a previous run's file.
    Remove-Item $path -Force
  }
  $wb.SaveAs($path, 51)
  $wb.Close($false)

  $wb2 = $Excel.Workbooks.Open($path, 0, $true)
  $sh2 = $wb2.Worksheets.Item(1)
  $v   = $sh2.Range("A1:B4").Value2
  for ($r = $v.GetLowerBound(0); $r -le $v.GetUpperBound(0); $r++) {
    "Row $r : [$($v[$r, $v.GetLowerBound(1)]), $($v[$r, $v.GetLowerBound(1)+1])]"
  }
  $wb2.Close($false)
} finally {
  foreach ($name in 'sh','sh2','wb','wb2') {
    $val = Get-Variable -Name $name -ValueOnly -ErrorAction SilentlyContinue
    if ($val) { [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($val); Set-Variable $name $null }
  }
  if ($Excel) {
    $Excel.Quit()
    [void][System.Runtime.InteropServices.Marshal]::ReleaseComObject($Excel)
  }
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
  [GC]::Collect(); [GC]::WaitForPendingFinalizers()
}
'@
```

## Safety Rules

Excel writes are **destructive** and not transactional. Apply these rules to any
operation that touches an existing user workbook:

1. **Confirm the target path** with the user before opening read-write.
2. **Default to read-only** (`ReadOnly = $true`) when you only need to read.
3. **Copy before mutating** a user-authored workbook. Save the modified copy
   under a new name and let the user diff.
4. **Never call `.SaveAs(...)` over an existing file** without an explicit
   `Test-Path $path` check + user confirmation — `DisplayAlerts = $false` makes
   the overwrite silent. Either delete the prior file deliberately or pick a
   new path.
5. **Macro-enabled files (`.xlsm`/`.xlsb`)** can run code on open. Treat unknown
   workbooks as untrusted; open with macros disabled or in a sandbox profile.
6. **Cell formulas can reference external workbooks/links.** Set
   `UpdateLinks = 0` on Open if you don't want side effects.
7. **Bulk writes** should use the array-set pattern (`Range.Resize(...).Value2 =
   $buf`) inside `try/finally`; if the script aborts mid-write, the workbook is
   left in a partial state — don't leave it Saved.

## Notes

- **Headless** = `Visible = $false` + `DisplayAlerts = $false` +
  `ScreenUpdating = $false`. The `ScreenUpdating` flag is the biggest perf win
  for bulk writes.
- **STA** is required for stable COM interaction: `pwsh -STA -Command ...`.
- **`Value2` vs `Value`:** `Value2` skips currency/date conversion and is faster.
  Dates come back as OLE Automation doubles — convert with
  `[DateTime]::FromOADate($d)`.
- **Indexing:** sheet indexes, row numbers, and column numbers on `Cells.Item(r,c)`
  are 1-based. The `object[,]` array returned by `Range.Value2` is also 1-based
  when Excel hands it back — always iterate with `GetLowerBound(0)` /
  `GetUpperBound(0)` so the same code works for both reads and any
  PowerShell-allocated 0-based buffers you assign to `.Value2`.
- **Zombie processes:** if `EXCEL.EXE` lingers, you missed a `ReleaseComObject`
  on a Range/Sheet/Workbook. Always release intermediate refs in long sessions.
- **Companion skill:** the `xlsx` skill manipulates `.xlsx` files directly
  without launching Excel — prefer it for pure file IO; prefer this skill for
  anything that needs the Excel engine.

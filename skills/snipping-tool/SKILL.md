---
name: snipping-tool
description: Capture Windows screenshots from PowerShell — full screen, a specific monitor, or a rectangle — and save to PNG or copy to clipboard. Use when the user asks to "take a screenshot", "screenshot the right monitor", "grab a screen capture and save it to disk", "snip a region", "open the Snipping Tool", or "screenshot to clipboard". Covers SnippingTool.exe /clip for the interactive flow, programmatic capture via System.Drawing + System.Windows.Forms.Screen, multi-monitor enumeration, and routing to either a PNG file or the clipboard.
metadata:
  openclaw:
    emoji: "✂️"
    os: ["win32"]
    requires:
      bins: ["pwsh"]
---

# Snipping Tool (Windows screen capture)

Capture Windows screenshots from PowerShell, programmatically or by launching the built-in Snipping Tool for an interactive selection. Routes to a PNG file on disk or to the clipboard, supports multi-monitor setups, and avoids needing any third-party install.

## When to Use

✅ **USE this skill when:**

- User asks for a screenshot of the whole screen, a specific monitor, or a coordinate region
- User wants to launch the interactive Snipping Tool overlay
- User wants the screenshot copied straight to the clipboard so they can paste it into a chat / doc
- User wants to capture on a timer or as part of a script

## When NOT to Use

❌ **DON'T use this skill when:**

- User wants to **save** an image **already** on the clipboard → use the `windows-clipboard` skill
- User wants to capture only a single window with title-bar awareness, or wants UI-element awareness → use a dedicated UI-automation tool (this skill is a focused capture surface, not full UI automation)
- User is on macOS → use `screencapture` or the `peekaboo image` subcommand

## Setup

No install required. PowerShell 7 (`pwsh`) and the `System.Drawing` / `System.Windows.Forms` assemblies ship on Windows. `SnippingTool.exe` is preinstalled on Windows 10/11 at `C:\Windows\System32\SnippingTool.exe` (Win 10) or shipped as the "Snipping Tool" Store app on Windows 11 (`C:\Users\<you>\AppData\Local\Microsoft\WindowsApps\SnippingTool.exe`).

## Interactive Capture (user picks the region)

Launch the Snipping Tool with its clip mode — it dims the screen and lets the user drag a rectangle. The captured region lands on the clipboard.

```powershell
# Preferred on Windows 11: opens the modern snip overlay.
Start-Process 'ms-screenclip:'
```

The `ms-screenclip:` URI is the path the system Print-Screen hotkey uses and works on all current Windows 11 builds. `SnippingTool.exe /clip` is the older Win 10 / pre-22H2 fallback; the legacy executable was removed from some 22H2+ builds and may launch the main window instead of clip mode.

After the user finishes the snip, the image is on the clipboard. **Don't trust `WaitForExit` on the launcher process** — `ms-screenclip:` and `SnippingTool.exe` both fork to a host process and the launched PID exits immediately, so `WaitForExit` returns before the user has drawn. Poll the clipboard for an image instead:

```powershell
Add-Type -AssemblyName System.Windows.Forms

Start-Process 'ms-screenclip:'

# Wait up to 60s for the user to finish the snip.
$deadline = (Get-Date).AddSeconds(60)
while ((Get-Date) -lt $deadline -and -not [System.Windows.Forms.Clipboard]::ContainsImage()) {
  Start-Sleep -Milliseconds 250
}
if (-not [System.Windows.Forms.Clipboard]::ContainsImage()) {
  Write-Warning "User did not complete a snip within 60s; nothing on the clipboard."
}
```

Then read the clipboard image (see `windows-clipboard`) to save it to disk.

## Programmatic Full-Screen Capture

Captures everything across all monitors as one large bitmap.

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

$virtual = [System.Windows.Forms.SystemInformation]::VirtualScreen
$bmp = New-Object System.Drawing.Bitmap $virtual.Width, $virtual.Height
$g   = $null
try {
  $g = [System.Drawing.Graphics]::FromImage($bmp)
  $g.CopyFromScreen($virtual.Location, [System.Drawing.Point]::Empty, $virtual.Size)

  $out = Join-Path $env:USERPROFILE "Pictures\screen-$(Get-Date -Format yyyyMMdd-HHmmss).png"
  $bmp.Save($out, [System.Drawing.Imaging.ImageFormat]::Png)
  Write-Host "Saved $out"
} finally {
  if ($g)   { $g.Dispose() }
  if ($bmp) { $bmp.Dispose() }
}
```

Use `VirtualScreen` for "everything"; use `PrimaryScreen.Bounds` for just monitor 1.

## Capture a Specific Monitor

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# Enumerate monitors
[System.Windows.Forms.Screen]::AllScreens | ForEach-Object {
    "{0,-20} {1,-25} Primary={2}" -f $_.DeviceName, $_.Bounds, $_.Primary
}

# Capture monitor #1 (zero-indexed)
$screen = [System.Windows.Forms.Screen]::AllScreens[1]
$b      = $screen.Bounds
$bmp    = New-Object System.Drawing.Bitmap $b.Width, $b.Height
$g      = $null
try {
  $g = [System.Drawing.Graphics]::FromImage($bmp)
  $g.CopyFromScreen($b.Location, [System.Drawing.Point]::Empty, $b.Size)
  $bmp.Save("$env:USERPROFILE\Pictures\monitor1.png", [System.Drawing.Imaging.ImageFormat]::Png)
} finally {
  if ($g)   { $g.Dispose() }
  if ($bmp) { $bmp.Dispose() }
}
```

Single-monitor setups return one entry; treat `AllScreens[0]` as the primary.

## Capture a Coordinate Rectangle

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# x, y, width, height in virtual-screen coordinates
$rect = New-Object System.Drawing.Rectangle 100, 100, 800, 600

$bmp = New-Object System.Drawing.Bitmap $rect.Width, $rect.Height
$g = [System.Drawing.Graphics]::FromImage($bmp)
$g.CopyFromScreen($rect.Location, [System.Drawing.Point]::Empty, $rect.Size)
$bmp.Save("$env:TEMP\region.png", [System.Drawing.Imaging.ImageFormat]::Png)
$g.Dispose(); $bmp.Dispose()
```

Negative `x` is valid — monitors arranged to the left of the primary live in the negative-X virtual-screen space.

## Capture Straight to Clipboard

For the "paste into chat / paste into a doc" workflow.

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

$b   = [System.Windows.Forms.Screen]::PrimaryScreen.Bounds
$bmp = New-Object System.Drawing.Bitmap $b.Width, $b.Height
$g   = $null
try {
  $g = [System.Drawing.Graphics]::FromImage($bmp)
  $g.CopyFromScreen($b.Location, [System.Drawing.Point]::Empty, $b.Size)
  [System.Windows.Forms.Clipboard]::SetImage($bmp)
  Write-Host "Screenshot is on the clipboard. Paste with Ctrl+V."
} finally {
  if ($g)   { $g.Dispose() }
  if ($bmp) { $bmp.Dispose() }
}
```

`Clipboard::SetImage` is a WinForms API that requires an STA thread. PowerShell 7's default is MTA — if you hit `Current thread must be set to single thread apartment (STA) mode`, relaunch via `pwsh -STA -Command "..."` (see `windows-clipboard`).

## Reusable Function

Drop this into your profile or session to make capture a one-liner.

```powershell
function Save-Screenshot {
    [CmdletBinding()]
    param(
        [string] $Path = (Join-Path $env:USERPROFILE "Pictures\screen-$(Get-Date -Format yyyyMMdd-HHmmss).png"),
        [ValidateSet('Primary','Virtual','Index')] [string] $Mode = 'Primary',
        [int] $Index = 0,
        [switch] $Clipboard
    )
    Add-Type -AssemblyName System.Windows.Forms
    Add-Type -AssemblyName System.Drawing

    $b = switch ($Mode) {
        'Primary' { [System.Windows.Forms.Screen]::PrimaryScreen.Bounds }
        'Virtual' { [System.Windows.Forms.SystemInformation]::VirtualScreen }
        'Index'   { [System.Windows.Forms.Screen]::AllScreens[$Index].Bounds }
    }

    $bmp = New-Object System.Drawing.Bitmap $b.Width, $b.Height
    $g   = $null
    try {
        $g = [System.Drawing.Graphics]::FromImage($bmp)
        $g.CopyFromScreen($b.Location, [System.Drawing.Point]::Empty, $b.Size)

        if ($Clipboard) {
            [System.Windows.Forms.Clipboard]::SetImage($bmp)
            Write-Output 'clipboard'
        } else {
            $bmp.Save($Path, [System.Drawing.Imaging.ImageFormat]::Png)
            Write-Output $Path
        }
    } finally {
        if ($g)   { $g.Dispose() }
        if ($bmp) { $bmp.Dispose() }
    }
}

# Examples
Save-Screenshot                     # primary monitor → PNG in ~/Pictures
Save-Screenshot -Mode Virtual       # everything → PNG
Save-Screenshot -Mode Index -Index 1
Save-Screenshot -Clipboard          # primary → clipboard
```

## Safety Rules

1. **Confirm before writing files.** Screenshots can contain credentials, chat windows, personal info. Default to a path the user named, do not silently dump to a public folder.
2. **Do not send captures off-machine without explicit consent.** Saving locally is one thing; uploading is another.
3. **Beware of HDR / DRM-protected content.** `CopyFromScreen` returns black for DRM-protected video (Netflix, Disney+ in some browsers) — surface this to the user instead of pretending the capture succeeded.
4. **Multi-user / RDP sessions:** `CopyFromScreen` captures the calling user's session. From `SYSTEM`-context scheduled tasks the screen is the winlogon desktop, not the user's desktop — run capture jobs as the interactive user.

## Notes

- DPI: `CopyFromScreen` uses physical pixels under per-monitor-DPI awareness. If your screenshots come out smaller than expected, mark the host process DPI-aware or use `pwsh.exe`'s default manifest (PowerShell 7 is per-monitor-aware).
- The Windows 11 Store app for Snipping Tool replaces the legacy Snip & Sketch. Both expose the `/clip` argument for direct-to-clipboard capture.
- For animations / videos use the Xbox Game Bar (`Win+G`) or the new Snipping Tool record mode — neither has a stable CLI. This skill stays focused on still captures.
- Bitmap objects are unmanaged resources. Always `Dispose()` (or wrap in `try/finally`) to avoid GDI handle leaks in long-lived shells.

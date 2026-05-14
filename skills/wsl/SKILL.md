---
name: wsl
description: Drive the Windows Subsystem for Linux (WSL) from Windows. Use when the user asks to "run something in WSL", "open my Ubuntu", "use Linux from Windows", "exec a bash command", "share files between Windows and WSL", "list / start / shut down distros", "export / import a distro", or "configure the WSL default user". Covers wsl --list, wsl -d for a specific distro, one-shot commands via wsl -- and wsl --exec, the \\wsl.localhost UNC path, wsl --shutdown, wsl --export / --import, and /etc/wsl.conf.
metadata:
  openclaw:
    emoji: "🐧"
    os: ["win32"]
    requires:
      bins: ["pwsh", "wsl"]
---

# Windows Subsystem for Linux (WSL)

WSL runs real Linux distributions on a Windows host with no full VM overhead. From Windows you drive it through one binary: `wsl.exe`.

## When to Use

- ✅ "Run this Linux command from Windows": `wsl -- <cmd>` or `wsl --exec <cmd>`
- ✅ Open an interactive Linux shell: `wsl -d <distro>`
- ✅ Read/write Linux files from Windows tooling via `\\wsl.localhost\<distro>\...`
- ✅ Read/write Windows files from inside WSL via `/mnt/c/...`
- ✅ Back up a distro (`wsl --export`) or clone it (`wsl --import`)
- ✅ Restart the WSL VM after networking/CPU/RAM weirdness (`wsl --shutdown`)

## When NOT to Use

- ❌ "Install Docker on Windows" — Docker Desktop handles its own WSL backend; don't touch its distros (`docker-desktop`, `docker-desktop-data`).
- ❌ Cross-platform desktop GUI automation — use `windows-ui` / Peekaboo / etc., not WSL.
- ❌ Running Windows-only `.exe` from inside WSL is fine but odd; prefer doing it from Windows.
- ❌ Hardware/USB device access — limited; needs `usbipd-win` setup.

## Setup

WSL 2 ships with Windows 10/11. Bootstrap from an elevated PowerShell:

```powershell
wsl --install                          # installs WSL + default Ubuntu
wsl --install -d <DistroName>          # specific distro; see `wsl --list --online`
wsl --list --online                    # list distros available from MS Store
wsl --update                           # update the WSL kernel
wsl --version                          # WSL + kernel + Windows build
wsl --status                           # default distro, default version, kernel
```

Verify and discover what's installed:

```powershell
wsl --list --verbose
#   NAME              STATE     VERSION
# * OpenClawGateway   Running   2
```

The asterisk marks the **default distro** used when you don't pass `-d`.

## Running Commands

### Interactive shell

```powershell
wsl                          # open default distro at $HOME (Linux user)
wsl -d Ubuntu                # open named distro
wsl --cd ~                   # start in Linux home
wsl --cd /mnt/c/Projects     # start in a Linux path that maps to C:\Projects
wsl -u root                  # log in as a different Linux user
```

> `--cd` expects a **Linux** path. Passing a Windows path like `C:\Projects` is silently ignored on current builds. Convert manually — drive letters mount under `/mnt/<letter>/`, so `C:\Projects` becomes `/mnt/c/Projects`.

### One-shot command

Two forms — they behave differently:

```powershell
# `wsl -- <cmd>`: runs <cmd> via the user's login shell (so aliases, .bashrc, PATH apply)
wsl -- uname -a
wsl -- bash -lc 'echo $PATH; which python'

# `wsl --exec <cmd>`: bypasses the shell (faster, no .bashrc, no shell expansion in WSL)
wsl --exec /usr/bin/uname -a
wsl --exec echo hello
```

Rule of thumb: use `wsl --exec` for deterministic scripted calls; use `wsl -- bash -lc '...'` when you need shell features (pipes, globbing, env from `.bashrc`).

> [!warning] `wsl -- <cmd>` parses the args on the Windows side. `wsl -- ls -la /tmp` works, but `wsl -- echo $HOME` evaluates `$HOME` in **PowerShell**, not bash. Wrap it: `wsl -- bash -lc 'echo $HOME'`.

### Target a specific distro

```powershell
wsl -d Ubuntu -- whoami
wsl -d Ubuntu -u root -- apt update
```

## File Sharing

### Windows files from Linux

Mounted automatically:

```bash
ls /mnt/c/Users/<you>/Documents
```

Drive root letter is lowercased: `/mnt/c`, `/mnt/d`. `/etc/wsl.conf` can change the mount root.

### Linux files from Windows

Available over UNC paths (no `net use` needed; just open the path):

```powershell
# Modern path (recommended; works as a real DNS-style host)
explorer.exe \\wsl.localhost\OpenClawGateway\home

# Legacy alias (still works; identical contents)
explorer.exe \\wsl$\OpenClawGateway\home

# From PowerShell
Get-ChildItem \\wsl.localhost\OpenClawGateway\etc | Select-Object -First 5
Get-Content   \\wsl.localhost\OpenClawGateway\etc\os-release
```

Both `\\wsl.localhost\` and `\\wsl$\` work. Prefer `\\wsl.localhost\` — it interacts more cleanly with some Windows APIs.

> [!info] Performance: cross-OS file access is slower than native. Keep project source on the OS where the tooling runs — Node/Python projects you'll build with Linux tools live under `~/` (`\\wsl.localhost\<distro>\home\...`), not `/mnt/c/...`.

## Lifecycle

```powershell
wsl --terminate <distro>     # stop one distro (utility VM keeps running for others)
wsl --shutdown               # stop ALL distros + the WSL2 utility VM (firmest reset)
wsl --list --running         # which distros are currently up
```

> [!warning] **`wsl --shutdown` is a hard kill of every running Linux process in every distro.** Unsaved editor buffers, in-flight builds, background services, detached `tmux`/`screen` sessions, and database servers are all terminated without notice. Confirm with the user, ask them to save work and stop services first, and prefer `wsl --terminate <distro>` to reset a single distro instead of nuking everything.

`wsl --shutdown` is the cure for almost all "WSL is acting weird" issues: stale DNS, memory hog, network adapter glitches. The next `wsl` call cold-boots a fresh VM.

## Backup / Clone / Move (`--export` / `--import`)

Export the entire distro to a tarball:

```powershell
wsl --export Ubuntu D:\backups\ubuntu.tar
```

Import it back (same machine, different name; or a new machine):

```powershell
wsl --import UbuntuCopy D:\WSL\UbuntuCopy D:\backups\ubuntu.tar --version 2
```

After import the new distro logs in as `root` — to set a default user, edit `/etc/wsl.conf` inside the distro (see below) and then `wsl --terminate <name>`.

Remove a distro entirely (destructive — confirm with user):

```powershell
wsl --unregister UbuntuCopy
```

`--unregister` deletes the VHD. There's no recycle bin. Always `--export` first if the distro has any state worth keeping.

## `/etc/wsl.conf` (per-distro)

Lives inside the distro at `/etc/wsl.conf`. Touch it from inside WSL:

```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[user]
default=openclaw

[boot]
systemd=true

[interop]
appendWindowsPath=true

[automount]
enabled=true
options="metadata,umask=22,fmask=11"
root=/mnt/
EOF
```

Then from Windows:

```powershell
wsl --terminate <distro>     # next start picks up the new config
```

Common keys:

- `[user] default=<linux-username>` — non-root default for interactive sessions
- `[boot] systemd=true` — enable systemd (WSL 0.67.6+)
- `[interop] appendWindowsPath=false` — drop Windows PATH from Linux $PATH (cleaner builds)
- `[automount] options="metadata"` — preserve Linux permissions on `/mnt/c`

## `.wslconfig` (global, in user profile)

Affects all distros. Lives at `%USERPROFILE%\.wslconfig` on the Windows side:

```ini
[wsl2]
memory=8GB
processors=4
swap=4GB
localhostForwarding=true
```

Apply with `wsl --shutdown` (next start re-reads it).

## Notes

- `wsl --list` produces UTF-16 output by default. If you pipe it to something that mangles it, force UTF-8: `$env:WSL_UTF8=1; wsl --list --verbose`.
- The default WSL hostname inside Linux is the Windows machine name. Override with `[network] hostname=...` in `/etc/wsl.conf`.
- DNS inside WSL is auto-generated. If it breaks (often after VPN connect), `wsl --shutdown`. To stop auto-generation: `[network] generateResolvConf=false` in `/etc/wsl.conf` and write your own `/etc/resolv.conf`.
- WSL 1 (`--version 1`) is a translation layer with no real Linux kernel — slower, more compatible with antivirus. Avoid unless explicitly requested.
- GPU access: WSL 2 supports CUDA on NVIDIA, DirectX12 generally. No setup needed beyond an up-to-date Windows GPU driver.
- File locking across the OS boundary is incomplete — don't run a Windows editor and a Linux git on the same working tree simultaneously.
- Some hosts ship with a reserved or system-managed distro (e.g. a gateway runtime, a developer-platform VM). Confirm with the user before modifying or unregistering any distro you didn't create.

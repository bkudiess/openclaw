---
name: spotify-player
description: Terminal Spotify playback, search, and device control on macOS, Linux, and Windows via `spogo` (preferred) or `spotify_player` (fallback). Use when the user says "play Spotify", "pause Spotify", "search for a song", "skip to the next track", "switch Spotify to my laptop", or wants to drive Spotify without the desktop app's GUI. Spotify itself at spotify.com.
metadata:
  openclaw:
    emoji: "🎵"
    os: ["darwin", "linux", "win32"]
    requires:
      anyBins: ["spogo", "spotify_player"]
    install:
      - id: "brew-spogo"
        kind: "brew"
        os: ["darwin", "linux"]
        formula: "spogo"
        tap: "steipete/tap"
        bins: ["spogo"]
        label: "Install spogo (brew)"
      - id: "brew-spotify-player"
        kind: "brew"
        os: ["darwin", "linux"]
        formula: "spotify_player"
        bins: ["spotify_player"]
        label: "Install spotify_player (brew)"
      - id: "winget-spotify-player"
        kind: "winget"
        os: ["win32"]
        packageId: "aome510.spotify-player"
        bins: ["spotify_player"]
        label: "Install spotify_player (winget)"
      - id: "cargo-spotify-player"
        kind: "cargo"
        os: ["darwin", "linux", "win32"]
        crate: "spotify_player"
        bins: ["spotify_player"]
        label: "Install spotify_player (cargo)"
---

# spogo / spotify_player

Use `spogo` **(preferred)** for Spotify playback/search where available. Fall back to `spotify_player` if `spogo` isn't installed.

## Requirements

- Spotify Premium account (free accounts cannot drive playback over the Connect API).
- Either `spogo` or `spotify_player` installed.

## OS notes

| OS | Recommended install | Config dir |
|---|---|---|
| macOS | `brew install steipete/tap/spogo` (or fall back to `brew install spotify_player`) | `~/.config/spotify-player` |
| Linux | `brew install steipete/tap/spogo` (or distro package on Arch via AUR / `cargo install spotify_player` elsewhere) | `~/.config/spotify-player` (or `$XDG_CONFIG_HOME/spotify-player`) |
| Windows | `winget install aome510.spotify-player` (no `spogo` winget package yet — use `spotify_player`) | `%USERPROFILE%\.config\spotify-player\` (the crate uses `$HOME\.config\`, not `%APPDATA%\`) |

The `spogo` upstream currently ships only on the Homebrew tap, so Windows users default to `spotify_player`. The CLI surface below works on both; differences are called out per-section.

## spogo setup (macOS / Linux only)

```sh
spogo auth import --browser chrome
```

Imports Spotify cookies from your local browser profile. Re-run if playback returns "401 Unauthorized" (cookie rotation).

## Common CLI commands

### Search

```sh
spogo search track "query"                  # spogo
spotify_player search "query"               # spotify_player
```

### Playback

```sh
spogo play              # spogo
spogo pause
spogo next
spogo prev

spotify_player playback play       # spotify_player
spotify_player playback pause
spotify_player playback next
spotify_player playback previous
```

### Devices

```sh
spogo device list
spogo device set "<name|id>"

spotify_player connect             # spotify_player: interactive device picker
```

### Status

```sh
spogo status
spotify_player playback status
```

### Like / save the current track

```sh
spotify_player like                # spotify_player
# spogo: no built-in "like" command; use Spotify UI or web API
```

## Notes

- Config folder: `~/.config/spotify-player` (macOS / Linux) / `%USERPROFILE%\.config\spotify-player\` on Windows (the `spotify_player` crate uses `$HOME\.config\`, not the platform-standard `%APPDATA%\`). Edit `app.toml` there to set theme, default device, etc.
- For Spotify Connect integration set a user `client_id` in `app.toml`. The default client id is shared and rate-limited.
- `spotify_player` is also a full TUI — launch it with no args and press `?` for keyboard shortcuts.
- Cookie-based auth (the `spogo auth import` path) breaks any time Spotify rotates the cookie format; `spotify_player`'s OAuth flow is more durable but requires a browser round-trip on first run.

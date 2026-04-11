# Discord Quest Automation

![Build Artifacts](https://img.shields.io/github/actions/workflow/status/markterence/discord-quest-completer/rust-check.yml?branch=main&style=flat&label=build%20artifacts)
![Release Build](https://img.shields.io/github/actions/workflow/status/markterence/discord-quest-completer/build-release.yaml?branch=main&style=flat&label=build%20release)

A Tauri desktop app that simulates Discord-detectable game activity using lightweight runner executables. This repository includes the refreshed desktop interface, updated packaging flow, and a guided usage tutorial for end users.

## Overview
Discord Quest Automation helps you test or simulate game activity without downloading full game files. The app is centered around verified Discord-detectable game metadata and provides a focused dashboard for searching games, launching lightweight runner processes, stopping them cleanly, and checking status from one place.

## Full Video Tutorial



<video src="https://github.com/user-attachments/assets/88c96ac4-6f34-42dd-b4a6-8739fae50eab" controls width="700"></video>



## Releases
To download the packaged app or source code, visit the [Releases](https://github.com/msxfury/Discord-Quest-Completer-v1.0/releases/tag/2026-v.1.1.0) page.

## Video Tutorial
A full usage walkthrough is included in this repository for users who want to watch the process before running the app.

- Watch the tutorial video here: [Discord-Quest-Automation.mp4](assets/tutorial/Discord-Quest-Automation.mp4)
- Video path inside the repo: `assets/tutorial/Discord-Quest-Automation.mp4`

The tutorial is useful if you want to see:
- how to launch the app
- how to search for a Discord-detectable game
- how to add a game to the dashboard
- how to use the available actions
- how to verify status behavior during use

## Important Notes
- The current production target is Windows.
- Frontend development can run on Windows, macOS, and Linux.
- The runner workflow is currently Windows-focused.
- Use RPC-related features carefully. Depending on use, they may violate Discord terms or platform rules.
- Always extract packaged ZIP releases fully before running the desktop executable.

## Features
- Search and add detectable Discord games
- Simulate game processes with lightweight runner binaries
- Manage start and stop actions from a dashboard UI
- Use a modern dashboard with clearer visual hierarchy and improved status visibility
- Run an optional RPC test flow for activity simulation
- Use packaged Windows releases without needing to open the source in an editor

## Updated Design and Backend Integration
This project includes a major UI/UX refresh and backend/integration updates completed by [msxfury](https://github.com/msxfury).

### Design Updates
- Reworked the app into a modern dark SaaS-style dashboard with stronger visual hierarchy.
- Redesigned the header, search area, game list, game actions panel, status surfaces, and empty states.
- Improved readability using better spacing, typography scale, contrast, alignment, and panel separation.
- Added smoother interaction feedback such as hover states, focus states, transitions, and stronger action emphasis.
- Standardized visual tokens including colors, glow language, border radii, and panel treatment for a more consistent interface.
- Updated branding assets and icon usage across the desktop experience.

### Backend and Integration Updates
- Applied integration-level updates to keep the Tauri + frontend workflow stable.
- Updated runner/resource wiring and icon synchronization for desktop runtime consistency.
- Resolved startup/build path issues that affected local execution and packaged output behavior.
- Improved packaging structure for release ZIPs so users receive only the files needed to run the app.
- Added supporting reliability fixes while preserving existing core functionality and command behavior.

## Contributor
- [msxfury](https://github.com/msxfury) - Led design, integration updates, release polish, and packaging improvements for this version.

## Socials
- Follow [msxfury](https://github.com/msxfury) on Instagram: [msxfury](https://www.instagram.com/__msxfuryyy___)
- Subscribe to [msxfury](https://github.com/msxfury) on YouTube: [msxfury](https://www.youtube.com/@msxfury)

## Credits
- Original project by [markterence](https://github.com/markterence)
- UI/UX redesign and integration updates by [msxfury](https://github.com/msxfury)

## Tech Stack
- Rust
- Vue 3
- Vite
- TypeScript
- Tauri 2

## What This App Does
Discord Quest Automation works by pairing a desktop UI with lightweight runner executables. Instead of requiring full game installations, it allows you to work with Discord-detectable application metadata and launch lightweight process simulations through the packaged Windows runner.

This project is mainly useful for:
- testing the desktop flow
- checking Discord-detectable game metadata behavior
- validating runner launch and stop actions
- exploring the interface and status handling
- educational or controlled testing scenarios

## Prerequisites
Install these first on any OS:
- Node.js 20+
- Rust stable toolchain
- Tauri prerequisites for your OS: https://tauri.app/start/prerequisites/

Then enable pnpm via Corepack:

```bash
corepack enable
corepack prepare pnpm@8.15.3 --activate
```

## Quick Start

### 1. Install dependencies

```bash
pnpm install
```

### 2. Run frontend only

```bash
pnpm dev
```

### 3. Build frontend

```bash
pnpm build
```

## Desktop App (Tauri)

### Windows (supported)
Build the Windows runner and copy it into Tauri resources before running desktop mode.

```powershell
cargo build --release --manifest-path .\src-win\Cargo.toml
Copy-Item .\src-win\target\release\src-win.exe .\src-tauri\resources\src-win.exe -Force
New-Item -ItemType Directory -Path .\src-tauri\target\release\data -Force | Out-Null
Copy-Item .\src-win\target\release\src-win.exe .\src-tauri\target\release\data\src-win.exe -Force
pnpm tauri dev
```

### macOS and Linux (current status)
- Frontend development works with `pnpm dev` and `pnpm build`.
- Native runner behavior is currently Windows-focused.
- Tauri desktop experimentation is possible, but full runner parity is not yet implemented.

## How To Use The App

### Basic Flow
1. Launch the desktop app.
2. Search for a Discord-detectable game in the search bar.
3. Add the game to your active game list.
4. Select the game in the dashboard.
5. Use the available actions in the right-side panel.
6. Watch the status area to confirm the current state.

### What To Expect In The UI
- `Games` panel: shows added or active game entries.
- `Game Actions` panel: shows actions available for the selected game.
- `Status` card: shows whether the app is idle, connected, or running activity.
- `Test RPC`: allows you to test the RPC-related activity flow.

### Packaged App Usage
If you downloaded a packaged Windows release:
1. Extract the ZIP fully.
2. Open the extracted folder.
3. Keep `discord-quest-completer.exe`, `data`, and `resources` together.
4. Launch the main `.exe` from the extracted folder.
5. Do not run the app directly from inside the ZIP viewer.

## Game List Source
The app expects a bundled fallback file:
- `src/assets/gamelist.json`

You can populate it using Discord detectable applications endpoint data.

## Troubleshooting

### App shows localhost or cannot reach page
This usually means the wrong build output was launched or the app was not packaged from the proper release flow.

Recommended fixes:
- use the packaged release build, not a dev-only executable
- fully extract the ZIP before opening the app
- keep the executable and required folders together

### Port `1420` already in use
Stop the process using that port, then rerun.

PowerShell:

```powershell
$pid = (Get-NetTCPConnection -LocalPort 1420 -State Listen -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty OwningProcess)
if ($pid) { Stop-Process -Id $pid -Force }
```

macOS/Linux:

```bash
lsof -ti :1420 | xargs kill -9
```

### Rust toolchain not configured

```bash
rustup default stable
cargo --version
```

### Missing `resources/src-win.exe`
Re-run the Windows runner build and copy commands from the Windows section above.

## Project Scripts

```bash
pnpm dev              # Vite dev server
pnpm build            # Type-check + production frontend build
pnpm preview          # Preview built frontend
pnpm tauri dev        # Run Tauri dev mode
pnpm build:runner:win # Build runner binary (Windows)
```

## Security and Compliance Disclaimer
This project is for educational and testing use. You are responsible for complying with Discord Terms of Service and any relevant software, game, or platform policies. The maintainers are not responsible for account actions, bans, restrictions, or damages resulting from use of this software.

## License
MIT License

# Discord Quest Completer

![Build Artifacts](https://img.shields.io/github/actions/workflow/status/markterence/discord-quest-completer/rust-check.yml?branch=main&style=flat&label=build%20artifacts)
![Release Build](https://img.shields.io/github/actions/workflow/status/markterence/discord-quest-completer/build-release.yaml?branch=main&style=flat&label=build%20release)

A Tauri desktop app that simulates Discord-detectable game activity using lightweight runner executables.

## Overview
Discord Quest Completer helps you test or simulate game activity without installing full game files. The app focuses on verified Discord game metadata and provides a fast dashboard for selecting, launching, and stopping game runner processes.

## Important Notes
- The **current production target is Windows**.
- You can run the **frontend development workflow** on Windows, macOS, and Linux.
- Use the RPC feature carefully. It may violate Discord terms depending on how it is used.

## Features
- Search and add detectable Discord games
- Simulate game processes with lightweight runner binaries
- Manage start/stop actions from a dashboard UI
- Optional RPC test flow for activity simulation

## Updated Design and Backend Integration
This project includes a major UI/UX refresh and backend/integration updates completed by [msxfury](https://github.com/msxfury).

### Design Updates
- Reworked the app into a modern dark SaaS-style dashboard with stronger visual hierarchy.
- Redesigned core surfaces: header, search, game list, actions panel, status card, and empty states.
- Improved readability with better spacing, typography scale, contrast, and alignment.
- Added smoother interaction feedback (hover/focus states, transitions, action emphasis).
- Standardized design tokens (colors, radii, glow/shadow language) for consistency.

### Backend and Integration Updates
- Applied integration-level updates to keep the Tauri + frontend flow stable.
- Updated runner/resource wiring and icon synchronization for desktop runtime consistency.
- Resolved startup/build path issues affecting local execution.
- Added supporting reliability fixes while preserving existing core behavior.

## Contributor
- [msxfury](https://github.com/msxfury) - Led design and integration updates for this release.

## Credits

- Original project by [markterence](https://github.com/markterence)
- UI/UX redesign and integration updates by [msxfury](https://github.com/msxfury)

## Tech Stack
- Rust
- Vue 3 + Vite + TypeScript
- Tauri 2

## Prerequisites
Install these first on any OS:
- Node.js 20+
- Rust (stable toolchain)
- Tauri prerequisites for your OS: https://tauri.app/start/prerequisites/

Then enable pnpm via Corepack:

```bash
corepack enable
corepack prepare pnpm@8.15.3 --activate
```

## Quick Start

### 1) Install dependencies

```bash
pnpm install
```

### 2) Run frontend only (all OS)

```bash
pnpm dev
```

### 3) Build frontend (all OS)

```bash
pnpm build
```

## Desktop App (Tauri)

### Windows (supported)
Build the Windows runner and copy it into Tauri resources before running desktop mode.

PowerShell:

```powershell
cargo build --release --manifest-path .\src-win\Cargo.toml
Copy-Item .\src-win\target\release\src-win.exe .\src-tauri\resources\src-win.exe -Force
New-Item -ItemType Directory -Path .\src-tauri\target\release\data -Force | Out-Null
Copy-Item .\src-win\target\release\src-win.exe .\src-tauri\target\release\data\src-win.exe -Force
pnpm tauri dev
```

### macOS and Linux (current status)
- Frontend development works (`pnpm dev`, `pnpm build`).
- Native runner behavior is currently Windows-focused.
- Tauri desktop can be explored, but full runner parity is not yet implemented.

## Game List Source
The app expects a bundled fallback file:
- `src/assets/gamelist.json`

You can populate it using Discord detectable applications endpoint data.

## Troubleshooting

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
Re-run the Windows runner build + copy commands from the Windows section.

## Project Scripts

```bash
pnpm dev              # Vite dev server
pnpm build            # Type-check + production frontend build
pnpm preview          # Preview built frontend
pnpm tauri dev        # Run Tauri dev mode
pnpm build:runner:win # Build runner binary (Windows)
```

## Security and Compliance Disclaimer
This project is for educational and testing use. You are responsible for complying with Discord Terms of Service and applicable software/game policies. The maintainers are not responsible for account actions, bans, or damages resulting from usage.

## License
MIT License


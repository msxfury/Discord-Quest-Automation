# Discord Quest Automation

Discord Quest Automation is a Windows-focused Tauri desktop application for selecting Discord-detectable games, creating lightweight runner executables, and managing simulated game activity from a clean dashboard.

Original creator: [msxfury](https://github.com/msxfury)

Join Discord server and support us!

Join Our Discord! [https://discord.gg/BrFNPkk5Kw](https://discord.gg/BrFNPkk5Kw)

## Overview

This app helps users work with Discord-detectable game metadata without installing full game files. You can search the game database, add games to your active list, create lightweight runner files, start or stop runner processes, and monitor the current status from the desktop UI.

The project uses:

- Tauri 2 for the desktop shell
- Vue 3, Vite, and TypeScript for the frontend
- Rust for native desktop commands and Windows runner support
- pnpm for package management

## Important Notes

- The primary supported desktop target is Windows.
- macOS and Linux can run the frontend development workflow, but the native runner behavior is Windows-focused.
- The app uses Vite on port `1420` in development.
- Do not use port `1420` for another app while running this project.
- If port `1420` is already in use, the Tauri dev app may fail to start or may show a blank/stale frontend.
- The RPC-related functionality should be used carefully. It may violate Discord or third-party service terms depending on how you use it.
- This project is provided for educational, testing, and development purposes.

## Features

| Feature | Description |
| --- | --- |
| Game search | Search Discord-detectable games by name, alias, or executable metadata. |
| Active game list | Add selected games to a local dashboard list for quick actions. |
| Runner creation | Create lightweight Windows runner executables for selected game entries. |
| Start and stop controls | Launch and stop runner processes directly from the UI. |
| Status tracking | See selected game, running state, and action readiness in the dashboard. |
| Discord RPC test flow | Optional activity simulation flow for testing behavior. |
| Bundled fallback handling | Uses bundled or remote game list sources when available. |
| Desktop integration | Tauri-powered Windows desktop app with bundled resources and app icons. |

## User Guide

### 1. Start the app

For desktop development mode on Windows, run:

```powershell
pnpm tauri dev
```

The app opens as a desktop window. The Vite frontend also runs at:

```text
http://localhost:1420
```

### 2. Search for a game

Use the search field in the dashboard to type a game name, alias, or executable name. Matching Discord-detectable games appear in the search results.

### 3. Add a game

Click `Add Game` on a search result. The game is added to your active game list.

### 4. Select a game

Click a game in the active game list. The actions panel updates for the selected game.

### 5. Create or run a game

If the runner file does not exist, create/install the runner first. Once available, use the play/start action to launch the lightweight process.

### 6. Stop a running game

Use the stop action from the selected game controls. The app attempts to stop the related runner process.

## Prerequisites

Install these before running the project:

- Node.js 20 or newer
- pnpm 8.15.3
- Rust stable toolchain
- Tauri prerequisites for Windows
- Microsoft Visual Studio Build Tools with C++ workload on Windows
- WebView2 Runtime on Windows

Enable pnpm through Corepack:

```bash
corepack enable
corepack prepare pnpm@8.15.3 --activate
```

Verify Rust:

```bash
rustup default stable
cargo --version
```

## Quick Start

Install dependencies:

```bash
pnpm install
```

Run the frontend only:

```bash
pnpm dev
```

Build the frontend:

```bash
pnpm build
```

Run the Tauri desktop app:

```bash
pnpm tauri dev
```

## Desktop App (Tauri)

### Windows

Windows is the main supported target.

Before running desktop mode, build the Windows runner and copy it into the Tauri resource locations:

```powershell
cargo build --release --manifest-path .\src-win\Cargo.toml
Copy-Item .\src-win\target\release\src-win.exe .\src-tauri\resources\src-win.exe -Force
New-Item -ItemType Directory -Path .\src-tauri\target\release\data -Force | Out-Null
Copy-Item .\src-win\target\release\src-win.exe .\src-tauri\target\release\data\src-win.exe -Force
pnpm tauri dev
```

Build a desktop bundle:

```bash
pnpm tauri build
```

## macOS and Linux

The frontend workflow can be used on macOS and Linux:

```bash
pnpm install
pnpm dev
pnpm build
```

Native runner behavior is currently Windows-focused. Tauri may run for development exploration, but full runner support is not guaranteed on macOS or Linux.

## Port Usage

| Port | Used by | Notes |
| --- | --- | --- |
| `1420` | Vite/Tauri dev frontend | Required for `pnpm tauri dev`. Do not use this port for another app while developing. |

If port `1420` is occupied, stop the process using it before starting the app.

PowerShell:

```powershell
$pid = (Get-NetTCPConnection -LocalPort 1420 -State Listen -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty OwningProcess)
if ($pid) { Stop-Process -Id $pid -Force }
```

macOS/Linux:

```bash
lsof -ti :1420 | xargs kill -9
```

## Troubleshooting

### Search does not show games

Make sure the app can load a valid game list. Restart the Tauri app after code changes:

```bash
pnpm tauri dev
```

### Port 1420 already in use

Stop the process using port `1420`, then run the app again.

### Missing `src-win.exe`

Rebuild and copy the Windows runner:

```powershell
cargo build --release --manifest-path .\src-win\Cargo.toml
Copy-Item .\src-win\target\release\src-win.exe .\src-tauri\resources\src-win.exe -Force
New-Item -ItemType Directory -Path .\src-tauri\target\release\data -Force | Out-Null
Copy-Item .\src-win\target\release\src-win.exe .\src-tauri\target\release\data\src-win.exe -Force
```

### Rust is not configured

Install Rust and set the stable toolchain:

```bash
rustup default stable
cargo --version
```

### Tauri build fails on Windows

Confirm that Visual Studio Build Tools, the C++ workload, Windows SDK, Rust, Node.js, and WebView2 Runtime are installed.

## Project Scripts

| Command | Description |
| --- | --- |
| `pnpm dev` | Start the Vite frontend dev server on port `1420`. |
| `pnpm build` | Type-check and build the frontend. |
| `pnpm preview` | Preview the production frontend build. |
| `pnpm tauri dev` | Start the full Tauri desktop app in development mode. |
| `pnpm tauri build` | Build the Tauri desktop application bundle. |
| `pnpm build:runner:win` | Build the Windows runner executable. |
| `pnpm copy:runner:win` | Copy the runner into Tauri resources. |
| `pnpm copy:resources` | Copy the runner into the release data resource path. |
| `pnpm sync:runner` | Build and copy the Windows runner resources. |

## Credits

- Original creator: [msxfury](https://github.com/msxfury)
- Credits: [msxfury](https://github.com/msxfury)

## Security and Compliance Disclaimer

This project is provided for educational, testing, and development purposes only. You are responsible for how you use it. Using simulated activity, process runners, or RPC-related behavior may violate Discord Terms of Service, game policies, platform rules, or local laws. The project authors and maintainers are not responsible for bans, account action, data loss, policy violations, damages, or misuse.

## All Rights Reserved Disclaimer

All Rights Are Reserved. No permission is granted to copy, redistribute, sublicense, sell, rebrand, or claim ownership of this project or its assets unless explicit written permission is provided by the rights holder. Removing the license file does not grant additional usage rights.

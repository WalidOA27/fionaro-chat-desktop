# Fionaro Chat Desktop & Web

**Fionaro Chat** is a fully self-hosted, rebranded fork of [Element Web](https://github.com/element-hq/element-web) — a Matrix client built with the [Matrix JS SDK](https://github.com/matrix-org/matrix-js-sdk), React, and Electron.

This fork connects exclusively to the Fionaro infrastructure and removes all third-party analytics, telemetry, and external dependencies.

- **Web client:** [https://chat.fionaro.pw](https://chat.fionaro.pw)
- **Desktop packages:** [GitHub Releases](https://github.com/WalidOA27/fionaro-chat-desktop/releases)
- **Android APK:** [fionaro-chat-android](https://github.com/WalidOA27/fionaro-chat-android)

---

## Quick Install

### Debian/Ubuntu (.deb)

```bash
curl -sL https://github.com/WalidOA27/fionaro-chat-desktop/releases/download/v1.0.0/fionaro-chat-desktop_1.0.0_amd64.deb \
  -o /tmp/fionaro-chat-desktop.deb
sudo apt install /tmp/fionaro-chat-desktop.deb
# Launch: fionaro-chat-desktop
```

### Arch Linux (AUR)

No AUR package yet. For now, use the AppImage or .tar.gz:

```bash
# AppImage (portable, no install needed)
curl -sL https://github.com/WalidOA27/fionaro-chat-desktop/releases/download/v1.0.0/Fionaro-Chat-1.0.0.AppImage \
  -o ~/Fionaro-Chat.AppImage
chmod +x ~/Fionaro-Chat.AppImage
~/Fionaro-Chat.AppImage
```

### Any Linux (.tar.gz)

```bash
curl -sL https://github.com/WalidOA27/fionaro-chat-desktop/releases/download/v1.0.0/fionaro-chat-desktop-1.0.0.tar.gz \
  -o /tmp/fionaro-chat-desktop.tar.gz
tar -xzf /tmp/fionaro-chat-desktop.tar.gz -C ~/
~/fionaro-chat-desktop-1.0.0/fionaro-chat-desktop
```

### Web (no install)

Open [https://chat.fionaro.pw](https://chat.fionaro.pw) in any modern browser. Installable as PWA.

---

## What's Changed

### Infrastructure & Endpoints
All Matrix and related endpoints point to the Fionaro infrastructure:

| Service | Endpoint | Backend |
|---------|----------|---------|
| Homeserver | `https://matrix.fionaro.pw` | Synapse |
| Element Call | `https://call.fionaro.pw` | Element Call v0.21.0 |
| Push | `https://push.fionaro.pw` | ntfy (UnifiedPush) |
| Auth | `https://auth.fionaro.pw` | Matrix Auth |
| LiveKit SFU | `https://livekit.fionaro.pw` | LiveKit |
| TURN | `turn.fionaro.pw` | coturn |
| Registration | `https://matrix.fionaro.pw/register` | Synapse |

### Distribution
- **Web:** Served at `chat.fionaro.pw` (Caddy, Let's Encrypt)
- **Desktop:** `.deb`, `.AppImage`, `.tar.gz` via GitHub Releases
- No Google Play, no Microsoft Store, no macOS App Store
- No analytics, telemetry, or crash reporting to external services

### Variant System
The desktop app uses Electron Builder's variant system. To build with Fionaro branding:

```bash
VARIANT_PATH=fionaro/release/build.json npx electron-builder --linux tar.gz deb AppImage
```

Default variant (Element) is untouched — the Fionaro variant lives in `apps/desktop/fionaro/`.

### Group Call Fix (v1.0.0)

**Problem:** In Element Call v0.21.0, group calls create a new Matrix room per call via `fet()` → `createRoom()`. Each participant created their own room, so users never saw each other.

**Root cause (Web/Desktop):** The Element Call widget loads without `skipLobby=true` by default (only sets it when Shift+click). Without `skipLobby`, EC shows a lobby → user clicks "Join" → `fet()` runs → `createRoom()` → each user gets a different room.

**Fix (3 parts):**

| Component | Before (broken) | After (fixed) | Layer |
|-----------|-----------------|---------------|-------|
| EC base URL | `Developer.elementCallUrl` (Labs setting) | `element_call.url` from `SdkConfig` | `Call.ts` |
| Path collision | `/` → EC home page renders | `/room/` → EC call view renders | `Call.ts` |
| `skipLobby` | Only on Shift+click | Always forced for group calls | `Call.ts` |
| Call room | `createRoom` via widget API | Original Matrix room (shared) | `appendRoomParams` |

**`appendRoomParams` forces for all non-DM group calls:**

```typescript
params.set("skipLobby", "true");
params.set("returnToLobby", "false");
```

### Stale Device Cleanup

When a call crashes without a clean hangup, stale `m.call.member` entries remain in room state. These show as "waiting media" and prevent proper ringing notifications.

**Fix in `Call.ts`:** `ElementCall.clean()` — previously a noop — now reads `org.matrix.msc3401.call.member` state events from room state on startup and removes stale entries belonging to the current user but from different device IDs:

```typescript
public async clean(): Promise<void> {
    // Removes stale call.member entries for the current user
    // (previous sessions, crashed clients) by sending empty
    // state events for those device IDs.
}
```

### GPU Compatibility

Electron + RADV (AMD Vulkan) crashes the GPU process on Linux. The fix runs GPU in-process (`--in-process-gpu`) when detected.

```typescript
// Auto-detected: lspci shows AMD/Radeon → in-process-gpu
// Fallback (AppImage sandbox, no lspci) → in-process-gpu
```

Users can override: `--no-in-process-gpu` to restore separate GPU process.

### Push & Notifications
- Web: Web Push / PWA
- Desktop: Notifications via system tray (no external push gateway needed)
- Desktop: Notification sounds replaced (no "Element Default" / "Element Fade")

### Removed / Disabled
- Sentry / PostHog integration (all analytics nulled)
- Rageshake / bug report endpoint (empty URL)
- Element branding strings, logos, and meta tags
- All Element-specific URLs and endpoints
- Mobile redirect (disabled — Fionaro Chat has no native mobile apps)
- `welcome_background_url` removed (custom background via CSS body)
- `mobile_guide_toast` disabled
- `mobile_builds` set to null

---

## Build Instructions

### Prerequisites
- Node.js ≥22
- pnpm (use the version in `package.json` — currently `11.2.2`)
- Rust toolchain (for native modules — optional)

### Build Webapp

```bash
git clone https://github.com/WalidOA27/fionaro-chat-desktop.git
cd fionaro-chat-desktop
pnpm install
rm -rf apps/web/webapp apps/web/config.json
pnpm exec nx build element-web --skip-nx-cache
```

Webapp output: `apps/web/webapp/`

### Build Desktop Packages

```bash
cd apps/desktop
ln -s ../web/webapp ./webapp
npx asar p webapp webapp.asar
VARIANT_PATH=fionaro/release/build.json npx electron-builder --linux tar.gz deb AppImage
```

### Run in Dev Mode

```bash
cd apps/desktop
VARIANT_PATH=fionaro/release npx electron . --no-sandbox
```

### Output

```
apps/desktop/dist/
├── fionaro-chat-desktop_1.0.0_amd64.deb
├── Fionaro-Chat-1.0.0.AppImage
└── fionaro-chat-desktop-1.0.0.tar.gz
```

---

## Architecture

```
fionaro-chat-desktop/
├── apps/
│   ├── web/              # Element Web webapp (React SPA)
│   │   ├── src/          # Source code (incl. Call.ts fixes)
│   │   ├── webapp/       # Built webapp output
│   │   └── res/          # Themes, icons, manifest
│   └── desktop/          # Electron wrapper
│       ├── src/          # Electron main process
│       ├── fionaro/      # Fionaro variant (build.json, config.json)
│       ├── build/        # Desktop icon
│       └── dist/         # Build outputs (.deb, .AppImage, .tar.gz)
├── docs/                 # Element Web documentation
└── fionaro-desktop.patch # Reapplicable cumulative patch (1285 lines)
```

## Patch Files

All Fionaro modifications are available as reapplicable `.patch` files in the repo root:

| Patch | Lines | What it covers |
|-------|-------|----------------|
| `fionaro-desktop.patch` | 1285 | All changes (config, branding, Call.ts, build system, etc.) |
| `fionaro-call-fix.patch` | 50 | Group call fix only (Call.ts + IConfigOptions.ts) |
| `fionaro-web-branding.patch` | 90 | Web branding only (index.html, manifest, config.json) |

To apply to a clean Element Web source:

```bash
git checkout v1.12.23
git apply fionaro-desktop.patch
```

---

## Security Notes

> **This fork is NOT affiliated with Element or the Element Web project.**  
> It is a completely independent, self-hosted deployment of the Element Web codebase.

### What is NOT in this fork
- No upstream Element keypairs, certificates, or credentials
- No connection to Element's infrastructure
- No Element branding or trademarked assets
- No third-party analytics, telemetry, or crash reporting

### Audit Surface
- All Matrix traffic goes through `matrix.fionaro.pw` (Synapse)
- All calls go through `call.fionaro.pw` (Element Call) + `livekit.fionaro.pw` (LiveKit)
- Push notifications via `push.fionaro.pw` (ntfy)
- TURN via `turn.fionaro.pw` (coturn)
- Web client served at `chat.fionaro.pw` (Caddy, Let's Encrypt auto-HTTPS)

---

## License

This fork inherits the dual license of Element Web:
- GNU AGPL v3 (or later) — see [LICENSE](LICENSE)
- GNU GPL v3 (or later)
- Element Commercial License (available from Element)

> **This fork does NOT grant any rights to Element branding, trademarks, or infrastructure.**  
> All Element branding has been removed and replaced with Fionaro branding.

---

*Forked from [element-hq/element-web](https://github.com/element-hq/element-web) at tag `v1.12.23`.*

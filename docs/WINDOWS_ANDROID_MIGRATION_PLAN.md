# Windows 11 Binary + Android App Migration Plan

## 1) Current state analysis

### Build/runtime topology today

- The repository already supports a desktop Electron app (`src/app`) packaged by gulp/electron-packager, including an explicit Windows packaging task (`app:package-win32`).
- The repository also contains a browser/web target (`src/web`) with a Python websocket proxy backend that bridges browser websocket traffic to ManageSieve TCP/TLS.
- Shared protocol/editor/UI code lives mostly in `src/common` and is copied into app/web/webextension builds by gulp tasks.

### Platform coupling found during analysis

1. **Electron main-process coupling**
   - `src/app/sieve.mjs` uses Electron APIs (`BrowserWindow`, `ipcMain`, dialogs, safeStorage, app lifecycle).
   - This is expected for desktop, but is not portable to Android.

2. **Node/Electron renderer coupling**
   - `src/app/app.mjs` and multiple app libs use `require('electron')`, `require('fs')`, `require('net')`, `require('tls')`.
   - `src/app/libs/libManageSieve/SieveClient.mjs` depends on raw Node sockets (`net`, `tls`).

3. **Web target has residual Electron dependency**
   - `src/web/static/app.mjs` still calls `require("electron").clipboard` for copy/paste actions.
   - This must be replaced before Android browser-webview packaging.

4. **Transport split already exists (good for migration)**
   - Desktop app uses a TCP/TLS ManageSieve client implementation (`src/app/libs/libManageSieve/SieveClient.mjs`).
   - Web uses a websocket transport (`src/web/static/libs/libManageSieve/SieveClient.mjs`).
   - This split can become an explicit runtime adapter layer.

5. **Packaging gap for Android**
   - There is no Android project/tooling (no Capacitor/Cordova/TWA/React Native wrapper).
   - Existing web frontend + websocket backend is the best base for Android packaging.

## 2) Migration strategy (recommended)

Use a **dual-runtime architecture**:

- **Windows 11 binary:** continue using Electron (already working path), but modernize and harden runtime boundaries.
- **Android app:** package the web frontend in a WebView shell (recommended: Capacitor) and connect to ManageSieve via websocket backend endpoint.

This minimizes risky rewrites while enabling later feature changes.

## 3) Target architecture

Create a strict `Platform Services` interface consumed by shared UI/business logic:

- ClipboardService (`copy`, `paste`)
- DialogService (`openFile`, `saveFile`)
- StorageService (preferences + encrypted secrets)
- TransportFactory (`tcp/tls` for desktop, `ws/wss` for web/android)
- ExternalLinkService (`openExternal`)
- DiagnosticsService (`openDevTools`, `reload`) [desktop-only optional]

Then provide implementations:

- `platform/electron/*`
- `platform/browser/*` (used by web + Android)

Outcome: code in `src/common` and most UI modules depends on interfaces, not direct `electron`/`require` globals.

## 4) Phased execution plan

### Phase 0 — Baseline and constraints (1 sprint)

1. Define supported matrix:
   - Windows 11 x64 executable (installer/zip)
   - Android 11+ app package (APK/AAB)
2. Freeze behavior with smoke tests around:
   - account CRUD
   - script list/open/save/activate
   - import + settings
3. Add CI jobs for lint + tests on Linux (existing), Windows, and Android web build checks.

## Phase 1 — Decouple platform APIs (2–3 sprints)

1. Introduce `Platform Services` contracts in shared code.
2. Replace direct `require('electron')` calls in frontend flow with injected service usage.
3. Move file-system and dialog actions fully behind Electron IPC.
4. Replace `src/web/static/app.mjs` clipboard dependency with browser clipboard API fallback and permission-aware UX.

**Deliverable:** shared/frontend code runs with no direct Electron imports.

## Phase 2 — Transport unification (1–2 sprints)

1. Define `SieveTransport` abstraction and standard lifecycle/events.
2. Keep current implementations:
   - Node TCP/TLS transport for desktop.
   - WebSocket transport for web/android.
3. Move connection policy (TLS mode, cert/pinning, timeout/reconnect strategy) into config layer.

**Deliverable:** account/session code is transport-agnostic.

## Phase 3 — Windows 11 production hardening (1–2 sprints)

1. Upgrade Electron security posture:
   - disable `nodeIntegration` in renderer where possible,
   - enable `contextIsolation`,
   - expose minimal preload bridge APIs.
2. Keep `safeStorage`-based secret handling for Windows credentials.
3. Produce signed distributables (MSIX or signed EXE/ZIP pipeline).
4. Add Windows-specific smoke/e2e packaging validation.

**Deliverable:** secure Windows-native binary pipeline.

## Phase 4 — Android app packaging (2–3 sprints)

1. Create Android wrapper project (Capacitor recommended).
2. Package `src/web/static` app as web assets.
3. Implement mobile platform service adapter:
   - clipboard via web APIs / Capacitor plugin,
   - file import/export via Android file picker plugin,
   - secure local storage via Capacitor secure storage plugin.
4. Configure network security:
   - enforce HTTPS/WSS,
   - certificate pinning (if app-owned backend),
   - clear error UX for untrusted endpoints.
5. Add touch/mobile UX pass:
   - responsive layouts,
   - virtual keyboard overlap handling,
   - larger hit targets.

**Deliverable:** installable Android app connected to websocket backend.

## Phase 5 — Release engineering + observability (1 sprint)

1. Versioning alignment across Electron + web/android artifacts.
2. Crash/error telemetry hooks (privacy-preserving).
3. Store-ready release checklist (Play Store metadata, permissions, privacy policy).
4. Rollout strategy: internal -> beta -> production.

## 5) Key technical decisions to make early

1. **Android connectivity model**
   - Preferred: remote websocket backend over HTTPS/WSS.
   - Alternative: embedded local proxy is not recommended (complexity/security).

2. **Credential model**
   - Windows: keep `safeStorage`.
   - Android: use OS keystore-backed secure storage plugin.

3. **Certificate handling policy**
   - Keep strict defaults; only allow override with explicit, auditable user action.

4. **Packaging format on Windows**
   - Zip only (quick) vs installer/MSIX (enterprise-friendly updates/signing).

## 6) Risks and mitigations

- **Risk:** hidden Electron assumptions in shared modules.
  - **Mitigation:** static scan for `require('electron')`, `window.require`, Node core module usage outside platform adapters.
- **Risk:** websocket backend operational complexity.
  - **Mitigation:** provide containerized backend deployment templates and health checks.
- **Risk:** mobile UX degradation on complex editor flows.
  - **Mitigation:** prioritize key workflows and mobile-specific UI tuning before feature expansion.
- **Risk:** TLS/cert UX confusion.
  - **Mitigation:** unified trust-state indicators and explicit recovery flows.

## 7) Suggested first implementation tickets

1. Create `Platform Services` interfaces + dependency injection entrypoint.
2. Refactor clipboard/file/dialog actions out of `src/app/app.mjs` and `src/web/static/app.mjs` into adapters.
3. Remove Electron dependency from web build target.
4. Add cross-target smoke tests for account/script lifecycle.
5. Scaffold Capacitor Android project and wire web build output.
6. Add Windows signing + packaging CI draft.

## 8) Definition of done for migration

Migration is complete when:

- Windows 11 artifact is signed, installable, and passes smoke tests.
- Android APK/AAB is installable, connects via WSS backend, and passes core workflow tests.
- Shared business/UI code no longer directly imports Electron/Node APIs.
- Platform-specific implementations are isolated and documented.

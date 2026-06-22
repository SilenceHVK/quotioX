# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Quotio is a native macOS menu bar app (SwiftUI, Swift 6) that wraps the external [CLIProxyAPI](https://github.com/luispater/CLIProxyAPI) binary, providing OAuth/account management, quota tracking, and CLI agent configuration for multiple AI providers.

`AGENTS.md` is the authoritative short-form contributor guide. The notes below add the build/run details and architectural shape that aren't in it.

## Commands

Build (debug, normal validation after code changes):
```bash
xcodebuild -project Quotio.xcodeproj -scheme Quotio -configuration Debug build
```

Build debug and launch the app (kills existing instances first):
```bash
./scripts/debug.sh
```

Release pipeline (only touch when changing packaging/notarization/appcast):
```bash
./scripts/build.sh          # archive, extract, verify bundled proxy, ad-hoc sign
./scripts/release.sh        # full release flow
./scripts/bump-version.sh [major|minor|patch|X.Y.Z]   # bumps MARKETING_VERSION + CURRENT_PROJECT_VERSION in project.pbxproj
```

Screenshot generation (`scripts/capture-screenshots.ts`) uses Bun:
```bash
cd scripts && bun install
```

There is **no automated test suite**. Validate by building and exercising the affected flow (OAuth, proxy lifecycle, menu bar, agent config) by hand. For UI changes, also verify light/dark mode.

## Local dev scheme (optional)

`Config/Local.xcconfig` (gitignored) overrides bundle id, product name, and app icon so a "Quotio Dev" build can coexist with a production install. The file's example sets `PRODUCT_BUNDLE_IDENTIFIER = dev.quotio.desktop.dev` and `ASSETCATALOG_COMPILER_APPICON_NAME = AppIconDev`. Pair it with a duplicated, unshared Xcode scheme — see `Config/Local.xcconfig.example` for the full procedure.

## Architecture

### Headless-capable bootstrap

`Quotio/QuotioApp.swift` is unusual: the app must fully initialise even when launched at login with **no window opened** (when `showInDock=false` puts it in `.accessory` policy). The pattern:

- `AppBootstrap.shared` (`@MainActor` singleton) owns `QuotaViewModel` and `LogsViewModel` and exposes idempotent `initializeIfNeeded()`.
- `AppDelegate.applicationDidFinishLaunching` calls `initializeIfNeeded()` immediately so proxy auto-start and quota fetching happen regardless of window visibility.
- The SwiftUI `.task` on `ContentView` also calls `initializeIfNeeded()` — it's a no-op the second time.
- `AppDelegate` also manages activation-policy promotion (accessory ↔ regular) and a fair amount of window-focus reassertion logic; treat that block as load-bearing.

When adding new app-wide initialisation, hook it into `AppBootstrap.performFullInitialization()`, not into the view's `.task`.

### Three operating modes

`OperatingModeManager.shared` (`Quotio/Models/OperatingMode.swift`) drives sidebar visibility and which services run:

- **Local Proxy Mode** — runs the bundled `CLIProxyAPI` binary locally; full feature set including Agents, API Keys, Logs, Fallback.
- **Remote Proxy Mode** — points at an external CLIProxyAPI instance.
- **Quota-Only / Monitor Mode** — no proxy; only scans local auth files and fetches quotas directly. Most provider, fallback, and agent UI hides.

`ContentView` in `QuotioApp.swift` reads `modeManager.isLocalProxyMode` / `isRemoteProxyMode` / `isMonitorMode` to decide which nav items and status row to render. New screens that only make sense in one mode should gate visibility there.

### Layering

```
Views/Screens/*           SwiftUI screens (one per nav page)
Views/Components/*        Reusable SwiftUI (sheets, cards, rows)
Views/Onboarding/*        First-run flow

ViewModels/               @Observable, @MainActor — QuotaViewModel is the central state container

Services/                 Business logic, networking, process management, persistence
  Proxy/                  CLIProxyManager (binary lifecycle), ProxyBridge, ProxyStorageManager
  QuotaFetchers/          One file per provider — see "Adding a quota fetcher"
  Antigravity/            SQLite + protobuf + process control to switch Antigravity IDE accounts
  Tunnel/                 cloudflared tunnel manager

Models/                   Plain data + small managers (e.g. MenuBarSettingsManager)
```

Concurrency rules (also in AGENTS.md): UI-facing mutable state is `@MainActor`; async services with mutable state are `actor`s; DTOs crossing actor boundaries are `Sendable`. Do not put networking, persistence, process management, or OAuth logic directly in SwiftUI views.

### Quota fetcher pattern

Each provider has its own fetcher in `Quotio/Services/QuotaFetchers/`. They differ in how they discover credentials and read quota:

- Some hit a provider HTTP API using auth files in `~/.config/...` (OpenAI, Copilot, Antigravity).
- Some shell out to the provider's CLI (`claude usage`).
- Some parse the provider CLI's auth file directly (Codex, Gemini).
- Cursor/Trae read from browser session / SQLite.

Results merge into `QuotaViewModel.providerQuotas: [AIProvider: [String: ProviderQuotaData]]`. The menu bar reads from there via `AppBootstrap.quotaItems`. When you add a fetcher, also register it in the refresh path and add the provider to `AIProvider`.

### Menu bar

`StatusBarManager` + `StatusBarMenuBuilder` build a native `NSMenu` (not a SwiftUI popover). It must rebuild on changes to mode, language, quota data, and several menu-bar settings — see the chain of `.onChange(...)` handlers on `ContentView` in `QuotioApp.swift`. Forgetting to wire a new setting into `bootstrap.updateStatusBar()` / `statusBarManager.rebuildMenuInPlace()` will leave the menu stale.

### App lifecycle quirks

- `applicationShouldTerminateAfterLastWindowClosed` returns `false` — closing the window keeps the menu-bar agent alive.
- `applicationWillTerminate` synchronously waits up to 1.5s on a `DispatchSemaphore` for `TunnelManager.stopTunnel()` before falling back to `cleanupOrphans()`. Don't make that path async-only.
- `AtomFeedUpdateService` polls CLIProxyAPI's release feed every 5 minutes with ETag caching — started from `AppDelegate`, stopped in `applicationWillTerminate`.

## Invariants (from AGENTS.md — repeated because they're easy to break)

- `ProxyStorageManager`: never delete the current proxy version.
- `AgentConfigurationService`: never overwrite existing backups.
- `ProxyBridge`: target host must remain localhost.
- `CLIProxyManager`: base URL must point directly to CLIProxyAPI.
- Avoid `Text("localhost:\(port)")` — interpolating an `Int` in a `Text` triggers locale grouping (`localhost:8,317`). Use `Text("localhost:" + String(port))`.
- Never log or commit tokens, OAuth codes, cookies, auth headers, or local config files.

## Git

- Never commit to `master` — work on a feature branch and open a PR.
- Before committing: inspect `git status`, `git diff`, `git diff --cached`. Check for secrets and stray build output (`build/`, `release/`).
- Treat untracked files as user-owned; don't `git clean` or overwrite them.
- Conventional Commits style — recent history uses `fix:`, `chore:`, `docs:` prefixes.

## Further reading

- `AGENTS.md` — concise contributor rules (source of truth, keep in sync if you change ours).
- `docs/codebase-summary.md` — deeper module-by-module breakdown (slightly older but still useful).
- `docs/codebase-structure-architecture-code-standards.md` — coding standards.
- `docs/project-overview-prd.md` — product context.
- `scripts/README.md` and the inline help in each `scripts/*.sh` — release tooling details.

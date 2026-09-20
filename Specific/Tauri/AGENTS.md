# AGENTS.md — Tauri Best Practices

This file tells AI coding agents (and humans) how to work in this Tauri codebase. Follow it unless a maintainer explicitly overrides a rule in the task description. When a rule here conflicts with a quick fix, the rule wins.

> **Treat the frontend as untrusted web content.** Tauri's security model assumes the webview can be compromised (a bad dependency, an XSS bug, injected remote content). The Rust core, the capability files, and the CSP are what limit the damage. Almost every rule below follows from that idea.

---

## 1. Project assumptions

- **Framework version:** Tauri **2.x**. Do not write Tauri 1.x code. Agents trained on older material often produce v1 patterns; see the table below and reject them.
- **Backend:** Rust (stable toolchain) in `src-tauri/`. **Frontend:** any framework that builds to static HTML/JS/CSS, in `src/`. Server-side rendering is not available; use a static/SPA build (for example, Next.js needs static export).
- **Commands:** Do not invent scripts. Read `package.json` and `src-tauri/Cargo.toml`, then use what exists. Typical entry points are `npm run tauri dev`, `npm run tauri build`, `cargo test`, `cargo clippy`, and `cargo fmt`. If a script you need does not exist, say so rather than guessing.
- **Version alignment:** `tauri`, `tauri-build`, the `@tauri-apps/cli`, `@tauri-apps/api`, and every official plugin (Rust crate and npm package) must stay on the same major and minor line. Mismatches cause build errors or runtime failures. Use `tauri info` to inspect versions.
- **Single version source:** Omit `version` from `tauri.conf.json` so it inherits from `src-tauri/Cargo.toml`. Keep `package.json` in sync when you bump.

### Tauri v1 patterns that are wrong here

| Do not use (v1)                                   | Use instead (v2)                                              |
| ------------------------------------------------- | ------------------------------------------------------------- |
| `tauri.allowlist` in `tauri.conf.json`            | Capability files in `src-tauri/capabilities/`                 |
| `@tauri-apps/api/tauri` (`invoke`)                | `@tauri-apps/api/core`                                        |
| `@tauri-apps/api/fs`, `/shell`, `/dialog`, `/http`| Official plugins: `@tauri-apps/plugin-fs`, `-shell`, `-dialog`, `-http` |
| `tauri::Window` for webview-level APIs            | `tauri::WebviewWindow` and `Manager::get_webview_window`      |
| `build.devPath` / `build.distDir`                 | `build.devUrl` / `build.frontendDist`                         |
| `window.emit` without importing a trait           | `use tauri::Emitter;` (and `Listener` for `listen`)           |
| Top-level `tauri` key in config                   | `app` key; `productName` and `identifier` at the top level    |
| `main.rs` containing all app setup                | `lib.rs` with `pub fn run()`; `main.rs` only calls it         |

## 2. Trust boundaries and project layout

| Location                         | Runs in            | Trust     | Notes                                                              |
| -------------------------------- | ------------------ | --------- | ------------------------------------------------------------------ |
| `src/`                           | System webview     | Untrusted | Web APIs plus the narrow surface granted by capabilities           |
| `src-tauri/src/`                 | Native Rust process| Trusted   | Commands, state, plugins, all OS access                            |
| `src-tauri/capabilities/*.json`  | Build-time config  | Policy    | Defines what each window/webview may call                          |
| `src-tauri/tauri.conf.json`      | Build-time config  | Policy    | CSP, windows, bundle settings, updater                             |

Recommended Rust layout (add modules as the app grows):

- `src-tauri/src/main.rs` — entry point, calls `app_lib::run()`.
- `src-tauri/src/lib.rs` — builder, plugins, state, command registration.
- `src-tauri/src/commands.rs` (or `commands/` per domain) — thin `#[tauri::command]` functions.
- `src-tauri/src/state.rs` — types passed to `manage()`.
- `src-tauri/src/error.rs` — one serializable app error type.
- Business logic lives in plain modules that do **not** depend on `tauri`, so they can be unit-tested without a runtime.

Frontend rule: all calls into Rust go through **one** typed module (for example `src/lib/api.ts`). No raw `invoke('...')` string literals scattered across components.

## 3. Security rules (non-negotiable)

Never weaken these to make a feature "just work". If a feature seems to require it, stop and propose a safer design.

### 3.1 Capabilities and permissions (least privilege)

- Every window and webview gets **only** the permissions it needs. Capability files live in `src-tauri/capabilities/` and target windows by `label`.
- Do not grant blanket permissions. Prefer specific ones (`fs:allow-read-text-file`) over broad ones, and always attach **scopes** to filesystem, shell, and HTTP permissions. Scope entries look like `{ "identifier": "fs:allow-read-text-file", "allow": [{ "path": "$APPDATA/**" }] }`.
- Never use a wildcard scope such as `**` at the filesystem root, and use `deny` scopes for sensitive subpaths inside an allowed tree.
- Split capabilities by trust level: a window that shows user-generated or third-party content gets a separate, minimal capability from the main window.
- Use the `platforms` field when a permission should only apply on some operating systems.
- Do **not** grant capabilities to remote URLs (the `remote` option) unless the task explicitly requires it. If it does, restrict to exact HTTPS origins and the smallest possible permission set.
- Review every capability diff in code review as a security change.

### 3.2 Restrict your own commands

By default, commands you define with `#[tauri::command]` are callable from any window. Restrict them:

1. List every app command in `build.rs` via `tauri_build::AppManifest::new().commands(&[...])`.
2. Grant each one explicitly in a capability file using its generated `allow-<command-name>` permission (kebab-case).
3. A command that is registered in `generate_handler!` but not granted in a capability is unreachable. That is the safe default; do not "fix" it by loosening the policy.

The generated permission schemas are refreshed on build, so run a dev build after adding a command to get autocompletion for the new permission identifiers.

### 3.3 Content Security Policy

- Always set `app.security.csp` in `tauri.conf.json`. Start from `default-src 'self'` and add only what you need.
- `connect-src` must include `ipc: http://ipc.localhost` so the IPC bridge works. Add the exact API origins your frontend calls; do not use `*`.
- No `'unsafe-eval'`. Avoid `'unsafe-inline'` for scripts (Tauri injects nonces and hashes for local scripts). `'unsafe-inline'` for styles is a common, lower-risk exception.
- If the dev server's hot-reload websocket is blocked, use the `devCsp` option for development only. Never loosen the production `csp` to fix a dev-time problem.
- Keep `app.security.freezePrototype` set to `true`.
- Do not set any `dangerous*` config keys, and never disable CSP modification.

### 3.4 Global API and isolation

- Keep `app.withGlobalTauri` **false**. Import from `@tauri-apps/api` instead of exposing `window.__TAURI__` to every script on the page.
- If the app loads code you do not fully control (large dependency trees, plugins, user scripts), consider the **isolation pattern** so IPC messages are checked and encrypted in a sandboxed frame before reaching Rust.

### 3.5 Command input validation

- **Validate every argument in Rust** as if an attacker sent it. Check length, range, format, and allowed values. TypeScript types are erased at runtime and prove nothing.
- Never build filesystem paths, shell commands, SQL, or URLs from raw frontend strings. For paths: resolve against a known base (for example `app.path().app_data_dir()`), canonicalize, and verify the result is still inside the base. Reject `..` traversal and absolute paths you did not expect.
- Prefer the official `dialog` plugin for user-chosen files, and the `fs` plugin with scopes, over custom commands that accept arbitrary paths.
- Do not return raw internal errors, stack traces, or absolute paths to the frontend. See §5.

### 3.6 Processes, sidecars, and external links

- Never spawn a shell with a concatenated string. Use argument arrays, and validate each argument.
- Sidecars go in `bundle.externalBin` (named with the target-triple suffix) and are granted through **scoped** shell permissions with `sidecar: true` and a validated argument list. Do not allow arbitrary program execution.
- Open external links through the official opener plugin with a scope limited to `https:` (and `mailto:` if needed). Never pass unvalidated URLs to the OS.
- Restrict in-app navigation: window navigation handlers (`on_navigation`) should reject any URL that is not your app's own origin. Do not load remote pages into a window that has IPC access.

### 3.7 Secrets and data

- Never ship secrets, API keys, or signing keys in the frontend bundle or in Rust source. Anything inside the app is extractable.
- Store user credentials in the OS credential store (Keychain, Credential Manager, Secret Service), for example with a maintained keyring crate accessed from Rust. Do not use `localStorage`, plain JSON files, or the store plugin for tokens.
- Log without secrets: no tokens, passwords, or full user paths at info level.
- Treat deep-link URLs, file associations, CLI arguments, and clipboard content as untrusted input.

### 3.8 Supply chain

- Commit `Cargo.lock` (this is an application) and the JavaScript lockfile. Build CI from a clean checkout with locked dependencies (`cargo build --locked`, `npm ci`).
- Run `cargo audit` or `cargo deny`, and `npm audit`, when dependencies change. Prefer official `tauri-apps` plugins. Vet community plugins for maintenance and permissions before adding.
- Justify any `unsafe` block in a comment. Prefer safe abstractions.

## 4. Configuration rules

- `identifier` is a unique reverse-domain string (for example `com.yourcompany.yourapp`). Do not end it with `.app` (it conflicts with the macOS bundle extension). Changing it after release changes the app's data directory and update identity, so treat it as permanent.
- Set `visible: false` on the main window and show it from the frontend once it has rendered (see the reference `api.ts`) to avoid a white flash. This needs the `core:window:allow-show` permission.
- Keep `build.beforeDevCommand`, `devUrl`, `beforeBuildCommand`, and `frontendDist` consistent with the frontend tooling. Do not hard-code machine-specific paths.
- Enable only the Cargo features you use (for example `protocol-asset` only if you serve local assets, and never the `devtools` feature in release builds).
- Use the asset protocol only with a narrow scope, and prefer `convertFileSrc` for loading local files into the webview.

## 5. Commands and IPC conventions

1. **Async by default.** Commands without `async` run on the main thread and block the UI and event loop. Use `async fn`, or `#[tauri::command(async)]`, for anything beyond trivial work.
2. **Return `Result<T, E>`** where `E: Serialize`. Async commands that borrow arguments (`&str`, `State<'_, T>`) **must** return `Result`.
3. **One serializable error type** (see `error.rs`) with a stable machine-readable `code` and a safe, user-facing `message`. Log the detailed cause in Rust; do not send it to the frontend.
4. **Naming:** command functions are `snake_case`. Tauri maps frontend `camelCase` argument names to Rust `snake_case` parameters automatically. Serialized structs use `#[serde(rename_all = "camelCase")]`.
5. **Keep commands thin:** validate input, call into plain modules, map errors. No business logic in the command body.
6. **Blocking or CPU-heavy work** goes through `tauri::async_runtime::spawn_blocking` (or a dedicated thread). Never call `std::thread::sleep` or blocking IO directly in an async command.
7. **Shared state:** register with `Builder::manage(...)` once and read with `State<'_, T>`. Requesting a type that was never managed panics at runtime, so keep registration in one place. Use interior mutability (`std::sync::Mutex`) and **do not hold a `std::sync::Mutex` guard across an `.await`**; use an async-aware lock if you must.
8. **Streaming and progress:** use `tauri::ipc::Channel<T>` for ordered, high-throughput data from a command. Use events (`Emitter::emit`, `emit_to`) for occasional notifications that any listener may receive. Return `tauri::ipc::Response` for large binary payloads instead of a JSON array of numbers.
9. **Keep the IPC surface coarse.** Send batches, not thousands of tiny calls. IPC payloads are serialized; do not push megabytes of JSON through repeatedly.
10. **New-command checklist** (all in the same change):
    1. Add the Rust command with input validation and a test.
    2. Register it in `generate_handler!`.
    3. List it in `AppManifest::commands` in `build.rs`.
    4. Grant `allow-<name>` in the correct capability file.
    5. Add a typed wrapper in the frontend API module.
    6. Update shared types if the frontend duplicates any Rust struct (consider generating bindings with a tool such as `tauri-specta` to avoid drift).

## 6. Rust core rules

- No `unwrap()` or `expect()` in code paths reachable at runtime. Reserve `expect` for one-time startup failures (for example `run()`), and return `Err` from `setup` rather than panicking.
- Use `thiserror` for error enums. Do not use `Box<dyn Error>` as a command error type.
- Use the `log` facade with the official log plugin (or `tracing`). Configure a lower level for release builds.
- Do long-lived background work in a task spawned from `setup` using a cloned `AppHandle`. Provide a way to stop it, and clean up in a `RunEvent::ExitRequested` or `Exit` handler.
- Use `app.path()` for platform data locations (`app_data_dir`, `app_config_dir`, `app_log_dir`, `app_cache_dir`). Never write next to the executable or rely on the current working directory.
- Use `std::path::Path`/`PathBuf` APIs, not string concatenation, for paths.
- Gate platform-specific code with `#[cfg(desktop)]`, `#[cfg(mobile)]`, or `#[cfg(target_os = "...")]`, and keep the fallback behavior explicit.
- Run `cargo fmt` and `cargo clippy --all-targets -- -D warnings` before finishing. Fix warnings; do not blanket-`allow` them.

## 7. Frontend rules

- Write the frontend as a normal web app. Reach native features only through the typed API module and official `@tauri-apps/plugin-*` packages.
- Feature-detect Tauri with `isTauri()` from `@tauri-apps/api/core` so the UI can also run in a browser, Storybook, or tests.
- Never render untrusted HTML via `innerHTML`/`dangerouslySetInnerHTML`. Sanitize or render as text. An XSS bug in the webview is a foothold into your capabilities.
- `listen(...)` returns a promise of an unlisten function. Call it on unmount to avoid leaks and duplicate handlers.
- Do not use Node.js APIs or assume Chromium. See §8.
- Do not store secrets in `localStorage`, `IndexedDB`, or the bundle.
- Keep the frontend bundle lean: code-split routes, avoid heavy dependencies, and load large assets lazily.

## 8. Webview compatibility

Tauri uses the operating system's webview, not a bundled browser:

| Platform | Engine                                  | Watch out for                                              |
| -------- | --------------------------------------- | ---------------------------------------------------------- |
| Windows  | WebView2 (Chromium-based, evergreen)    | Runtime must be present; choose a `webviewInstallMode`     |
| macOS/iOS| WKWebView (WebKit)                      | Tied to the OS version; some web APIs lag Chromium         |
| Linux    | WebKitGTK                               | Version varies by distro; verify features on older LTS     |
| Android  | System WebView                          | Varies by device and update state                          |

Rules:

- Do not assume the newest web platform features exist everywhere. Check compatibility, use a sensible build target/browserslist, and provide fallbacks.
- Test on **every** OS you ship, not just the developer's machine. Rendering, fonts, scrolling, drag-and-drop, and printing differ.
- Do not rely on DevTools-only behavior. Verify behavior in a release build.

## 9. Windows, menus, tray, and lifecycle

- Use `tauri-plugin-single-instance` (desktop only, registered **first**) unless the app is intentionally multi-instance. Focus the existing window in its callback.
- Persist and restore window size and position with the window-state plugin. Handle monitors that are no longer connected.
- Build menus with `tauri::menu`. Include standard Edit menu items on macOS so copy/paste shortcuts work, and prefer predefined items over custom handlers where they exist.
- Follow platform conventions: on macOS, apps often stay running with no windows and reopen on activation; on Windows and Linux, closing the last window usually quits. If you add close-to-tray, make quitting explicit and discoverable.
- Use the deep-link plugin for custom URL schemes and validate every incoming URL.
- Keep accessibility working: semantic HTML, keyboard navigation, focus management, and labels.

## 10. Plugins

- Prefer official plugins from `tauri-apps/plugins-workspace` over custom code.
- Adding a plugin always touches four places: the Rust dependency, `.plugin(...)` in `run()`, the npm package (if it has a JS API), and the **capability permissions**. A plugin without permissions is unreachable from the frontend by design.
- Grant plugin permissions with scopes and the narrowest identifiers available.
- Write a custom plugin (with its own permissions) when a native capability is reused across windows or projects, rather than growing a large `commands.rs`.
- Confirm desktop vs. mobile support in the plugin's docs before using it on both.

## 11. Data storage

- Small preferences and non-secret settings: the store plugin or a JSON/TOML file under `app_config_dir`, written atomically (write a temp file, then rename). Version the schema and write migrations.
- Structured or large data: SQLite from Rust (for example `rusqlite` or `sqlx`) or the SQL plugin with tight permissions. The database handle lives in Rust state, never in the frontend.
- Keep frontend caches disposable. The source of truth for important data lives in Rust.
- Secrets: OS credential store only (see §3.7).

## 12. Performance and binary size

- Add the release profile settings in the reference `Cargo.toml` (`lto`, `codegen-units = 1`, `opt-level = "s"`, `strip`). `panic = "abort"` shrinks the binary but disables unwinding, so keep it only if you do not depend on catching panics.
- Keep `setup` fast. Defer heavy initialization to background tasks and show the window when the UI is ready.
- Measure before optimizing: use the webview's DevTools Performance panel for the frontend, and `cargo flamegraph`/`tracing` for Rust hot paths.
- Avoid chatty IPC and large JSON payloads (see §5). Use `Channel` or raw responses for big or streaming data.
- Trim dependencies. Every crate and npm package adds build time, binary size, and attack surface.

## 13. Mobile (only if the project targets iOS/Android)

- Keep the standard v2 split: `lib.rs` with `#[cfg_attr(mobile, tauri::mobile_entry_point)]` on `run()`, `main.rs` calling into it, and `crate-type = ["staticlib", "cdylib", "rlib"]` in `[lib]`.
- Use `#[cfg(desktop)]` for desktop-only plugins (single-instance, window-state, global shortcuts) and `platforms` in capabilities to scope permissions per OS.
- Test on real devices and simulators. Mobile has different lifecycle, permissions prompts, and webview versions.
- Android needs a keystore for release builds; iOS needs provisioning and signing. Keep credentials in CI secrets.

## 14. Build, signing, and release

- **Sign every production build.** macOS: Developer ID signing plus notarization. Windows: a code-signing certificate or a supported cloud signing service. Linux: sign release artifacts and checksums as your distribution requires. Keep all credentials in CI secrets.
- **Updater:** Use the official updater plugin. Generate a signing key pair with the Tauri CLI, embed only the **public** key in `tauri.conf.json`, set `bundle.createUpdaterArtifacts`, and provide the private key to CI through environment variables. Never commit the private key. Serve update manifests over HTTPS. The updater signature is separate from OS code signing; you need both.
- Test the whole update path (old release to new release) on every platform before publishing.
- Windows: decide the WebView2 install strategy (`webviewInstallMode`) deliberately. Linux CI needs the WebKitGTK and related system packages installed.
- Use `tauri-apps/tauri-action` (or an equivalent CI job) to build per-platform artifacts from a clean checkout.
- Use separate identifiers and update channels for beta and stable so they do not overwrite each other's data or updates.
- Bump versions in `Cargo.toml` and `package.json` together (and keep `tauri.conf.json` inheriting the Cargo version). Update `Cargo.lock`.

## 15. Testing

- **Rust unit tests** for validators, services, and parsers (see the tests in the reference `commands.rs`). Keep logic independent of `tauri` so tests do not need a runtime.
- **Command/integration tests** can use Tauri's `tauri::test` mock runtime to build a mock app and exercise commands and state.
- **Frontend tests** should mock the IPC layer with `mockIPC` from `@tauri-apps/api/mocks` rather than calling real Rust.
- **End-to-end tests** use WebDriver through `tauri-driver`, which supports Windows and Linux desktop builds. There is no macOS desktop driver, so cover macOS with manual/smoke checks or CI runs of the packaged app.
- Add tests for every new command: valid input, invalid input, boundary sizes, and the failure path.
- Add a regression test when fixing a security bug.
- Always verify anything touching capabilities, CSP, paths, sidecars, or the updater in a **release/packaged** build. Dev builds hide policy and packaging bugs.

## 16. Reference implementation

Complete minimal files showing the required patterns. Adapt names to the project; do not remove the security controls. Notes:

- The `_lib` suffix on the library name avoids a bin/lib name collision that can occur on Windows.
- `identifier` and `productName` are examples. Replace them with your real values before releasing.
- Add icons under `src-tauri/icons/` (generate them with `tauri icon`).
- Add makers/targets, signing, and updater settings as your release plan requires.

```toml src-tauri/Cargo.toml
[package]
name = "app"
version = "0.1.0"
description = "A Tauri application"
edition = "2021"
rust-version = "1.77.2"

[lib]
# The `_lib` suffix keeps the library name distinct from the binary name.
# This avoids a name collision that appears on Windows.
name = "app_lib"
crate-type = ["staticlib", "cdylib", "rlib"]

[build-dependencies]
tauri-build = { version = "2", features = [] }

[dependencies]
tauri = { version = "2", features = [] }
tauri-plugin-log = "2"
serde = { version = "1", features = ["derive"] }
thiserror = "2"
log = "0.4"

# Desktop-only plugins. Not available on Android or iOS.
[target.'cfg(not(any(target_os = "android", target_os = "ios")))'.dependencies]
tauri-plugin-single-instance = "2"

[profile.release]
codegen-units = 1   # Better optimization at the cost of compile time
lto = true          # Link-time optimization
opt-level = "s"     # Optimize for size
panic = "abort"     # Smaller binary; disables unwinding
strip = true        # Remove debug symbols
```

```rust src-tauri/build.rs
// Build script. Registers every app command so that each one requires an
// explicit `allow-<command>` permission in a capability file.
// When you add a command, add its name here AND grant it in capabilities.
fn main() {
    tauri_build::try_build(
        tauri_build::Attributes::new().app_manifest(
            tauri_build::AppManifest::new().commands(&[
                "get_app_info",
                "save_note",
                "list_notes",
                "run_job",
            ]),
        ),
    )
    .expect("failed to run the Tauri build script");
}
```

```json src-tauri/tauri.conf.json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "app",
  "identifier": "com.example.desktop",
  "build": {
    "beforeDevCommand": "npm run dev",
    "devUrl": "http://localhost:1420",
    "beforeBuildCommand": "npm run build",
    "frontendDist": "../dist"
  },
  "app": {
    "withGlobalTauri": false,
    "windows": [
      {
        "label": "main",
        "title": "app",
        "width": 1000,
        "height": 700,
        "visible": false
      }
    ],
    "security": {
      "freezePrototype": true,
      "csp": {
        "default-src": "'self'",
        "connect-src": "ipc: http://ipc.localhost",
        "img-src": "'self' data:",
        "style-src": "'self' 'unsafe-inline'",
        "object-src": "'none'",
        "base-uri": "'none'",
        "frame-ancestors": "'none'"
      }
    }
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ]
  }
}
```

```json src-tauri/capabilities/default.json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "main-window",
  "description": "Minimal permissions for the main window only",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "core:window:allow-show",
    "allow-get-app-info",
    "allow-save-note",
    "allow-list-notes",
    "allow-run-job"
  ]
}
```

```rust src-tauri/src/main.rs
// Prevents an extra console window on Windows in release builds. DO NOT REMOVE.
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    app_lib::run();
}
```

```rust src-tauri/src/error.rs
use serde::{ser::SerializeStruct, Serialize, Serializer};

/// Result alias used by all commands.
pub type AppResult<T> = Result<T, AppError>;

/// The single error type returned to the frontend.
///
/// Only a stable `code` and a safe `message` are serialized. Detailed causes
/// (I/O errors, internal failures) are logged in Rust and never sent to the
/// webview.
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    /// The caller sent invalid data. The message is safe to show to the user.
    #[error("{0}")]
    InvalidInput(String),

    #[error("resource not found")]
    NotFound,

    #[error("an I/O error occurred")]
    Io(#[from] std::io::Error),

    /// The String holds a detail for the log only; it is not sent to the UI.
    #[error("an internal error occurred")]
    Internal(String),
}

impl AppError {
    fn code(&self) -> &'static str {
        match self {
            AppError::InvalidInput(_) => "invalid_input",
            AppError::NotFound => "not_found",
            AppError::Io(_) => "io",
            AppError::Internal(_) => "internal",
        }
    }
}

impl Serialize for AppError {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        if matches!(self, AppError::Io(_) | AppError::Internal(_)) {
            // Keep the details in the log; the frontend gets a generic message.
            log::error!("{:?}", self);
        }

        let mut state = serializer.serialize_struct("AppError", 2)?;
        state.serialize_field("code", self.code())?;
        state.serialize_field("message", &self.to_string())?;
        state.end()
    }
}
```

```rust src-tauri/src/state.rs
use std::sync::Mutex;

/// A stored note. Kept private to the backend; the frontend receives summaries.
pub struct Note {
    pub title: String,
    pub body: String,
}

/// Application state registered once with `Builder::manage`.
///
/// Uses `std::sync::Mutex` because locks are never held across an `.await`.
#[derive(Default)]
pub struct AppState {
    pub notes: Mutex<Vec<Note>>,
}
```

```rust src-tauri/src/commands.rs
use serde::Serialize;
use tauri::{ipc::Channel, AppHandle, State};

use crate::error::{AppError, AppResult};
use crate::state::{AppState, Note};

const MAX_TITLE_LEN: usize = 200;
const MAX_BODY_LEN: usize = 10_000;
const MAX_NOTES: usize = 1_000;
const MAX_JOB_STEPS: u32 = 100;

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct AppInfo {
    name: String,
    version: String,
}

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct NoteSummary {
    title: String,
    length: usize,
}

#[derive(Clone, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct JobProgress {
    step: u32,
    total: u32,
}

/// Pure validation, kept free of Tauri types so it is trivially testable.
pub(crate) fn validate_note(title: &str, body: &str) -> AppResult<()> {
    if title.trim().is_empty() {
        return Err(AppError::InvalidInput("title must not be empty".into()));
    }
    if title.chars().count() > MAX_TITLE_LEN {
        return Err(AppError::InvalidInput(format!(
            "title must be at most {MAX_TITLE_LEN} characters"
        )));
    }
    if body.chars().count() > MAX_BODY_LEN {
        return Err(AppError::InvalidInput(format!(
            "body must be at most {MAX_BODY_LEN} characters"
        )));
    }
    Ok(())
}

/// Async so it never blocks the main thread.
#[tauri::command]
pub async fn get_app_info(app: AppHandle) -> AppInfo {
    let info = app.package_info();
    AppInfo {
        name: info.name.clone(),
        version: info.version.to_string(),
    }
}

/// Uses `State<'_, _>`, so this async command must return `Result`.
#[tauri::command]
pub async fn save_note(
    state: State<'_, AppState>,
    title: String,
    body: String,
) -> AppResult<usize> {
    validate_note(&title, &body)?;

    let mut notes = state
        .notes
        .lock()
        .map_err(|_| AppError::Internal("notes lock poisoned".into()))?;

    if notes.len() >= MAX_NOTES {
        return Err(AppError::InvalidInput("note limit reached".into()));
    }

    notes.push(Note {
        title: title.trim().to_owned(),
        body,
    });
    Ok(notes.len())
}

#[tauri::command]
pub async fn list_notes(state: State<'_, AppState>) -> AppResult<Vec<NoteSummary>> {
    let notes = state
        .notes
        .lock()
        .map_err(|_| AppError::Internal("notes lock poisoned".into()))?;

    Ok(notes
        .iter()
        .map(|note| NoteSummary {
            title: note.title.clone(),
            length: note.body.chars().count(),
        })
        .collect())
}

/// Demonstrates streaming progress over a `Channel` and moving blocking work
/// off the async executor with `spawn_blocking`.
#[tauri::command]
pub async fn run_job(steps: u32, on_progress: Channel<JobProgress>) -> AppResult<u32> {
    if steps == 0 || steps > MAX_JOB_STEPS {
        return Err(AppError::InvalidInput(format!(
            "steps must be between 1 and {MAX_JOB_STEPS}"
        )));
    }

    tauri::async_runtime::spawn_blocking(move || {
        for step in 1..=steps {
            // Stand-in for real CPU-bound or blocking work.
            std::thread::sleep(std::time::Duration::from_millis(50));

            on_progress
                .send(JobProgress { step, total: steps })
                .map_err(|error| AppError::Internal(error.to_string()))?;
        }
        Ok(steps)
    })
    .await
    .map_err(|error| AppError::Internal(error.to_string()))?
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rejects_empty_title() {
        assert!(validate_note("   ", "body").is_err());
    }

    #[test]
    fn rejects_oversized_title() {
        let title = "x".repeat(MAX_TITLE_LEN + 1);
        assert!(validate_note(&title, "body").is_err());
    }

    #[test]
    fn rejects_oversized_body() {
        let body = "x".repeat(MAX_BODY_LEN + 1);
        assert!(validate_note("Title", &body).is_err());
    }

    #[test]
    fn accepts_valid_note() {
        assert!(validate_note("Title", "Body").is_ok());
    }
}
```

```rust src-tauri/src/lib.rs
mod commands;
mod error;
mod state;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    let builder = tauri::Builder::default();

    // Desktop only. The single-instance plugin must be registered first.
    #[cfg(desktop)]
    let builder = builder.plugin(tauri_plugin_single_instance::init(|app, _argv, _cwd| {
        use tauri::Manager;

        if let Some(window) = app.get_webview_window("main") {
            let _ = window.unminimize();
            let _ = window.show();
            let _ = window.set_focus();
        }
    }));

    let log_level = if cfg!(debug_assertions) {
        log::LevelFilter::Debug
    } else {
        log::LevelFilter::Info
    };

    builder
        .plugin(
            tauri_plugin_log::Builder::default()
                .level(log_level)
                .build(),
        )
        .manage(state::AppState::default())
        .invoke_handler(tauri::generate_handler![
            commands::get_app_info,
            commands::save_note,
            commands::list_notes,
            commands::run_job,
        ])
        .setup(|_app| {
            log::info!("application setup complete");
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running the Tauri application");
}
```

```ts src/lib/api.ts
// The ONLY module that talks to the Rust backend.
// Components import these typed functions; they never call `invoke` directly.
import { Channel, invoke, isTauri } from '@tauri-apps/api/core';
import { getCurrentWindow } from '@tauri-apps/api/window';

export interface AppInfo {
  name: string;
  version: string;
}

export interface NoteSummary {
  title: string;
  length: number;
}

export interface JobProgress {
  step: number;
  total: number;
}

export type AppErrorCode = 'invalid_input' | 'not_found' | 'io' | 'internal';

/** Error thrown by every wrapper in this module. */
export class CommandError extends Error {
  readonly code: AppErrorCode;

  constructor(code: AppErrorCode, message: string) {
    super(message);
    this.name = 'CommandError';
    this.code = code;
  }
}

function isErrorPayload(value: unknown): value is { code: AppErrorCode; message: string } {
  return (
    typeof value === 'object' &&
    value !== null &&
    'code' in value &&
    'message' in value &&
    typeof (value as { code: unknown }).code === 'string' &&
    typeof (value as { message: unknown }).message === 'string'
  );
}

async function call<T>(command: string, args?: Record<string, unknown>): Promise<T> {
  if (!isTauri()) {
    throw new CommandError('internal', 'Desktop features are unavailable outside the app.');
  }

  try {
    return await invoke<T>(command, args);
  } catch (error) {
    if (isErrorPayload(error)) {
      throw new CommandError(error.code, error.message);
    }
    throw new CommandError('internal', 'Unexpected error');
  }
}

export function getAppInfo(): Promise<AppInfo> {
  return call<AppInfo>('get_app_info');
}

export function saveNote(title: string, body: string): Promise<number> {
  return call<number>('save_note', { title, body });
}

export function listNotes(): Promise<NoteSummary[]> {
  return call<NoteSummary[]>('list_notes');
}

/** Runs a job in Rust and reports progress through a Channel. */
export function runJob(steps: number, onProgress: (progress: JobProgress) => void): Promise<number> {
  const channel = new Channel<JobProgress>();
  channel.onmessage = onProgress;
  // Rust parameter `on_progress` maps to the camelCase key `onProgress`.
  return call<number>('run_job', { steps, onProgress: channel });
}

/**
 * The main window is created hidden (see tauri.conf.json).
 * Call this once the first UI render has completed to avoid a white flash.
 */
export async function showMainWindow(): Promise<void> {
  if (!isTauri()) {
    return;
  }
  await getCurrentWindow().show();
}
```

## 17. Workflow for agents

**Before you change code**

1. Read `tauri.conf.json`, every file in `src-tauri/capabilities/`, `build.rs`, `lib.rs`, and the frontend API module. Check whether the capability you need already exists.
2. Decide which side each change belongs to. A feature that crosses the boundary needs a command, registration, manifest entry, permission, frontend wrapper, and tests together.

**While you work**

- Make the smallest change that solves the task. Do not refactor unrelated code or bump dependency minors as a side effect.
- Prefer official plugins and documented Tauri APIs over custom or clever workarounds.
- Use the v2 APIs in the table in §1. If a snippet you remember uses `allowlist` or `@tauri-apps/api/tauri`, it is v1 and wrong here.

**Before you finish**

- [ ] `cargo fmt`, `cargo clippy --all-targets -- -D warnings`, and `cargo test` pass.
- [ ] Frontend typecheck, lint, and tests pass using the scripts in `package.json`.
- [ ] Every new command validates its input in Rust, returns a serializable error, and is async unless trivially cheap.
- [ ] Every new command is in `generate_handler!`, in `build.rs`, and granted in the correct capability, and nowhere else.
- [ ] No new broad permission, wildcard scope, `remote` capability, `dangerous*` config key, `'unsafe-eval'`, or `withGlobalTauri: true`.
- [ ] No frontend-supplied string is used to build a path, shell command, URL, or SQL statement without validation.
- [ ] No secrets, tokens, keys, or credentials in source, logs, fixtures, or the frontend bundle.
- [ ] New dependencies are justified, maintained, and audited, and Tauri crate/package versions are still aligned.
- [ ] Changes to capabilities, CSP, paths, sidecars, or the updater were checked in a **release** build.

**Never do these**

- Loosen a capability, scope, or CSP to get a feature working.
- Call raw `invoke('...')` from components instead of the API module.
- Block the main thread with a synchronous command that does IO or heavy computation.
- Return raw error strings, stack traces, or absolute paths to the frontend.
- Load remote content into a window that has IPC access.
- Commit `.env` files, private updater keys, certificates, keystores, or signing credentials.
- Claim a fix works without running it. If you could not build or run the app, say so.

## 18. Further reading

- Security overview: https://v2.tauri.app/security/
- Capabilities: https://v2.tauri.app/security/capabilities/
- Permissions: https://v2.tauri.app/security/permissions/
- Content Security Policy: https://v2.tauri.app/security/csp/
- Calling Rust from the frontend: https://v2.tauri.app/develop/calling-rust/
- Calling the frontend from Rust (events, channels): https://v2.tauri.app/develop/calling-frontend/
- Plugins: https://v2.tauri.app/plugin/
- Updater plugin: https://v2.tauri.app/plugin/updater/
- Testing: https://v2.tauri.app/develop/tests/
- Signing and distribution: https://v2.tauri.app/distribute/
- Migrating from Tauri 1: https://v2.tauri.app/start/migrate/from-tauri-1/

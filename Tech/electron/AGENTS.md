# AGENTS.md — Electron Best Practices

This file tells AI coding agents (and humans) how to work in this Electron codebase. Follow it unless a maintainer explicitly overrides a rule in the task description. When a rule here conflicts with a quick fix, the rule wins.

> **Treat the renderer as an untrusted web page.** Almost every rule below follows from that one idea. Electron gives web content a path to the operating system; our job is to keep that path as narrow as possible.

---

## 1. Project assumptions

- **Language:** TypeScript with `strict` enabled. No `any` in IPC, preload, or main-process code.
- **Electron:** Stay on a currently supported stable major (Electron supports the latest three stable majors). Do not pin to an old major to avoid a migration. Chromium security fixes ship through Electron upgrades.
- **Module format:** Main process may be ESM or CJS. **Sandboxed preload scripts must be bundled to CommonJS.**
- **Bundling:** Main, preload, and renderer are built as three separate bundles. Preload must be bundled (not `require`d from loose files) because a sandboxed preload cannot load arbitrary local modules.
- **Packaging:** Electron Forge (with `@electron/fuses`) or electron-builder. Check `package.json` and the existing config before adding either.
- **Commands:** Do not invent scripts. Read `package.json` and use what exists. Typical scripts are `dev`, `build`, `typecheck`, `lint`, `test`, `test:e2e`, and `package`/`make`. If a script you need does not exist, say so instead of guessing.

## 2. Process model and project layout

Electron has three kinds of code with different trust levels. Keep them in separate directories and never import across the boundary except for `shared/`.

| Directory        | Runs in                    | Trust     | May use                                        |
| ---------------- | -------------------------- | --------- | ---------------------------------------------- |
| `src/main/`      | Main process (Node.js)     | Trusted   | Node, Electron main APIs, filesystem, native modules |
| `src/preload/`   | Preload (isolated world)   | Bridge    | `contextBridge`, `ipcRenderer` only            |
| `src/renderer/`  | Chromium renderer          | Untrusted | Web APIs and the `window.desktop` bridge only  |
| `src/shared/`    | Both                       | n/a       | Pure types, constants, and validators. No Node/Electron imports |

Rules:

1. The renderer never imports from `electron`, `node:*`, or `src/main/`.
2. The main process never imports from `src/renderer/`.
3. `src/shared/` contains no side effects and no runtime dependency on Electron or Node.
4. Heavy or blocking work does not run in the main process (see §8).

## 3. Security rules (non-negotiable)

Never weaken any of these to make a feature "just work". If a feature seems to require it, stop and propose a safer design.

### 3.1 `BrowserWindow` defaults

Every window must be created with:

- `contextIsolation: true`
- `nodeIntegration: false`
- `sandbox: true`
- `webSecurity: true` (never set to `false`)
- `allowRunningInsecureContent: false`
- `nodeIntegrationInWorker: false` and `nodeIntegrationInSubFrames: false` (the defaults; do not change)

Do **not** use `enableRemoteModule` / `@electron/remote`, the `<webview>` tag, or `webPreferences.experimentalFeatures`. Use `WebContentsView` (not the deprecated `BrowserView`) when embedding web content.

### 3.2 Preload and `contextBridge`

- Expose a **small, named, purpose-built API** through `contextBridge.exposeInMainWorld`. One function per capability.
- **Never** expose `ipcRenderer`, `ipcRenderer.send`, `ipcRenderer.invoke`, `require`, `process`, or a generic "send any channel" helper.
- Strip the `IpcRendererEvent` from listener callbacks. Renderer callbacks receive data only.
- Return an unsubscribe function from every `on*` subscription so the renderer can clean up.

### 3.3 IPC validation

- Prefer `ipcMain.handle` / `ipcRenderer.invoke` (request/response). Do not use `sendSync`; it blocks the renderer.
- **Validate the sender** in every handler: check `event.senderFrame` origin/URL against what you expect. Reject and throw otherwise.
- **Validate every payload** in the handler as if it came from an attacker. Check type, shape, length, and allowed values. Do not trust TypeScript types at the IPC boundary; they are erased at runtime.
- Define channel names once in `src/shared/ipc.ts`. No string literals for channels elsewhere.
- Never build file paths, shell commands, SQL, or URLs from raw renderer input. Resolve paths against an allowed base directory and reject anything that escapes it.

### 3.4 Navigation and new windows

- Register `setWindowOpenHandler` and a `will-navigate` handler on **every** `webContents` (do it once via `app.on('web-contents-created')`).
- Default to `{ action: 'deny' }` for new windows. Open external links in the user's browser only after validating the URL.
- Only pass `https:` (and, if needed, `mailto:`) URLs to `shell.openExternal`. Never pass unvalidated URLs; `file:`, custom schemes, and `javascript:` URLs can execute code.
- Do not load remote content into a window that has a privileged preload. If remote content is unavoidable, use a separate window/session with **no** preload and no bridge.

### 3.5 Permissions, certificates, sessions

- Set `session.setPermissionRequestHandler` and `session.setPermissionCheckHandler` to **deny by default**, then allow specific permissions for specific origins.
- Never call `event.preventDefault()` on `certificate-error` to ignore TLS errors, and never use `--ignore-certificate-errors`.
- Use a dedicated `session.fromPartition('persist:name')` for any window that loads different content than the main app.

### 3.6 Content Security Policy

- Ship a strict CSP for the production renderer. Start from `default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'`.
- No `'unsafe-inline'` or `'unsafe-eval'` for scripts in production. Dev servers may need relaxed rules; apply the strict policy only when `app.isPackaged` is true, and test the packaged build.

### 3.7 Loading local content

- Prefer serving the renderer through a **custom protocol** registered with `protocol.handle` over raw `file://`. `protocol.registerFileProtocol` and its siblings are deprecated.
- If you do use `file://`, only load the known `index.html`, and validate paths in any handler.

### 3.8 Fuses (production hardening)

Flip Electron fuses at package time using `@electron/fuses`. At minimum:

- `RunAsNode: false`
- `EnableNodeOptionsEnvironmentVariable: false`
- `EnableNodeCliInspectArguments: false`
- `EnableEmbeddedAsarIntegrityValidation: true`
- `OnlyLoadAppFromAsar: true`
- `EnableCookieEncryption: true`

Also package with `asar: true`. See the reference `forge.config.ts` in §14.

### 3.9 Secrets and dependencies

- Never hard-code secrets, API keys, or signing credentials in the app. Anything shipped in the app bundle is public.
- Store user secrets with `safeStorage` (check `safeStorage.isEncryptionAvailable()` first; on Linux also check `getSelectedStorageBackend()` and treat `basic_text` as unencrypted). Do not use `localStorage`, plain JSON files, or `electron-store` alone for tokens.
- Do not add a dependency to the **main** process without checking that it is maintained, has few transitive dependencies, and does not run install scripts you cannot audit. Run `npm audit` (or your package manager's equivalent) when dependencies change.
- Never `eval`, `new Function`, or `vm.runInThisContext` on data that originated outside the app.

## 4. IPC conventions

1. **Name channels** as `domain:action` in kebab-case, e.g. `settings:set-theme`, `files:read-text`.
2. **One handler per channel**, registered in `src/main/ipc/` (one file per domain when the app grows).
3. **Request/response** uses `handle`/`invoke`. **Main → renderer pushes** use `webContents.send` and are exposed in preload as a subscription.
4. Payloads must be **structured-clone serializable** (plain objects, arrays, strings, numbers, booleans, `Uint8Array`). No class instances, functions, or DOM nodes.
5. Errors thrown in a handler reach the renderer as a rejected promise with only a message. Do not leak stack traces, absolute paths, or secrets in error messages.
6. Add a shared type for every request and response in `src/shared/`, and a runtime validator for every request.
7. Keep handlers thin. Put logic in plain, unit-testable modules that do not import `electron`.

## 5. Main process rules

- **Never block the event loop.** No `*Sync` filesystem calls after startup, no CPU-heavy loops, no synchronous child processes. Use async APIs.
- Do CPU-heavy or crash-prone work in a `utilityProcess` (or a worker thread for pure JS). Use `child_process` only when you need an external binary, and always pass an argument array, never a concatenated shell string.
- Create windows **only after** `app.whenReady()`.
- Use `app.requestSingleInstanceLock()` unless the app is intentionally multi-instance. Handle `second-instance` to focus the existing window and to receive deep-link arguments.
- Quit behavior: on `window-all-closed`, call `app.quit()` except on macOS (`process.platform === 'darwin'`). On macOS, re-create a window on `activate` when none exist.
- Store user data under `app.getPath('userData')` and temp data under `app.getPath('temp')`. Never write into the install directory or `app.getAppPath()`.
- Do not assume `__dirname` is a real directory on disk when packaged; files inside `app.asar` cannot be executed or opened by external programs. Put native binaries and assets that must be executed in `extraResource`/`asarUnpack`, and resolve them via `process.resourcesPath`.
- Catch and log `uncaughtException` and `unhandledRejection` at the top level. Do not silently swallow them.
- Use `app.isPackaged` (not `NODE_ENV`) to decide dev vs. production behavior for anything security-relevant.

## 6. Preload rules

- Keep preload tiny. It should contain only the `contextBridge` surface and thin `ipcRenderer` calls.
- No business logic, no heavy imports, no Node APIs other than what Electron exposes to sandboxed preloads (`contextBridge`, `ipcRenderer`, `webFrame`, `crashReporter`, and a limited `process`).
- Export the bridge's type (`DesktopApi`) so the renderer gets typed access without importing runtime code from preload.

## 7. Renderer rules

- Write the renderer as a normal web app that could also run in a browser. Access desktop features only through `window.desktop`.
- Feature-detect the bridge (`window.desktop` may be absent in Storybook, browser dev, or tests) and degrade gracefully.
- Never render untrusted HTML with `innerHTML`/`dangerouslySetInnerHTML`. Sanitize or use text nodes.
- Do not depend on Node globals (`process`, `Buffer`, `__dirname`) in renderer code or its dependencies.
- Unsubscribe from bridge events on unmount to avoid leaks.

## 8. Performance

- **Startup:** Create the window with `show: false` and call `win.show()` on `ready-to-show` to avoid a white flash. Defer non-critical work until after the first window is shown. Lazy-`import()` heavy modules in main.
- Measure before optimizing. Use `console.time`/`performance.now` around startup phases and the Chromium DevTools Performance tab for the renderer.
- Keep the IPC surface **coarse-grained**. Send batches, not thousands of tiny messages. Never stream large binary blobs through IPC in a tight loop; use `MessagePort` (`MessageChannelMain`) or files for large data.
- Avoid loading large native or JS dependencies in the main process "just in case".
- Do not run animations or timers in hidden windows. Pause work on `blur`/`hide` when possible, and set `backgroundThrottling` deliberately (it is on by default; only disable it with a documented reason).
- Bundle and tree-shake the renderer. Keep the packaged app lean by excluding dev dependencies, source maps you do not ship, and unused locales.

## 9. Windows, menus, and platform behavior

- Persist and restore window bounds. Validate saved bounds against currently connected displays before applying them.
- Provide a real application menu on macOS (Edit menu roles are required for copy/paste shortcuts to work) and keep standard accelerators. Prefer `role`-based menu items over custom handlers when a role exists.
- **Windows:** call `app.setAppUserModelId(...)` early so notifications and taskbar grouping work. Handle installer/Squirrel startup events if using Squirrel.
- **macOS:** respect the app-stays-alive-without-windows convention, and support dark mode via `nativeTheme`.
- **Linux:** do not assume a system tray, a keyring, or a specific desktop environment. Feature-check and degrade gracefully.
- Use `path.join`/`path.resolve` and `node:path`. Never build paths with string concatenation or hard-coded separators.
- For deep links: call `app.setAsDefaultProtocolClient`, handle `open-url` on macOS, and read `argv` from `second-instance` on Windows/Linux. Treat deep-link data as untrusted input.
- Support accessibility: keyboard navigation, focus management, and screen-reader labels in the renderer; do not disable Chromium's accessibility features.

## 10. Data storage

- Small settings: a JSON file or `electron-store` in `userData`, written atomically (write to a temp file, then rename). Version your schema and write migrations.
- Structured or large data: SQLite via a maintained binding, opened **in the main process or a utility process**, never from the renderer.
- Keep any renderer-side cache (`localStorage`, IndexedDB) disposable. The source of truth for anything important lives in main.
- Do not store secrets in any of the above. See §3.9.

## 11. Native modules

- Rebuild native modules against Electron's ABI with `@electron/rebuild` (Forge does this automatically). A module compiled for plain Node will crash in Electron.
- Prefer N-API/Node-API modules or prebuilt binaries (`prebuildify`) so they survive Electron upgrades.
- Unpack native `.node` files from asar (`asarUnpack`) when the packager requires it.
- Test the **packaged** app on each target OS/architecture. Native failures often appear only in packaged builds.

## 12. Packaging, signing, and updates

- **Sign and notarize** every release build. macOS: Developer ID signing plus notarization and hardened runtime with the minimum required entitlements. Windows: code-signing certificate (or a cloud signing service). Never ship unsigned production builds.
- Keep signing credentials in CI secrets. Never commit certificates, keys, or Apple credentials.
- Auto-update over HTTPS from a location you control, using Electron's `autoUpdater` (for example via a hosted update service) or `electron-updater`. Verify signatures; do not roll your own downloader. Note that Electron's built-in `autoUpdater` does not support Linux.
- Test the full update path (old version → new version) on every platform before releasing.
- Release channels (stable/beta) must use distinct feeds and, ideally, distinct app IDs and `userData` directories.
- Build reproducibly in CI from a clean checkout with a locked dependency file (`package-lock.json`, `pnpm-lock.yaml`, or `yarn.lock`).

## 13. Testing, logging, and diagnostics

**Testing**

- Unit test pure logic (validators, services, reducers) with the project's test runner. Keep Electron imports out of that logic so no Electron runtime is needed.
- End-to-end tests use Playwright's Electron support (`_electron.launch`) against a built app. Cover: app launches, main window loads, one IPC round trip, and quit.
- Add a test for every new IPC handler that covers valid input, invalid input, and an untrusted sender.
- Add a regression test whenever a security bug is fixed.

**Logging**

- Use one logger in main (for example `electron-log` or `pino`) writing to `app.getPath('logs')`. Do not log secrets, tokens, or full user file paths at info level.
- Forward renderer errors to main through a narrow bridge method if you need them in the log file.

**Crash and error reporting**

- Enable `crashReporter` (and/or a service such as Sentry) with user consent and an explicit privacy policy. Scrub personal data before upload.
- Handle `render-process-gone` and `child-process-gone`: log the reason, then reload or recover the window gracefully.

## 14. Reference implementation

These complete, minimal files show the required patterns. Adapt names to the project; do not remove the security controls.

```ts src/shared/ipc.ts
// Single source of truth for IPC channel names and payload validation.
// This file must stay free of Electron and Node imports.

export const IPC_CHANNELS = {
  getVersion: 'app:get-version',
  setTheme: 'settings:set-theme',
  updateAvailable: 'updates:available',
} as const;

export const THEMES = ['light', 'dark', 'system'] as const;
export type Theme = (typeof THEMES)[number];

export function isTheme(value: unknown): value is Theme {
  return typeof value === 'string' && (THEMES as readonly string[]).includes(value);
}
```

```ts src/preload/index.ts
// Preload script. Runs in an isolated world with access to a limited Electron API.
// Bundle this file to CommonJS so it works with `sandbox: true`.
import { contextBridge, ipcRenderer } from 'electron';
import { IPC_CHANNELS, type Theme } from '../shared/ipc';

const api = {
  getVersion: (): Promise<string> => ipcRenderer.invoke(IPC_CHANNELS.getVersion),

  setTheme: (theme: Theme): Promise<void> =>
    ipcRenderer.invoke(IPC_CHANNELS.setTheme, theme),

  // Subscription helper: strips the IpcRendererEvent and returns an unsubscribe function.
  onUpdateAvailable: (callback: (version: string) => void): (() => void) => {
    const listener = (_event: Electron.IpcRendererEvent, version: string): void => {
      callback(version);
    };
    ipcRenderer.on(IPC_CHANNELS.updateAvailable, listener);
    return () => {
      ipcRenderer.removeListener(IPC_CHANNELS.updateAvailable, listener);
    };
  },
};

export type DesktopApi = typeof api;

contextBridge.exposeInMainWorld('desktop', api);
```

```ts src/renderer/global.d.ts
// Types the bridge exposed by the preload script.
// Type-only import: nothing from preload is bundled into the renderer.
import type { DesktopApi } from '../preload/index';

declare global {
  interface Window {
    // Optional because the renderer may also run outside Electron (browser dev, Storybook, tests).
    desktop?: DesktopApi;
  }
}

export {};
```

```ts src/main/index.ts
// Main process entry point. Assumes the main bundle is emitted as CommonJS
// (so `__dirname` is available) and that the preload bundle is emitted as
// `out/preload/index.js` next to `out/main/index.js`.
import { app, BrowserWindow, ipcMain, nativeTheme, session, shell } from 'electron';
import path from 'node:path';
import { pathToFileURL } from 'node:url';
import { IPC_CHANNELS, isTheme } from '../shared/ipc';

const isDev = !app.isPackaged;
// Dev server URL is provided by the dev tooling (for example Vite). Empty in production.
const DEV_SERVER_URL = process.env.VITE_DEV_SERVER_URL ?? '';
const RENDERER_INDEX = path.join(__dirname, '../renderer/index.html');
const RENDERER_INDEX_URL = pathToFileURL(RENDERER_INDEX).toString();

let mainWindow: BrowserWindow | null = null;

// ---------------------------------------------------------------------------
// Trust helpers
// ---------------------------------------------------------------------------

/** True if the URL points at our own renderer (dev server or packaged index.html). */
function isAppUrl(rawUrl: string): boolean {
  try {
    const url = new URL(rawUrl);
    if (isDev && DEV_SERVER_URL) {
      return url.origin === new URL(DEV_SERVER_URL).origin;
    }
    return url.protocol === 'file:' && url.origin + url.pathname === RENDERER_INDEX_URL;
  } catch {
    return false;
  }
}

/** Every IPC handler must call this first. */
function assertTrustedSender(frame: Electron.WebFrameMain | null): void {
  if (!frame || !isAppUrl(frame.url)) {
    throw new Error('IPC call rejected: untrusted sender');
  }
}

/** Only https (and mailto) links may be opened in the user's default handler. */
function isSafeExternalUrl(rawUrl: string): boolean {
  try {
    const { protocol } = new URL(rawUrl);
    return protocol === 'https:' || protocol === 'mailto:';
  } catch {
    return false;
  }
}

// ---------------------------------------------------------------------------
// Global hardening: applies to every webContents, including future ones
// ---------------------------------------------------------------------------

app.on('web-contents-created', (_event, contents) => {
  contents.setWindowOpenHandler(({ url }) => {
    if (isSafeExternalUrl(url)) {
      void shell.openExternal(url);
    }
    return { action: 'deny' };
  });

  contents.on('will-navigate', (event, url) => {
    if (!isAppUrl(url)) {
      event.preventDefault();
    }
  });

  // <webview> is never allowed.
  contents.on('will-attach-webview', (event) => {
    event.preventDefault();
  });
});

function configureSession(): void {
  const ses = session.defaultSession;

  // Deny every permission by default; allow specific ones deliberately.
  ses.setPermissionRequestHandler((_webContents, _permission, callback) => {
    callback(false);
  });
  ses.setPermissionCheckHandler(() => false);

  // Strict CSP in production only (dev servers need looser rules for HMR).
  if (!isDev) {
    ses.webRequest.onHeadersReceived((details, callback) => {
      callback({
        responseHeaders: {
          ...details.responseHeaders,
          'Content-Security-Policy': [
            "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; " +
              "img-src 'self' data:; object-src 'none'; base-uri 'none'; frame-ancestors 'none'",
          ],
        },
      });
    });
  }
}

// ---------------------------------------------------------------------------
// IPC handlers: validate the sender AND the payload
// ---------------------------------------------------------------------------

function registerIpcHandlers(): void {
  ipcMain.handle(IPC_CHANNELS.getVersion, (event) => {
    assertTrustedSender(event.senderFrame);
    return app.getVersion();
  });

  ipcMain.handle(IPC_CHANNELS.setTheme, (event, theme: unknown) => {
    assertTrustedSender(event.senderFrame);
    if (!isTheme(theme)) {
      throw new Error('Invalid theme');
    }
    nativeTheme.themeSource = theme;
  });
}

// ---------------------------------------------------------------------------
// Window creation
// ---------------------------------------------------------------------------

function createMainWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    show: false, // show on 'ready-to-show' to avoid a white flash
    webPreferences: {
      preload: path.join(__dirname, '../preload/index.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
      webSecurity: true,
      allowRunningInsecureContent: false,
    },
  });

  win.once('ready-to-show', () => win.show());

  win.webContents.on('render-process-gone', (_event, details) => {
    console.error('Renderer process gone:', details.reason);
  });

  if (isDev && DEV_SERVER_URL) {
    void win.loadURL(DEV_SERVER_URL);
  } else {
    void win.loadFile(RENDERER_INDEX);
  }

  win.on('closed', () => {
    mainWindow = null;
  });

  return win;
}

// ---------------------------------------------------------------------------
// App lifecycle
// ---------------------------------------------------------------------------

process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
});
process.on('unhandledRejection', (reason) => {
  console.error('Unhandled rejection:', reason);
});

const hasSingleInstanceLock = app.requestSingleInstanceLock();

if (!hasSingleInstanceLock) {
  app.quit();
} else {
  app.on('second-instance', () => {
    if (mainWindow) {
      if (mainWindow.isMinimized()) mainWindow.restore();
      mainWindow.focus();
    }
  });

  void app.whenReady().then(() => {
    if (process.platform === 'win32') {
      app.setAppUserModelId(app.getName());
    }

    configureSession();
    registerIpcHandlers();
    mainWindow = createMainWindow();

    app.on('activate', () => {
      if (BrowserWindow.getAllWindows().length === 0) {
        mainWindow = createMainWindow();
      }
    });
  });

  app.on('window-all-closed', () => {
    if (process.platform !== 'darwin') {
      app.quit();
    }
  });
}
```

```ts forge.config.ts
// Electron Forge configuration that flips security fuses at package time.
// Requires: @electron-forge/shared-types, @electron-forge/plugin-fuses, @electron/fuses.
// Add makers, code signing, and notarization for your release targets.
import type { ForgeConfig } from '@electron-forge/shared-types';
import { FusesPlugin } from '@electron-forge/plugin-fuses';
import { FuseV1Options, FuseVersion } from '@electron/fuses';

const config: ForgeConfig = {
  packagerConfig: {
    asar: true,
  },
  rebuildConfig: {},
  makers: [],
  plugins: [
    new FusesPlugin({
      version: FuseVersion.V1,
      [FuseV1Options.RunAsNode]: false,
      [FuseV1Options.EnableCookieEncryption]: true,
      [FuseV1Options.EnableNodeOptionsEnvironmentVariable]: false,
      [FuseV1Options.EnableNodeCliInspectArguments]: false,
      [FuseV1Options.EnableEmbeddedAsarIntegrityValidation]: true,
      [FuseV1Options.OnlyLoadAppFromAsar]: true,
    }),
  ],
};

export default config;
```

## 15. Workflow for agents

**Before you change code**

1. Read the relevant files in `src/main/`, `src/preload/`, and `src/shared/`. Check whether the capability already exists before adding a new IPC channel.
2. Identify which process each change belongs to. If a change crosses the renderer/main boundary, plan the shared types, validator, handler, preload method, and test together.

**While you work**

- Make the smallest change that solves the task. Do not refactor unrelated code or bump dependency majors as a side effect.
- Add or update the shared type, the runtime validator, the handler, the preload method, and a test in the same change.
- Prefer boring, documented Electron APIs over clever workarounds.

**Before you finish**

- [ ] Typecheck, lint, and unit tests pass using the scripts defined in `package.json`.
- [ ] No new `nodeIntegration: true`, `contextIsolation: false`, `sandbox: false`, or `webSecurity: false`.
- [ ] No new raw `ipcRenderer` exposure, no `sendSync`, no dynamic channel names from renderer input.
- [ ] Every new IPC handler validates the sender and the payload.
- [ ] Every new `shell.openExternal`, `loadURL`, `fs`, or `child_process` call is fed only validated input.
- [ ] No secrets, tokens, or credentials in source, logs, or fixtures.
- [ ] The change was checked in a **packaged** build if it touches preload, native modules, protocols, file paths, or asar.
- [ ] New dependencies are justified, maintained, and audited.

**Never do these**

- Disable a security control to get a feature working.
- Load remote URLs into a window with a privileged preload.
- Add `remote`, `<webview>`, or `BrowserView`.
- Ignore certificate errors or add `--no-sandbox` to production code.
- Commit `.env` files, certificates, or signing keys.
- Claim a fix works without running it. If you could not run the app or tests, say so.

## 16. Further reading

- Electron security checklist: https://www.electronjs.org/docs/latest/tutorial/security
- Context isolation: https://www.electronjs.org/docs/latest/tutorial/context-isolation
- Process sandboxing: https://www.electronjs.org/docs/latest/tutorial/sandbox
- IPC tutorial: https://www.electronjs.org/docs/latest/tutorial/ipc
- Fuses: https://www.electronjs.org/docs/latest/tutorial/fuses
- Performance: https://www.electronjs.org/docs/latest/tutorial/performance
- Code signing: https://www.electronjs.org/docs/latest/tutorial/code-signing
- Automated testing: https://www.electronjs.org/docs/latest/tutorial/automated-testing

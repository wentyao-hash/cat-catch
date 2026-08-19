# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

猫抓 (Cat-Catch) is a Manifest V3 browser extension (Chrome/Edge/Firefox) that sniffs network requests and page media to find downloadable video/audio/stream resources, then lets the user preview, parse (m3u8/HLS, DASH/mpd), and download them. It's plain HTML/CSS/JS — no bundler, no framework, no build step required for development. There is no `package.json`; `js/` and `lib/` files are loaded directly as `<script>` tags / MV3 background scripts.

The UI and code comments are primarily Chinese; commit/PR discussion and code comments in this repo are written in Chinese. Match the existing style when editing nearby code rather than switching to English.

## Build & packaging commands

Packaging uses `just` (see `justfile`). There's no test suite or linter beyond basic existence/format checks.

```bash
just install      # npm install + npm install -g crx3 (only needed for crx signing)
just validate     # sanity-check manifest.json via node -e
just prepare      # copy source into build/ (catch-script, css, img, js, lib, _locales, *.js, *.html)
just build-zip    # prepare + zip -> cat-catch<version>.zip
just build-crx    # prepare + sign with private-key.pem -> cat-catch<version>.crx (generates key if absent)
just build        # build-crx + build-zip
just quick        # build-zip only
just lint         # checks manifest.json parses and that core files exist (not a real linter)
just clean        # remove build/, dist/, web-ext-artifacts/, *.crx, *.zip, private-key.pem
just version      # print version from manifest.json
just status       # print version + icon/build-artifact presence
```

There is no automated test runner. To verify a change, load the unpacked extension directly:

1. Open `chrome://extensions`, enable "Developer mode".
2. "Load unpacked" → select the repo root (uses `manifest.json`, MV3 service worker).
3. For Firefox, load `manifest.firefox.json` as `manifest.json` (or use `web-ext`) — Firefox uses the MV2-style `background.scripts` array instead of a service worker.
4. Reload the extension after editing `js/background.js`, `catch-script/*.js`, or manifest files; content scripts/HTML pages usually just need a page refresh.

Bump `manifest.json` **and** `manifest.firefox.json` versions together, and add a changelog entry to `CHANGELOG.md` (format: `### x.y.z` with `[Added]/[Fixed]/[Updated]/[i18n]` bullet lines, newest on top) for any user-visible change.

## Architecture

### Two manifests, one source tree

- `manifest.json` — Chrome/Edge (MV3 service worker: `js/background.js`, which itself `importScripts()`s `polyfill.js`, `function.js`, `templates.js`, `init.js` when `G` isn't already defined).
- `manifest.firefox.json` — Firefox (MV3 syntax but `background.scripts` array, since Firefox doesn't support the same service-worker loading). Loads the same four support files plus `background.js` as separate `<script>`-like entries.

Keep both manifests' `permissions`, `commands`, and `version` in sync when changing capabilities.

### Global state (`G`)

`js/init.js` defines a single global object `G` that holds all runtime state and default settings (`G.OptionLists` — extension/MIME/regex filter rules, download options, UI options, etc.), plus `G.scriptList` — the registry of injectable "catch scripts" (see below). Nearly every other script assumes `G` already exists and mutates it directly; there's no module system or dependency injection. `js/function.js` holds shared pure-ish helper functions (formatting, byte/time conversion, filename building, etc.) and `js/templates.js` implements a small template/pipe mini-language (`Template` class) used to build copy/export strings (curl commands, IDM/download-manager links, etc.) from resource metadata.

### Background service worker (`js/background.js`)

Central hub. Responsibilities:

- Listens on `chrome.webRequest.onSendHeaders` / `onResponseStarted` to detect media requests by extension, MIME type, and user-defined regex rules (`G.OptionLists.Ext/Type/Regex`), matching against `cacheData` keyed by `tabId`.
- Persists captured resources into `chrome.storage.session` (fallback `chrome.storage.local`) as `MediaData`, with periodic `chrome.alarms` cleanup (`nowClear`/`clear`/`save`).
- Handles a large `chrome.runtime.onMessage` dispatch table (`Message.Message == "..."`) used by every popup/options/preview page to talk back to the background: `script` (toggle-inject a catch-script), `scriptI18n`, `HeartBeat` (keeps the MV3 worker alive via a long-lived port + `chrome.runtime.getPlatformInfo` polling, working around Chrome's 5-min service-worker kill), `clearData`, `clearRedundant`, `addMedia` (from content-script/catch-script), etc.
- Wires up `chrome.commands` (keyboard shortcuts) and `chrome.contextMenus`, matching the `commands` block in the manifests.

### Content scripts vs. "catch scripts" — the key architectural split

- `js/content-script.js` — declared in the manifest, runs on every page at `document_start` in the isolated world. Handles lightweight, always-on things: reading `<video>/<audio>.currentSrc`, playback speed/volume control, relaying messages to background.
- `catch-script/*.js` — **not** declared in the manifest. These are injected on-demand via `chrome.scripting.executeScript` from `js/background.js` (`Message.Message == "script"`), driven by the `G.scriptList` map in `js/init.js`. Each entry defines its injection world (`MAIN` vs `ISOLATED`), whether all frames get it, whether it needs the `catch-script/i18n.js` file injected alongside it (for `data-i18n`-style strings inside injected DOM), and whether toggling it requires a page reload:
  - `search.js` — "deep search": scans page JS context (including inside webworkers) for URLs/base64/hex-encoded media, JSON-LD, etc.
  - `catch.js` (`CatCatcher` class) — "cache capture": hooks `MediaSource`/proxies to catch blob-based/MSE video that never appears as a discrete network request.
  - `recorder.js` / `recorder2.js` — in-page video element recorder (canvas/MediaRecorder) vs. screen-capture-based recorder.
  - `webrtc.js` — captures WebRTC media streams.
  Toggling one of these from the popup adds/removes the tab from `script.tabId` (a `Set`) and injects or (if `refresh` is set) reloads the tab to remove it — there's no clean "uninject" API, so removal for `refresh:true` scripts is done via reload.

### Extension pages (each is an independent HTML document with its own JS entry point, but all share `polyfill.js` → `init.js` → `function.js` → `templates.js` as a common prelude)

| Page | Entry JS | Purpose |
|---|---|---|
| `popup.html` | `js/popup.js` (+ `popup-utils.js`, `media-control.js`) | Main UI: lists captured resources for the current/all tabs, filtering, download triggers, in-page video control panel. Also usable as a side panel (`side_panel` in manifest). |
| `options.html` | `js/options.js` | Settings UI for `G.OptionLists` (filter rules, block lists, custom copy templates, UI prefs), synced via `chrome.storage.sync`. |
| `downloader.html` | `js/downloader.js` (+ `m3u8.downloader.js`) | Generic multi-fragment downloader (`Downloader` class): threaded fetch with retry, optional streaming save via `lib/StreamSaver.js`, optional in-browser FFmpeg transcode. |
| `m3u8.html` | `js/m3u8.js` (+ `m3u8.downloader.js`, `lib/hls.min.js`, `lib/m3u8-decrypt.js`, `lib/mux.min.js`) | HLS/m3u8 parser & downloader: playlist parsing, key/decryption handling, slice download/merge. |
| `mpd.html` | `js/mpd.js` (+ `lib/mpd-parser.min.js`) | DASH/mpd manifest parser, can hand off derived m3u8 to `m3u8.html`. |
| `preview.html` | `js/preview.js` (+ `popup-utils.js`) | `FilePreview` class: batch preview/list UI for many captured files (drag/drop, filtering, HLS preview). |
| `json.html` | `js/json.js` (+ `lib/jquery.json-viewer.js`) | Pretty-prints captured JSON resources. |
| `install.html` | `js/install.js` | Post-install / onboarding page, bilingual (zh/en) toggle via CSS classes, no `chrome.i18n`. |

Cross-page communication is via URL query parameters (each page reads `new URL(location.href).searchParams` for things like `url`, `requestHeaders`, `tabid`, `title`, `filename`) plus `chrome.runtime.sendMessage`/`onMessage` back to the background worker — there's no shared app state beyond `chrome.storage` and the background's in-memory `cacheData`.

### i18n

Two parallel i18n systems:
- `_locales/<lang>/messages.json` — standard `chrome.i18n` messages, used in HTML via `__MSG_x__` placeholders (manifest strings) and in JS via `chrome.i18n.getMessage`. `js/polyfill.js` patches `chrome.i18n.getMessage` for browsers where it's missing (fetches `_locales/zh_CN/messages.json` as a fallback).
- `js/i18n.js` — a DOM post-processing pass run on every extension page: replaces `[data-i18n]`, `[data-i18n-outer]`, `<i18n>`, `[data-i18n-placeholder]`, and `document.title` using the `i18n()` helper from `function.js`.
- `catch-script/i18n.js` — separate i18n bundle injected into the page's `MAIN`/`ISOLATED` world alongside catch-scripts (since those run outside the extension page context and can't reach `chrome.i18n` directly).

`en/messages.json` is the source of truth. `tools/sync-locales.js` (`node tools/sync-locales.js`) reconciles every other locale's `messages.json` against `en`'s key set/order, reporting added/removed keys — run it after adding or renaming an `en` message key so other locales don't silently miss it.

### Third-party libraries (`lib/`)

All vendored, not npm-managed. `lib/third-party-libraries.md` documents each file's upstream source, license, and exact version/build instructions — update that file whenever a `lib/*.min.js` is upgraded, and keep the version there in sync with what's actually vendored.

---
name: mygo-plugins
description: MyGo's official plugins — fetch, WebSocket, sqlite and watch for web pages, and updater, terminal, glass, game, loading, sfsymbols, permissionflow, render, automation as all-Go packages — and writing a plugin. Use when a page needs CORS-free HTTP, a WebSocket with handshake headers, a local database without cgo or file watches; when native UI needs a terminal view, Liquid Glass, a game loop or gamepads, loading indicators, SF Symbols, the system's permissions, an update window, videos of the UI, or another process such as an agent drives the app; or a Go feature should reach pages as a plugin.
---

# Plugins

A plugin adds a feature in a Go package under `plugins/` the app registers
with `mygo.Use(…)` before `App.Run`; the four web plugins below also have
an npm half the frontend imports — the rest are all Go, for native UI and
Go code, with nothing to register or install. Each plugin's page
`$MYGO/docs/plugins/<name>.md`, where `MYGO=$(go list -m -f '{{.Dir}}'
github.com/egoist/mygo)`, lists its options and per-platform notes: read
the one you are about to register or `New` (version protocol at
`../mygo-usage`).

## Where the rest is

| Material | Reached for |
|---|---|
| `../mygo-web/SKILL.md` | page loading, trust, bindings, the generated client — what a web plugin's calls ride on |
| `../mygo-desktop/SKILL.md` | permissionflow's guide UX in the desktop services, `mygo.Updater`'s signing keys and release sources |
| `../mygo-components/SKILL.md` | widget choice; sfsymbols's SVG fallback in the icon row |
| `../mygo-motion/SKILL.md`, `../mygo-drawing/SKILL.md` | render's frames (`ui.Recorder`), glass's and shaders' frame cost |
| `../mygo-usage/SKILL.md` | the automation CLI in the verify loop, and the version protocol |

## Web plugins — Go half plus an `@mygo-plugins/*` package

| Need | Plugin | The rule that bites |
|---|---|---|
| HTTP from Go — no CORS, any header, body streams to the page, abort cancels in Go | `fetch` | Never assign it to `globalThis.fetch`: non-http(s) URLs (`mygo://`, `data:`, `blob:`) must fall through to the webview's; `mode`/`credentials`/`cache`/`integrity` are ignored; cookies are the Go `http.Client`'s `Jar`, never the webview's; `Allow` sees the first request only — refuse redirects with the client's `CheckRedirect`; a URL carrying credentials (`https://user:pass@host`) is rejected in the page, before Go sees it |
| A WebSocket whose connections Go makes | `websocket` | Any header rides the opening handshake (`Authorization`, `Cookie`); the page's CSP `connect-src` does not apply; the client's `Timeout` and `CheckRedirect` are ignored; a fast server queues in Go past 1 MiB the page has unread, and a message over `MaxMessageSize` (64 MiB) fails the connection |
| A local database, without cgo | `sqlite` — since v0.9.0 a native module: `MYGO_NATIVE_SQLITECORE` names a different library, `sqlite.LibraryPath` and `MYGO_SQLITE_LIBRARY` are gone | `mygo.Use(sqlite.Plugin)` only for a page that imports the npm package — Go and native UI call `sqlite.Open` directly; files live under `PathUserData`; absolute paths, `..`, symlinked databases and Windows device names are rejected; one statement per `execute`, transactions only through `db.transaction` (`BEGIN IMMEDIATE`, rolls back as a whole); `ATTACH`/`DETACH`/`VACUUM INTO` are rejected, extensions cannot load; a connection belongs to its page and closes when it navigates away |
| Page-side file watches | `watch` | Pages watch roots **by name**, never by path, so bare `watch.Plugin` lets them watch nothing — name roots in `Roots` (must exist when `mygo.Use` runs) or grant at run time through `Resolve`; a batch is the diff since the last — a hint to re-read, not a changelog; 50 ms debounce; a removed root ends the iteration with a `remove` of `.` |

Using one: `mygo.Use(fetch.New(fetch.Options{Client: …, Allow: …}), …)`
before `App.Run`, and `bun add @mygo-plugins/<name>` in the frontend;
`fetch` and `websocket` take an `http.Client` and an `Allow` function.
Calling the npm half in an app that skips the Go half rejects with an
error that says to call `mygo.Use`.

## All-Go plugins — native UI and Go code

| Need | Package | The rule that bites |
|---|---|---|
| A Sparkle-style update window | `plugins/updater` | Checks 10 s after launch, then daily; a version the user skipped stays skipped in background checks; choices live in `updater.json` under `PathUserData`; `updater.MenuItem()` goes after About; register `updater.Plugin` or `native.Plugin` (draws the window without a webview), never both — built on `mygo.Updater`, whose signing and sources `../mygo-desktop` covers |
| A terminal view | `plugins/terminal` | Ghostty's emulator is a native library the CLI puts into the app; the program starts when a view first shows it (or at 80×24 half a second after `New`); one view at a time; `term.Close()` when the window closes |
| Liquid Glass, scroll edges, backdrop blur | `plugins/glass` | Materials, not backgrounds: what shows through is what painted under the element in the same window; `glass.Group` merges at most 8 panes; Reduce Transparency draws opaque surfaces; on CPU rendering a change under a pane redraws what the blur reaches |
| A game loop, camera, gamepads | `plugins/game` | Fixed steps (`game.Rate`, 60/s) whatever the display; `Draw`'s `alpha` interpolates between the last two steps; pads read in one standard layout and `JustPressed` needs last frame's `State`; a backgrounded app on macOS reads no pad as pressed |
| Loading indicators | `plugins/loading` | 29 indicators, `loading.All()` lists them for a picker; zero `Options` is loading.dev's default; one out of view asks no frames; Reduce Motion freezes it — name it with `Label` |
| Apple's SF Symbols on macOS | `plugins/sfsymbols` | Drawn from the running system — `Symbol`, `Image`, `Names()` yield nothing on any other OS: keep an SVG `.Fallback` (see `../mygo-components`); painter sizes are 8–512 px |
| The system's permissions | `plugins/permissionflow` | `Request` needs the Info.plist usage string (`macos.infoPlist` in the config) — without it macOS ends the app (`ErrUsageDescription`); call `Request`/`Wait`/`Guide` from a goroutine, not the main thread; Screen Recording's answer arrives only after a restart (the desktop row of `../mygo-desktop` carries the guide UX) |
| Videos of native UI | `plugins/render` | A `render.Composition`'s view builds each frame from its number (`ui.Recorder` underneath) — encoded on the GPU, CPU where none runs (`../mygo-drawing`) |
| Another process drives the app — e2e tests, agents | `plugins/automation` | Opt-in: imported under a build tag and left out of release builds; run with `MYGO_AUTOMATION` naming a socket; its `mygo-automation` CLI snapshots windows and acts by short ref — `go install github.com/egoist/mygo/plugins/automation/cmd/mygo-automation`, and `mygo-automation skills` prints the version-matched guide |

## Writing a plugin

A `mygo.Plugin{Name, Service, Setup}` — the service is bound like those of
`mygo.Bind`, as `plugin:<name>`, outside the generated TypeScript client;
`Setup` runs when the app uses it. The JavaScript package calls it through
`call` of `mygo-runtime`, listed as a peer dependency (so it uses the
app's copy) and a development dependency (for its own builds), and only
the app's trusted pages may call, as with every bound method. `mygo.Use`
panics at startup on a taken name or a service method whose types cannot
cross to JavaScript, so mistakes show before the first window.

## Verify

Build and test as `../mygo-usage` says. A web plugin's npm half is checked
by the frontend typecheck against the regenerated client (`../mygo-web`);
the tester cannot answer the browser side — drive it by hand or through
the automation plugin. The all-Go plugins verify as their domain skills
say: the widgets, motion and drawing skills for view material, the desktop
skill for dialogs and permissions, `mygo-automation` for the e2e path.
Done when: every option passed exists in the copy `go list -m -f
'{{.Dir}}'` names, and the browser-side paths of a web plugin are named in
the handoff as unverified.

---
name: mygo-web
description: Web frontends of a MyGo app — how pages load (dev server, embedded dist, custom protocols), the trust model for who may call Go, bound services with channels and typed events, the generated TypeScript client and mygo-runtime, and the plugin calls pages make (the plugins themselves are mygo-plugins). Use when a window shows a web page, the frontend must call Go or receive events from it, or a page's calls, events, channels or generated client need work. Native UI windows are mygo-components and mygo-usage; desktop services are mygo-desktop.
---

# Web frontends

A window shows a web page in the system webview; the frontend calls Go
through **bound services**, streams values through **channels**, and
receives **typed events**; `mygo generate` writes the TypeScript client,
checked and documented on both sides. A window of native UI skips every
piece of this: its view calls Go directly.

Scope: pages and what serves them.

## Where the rest is

| Material | Reached for |
|---|---|
| `$MYGO/docs/bindings.md`, where `MYGO=$(go list -m -f '{{.Dir}}' github.com/egoist/mygo)` | bound services, the types table, errors, events, channels, the generated client, who may call, `Page.Eval` |
| `$MYGO/docs/frontend.md` | how pages load, custom protocols, trust and untrusted pages, `mygo-runtime` exports, custom title bars, links, downloads |
| `../mygo-plugins/SKILL.md` | the official plugins — the four web ones and the all-Go packages — and writing one |
| `$MYGO/docs/i18n.md` | ICU catalogs in Go and in pages |

## Resolve the version, then read the section

Protocol at `../mygo-usage`: read the section you are about to use in the
copy `go list -m -f '{{.Dir}}'` names, prose included. Done when every
option you pass exists in that copy.

## Need → API, and the rule that bites

| Need | API | The rule that bites |
|---|---|---|
| Pages call Go | `mygo.Bind(v)`, `mygo.BindAs("Name", v)` | Every call runs on a goroutine of its own — guard shared state with a mutex; `Bind` panics at startup on a taken name or a type JSON cannot encode |
| Stream values during one call | a `*mygo.Channel[T]` parameter | One channel serves one call; it closes when the method returns; `Send` backpressures past 1 MiB; the page going away cancels the method's context |
| Push changes to pages | `mygo.NewEvent[T]("name")` + `Emit`/`Broadcast`/`EmitPage` | Events emitted before the page is ready deliver once it is; `Emit`'s error ends the loop feeding a closed window; names are unique and `mygo:` is reserved |
| Run code in the page | `Page.Eval` / `mygo.EvalAs[T]` | Statements run as an async function body — `return` produces the value; a throw is an `*mygo.EvalError` |
| Attach to the caller | `mygo.CallerWindow(ctx)`, `mygo.CallerPage(ctx)` | A leading `context.Context` cancels when the page navigates away or its window closes — pass it to slow work |
| The page cancels a call | the client method's `.with({ signal })` — an `AbortSignal`, as `fetch` takes; untyped `callWith(method, args, options)` | Aborting cancels the method's context: take the leading `context.Context` and pass it to the work; a method that takes none runs to its end while the promise rejects with the signal's reason |
| Decide who may call | `PageOptions.TrustedOrigins`, `PageOptions.Untrusted` | Trust follows the page's URL as it navigates; frames never call; `"*"` hands Go to every site the page shows |
| Serve app content at a URL | `mygo.Protocol.Handle` | Handlers run on their own goroutines and stream; each scheme is its own origin with its own CORS; an iframe of another origin loads its subresources through the scheme too (WebView2 127+ on Windows); WebKitGTK plays no media from a custom scheme — serve it from a loopback HTTP server |
| A Windows build without the webview | the `mygo_nowebview` build tag | links no WebView2 — a page window becomes a build-time decision |
| Values across the boundary | Go `encoding/json/v2` rules | `[]byte` arrives base64, `time.Duration` in nanoseconds, ids past 2⁵³ carry `json:"id,string"`, named constants become unions, `MarshalJSON` becomes `unknown`; fields need `json` tags to keep their names |
| Report a Go error to the page | return the error | The promise rejects with a `CallError`; its message is the error's text and `err.method` names the call |
| Page without a build step | `window.mygo.call/on/channel` | Missing arguments arrive in Go as zero values |

## The generated client

`mygo generate` builds the app and runs it with `MYGO_GENERATE` set:
`main` runs up to `App.Run`. Work that must not happen then — migrating a
database, claiming a device — goes in `App.WhenReady`. Each method binds
to a typed call — `add: bind<[string], Note>("Notes.Add", (t) => [t])` —
whose `.with(options)` carries an `AbortSignal`; Go doc comments become
TSDoc; `mygo dev` and `mygo build` regenerate it, so it is rarely run by
hand.

## Plugins

The official plugins — four web ones (`fetch`, `websocket`, `sqlite`,
`watch`, each a Go half plus an `@mygo-plugins/*` package) and nine
all-Go packages (`updater`, `terminal`, `glass`, `game`, `loading`,
`sfsymbols`, `permissionflow`, `render`, `automation`) — and writing a
plugin of your own are `../mygo-plugins`. A plugin's calls ride on
everything this skill covers: trust decides who may call, `Allow` gates
what leaves, and the npm half typechecks against the generated client.

Writing one: a `mygo.Plugin{Name, Service, Setup}`, its service bound under
`plugin:<name>` outside the generated client, its JavaScript package calling
through `mygo-runtime` as a peer dependency.

## Verify

Build and test as `../mygo-usage` says. The generated client is checked by
compiling it: keep the frontend's typecheck in the loop. The tester cannot
answer for the browser side — a page's rendering, real navigation, custom
protocol behaviour, Linux media: drive those by hand or through the
automation plugin.

Done when: the frontend typecheck passes on the regenerated client, and
each of those browser-side paths is named in the handoff as unverified,
left for a human or the automation plugin to drive.

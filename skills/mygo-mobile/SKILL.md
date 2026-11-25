---
name: mygo-mobile
description: MyGo apps on phones and tablets — the same Go on iOS and Android, native UI drawn by Metal or OpenGL ES, pages in the system web view; developing on a simulator, emulator or device with mygo dev, signing and publishing .ipa/.apk/.aab to App Store Connect and Google Play, the system's tabs and pushed stacks, fingers and the keyboard, the ui/mobile widget kit, device APIs and the device plugins (maps, location, camera, biometrics, push, purchases), and native code in Swift and Kotlin. Use when the app targets iOS or Android, builds for a store, uses a phone idiom — bottom sheet, swipe row, wheels, pull to refresh — reads a device API or plugin, or writes platform code for a phone.
---

# MyGo on phones

The same app runs on iPhones, iPads, Android phones and tablets: the Go
code, its bound services and its events are the desktop's; windows of web
pages show in the system web view, windows of native UI draw with Metal
(iOS) or OpenGL ES (Android). Scope: what differs on phones. Widget choice
stays `../mygo-components` (the inventory reconciles the same); desktop
services stay `../mygo-desktop`, minus what phones lack, below.

## Where the rest is

| Material | Reached for |
|---|---|
| `$MYGO/docs/mobile/`, where `MYGO=$(go list -m -f '{{.Dir}}' github.com/egoist/mygo)` | these pages in full: `README.md` (devices, dev loop, behaviour), `building.md` (signing, publishing, links, icons), `interface.md` (tabs, fingers, text, kit), `plugins.md` (device APIs, plugins, native code) |
| `$MYGO/docs/native-modules.md` | Go interfaces implemented in Swift and Kotlin — the deep half of device code |
| `../mygo-plugins/SKILL.md` | the registration-frame rules a plugin's Go half follows |
| `../mygo-usage/SKILL.md` | build/test commands, the version protocol, the two clocks in tests |

## Build, run, publish

```sh
mygo doctor                    # what a phone target is missing, and how to get it
mygo devices                   # what -device takes; -json for scripts
mygo dev -platform ios         # the booted simulator, or -device <name|UDID>
mygo dev -platform android     # the device or emulator adb reaches
mygo build -platform ios       # dist/ios-arm64/<slug>-<version>.ipa, signed
mygo build -platform android   # dist/android/<slug>-<version>.apk and .aab
mygo publish -platform ios     # the .ipa to App Store Connect
mygo publish -platform android # the .aab to a Google Play track
```

| Step | The rule that bites |
|---|---|
| Tooling | iOS needs a Mac with Xcode — the only cgo MyGo ever needs; Android needs Android Studio's SDK and NDK on any system; `mygo doctor` warns, never fails, and a desktop build needs none of it |
| Dev loop | A Go change rebuilds and relaunches in a few seconds and **restarts the app** — `mygo.KeepState(&a)` brings unexported fields back across restarts and system kills; `App.OnSaveState` / `App.RestoredState` for more control; a change of the host, a plugin or the resources rebuilds the whole app |
| `devUrl` on a device | The simulator reaches the Mac's dev server; a device is off its network stack — the dev server must listen on the network (`vite --host`) and Android reaches it through `adb reverse`; without `devUrl` the app embeds `frontendDist` |
| Signing | The `.ipa` takes the installed provisioning profile that fits the identifier and `ios.distribution`; the CLI refuses a build naming Apple's private methods (App Review turns them down); Android release builds need `android.keystore` with `MYGO_ANDROID_KEYSTORE_PASSWORD` and `MYGO_ANDROID_KEY_PASSWORD` — else they sign with the debug key, which Google Play refuses |
| Publishing | App Store Connect needs `MYGO_ASC_KEY_ID`, `MYGO_ASC_ISSUER_ID`, `MYGO_ASC_KEY` (a `.p8` path, never on the command line); Google Play needs `MYGO_PLAY_SERVICE_ACCOUNT`, `-track internal/alpha/beta/production`; Play takes a first upload only from its own console, and an unreviewed app only with `-status draft` |
| Platform config | `ios`: `infoPlist` (permission strings, `LSUIElement`), `team`, `associatedDomains`, `launchScreen`, `backgroundModes`, `multipleWindows`; `android`: `permissions`, `keystore`, `adaptiveIcon` (keep the mark in the middle 72/108), `appLinks`, `splash`; deep links need the host's `apple-app-site-association` / `assetlinks.json` |

## How phones behave

- **One screen**: the window shown last fills the screen; `Window.Push` /
  `Pop` is the navigation. Sizes, positions and title bars have no say;
  `Window.Bounds` is the screen's.
- Native UI lays out inside the safe area (`Context.SafeArea`: bars,
  cutout, keyboard); pages lay out inside the bars and above the keyboard
  on Android.
- **Android's back** pops the pushed window, then goes back in a page,
  closes what Escape closes in native UI, goes back in a `ui.Router`, then
  backgrounds the app — and the system's predictive back follows the
  gesture.
- **Lifecycle**: the app never quits itself. Background is
  `Power.OnSuspend`, return is `App.OnActivate` and `"resume"`; frames
  stop in between. The system may end a backgrounded app:
  `App.OnSaveState` keeps what the next launch finds in
  `App.RestoredState`. `App.OnMemoryWarning` fires as memory runs short —
  native UI drops its textures and text caches and makes them next frame.
- **What phones lack does nothing**: menu bars, trays, the Dock, global
  shortcuts, the updater (stores update apps).
- Dialogs: `Dialog.Open` gives copies of chosen files in temporary files;
  `Dialog.Save` a temporary file to write; `Dialog.SaveCopy` asks where a
  file the app made goes. Secrets go to the Keychain (iOS) and to storage
  encrypted with a keystore key (Android). Notifications ask the first
  time.
- **Android loads no custom schemes**: the app's frontend and other
  schemes are served at `https://<scheme>.localhost`; a form posted to an
  app scheme arrives without its body — post with `fetch` or through bound
  methods.

## Interface

- **Tabs and stacks**: `mygo.NewTabs(TabItem{Title, Symbol|Icon, Window},
  …)` shows the system's tab bar; `list.Push(win)` pushes onto a tab's
  stack, back button / iOS edge swipe / Android back pop it. A tab names
  an SF Symbol for iOS and a PNG (template-drawn) for Android and iOS
  without the symbol; `tabs.SetBadge(i, "3")` counts. On desktops the same
  calls degrade: `Tabs.Select` shows a window, a push shows a window.
- **System stack**: `app.router.SystemStack = true` makes the router's
  history the system's navigation, `Route.Title` naming the bar; show no
  bar of your own while `router.InSystemStack(c)`. On desktops and in
  browsers, nothing changes.
- **Fingers**: a tap is a click, reaching what it nearly touches by
  14 DIPs; a moving finger locks to its axis and the innermost scroller of
  that axis, flings on lift, and stops on landing without tapping; past
  the ends content rubber-bands; half a second still is
  `Element.LongPressed` (the device ticks); two fingers pinch and turn
  (`InputEvent`'s `InputZoom`, `Rotation`); `list.PullToRefresh(&flag)`
  guards the pull. From the left edge on iOS, a finger follows a router
  page back.
- **Keyboard**: the input's client speaks to the system's suggestions and
  input methods; steer with
  `Keyboard(ui.KeyboardEmail).ReturnKey(ui.ReturnNext).Capitalize(…).Autocorrect(false).Autofill(ui.AutofillEmail)`.
  `ReturnNext` moves focus on.
- **The phone kit** `ui/mobile` — apps that don't import it don't pay for
  it; `mobile.SetLook` chooses a design whatever the system:

| Widget | It is |
|---|---|
| `Sheet`, `ActionSheet` | Bottom panels with detents a finger drags away |
| `Drawer` | A panel sliding from the side |
| `PageView`, `PageDots` | Pages a finger swipes between, built as they come near |
| `SwipeRow` | A row swiped aside to reveal actions |
| `Wheel`, `DatePicker`, `TimePicker` | Spinning wheels, a tick as the choice changes |
| `TopBar`, `BottomBar`, `FloatingButton` | Bars the app draws, for one window that routes within it |
| `ListRow`, `Section` | Rows and their groups, with a press's look |
| `Ink` | Material's ripple |
| `Zoomable` | Content fingers pinch, turn and drag, as a photo |

- On iPad a window can be a scene of its own beside the app's others:
  `"ios": {"multipleWindows": true}` in the config and `OwnScene: true` on
  the window.
- Pictures from the network take `netimage.Image` (`../mygo-components`),
  which caches on disk and shows offline.

## Device APIs and plugins

The device APIs are package-level, as on desktops; they do nothing on
desktops, or report `errors.ErrUnsupported`:

| | |
|---|---|
| `mygo.Haptics` | A selection's, an impact's or a task's end, felt |
| `mygo.SystemBars` | The bars' icon color; hiding them |
| `mygo.SetOrientations` | How the screen may turn |
| `mygo.Share` | The system share sheet of text, links and files |
| `mygo.Network` / `mygo.Power.Battery` | Online and metered; the battery's level and low power mode |
| `mygo.SetBarItems` | The window's system navigation bar's buttons |
| `Context.SafeArea` | What covers the window's edges — draw in the `Overlay` |

What needs permissions or a large dependency is a plugin, which the app
has only if it imports it, and which adds its usage descriptions and
permissions to the app:

| Package (`plugins/…`) | What it does |
|---|---|
| `maps` | A map in native UI with pins — MapKit; osmdroid with OpenStreetMap tiles, no key |
| `location` | Where the device is, and following it |
| `imagepicker` | The system's photo/video picker, no permission needed, and its camera screen |
| `camera` | A live native-UI preview, and photos |
| `videoplayer` | Video in native UI |
| `biometrics` | Face ID, Touch ID, fingerprints, the passcode |
| `sensors` | Accelerometer, gyroscope, magnetometer |
| `push` | APNs and Firebase tokens, messages and taps |
| `purchases` | In-app purchases and subscriptions |
| `permissionflow` | Whether the app may use the camera, microphone, photos…, asking, and the system's settings |
| `preferences` | Small values kept between runs, every platform |
| `sqlite` | SQLite with full-text search, compiled into the app |
| `background` | Go code run while the app is closed, as the system finds time |

## Native code

Device code is best a **native module**: a Go interface, implemented in
Swift (iOS) or Kotlin (Android), that Go calls as any Go value —
`mygo native new -lang swift,kotlin torch`, the interface marked
`//mygo:native`, its Info.plist keys, entitlements and Gradle
dependencies joining the app's. A plugin can also bring Swift and Kotlin
that Go calls with JSON (`mygo-plugin.json` names the sources and the
class); `PluginHost.registerView` registers a system view that
`mygo.NewPluginView` places in a window of native UI, under the content,
as `plugins/videoplayer` does. Both are `$MYGO/docs/mobile/plugins.md`
and `$MYGO/docs/native-modules.md`'s to detail; desktop native modules
are `../mygo-desktop`.

## Testing

Unit tests run views in a `ui.Tester` as on the desktop, with
`FingerDown`/`FingerMove`/`FingerUp` and `SetInsets`; `golden.Match`
compares pixels — generate goldens **on the platform that runs the
tests** (`MYGO_UPDATE_GOLDEN=1`), text differs from one to another.
Frame stats reach a simulator through `SIMCTL_CHILD_MYGO_FRAME_STATS=1`
and a device through `adb shell am start … --es mygo.env
MYGO_FRAME_STATS=1`. End-to-end tests drive the running app from another
process, simulators and devices alike, through the automation plugin
(`../mygo-plugins`); a four-finger tap opens the inspector in a
development build.

## Verify

Build and test as `../mygo-usage` says. The tester proves views, fingers
and text; it does not prove store signing, a real device's permissions
prompt, or the stores' review rules — report those as unverified for a
human with the credentials. Done when: `mygo build` for each target
succeeds, the tests pass on the platform they target, and every
phone-lacked service the change touched was caught by the behaviour list
above, not by a runtime nil.

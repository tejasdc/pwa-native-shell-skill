---
name: pwa-native-shell
description: Use when wrapping a Progressive Web App in a native shell to unlock platform features the browser can't provide — WKWebView / WebView, widgets, share extensions, watch complications, Siri / App Intents, HealthKit, Bluetooth, native push. Triggers on "wrap this PWA in an iOS app", "build a widget for this web app", "add a share extension", "make it a real native app", "publish this to TestFlight / App Store as a wrapper", "universal links open the app but don't route", "the WebView is loading every foreground", "Send to Inbox does nothing on the phone", "camera / mic doesn't work in the shell", "PWA header overlaps status bar in the wrapper", "external links are dead in the app", "the wrapped app looks different from the PWA in Safari", "the app has a scroll bar", "widget can't read the user's data", "share sheet doesn't deliver the capture". Encodes: the invariants that hold for any PWA-in-shell (state ownership, live-sync self-healing, in-page inbound delivery, App Group extensions, main-frame origin-checked bridge, delegate coverage for every web-platform surface, off-domain routing, shell attribution, debug affordances as build inputs, lifecycle hooks that forward not act, separate installation, source-of-truth in PWA `src/`). Includes the parameter checklist a new PWA's PRINCIPLES.md fills in before shell design, and iOS-specific references for WKWebView / WKUIDelegate / App-Bound Domains / Universal Links / App Group outbox. Do NOT use for standalone native apps with no web layer (use platform skills) or for a PWA that will not be wrapped (use `pwa-that-doesnt-suck` alone).
---

# PWA in a native shell

The default LLM output for "wrap this PWA in an iOS app" is a WKWebView that reloads on every foreground (aborting the sync channel and discarding drafts), silently drops Universal Links, has no delegate for `alert`/`confirm`/`prompt` so half the product's actions silently no-op, tries to open external links against `WKAppBoundDomains = true` and hits App-bound domain failure, ships an unaudited `log` bridge that writes arbitrary page text to `os.Logger`, and has `isInspectable = true` in Release. Tejas caught the reload bug on the third TestFlight build; a full audit surfaced eleven more class-mate mistakes. This skill is what that audit taught.

## What this skill is not

This skill covers **the shell** (the native app hosting a PWA). Load `pwa-that-doesnt-suck` alongside it for **the PWA itself**; that skill's Invariant 6 ("installed PWA has its own cookies / subscription / permissions") and `references/service-worker-lifecycle.md` Rule A ("the installed PWA updates itself") are load-bearing *inputs* to the shell — the shell respects them and never restates them.

For iOS distribution mechanics (TestFlight, signing, App Store submission), load `ios-app-store-launch` alongside; its `references/testflight-internal-and-shells.md` covers the Xcode 26 rsync gotcha and the visual-verify screenshot gate.

## The invariants — every PWA-in-shell obeys these

Each rule: what it forbids, the failure it prevents, how to check it.

**I1. The shell hosts a running program, not a page.** Exactly two document loads are legitimate: the first, and recovery after the content process is terminated. Never `reload()`, `load(url)`, `go(back/forward)` across documents, or re-instantiate the WebView to refresh, to deliver a URL, or to apply an update. **Prevents:** aborted sync channels, discarded drafts and in-memory caches, re-hydration on every foreground. **Check:** grep the shell for `load(`, `reload(`, `WKWebView(` — each hit names its true-recovery condition or is removed.

**I2. The PWA is the sole owner of its data and its code lifecycle.** The shell never reads, writes, clears or duplicates cookies, IndexedDB, Cache Storage or service-worker registrations; it never uses a non-persistent data store; it never decides when new code applies — the PWA's service-worker update flow does (`pwa-that-doesnt-suck` Rule A). **Prevents:** two owners of one fact; updates that fire while a recording or an unsaved draft is live. **Check:** no `WKWebsiteDataStore` API calls except `.default()`; no update logic in the shell.

**I3. Every inbound event is delivered into the running page as an event the PWA acknowledges.** Universal links, share payloads, widget taps, App Intents, push taps: all arrive through one bridge channel (`window.dispatchEvent` or a `native.*.list/ack` pair), never as a new document load. The only exception is a cold start before the first load, where the inbound URL *is* the first load. **Prevents:** dropped deep links, cross-document back-swipes, duplicate deliveries. **Check:** each entry point in `Info.plist`/entitlements has a handler and that handler ends in `evaluateJavaScript`, not `load`.

**I4. Extensions and widgets are peers through the App Group, never subviews and never credential holders.** They cannot see the WebView's cookie jar and must not get their own backend credential by default. A share extension writes an immutable, idempotently-keyed snapshot to an outbox; a widget reads a projection with an explicit `asOf`. The PWA inside the shell is the only process that talks to the backend as the user, and it delivers through the app's existing capture path with that path's retry and receipt semantics. Minting a device credential for an extension is a separate auth design, reviewed as High-risk. **Prevents:** a second delivery path with its own failure modes; token rotation surfaces; duplicate captures. **Check:** extension targets contain no network code and no Keychain sharing unless a written credential design exists.

**I5. The bridge exposes capabilities, not policy, and trusts only the app's own main frame.** Inject `forMainFrameOnly: true`; verify `frameInfo.isMainFrame` and `securityOrigin.host` on every message; enumerate methods; allow-list arguments (URL schemes, haptic styles); never accept free text destined for logs or storage; never expose a method that reads data the PWA owns. Each method carries a one-line reason the web platform cannot do it. **Prevents:** iframe or compromised-bundle escalation to app-switches and device features; reply-id collisions across frames. **Check:** the method table in the shell's `PRINCIPLES.md` has a "why native" column with no blanks.

**I6. Every web-platform surface the PWA uses has a native owner in the shell.** `alert`/`confirm`/`prompt`, media-capture permission, `window.open`/`target="_blank"`, off-domain navigation, file inputs with camera, downloads/`blob:` URLs, fullscreen, provisional-load failure. An unimplemented `WKUIDelegate`/`WKNavigationDelegate` method is a silent no-op in WKWebView, not an error. **Prevents:** actions that exit without UI, dead links, crashes on first mic use. **Check:** the capability inventory (P6) maps each API to a delegate method or a written "not used".

**I7. `Info.plist` and entitlements are derived from the PWA's capability inventory, and every claim is handled.** Usage strings for each privacy-protected API the PWA calls; `WKAppBoundDomains` from the origins the client actually contacts; associated domains only with the server file and the app-side handler both present. **Prevents:** crashes on privacy access; entitlements that route events to a handler that does not exist. **Check:** diff the plist against P1/P6 answers before each shell build.

**I8. Off-domain flows are enumerated and each is given a home.** App-bound WebViews *fail* navigation outside the list (WebKit: *"any attempt to navigate away from an app-bound domain will fail"*). Third-party links and sign-ins open in `SFSafariViewController` over the live workspace; `mailto:`/`tel:`/other schemes go to the system handler; anything that must return into the PWA (an OAuth redirect) returns to the PWA's own origin or via a universal link. **Prevents:** dead links in user content; unrecoverable provider sign-in from the phone. **Check:** P7 lists every external link class with its handler.

**I9. The shell is visible to the PWA and to the backend.** Tag the user agent with the shell build; expose `info()`; forward lifecycle facts the web platform does not deliver as events. Bug reports and server logs from the shell must be distinguishable from the browser and the home-screen install. **Prevents:** diagnosing shell bugs from reports labeled "browser". **Check:** one bug report or server log line from the shell carries the tag or its parsed derivative (e.g. a `shell` field in the report's environment section); if only the UA is tagged and nothing consumes it, the check has not passed.

**I10. Debug and distribution affordances are explicit build inputs.** `isInspectable`, allowed-arbitrary-loads, verbose logging: each is read from a build setting or plist key the ship script sets, never an unconditional constant. **Prevents:** an App Store build inheriting Web Inspector access to the session cookie. **Check:** grep for `isInspectable = true` literal.

**I11. Lifecycle hooks forward facts; they never act.** On foreground/background the shell does nothing to the page. `visibilitychange` fires inside the WebView already; if the PWA is stale after a long background, fix the PWA's reconnect logic, not the shell. **Prevents:** the "compensating reload" class. **Check:** no `didBecomeActive`/`scenePhase` observer touches the WebView.

**I12. The shell is a separate installation of the PWA.** Its cookies, service worker, IndexedDB, permissions and push state are distinct from the browser and the home-screen icon (`pwa-that-doesnt-suck` Invariant 6). Sign-in happens again; nothing migrates; storage is best-effort under the OS's eviction rules. Design and document it as such. **Prevents:** "I was logged in on the browser" reports treated as bugs; assumptions that drafts follow the user across installs. **Check:** the runbook's connect-a-device procedure names the shell as its own environment.

**I13. Truth lives in the PWA source, not in the shell's design doc.** Every assumption the shell makes about the PWA — transport, storage, dialogs, links, routing, capture — is verified by a cited PWA line at design time and again at review. The design doc records assumptions; `src/` records facts. **Prevents:** the whole class of bug this skill's incident-source audit is about. **Check:** each row in the shell's principles table has a `file:line`.

## The parameter checklist — every new PWA answers these before its shell is designed

Write these answers into `apps/<platform>/PRINCIPLES.md` (or equivalent) in the PWA's repo. This is the gate before any shell code.

**P1. Identity and domains.** Canonical origin; bundle ID / team / ASC record; every host the client contacts (API, media, auth, sync) → the `WKAppBoundDomains` list (max 10; subdomain coverage is best-avoided).

**P2. Sync model.** Transport (WebSocket / SSE / long-poll / periodic); what the server does on (re)connect; whether the client pulls on `visibilitychange`; what is lost when the content process dies.

**P3. Persistence.** Named browser stores; what is authoritative locally vs. server; the draft / unsaved-state model; eviction tolerance.

**P4. Update model.** How new code is offered and applied; the gates (during-recording refuse, unsaved-draft prompt, etc.).

**P5. Auth.** Mechanism; cookie attributes; rpID; any redirect-based flow returning to the origin; third-party sign-ins needed from the phone; session lifetime; the separate-installation consequence.

**P6. Browser API inventory** — grep the PWA source: microphone / camera (`getUserMedia`), file inputs (`capture`, `accept="image/*"` → camera option), geolocation, notifications / Web Push (WKWebView has no Web Push — native push is a separate design), `navigator.share`, clipboard, WebAuthn, `alert`/`confirm`/`prompt`, `window.open` / `target="_blank"`, downloads / `blob:`, fullscreen, wake lock.

**P7. Off-domain flows.** Every external link class in user content and product UI; which need to return to the PWA.

**P8. Routing and deep links.** Are routes in the URL; the inbound-navigation contract the PWA accepts; cold-start restore.

**P9. Diagnostics.** How the PWA attributes host / display mode; what it must never log; the client telemetry channel to reuse; where shell-side logs may go.

**P10. Inbound capture.** Is there a durable capture ingress; who owns delivery, retry, idempotency; what a share extension accepts (types, count, size); latency requirement — must a share land before the app is next opened? (Decides outbox-vs-credential; I4.)

**P11. Widget data.** Which facts, from which single state source; snapshot format and `asOf`; tap targets; whether private text may sit in the App Group; quick actions.

**P12. Other native features planned** (watch, Siri / App Intents, HealthKit, native push): which need their own data or credential; entitlements; rebuild triggers.

**P13. Distribution and security profile.** TestFlight internal vs. App Store; inspectable policy; encryption declaration facts (what crypto the PWA performs).

**P14. Visual contract.** Theme(s), safe-area handling, status bar, viewport-fit, keyboard geometry. Owned by the app's visual verification gate (screenshot-based), not by this skill.

## The design review — before any shell code

Run this checklist for every shell change (initial build, new capability, refactor). Cite the invariant each answer defends.

1. **State test (I1).** Does the change call `load`, `reload`, `go`, create a WebView, or touch `WKWebsiteDataStore`? Name the true-recovery condition or drop it.
2. **Ownership test (I2, I11).** Which PWA owner already does this (sync, updates, capture, diagnostics)? A shell feature that duplicates an owner is a bug.
3. **Capability inventory (I6, I7, P6).** Grep the PWA for the browser API touched; list every call site; map each to a delegate method, a plist key, or a written "not used".
4. **Delegate coverage (I6).** For each new `WKWebViewConfiguration` flag, list the delegate methods that become load-bearing and confirm they exist.
5. **Bridge surface (I5).** Main-frame-only? origin-checked? arguments allow-listed? reply bounded? Could the PWA do it without native code?
6. **Inbound path (I3, I4).** For every new entry point: does the payload land in-page or in an outbox, who acks it, what is the idempotency key?
7. **Off-domain (I8, P7).** Which external link classes does the change touch and where do they open?
8. **Privacy (I5, P9).** Does anything cross into `os.Logger`, App Group files, or the share sheet that the PWA's diagnostics allowlist forbids?
9. **Attribution (I9).** Does the UA / `info()` change so reports from the new build are distinguishable?
10. **Build inputs (I10).** Any new debug flag is a build setting, not a literal.
11. **Source citation (I13).** Every PWA assumption in the change has a `file:line` in the commit message.
12. **Device evidence.** Name the one on-device check the operator runs and the log line or screen that proves it.

## iOS specifics — the mechanical fixes

**WebKit delegate coverage.** WKWebView has no built-in UI for `alert`, `confirm`, `prompt`, media permission, `target="_blank"`, downloads, or provisional-load failure. Each unimplemented delegate method is a silent no-op — the PWA's action exits without UI. `WKUIDelegate` owns the dialog/new-window methods; `WKNavigationDelegate` owns the navigation-failure and download policies. Implement all seven for any PWA that uses any of them:
- `webView(_:runJavaScriptAlertPanelWithMessage:...)` (WKUIDelegate) → `UIAlertController` with OK.
- `webView(_:runJavaScriptConfirmPanelWithMessage:...)` (WKUIDelegate) → OK / Cancel returning Bool.
- `webView(_:runJavaScriptTextInputPanelWithPrompt:...)` (WKUIDelegate) → text field returning String?.
- `webView(_:requestMediaCapturePermissionFor:...)` (WKUIDelegate, iOS 15+) → grant iff `origin.host == webAppURL.host`, deny otherwise. Not implementing this falls back to WebKit's own per-origin prompt (the OS TCC prompt still appears once regardless — that's the "single prompt" the PWA cares about).
- `webView(_:createWebViewWith:for:windowFeatures:)` (WKUIDelegate) → off-domain http(s) → SFSafariViewController, return nil; same-origin http(s) → deliver in-page via the same route-delivery event the Universal-Link handler uses (**never** `webView.load(request)` — that unloads the running PWA; I1); `blob:` → route through the download path (SFSafariViewController does not accept `blob:` and `UIApplication.open(blob:)` is a silent no-op); other schemes on `.linkActivated` → `UIApplication.open`.
- `webView(_:didFailProvisionalNavigation:withError:)` (WKNavigationDelegate) → native retry only on first-cold-launch failure where `webView.url == nil`; later failures are the PWA's own error UI.
- Downloads (WKNavigationDelegate, iOS 14.5+): in `decidePolicyFor`, if `navigationAction.shouldPerformDownload` or the target is `blob:` on the main frame, return `.download`. Implement `webView(_:navigationAction:didBecomeDownload:)` + `WKDownloadDelegate` writing to a temp file, then present the file via `UIActivityViewController`. Without this, `<a download>` navigates the main frame to the raw payload — replacing the running PWA with JSON, an audio player, or an image.

**Every WebKit completion handler must be called on every path.** WKWebView's `CompletionHandlerCallChecker` raises `NSInternalInconsistencyException` if any of the delegate closures above is released uncalled — dropped when `rootViewController()` is nil, or when the presenter is mid-transition, or when an alert is rejected. Route every alert/dialog helper through a wrapper that calls the cancel completion on any presentation failure.

**App-Bound Domains and off-domain routing.** `WKWebViewConfiguration.limitsNavigationsToAppBoundDomains = true` restricts navigation to the entitled domain list. Any other host in the main frame *fails* (not opens externally). `WKAppBoundDomains` in `Info.plist` lists at most 10 hosts (subdomain coverage is best avoided). In `decidePolicyFor`: if the URL's host isn't the PWA origin and scheme is http(s), cancel the navigation and present `SFSafariViewController`. Non-http schemes (`mailto:`, `tel:`, custom app schemes) go to `UIApplication.shared.open` gated on `.linkActivated` (never scripted).

**Universal Links → in-page delivery.** `com.apple.developer.associated-domains` entitlement with `applinks:<domain>`; server `/.well-known/apple-app-site-association` with `applinks.details[].appIDs = ["<TEAM>.<BUNDLE>"]` and components.

SwiftUI's App lifecycle documents `.onOpenURL` as the Universal Link receiver; `.onContinueUserActivity(NSUserActivityTypeBrowsingWeb)` is the UIKit-era path. Both may fire for the same tap — wire both and dedupe in the delivery layer (drop a URL identical to the last one within ~1 second).

**Delivery mechanism — prefer a router-owned CustomEvent, not raw `pushState + popstate`.** Modern routers (TanStack Router, and any React Router setup that keys entries by `history.state.__TSR_key` / `__NEXT_ROUTER_STATE_TREE` / similar) *patch* `window.history.pushState`; a shell call to `pushState(null, '', path)` already navigates the router, and the synthetic `popstate` that follows corrupts the entry index — TanStack's blocker in particular can resolve to `history.go(0)` (a full reload) when the user chooses "stay." The correct shape:
- Shell: `evaluateJavaScript("window.dispatchEvent(new CustomEvent('<app>:navigate', {detail: {href}}))")`.
- PWA: register the listener at **module evaluation** (not inside the router-construction function — that only runs after your `main.tsx` awaits app open, and the shell's `didFinish` dispatch precedes that). Buffer the last href in a module-level variable; on the first `makeRouter` call, set a module-level `activeRouter` reference and replay the buffered href immediately via `router.navigate({href, replace: true})`. Live dispatches after mount call `activeRouter.navigate({href})` directly. Clear `activeRouter` in the RouterProvider component's unmount cleanup so a stale reference cannot receive a dispatch in test setups.
- Pass an `href` (not `{to, search}`) to `router.navigate` — the router's `buildLocation` runs the router's JSON-aware `parseSearch` on the search string (so `"3"` and `"true"` and objects arrive correctly typed) and preserves the URL fragment. Rebuilding search from `URLSearchParams` strings arrives quoted and drops the hash.
- Cold-start replay uses `replace: true` so the back stack does not contain the initial `/` the user never saw; live dispatches push normally.
- Raw `pushState + popstate` is acceptable only for a router that neither patches `pushState` nor uses `history.state` — check by opening the framework's history module.

On cold-start with a link, `makeUIView` usually runs **before** the SwiftUI receiver callback fires, so the "link becomes the initial `URLRequest`" branch is often not exercised — the delivery happens at `didFinish` instead. Design for both cases; keep both branches so a fast callback still works.

Use `URLComponents.percentEncodedPath` / `percentEncodedQuery` / `percentEncodedFragment` to construct the delivered path — `URL.path` decodes percent-encoded segments and returns `""` for a bare-origin link. Default the path to `/`.

**App Group outbox (Share Extension recommended v1).** Group container `group.<bundle>` shared between the app and the extension. Extension writes an immutable snapshot `<container>/Captures/<uuid>.json` + attachment files atomically. Main app on `scenePhase == .active` enumerates the outbox and dispatches `thnk:capture` events (or a `native.captures.list()` / `ack(id)` pair). The PWA feeds its existing capture path and acks *only* on server-accepted receipt; the shell deletes on ack. Never `load()` to deliver (I3). Snapshot's `uuid` is the idempotency key across retries. The alternative — extension posts directly with a minted device token — is a High-risk auth design (new credential surface, rotation, revocation) chosen only if delivery-before-next-app-open is a stated product requirement.

**`WKWebViewConfiguration` invariants for a passive PWA host:**
- `config.userContentController.addUserScript(...)` with `forMainFrameOnly: true` for the bridge (I5).
- `config.applicationNameForUserAgent = "<app>/<build>"` for shell attribution (I9). **Emitting the tag is half the fix.** Wire a consumer on the PWA side — a `shell` field in `diagnosticEnvironment()` extracted by `agent.match(/<app>\/(\d+)/)` — and use it to override `displayMode` from `browser` to `shell`. Without a consumer, the "one server log line shows the tag" check cannot pass. If server-side attribution matters, add a request serializer that logs the UA; the UA tag emission alone changes no log line by itself.
- `webView.isInspectable = AppConfig.allowWebInspector` reading a plist key **that the ship script actually passes**. A hardcoded `true` in `project.yml` is the same literal in a different file — it does nothing that changes between builds. Wire it as an xcodebuild build setting: `THNKAllowWebInspector: "$(THNK_ALLOW_WEB_INSPECTOR)"` in the plist, Debug default `YES` and Release default `NO` in the project's config table, and the ship script passes `THNK_ALLOW_WEB_INSPECTOR=YES` for TestFlight archives. When a plist value comes from a build-setting substitution, Info.plist preprocessing turns it into the string `"YES"` / `"NO"` — the reader must accept both `Bool` and those strings (I10).
- `webView.scrollView.bounces = false`, `showsVerticalScrollIndicator = false`, `contentInsetAdjustmentBehavior = .never` — the outer WKWebView is a passive host; the PWA owns scrolling.
- `webView.isOpaque = false` with `backgroundColor` set to the PWA's body background so no white flash appears during load.

**Verification gate.** No shell change reaches TestFlight without a simulator screenshot + assertion pass. Boot simulator, install the release build, launch against the production URL, wait for first paint, `xcrun simctl io booted screenshot`, then a Python image analysis asserts: no white band, status bar readable, PWA header not overlapping system chrome, bottom tab bar dark. Wire it into the ship script as a hard gate (`SKIP_VERIFY=1` only bypass).

## Setup for a new PWA-in-shell

1. Load `pwa-that-doesnt-suck` alongside this skill.
2. Write `apps/ios/PRINCIPLES.md` (or platform-equivalent) answering P1–P14 with `file:line` citations from the PWA source (I13).
3. Run the design-review checklist against the initial shell design.
4. Scaffold the shell — WKWebView + main-frame origin-checked bridge, delegate methods for every P6 API, off-domain SFSafariViewController, Universal Link → in-page delivery.
5. Wire the visual verify gate (screenshot-based) as a hard pre-upload check.
6. Ship one build. On-device check: the P6 dialog surfaces, one Universal Link, one off-domain link, one mic use.
7. Commit `PRINCIPLES.md` and the design-review record alongside the first shell commit.

## Anti-triggers

- A pure static native mobile app with no web layer → use platform skills (Swift, Kotlin).
- A PWA that will never be wrapped → use `pwa-that-doesnt-suck` alone.
- App Store submission mechanics (screenshots, privacy questionnaire, review) → load `ios-app-store-launch`.

## What this skill deliberately does NOT teach

- Web platform correctness inside the PWA. That's `pwa-that-doesnt-suck`.
- App Store submission mechanics. That's `ios-app-store-launch`.
- Visual design taste. Load a design skill alongside.
- Android WebView specifics. The invariants generalize; Android delegate names and entitlements differ. Contributions welcome.

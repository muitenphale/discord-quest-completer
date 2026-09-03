# Architecture

This document describes how Orion is structured internally. It is intended for contributors and the curious, not as a user guide. Last reviewed against `index.js` **v4.10.11**.

## High-level overview

Orion's core is a single-file IIFE (`index.js`) that runs inside the Discord desktop client's DevTools console. It discovers Discord's internal webpack stores at runtime, introspects the currently-available quests, and coordinates a handler per task type through a small task runner. There are no external dependencies and no build step.

It holds no persisted state. The only global it sets is the in-memory `window.orionLock` re-entry guard; the entire lifecycle (paste → completion → cleanup) lives in the IIFE closure. (The Vencord plugin port persists its settings via Vencord's DataStore, but the userscript itself persists nothing.)

One subsystem needs more than the renderer: the `ACHIEVEMENT_IN_ACTIVITY` bypass has to POST to `*.discordsays.com`, which Discord's renderer CSP blocks. Completing those quests on Discord Desktop therefore requires either the **localhost relay** (`tools/orion-relay/`) or the **Vencord plugin's native module**. The renderer alone cannot escape the CSP.

## File layout

```
OrionQuest/
├── index.js                       # single-file distributable, the actual userscript
├── eslint.config.mjs              # ESLint flat config scoped to index.js
├── README.md                      # end-user facing docs
├── CONTRIBUTING.md
│
│   # Vencord/Equicord userplugin port (functional, in sync). These sit at the
│   # repo root on purpose: UserpluginInstaller clones the repo straight into
│   # src/userplugins/, and Vencord only loads index.ts(x) + native.ts from the
│   # top level of a plugin folder. A subdirectory is never scanned.
├── index.tsx                      # plugin entry, /orion slash command, lifecycle
├── orion.ts                       # store loading, main cycle loop, dashboard registry
├── tasks.ts                       # per-type handlers incl. the OAuth bypass
├── native.ts                      # main-process IPC: CSP-exempt discordsays POSTs
├── traffic.ts                     # FIFO request queue with backoff
├── patcher.ts                     # RunningGameStore monkey-patch + RPC dispatch
├── settings.ts                    # Vencord settings schema
├── hooks.ts                       # settings-to-engine bridge (imports nothing, breaks the cycle)
├── types.ts  ├── util.ts
│
├── docs/
│   ├── ARCHITECTURE.md            # this file
│   └── VENCORD-PLUGIN.md          # userplugin install + usage guide
├── tools/
│   ├── orion-relay/               # localhost HTTP relay (no client mod needed for the bypass)
│   │   ├── orion-relay.ps1  ├── start-relay.cmd  └── README.md
│   └── orion-vencord-bundle/      # non-tech installer (INSTALL.cmd + README.txt + prebuilt dist)
└── .github/
    └── workflows/                 # CI: lint + syntax check
```

## Module map (inside `index.js`)

The file is a layered IIFE; each "module" is a `const` object or function in the outer closure.

| Module          | Responsibility                                                                                  |
| --------------- | ----------------------------------------------------------------------------------------------- |
| `CONFIG`        | User-tunable constants (name, version, colors, log limit, hide-activity)                         |
| `SYS`           | Frozen internal limits (max task time, retries, failure threshold, `IS_DESKTOP`)                |
| `RUNTIME`       | Mutable runtime state (running flag, cleanups, `autoEnroll`/`autoClaim`/`playSound`/`randomDelay`)|
| `Sound`         | Web Audio completion cue; reuses one `AudioContext`                                              |
| `Consent`       | Pre-authorization OAuth consent popup for the bypass (per-app, per-run, default decline)         |
| `esc()` / `notExpired()` | HTML-escape helper for server-controlled strings; NaN-safe quest-expiry check          |
| `ICONS`         | Inline SVG sprites used by the dashboard                                                         |
| `CONST`         | Frozen event names (`CONST.EVT`) and one blacklisted quest id (`CONST.ID`)                       |
| `ErrorHandler`  | Classifies HTTP errors into retryable / client / skippable                                       |
| `Traffic`       | Request queue with exponential backoff, rate-limit awareness, retry ceiling                      |
| `Mods`          | Reference to the Discord webpack stores discovered at boot                                       |
| `Patcher`       | Injects fake running-game records into `RunningGameStore`, incl. the visible/candidate getters; also flips `status.showCurrentGame` for hide-activity |
| `Logger`        | Quest picker UI, dashboard renderer, log ring-buffer                                             |
| `Tasks`         | Per-task-type handlers + `_bypassPost` transport picker + `bypassAchievement`                    |
| `loadModules()` | Dual-path module extraction (Vencord API + native fallback)                                      |
| `main()`        | Entry point. Discovers stores, renders picker, runs the task pipeline                           |

## Runtime sequence

```
paste into console → IIFE + orionLock guard → loadModules() (Mods = {...})
  → Logger.showQuestPicker() (checkboxes + filters, user clicks START)
  → main() task loop: JIT enroll → run handler per type → cleanup
```

## Webpack store discovery

Discord ships its stores in minified webpack bundles whose exported paths (`e.Z`, `e.A`, `e.ZP`, …) change every build, so hardcoded paths rot within days. Since v4.6, `loadModules()` uses a dual path:

1. **Vencord integration.** If `window.Vencord.Webpack` is present, request stores/props via Vencord's injected API. Bypasses recent Discord Stable runtime limitations entirely.
2. **Native fallback.** For vanilla Canary/PTB, push a fake chunk to `webpackChunkdiscord_app` to capture the module registry, guard against Sentry's secondary runtime by picking the `__webpack_require__` with the largest cache, then match stores by `constructor.displayName` (e.g. `"QuestStore"`), not by minified key.

Vanilla Discord **Stable** no longer exposes the live cache post-boot, so Stable users need Vencord (issue #20).

## Task types

| Type                      | Mechanism                                                                                 | Automatable        |
| ------------------------- | ----------------------------------------------------------------------------------------- | ------------------ |
| `PLAY_ON_DESKTOP`         | Inject a fake running game into `RunningGameStore`; Discord's heartbeat reports progress   | Yes                |
| `STREAM_ON_DESKTOP`       | Spoof `ApplicationStreamingStore.getStreamerActiveStreamMetadata`                          | No, see below      |
| `WATCH_VIDEO` / `_ON_MOBILE` | Poll the video-progress endpoint with natural float timestamps at `rnd(3500,4750)`ms   | Yes                |
| `ACTIVITY`                | Heartbeats against a voice-channel stream key                                              | Yes                |
| `ACHIEVEMENT_IN_ACTIVITY` | Heartbeat spoof first; on rejection, the OAuth → discordsays progress forgery (below)      | Yes, with consent  |

### `STREAM_ON_DESKTOP` does not complete, and the spoof is why

Discord decides which stream quests are progressing in `QuestProgressManager.getActivelyProgressingStreamOnDesktopQuests()`. Read off Stable 1.0.9255 and Canary 1.0.1148, which carry the same module:

```js
getActivelyProgressingStreamOnDesktopQuests() {
    let e = new Map, t = E.A.getCurrentUserActiveStream();
    if (null == t || 2 > h.Ay.countVoiceStatesForChannel(t.channelId)) return e;
    let n = E.A.getStreamerActiveStreamMetadata();
    if (null == n) return e;
    ...
}
```

Three conditions have to hold before the one Orion fakes is even read:

1. `ApplicationStreamingStore.getCurrentUserActiveStream()` returns a stream, so you have to actually be Going Live.
2. `SortedVoiceStateStore.countVoiceStatesForChannel(stream.channelId)` is at least 2, so somebody else has to be in the channel with you.
3. `getStreamerActiveStreamMetadata()` returns metadata. This is the only one the engine patches.

`initiateHeartbeat` then calls `getCurrentUserActiveStream()` a second time and calls `terminateHeartbeat` if it is null, so condition 1 is checked twice and the stream key is derived from the object it returns rather than from anything the metadata carries.

Measured on both branches with the engine's own spoof installed and nothing else: `getCurrentUserActiveStream()` stays `null`, so the set comes back empty and Discord never opens a heartbeat for the quest. The task therefore runs its 90 second no-heartbeat watchdog and aborts. Nothing about it is silent, but it also never completes.

Making it work means faking conditions 1 and 2 as well, and then finding out whether the server accepts a heartbeat carrying a stream key for a stream that was never created. That last part is unknown: the `ACTIVITY` path does get synthesized stream keys accepted, but for a voice channel that genuinely exists. Tracked in [#75](https://github.com/nyxxbit/discord-quest-completer/issues/75), and not implemented on a guess.

### Where the application id comes from

`PLAY_ON_DESKTOP`, `STREAM_ON_DESKTOP` and `ACHIEVEMENT_IN_ACTIVITY` all need the quest's application id: the first two impersonate that app as a running process, the third builds the `{appId}.discordsays.com` host from it. Discord has moved where it lives. It used to sit once per quest on `config.application.id`; under `taskConfigV2` it sits per task, at `config.taskConfigV2.tasks.<KEY>.applications[0].id`. `Tasks.appIdFor` reads the task-level path first and falls back to the legacy field, and every consumer goes through the id resolved onto the task rather than re-reading the config.

This matters because the failure is silent. Reading only the legacy field yields `0`/`null`, which produced a fake process Discord could never match to the quest (no heartbeat, quest frozen at 0%) and made the achievement bypass return early without attempting anything ([#43](https://github.com/nyxxbit/discord-quest-completer/issues/43)). Game/stream quests with no resolvable id are now skipped loudly instead of run, and a game/stream task that gets no dispatch within 90s of the last one aborts with a reason rather than idling until the 25-minute timeout.

### What the 90 second watchdog is counting

Discord beats every 60 seconds. `QuestProgressManager.calculateHeartbeatDurationMs` returns a flat `Millis.MINUTE` until under a minute of the target remains, measured at 60108ms on a live game quest, and the sender does not retry: a failed beat dispatches `QUESTS_SEND_HEARTBEAT_FAILURE` and the next attempt is the tick already scheduled 60s out.

So the watchdog has one beat of slack, and it used to be rearmed only by a success. One failed beat therefore ended the task at 90s, 30 seconds before Discord would have recovered it, and reported it as Discord going quiet when Discord had in fact answered with an error. It also capped `MAX_TASK_FAILURES` at one for these two task types &mdash; a second consecutive failure needs 120s and the watchdog fired first &mdash; so the branch that names Discord's own status code had never run for a game quest. That is the second half of [#74](https://github.com/nyxxbit/discord-quest-completer/issues/74).

A failure now rearms the watchdog as well, leaving consecutive failures as what gives up, and the watchdog line carries the failed beats so silence and errors stop reading the same. The exception is a failure before any credited beat: that keeps the original deadline, because "Discord is not accepting the injected process" is the #43 answer and is worth a beat and a half rather than five minutes of retries. The rule lives in `heartbeatWatchdog.ts` with tests; `index.js` carries the same logic inline, and gained the failure subscription it never had.

`ACHIEVEMENT_IN_ACTIVITY` is validated by the activity backend (`discordsays.com`), not the client, so there is no client heartbeat to forge, and Discord rejects those with 403. The bypass instead authorizes against that backend and reports progress to it directly. Quests for age-gated or delisted activities still can't be done: `/proxy-tickets` returns HTTP 403 code `50165` and the activity won't launch even manually, so they're skipped.

## ACHIEVEMENT bypass (OAuth → discordsays)

Runs only after the heartbeat path is rejected, and only with the user's consent (see below). Steps, in `Tasks.bypassAchievement` (`index.js`) / `tasks.ts`:

1. `POST /oauth2/authorize` for the quest's app (`authorize:true`, scope `identify applications.commands applications.entitlements`) → extract `code` from the returned `location` URL.
2. `POST /applications/{appId}/proxy-tickets` → a proxy ticket.
3. Build the `{appId}.discordsays.com` referrer carrying that ticket.
4. `POST {appId}.discordsays.com/.proxy/acf/authorize {code}` → a Discord Says token.
5. `POST {appId}.discordsays.com/.proxy/acf/quest/progress {progress: target}`.

Trust model: Discord delegates progress validation to the activity backend, so a forged progress POST from an authorized session is accepted. Steps 4-5 are the ones blocked by the renderer CSP and routed through a transport (below).

**Grant lifecycle.** Before step 1, Orion snapshots the app's existing OAuth grants (`GET /oauth2/tokens`). The snapshot is a **precondition**: if it fails, the bypass aborts before authorizing, so it never creates a grant it can't later identify. A `finally` block revokes only grants for that app that were absent from the snapshot, whether the flow succeeded or threw, so a failed bypass never leaves the app authorized and an authorization that existed before the run is never touched (a manual authorization made in parallel *during* the run is the one edge it can't distinguish).

**Consent.** The OAuth authorization is gated on explicit user consent. In the userscript, `Consent.ask()` shows a popup (app name, scopes, revoke note) before step 1 and defaults to decline. In the Vencord plugin, the `achievementBypass` setting (off by default) is the consent gate, which also covers the non-interactive `/orion start` and Auto-Start paths.

## Transport picker (`_bypassPost`)

Steps 4-5 hit `*.discordsays.com`, which the renderer CSP forbids. `Tasks._bypassPost` tries transports in priority order, first hit wins:

1. **Localhost relay** on `127.0.0.1:43210` (CSP allows `connect-src http://127.0.0.1:*`). No client mod.
2. **Vencord native module** via `VencordNative.pluginHelpers.OrionQuests` (main-process fetch).
3. **DiscordNative HTTP probes.** Best-effort, in case a future build exposes a generic HTTP method.
4. **Direct fetch.** Works on web Discord (no CSP), blocked on Desktop.

The Vencord port skips the picker and calls its native module directly. Every transport uses `redirect: "error"` so a 3xx from discordsays can't bounce the auth token / proxy-ticket referrer to another host.

## Localhost relay (`tools/orion-relay/`)

A ~170-line PowerShell HTTP listener that exists purely to escape the renderer CSP: the userscript can reach `127.0.0.1` but not `discordsays.com`, and the relay (outside the browser sandbox) can reach `discordsays.com`. Endpoints: `GET /health` (relay detection) and `POST /proxy` (forward to discordsays). Hardening: scheme pinned `https`, host pinned to `^[0-9]+\.discordsays\.com$`, path pinned to the two `.proxy/acf/*` endpoints, no redirect following, CORS reflected only to Discord origins, a strict request-header allowlist, an inbound `Host` check, and a body-size cap. Residual: while running, any local process can still drive forged progress through it, but only to those two endpoints, and a caller without a valid OAuth code/DS token gets rejected by discordsays anyway.

## Vencord native module (`native.ts`)

A main-process IPC bridge that performs the CSP-exempt discordsays POSTs for the plugin (and, when installed, for the userscript via `pluginHelpers`). This is a trust boundary. It runs privileged and CSP-free, so it validates every renderer-supplied value that shapes the request: `appId` and `questId` must be numeric and the `Referer` must be `https` pointing exactly at `{appId}.discordsays.com`. It also uses `redirect: "error"`.

## Traffic layer

`Traffic.enqueue(path, body)` carries the calls that report progress to the quest API: FIFO with jittered gaps, exponential backoff on `429` (Retry-After aware), retries on `5xx` up to `SYS.MAX_RETRIES`, and `4xx` propagated to callers. On shutdown, queued and deferred requests are rejected so awaiters never hang.

It is not the only egress point, and an earlier revision of this document was wrong to say it was. The queue issues `POST` only, so it covers exactly three endpoints:

| through the queue | direct |
| --- | --- |
| `POST /quests/{id}/enroll` | `GET /applications/public` (game metadata, and the same lookup inside the bypass) |
| `POST /quests/{id}/heartbeat` | `POST /quests/{id}/claim-reward` |
| `POST /quests/{id}/video-progress` | the whole OAuth chain: `GET`/`DELETE /oauth2/tokens`, `POST /oauth2/authorize`, `POST /applications/{id}/proxy-tickets` |
| | the raw `fetch()` calls to the local relay and to `*.discordsays.com` |

The pacing that matters for looking like a normal client is on the progress reports, which are the repeated, patterned calls. The direct ones fire once per task and are shaped like the actions a user takes by hand, so they are not queued. Anything new that reports progress on a loop belongs in the queue.

## Cleanup lifecycle

Every long-running subscription (Dispatcher events, safety timers, patched store methods) registers a `finish` callback in `RUNTIME.cleanups`. On **STOP** (or unload): `RUNTIME.running = false`; every cleanup runs (unsubscribe, restore patched methods, clear timers); the dashboard DOM is removed; `window.orionLock` is released. Cleanups are idempotent, so a double-stop is safe.

## Anti-detection posture

All already in code, not proposals:

- **JIT enrollment** (v4.4): quests enrolled one at a time right before execution. Both engines can turn it off (`autoEnroll`), which leaves quests you have not accepted untouched and pending until you accept them in Discord yourself.
- **Randomized intervals**: every polling/heartbeat loop uses `rnd(min,max)` ranges.
- **Realistic PIDs** for injected games (multiples of 4).
- **Natural video timestamps**: 6-decimal float seconds; cadence `rnd(3500,4750)`ms (2x faster than Discord's native 7-9.5s player loop).
- **Concurrency**: games at 1, videos at 2 (both exposed as Vencord settings).
- **Optional `randomDelay`** (off by default): a `rnd(60_000,1_800_000)`ms idle gap between cycles. The Vencord port currently uses a fixed `rnd(2500,4500)`ms inter-cycle wait instead.

## Security posture

- **DOM**: all server-controlled strings (quest/reward names, log text) are HTML-escaped via `esc()` before they touch `innerHTML`; the consent popup sets the app name via `textContent`. No inline event handlers anywhere (Discord's CSP rejects them).
- **OAuth**: explicit consent before authorizing; snapshot-precondition + `finally` revoke; never logs tokens, auth codes, or grant ids.
- **SSRF**: numeric-`appId` validation and `redirect: "error"` on every discordsays request; the native module additionally validates `questId` and `Referer`; the relay pins scheme/host/path.
- **Relay**: Discord-only CORS, header allowlist, Host check, body cap (see above).

## Compatibility

- **Discord Desktop** (Stable, PTB, Canary). Stable needs Vencord for store discovery (issue #20).
- **Browsers / script-injection mobile browsers**: web-compatible quests only; `GAME`/`STREAM` are filtered out via `SYS.IS_DESKTOP`.
- **ACHIEVEMENT bypass on Desktop**: needs the relay or the Vencord plugin (CSP can't be escaped from the renderer alone). On web Discord the direct fetch works.

## Vencord plugin (repo root: `index.tsx` + siblings)

A functional port, in sync with the userscript. It replaces manual webpack walking with `findStore`/`findByProps`, uses Vencord's settings instead of `CONFIG`, performs the CSP-exempt POSTs in `native.ts`, and exposes `/orion start|stop|status`. See `docs/VENCORD-PLUGIN.md`.

The optional enrollment watcher (`watchForEnrollments`, off by default) lives in `index.tsx`, not in the engine: `startOrion`'s teardown runs whenever the queue drains, so a watcher inside that lifecycle would stop watching the moment it succeeded. Its lifetime is the plugin's instead &mdash; armed by plugin load and `/orion start`, disarmed by `/orion stop` and by the plugin being disabled.

### Why the plugin sources live at the repo root

Vencord's build enumerates only the direct children of `src/userplugins` (`globPlugins` / `globNativesPlugin` in `scripts/build/build.mjs` + `scripts/build/common.mjs`): the plugin entry must be `index.ts`/`index.tsx` at the top level of the plugin folder, and native IPC must be `native.ts` (or `native/index.ts`) beside it. Nested subdirectories are never scanned. nin0's `UserpluginInstaller` runs a plain `git clone <repo>` into `src/userplugins`, so the clone root *is* the plugin folder, which means the plugin has to be the repo root for one-click install and in-app updates (`git fetch` + `git rebase origin/HEAD`) to work.

The userscript's `index.js` stays at the root next to `index.tsx`. Both resolvers prefer the `.tsx`:

- **Vencord/esbuild** resolves the directory import through `resolveExtensions`, where `.tsx` precedes `.js`.
- **UserpluginInstaller** scans for `index.ts`/`index.tsx`/`index.js`/`index.jsx` and keeps the last match in `readdir` order; libuv sorts entries with `strcmp` on POSIX and NTFS returns them collated, so `index.tsx` always comes after `index.js`.

`index.js` is therefore never mistaken for the plugin entry, and the userscript keeps its raw URL.

## Contributing

See `CONTRIBUTING.md` in the repo root.

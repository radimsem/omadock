# How omadock is built, and what its maintainers accept

Ticket: radimsem/omadock#2. Researched 2026-10-28 against `thepathless/omadock` at v4.8.0 (`main` and `experimental` are the same tree, `cdaac69`/`2ab0ed5`). File:line references are into that tree. PR and issue numbers are `thepathless/omadock` unless marked otherwise.

## Short answer

- **Shape.** `DockHost.qml` is the plugin entry (`manifest.json` `kinds: ["overlay"]`). It spawns one `Dock.qml` per monitor and owns the IPC target `omadock` and a shared `QtObject` store. `Dock.qml` is the composition root (2382 lines); behaviour lives in 17 stateless `components/logic/*Logic.qml` modules that take the dock root as first argument, plus pure, Qt-free root `.js` modules (`DockModel.js`, `DockTasks.js`, ...) tested under Node.
- **A Focus feature should be new modules, not growth of the roots.** `Dock.qml` is at its ratchet ceiling (2382/2382, zero headroom) and `DockModel.js` has 1 line spare (1325/1326). Accepted PRs add a pure `.js` module plus a `components/` or `components/logic/` file and touch the roots by a few lines (#67, #71, #72 all say so in their bodies).
- **IPC is small and mostly read-only or config-key setters.** The task HUD (`reportTask`/`clearTask`/`taskStatus`) is the one generic "push state from outside" channel. It is keyed by app id, capped at 64 entries, cleared on focus, and has no timers.
- **Notifications are observed, never held.** The dock reads the shell's notification service (or, as an overlay plugin without it, Omarchy's active popup JSON files via a Python inotify watcher). Nothing in the dock can intercept or delay a notification. "Jev holds notifications" is not something a dock plugin can do on today's pipeline; the dock can only change what it badges or bounces.
- **Acceptance bar.** Outside PRs that landed were 5 to 53 files, targeted `experimental`, came with Node unit tests (and sometimes a live suite), kept the gates green, were opt-in or feature-detected, and were event-driven. The maintainer then adds follow-up commits for integration and credits the author in the changelog.
- **Precedents.** A long-running, event-driven Python helper has precedent and was praised (`scripts/notification-popups.py`, `scripts/drive-removal-watch.py`). Network calls: none anywhere in the plugin; no precedent and no stated objection. Dependency on an external tool: several spawned/optional ones (`udevadm`, `canberra-gtk-play`, `omarchy plugin ...`, `qs ipc`), always feature-detected or failing quietly; nothing mentions herdr. Polling is the one thing the maintainer has explicitly said breaks the project's contract (#3).

## 1. Module layout and responsibilities

| Path | Responsibility | Evidence |
| :--- | :--- | :--- |
| `manifest.json` | Overlay plugin, `keepLoaded: true`, entry `DockHost.qml`, v4.8.0 | `manifest.json` |
| `DockHost.qml` (295 lines) | One `Dock` per monitor (`Variants`), `orderedDocks()` (focused monitor first), the `IpcHandler` for `omadock` when several docks run, `sharedStore` (`minimizedOrigins`, `parkedAt`, `parkSlots`, `taskStatus`, `appGrid`) | `DockHost.qml:126` (IpcHandler), `:78-88` (sharedStore) |
| `Dock.qml` (2382 lines) | Composition root: all state properties, `Process`/`Timer` objects, signal wiring, and one-line delegating wrappers (`function x() { return fooLogic.x(root) }`). A second `IpcHandler` there is enabled only with a single dock (`ipcEnabled`). | `Dock.qml:1904-1957`, comment at `:1902` |
| `components/logic/*.qml` (17) | Stateless function bags: `DockWindowLogic` (30 KB), `DockConfigLogic`, `DockContextLogic`, `DockNotifLogic`, `DockGroupsLogic`, `DockPinLogic`, `DockFolderLogic`, `DockEventLogic` (Hyprland raw events), `DockSlotLogic` (park/restore), ... Pure by rule: no Timer/Process/visuals; functions take `root` first. | `components/logic/`, `tests/static/structure-check.py:55-61` |
| `DockModel.js` (1325) | Pure model: id normalisation (`cliAppId`, `stripDesktop`), notification matching, badge-count persistence format, pinned/group parsing and every safety bound, `buildEntries` (pinned, running, grouped sections) | `DockModel.js:22`, `:78`, `:175`, `:344-465`, `:619`, `:1014` |
| Other root `.js` (all Qt-free, run under Node `vm` in tests) | `DockTasks.js` (task HUD), `DockLabels.js`, `DockLayout.js`, `DockPresets.js`, `DockFit.js`, `DockIcons.js`, `DockKeys.js`, `DockFloating.js`, `DockGridDrop.js`, `Plan.js`, `Buttons.js`, `SettingsSearch.js`, `DockMarks*.js`, `DockMotion.js`, `DockOutput.js` | CI syntax gate, `.github/workflows/ci.yml` |
| `components/*.qml` | Visual pieces: `DockCard`, `DockItem` (784 lines, renders the badge), `BadgeMark`, `DockAppGroupItem`, `AppGroupPopup`, `FolderPopup`, `DockContextMenu`, `SettingsPanel`, `FloatingDockWatch`, `AppGridHost`, `CappedFileView` (size-capped, watched file reads) | `components/` |
| `components/settings/*.qml` | One file per settings page, plus row family. New setting rows need an English key in `i18n/en.json` or CI fails. | `ci.yml` "Translation catalog" step |
| `scripts/` | Python/sh helpers spawned via `Process`: `notification-popups.py`, `terminal-hosts.py`, `drive-removal-watch.py`, `list-folder.py`, `list-drives.py`, `eject-drive.py`, `pick-folder.py`, `drop-check.py`, `omadock-task.sh`, `bind-keys.sh`, `notify.sh` | `scripts/` |
| Persistence | Config `~/.config/omarchy/omadock.json` (read through size-capped `CappedFileView`, written atomically); badge state `~/.local/state/omarchy/omadock-badges.json` | `Dock.qml:42-46` |

Per-monitor docks are opt-in (`multiMonitor`, default off); this came from outside PR #11 and was praised for exactly that ("mode opt-in and off by default").

## 2. IPC surface

Target `omadock`, called as `qs -p /usr/share/omarchy/shell ipc call omadock <fn> ...` (README "IPC & Keyboard Shortcuts"). The host handler is `DockHost.qml:126-295`.

| Function | Kind | Notes |
| :--- | :--- | :--- |
| `status()` | read | JSON: version, per-dock visibility, `notificationBadges`, `terminalHosts`, `terminalApps`, `model` (`DockHost.qml:129`) |
| `state()`, `itemGeometry()` | read | For live tests and bench (added in #33, praised as "no mutation surface") |
| `presets()` | read | JSON list with `active` and `builtin` |
| `minimizeActive`, `restoreLast`, `restoreAddress(address)` | window action | Park/restore bookkeeping |
| `toggleVisibility`, `reveal`, `hide` | visibility | All docks |
| `openSettings`, `openSettingsPage(page)`, `closeSettings` | UI | Focused monitor's dock |
| `setAlignment`, `setLayout`, `setPosition` | config-key setters | Persisted to `omadock.json`; other docks pick up the file change |
| `applyPreset(name)`, `savePreset(name)`, `deletePreset(id)` | preset mutation | Return `ok`/`not found`/id |
| `reportTask(appId, status, exitCode, title, duration)` -> `ok`/`invalid`, `clearTask(appId)`, `taskStatus()` | external push channel | `DockHost.qml:280-293` |

There is no event or subscription IPC out of the dock; everything is request/response. Two details that matter to any wrapper script: `qs ipc call` prints `Function not found.` on stdout and exits 0 for an unknown function, so `scripts/omadock-task.sh` probes `qs ipc show` first (`omadock-task.sh:27-42`); and with several docks the host handler owns the target while each dock's own handler is disabled (`ipcEnabled: false`, `DockHost.qml:107`).

## 3. Notification-badge pipeline

1. **Sources (two, mutually exclusive).**
   - The first-party service `omarchy.notifications`, resolved through `shell.serviceFor(...)` with a startup retry timer capped at 25 ticks x 200 ms so it "can't keep the event loop awake forever (zero-CPU invariant)" (`Dock.qml:1478-1492`, `DockNotifLogic.qml:16`). Its `popupModel` rows feed `onRowsInserted` etc. (`Dock.qml:1546-1575`).
   - Fallback for overlay plugins that never receive that service: `Process notificationPopupWatch` runs `scripts/notification-popups.py` only while `(showNotificationBadges || urgentOnNotification) && !notifService` (`Dock.qml:1509-1530`). The script reads Omarchy's active popup JSON files under `~/.local/state/omarchy/notifications`, with byte/row caps, `O_NOFOLLOW`, and a blocking inotify loop ("Blocking inotify reads consume no idle CPU"), docstring: "Read only Omarchy's active popup files, never history or a second D-Bus server" (`scripts/notification-popups.py:2`, `:66-110`).
2. **Counting.** `processNotifRowSticky` dedupes by `DockModel.notificationRowKey`, resolves target apps with `DockModel.notificationCounts`/`findNotificationTargets` (candidate-token matching, `DockModel.js:175-362`), skips the focused app, and bumps `notificationBadges` (`DockNotifLogic.qml:27-48`). Seen-keys are bounded to 512. Counts are sticky until the app is focused (`clearNotificationBadgesFor`) and persist across restart (`omadock-badges.json`, `DockModel.js:408-465`, README section 10). A 20 ms debounce timer coalesces rebuilds (`Dock.qml:1533-1538`).
3. **Urgency.** `handleNotificationReceived` bounces tiles (`urgentMap`) unless the app is focused, and plays a sound via `canberra-gtk-play` on the primary dock only unless DND (`DockNotifLogic.qml:60-125`).
4. **Rendering.** `DockItem.qml` computes `taskInfo` and calls `DockTasks.resolveBadgeStack(notificationCount, taskInfo, ..., mediaInfo, privacyInfo)` (`DockItem.qml:38`, `:55`). Priority: error 10 > privacy 9 > notification count 8 > running/progress 7 > media 6 > success 5. Primary and secondary badges render through `BadgeMark.qml`; folders sum member counts (`DockModel.groupBadgeTotal`, `:465`).

Constraint for Focus: the dock sits downstream of notification display. It cannot hold a notification, only choose whether to badge, bounce or sound for it. Holding would need Omarchy's notification service (or a separate notification daemon), which is outside this repository.

## 4. Sections and app groups

- **Model sections.** `DockModel.buildEntries(...)` returns `{ pinned, running, grouped }` (`DockModel.js:1014`, `:1190`); `Dock.qml` exposes them as `pinnedSection`, `runningSection`, `groupedSection` (`Dock.qml:446-464`). A divider separates pinned from running (`hasSeparator`, `:307`) and a second one precedes the folders/drives group (`hasFolderSeparator`, `:325`).
- **Layout "sections".** `splitSections` (config key, default false) splits the dock into panels, `sectionSpacing` (default 18) is the gap, and `DockLayout.placement(layout, alignment, splitSections, hasRightGroup)` yields `{ panel, split, align }`. "Spread" alignment puts folders and drives in a right group (`DockLayout.js:25`, `Dock.qml:312`, `:642`). Split sections came from PR #14.
- **App groups.** Stored in `omadock.json` as `{ id, name, icon, apps, cols, before }`, bounded by `boundAppGroups` (`DockModel.js:619-642`). Pinned apps and groups share one ordered run (`pinnedRow`, `:651`); `group.before` anchors a group to the pinned app it stands before. Operations (`openAppGroup`, `createAppGroupFromDrop`, `addAppToGroup`, `ungroupAppGroup`, `moveAppGroup`, ...) live in `components/logic/DockGroupsLogic.qml`, wrapped one-liner in `Dock.qml:1227-1249`. UI: `DockAppGroupItem.qml` (2x2 live preview), `AppGroupPopup.qml` (stays open, 2 to 6 columns), and `components/settings/SettingsGroups.qml`. Folders auto-dissolve at one app (README section 6).
- A Focus (an app set with its own countdown) would most naturally reuse the group shape (`id`, `apps`, ordering via `before`) and the `bound*` parsing discipline in `DockModel.js`, with its own pure module for the state machine. That is an inference from the code, not a maintainer statement.

## 5. How DockTasks and reportTask work

- **Entry.** `reportTask` (`DockHost.qml:280`) calls `DockTasks.normalizeTaskStatus(DockModel, appId, status, exitCode, title, duration)` and, on success, replaces `sharedStore.taskStatus` with `DockTasks.withTaskStatus(...)`. Returns `"invalid"` for an empty/unknown id.
- **Normalisation** (`DockTasks.js:19-62`). `appId` may be `primary|fallback` (e.g. `claude|foot`); both go through `DockModel.cliAppId`, the single alias table. Status words are folded to `running`, `success`, `error`, `privacy`, `progress` (`progress:NN` parsed), title clipped to 120 chars (`MAX_TASK_TITLE`), duration and exit code coerced to numbers.
- **Store.** One immutable-style map keyed by app id, max 64 entries (`MAX_TASK_STATUS_ENTRIES`, oldest dropped). Docks read it through `readonly property var taskMap` (`Dock.qml:125`); `sharedStore` is the single writable owner, so all monitors agree.
- **Lookup.** `taskStatusFor` does exact/alias match, then a second pass on `fallbackId` only if the primary app is not on the dock (`hasDockApp`). The match rule is deliberately stricter than the notification matcher (`isSameApp`: same id or reverse-DNS suffix after a dot) so `foot` does not badge `footclient` (maintainer fix on #67).
- **Display.** `resolveBadgeStack` ranks candidates; `TASK_GLYPH`/`TASK_COLOR` are the only place a status colour may be spelled (a unit test scans for stray colours; changelog v4.6.1). The hover chip text comes from `chipLabel` with `Tr.t` for translation.
- **Clearing.** On focus or click: `clearUrgentApp` also calls `clearTaskStatusFor` (`DockNotifLogic.qml:238-243`). `clearTask` over IPC does the same.
- **Producer.** `scripts/omadock-task.sh` wraps a command (reports on exit with exit code and duration) or calls `report|clear|status|progress|mic` directly. It walks `/proc/<pid>/stat` parents once at start to guess the host app, with env hints for `claude`, `aider`, `cursor` and others (`omadock-task.sh:44-130`). `OMADOCK_SHELL_PATH` overrides the shell directory for tests.
- Tests: `tests/unit/task-status.test.mjs` (21.6 KB) and `tests/live/tasks.sh` (drives `reportTask` -> `taskStatus()` -> `clearTask` and reads the drawn badge on a hidden second Quickshell instance; `tests/live/tasks-shell.qml`).

For Focus: this is the existing, accepted way for an external process (such as an agent Sessions/Inbox watcher) to put per-app state on a tile. Its limits are one entry per app id, a fixed status vocabulary and no event back out.

## 6. CI gates

Workflow `.github/workflows/ci.yml` (runs on every PR and push to `main`/`experimental`), plus `pr-base.yml` (a PR whose base is not `experimental` fails) and `pr-autotarget.yml` (a bot retargets `main` PRs to `experimental` and comments).

| Gate | What it enforces |
| :--- | :--- |
| `node --check` over every root `.js` | All 17 root JS modules parse (`DockTasks.js` was added to this list by the maintainer after #67, because the list had omitted it) |
| `node --test tests/unit/*.test.js *.test.mjs` | Pure-module unit tests (~25 files); `#72` reports 268 Node tests |
| `python3 -m py_compile scripts/*.py tests/unit/*.py` and `unittest discover tests/unit` | Python syntax and tests; 116 Python tests per #67 to #72 |
| `tests/static/i18n-en.py --check` | Every `Tr.t` / settings row has an `i18n/en.json` entry |
| `tests/static/security-grep.py` | **Fails** on: RichText/StyledText/MarkdownText; any `Text`/`Label` without `Text.PlainText`; `eval(`, `new Function(`, `new RegExp(` on a non-literal; `"sh","-c","<literal>" +` shell concatenation; inline `"python3","-c"`. **Lists, not fails**: `hyprctl eval` call sites and `notify-send`. Skips `tests/`, `docs/`, `.github/`. (`security-grep.py:2-27`) |
| `tests/static/structure-check.py` | Every `.qml/.js/.mjs/.sh` file is at most **800 lines**; `components/logic/*.qml` contain no Timer/Process/visual items and take `root` first; ratchet ceilings listed below |
| `qmllint --silent` (Arch container, Qt 6.11) | Parse errors only; unresolved imports are warnings. Release checklist runs typed lint on an Omarchy host |
| `shaders-in-sync.sh` | Committed `.qsb` equals compiled `.frag` |
| `tests/manifest-check.sh` | Mirror of `omarchy plugin validate` |
| Not in CI | `tests/smoke-test.sh` and `tests/live/*` (need a Quickshell/Hyprland session); the PR template asks for smoke, and maintainer comments cite live suites |

**The Dock.qml ratchet.** `tests/static/structure-check.py:31-33`: `ROOT_CEILINGS = {"Dock.qml": 2382, "DockModel.js": 1326}`. Measured: `Dock.qml` is 2382 lines (zero headroom), `DockModel.js` 1325 (one spare). The docstring says ceilings "may shrink over time, never grow", and the failure text repeats "may only be lowered, never raised". The history is softer than the text: `git log -L` on that line shows the maintainer raised it inside feature commits (2329 -> 2415 in `feat(restore)` on 2026-10-08, 2415 -> 2420 in `feat(folders)`, 2408 -> 2415 -> 2418 for `feat(layout)`), then lowered it again in refactors (2438 -> 2329 in `refactor(dock): route Hyprland IPC events from a logic module`, 2418 -> 2412, 2383 -> 2382 in v4.6.0). Outside PRs have stayed at or under the ceiling: #67 reported "2381 ... strictly under the 2383 ratchet", #71 "exactly 2382", #72 "stays at its 2382-line ceiling: one property added, one duplicate blank line removed". `AGENTS.md` ("Maintainability Rules") is referenced by the gates but is gitignored (`.gitignore`), so its text was not readable; `tests/static/structure-check.py` is the checked-in statement of the rules. Live suites also have per-file ratchets (`SHELL_CEILINGS`, `structure-check.py:36-47`).

PR template (`.github/PULL_REQUEST_TEMPLATE.md`): "What and why", then a verification checklist (node check/test, python unittest, qmllint, security-grep + structure-check + shaders-in-sync, manifest-check + `omarchy plugin validate`, smoke test).

## 7. Zero-CPU and performance norms

- The README's headline claim is "strictly **0.00% background CPU**" and "zero-CPU autohide ... on Hyprland events only" (`README.md:41`, `:220`). Release notes verify it as jiffies per 6 s idle, target at most 3 (`README.md:832`, `:862`, `:937`), and `tests/bench/bench.py` measures CPU/RAM/VRAM and dock on/off cost.
- Event-driven only. Concretely in the code: inotify block reads (`notification-popups.py`), `udevadm monitor` plus POLLPRI on `/proc/self/mounts` (`drive-removal-watch.py:3-13`, `Dock.qml:1183`), `CappedFileView` file watches, Hyprland `rawEvent` handlers, single-shot debounce timers (20, 100, 120, 180, 300, 600 ms), and the capped 25-tick startup retry. A `Process` is gated by a `running:` binding so it exists only when its feature is on.
- Maintainer comments that apply the norm: #67 "keeping it purely reactive, no timers anywhere in the diff, holds the 0.00% idle CPU line"; #31 "block-uevent watcher with POLLPRI mount-table refresh is the correct zero-idle-CPU design"; #70 "both timers are single-shot so the zero-CPU invariant holds"; #73 "the cost-until-used property matters in a zero-CPU dock" (lazy `Loader`s); #3 on polling `/proc` for TUI identity: "the polling overhead and race conditions you hit ... would also break Omadock's zero-CPU contract".
- Nuance: `scripts/terminal-hosts.py` does walk `/proc`, but as a one-shot run per window-list change, debounced 100 ms and bounded to known CLI names (`Dock.qml:950-972`). So on-event scans are accepted; background polling is not.
- Helpers outlive the shell. the changelog records that `Process` helpers "do not exit with the Quickshell process" and the live harness leaked 80 of them before `teardown.sh` was added (`README.md:866`). Any long-lived helper must exit when its stdin/parent goes away, and must be gated by a `running:` binding.

## 8. What outside contributions landed, and in what shape

Open and closed PRs were enumerated with `gh pr list -R thepathless/omadock --state all` (77 numbers; one open: #77).

| PR | Author | Size | Tests / docs | Outcome |
| :--- | :--- | :--- | :--- | :--- |
| #67 Task HUD | NothingManTR (outside) | +637/-17, 9 files, 1 commit; new `DockTasks.js`, `scripts/omadock-task.sh`, `task-status.test.mjs` +144 | 218 Node + 116 Python passing, stated ratchet compliance, motivation section, no new timers | Merged to `experimental` 2026-10-15, 17 h after opening. Maintainer added a fix-up (substring match bleed, empty colour), put `DockTasks.js` in CI, wrote `tests/live/tasks.sh`, credited in v4.6.0 changelog. |
| #71 media/privacy/progress badges | NothingManTR | +453/-45, 7 files; extends `DockTasks.js`, `DockPinLogic.qml` +111 (MPRIS, PipeWire) | 254 Node + 116 Python, live `tasks.sh`, names the priority hierarchy and ceiling | Merged 2026-10-17, v4.7.0. Maintainer checked "Pipewire exists on Quickshell 0.3.1". |
| #72 floating workspaces | priard (labelled CONTRIBUTOR, frequent author) | +267/-7, 11 files, 2 commits (prerequisite `CappedFileView.missing()` first) | `floating.test.mjs`, README section, settings row, feature detection, "Not yet tested by hand" list stated honestly | Merged 2026-10-20 same day, v4.8.0. Review praised the seam and failure-mode handling. |
| #11 multi-monitor | G-Pappas | +319/-13, 5 files | Docs and settings toggles; opt-in default off | Merged; maintainer asked the author to verify on a multi-monitor rig, who did. |
| #50 restore to own workspace | lordwotton29 | +1573/-152, 20 files | Live `place-restore.sh` | Integrated, v4.1.0. |
| #1, #2, #3, #4, #8, #10, #13 | assada, NothingManTR, Tech0001, tdslot, priard | various | | Closed, not merged as-is: the maintainer integrated the work on `experimental` or `main` by rewriting around the new architecture, or fixed the same bug independently, and credited the author in the changelog. |
| #27 to #33 | priard | small to +1373 | | Closed with "integrated into `experimental` as merge <sha>"; the repo has a habit of closing a PR after merging its commits by hand. |
| #77 perf(model) | Reflectionist | +31/-23, 4 files | | Open since 2026-10-27, no comments yet. |

Patterns:

- **Base branch.** Every PR must target `experimental`; `main` receives only verified release batches (`pr-base.yml`, `pr-autotarget.yml`, comment text on every PR).
- **Size.** Outside feature PRs are 5 to 53 files and from hundreds of lines to a few thousand; the maintainer merges large ones (#47 +2332, #49 +2871, #64 +2407 by priard). Pure additions of a module plus a thin hook are the fast path (#67 merged in 17 h).
- **Tests.** Every merged feature PR carries Node unit tests for its pure module; helper scripts carry Python tests (`test_drive_removal_watch.py`, `test_list_folder.py`); anything with an IPC or drawn result gets a live suite (the maintainer wrote `tasks.sh` for #67 when the author had not).
- **Docs.** README section plus config key documented, and a settings row with an `i18n/en.json` key. The maintainer writes the changelog entry and credits the author by handle.
- **Tone.** Warm and specific, names the two or three design choices that were liked, lists the integration defects found and fixed, and states exactly what was run ("244 Node + 116 Python tests, qmllint 0 errors, structure-check PASSED ..."). PR authors open with a motivation section and a performance/ceiling statement; maintainers value being told the ceiling and idle-CPU status up front.
- **What gets friction.** A PR cut from an old baseline (#4, #8, #13) gets re-integrated rather than merged. Substring matching of app ids was the one correctness defect found in the HUD (#67), and a missing CI list entry. Neither was a rejection.
- **Open issues** show the maintainer's current pain points: id-collapsing (#76), a latched autohide (#75).

## 9. Precedent or objection: Python helper, model API, herdr

**Long-running Python helper process: precedent exists and is approved.**
- `scripts/notification-popups.py` is a persistent process (blocking `select` on an inotify fd, JSON snapshot per line on stdout), spawned by `Process notificationPopupWatch` only while its feature is on and the shell service is absent (`Dock.qml:1509`). Maintainer review of #27: "The popup-file fallback with timestamp-based `newPopupRows` diffing is clean, and we especially like the priming guard so a shell restart doesn't bounce every app".
- `scripts/drive-removal-watch.py` (udev + POLLPRI) was praised in #31 as "the correct zero-idle-CPU design for this codebase".
- Short-lived helpers: `list-folder.py` (run under `timeout -k 2 10`, with a stale-result guard, `Dock.qml:1117-1123`), `terminal-hosts.py`, `list-drives.py`, `eject-drive.py`, `pick-folder.py`, `drop-check.py`.
- Rules a new helper must meet: no inline `python3 -c` (security-grep fails it); data goes in argv or stdin, never concatenated into `sh -c`; size and count caps and `O_NOFOLLOW` style hardening in the existing readers; Python tests under `tests/unit/`; `py_compile` clean; and it must not poll. The only stated objection is to polling (#3), not to Python or to a resident process.

**Network call to a model API: no precedent, no stated objection.**
- Full-tree grep (`curl`, `http(s)://`, `XMLHttpRequest`, `fetch(`, `WebSocket`, `socket`, `network`) over `.qml/.js/.py/.sh` finds only `xdg-open`/`omarchy-launch-webapp` URLs, sponsor links in `SettingsAbout.qml`, URL parsing of notification bodies and webapp Exec lines (`DockModel.js:57-65`, `:231`), and the bench's `socket.gethostname()`. The plugin makes no network request of its own.
- Searching every PR and issue body, comment and review in `thepathless/omadock` for "network", "LLM", "model API", "daemon", "long-running", "background process", "external tool" and "dependency" found no maintainer statement about network access. The only comment adjacent to the question is PR #1's "No new dependencies" from the contributor.
- Constraints that will apply regardless: the marketplace-review security rules encoded in `security-grep.py` ("as the Omarchy marketplace reviews it"), the zero-CPU stance (an HTTP call must be request-driven and bounded, with no retry loops), the `timeout -k` precedent for anything that can stall, and a default-off setting like #11 (multi-monitor "opt-in and off by default").
- The one analogous external call in the dock is `omarchy plugin enable|disable omarchy.dock` from #72, which was accepted because it is feature-detected (`$OMARCHY_PATH/shell/plugins/dock` must exist) and runs only when the state differs.

**Dependency on herdr (or any external tool): no mention; approved analogues exist.**
- `herdr` does not appear in the repository, its PRs or its issues. The nearest precedents are: `udevadm` (`Dock.qml:1184`), `canberra-gtk-play` (`DockNotifLogic.qml:123`), `gtk-launch`/`uwsm-app`, `qs ipc` in `omadock-task.sh`, and `omarchy plugin ...` in #72. All are on an Omarchy install by default or are feature-detected, and the dock degrades silently (e.g. #73: "an Omarchy refactor degrades gracefully instead of leaving a dead button").
- `scripts/omadock-task.sh` already knows about AI coding CLIs by name (`claude`, `aider`, `cursor`, `gemini`, ...) and walks `/proc` to attribute them to a terminal tile. herdr is not on that list.
- Inference: a herdr integration would need to be optional, detected at runtime, and silent when absent. Reading herdr state through a socket or file watch fits the event-driven norm; a poller does not.

## 10. Open questions for the Focus design

1. Holding notifications: the dock has no hook for it. Either Jev's verdict only gates the dock's own badge/urgency/sound (feasible in `DockNotifLogic.qml`), or holding happens in Omarchy's notification service (outside this repo). Decide which before scoping.
2. New windows: `DockEventLogic.handleRawEvent` already receives Hyprland raw events (`Dock.qml:110`, `:1473`); admitting or holding a new window means acting there, and every Hyprland action is routed through `hyprDispatch`/`luaString` (`Dock.qml:1769-1770`), which `security-grep` lists for review.
3. Where Focus state lives: the group shape in `omadock.json` with `bound*` parsing in `DockModel.js` (1 line of ceiling left, so a new `DockFocus.js` module is required), shared across docks through `sharedStore` like `taskStatus`.
4. An HTTP call to Jev from QML (`XMLHttpRequest`) or from a Python helper is new ground. No objection is on record, but the first such PR will set the precedent; a default-off setting, a request timeout and zero polling are the safest framing.

## Sources

- Repository tree at `cdaac69` (v4.8.0): files cited inline. `git log -L32,32:tests/static/structure-check.py` for ceiling history.
- PRs and issues read through `gh pr view` / `gh issue view` / `gh pr list` / `gh issue list` against `thepathless/omadock`: #1, #2, #3, #4, #8, #10, #11, #13, #27 to #33, #43, #50, #67, #71, #72, #77, issues #5, #6, #7, #9, #66, #75, #76. Keyword sweep over every PR and issue body, comment and review for polling, daemon, network, LLM, dependency and herdr terms.
- Not available: `AGENTS.md` (gitignored upstream), so its "Maintainability Rules" prose was not read.

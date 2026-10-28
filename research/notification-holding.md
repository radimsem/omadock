# How can omadock hold or delay notifications in Omarchy 4.5?

Ticket: radimsem/omadock#5. Map: radimsem/omadock#1.

Sources read:

- `omarchy`: `/home/radimsem/personal/projects/omarchy` at tag `v4.5.0-rc2`. `upstream/quattro` has no newer commit touching `shell/plugins/notifications` or `docs/notifications.md` (`git log HEAD..upstream/quattro -- <those paths>` is empty).
- `quickshell`: commit `2b4501a` (0.3.2.r7, the version installed on this desktop), `src/services/notifications/`, read from the GitHub mirror.
- `qtbase` v6.11.0 for one D-Bus default.
- `omadock`: this branch, at `cdaac69` (v4.8.0).
- The live desktop (read-only): Omarchy 4.5.0rc2-1; `GetCapabilities`, `GetServerInformation` and `omarchy-shell notifications isDnd` were called. No DND toggle, no notification sent, no `shell.json` edit.

## Short answer

1. **Omarchy 4.5 has no per-source hold.** The only quiet rule is a global Do Not Disturb boolean (`Delivery.js:45-50`). There is no admission hook, no per-app rule and no IPC to hold or release one source. A per-source timed snooze (keyed by `groupKey`, persisted in `quiet.json`) existed in rc1 and was deleted in rc2 by commit `7bab7c2f3` ("Remove notification snooze", 2026-10-23).
2. **A third-party plugin cannot reach the notification service at all.** omadock is an `overlay` plugin; only full `bar` plugins get a narrow `omarchy.notifications` proxy, and it exposes only `doNotDisturb` and `setDoNotDisturb` (`shell.qml:368`, `shell.qml:482`, `PluginFirstPartyServiceApi.qml:33-35`). Everything else goes through `omarchy-shell notifications <method>` or the state files.
3. **The one route that works on stock 4.5 is DND plus a ledger.** The plugin turns DND on for the Focus, watches `history.json` for rows with `closed_reason: "silenced"`, keeps its own copy, and draws its own cards. It holds every notification that does not bypass DND, not only the ones from other Focuses. A held Slack message loses its live sender, so the `default` action (jump to the exact conversation) is gone.
4. **A proxy daemon is blocked.** The shell registers the bus name without `ALLOW_REPLACEMENT` (Qt default, `server.cpp:117`), so a proxy cannot take the name while the shell runs. Only disabling `omarchy.notifications` frees it, which removes the whole Omarchy notification UI.
5. **The clean fix is a small upstream seam**: timed hold rules keyed by `groupKey`, plus IPC to list and release. About 100 lines, and it restores a feature the maintainers just removed, so acceptance is uncertain.

**Recommendation for v1:** ship the DND plus ledger route behind a clearly named "hold everything else" mode, and feature-detect an upstream `hold` IPC for true per-source holding. Open the upstream PR in parallel (sketch in section 5). Do not ship a bus proxy, a second `NotificationServer`, or a clone of the notifications plugin in v1.

Side finding that blocks any notification work: omadock's current observation path is the 4.0 layout and sees nothing on 4.5 (section 1).

Not verified: how Chromium or Slack react to a `NotificationClosed(Dismissed)` for a message the user never saw; which apps set the `transient` hint; omadock behaviour on a live 4.5 session (omadock is not installed on this desktop).

## 1. What omadock does today (and why it is stale on 4.5)

| Fact | Evidence |
|---|---|
| The dock asks for `shell.serviceFor("omarchy.notifications")` and falls back to a Python watcher | `components/logic/DockNotifLogic.qml:16-24`, `Dock.qml:1480-1499`, `Dock.qml:1509-1533` |
| On 4.5 the service lookup returns null for omadock: `pluginOwnsTarget` is true only for the plugin's own id, and the first-party proxy is built only for `bar` plugins | `shell.qml:393-401`, `shell.qml:643-647`, `shell.qml:368-370`, `docs/omarchy-shell.md:52` |
| `popupModel`, which the dock reads, does not exist in the 4.5 service. The live cards are a private `ListModel` named `toasts` | `Dock.qml:1547-1570` vs `Service.qml:161-167` |
| The fallback watcher reads one JSON object per popup from `~/.local/state/omarchy/notifications/*.json` and skips anything that is not an object | `scripts/notification-popups.py:32-46` |
| 4.5 writes `live.json` and `history.json` (arrays) and `quiet.json` (`{dnd}`), and removes 4.0's per-popup files by migration | `docs/notifications.md:39-49`, `Store.js:261` |
| On this desktop the watcher prints `[]`, and `notifications.json` (the dock's DND file, `Dock.qml:1351-1356`) does not exist | `python3 -I scripts/notification-popups.py --once` printed `[]`; `ls ~/.local/state/omarchy` |

Consequences: sticky badges and the urgency bounce see no notification on 4.5, and `isDndActive` (`Dock.qml:1376-1389`) is always false, so the chime plays during DND (`DockNotifLogic.qml:130`). The third-party plugin installed here, `shavanced.notification-center`, is on the same 4.0 layout (`BarWidget.qml:7-9`, `46-50`).

Whatever holding route is chosen, the feed has to move to the 4.5 files or the bus first. That is a prerequisite, not part of this ticket.

## 2. How a notification flows in 4.5 (the facts every route depends on)

```
app --Notify--> Quickshell NotificationServer singleton --signal--> Service.handleNotification
```

- The shell owns the name through `NotificationServer` (`Service.qml:1339-1349`), with capabilities `persistence body body-markup body-hyperlinks actions icon-static` (live `GetCapabilities` returns exactly those six).
- `handleNotification` snapshots the notification, then asks `Delivery.quietReason(row, doNotDisturb)` (`Service.qml:590-602`). The reason can only be `silenced`, and only when DND is on (`Delivery.js:45-50`).
- Three kinds bypass DND: `app_name == omarchy-action`; urgency critical and `app_name == notify-send`; a row with a detected verification code (`Delivery.js:11-15`, `Delivery.js:48`, `docs/notifications.md:70-74`). Chat apps' own critical urgency does not bypass.
- A silenced row is written to `history.json` with `closed_reason: "silenced"` unless it is ephemeral: the `transient` hint, or `app_name` `notify-send` or `omarchy-action`. Ephemeral rows are dropped (`Service.qml:604`, `Delivery.js:37-41`).
- The shell then calls `release(key)`, which sets `tracked = false` on the sender's `Notification` (`Service.qml:605`, `719-731`). In Quickshell that is `close(Dismissed)` (`notification.cpp:238-242`), and an untracked notification is deleted and answered with `NotificationClosed(id, Dismissed)` (`server.cpp:202-212`). **The sender is told the user dismissed it.** The live object, with its libnotify actions, is gone (`actionsOf` needs `refs[key]`, `Service.qml:888-892`).
- A later `replaces_id` update finds no live object (`keyForOriginal` needs `refs[key]`, `Service.qml:570-575`), so it becomes a separate row, not an in-place update.
- What a history row keeps: key, `originalId`, `senderPid`, `execArgv`, `execActions`, glyph, app, `appIcon`, summary, body (cut to 4K), `rawBody`, source, `groupKey`, urgency, timeout, ts, plus `closed_ts` and `closed_reason`. Never images, links, codes or libnotify actions (`Store.js:161-162`, `183-189`).
- History keeps at most 100 entries and 24 hours, shared with every other closed card (`Store.js:149-151`, `193-198`).
- Replay (`showHistory`) puts back the **newest six** rows of any reason, as restored cards with a 20 s grace; critical ones wait for a click (`Service.qml:1354-1361`, `Store.js:83`, `126-138`). There is no replay by source or by key.
- A replayed card has no sender. A click tries the stored `execArgv`, then a window by pid, a window by source, the source's site, the first link, then `omarchy-hyprland-focus-app <app>` (`Service.qml:1065-1089`, `docs/notifications.md:35`).
- At most 100 notifications are live at once; the 101st is untracked and dropped (`Service.qml:173-184`, `582-585`).

## 3. The feed: how omadock can see notifications (needed by every hold route)

| Route | Cost | Sees | Limits |
|---|---|---|---|
| F1. Watch `history.json` and `live.json` with `FileView` (`watchChanges`) | None upstream. Writes land within 250 ms of a change (`Service.qml:1215-1219`). Files are 0600 (`docs/notifications.md:47`). | App, summary, body (4K), source, `groupKey`, urgency, `originalId`, `closed_reason` | No libnotify actions, no images. Ephemeral silenced rows never appear. History is capped at 100 and 24 h. |
| F2. `busctl --user monitor` with a narrow match on `Notify`, `NotificationClosed`, `ActionInvoked` | A child process blocked on the bus socket, so no idle CPU. Started without an access error on this desktop for 2 s (`timeout 2 busctl --user monitor ...`); no message arrived in that window. | Full arguments: action list, every hint, `replaces_id`, expire timeout, before the shell's decision | Read only. It cannot drop or delay anything. A broad match would read every message on the bus, so keep the match narrow. |
| F3. A second `NotificationServer` in the plugin | See route R7. High risk. | Live objects with actions | Not recommended. |

F1 is enough for a ledger and for Jev's judgement. F2 is only worth adding if v1 needs actions or the ephemeral rows.

## 4. Routes for holding and releasing

Legend for "Breaks": **actions** = libnotify action buttons and the `default` click; **replaces_id** = in-place updates; **critical** = urgency critical; **click** = where a click goes.

### R1. Global DND for the Focus, ledger from `history.json`, own cards (works on stock 4.5)

- **How:** `omarchy-shell notifications setDnd true` on Focus start, restore on end (`Service.qml:1395-1399`, `docs/notifications.md:68`). Watch F1. Copy each `silenced` row into an omadock ledger. Draw the admitted ones as a card above the dock. Release by showing the ledger list; `showHistory` is a fallback for the newest six only.
- **Cost:** A plugin runs the CLI itself; there is no facade call for it. The ledger needs its own 0600 file with the shell's discipline (no codes, bounded size, 24 h or user-chosen retention), because history is capped at 100 and shared. DND is persisted in `quiet.json` ("a reload must not lift it", `Service.qml:61`), so a crash during a Focus leaves the desktop silent. omadock must record whether it, not the user, turned DND on, and undo that marker on the next start.
- **User sees:** The bar's DND indicator lights (`bar/indicators/Dnd.qml:8-9`). Cards stop for everything except Omarchy's own toasts, `notify-send` critical alerts and verification codes. Admitted cards come from omadock's UI, not the shell's.
- **Breaks:**
  - Scope: it holds everything that does not bypass, including sources that belong to no Focus and to the current Focus. This does not meet "hold sources that belong to another Focus". Fixing it means omadock re-presenting those rows itself.
  - Actions: the live sender is released, so buttons and the `default` jump are gone. The click falls back to window by source, site URL, link, or focus-by-app-name, which for Slack on the web opens the site, not the conversation.
  - replaces_id: updates to a held message arrive as new rows. The ledger can fold them with `originalId` plus app within a session, since ids restart with the shell.
  - Critical: held like the rest unless the sender is bare `notify-send`. A restored critical card never expires.
  - Ephemeral rows (`transient` hint, `notify-send`, `omarchy-action`) are dropped, not held, so they are unreachable.
  - Eviction: a chatty source can push older rows out of the 100-entry history before the ledger copies them if omadock was not running.
- **Verdict:** **Use for v1**, labelled honestly as "hold everything else".

### R1b. R1 plus re-sending unassigned rows through `omarchy-action`

Re-send rows the plugin wants visible with `omarchy-notification-send --app-name omarchy-action ...`, which bypasses DND (`Delivery.js:12`, `docs/notifications.md:84-107`).

- **Cost:** Low code, but the shell then sees the sender as `omarchy-action`: no grouping per app (`Markup.js:141-147`), ephemeral, low urgency by default, original icon and source lost unless re-passed with `-i`/`--exec`.
- **Breaks:** actions (only `--exec`/`--action` argv commands, no libnotify), replaces_id, click (becomes whatever argv the plugin sets).
- **Verdict:** Reject for v1. It bends a bypass meant for Omarchy's own confirmations, and the result looks like a different app.

### R2. Per-app quiet rules in Service.qml or Delivery.js

- **Status:** None exist. `quietReason` takes the row and one boolean (`Delivery.js:45`). Git history shows a per-source snooze keyed by `groupKey`, with an expiry, closed reason `snoozed` and persistence in `quiet.json`, was added in `4671ae2d3` and shipped in `v4.5.0-rc1`; `7bab7c2f3` removed it before rc2, with `a4c2a306c` and `cbd44ba1d` trimming the surrounding options and the panel. It never had IPC (the rc1 IPC list has no snooze method).
- **Verdict:** Nothing to call. The deleted code is the best reference for section 5.

### R3. Existing "hold" machinery: the deck hold

- **How:** New cards wait in `held` while the deck is open under the pointer or in keyboard mode, released after at most 30 s (`Service.qml:234-258`). The `keyboard` IPC enters keyboard mode (`Service.qml:1427-1431`).
- **Breaks:** It only runs when at least one card exists (`startKeyboard` returns false otherwise, `Service.qml:305`), it ends after 10 idle seconds (`Service.qml:284`), and it grabs exclusive keyboard focus on the notification surface (`docs/notifications.md:196`), which swallows typing. Wrong tool for Writing.
- **Verdict:** Reject.

### R4. Observe, then dismiss over IPC

- **How:** Watch F2 or F1, then `omarchy-shell notifications dismiss "<summary>"` (`Service.qml:1437-1446`).
- **User sees:** The card animates in over 250 ms and out over 200 ms (`Service.qml:442-443`): a flash. The sender gets `Dismissed`, and history records `dismissed`, indistinguishable from the user clicking away.
- **Breaks:** Summary substring matching can close unrelated cards. actions, click: gone. No ledger entry with a hold reason.
- **Verdict:** Reject.

### R5. A hook that runs per notification

- **Status:** The notifications plugin has no hook point (no `omarchy-hook` call in `shell/plugins/notifications/`). A synchronous script per arrival would delay every card. A declarative rule (section 5) avoids that.
- **Verdict:** Not available; do not propose one.

### R6. D-Bus proxy daemon or name takeover

- **Why it fights the shell:** `NotificationServer` registers `org.freedesktop.Notifications` with Qt's `registerService(name)` (`server.cpp:117`), whose defaults are `DontQueueService` and `DontAllowReplacement` (`qdbusconnectioninterface.h:63-65`). A second claimant using `REPLACE_EXISTING` is refused. The shell retries only when the current owner leaves (`server.cpp:48-59`, `131-134`). On the live bus the owner is the shell's `quickshell` process (`busctl --user list`).
- **Two ways it can still happen:**
  - Disable `omarchy.notifications` (`disabledPlugins[]` in `shell.json`) and run a full replacement daemon. You lose the cards, decks, keyboard control, history and `quiet.json`. The `notifications` IPC target disappears, which breaks `omarchy-notification-wait` (`docs/notifications.md:184`), the Super+comma bindings and the bar DND toggle. This route also needs a `shell.json` edit, which is out of bounds here.
  - Put apps behind `xdg-dbus-proxy` (installed) so their `Notify` calls go to a queue first. This works only for apps started under the wrapper, not for running apps or for web apps inside the browser.
- **Verdict:** Reject for v1. Takeover is graceful on failure (the shell re-registers), but the feature loss is not.

### R7. A second `NotificationServer` inside the plugin

- **What it gives:** `NotificationServer` is a process-wide singleton and every QML instance connects to the same `notification` signal (`server.cpp:62-65`, `qml.cpp:15-30`). A plugin instance would receive the live `Notification` objects, actions included. `tracked` is one flag on the shared object and is **false by default** (`notification.hpp:209`). A plugin that sets `tracked = true` after the shell has set it false (the shell does that on the DND path, `Service.qml:605`) would keep the object alive and unanswered, preserving actions and `replaces_id`.
- **Why not:**
  - Order: the plugin's handler must run after the shell's. Connection order depends on load order, which is not stable across reloads.
  - Capabilities: constructing the server copies the plugin's `support` flags over the shell's (`qml.cpp:17`). Any mismatch with the six flags the shell advertises silently turns off actions or markup for every app.
  - Reloads: construction calls `switchGeneration` (`qml.cpp:26`). With `keepOnReload` true (the default, `qml.hpp:139`) it re-emits every live notification with `tracked = false` to all servers including the shell's (`server.cpp:74-79`). With `keepOnReload: false` it expires and deletes them all (`server.cpp:88-92`). Plugins reload on every file save (`manual/32-shell-plugins.md:73`).
  - A held object that nobody closes leaks until the next generation switch.
- **Verdict:** Reject for v1. Worth revisiting only as a documented upstream "parking" seam (section 5, step 2).

### R8. Clone `omarchy.notifications` and patch the copy

- **How:** `omarchy plugin clone omarchy.notifications` copies the plugin to `~/.config/omarchy/plugins/<user>.notifications` and keeps the IPC identity (`docs/omarchy-shell.md:70-81`). The patch adds hold rules.
- **Cost:** About 3,500 lines of notification code fork away from upstream, which changed the plugin in 56 commits between the engine swap (`4671ae2d3`, 2026-10-19) and rc2 (2026-10-27). Every security fix in `Security.js`, `Markup.js` or `Store.js` is missed until someone re-merges. It cannot ship inside the omadock repo as one plugin; it is a second plugin the user clones.
- **Breaks:** Nothing, if the patch is right. The held object can stay tracked, so actions and `replaces_id` survive.
- **Verdict:** Fallback for people who want per-source holds before upstream moves. Not a default.

### R9. A minimal upstream seam (recommended in parallel)

Detailed in section 5.

## 5. Upstream seam: sketch of a minimal PR

Goal: hold rules keyed by `groupKey`, each with an expiry, plus IPC to list and release. It reuses the history, the `silenced` path and `restoreRows`.

**Behaviour**

- `hold <groupKey> <seconds>`: add or refresh `{ groupKey, until }`. `groupKey` is what `Markup.regroup` produces: `app:slack` or `web:app.slack.com` (`Markup.js:141-147`). The expiry caps a crashed plugin: a stale rule ends on its own.
- `unhold <groupKey>`.
- `held`: JSON of live rules and the history rows whose reason is `held`.
- `release <groupKey>`: replay those rows with `restoreRows(rows, true)` and drop them from history so they do not show twice.
- Rules apply only after the bypass checks, so Omarchy toasts, `notify-send` critical alerts and verification codes always show.

**Code (sketch against rc2)**

```js
// Delivery.js
function holdsRow(row, holds, now) {
  var key = String((row && row.groupKey) || "")
  for (var i = 0; i < holds.length; i++)
    if (holds[i].groupKey === key && holds[i].until > now) return true
  return false
}

function quietReason(row, doNotDisturb, holds, now) {
  if (bypassesQuiet(row)) return ""
  if (String((row && row.code) || "")) return ""
  if (doNotDisturb) return "silenced"
  return holdsRow(row, holds || [], now) ? "held" : ""
}
```

```qml
// Service.qml
property var holds: []   // persisted in quiet.json as { dnd, holds: [{ groupKey, until }] }
...
var muted = Delivery.quietReason(row, doNotDisturb, holds, Date.now() / 1000)
if (muted) {
  // A held row is recorded even when transient: it was held on purpose.
  if (muted === "held" ? row.app !== "notify-send" && row.app !== "omarchy-action"
                       : !Delivery.isEphemeral(row)) storeRecord(row, muted)
  ...unchanged...
}
...
// ShellIpc: hold(groupKey, seconds), unhold(groupKey), held(), release(groupKey)
```

`Store.parseQuiet` and `serialiseQuiet` carry `holds` (bounded: 32 rules, 12 h maximum, invalid entries dropped). `closed_reason` accepts `held` already (`Store.js:187`, regex `^[a-z]{1,32}$`). Add the IPC methods to `docs/notifications.md` and a case to `test/shell.d/notifications-test.sh`.

Size estimate: about 25 lines in `Delivery.js` and `Store.js`, 50 in `Service.qml`, 10 in docs, 30 in tests.

**Step 2 (not minimal): park the live object.** Keep the `Notification` tracked while held, reusing the `pending.row`/`held` fields (`Service.qml:234-258`, `620-628`). That preserves the `default` action and `replaces_id`. It needs: skip `storePut` for parked rows (`Service.qml:617`), or a restart restores them as visible cards with a 20 s grace (`Store.js:83`); a separate cap, since parked rows count against the 100 live slots and the 101st is dropped (`Service.qml:173-184`); and handling for a sender that withdraws (`Service.qml:633-642`).

**Acceptance risk:** the maintainers removed snooze (`7bab7c2f3`: "the bell is a plain Do Not Disturb toggle again"), trimmed notification IPC "nothing calls" (`a4c2a306c`), and state that "notifications have no settings of their own" (`docs/notifications.md:80`). A PR must argue it is for a plugin API (no UI, no setting, rules that expire) and should not revive the panel or bell states. Raise it as an issue with the sketch before writing code. Rejection is a real outcome, so v1 cannot depend on it.

## 6. Comparison

| Route | Per-source | Keeps actions | Keeps `replaces_id` | Critical | Needs upstream | v1 |
|---|---|---|---|---|---|---|
| R1 DND + ledger | No (all non-bypass) | No | No (folded in ledger) | Held, except bare `notify-send` | No | Yes |
| R1b re-send as `omarchy-action` | Partly | No | No | Re-set by plugin | No | No |
| R3 deck hold | No | Yes | Yes | n/a | No | No |
| R4 observe then dismiss | Yes | No | No | n/a | No | No |
| R6 proxy / takeover | Yes | Depends | Depends | Depends | No | No |
| R7 second server | Yes | Yes | Yes | Yes | No | No |
| R8 clone and patch | Yes | Yes (with parking) | Yes | Yes | No | Fallback |
| R9 upstream hold rule | Yes | No (step 2: yes) | No (step 2: yes) | Held | Yes | Parallel |

## 7. Decisions needed

1. Accept "hold everything else" for v1 on stock 4.5, or wait for per-source support? The first ships now and meets "held items are always reachable" through the ledger; it does not meet "only other Focuses' sources".
2. Open the upstream issue first (cheap, shows appetite) or send the PR directly?
3. Whether the ledger may store message text. It must, to stay reachable after the shell's 24 h window, so it needs a retention setting and the shell's redaction rules.

Suggested v1 order: (a) move the feed to `history.json`, `live.json` and `quiet.json` (also repairs badges and the DND chime); (b) the ledger and restore-on-crash marker; (c) R1 hold with admission cards; (d) feature-detect `omarchy-shell notifications held` for the upstream path.

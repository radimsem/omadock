# Which signals identify a newly opened window or site?

Resolves [radimsem/omadock#4](https://github.com/radimsem/omadock/issues/4). Part of the Focus map in #1.

Researched 2026-10-28 on Hyprland 0.56.2, Quickshell 0.3.2 (Omarchy build), Omarchy v4.5.0-rc2, omadock v4.8.0 (`cdaac69`).

## Short answer

1. **A new window is announced reliably, with no setup.** Hyprland's `openwindow` event carries address, workspace name, class and title. omadock already listens for it (`components/logic/DockEventLogic.qml`) and already holds `{ title, address, appId }` for every window (`DockModel.buildEntries`). Title changes arrive as `windowtitlev2`, focus changes as `activewindowv2`. Nothing new is needed to know that a window opened and what app it is.
2. **Omarchy web apps identify their site by class, for free.** `omarchy-webapp-install` writes `StartupWMClass=chrome-<host>_<path>-Default`, derived from the launch URL (`chrome-x.com__-Default`, `chrome-github.com__-Default`). That is the Wayland app id of the window for as long as it lives, so "x.com belongs to Social" works for any Omarchy web app with a string match. It names the URL the app was launched at, not wherever the user navigates later.
3. **A site opened in an ordinary Chromium or Firefox tab is not identifiable without help.** The class is just `chromium` or `firefox`. The title is the page title plus a product suffix (`AKV Look Directions — Mozilla Firefox` on this machine). No Wayland or Hyprland field carries the URL. The demo's "tidepool.dev · Fits Building" for a normal tab needs either the title as a hint or an opt-in extension.
4. **The only realistic opt-in for URLs is a small extension plus a native-messaging host, not a DevTools port.** Omarchy already ships exactly this pattern for Chromium (`copy-url` extension, `com.omarchy.copy_url` host, installed into every Chromium-family profile root). A DevTools port is blocked on the default profile since Chrome 136, and on a separate profile it cannot see the user's real tabs. Firefox has the same shape (WebExtension plus native host) but needs a signed extension.
5. **Terminal apps are already solved by class and `/proc`.** Omarchy gives TUIs `org.omarchy.<binary>` ids (or fixed ones), and omadock already walks `/proc` for the foreground CLI.

## Ranked recommendation for v1

| Rank | Signal | Covers | Setup | Ship in v1 |
| --- | --- | --- | --- | --- |
| 1 | Window class (`appId`) and title from the existing model, triggered by `openwindow`, `windowtitlev2`, `activewindowv2` | Every native app, terminals, Omarchy TUIs, browser windows at app level | None | Yes, the base case |
| 2 | Host parsed from an Omarchy web app class (`chrome-<host>_<path>-Default`) | Omarchy web apps (X, GitHub, YouTube, Discord, ChatGPT and so on) | None | Yes, small pure function |
| 3 | Browser window title as a hint for tabs | Ordinary Chromium and Firefox windows, coarse | None | Yes, but label it a hint and decide what may reach cloud Jev |
| 4 | Chromium extension plus native-messaging host reporting the active tab URL | Exact site for ordinary Chromium-family tabs | One extension, one host manifest, per-browser | Design the input as `url?` now, build as an opt-in follow-up |
| 5 | Firefox WebExtension plus native host | Same, for Firefox | Signed extension, host manifest | Later, only if Firefox users ask |
| 6 | DevTools or WebDriver BiDi port | Tab URLs | Separate profile or flag, open local port | No |
| 7 | AT-SPI address bar, session files | Tab URLs | Accessibility forced on, or parsing private profile files | No |

Rationale: ranks 1 to 3 need nothing installed, are event-driven (no polling, which omadock's maintainers object to, see #2), and already match the existing window model. Rank 4 is the only route to a real URL, but it puts a browser extension on the user's machine and sends URLs to Jev, which belongs behind the open "Privacy envelope" question on the map. Build the Focus judgment input as `{ appId, title, host?, url? }` so ranks 2 and 4 slot in without a contract change.

## Hyprland IPC events

Source: [Hyprland wiki, IPC](https://wiki.hypr.land/IPC/) (page source: [content/ipc/_index.md](https://github.com/hyprwm/hyprland-wiki/blob/main/content/ipc/_index.md), fetched via Context7 and GitHub). Events stream on `$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock` as `EVENT>>DATA\n`.

| Event | Data | Use for Focus |
| --- | --- | --- |
| `openwindow` | `WINDOWADDRESS,WORKSPACENAME,WINDOWCLASS,WINDOWTITLE` | The "new window" trigger. Class is the app id at map time. Title is the title at map time and is often a placeholder. |
| `windowtitlev2` | `WINDOWADDRESS,WINDOWTITLE` | Title changed. The site-change hint for browsers and the status hint for terminals. |
| `windowtitle` | `WINDOWADDRESS` | Same, without the title. Prefer v2. |
| `activewindowv2` | `WINDOWADDRESS` | Focus moved. The "active site changed" trigger for a window whose title then differs. |
| `activewindow` | `WINDOWCLASS,WINDOWTITLE` | Same, v1. The title can contain commas, so split on the first comma only. |
| `closewindow` | `WINDOWADDRESS` | Window gone. |
| `movewindowv2` | `WINDOWADDRESS,WORKSPACEID,WORKSPACENAME` | Moved between workspaces. |
| `urgent` | `WINDOWADDRESS` | Window asked for attention. |
| `fullscreen`, `changefloatingmode`, `pin` | state flags | Not identity signals. |

Facts that matter:

- `openwindow` gives the class but not the pid, the initial title, or anything about URLs. The full record comes from `hyprctl clients -j` (keys seen on 0.56.2: `address, class, initialClass, title, initialTitle, pid, xwayland, xdgTag, xdgDescription, workspace, monitor, tags, ...`).
- `class` and `title` change over a window's life, `initialClass` and `initialTitle` do not. The Hyprland window-rule docs say static rules "always" see the initial values ([window-rules.md](https://github.com/hyprwm/hyprland-wiki/blob/main/content/configuring/core/rules/window-rules.md)). Web apps keep their launch class.
- `xdgTag` and `xdgDescription` (from the xdg-toplevel-tag protocol) exist as fields and as a rule matcher (`xdg_tag`), but were empty on all five windows on this machine. No usable signal today.
- Quickshell's own source notes that Hyprland sometimes sends `windowtitlev2` and `urgent` before `openwindow` ([connection.cpp](https://github.com/quickshell-mirror/quickshell/blob/master/src/wayland/hyprland/ipc/connection.cpp)). Key state by address and tolerate any order.
- Verified locally: `hyprctl version` reports 0.56.2; a short listen on socket2 produced no events because nothing happened in the window, so the event shapes above are from the wiki and Quickshell's parser, not from a live capture.

## What Quickshell's Hyprland module exposes

Sources: [Hyprland singleton](https://quickshell.org/docs/v0.3.1/types/Quickshell.Hyprland/Hyprland), [HyprlandToplevel](https://quickshell.org/docs/v0.3.1/types/Quickshell.Hyprland/HyprlandToplevel), [HyprlandEvent](https://quickshell.org/docs/v0.3.1/types/Quickshell.Hyprland/HyprlandEvent), [Wayland Toplevel](https://quickshell.org/docs/v0.3.1/types/Quickshell.Wayland/Toplevel). Source read from Quickshell master, the installed build is 0.3.2, so check the behaviours below against it before relying on them.

- `Hyprland.rawEvent(HyprlandEvent)`: every socket2 event. `event.name`, `event.data` (unparsed), `event.parse(argumentCount)`. The argument count is required because the last argument may contain commas (titles do).
- `Hyprland.toplevels` (model of `HyprlandToplevel`): `address`, `title`, `workspace`, `monitor`, `activated`, `urgent`, `wayland` (the Wayland Toplevel handle), `lastIpcObject`.
- `Hyprland.refreshToplevels()`: re-runs `j/clients`.
- `Hyprland.activeToplevel`, `focusedMonitor`, `focusedWorkspace`, `dispatch(request)`, `usingLua`.
- Wayland `Toplevel` (from `ToplevelManager.toplevels` or `HyprlandToplevel.wayland`): `appId`, `title`, `activated`, `minimized`, `fullscreen`, `parent`, `screens`.

Gotchas, read from Quickshell's source:

- On `openwindow` Quickshell creates the `HyprlandToplevel` with address, title and workspace only. **It drops the class and does not fill `lastIpcObject`.** `lastIpcObject` is filled by `refreshToplevels()`, which Quickshell runs at startup and on `configreloaded`. The docs warn it "is not updated unless the toplevel object is fetched again". omadock's comment at `Dock.qml:540` says the same.
- `HyprlandToplevel.title` does update live from `windowtitlev2`.
- So the right fields are **`toplevel.appId` and `toplevel.title` from the Wayland handle** for identity, and `Hyprland.rawEvent` as the trigger. That is what `DockModel.buildEntries` already does. Do not read `class` or `title` from `lastIpcObject` for a new window.
- The Wayland handle is `null` until the address is reported, so a freshly opened window may need one model rebuild before its `appId` is visible. omadock handles this with its 40 ms `modelTimer`.

### Fit with omadock's existing window model

| Signal | Already in omadock | Where |
| --- | --- | --- |
| `openwindow` | Yes, marks the address as just opened (3 s) and restarts the model timer | `DockEventLogic.qml` |
| `windowtitlev2` | Yes, restarts the model timer and the terminal identity scan | `DockEventLogic.qml` |
| `activewindowv2` | Yes, clears urgency; active id comes from `ToplevelManager.activeToplevel` | `DockEventLogic.qml`, `Dock.qml` |
| Window `appId`, `title`, `address` | Yes, per window in the entry's `windowList` | `DockModel.buildEntries` |
| Web app host from class | Partly: `getCandidates` strips `chrome-`, `helium-`, `brave-` and similar prefixes and the `-Default` suffix to match launcher entries | `DockModel.js:118` |
| Terminal foreground CLI | Yes, limited to `agy` and `btop` | `scripts/terminal-hosts.py` |
| Tab URL | No | none |

What Focus adds: an event that fires once per new address and once per title change of the active window, carrying `{ address, appId, title }`, deduplicated. No new IPC is needed.

## Identifying a site in Chromium-family browsers without an extension

### Omarchy web apps (`--app`)

Launch path: `omarchy-webapp-install` writes a `.desktop` with `Exec=omarchy-launch-webapp "<url>"`, which runs `<browser> --app="<url>"` (`omarchy/bin/omarchy-webapp-install`, `omarchy/bin/omarchy-launch-webapp`, Omarchy v4.5.0-rc2). The supported browsers are Chrome, Brave, Edge, Opera, Vivaldi and Helium, with Chromium as the fallback.

The window class is deterministic:

- Chromium names an `--app=URL` window after `host + "_" + path` ([`GenerateApplicationNameFromURL`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/web_applications/web_app_helpers.cc)).
- The Linux shortcut and app id is `<kBrowserProcessExecutableName>-<app_name>-<profile dir name>` with illegal path characters replaced by `_` ([`GetShortcutFileName`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/web_applications/os_integration/web_app_shortcut_linux.cc)). The executable name constant is `chrome` on POSIX ([chrome_constants.cc](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/common/chrome_constants.cc)).
- Omarchy mirrors this in `omarchy-webapp-window-class` and writes it to `StartupWMClass`. Its acceptance test pins the shape, including with a custom `--user-data-dir` (`^chrome-.*_omarchy-ime-entry\.html-Default$`, `omarchy/test/acceptance.d/input-methods-test.sh`). A custom user-data dir does not change the class; only the profile directory name does (`-Default` becomes `-Profile_1`).

Real classes on this machine (from `~/.local/share/applications`, launchers installed by Omarchy):

| Web app | Class |
| --- | --- |
| X | `chrome-x.com__-Default` |
| GitHub | `chrome-github.com__-Default` |
| YouTube | `chrome-youtube.com__-Default` |
| ChatGPT | `chrome-chatgpt.com__-Default` |
| Figma | `chrome-figma.com__-Default` |
| WhatsApp | `chrome-web.whatsapp.com__-Default` |
| Basecamp | `chrome-launchpad.37signals.com__-Default` |
| Discord | `chrome-discord.com__channels_@me-Default` |

Every Omarchy web app gets a distinct class, because the class is the launch URL. Two web apps on the same host with different paths also differ. There was no Chromium window open at research time, so the classes above are from the launchers' `StartupWMClass`, which Omarchy derives from Chromium's rule, plus Omarchy's acceptance test, not from a live `hyprctl clients` capture.

Caveats:

- The class is the **launch** URL. If the user navigates inside the app window to another site, the class and the Wayland app id stay put. Treat it as "this app", which is what a web app means.
- Non-Chromium-base forks may change the prefix. omadock's `getCandidates` already tolerates `helium-`, `brave-`, `edge-`, `opera-`, `vivaldi-` and `google-chrome-` prefixes, but I could not verify Helium's real prefix without a Helium install. Check once with `hyprctl clients -j | jq '.[].class'` while a Helium `--app` window is open.
- `omarchy-cmd-browser-handoff` passes `--app=URL` to the already-running browser over its singleton socket, so every window shares one pid. `/proc/<pid>/cmdline` therefore shows the browser's first command line, not the app URL. The pid route does not work for Chromium.

### Ordinary browser windows and tabs

- Class is the product: the browser's desktop-file base name, for example `chromium` (see [`GetProgramClassClass`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/shell_integration_linux.cc) for the rule, which also honours `--class`).
- Title is the active tab's page title plus the product name. Sites often put their name in it (`GitHub - ...`, `(3) Home / X`), but nothing guarantees it. "tidepool.dev" would only show up if the page title says so.
- The class does not change when switching tabs; only `windowtitlev2` fires. So "the active site changed" is observable as a title change, and "which site" is a guess from the title.
- `--user-data-dir` and `--profile-directory` change profile data and the web-app suffix, not the class of ordinary windows.

## DevTools remote debugging and native messaging: realistic opt-ins?

### DevTools port (`--remote-debugging-port`)

What it would give: `GET http://127.0.0.1:<port>/json/list` returns one entry per tab with `url` and `title`, and the websocket protocol can subscribe to navigation events. That is the closest equivalent to Thariq's direct tab-URL read.

Why it is not a realistic v1 opt-in:

- **Blocked on the real profile.** Since Chrome 136, `--remote-debugging-port` and `--remote-debugging-pipe` "will no longer be respected if attempting to debug the default Chrome data directory", and must be paired with a non-standard `--user-data-dir` ([Chrome for Developers](https://developer.chrome.com/blog/remote-debugging-port)). A separate user-data dir is a different profile: none of the user's tabs, logins or Omarchy web apps. It cannot observe the browser they actually use. The post is about Chrome; I did not verify that plain Chromium and Helium apply the same block, so treat it as likely but unconfirmed for them.
- **Security cost.** The port is unauthenticated control of the browser, reachable by any local process. Google's stated reason for the change is cookie theft through this exact switch. Loopback-only binding limits remote hosts, not other local software.
- **Setup cost on Omarchy.** It means a flag in `~/.config/chromium-flags.conf` plus a separate profile. `omarchy-cmd-browser-handoff` refuses the singleton handoff when a `--user-data-dir` is present (`omarchy/bin/omarchy-cmd-browser-handoff`), which slows every launch and changes how web apps start.

Verdict: reject for v1. If ever offered, it is a developer-only mode, never the default path.

### Native messaging with an extension

What it gives: an extension with the `tabs` permission can read the active tab's URL and push it on `tabs.onActivated`, `tabs.onUpdated` and `windows.onFocusChanged`, event-driven with no polling. A native host on the other end of stdin/stdout can write to a socket or file the dock reads.

Mechanics, from the [Chrome native messaging docs](https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging):

- The extension declares `nativeMessaging`. The host manifest is JSON named after the host, with `allowed_origins` listing `chrome-extension://<id>/` (no wildcards).
- Linux per-user manifest dirs: `~/.config/google-chrome/NativeMessagingHosts/`, `~/.config/chromium/NativeMessagingHosts/` (and a system path under `/etc/chromium/`). Each browser reads its own directory.
- Framing is JSON over stdin/stdout with a 32-bit native-endian length prefix. Host to browser is capped at 1 MB, browser to host at 64 MiB.

**Omarchy already ships this, which cuts the cost.** In v4.5.0-rc2:

- `default/chromium/extensions/copy-url` is an MV3 extension with `activeTab` and `nativeMessaging` and a pinned `key` (so a fixed extension id). It calls `chrome.runtime.sendNativeMessage('com.omarchy.copy_url', { url })`.
- `omarchy-install-chromium-copy-url` writes `com.omarchy.copy_url.json` into `NativeMessagingHosts/` of Chromium, Chrome, Brave, Edge and their beta and nightly dirs. It exists on this machine (`~/.config/chromium/NativeMessagingHosts/com.omarchy.copy_url.json`).
- `config/chromium-flags.conf` loads the extensions with `--load-extension=...`.

So an "omadock tab reporter" can copy that pattern: one more bundled extension, one more host manifest, same installer layout. Costs and limits:

- **Extension permissions.** Reading any tab's URL needs the `tabs` permission (or host permissions). `activeTab`, which the existing extension uses, only grants access after a user gesture, so it cannot watch silently. This is a visible, broader permission and a real privacy decision.
- **`--load-extension` is gone from branded Chrome.** Chrome 137 removed it for branded builds; Chromium and Chrome for Testing keep it ([chromium-extensions PSA](https://groups.google.com/a/chromium.org/g/chromium-extensions/c/1-g8EFx2BBY/m/S0ET5wPjCAAJ)). It works for Omarchy's default Chromium, and for forks only if they keep the flag (Helium unverified). Chrome users would have to side-load by hand.
- **Mapping a tab to a Hyprland window.** The extension knows its browser window, not the Wayland window. Simplest correct rule: when the active Hyprland window is a Chromium-family window, report the active tab of the last-focused browser window, and let the dock pair it with that address. Multi-window browsers and `--app` windows need care. Web apps are already covered by their class.
- **Privacy.** A URL, unlike a title, is a precise fact. It should pass through whatever the privacy envelope decides (for example host only, never path or query) before it reaches a cloud Jev.
- **Security.** The host runs with the user's rights and `allowed_origins` pins it to one extension id. Risk is a compromised extension or a host that accepts input it should not. Keep the host write-only toward a user-owned socket.

Verdict: the only credible URL source. Do not ship in the first Focus PR. Design the signal contract so it can arrive later.

## Firefox equivalents

On this machine: class `firefox`, title `<page title> — Mozilla Firefox`, `initialTitle` `Mozilla Firefox` (from `hyprctl clients -j`).

- **No `--app` mode.** Firefox has no per-site window with its own app id on Linux, so nothing like the web-app class exists. Everything is title-based unless an extension helps.
- **Remote debugging.** `--remote-debugging-port` starts the Remote Agent, loopback only, default `localhost:9222`, WebDriver BiDi over WebSockets, no authentication, and it can extract cookies and session data ([Firefox Remote Agent security](https://firefox-source-docs.mozilla.org/remote/Security.html)). Firefox removed CDP, so there is no `/json/list` ([CDP retirement](https://fxdx.dev/cdp-retirement-in-firefox/)); a client must speak BiDi. Same verdict as Chromium: it is a test-automation interface, wrong for a passive dock. I did not verify whether it works against the default profile.
- **Native messaging.** Same shape: host manifest in `~/.mozilla/native-messaging-hosts/<name>.json` (or `/usr/lib/mozilla/native-messaging-hosts/`), with `allowed_extensions` holding the add-on id, and `nativeMessaging` in the extension's permissions ([MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Native_messaging), [native manifests](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Native_manifests)). Release Firefox requires a signed extension, so this means distributing through AMO or a self-hosted signed XPI. Heavier than Chromium.
- **Session files.** Firefox writes open tabs and URLs to `sessionstore-backups/recovery.jsonlz4` in the profile, at most every 15 seconds by default ([Mozilla session restore wiki](https://wiki.mozilla.org/Firefox/session_restore), from support answers rather than current source). It needs no extension, but it reads the user's private profile data, lags up to 15 s, and uses a compressed format. Not recommended; I did not open the file here.

## Terminal apps (`org.omarchy.*`)

From Omarchy v4.5.0-rc2:

| Class | Source | Notes |
| --- | --- | --- |
| `org.omarchy.<binary>` | `omarchy-launch-tui` and `omarchy-launch-or-focus-tui` | Default: `org.omarchy.$(basename $1)`, so `org.omarchy.btop`, `org.omarchy.nvim`, `org.omarchy.bash` |
| `org.omarchy.terminal` | `omarchy-launch-floating-terminal-with-presentation` | Title is `Omarchy`; the observed window here is this one |
| `org.omarchy.about`, `org.omarchy.screensaver` | about and screensaver launchers | Fixed ids |
| `org.omarchy.agent` | `omarchy-agent` | One class for every agent terminal; the terminal's real name survives only in `initialTitle`, which `omarchy-hyprland-focus-app` matches |
| `TUI.float`, `TUI.tile` | `omarchy-tui-install` | Apps installed through the TUI installer share two classes, so the class does not tell them apart |
| `com.mitchellh.ghostty`, `Alacritty`, `kitty`, `foot` | Plain shell windows | Class is the terminal, not the program. Title is shell-set (`omarchy: dotfiles` on this machine) |

All of these match Omarchy's `terminal` tag rule `(Alacritty|kitty|com.mitchellh.ghostty|foot|org\.omarchy\..*|TUI\..*)` (`omarchy/default/hypr/apps/terminals.lua`), a ready-made "is a terminal" test via the `tags` field in `hyprctl clients -j`.

For "what is running in this terminal", omadock's `scripts/terminal-hosts.py` walks `/proc/<pid>/task/*/children` from the window's pid (the terminal) down to a known CLI name, bounded by depth and count. It maps only `agy` and `btop` today. A Focus-oriented extension is a longer allowlist (editor, `claude`, `docker`), run on `openwindow` and `windowtitlev2`, which the dock already triggers. This is event-driven and local, which matches the maintainers' constraints. Titles in terminals can hold paths or commands, so they need the same privacy treatment as browser titles.

## Open items and what was not verified

- No Chromium, Helium or Brave window was open, so no live `openwindow` capture and no live web-app class check. Class shape rests on Chromium source, Omarchy's installer and acceptance test, and the launchers on disk.
- Helium's real web-app prefix and whether it keeps `--load-extension` and the remote-debugging restriction are unverified.
- Quickshell behaviour was read from master source; the installed build is 0.3.2. The docs are v0.3.1.
- Whether the `tabs` permission and URL-only reporting are acceptable to omadock's maintainers is a design decision, not a research result. It depends on the privacy envelope question on the map.
- AT-SPI (reading the omnibox via accessibility) was considered and not tested. Chromium keeps accessibility off unless an assistive technology is detected or `--force-renderer-accessibility` is set ([Chromium accessibility overview](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/accessibility/overview.md)), and a 2020 report says auto-detection was disabled for a time. It would also force a performance cost on every page. Not recommended.

## Sources

- Hyprland IPC: https://wiki.hypr.land/IPC/
- Hyprland window rules (`xdg_tag`, initial values): https://github.com/hyprwm/hyprland-wiki/blob/main/content/configuring/core/rules/window-rules.md
- Quickshell Hyprland module: https://quickshell.org/docs/v0.3.1/types/Quickshell.Hyprland/Hyprland , `.../HyprlandToplevel` , `.../HyprlandEvent` ; Wayland Toplevel: https://quickshell.org/docs/v0.3.1/types/Quickshell.Wayland/Toplevel
- Quickshell source: https://github.com/quickshell-mirror/quickshell/tree/master/src/wayland/hyprland/ipc
- Chromium web app name and app id: https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/web_applications/web_app_helpers.cc , https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/web_applications/os_integration/web_app_shortcut_linux.cc , https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/shell_integration_linux.cc , https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/common/chrome_constants.cc
- Chrome remote debugging change: https://developer.chrome.com/blog/remote-debugging-port
- Chrome native messaging: https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging
- `--load-extension` removal in branded Chrome: https://groups.google.com/a/chromium.org/g/chromium-extensions/c/1-g8EFx2BBY/m/S0ET5wPjCAAJ
- Firefox Remote Agent security: https://firefox-source-docs.mozilla.org/remote/Security.html ; CDP retirement: https://fxdx.dev/cdp-retirement-in-firefox/
- Firefox native messaging: https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Native_messaging , https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Native_manifests
- Firefox session restore: https://wiki.mozilla.org/Firefox/session_restore
- Local, read-only: Omarchy v4.5.0-rc2 (`bin/omarchy-webapp-install`, `omarchy-webapp-window-class`, `omarchy-launch-webapp`, `omarchy-launch-tui`, `omarchy-cmd-browser-handoff`, `omarchy-install-chromium-copy-url`, `default/chromium/extensions/copy-url`, `default/hypr/apps/terminals.lua`, `shell/plugins/dock/Dock.qml`); omadock `DockModel.js`, `Dock.qml`, `components/logic/DockEventLogic.qml`, `scripts/terminal-hosts.py`; `hyprctl clients -j` on the running desktop.

# Tor Browser — command scenarios for deep research

Every Tor Browser 16.0 launcher option and the Firefox options the bundled
binary accepts, grouped by workflow. Options from [`help.txt`](help.txt)
(`./start-tor-browser --help` plus `firefox.real --help`). Script options are
parsed by `Browser/start-tor-browser`; the rest are passed through to Firefox.

## How to invoke

```text
@deep-research For Tor Browser 16.0 on Linux, which Firefox command-line
switches are safe to use without weakening fingerprint uniformity, and which
ones (remote debugging, Marionette, custom profiles) create a linkable
identity? Cite Tor Project design docs, Firefox source, and real automation
write-ups; return a safe automation recipe.
```

---

## Scenario 1 — first launch, logging and app registration

| # | Command | How to use |
|---|---------|------------|
| 1 | `~/tor-browser/Browser/start-tor-browser` | Normal GUI launch |
| 2 | `~/tor-browser/Browser/start-tor-browser --detach` | Launch and return to shell |
| 3 | `~/tor-browser/Browser/start-tor-browser --verbose` | Tor + browser output in terminal |
| 4 | `~/tor-browser/Browser/start-tor-browser --log` | Log to `tor-browser.log` |
| 5 | `~/tor-browser/Browser/start-tor-browser --log ~/tb.log` | Custom log path |
| 6 | `~/tor-browser/Browser/start-tor-browser --register-app` | Create desktop entry |
| 7 | `~/tor-browser/Browser/start-tor-browser --unregister-app` | Remove desktop entry |
| 8 | `~/tor-browser/Browser/start-tor-browser --help` | Script + Firefox options |
| 9 | `desktop-file-validate ~/.local/share/applications/start-tor-browser.desktop` | Validate the entry |
| 10 | `~/tor-browser/Browser/firefox --version` | Bundled Firefox version |
| 11 | `~/tor-browser/Browser/firefox --full-version` | Version + build IDs |

**Research questions**
- What changes when `--register-app` rewrites `Exec=`, and why does it break a
  custom desktop entry?
- Does `--log` capture sensitive URLs, and where should logs be stored?
- Why does the launcher refuse to run as root, and what is the supported
  workaround?

**Sources**: Tor Browser manual, launcher script, Tor Project tickets on
desktop integration, Debian desktop-entry spec.

**Output**: a launch command and a known-good desktop entry, with log
retention rules.

---

## Scenario 2 — profiles and identity separation

| # | Command | How to use |
|---|---------|------------|
| 12 | `~/tor-browser/Browser/start-tor-browser -P work` | Start a named profile |
| 13 | `~/tor-browser/Browser/start-tor-browser --profile ~/tb-profiles/work` | Profile at a path |
| 14 | `~/tor-browser/Browser/start-tor-browser --ProfileManager` | Profile picker |
| 15 | `~/tor-browser/Browser/start-tor-browser --migration` | Import from another browser |
| 16 | `~/tor-browser/Browser/start-tor-browser --first-startup` | Post-install actions only |
| 17 | `~/tor-browser/Browser/start-tor-browser --safe-mode` | Disable extensions/themes for one run |
| 18 | `cp -a ~/tor-browser/Browser/TorBrowser/Data/Browser/profile.default ~/tb-profiles/work` | Clone a profile before first use |
| 19 | `ls ~/tor-browser/Browser/TorBrowser/Data/Browser/` | See profile storage |

**Research questions**
- Do separate profiles share the same Tor circuits or entry guards?
- Does `--migration` import bookmarks only, or also cookies/history that link
  identities?
- Is `--safe-mode` enough to diagnose a broken extension without losing data?

**Sources**: Firefox profile docs, Tor Browser design doc on linkability, Tor
Project support entries.

**Output**: an identity-separation plan with one profile per activity and a
note on shared Tor state.

---

## Scenario 3 — open pages, windows and search

| # | Command | How to use |
|---|---------|------------|
| 20 | `~/tor-browser/Browser/start-tor-browser https://example.com` | Open a URL |
| 21 | `~/tor-browser/Browser/start-tor-browser --browser https://example.com` | Force a browser window |
| 22 | `~/tor-browser/Browser/start-tor-browser --new-window https://example.com` | New window |
| 23 | `~/tor-browser/Browser/start-tor-browser --new-tab https://example.com` | New tab |
| 24 | `~/tor-browser/Browser/start-tor-browser --private-window https://example.com` | Private window |
| 25 | `~/tor-browser/Browser/start-tor-browser --private-window` | Private window, no URL |
| 26 | `~/tor-browser/Browser/start-tor-browser --search 'tor browser hardening'` | Search with default engine |
| 27 | `~/tor-browser/Browser/start-tor-browser --preferences` | Open Preferences |
| 28 | `~/tor-browser/Browser/start-tor-browser --setDefaultBrowser` | Set as default browser |

**Research questions**
- In Tor Browser, does `--private-window` add any protection beyond the
  already-private default mode?
- Which search engine is default in 16.0, and can `--search` leak the query?
- Does `--setDefaultBrowser` create an OS-level linkability signal?

**Sources**: Tor Browser manual, Firefox CLI docs, Tor Project blog on search.

**Output**: a navigation command set that matches the intended identity and a
note on what each option does *not* isolate.

---

## Scenario 4 — headless screenshots and kiosk display

| # | Command | How to use |
|---|---------|------------|
| 29 | `~/tor-browser/Browser/start-tor-browser --headless https://example.com` | No GUI |
| 30 | `~/tor-browser/Browser/start-tor-browser --headless --screenshot ~/shot.png https://example.com` | Screenshot to file |
| 31 | `~/tor-browser/Browser/start-tor-browser --headless --screenshot https://example.com` | Screenshot to cwd |
| 32 | `~/tor-browser/Browser/start-tor-browser --headless --window-size 1280,900 --screenshot ~/shot.png https://example.com` | Set viewport |
| 33 | `~/tor-browser/Browser/start-tor-browser --kiosk https://example.com` | Kiosk mode |
| 34 | `~/tor-browser/Browser/start-tor-browser --kiosk --kiosk-monitor 1` | Kiosk on a monitor |
| 35 | `~/tor-browser/Browser/start-tor-browser --disable-pinch` | Disable pinch gestures |

**Research questions**
- Does headless mode use the same Tor network path and fingerprint defenses?
- Is `--screenshot` affected by the letterboxing/window-size defenses?
- What kiosk lockdown is missing (address bar, shortcuts, downloads)?

**Sources**: Firefox headless docs, Tor Browser letterboxing docs, kiosk
deployment write-ups.

**Output**: a screenshot recipe with reproducible viewport and a warning about
what it changes.

---

## Scenario 5 — developer tools and local debugging

| # | Command | How to use |
|---|---------|------------|
| 36 | `~/tor-browser/Browser/start-tor-browser --jsconsole` | Browser Console |
| 37 | `~/tor-browser/Browser/start-tor-browser --devtools` | DevTools on load |
| 38 | `~/tor-browser/Browser/start-tor-browser --jsdebugger` | Browser Toolbox |
| 39 | `~/tor-browser/Browser/start-tor-browser --jsdebugger --wait-for-jsdebugger` | Wait for debugger attach |
| 40 | `~/tor-browser/Browser/start-tor-browser --MOZ_LOG=nsHostResolver:5` | NSPR logging |
| 41 | `~/tor-browser/Browser/start-tor-browser --MOZ_LOG_FILE=~/tb-moz.log --MOZ_LOG=nsHostResolver:5` | Log to file |
| 42 | `~/tor-browser/Browser/start-tor-browser --g-fatal-warnings` | Turn warnings fatal |

**Research questions**
- Do DevTools requests bypass the browser's proxy settings?
- Which `MOZ_LOG` modules reveal DNS/hostnames, and how do you scrub them?
- Can the Browser Toolbox connect to a remote device without weakening Tor
  Browser's isolation?

**Sources**: Firefox remote debugging docs, Tor Browser design doc, Mozilla
logging docs.

**Output**: a local-only debugging recipe with a log-scrubbing step.

---

## Scenario 6 — automation interfaces (high risk)

| # | Command | How to use |
|---|---------|------------|
| 43 | `~/tor-browser/Browser/start-tor-browser --marionette` | Enable Marionette |
| 44 | `~/tor-browser/Browser/start-tor-browser --remote-debugging-port 9222` | WebDriver BiDi / CDP-style port |
| 45 | `~/tor-browser/Browser/start-tor-browser --remote-debugging-port 9222 --remote-allow-hosts 127.0.0.1` | Restrict Host header |
| 46 | `~/tor-browser/Browser/start-tor-browser --remote-debugging-port 9222 --remote-allow-origins http://127.0.0.1:9222` | Restrict Origin |
| 47 | `~/tor-browser/Browser/start-tor-browser --remote-debugging-port 9222 --remote-allow-system-access` | Privileged access (dangerous) |
| 48 | `~/tor-browser/Browser/start-tor-browser --start-debugger-server 6000` | Legacy devtools server |
| 49 | `~/tor-browser/Browser/start-tor-browser --start-debugger-server ws:6000` | WebSocket devtools server |
| 50 | `~/tor-browser/Browser/start-tor-browser --allow-remote` | Accept remote commands |
| 51 | `~/tor-browser/Browser/start-tor-browser --dbus-service /usr/bin/torbrowser` | D-Bus application service |
| 52 | `~/tor-browser/Browser/start-tor-browser --origin-to-force-quic-on https://example.com` | Force QUIC for one origin |

**Research questions**
- Which of these interfaces are reachable from other local users, and what is
  the default bind address?
- Does enabling Marionette alter the JS fingerprint or timing in a detectable
  way?
- Is there a supported Selenium/Playwright path that keeps Tor Browser's
  patches intact?

**Sources**: Marionette docs, Firefox remote agent security guidelines, Tor
Project automation tickets, Selenium/Playwright Tor Browser guides.

**Output**: an automation setup bound to loopback, with a written risk
statement and a teardown step.

---

## Scenario 7 — X11 and display control

| # | Command | How to use |
|---|---------|------------|
| 53 | `~/tor-browser/Browser/start-tor-browser --display :1 https://example.com` | Use a specific X display |
| 54 | `~/tor-browser/Browser/start-tor-browser --sync` | Synchronous X calls (debugging) |
| 55 | `DISPLAY=:1 ~/tor-browser/Browser/start-tor-browser` | Display via environment |

**Research questions**
- Does running on a non-default display or nested X server change the
  fingerprint?
- Is `--sync` safe for normal use, or only for debugging crashes?
- How do you run Tor Browser in a disposable VM/container without breaking its
  sandbox?

**Sources**: X11 docs, Tor Browser sandbox docs, Whonix/Tails design docs.

**Output**: a display isolation recipe for testing, not for daily use.

---

## Complete option coverage

Every option in [`help.txt`](help.txt) appears at least once.

| Surface | Options | Scenario |
|---------|---------|----------|
| Tor Browser script | `--verbose`, `--log [file]`, `--detach`, `--register-app`, `--unregister-app`, `-h/--help` | 1 |
| Version | `-v/--version`, `--full-version` | 1 |
| Profiles | `-P`, `--profile`, `--ProfileManager`, `--migration`, `--first-startup`, `--safe-mode` | 2 |
| Navigation | URL positional, `--browser`, `--new-window`, `--new-tab`, `--private-window`, `--search`, `--preferences`, `--setDefaultBrowser` | 3 |
| Capture/kiosk | `--headless`, `--screenshot`, `--window-size`, `--kiosk`, `--kiosk-monitor`, `--disable-pinch` | 4 |
| DevTools/logging | `--jsconsole`, `--devtools`, `--jsdebugger`, `--wait-for-jsdebugger`, `--MOZ_LOG`, `--MOZ_LOG_FILE`, `--g-fatal-warnings` | 5 |
| Automation | `--marionette`, `--remote-debugging-port`, `--remote-allow-hosts`, `--remote-allow-origins`, `--remote-allow-system-access`, `--start-debugger-server`, `--allow-remote`, `--dbus-service`, `--origin-to-force-quic-on` | 6 |
| X11 | `--display`, `--sync` | 7 |

Operational notes:
- The launcher refuses to run as root.
- Bundled Tor exposes SOCKS on `127.0.0.1:9150`; system Tor uses `9050`.
  Point CLI tools at 9150 with `torsocks -P 9150` when sharing Tor Browser's
  Tor.
- `TOR_CONTROL_PASSWD` is read by the launcher for control-port access; do not
  put a real secret in a shell history or a world-readable file.
- `--register-app` may rewrite `Exec=` to an upstream form; re-apply a
  desktop-spec-valid `Exec=/home/user/tor-browser/Browser/start-tor-browser
  --detach %U` if validation fails.

Full help text: [`help.txt`](help.txt); authoritative docs:
https://support.torproject.org/tbb/ and
https://firefox-source-docs.mozilla.org/.

# Burp Suite Community Edition — intercepting proxy & manual web testing

- **Version:** 2026.10.1-54773
- **Launcher:** `/usr/local/bin/burp` → `/opt/BurpSuiteCommunity`
- **Type:** GUI Java application (no headless scanner in Community)
- **Vendor docs:** https://portswigger.net/burp/documentation
- **Academy (free labs):** https://portswigger.net/web-security

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Burp sits between your browser and the target as an HTTP/S intercepting proxy.
You can pause, inspect, modify, replay, and fuzz requests. Community Edition
includes the manual toolset; the automated vulnerability scanner is Pro-only.

### What Community includes
| Tool | Purpose | Community limits |
|------|---------|------------------|
| **Proxy** | Intercept/modify browser traffic, history | Cannot save projects; history is session-only |
| **Repeater** | Manually edit and resend requests | Full |
| **Intruder** | Automated request fuzzing | **Throttled** (rate-limited), still usable |
| **Decoder** | Encode/decode/hash data | Full |
| **Comparer** | Diff requests/responses | Full |
| **Sequencer** | Token randomness analysis | Full |
| **Extender** | Load BApp Store / custom extensions | Full (BApp Store available) |
| **Logger / Organizer** | Traffic logging / saved requests | Session-only |
| **Scanner** | Automated vuln scanning | **Not included** (Pro only) |

## Launch

```bash
burp                 # normal GUI launch
burp --version       # print version
burp --help          # all launcher flags
```

Useful launcher flags:

| Flag | Meaning |
|------|---------|
| `--use-defaults` | Start with default settings (troubleshooting) |
| `--disable-extensions` | Don't load extensions at startup |
| `--data-dir <dir>` | Use a custom data directory |
| `--project-file <file>` | Open/create a project file (Pro-oriented) |
| `--config-file <file>` | Load project config (repeatable) |
| `--user-config-file <file>` | Load user config (repeatable) |
| `--diagnostics` | Print diagnostic info |
| `--disable-auto-update` | Suppress auto-update |
| `--collaborator-server` | Run Collaborator server mode |

## First-run setup

1. **Start Burp** → choose *Temporary project* → *Use Burp defaults*.
2. **Proxy → Options** → confirm listener `127.0.0.1:8080`.
3. **Browser**: set HTTP/HTTPS proxy to `127.0.0.1:8080`.
   - Firefox: Settings → Network Settings → Manual proxy.
   - Chromium: launch with `--proxy-server=127.0.0.1:8080`.
   - Or use **Proxy → Intercept → Open browser** (Burp's bundled Chromium).
4. **Install CA cert**:
   - Browse to `http://burp` → *CA Certificate* → download `cacert.der`.
   - Firefox: Settings → Privacy & Security → Certificates → View Certificates →
     Authorities → Import → trust for websites.
   - Chromium: `chrome://settings/certificates` → Authorities → Import.
   - System (Debian): copy to `/usr/local/share/ca-certificates/burp.crt`
     (PEM) and run `sudo update-ca-certificates`.
5. **Disable browser QUIC/HTTP3** if HTTPS interception fails (Chromium:
   `chrome://flags/#enable-quic` → Disabled).

## Core workflows

### 1. Intercept and modify

```
Proxy → Intercept → Intercept is on
Browser → perform action
Burp → edit request → Forward / Drop / Action
```

Right-click a request → **Send to Repeater** (Ctrl+R) or **Send to Intruder** (Ctrl+I).

### 2. Repeater

- Edit method, path, headers, body.
- **Send** (Ctrl+Space) and inspect response.
- Use tabs to compare variants.
- Right-click → *Change request method*, *Paste URL as form parameter*, etc.
- Good for auth bypass, IDOR, parameter tampering, header injection.

### 3. Intruder (Community: throttled)

1. Send request to Intruder.
2. **Positions** tab → *Clear §* → select payload position → *Add §*.
3. **Payloads** tab → choose payload type (Simple list, Runtime file, Numbers,
   Brute forcer, etc.).
4. **Options** tab → threads, retries, grep-match.
5. **Start attack**.

Common attacks:
- Username/password brute force (careful with lockouts).
- Numeric IDOR enumeration (`/user/§1§`).
- Parameter fuzzing with a wordlist.
- Header/role tampering.

> Community Intruder is deliberately rate-limited. For heavy fuzzing use
> `ffuf`, `arjun`, `dalfox`, `nuclei`, or Pro.

### 4. Decoder

Encode/decode: URL, HTML, Base64, hex, ASCII hex, Gzip, JWT (via extension),
hash (SHA/MD5). Useful for cookies, tokens, and obfuscated parameters.

### 5. Comparer

Paste two requests/responses → word/byte diff. Useful for blind boolean logic,
access-control differences, and error-message analysis.

### 6. Sequencer

Capture a token (session ID, CSRF) → analyze entropy. Flags weak/predictable
tokens.

### 7. Extender / BApp Store

**Extensions → BApp Store** → install. High-value extensions:
- **Autorize** — access-control/IDOR checks.
- **JWT Editor** — JWT inspection and attacks.
- **Param Miner** — hidden parameter/header discovery.
- **Turbo Intruder** — high-speed request engine (useful even in Community).
- **Logger++** — advanced traffic logging.
- **Hackvertor** — encoding/swiss-army conversions.
- **Collaborator Everywhere** — OAST-style probing (needs a Collaborator server;
  Community can use the public server with limits).

Custom extensions: Java/Python/Ruby via Extender → Extensions → Add.

## Keyboard shortcuts (defaults)

| Shortcut | Action |
|----------|--------|
| Ctrl+R | Send to Repeater |
| Ctrl+I | Send to Intruder |
| Ctrl+Space | Send request (Repeater) |
| Ctrl+F | Search |
| Ctrl+Shift+F | Search across all tools |
| Ctrl+U | URL-encode selection |
| Ctrl+Shift+U | URL-decode selection |
| Ctrl+B | Base64-encode selection |
| Ctrl+Shift+B | Base64-decode selection |

## CLI recipes

```bash
# Launch
burp

# Launch with a clean config (extension troubleshooting)
burp --use-defaults

# Launch without extensions
burp --disable-extensions

# Custom data dir
burp --data-dir ~/.burp-data

# Version check
burp --version
```

## Project/user config files

Burp stores user settings under `~/.BurpSuite/` (or the configured data dir).
You can export/import options via **User options → Import/Export**. For
repeatable setups, keep a `--user-config-file` JSON and launch with it.

## Gotchas

- **No Scanner** in Community — use ZAP, nuclei, nikto, dalfox for automation.
- **No project save** in Community — export interesting requests or keep notes.
- Intruder is throttled; plan around it.
- HTTPS failures are almost always the CA cert or QUIC.
- Large responses can freeze the UI; use *Proxy → Options → response limits*.
- Extensions can break between Burp releases; disable them first when debugging.
- Burp binds `127.0.0.1:8080` by default — do not expose it to untrusted networks.

## References

- Docs: https://portswigger.net/burp/documentation
- Community vs Pro: https://portswigger.net/burp/communitydownload
- Web Security Academy: https://portswigger.net/web-security
- BApp Store: https://portswigger.net/bappstore

# mitmproxy — command scenarios for deep research

Every mitmproxy command form, grouped by scenario. `mitmproxy`, `mitmdump`, and
`mitmweb` share one option set; flags from the installed build (`help.txt`).

## How to invoke

```text
@deep-research For mitmproxy 12.3, which mode is correct for intercepting an
Android app that ignores the system proxy, and what certificate installation
steps still work on Android 14? Cite mitmproxy docs, Android docs, and current
community writeups; return exact commands.
```

---

## Scenario 1 — choose a frontend and mode

| # | Command | How to use |
|---|---------|------------|
| 1 | `mitmproxy` | Interactive TUI, regular HTTP proxy on 8080 |
| 2 | `mitmdump` | Headless dump, scriptable |
| 3 | `mitmweb` | Web UI frontend |
| 4 | `mitmproxy -m regular` | Explicit regular HTTP proxy |
| 5 | `mitmproxy -m local` | Local mode (transparent via OS redirect) |
| 6 | `mitmproxy -m transparent` | Transparent proxy (packet redirect) |
| 7 | `mitmproxy -m socks5` | SOCKS5 proxy mode |
| 8 | `mitmproxy -m reverse:https://example.com` | Reverse proxy to one upstream |
| 9 | `mitmproxy -m upstream:http://proxy:3128` | Chain to an upstream proxy |
| 10 | `mitmproxy -m wireguard` | WireGuard VPN mode, keys created on first run |
| 11 | `mitmproxy -m wireguard:/path/wg.conf` | WireGuard with explicit key material |
| 12 | `mitmproxy -m regular@127.0.0.1:8081` | Per-mode bind override |
| 13 | `mitmproxy -m regular -m socks5` | Spawn multiple proxy types |

**Research questions**
- Which modes work without root, and which require packet redirection?
- WireGuard mode versus a real VPN profile: what does each hide or expose?

**Sources**: mitmproxy mode docs, WireGuard mode docs, issues on Android.

**Output**: one correct mode per client type, with the privilege requirement
noted.

---

## Scenario 2 — observe a browser session

| # | Command | How to use |
|---|---------|------------|
| 14 | `mitmproxy --listen-host 127.0.0.1 -p 8080` | Bind explicitly |
| 15 | `mitmproxy --anticache` | Avoid 304s so bodies are visible |
| 16 | `mitmproxy --anticomp` | Ask servers for uncompressed bodies |
| 17 | `mitmproxy --showhost` | Build URLs from Host headers |
| 18 | `mitmproxy --show-ignored-hosts` | Keep ignored flows in the UI |
| 19 | `mitmproxy --stickycookie '~d example.com'` | Pin cookies for a domain |
| 20 | `mitmproxy --stickyauth '~d example.com'` | Pin auth for a domain |
| 21 | `mitmproxy --console-layout vertical` | TUI layout |
| 22 | `mitmproxy --no-console-layout-headers` | Hide layout headers |
| 23 | `mitmproxy -q` | Quiet logging |
| 24 | `mitmproxy -v` | Verbose logging |

**Research questions**
- Does `--anticache` alter requests in a way a target could detect?
- When does `--showhost` mislead analysis (spoofed Host headers)?

**Sources**: mitmproxy options docs, proxy theory, issues on anticache.

**Output**: a browser configured to proxy through mitmproxy with a trusted CA.

---

## Scenario 3 — capture, replay, and script

| # | Command | How to use |
|---|---------|------------|
| 25 | `mitmdump -w flows.mitm` | Stream all flows to a file |
| 26 | `mitmdump -w '+flows-%Y%m%d.mitm'` | Append, rotate by date |
| 27 | `mitmproxy -r flows.mitm` | Read flows from file |
| 28 | `mitmproxy -C requests.mitm` | Client replay from a file |
| 29 | `mitmproxy -S responses.mitm` | Server replay from a file |
| 30 | `mitmproxy -S responses.mitm --server-replay-extra 404` | Answer unmatched requests with 404 |
| 31 | `mitmproxy -S responses.mitm --server-replay-kill-extra` | Kill unmatched requests (deprecated form) |
| 32 | `mitmproxy -S responses.mitm --server-replay-reuse` | Reuse responses multiple times |
| 33 | `mitmproxy -S responses.mitm --server-replay-refresh` | Refresh date/cookie headers |
| 34 | `mitmproxy -s script.py` | Run an addon script |
| 35 | `mitmproxy -s a.py -s b.py` | Multiple scripts, in order |
| 36 | `mitmproxy --set terperrors=true` | Set any option ad hoc |
| 37 | `mitmproxy --options` | List every option and default |
| 38 | `mitmproxy --commands` | List every command signature |

**Research questions**
- `--server-replay-extra` semantics: forward, kill, or synthesize a status?
- Which script events fire for WebSocket and raw TCP flows?

**Sources**: mitmproxy replay docs, addon API docs, examples repo.

**Output**: a deterministic replay harness for a fixed request set.

---

## Scenario 4 — rewrite traffic on the fly

| # | Command | How to use |
|---|---------|------------|
| 39 | `mitmproxy -M '/example\.com/api/ https://staging.example.com/api/'` | Map remote URL |
| 40 | `mitmproxy --map-local '/static/ /tmp/static/'` | Map remote to local files |
| 41 | `mitmproxy -B '/api/ "admin":false/"admin":true/'` | Replace body text |
| 42 | `mitmproxy -B '/api/ @/tmp/body.json'` | Replace body from a file |
| 43 | `mitmproxy -H '/example\.com/ X-Forwarded-For/127.0.0.1'` | Add/replace a header |
| 44 | `mitmproxy -H '/example\.com/ Cookie/'` | Remove a header (empty value) |
| 45 | `mitmproxy -H '/example\.com/ X-Test/@/tmp/value.txt'` | Header value from a file |

**Research questions**
- Pattern separator handling when URLs contain `/`?
- Flow-filter syntax shared by map/modify options?

**Sources**: mitmproxy map/modify docs, filter expression reference.

**Output**: a rewrite rule set that is reproducible and reversible.

---

## Scenario 5 — TLS and certificates

| # | Command | How to use |
|---|---------|------------|
| 46 | `mitmproxy --certs 'example.com=/path/cert.pem'` | Present a specific cert |
| 47 | `mitmproxy --certs '*=/path/wildcard.pem'` | Wildcard default cert |
| 48 | `mitmproxy --cert-passphrase '...'` | Decrypt a protected key (visible in process list) |
| 49 | `mitmproxy -k` | Skip upstream TLS verification |
| 50 | `mitmproxy --emudhra` | Sign with the compromised eMudhra intermediate |
| 51 | `mitmproxy --emudhra-old` | Prefer the Sectigo cross-sign |
| 52 | `mitmproxy --set confdir=/path/conf` | Custom CA/config directory |

**Research questions**
- Why are `--emudhra` certs useful only for specific legacy trust stores?
- How to protect `--cert-passphrase` (config.yaml versus CLI)?

**Sources**: mitmproxy cert docs, eMudhra disclosure writeups, OpenSSL docs.

**Output**: a CA/cert plan appropriate to the client's trust store.

---

## Scenario 6 — scope and access control

| # | Command | How to use |
|---|---------|------------|
| 53 | `mitmproxy --allow-hosts 'example\.com'` | Intercept only matching hosts |
| 54 | `mitmproxy --ignore-hosts '^10\.'` | Bypass interception for matches |
| 55 | `mitmproxy --tcp-hosts '.*:443'` | Treat matches as raw TCP/TLS |
| 56 | `mitmproxy --proxyauth user:pass` | Require proxy authentication |
| 57 | `mitmproxy --proxyauth @/path/htpasswd` | htpasswd-backed proxy auth |
| 58 | `mitmproxy --upstream-auth user:pass` | Basic auth to an upstream proxy |
| 59 | `mitmproxy --no-server` | Do not start the proxy server |
| 60 | `mitmproxy --no-rawtcp` | Disable raw TCP handling |
| 61 | `mitmproxy --no-http2` | Disable HTTP/2 |
| 62 | `mitmproxy --store-streamed-bodies` | Keep streamed bodies inspectable |

**Research questions**
- Do `--ignore-hosts` and `--allow-hosts` match IP or hostname in each mode?
- Memory cost of `--store-streamed-bodies` on large downloads?

**Sources**: mitmproxy proxy options docs, issues on transparent mode matching.

**Output**: a scope boundary enforced at the proxy, not only by discipline.

---

## Scenario 7 — filters and interactive triage

| # | Command | How to use |
|---|---------|------------|
| 63 | `mitmproxy --intercept '~m POST'` | Pause POST requests for editing |
| 64 | `mitmproxy --intercept '~d example.com & ~p 443'` | Combine filters |
| 65 | `mitmproxy --view-filter '~t json'` | Show only JSON responses |
| 66 | `mitmproxy --view-filter '~s 500'` | Show only server errors |

**Research questions**
- Full filter-expression grammar and precedence for `&`, `|`, `!`?
- Does `--intercept` apply before or after modify options?

**Sources**: mitmproxy filter docs, addon event order docs.

**Output**: a repeatable triage view that highlights only relevant flows.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| General | `-h`/`--help`, `--version`, `--options`, `--commands`, `--set`, `-q`, `-v` | 3 |
| Mode | `-m`/`--mode` (regular, local, transparent, socks5, reverse:, upstream:, wireguard) | 1 |
| Display | `--anticache`, `--showhost`, `--show-ignored-hosts`, `--anticomp`, `--console-layout`, `--console-layout-headers` | 2 |
| Input/output | `-r`/`--rfile`, `-w`/`--save-stream-file`, `-C`/`--client-replay`, `-S`/`--server-replay` | 3 |
| Replay | `--server-replay-kill-extra`, `--server-replay-extra`, `--server-replay-reuse`, `--server-replay-refresh` | 3 |
| Rewrite | `-M`/`--map-remote`, `--map-local`, `-B`/`--modify-body`, `-H`/`--modify-headers` | 4 |
| Proxy | `--listen-host`, `-p`/`--listen-port`, `-n`/`--no-server`, `--ignore-hosts`, `--allow-hosts`, `--tcp-hosts`, `--proxyauth`, `--upstream-auth`, `--store-streamed-bodies`, `--rawtcp`, `--http2` | 6 |
| TLS | `--certs`, `--cert-passphrase`, `-k`/`--ssl-insecure`, `--emudhra`, `--emudhra-old` | 5 |
| Session | `-s`/`--scripts`, `--stickycookie`, `--stickyauth` | 2, 3 |
| Filters | `--intercept`, `--view-filter` | 7 |

Full per-flag text: [`help.txt`](help.txt).

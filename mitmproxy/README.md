# mitmproxy — scriptable intercepting proxy

- **Version:** 12.3.1
- **Binaries:** `mitmproxy` (TUI), `mitmweb` (web UI), `mitmdump` (CLI)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Docs:** https://docs.mitmproxy.org/stable/
- **Source:** https://github.com/mitmproxy/mitmproxy

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

mitmproxy is a TLS-capable intercepting proxy with three frontends and a Python
addon API. It can inspect, modify, replay, and record HTTP/1.1, HTTP/2, WebSocket,
and (partially) HTTP/3 traffic. Because it is scriptable, it is ideal for
automation, custom attacks, and evidence collection.

## Use cases

| Situation | Tool/mode |
|-----------|-----------|
| Interactive interception | `mitmproxy` TUI |
| Browser-friendly UI | `mitmweb` |
| Automated capture/modify | `mitmdump -s script.py` |
| Transparent proxy on a router | `--mode transparent` |
| Reverse proxy in front of an app | `--mode reverse:https://backend` |
| Upstream corporate proxy chain | `--mode upstream:http://proxy:8080` |
| Replay captured traffic | `mitmdump -r flows -n` / client replay |
| Modify responses on the fly | addon `response` hook |
| Save evidence | `-w flows.mitm` |

## Frontends

```bash
mitmproxy                 # interactive TUI on :8080
mitmweb                   # web UI on :8081, proxy on :8080
mitmdump                  # console, best for scripts/pipelines
```

## Common options

| Option | Meaning |
|--------|---------|
| `-p, --listen-port <port>` | Proxy port (default 8080) |
| `--listen-host <host>` | Bind address |
| `-m, --mode <mode>` | `regular`, `transparent`, `reverse:URL`, `upstream:URL`, `socks5`, `wireguard` |
| `-w, --save-stream-file <file>` | Write flows to file |
| `-r, --read-flows <file>` | Read flows from file |
| `-s, --scripts <script>` | Load Python addon(s) |
| `--set <key=value>` | Set any option |
| `-n, --no-server` | Don't start the proxy (offline processing) |
| `-q, --quiet` | Quiet |
| `-v, --verbose` | Verbose |
| `--flow-detail <level>` | TUI/dump detail (0–4) |
| `--set block_global=false` | Accept non-local connections |
| `--set confdir=<dir>` | Custom config dir |
| `--set console_focus_follow=true` | TUI follows new flows |
| `--set validate_inbound_headers=false` | Tolerate malformed requests |
| `--set http2=true` | Enable HTTP/2 |
| `--set websocket=true` | WebSocket support |
| `--set upstream_cert=false` | Don't MITM upstream TLS (pass-through) |
| `--set keep_host_header=true` | Preserve Host header in reverse mode |
| `--set connection_strategy=lazy` | Connect on demand (transparent mode) |
| `-h, --help` | Full option list |

Run `mitmproxy --help` or `mitmdump --help` for the complete generated list
(11 KB+ — saved in [`help.txt`](help.txt)).

## Certificate setup

First run generates a CA under `~/.mitmproxy/`:

```
mitmproxy-ca-cert.pem     # install in browser/system
mitmproxy-ca-cert.cer     # Windows/Android
mitmproxy-ca-cert.p12     # PKCS#12
mitmproxy-ca.pem          # CA + private key (keep secret)
```

Install:

```bash
# Debian system trust
sudo cp ~/.mitmproxy/mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/mitmproxy.crt
sudo update-ca-certificates

# Firefox: import mitmproxy-ca-cert.pem as authority
# Android: install as CA cert (user cert; apps may pin)
```

## Filter expressions

Used by `--set view_filter=...`, `mitmdump` filters, and scripts.

| Expression | Matches |
|------------|---------|
| `~u regex` | URL |
| `~m method` | HTTP method |
| `~h header` / `~hq` / `~hs` | Header (any/request/response) |
| `~b regex` / `~bq` / `~bs` | Body (any/request/response) |
| `~s` / `~q` | Response / request |
| `~c code` | Status code |
| `~d domain` | Domain |
| `~t content-type` | Content type |
| `~a` | Asset (js/css/image) |
| `~e` | Error |
| `!`, `&`, `\|` | Not / and / or |

Examples:

```
~u example\.com & ~m POST
~c 500
~t application/json
~b "password"
```

## Addon scripting

Minimal addon:

```python
# addons/headers.py
from mitmproxy import http

def request(flow: http.HTTPFlow) -> None:
    flow.request.headers["X-Test"] = "1"

def response(flow: http.HTTPFlow) -> None:
    if "example.com" in flow.request.pretty_host:
        flow.response.headers["X-Debug"] = "mitm"
        flow.response.text = flow.response.text.replace("old", "new")

def error(flow):
    print("error:", flow.error)
```

Load:

```bash
mitmdump -s addons/headers.py
```

Useful hooks: `request`, `response`, `requestheaders`, `responseheaders`,
`websocket_message`, `tls_clienthello`, `error`, `done`.

## Recipes

```bash
# Interactive TUI
mitmproxy -p 8080

# Web UI
mitmweb -p 8080 --web-port 8081

# Save all traffic to a flow file
mitmdump -w session.mitm

# Read a flow file and print URLs
mitmdump -nr session.mitm --flow-detail 1

# Replay a saved flow against the original server
mitmdump -n -r session.mitm --set replay_kill_extra=true

# Transparent proxy (needs iptables/nftables redirect)
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8080
mitmproxy --mode transparent --set block_global=false

# Reverse proxy in front of a backend
mitmdump --mode reverse:https://internal.example.com -p 8443

# Chain through an upstream proxy
mitmdump --mode upstream:http://corp-proxy:3128

# Modify responses with a script
mitmdump -s addons/rewrite.py --set flow_detail=2

# Only show JSON API calls
mitmdump --set view_filter='~t application/json'

# Dump URLs only
mitmdump -q --flow-detail 0 --set save_stream_file=-
```

## Transparent mode details

For HTTPS you must also redirect 443 and have the client trust the mitmproxy CA.
On a Linux host forwarding from another device:

```bash
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 8080
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j REDIRECT --to-port 8080
mitmproxy --mode transparent --listen-host 0.0.0.0 --set block_global=false
```

Do not leave an open proxy bound to `0.0.0.0` on an untrusted network.

## Gotchas

- Certificate pinning breaks interception (mobile apps, some clients).
- HTTP/3/QUIC is not fully MITM-able; disable QUIC in the browser.
- `block_global=false` is required for non-loopback clients.
- Transparent mode needs NAT redirect rules; reverse mode does not.
- `--set upstream_cert=false` passes TLS through untouched (no visibility).
- Flow files can get large; filter or rotate.
- WebSocket/HTTP2 support varies by version; check `--help` options.

## References

- Docs: https://docs.mitmproxy.org/stable/
- Addons: https://docs.mitmproxy.org/stable/addons/overview/
- Modes: https://docs.mitmproxy.org/stable/concepts/modes/
- Filters: https://docs.mitmproxy.org/stable/concepts/filters/
- Source: https://github.com/mitmproxy/mitmproxy

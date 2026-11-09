# websocat — WebSocket client, server, and protocol bridge

- **Version:** 1.14.1
- **Binary:** `/usr/local/bin/websocat`
- **Full help snapshot:** [`help.txt`](help.txt) (the short help; this build hides
  further flags behind `--help=long`)
- **Upstream:** https://github.com/vi/websocat
- **Docs:** https://github.com/vi/websocat#readme and `websocat --help=doc`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

websocat is to WebSockets what netcat is to TCP: it connects stdin/stdout to a
WebSocket endpoint, or exposes stdin/stdout as a WebSocket server. In advanced
mode it takes **two addresses** — a connector and a listener, or two connectors —
and copies bytes between them, so it also works as a protocol bridge
(WebSocket to TCP, WebSocket to a subprocess, TCP to a file).

Unlike a browser, it gives you raw frames: you choose text or binary messages,
set handshake headers, force a subprotocol, ping on a timer, and inspect raw
payloads. That makes it the quickest way to exercise a WebSocket API that a
browser devtools session or an intercepting proxy cannot easily replay.

## Use cases

| Situation | Why websocat |
|-----------|--------------|
| A WebSocket endpoint needs manual probing | Send arbitrary frames from a shell, one line per message |
| Auth or Origin checks must be tested | Set `Origin`, `Cookie`, `Authorization`, or a subprotocol directly |
| A client library masks a protocol bug | Bypass the library and speak to the socket directly |
| Reproducing a captured session | Replay framed payloads from a file with `-1`/`-t` |
| Building a throwaway lab server | `-s` gives a mirror or command-backed server in one line |
| Bridging WebSocket to a local service | `ws-l:` plus `tcp:` in advanced mode |
| Measuring latency | `--ping-interval` plus `--print-ping-rtts` |

## Requirements & install layout

- Single static Rust binary at `/usr/local/bin/websocat`; no runtime
  dependencies and no configuration file.
- No root required for client use. Server mode binds a TCP port, so pick a high
  port and bind loopback explicitly unless you intend to expose it.
- Environment: `-e` exports `WEBSOCAT_URI` and `WEBSOCAT_CLIENT` to
  `exec:`/`cmd:`/`sh-c:` children; `--basic-auth` also reads
  `WEBSOCAT_BASIC_AUTH`. No other config path is used.
- TLS server mode needs a PKCS#12 archive (`--pkcs12-der`), not PEM files.
- Message size defaults to 65536 bytes (`-B`); large frames are split unless
  `-S` is set.

## Basic syntax

```bash
websocat ws://127.0.0.1:8080/ws                  # interactive client, stdin <-> ws
websocat -s 127.0.0.1:8080                        # simple server (mirror by default)
websocat --binary ws-l:127.0.0.1:8080 tcp:127.0.0.1:5678   # bridge
echo '{"op":"ping"}' | websocat -1 -t ws://127.0.0.1:8080/ws  # one message
websocat --help=doc                               # full manual with examples
```

The usage line is `websocat [FLAGS] [OPTIONS] <addr1> <addr2>`. One address is
simple mode; two addresses are advanced mode.

## Flag reference

Flags below are exactly those in the installed short help. `--help=long` and
`--help=doc` expose additional options that are not listed here.

### Connection and addressing

| Flag | Meaning |
|------|---------|
| `<addr1>` | Simple mode: WebSocket URL. Advanced mode: first address (connector or listener) |
| `<addr2>` | Advanced mode: second address; a listener here accepts one connection |
| `-s, --server-mode` | Simple server mode; argument is a port or `addr:port` |
| `--oneshot` | Serve only once, then exit |
| `--conncap <n>` | Maximum simultaneous connections in listening mode |
| `--restrict-uri <uri>` | When serving, accept only this URI, e.g. `/ws` |
| `-F, --static-file <uri:type:path>` | Serve a static file for non-WebSocket requests; repeatable |
| `--pkcs12-der <file>` | PKCS#12 archive (cert + key) for TLS server mode |
| `--pkcs12-passwd <pass>` | Password for the PKCS#12 archive |

### TLS and proxying

| Flag | Meaning |
|------|---------|
| `-k, --insecure` | Accept invalid certificates and hostnames on connect |
| `--socks5 <addr:port>` | Route through a SOCKS5 proxy, e.g. `127.0.0.1:9050` |

### HTTP handshake

| Flag | Meaning |
|------|---------|
| `-H, --header <name: value>` | Add a request header; repeatable. Quote as `-H='X: y'` |
| `--server-header <name: value>` | Add a header to the upgrade reply; repeatable |
| `--header-to-env <name>` | Forward an incoming request header to `H_*` for `exec:` addresses |
| `--origin <origin>` | Set the `Origin` request header |
| `--ua <useragent>` | Set the `User-Agent` request header |
| `--protocol <proto>` | Set `Sec-WebSocket-Protocol` when connecting |
| `--server-protocol <proto>` | Force `Sec-WebSocket-Protocol` when accepting |
| `--websocket-version <v>` | Override `Sec-WebSocket-Version` |
| `--basic-auth <base64>` | Add an `Authorization: Basic` header from a base64 blob |
| `--basic-auth-file <file>` | Same, with the base64 blob read from a file |

### Framing and message handling

| Flag | Meaning |
|------|---------|
| `-t, --text` | Send messages as WebSocket text frames |
| `-b, --binary` | Send messages as WebSocket binary frames |
| `--base64` | Base64-encode incoming binary frames to one line; decode outgoing when a binary prefix is configured |
| `-0, --null-terminated` | Use NUL instead of newline for line mode |
| `-S, --strict` | Drop over-long messages and incomplete lines instead of splitting |
| `-B, --buffer-size <bytes>` | Maximum message size in bytes (default 65536) |
| `--jsonrpc` | Wrap typed lines as JSON-RPC 2.0 calls: first word is the method |
| `--no-fixups` | Disable automatic protocol fixups; see the full manual |
| `-1, --one-message` | Send and/or receive only one message; combine with `-n`, `-u`, `-U` |
| `-n, --no-close` | Do not send a Close frame on EOF |

### Direction and lifecycle

| Flag | Meaning |
|------|---------|
| `-u, --unidirectional` | Copy data in one direction only |
| `-U, --unidirectional-reverse` | Copy in the other direction only; with `-u`, both are inhibited |
| `-E, --exit-on-eof` | Close a direction when the other side reaches EOF |
| `--max-messages <n>` | Maximum messages copied in one direction |
| `--max-messages-rev <n>` | Maximum messages copied in the other direction |
| `--close-status-code <code>` | Send a Close frame with this status code |
| `--close-reason <text>` | Reason text for the Close frame; needs `--close-status-code` |
| `-p, --preamble <text>` | Prepend text to copied data; repeatable |
| `-P, --preamble-reverse <text>` | Prepend text in the reverse direction; repeatable |

### Keepalive and diagnostics

| Flag | Meaning |
|------|---------|
| `--ping-interval <secs>` | Send a WebSocket ping every N seconds |
| `--ping-timeout <secs>` | Drop the connection if no pong arrives within N seconds |
| `--print-ping-rtts` | Print measured round-trip time to stderr after each pong |
| `-e, --set-environment` | Export `WEBSOCAT_URI`/`WEBSOCAT_CLIENT` to child commands |
| `-v` | Increase verbosity (repeatable) |
| `-q` | Suppress diagnostics except startup errors |
| `-h, --help <short\|long\|doc>` | Print short, long, or documented help |

### Address types

| Address | Meaning |
|---------|---------|
| `ws://`, `wss://` | WebSocket client (plain / TLS) |
| `ws-listen:`, `wss-listen:` | WebSocket server (plain / TLS) |
| `tcp:`, `tcp-listen:` | TCP client / listener |
| `ssl-listen:` | TLS listener |
| `sh-c:`, `cmd:` | Run a command line via `sh -c` (`cmd /C` on Windows) |
| `readfile:`, `writefile:`, `appendfile:` | Synchronous file read / truncate-write / append |
| `udp:`, `udp-listen:` | UDP peer / listener |
| `-` | Console stdin/stdout |
| `mirror:` | Echo output back to input |
| `literalreply:`, `literal:`, `random:` | Fixed reply, fixed output, random bytes |

### Overlays

| Overlay | Meaning |
|---------|---------|
| `broadcast:` | Reuse one connection for many clients, replying to all |
| `autoreconnect:` | Re-establish the underlying connection on error or EOF |
| `foreachmsg:` | Execute something for each incoming message |
| `log:` | Log each buffer passing through the connector |

## Recipes

```bash
# Interactive session: type a line, send a text frame, see replies
websocat ws://127.0.0.1:8080/ws

# Single request/response, safe for scripts
printf '{"op":"subscribe","channel":"orders"}\n' | websocat -1 -t ws://127.0.0.1:8080/ws

# Test Origin and cookie enforcement on an authorized target
websocat -H='Origin: https://app.example.com' -H='Cookie: session=LAB_VALUE' \
  ws://127.0.0.1:8080/ws

# Force a subprotocol and bearer token
websocat --protocol graphql-ws -H='Authorization: Bearer LAB_TOKEN' \
  wss://127.0.0.1:8443/graphql

# Accept a self-signed lab certificate
websocat -k wss://127.0.0.1:8443/ws

# Local throwaway server on loopback only
websocat -s 127.0.0.1:8080

# Bridge a browser-facing WebSocket to a local TCP service
websocat --binary ws-l:127.0.0.1:8080 tcp:127.0.0.1:5678

# Measure round-trip time while holding a session open
websocat --ping-interval 5 --print-ping-rtts -v ws://127.0.0.1:8080/ws

# Route through Tor (or any SOCKS5 proxy) for authorized external testing
websocat --socks5 127.0.0.1:9050 wss://example.com/ws
```

## Output & pipelines

Payload bytes go to stdout; diagnostics, verbosity, and ping RTTs go to stderr,
so `-q` plus a redirect gives clean output. Line mode makes each message one
line, which lets `jq`, `grep`, and `awk` consume JSON frames:

```bash
websocat -q ws://127.0.0.1:8080/ws | jq -c .
```

For binary protocols, add `--base64` to keep the stream line-oriented, or drop
to `--binary` and write raw frames to a file:

```bash
websocat -b ws://127.0.0.1:8080/ws > frames.bin
```

Handy handoffs: capture through `mitmproxy`/Burp when you need history, use
`websocat` when you need to replay or fuzz a frame sequence, and use
`--jsonrpc` when the peer expects JSON-RPC method framing.

## Gotchas & operational security

- `-H` can swallow following positional arguments. Put headers last or use
  `-H='Name: value'`, as the help warns.
- `-s <port>` binds where you tell it to. Omitting the address may expose the
  server beyond loopback; always write `127.0.0.1:PORT` in a lab.
- `sh-c:`, `cmd:`, and `foreachmsg:` execute commands. With `-e`, request data
  reaches the child environment through `H_*` variables — never expose such a
  server to an untrusted network.
- `-k` disables certificate and hostname checks; use it only against lab
  endpoints.
- Only SOCKS5 proxying is exposed by the short help; there is no HTTP CONNECT
  flag listed here.
- Default `-B 65536` splits larger outgoing messages; `-S` drops them instead.
  Either can look like application truncation.
- `-1` limits messages, while `--oneshot` limits connections. Mixing them up
  produces servers that stay up or clients that exit early.
- A clean exit does not prove the peer accepted the payload; check application
  responses, not just the exit code.

## Legal & safety

Use websocat only against WebSocket services you own or have written
authorization to test. Sending frames to third-party endpoints, bypassing
Origin checks, or replaying captured sessions against production systems can
violate terms of service and law. Running a server with `cmd:`/`sh-c:`
addresses is remote command execution by design; keep it on loopback, in a lab,
and never behind an untrusted interface.

## Related tools

- [`../burp/`](../burp/README.md) — intercept and replay WebSocket frames with
  history
- [`../mitmproxy/`](../mitmproxy/README.md) — scriptable interception when a
  frame sequence must be transformed
- [`../zap/`](../zap/README.md) — WebSocket fuzzing inside a scanning workflow
- [`../httpx/`](../httpx/README.md) — find live HTTP services before probing
  their upgrade endpoints

## References

- Installed short help: [`help.txt`](help.txt); extended help via
  `websocat --help=long` and `websocat --help=doc`
- Upstream repository and manual: https://github.com/vi/websocat
- Release notes: https://github.com/vi/websocat/releases
- RFC 6455 (WebSocket protocol): https://www.rfc-editor.org/rfc/rfc6455

# websocat — command scenarios for deep research

Installed build: websocat 1.14.1. Flags below come from `help.txt`; this build
hides further options behind `--help=long` and `--help=doc`.

## How to invoke

```text
@deep-research For websocat 1.14.1, how do WebSocket handshakes differ from
HTTP/2 and SSE in ways that break naive proxies, which flags best expose Origin,
cookie, and subprotocol enforcement, and how should binary framing, ping RTT,
and message-size limits be tested without crashing the peer? Cite RFC 6455,
websocat docs, and field reports; return a test matrix and a safe replay recipe.
```

---

## Scenario 1 — connect and inspect the handshake

| # | Command | How to use |
|---|---------|------------|
| 1 | `websocat -v ws://127.0.0.1:8080/ws` | Verbose connect; handshake detail on stderr |
| 2 | `websocat -q ws://127.0.0.1:8080/ws` | Quiet interactive session |
| 3 | `websocat --help=long` | Full flag surface |
| 4 | `websocat --help=doc` | Manual with address types and examples |
| 5 | `websocat --websocket-version 13 ws://127.0.0.1:8080/ws` | Pin the version header |

**Research questions**
- What does `-v` reveal about the upgrade response, and what does it omit?
- When is overriding `Sec-WebSocket-Version` useful, and what does the server
  do with an unsupported value?

**Sources:** RFC 6455, websocat docs, handshake output.
**Output:** a confirmed upgrade and a recorded handshake.

---

## Scenario 2 — headers, origin, and authentication

| # | Command | How to use |
|---|---------|------------|
| 6 | `websocat -H='Origin: https://app.example.com' ws://127.0.0.1:8080/ws` | Origin header |
| 7 | `websocat --origin https://app.example.com --ua 'Mozilla/5.0' ws://127.0.0.1:8080/ws` | Origin and UA helpers |
| 8 | `websocat --protocol graphql-ws ws://127.0.0.1:8080/graphql` | Force a subprotocol |
| 9 | `websocat --basic-auth "$(printf 'user:pass' \| base64 -w0)" ws://127.0.0.1:8080/ws` | Basic auth |
| 10 | `websocat --header-to-env Cookie sh-c:'env \| grep ^H_'` | Forward a header to a child |

**Research questions**
- Which of Origin, Cookie, and subprotocol checks are enforced server-side, and
  which are browser-only conventions?
- Does `--basic-auth-file` avoid the shell-history exposure of `--basic-auth`?

**Sources:** OWASP WebSocket guidance, RFC 6455, output.
**Output:** an auth/header test matrix.

---

## Scenario 3 — framing and message discipline

| # | Command | How to use |
|---|---------|------------|
| 11 | `printf '{"op":"ping"}\n' \| websocat -1 -t ws://127.0.0.1:8080/ws` | One text message |
| 12 | `head -c 128 /dev/urandom \| websocat -1 -b ws://127.0.0.1:8080/ws` | One binary message |
| 13 | `websocat --base64 -1 ws://127.0.0.1:8080/ws < payload.b64` | Base64 line mode |
| 14 | `websocat -B 1048576 -t ws://127.0.0.1:8080/ws` | Raise the message limit |
| 15 | `websocat -S -B 4096 ws://127.0.0.1:8080/ws` | Drop over-long messages |
| 16 | `websocat --jsonrpc ws://127.0.0.1:8080/rpc` | Type JSON-RPC lines |

**Research questions**
- How does `-B` interact with `-S`, and which behaviour looks like application
  truncation?
- Does `--jsonrpc` wrap parameters in an array, and what does the peer accept?

**Sources:** websocat docs, server responses, RFC 6455 framing.
**Output:** a framing profile and reproducible payloads.

---

## Scenario 4 — serving and bridging

| # | Command | How to use |
|---|---------|------------|
| 17 | `websocat -s 127.0.0.1:8080` | Loopback mirror server |
| 18 | `websocat --oneshot -s 127.0.0.1:8080` | Serve one connection |
| 19 | `websocat --conncap 5 -s 127.0.0.1:8080` | Cap concurrent clients |
| 20 | `websocat --restrict-uri /ws -F /index.html:text/html:index.html -s 127.0.0.1:8080` | Restrict upgrade path, serve a file |
| 21 | `websocat --binary ws-l:127.0.0.1:8080 tcp:127.0.0.1:5678` | WebSocket-to-TCP bridge |
| 22 | `websocat ws-l:127.0.0.1:8080 mirror:` | Explicit mirror address |

**Research questions**
- Which address types listen, and which connect, in advanced mode?
- What happens to non-WebSocket requests when `-F` is not supplied?

**Sources:** `--help=doc`, upstream examples, local test output.
**Output:** a lab server or bridge with a documented exposure boundary.

---

## Scenario 5 — TLS, SOCKS, keepalive, and close semantics

| # | Command | How to use |
|---|---------|------------|
| 23 | `websocat -k wss://127.0.0.1:8443/ws` | Accept a self-signed lab cert |
| 24 | `websocat --socks5 127.0.0.1:9050 wss://example.com/ws` | Route through SOCKS5 |
| 25 | `websocat --ping-interval 5 --ping-timeout 15 --print-ping-rtts -v ws://127.0.0.1:8080/ws` | Keepalive and RTT |
| 26 | `websocat -s 127.0.0.1:8443 --pkcs12-der server.p12 --pkcs12-passwd "$P12_PASS"` | TLS server |
| 27 | `websocat --close-status-code 1000 --close-reason done ws://127.0.0.1:8080/ws` | Controlled close |

**Research questions**
- Which TLS failures are masked by `-k`, and how is verification restored?
- How do ping timeouts surface to the application, and what close codes are
  observable at the peer?

**Sources:** RFC 6455 close semantics, TLS docs, output.
**Output:** a transport test record with measured RTT and close frames.

---

## Scenario 6 — replay, capture, and evidence

| # | Command | How to use |
|---|---------|------------|
| 28 | `websocat -q ws://127.0.0.1:8080/ws \| jq -c .` | Line-oriented JSON capture |
| 29 | `websocat -b ws://127.0.0.1:8080/ws > frames.bin` | Raw binary capture |
| 30 | `websocat -u -1 ws://127.0.0.1:8080/ws < payload.bin > reply.bin` | One-way send with reply |
| 31 | `websocat --max-messages 10 --max-messages-rev 10 ws://127.0.0.1:8080/ws` | Bounded session |
| 32 | `websocat -p 'header-line\n' -1 ws://127.0.0.1:8080/ws` | Prepend a preamble |
| 33 | `sha256sum frames.bin reply.bin` | Hash captured evidence |

**Research questions**
- Which direction does `-u` inhibit, and how is the opposite direction isolated?
- What provenance is needed before a captured frame sequence can be replayed?

**Sources:** websocat docs, capture files, hashes.
**Output:** a replayable, hashed frame set.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Environment | `-e --header-to-env` | 2 |
| Lifecycle | `-E -1 --oneshot -n --max-messages --max-messages-rev` | 3, 4, 6 |
| Format | `--jsonrpc -0 -S --base64 -B` | 3 |
| Server | `-s --conncap --restrict-uri -F --pkcs12-der --pkcs12-passwd` | 4, 5 |
| TLS/proxy | `-k --socks5` | 5 |
| Direction | `-u -U` | 6 |
| Framing | `-b -t` | 3 |
| HTTP | `-H --server-header --origin --ua --protocol --server-protocol --websocket-version --basic-auth --basic-auth-file` | 2 |
| Keepalive | `--ping-interval --ping-timeout --print-ping-rtts` | 5 |
| Close | `--close-status-code --close-reason` | 5 |
| Diagnostics | `-v -q -h` | 1 |
| Data shaping | `-p -P` | 6 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — handshake | 5 |
| 2 — headers/auth | 5 |
| 3 — framing | 6 |
| 4 — serving/bridging | 6 |
| 5 — TLS/keepalive | 5 |
| 6 — replay/evidence | 6 |

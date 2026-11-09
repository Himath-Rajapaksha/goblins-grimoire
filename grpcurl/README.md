# grpcurl — gRPC command-line client

- **Version:** development build (reports `dev build <no version set>`)
- **Binary:** `/usr/local/bin/grpcurl` (Go)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/fullstorydev/grpcurl

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

grpcurl speaks gRPC from the command line. It can list services and methods,
describe message and enum schemas, and invoke unary or streaming RPCs with JSON
or protobuf-text request bodies. It discovers schemas through the server's
reflection API when available, or from local `.proto` files or a compiled
`FileDescriptorSet` when reflection is disabled.

## Use cases

| Situation | Why grpcurl |
|-----------|-------------|
| gRPC service inventory | `list` and `describe` without code generation |
| API testing | Invoke methods with JSON bodies |
| Reflection exposure check | See whether schemas leak |
| Disabled reflection | Use local proto or protoset files |
| Streaming tests | Client/server/bidirectional streaming |
| Auth testing | Headers, TLS, mTLS, ALTS, Unix sockets |
| Schema extraction | Write `.proto` or protoset files from a live server |

## Requirements & install layout

- Static Go binary at `/usr/local/bin/grpcurl`; no runtime dependencies.
- No config file; all behaviour is flags.
- TLS is the default; `-plaintext` is required for cleartext HTTP/2.
- Reflection must be enabled on the server, or you must supply `-proto`,
  `-protoset`, or `-import-path`.
- `-d @` reads request JSON from stdin, which keeps large or secret payloads
  out of shell history.

## Basic syntax

```bash
# list services (reflection)
grpcurl -plaintext 127.0.0.1:50051 list

# describe a service or message
grpcurl -plaintext 127.0.0.1:50051 describe my.Service

# invoke a unary method
grpcurl -plaintext -d '{"name":"alice"}' 127.0.0.1:50051 my.Service/GetUser

# TLS with a custom CA
grpcurl -cacert ca.pem -d '{}' api.example.com:443 my.Service/Ping

# local proto files when reflection is off
grpcurl -plaintext -import-path ./proto -proto service.proto \
        -d '{}' 127.0.0.1:50051 my.Service/Ping
```

## Flag reference

### Addressing and transport

| Flag | Meaning |
|------|---------|
| `address` | `host:port`; bracket IPv6; Unix path with `-unix` |
| `-plaintext` | Cleartext HTTP/2, no TLS |
| `-unix` | Address is a Unix domain socket path |
| `-authority <name>` | `:authority` value; also SNI/verification name |
| `-servername <name>` | TLS SNI override (prefer `-authority`) |
| `-connect-timeout <sec>` | Connection timeout (default 10) |
| `-keepalive-time <sec>` | Keepalive probe interval |
| `-max-time <sec>` | Overall RPC deadline |
| `-max-msg-sz <bytes>` | Max response message size (default 4 MiB) |
| `-user-agent <ua>` | User-Agent suffix |

### TLS, mTLS, and ALTS

| Flag | Meaning |
|------|---------|
| `-cacert <file>` | Trusted roots for server verification |
| `-cert <file>` | Client certificate (needs `-key`) |
| `-key <file>` | Client private key (needs `-cert`) |
| `-insecure` | Skip certificate and hostname verification |
| `-alts` | Use Google ALTS |
| `-alts-handshaker-service <addr>` | ALTS handshaker service |
| `-alts-target-service-account <email>` | Expected peer service account (repeatable) |

### Schema sources

| Flag | Meaning |
|------|---------|
| `-use-reflection` | Use server reflection (default when no proto/protoset) |
| `-proto <file>` | Proto source file (repeatable) |
| `-import-path <dir>` | Import root for `-proto` (repeatable) |
| `-protoset <file>` | Encoded `FileDescriptorSet` (repeatable) |
| `-proto-out-dir <dir>` | Write resolved `.proto` files |
| `-protoset-out <file>` | Write a `FileDescriptorSet` |

### Requests and output

| Flag | Meaning |
|------|---------|
| `-d <data>` | Request body; `@` reads stdin |
| `-format <json\|text>` | Request format (default `json`) |
| `-allow-unknown-fields` | Ignore unknown JSON fields |
| `-emit-defaults` | Emit default values in JSON responses |
| `-format-error` | Format error responses with `-format` |
| `-msg-template` | Show an input template when describing |
| `-H <header>` | Header for reflection and RPC |
| `-rpc-header <header>` | Header only for the RPC |
| `-reflect-header <header>` | Header only for reflection |
| `-expand-headers` | Expand `${ENV}` in headers |
| `-v` / `-vv` | Verbose / very verbose (timing) |
| `-version` | Print version |

### Verbs

| Verb | Meaning |
|------|---------|
| `list` | List services, or methods of a named service |
| `describe` | Describe a service, method, message, or enum |
| *(none)* | Invoke the named method |

## Recipes

```bash
# 1. Inventory without reflection assumptions
grpcurl -plaintext 127.0.0.1:50051 list

# 2. Methods of one service
grpcurl -plaintext 127.0.0.1:50051 list my.Service

# 3. Describe a message and show a request template
grpcurl -plaintext -msg-template 127.0.0.1:50051 describe my.Request

# 4. Unary call with a JSON body
grpcurl -plaintext -d '{"id":1}' 127.0.0.1:50051 my.Service/Get

# 5. Body from stdin (keeps secrets out of history)
grpcurl -plaintext -d @ 127.0.0.1:50051 my.Service/Login < body.json

# 6. Auth headers with environment expansion
grpcurl -plaintext -expand-headers -H 'Authorization: Bearer ${TOKEN}' \
        -d '{}' 127.0.0.1:50051 my.Service/WhoAmI

# 7. mTLS
grpcurl -cacert ca.pem -cert client.pem -key client.key \
        -d '{}' api.example.com:443 my.Service/Ping

# 8. Reflection disabled: use local protos
grpcurl -plaintext -import-path ./proto -proto svc.proto \
        -d '{}' 127.0.0.1:50051 my.Service/Ping

# 9. Extract the schema
grpcurl -plaintext -protoset-out schema.protoset 127.0.0.1:50051 list
grpcurl -plaintext -proto-out-dir ./recovered 127.0.0.1:50051 describe my.Service

# 10. Server-streaming call
grpcurl -plaintext -d '{"topic":"events"}' 127.0.0.1:50051 my.Service/Subscribe
```

## Output & pipelines

- `list`/`describe` print text; `-protoset-out` and `-proto-out-dir` produce
  reusable schema files.
- `-d @` plus `jq` builds requests dynamically.
- `-format json` is default; `-format text` accepts protobuf text, with records
  separated by ASCII `0x1E` for streaming.
- `-v` includes response headers and trailers, useful for evidence.
- Pipe JSON responses into `jq`; `-emit-defaults` makes schemas explicit.

## Gotchas & operational security

- Reflection is frequently disabled in production; expect `list` to fail and
  fall back to local protos.
- `-plaintext` against a TLS port fails with a confusing handshake error, and
  vice versa.
- `-insecure` disables verification; never use it outside a lab.
- Streaming requests concatenate multiple JSON messages; malformed separators
  produce partial calls.
- `-d @` still exposes the body to process inspection on a shared host.
- Header values on the command line land in shell history; use
  `-expand-headers` with environment variables.
- Invoking mutating RPCs can change state; treat discovery and invocation as
  separate authorization decisions.
- The development build reports no version; record the binary hash instead.

## Legal & safety

Only call gRPC services you are authorized to test. Invocation can create,
modify, or delete data; prefer read-only methods first, and never use
`-insecure` against third parties.

## Related tools

- [`websocat/`](../websocat/README.md) — WebSocket counterpart
- `grpcui` — browser UI for reflection-enabled servers
- `protoc` — compile and decode protobuf
- [`ffuf/`](https://github.com/ffuf/ffuf) — HTTP fuzzing (not gRPC-native)

## References

- https://github.com/fullstorydev/grpcurl
- https://grpc.io/docs/

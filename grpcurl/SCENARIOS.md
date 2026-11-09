# grpcurl — command scenarios for deep research

Installed build: grpcurl development build (no version reported). Flags come
from `help.txt`.

## How to invoke

```text
@deep-research For a current grpcurl dev build, how should gRPC services be
inventoried when reflection is disabled, which reflection/describe calls leak
the most schema detail, and how are streaming and mTLS calls tested safely?
Cite grpcurl docs, gRPC reflection spec, and field reports; return a discovery
decision tree and a safe invocation matrix.
```

---

## Scenario 1 — service inventory

| # | Command | How to use |
|---|---------|------------|
| 1 | `grpcurl -plaintext 127.0.0.1:50051 list` | All services |
| 2 | `grpcurl -plaintext 127.0.0.1:50051 list my.Service` | Methods |
| 3 | `grpcurl -plaintext 127.0.0.1:50051 describe my.Service` | Descriptor |
| 4 | `grpcurl -plaintext -msg-template 127.0.0.1:50051 describe my.Request` | Template |
| 5 | `grpcurl -plaintext -v 127.0.0.1:50051 list` | Verbose |

**Research questions**
- What does a failed reflection call reveal about the server?
- Which descriptors expose internal types or deprecated methods?

**Sources:** gRPC reflection spec, output.
**Output:** a service and method inventory.

---

## Scenario 2 — unary invocation

| # | Command | How to use |
|---|---------|------------|
| 6 | `grpcurl -plaintext -d '{"id":1}' 127.0.0.1:50051 my.Service/Get` | Basic call |
| 7 | `grpcurl -plaintext -d @ 127.0.0.1:50051 my.Service/Get < body.json` | Stdin body |
| 8 | `grpcurl -plaintext -emit-defaults -d '{}' 127.0.0.1:50051 my.Service/Get` | Defaults shown |
| 9 | `grpcurl -plaintext -allow-unknown-fields -d '{"x":1}' ...` | Lenient parse |
| 10 | `grpcurl -plaintext -format text -d 'id: 1' ...` | Protobuf text |

**Research questions**
- How are JSON field names mapped to protobuf fields (camelCase, oneof)?
- What changes when `-emit-defaults` is set?

**Sources:** protobuf JSON mapping docs, output.
**Output:** validated request/response pairs.

---

## Scenario 3 — auth and transport

| # | Command | How to use |
|---|---------|------------|
| 11 | `grpcurl -plaintext -H 'Authorization: Bearer x' -d '{}' ...` | Header auth |
| 12 | `grpcurl -plaintext -expand-headers -H 'Authorization: Bearer ${TOKEN}' ...` | Env expansion |
| 13 | `grpcurl -cacert ca.pem -d '{}' api.example.com:443 my.Service/Ping` | Custom CA |
| 14 | `grpcurl -cacert ca.pem -cert c.pem -key c.key ...` | mTLS |
| 15 | `grpcurl -plaintext -unix /run/svc.sock list` | Unix socket |

**Research questions**
- Which headers go to reflection versus the RPC?
- How should client keys be supplied without shell history exposure?

**Sources:** grpcurl docs, gRPC auth docs.
**Output:** authenticated coverage of the service.

---

## Scenario 4 — schema extraction without reflection

| # | Command | How to use |
|---|---------|------------|
| 16 | `grpcurl -plaintext -import-path ./proto -proto svc.proto list` | Local protos |
| 17 | `grpcurl -plaintext -protoset schema.protoset list` | Compiled set |
| 18 | `grpcurl -plaintext -protoset-out out.protoset 127.0.0.1:50051 list` | Extract set |
| 19 | `grpcurl -plaintext -proto-out-dir ./recovered 127.0.0.1:50051 describe my.Service` | Extract proto |
| 20 | `protoc --decode_raw < response.bin` | Decode unknown bytes |

**Research questions**
- How are imports resolved when only some protos are provided?
- What does `-protoset-out` include beyond the listed service?

**Sources:** protobuf docs, grpcurl source, output.
**Output:** recovered schemas for offline analysis.

---

## Scenario 5 — streaming and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `grpcurl -plaintext -d '{"topic":"x"}' 127.0.0.1:50051 my.Service/Subscribe` | Server stream |
| 22 | `grpcurl -plaintext -d @ ... < stream.jsonl` | Client stream |
| 23 | `grpcurl -plaintext -vv -d '{}' ... > call.log 2>&1` | Timing evidence |
| 24 | `grep -E 'Response|Trailer' call.log` | Extract metadata |
| 25 | `sha256sum call.log out.protoset recovered/*.proto` | Hash evidence |

**Research questions**
- How are multiple streaming messages delimited in JSON and text formats?
- What evidence proves the call succeeded without side effects?

**Sources:** grpcurl docs, gRPC streaming spec, logs.
**Output:** a reproducible streaming test and hashed evidence.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Transport | `-plaintext -unix -authority -servername -connect-timeout -keepalive-time -max-time -max-msg-sz` | 3 |
| TLS | `-cacert -cert -key -insecure -alts*` | 3 |
| Schema | `-use-reflection -proto -import-path -protoset -proto-out-dir -protoset-out` | 4 |
| Request | `-d -format -allow-unknown-fields -emit-defaults -format-error -msg-template` | 2, 5 |
| Headers | `-H -rpc-header -reflect-header -expand-headers` | 3 |
| Output | `-v -vv -version` | 1, 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — inventory | 5 |
| 2 — unary | 5 |
| 3 — auth/transport | 5 |
| 4 — schema extraction | 5 |
| 5 — streaming/evidence | 5 |

# smuggler — HTTP request smuggling detector

- **Version:** packaged `smuggler.py` (Python 3)
- **Binary:** `/usr/local/bin/smuggler` (wrapper)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/defparam/smuggler

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

smuggler tests for HTTP request smuggling by sending crafted requests with
conflicting `Content-Length` and `Transfer-Encoding` headers (CL.TE, TE.CL,
TE.TE, and variants) and timing or response differences to detect
desynchronisation between a front-end proxy and back-end server. It uses a
payload configuration file and can exit on the first confirmed finding.

## Use cases

| Situation | Why smuggler |
|-----------|--------------|
| Proxy/backend desync | Detect CL.TE / TE.CL behaviour |
| CDN and load-balancer review | Identify inconsistent framing |
| Bug bounty | High-impact request smuggling candidates |
| Change validation | Confirm a fix after a patch |
| Config tuning | Custom payloads for unusual stacks |

## Requirements & install layout

- Python 3 script with wrapper at `/usr/local/bin/smuggler`.
- No API keys.
- Default payload set ships with the tool; `-c` loads a custom config.
- Needs a target endpoint that accepts POST (default method).
- Works best against HTTP/1.1 front ends; HTTP/2-only edges need different
  testing.

## Basic syntax

```bash
# test a URL
smuggler -u https://target/

# test a specific endpoint with a vhost
smuggler -u https://target/login -v internal.example.com

# stop at the first finding
smuggler -u https://target/ -x

# use GET and a custom timeout
smuggler -u https://target/ -m GET -t 10

# quiet, log to file
smuggler -u https://target/ -q -l smuggler.log

# custom payload config
smuggler -u https://target/ -c payloads.conf
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-u, --url <url>` | Target URL with endpoint |
| `-v, --vhost <host>` | Virtual host header value |
| `-x, --exit_early` | Exit on first finding |
| `-m, --method <method>` | HTTP method (default `POST`) |
| `-l, --log <file>` | Log file |
| `-q, --quiet` | Log only issues found |
| `-t, --timeout <sec>` | Socket timeout (default 5) |
| `--no-color` | Suppress ANSI colour |
| `-c, --configfile <file>` | Payload configuration file |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Basic POST test
smuggler -u https://target/ -q

# 2. Endpoint behind a vhost
smuggler -u https://target/api -v api.internal.example.com -q

# 3. Stop at the first confirmed issue
smuggler -u https://target/ -x -q

# 4. GET-based test
smuggler -u https://target/ -m GET -q

# 5. Longer timeout for slow edges
smuggler -u https://target/ -t 15 -q

# 6. Log everything colourlessly
smuggler -u https://target/ --no-color -l smuggle.log

# 7. Custom payloads for an unusual stack
smuggler -u https://target/ -c custom.conf -q

# 8. Verify a finding manually with a raw socket
printf 'POST / HTTP/1.1\r\nHost: target\r\nContent-Length: 6\r\nTransfer-Encoding: chunked\r\n\r\n0\r\n\r\nG' | nc target 80

# 9. Re-test after a patch
smuggler -u https://target/ -q -l retest.log
```

## Output & pipelines

- Console lines mark each payload as `ok`, `potential`, or `issue`.
- `-l` writes a log; `-q` keeps only findings.
- Findings require manual confirmation with a proxy or raw sockets before
  reporting.
- Positive results feed Burp Turbo Intruder, `h2csmuggler`, and manual
  desync exploitation.

## Gotchas & operational security

- Timing-based detection is noisy; network jitter causes false positives.
- A single confirmed issue still needs a reproducible PoC — do not report on
  scanner output alone.
- Smuggling tests can poison shared caches and affect other users. Coordinate
  timing and use a lab replica where possible.
- HTTP/2 front ends need HTTP/2-specific tests; this tool is HTTP/1.1-focused.
- `-x` hides additional findings after the first.
- Custom payloads can be destructive; review before loading.
- The tool may leave half-open connections; check the backend after testing.

## Legal & safety

Request smuggling can corrupt other users' requests and poison caches — a real
availability and integrity risk. Test only with explicit written authorization,
during agreed windows, and stop immediately if other traffic is affected.

## Related tools

- [`corsy/`](../corsy/README.md) — CORS misconfiguration
- [`x8/`](../x8/README.md) — parameter discovery
- [`wfuzz/`](../wfuzz/README.md) — request fuzzing
- [`testssl/`](../testssl/README.md), [`sslyze/`](../sslyze/README.md) — TLS layer

## References

- https://github.com/defparam/smuggler
- https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn

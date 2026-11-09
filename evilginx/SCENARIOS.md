# evilginx — command scenarios for deep research

Installed build: evilginx 3.3.0. Command-line flags come from `help.txt`; the
interactive console is upstream-documented and labelled `(console)` below.

## How to invoke

```text
@deep-research For evilginx 3.3.0, how do phishlet proxying and session-cookie
capture interact with modern MFA, conditional access, and token-binding
controls, which console workflows produce the cleanest evidence for a red-team
report, and how should the proxy and captured sessions be contained and torn
down? Cite the evilginx docs, MITRE ATT&CK T1557/T1539, and defender writeups;
return an authorization gate, a lab workflow, and a teardown checklist.
```

---

## Scenario 1 — lab bootstrap and preflight

| # | Command | How to use |
|---|---------|------------|
| 1 | `evilginx -v` | Confirm 3.3.0 |
| 2 | `mkdir -p ./lab/conf ./lab/phishlets ./lab/redirectors` | Isolated lab layout |
| 3 | `dig +short login.lab.example.test` | Verify DNS points at the lab proxy |
| 4 | `ss -ltnp \| grep -E ':(80\|443)\b'` | Check port availability |
| 5 | `evilginx -developer -c ./lab/conf -p ./lab/phishlets -t ./lab/redirectors` | Start lab proxy |

**Research questions**
- Which privileges are needed to bind 80/443, and what is the least-privilege
  alternative to running as root?
- Does DNS resolution match the certificate strategy in use?

**Sources:** evilginx docs, local `ss` output, DNS answers.
**Output:** a preflight record and a running lab proxy.

---

## Scenario 2 — phishlet and hostname configuration

| # | Command | How to use |
|---|---------|------------|
| 6 | `(console) config domain lab.example.test` | Base domain |
| 7 | `(console) config ipv4 external 192.0.2.10` | Public address in lures |
| 8 | `(console) phishlets hostname o365 login.lab.example.test` | Bind hostname |
| 9 | `(console) phishlets enable o365` | Enable proxying |
| 10 | `curl -skI https://login.lab.example.test/` | Verify the listener responds |

**Research questions**
- Which phishlet fields control hostname validation and cookie capture?
- How does a target UI change surface as a capture failure rather than an error?

**Sources:** phishlet YAML, upstream docs, proxy debug output.
**Output:** an enabled phishlet scoped to lab hostnames.

---

## Scenario 3 — lure lifecycle

| # | Command | How to use |
|---|---------|------------|
| 11 | `(console) lures create o365` | Create a lure |
| 12 | `(console) lures get-url 0` | Retrieve its URL |
| 13 | `curl -skI '<lure-url>'` | Check redirector behaviour |
| 14 | `(console) lures` | List active lures |
| 15 | `(console) lures delete 0` | Expire the lure |

**Research questions**
- What distinguishes a lure URL from the phishlet hostname, and how is the
  redirector selected?
- Which logs record lure access, and how long should URLs remain valid?

**Sources:** console output, redirector pages, proxy logs.
**Output:** a controlled lure with a defined lifetime.

---

## Scenario 4 — session review and defensive validation

| # | Command | How to use |
|---|---------|------------|
| 16 | `(console) sessions` | List captured sessions |
| 17 | `(console) sessions 0` | Inspect one session |
| 18 | `find ./lab/conf -maxdepth 2 -type f` | Locate stored session data |
| 19 | `tcpdump -ni any port 443 -c 20` | Capture lab TLS metadata (root/cap) |
| 20 | `grep -R 'cookie' ./lab/conf \| head` | Inspect cookie fields (lab only) |

**Research questions**
- Which authentication artefacts are captured, and which are not?
- What telemetry would a defender need to detect this proxy pattern?

**Sources:** session output, packet metadata, ATT&CK T1539.
**Output:** a session inventory and a detection hypothesis.

---

## Scenario 5 — teardown and evidence handling

| # | Command | How to use |
|---|---------|------------|
| 21 | `(console) phishlets disable o365` | Stop proxying |
| 22 | `pkill -INT evilginx` | Graceful shutdown |
| 23 | `tar -czf evilginx-evidence.tgz ./lab/conf && sha256sum evilginx-evidence.tgz` | Package and hash evidence |
| 24 | `ss -ltnp \| grep -E ':(80\|443)\b'` | Confirm listeners are gone |
| 25 | `rm -rf ./lab` | **Destructive:** delete lab data after retention approval |

**Research questions**
- What must be retained for the report, and what must be destroyed as sensitive?
- How is teardown proven to the client and to the defenders?

**Sources:** engagement data-handling rules, hashes, listener checks.
**Output:** a teardown record and a retention decision.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-v` | 1 |
| `-c` | 1, 5 |
| `-p` | 1, 5 |
| `-t` | 1 |
| `-debug` | 2 |
| `-developer` | 1 |

Console commands are covered in scenarios 2–5 and are marked `(console)`.

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — bootstrap | 5 |
| 2 — phishlet | 5 |
| 3 — lures | 5 |
| 4 — sessions | 5 |
| 5 — teardown | 5 |

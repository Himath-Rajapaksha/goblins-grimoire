# hydra — online password guessing

- **Version:** Hydra v9.5 (`hydra -h`)
- **Binary:** `/usr/bin/hydra` (Debian package `hydra`)
- **Help snapshots:** [`help.txt`](help.txt), [`help-full.txt`](help-full.txt)
- **Upstream:** https://github.com/vanhauser-thc/thc-hydra
- **Docs:** `man hydra`; https://github.com/vanhauser-thc/thc-hydra#readme

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

hydra guesses login/password pairs against a live network service. It speaks
dozens of protocols — SSH, RDP, SMB, FTP, HTTP forms, LDAP, databases, SNMP,
VNC, and more — and tries a user list against a password list (or a
user:password file) with configurable parallelism. It is the online counterpart
to hashcat: it tests credentials against the service, so it is slow, loud, and
subject to lockout policies.

The installed build lists its supported services in [`help.txt`](help.txt).
Some modules are **not compiled in**: `afp`, `ncp`, `oracle`, `sapr3`, and
`smb2`. The `smb` module targets SMB1-era authentication; modern Windows
commonly disables SMB1, so use `netexec` for SMB spraying.

## Use cases

| Situation | Why hydra |
|-----------|-----------|
| SSH with a known user list | Test a small password list against one service |
| HTTP login form | `http-post-form` with a failure/success string |
| RDP exposure | `rdp` module for a bounded credential check |
| Database exposure | MySQL, MSSQL, PostgreSQL, MongoDB, Redis |
| SNMP community strings | `snmp` module |
| Password spraying | One password × many users with `-u` and low `-t` |
| Multiple hosts | `-M targets.txt` with per-host results |

## Requirements & install layout

- Debian package `hydra`; binary `/usr/bin/hydra`. Version 9.5.
- No root needed for most modules; raw-socket modules and some service
  bindings may need privileges.
- Restore files are written to the current directory; `-R` resumes, `-I`
  ignores an existing restore file.
- Proxy support is via environment variables:
  `HYDRA_PROXY`, `HYDRA_PROXY_HTTP` (supports `socks4://`, `socks5://`,
  `connect://`, and proxy-list files).
- Module-specific options are shown with `hydra -U <service>` and passed with
  `-m`.

## Basic syntax

```bash
hydra [[[-l LOGIN|-L FILE] [-p PASS|-P FILE]] | [-C FILE]] \
  [-e nsr] [-o FILE] [-t TASKS] [-M FILE [-T TASKS]] [-w TIME] \
  [-W TIME] [-f] [-s PORT] [-x MIN:MAX:CHARSET] [-c TIME] \
  [-ISOuvVd46] [-m MODULE_OPT] [service://server[:PORT][/OPT]]
```

```bash
hydra -l admin -P passlist.txt ssh://10.10.10.5
hydra -L users.txt -p 'Summer2026!' rdp://10.10.10.5
hydra -L users.txt -P passlist.txt -M targets.txt smb
hydra -l admin -P passlist.txt 10.10.10.5 http-post-form \
  '/login:user=^USER^&pass=^PASS^:F=Invalid credentials'
```

## Flag reference

### Credentials

| Flag | Meaning |
|------|---------|
| `-l <login>` | Single username |
| `-L <file>` | Username list |
| `-p <pass>` | Single password |
| `-P <file>` | Password list |
| `-C <file>` | Colon-separated `login:pass` list |
| `-e nsr` | Try null (`n`), login-as-password (`s`), reversed login (`r`) |
| `-x <min:max:charset>` | Generate passwords by brute force (see `-x -h`) |
| `-y` | Disable symbols in `-x` generation |
| `-r` | Non-random shuffling for `-x` |
| `-u` | Loop users instead of passwords (implied by `-x`) |

### Targets and timing

| Flag | Meaning |
|------|---------|
| `server` | DNS name, IP, or CIDR (`192.168.0.0/24`) |
| `-M <file>` | Target list, one per line; `:` specifies a port |
| `-s <port>` | Non-default port |
| `-S` | SSL connect |
| `-t <n>` | Parallel connects per target (default 16) |
| `-T <n>` | Parallel connects overall for `-M` (default 64) |
| `-w <s>` | Wait time for a response (default 32) |
| `-W <s>` | Wait time between connects per thread (default 0) |
| `-c <s>` | Wait time per login attempt across all threads (forces `-t 1`) |
| `-4` / `-6` | IPv4 (default) / IPv6 |

### Behavior and output

| Flag | Meaning |
|------|---------|
| `-f` | Stop after the first found pair (per host with `-M`) |
| `-F` | Stop after the first found pair globally |
| `-o <file>` | Write found pairs to a file |
| `-b <format>` | Output format: `text` (default), `json`, `jsonv1` |
| `-R` | Restore a previous session |
| `-I` | Ignore an existing restore file |
| `-K` | Do not redo failed attempts (useful for `-M` mass runs) |
| `-q` | Do not print connection errors |
| `-v` / `-V` / `-d` | Verbose / show each attempt / debug |
| `-O` | Use old SSLv2/SSLv3 |
| `-U` | Show module usage details |
| `-m <opt>` | Module-specific option |

## Supported services (installed build)

```text
adam6500 asterisk cisco cisco-enable cobaltstrike cvs firebird ftp[s]
http[s]-{head|get|post} http[s]-{get|post}-form http-proxy http-proxy-urlenum
icq imap[s] irc ldap2[s] ldap3[-{cram|digest}md5][s] memcached mongodb mssql
mysql nntp oracle-listener oracle-sid pcanywhere pcnfs pop3[s] postgres
radmin2 rdp redis rexec rlogin rpcap rsh rtsp s7-300 sip smb smtp[s]
smtp-enum snmp socks5 ssh sshkey svn teamspeak telnet[s] vmauthd vnc xmpp
```

**Not compiled in:** `afp`, `ncp`, `oracle`, `sapr3`, `smb2`.

## Recipes

Authorized targets only. Start with a single user and a tiny password list to
measure lockout behavior.

**1. SSH with one user and a small list.**

```bash
hydra -l deploy -P passwords.txt -t 4 -f ssh://10.10.10.5
```

**2. Password spray: one password, many users, low parallelism.** Spraying
avoids lockout better than a full cross-product.

```bash
hydra -L users.txt -p 'Spring2026!' -t 2 -W 5 ssh://10.10.10.5
```

**3. HTTP POST form.** Get the exact field names and failure text from the
login page; `^USER^` and `^PASS^` are the placeholders. `F=` marks a failed
login, `S=` marks success.

```bash
hydra -l admin -P passwords.txt 10.10.10.5 http-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid username or password'
```

**4. HTTP Basic auth.**

```bash
hydra -L users.txt -P passwords.txt 10.10.10.5 http-get /admin/
```

**5. RDP with a bounded list.**

```bash
hydra -l administrator -P passwords.txt -t 2 -f rdp://10.10.10.5
```

**6. Multiple hosts, one service.**

```bash
hydra -L users.txt -P passwords.txt -M targets.txt -T 8 -f smb
```

**7. Null / same-as-login / reversed checks.** `-e nsr` is cheap and catches
weak defaults before a dictionary run.

```bash
hydra -L users.txt -e nsr ssh://10.10.10.5
```

**8. Brute-force a numeric PIN.** `-x 4:6:0-9` tries lengths 4-6 over digits.

```bash
hydra -l admin -x 4:6:0-9 -t 2 10.10.10.5 http-post-form \
  '/pin:pin=^PASS^:F=Wrong PIN'
```

**9. Machine-readable output.**

```bash
hydra -L users.txt -P passwords.txt -o evidence/hydra.json -b json ssh://10.10.10.5
```

## Output & pipelines

- A success line is `[22][ssh] host: 10.10.10.5 login: deploy password: ...`.
- `-o` writes the same lines; `-b json` produces structured records.
- `-V` prints every attempt, which is useful for a lab but very noisy on a real
  target.
- `-d` shows module-level debugging, including the HTTP form response matching.
- Exit status is not a reliable success signal; parse the output or the JSON.
- `-K` with `-M` keeps a mass run from retrying hosts that already failed.

## Gotchas & operational security

- **Lockout is the main risk.** Three failed attempts can lock a domain account
  for 30 minutes. Check policy first and use `-t 1`/`-c` for sensitive targets.
- **`-t 16` is aggressive.** It is the default per target; lower it for RDP,
  SSH with rate limits, and anything with an account-lockout policy.
- **SMB1 only.** `smb2` is not compiled in; modern Windows often rejects the
  `smb` module. Use `netexec smb` for SMB spraying.
- **HTTP form matching is brittle.** A dynamic error page, CSRF token, or
  JavaScript-rendered login will break `F=`/`S=` matching; confirm the exact
  strings with a manual request.
- **`-x` is exponential.** `-x 6:8:aA1` is enormous; bound the length and
  charset, and estimate the attempt count first.
- **Proxies change the source, not the authorization.** `HYDRA_PROXY` does not
  make unauthorized testing acceptable.
- **Restore files contain credentials.** They are written in the working
  directory; clean them up and keep them out of shared repos.
- **Success can be a false positive.** Some services return a valid-looking
  response for any credentials; confirm manually before reporting.

## Legal & safety

Use hydra only against systems you own or have **explicit written
authorization** to test. Online guessing is intrusive, can lock accounts, and
may violate computer-misuse law and terms of service. Never run it against
third-party services, and stop immediately if the rules of engagement set an
attempt limit.

## Related tools

- [`netexec/`](../netexec/README.md) — SMB/LDAP/WinRM spraying with lockout-aware
  controls.
- [`hashcat/`](../hashcat/README.md) — offline cracking once a hash is obtained.
- [`crunch/`](../crunch/README.md) — generate targeted password lists.
- [`cewl/`](../cewl/README.md) — build wordlists from target web content.
- [`kerbrute/`](../kerbrute/README.md) — Kerberos-based user enumeration and
  pre-auth checks.
- [`burp/`](../burp/README.md) — inspect the login request before form guessing.

## References

- Installed help snapshots: [`help.txt`](help.txt), [`help-full.txt`](help-full.txt)
- Upstream README: https://github.com/vanhauser-thc/thc-hydra
- Kali tool page: https://www.kali.org/tools/hydra/

# medusa — parallel online password brute forcer

- **Version:** Medusa v2.3 (`medusa -V`)
- **Binary:** `/usr/bin/medusa` (Debian package `medusa`)
- **Modules:** `/usr/lib/x86_64-linux-gnu/medusa/modules/*.mod`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/jmk-foofus/medusa
- **Docs:** `man medusa`; http://www.foofus.net/jmk/medusa/medusa.html

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Medusa guesses credentials against a live network service. It drives a
protocol-specific module, tries each username/password pair, and reports valid
logins. It is built for speed and parallelism across both hosts and logins,
which makes it the natural companion to `hydra` when one tool lacks a module or
you need to spread attempts across a host list.

Medusa is **online**: every attempt reaches the service. It is slow, noisy, and
subject to lockout, rate limiting, and account-lockout policy. It is not a hash
cracker; use [`john`](../john/README.md) or [`hashcat`](../hashcat/README.md)
for offline work.

## Installed modules

The installed build ships 25 modules:

| Category | Modules |
|----------|---------|
| Remote access | `ssh`, `telnet`, `rlogin`, `rsh`, `rexec`, `rdp`, `vnc`, `pcanywhere`, `vmauthd` |
| Web | `http`, `web-form`, `svn` |
| Mail | `smtp`, `smtp-vrfy`, `pop3`, `imap`, `nntp` |
| Databases | `mysql`, `mssql`, `postgres` |
| File / directory | `ftp`, `smbnt`, `cvs` |
| Network / other | `snmp`, `wrapper` |

List them from the installed build with `medusa -d`, and read a module's
options with `medusa -M <module> -q`.

## Use cases

| Situation | Why medusa |
|-----------|------------|
| SSH password spraying | `-M ssh` with a small password list |
| HTTP basic/digest/NTLM auth | `-M http` with `-m AUTH:` |
| HTML login form | `-M web-form` with `FORM-DATA` |
| SMB local or domain accounts | `-M smbnt` with `GROUP:` |
| RDP | `-M rdp` (not Windows 2003/XP) |
| SNMP community strings | `-M snmp` with `ACCESS:READ`/`WRITE` |
| Multiple hosts at once | `-H hosts.txt` and `-T` |
| Stop after the first hit | `-f` (host) or `-F` (any host) |

## Flag reference

### Target and credential inputs

| Flag | Meaning |
|------|---------|
| `-h HOST` | Single target hostname or IP |
| `-H FILE` | File of target hosts |
| `-u USER` | Single username |
| `-U FILE` | File of usernames |
| `-p PASS` | Single password |
| `-P FILE` | File of passwords |
| `-C FILE` | Combo file: `host:user:password` per line |
| `-e n/s/ns` | Also try no password (`n`) and password = username (`s`) |
| `-M MODULE` | Module name, without `.mod` |
| `-m PARAM` | Module parameter; repeat for multiple parameters |

### Connection and pacing

| Flag | Meaning |
|------|---------|
| `-n NUM` | Non-default TCP port |
| `-s` | Enable SSL |
| `-g NUM` | Give up connecting after `NUM` seconds (default 3) |
| `-r NUM` | Sleep `NUM` seconds between retries (default 3) |
| `-R NUM` | Retry `NUM` times (total attempts `NUM + 1`) |
| `-c NUM` | Microseconds to wait for socket readiness (default 500) |
| `-t NUM` | Concurrent logins |
| `-T NUM` | Concurrent hosts |
| `-L` | One username per thread instead of one username at a time |

### Control and output

| Flag | Meaning |
|------|---------|
| `-f` | Stop a host after the first valid credential |
| `-F` | Stop the whole audit after the first valid credential |
| `-O FILE` | Append found credentials and audit start/stop to a log |
| `-Z TEXT` | Resume from a previous scan map |
| `-b` | Suppress the startup banner |
| `-q` | Show the selected module's usage |
| `-v NUM` | Verbose level 0–6 |
| `-w NUM` | Error debug level 0–10 |
| `-d` | Dump all known modules |
| `-V` | Version |

## Module parameters

These are the options most often needed; run `medusa -M <module> -q` for the
authoritative list in this build.

| Module | Parameter | Meaning |
|--------|-----------|---------|
| `http` | `USER-AGENT:`, `DIR:`, `METHOD:`, `AUTH:BASIC\|DIGEST\|NTLM`, `DOMAIN:`, `CUSTOM-HEADER:` | HTTP auth attempts |
| `web-form` | `FORM:`, `DENY-SIGNAL:`, `FORM-DATA:<METHOD>?<FIELDS>`, `CUSTOM-HEADER:`, `USER-AGENT:` | HTML form login |
| `smbnt` | `GROUP:DOMAIN\|LOCAL\|BOTH`, `GROUP_OTHER:`, `PASS:PASSWORD\|HASH\|MACHINE`, `AUTH:LM\|NTLM\|LMv2\|NTLMv2`, `MODE:AUTO\|NETBIOS\|SMB2` | SMBv1–3 |
| `rdp` | `DOMAIN:`, `PASS:PASSWORD\|HASH` | RDP |
| `snmp` | `TIMEOUT:`, `SEND_DELAY:`, `VERSION:1\|2C`, `ACCESS:READ\|WRITE` | Community strings |
| `ftp` | `MODE:NORMAL\|EXPLICIT\|IMPLICIT` | FTP/FTPS |
| `mysql` | `PASS:PASSWORD\|HASH` | MySQL (non-SHA1 hashes only) |
| `ssh` | `BANNER:` | SSH client banner |

## Recipes

**1. SSH against one user and a small password list.**

```bash
medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M ssh -f
```

**2. Spray one password across a user list.**

```bash
medusa -h 10.10.10.5 -U ~/medusa-demo/users.txt -p 'Summer2026!' -M ssh -t 4
```

**3. HTTP basic auth with a custom path.**

```bash
medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M http \
  -m DIR:/admin/ -m AUTH:BASIC -n 8080
```

**4. HTML login form.**

```bash
medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M web-form \
  -m FORM:/login.php -m DENY-SIGNAL:'Invalid login' \
  -m FORM-DATA:'post?username=&password=&submit=Login'
```

**5. SMB local accounts.**

```bash
medusa -h 10.10.10.5 -u Administrator -P ~/medusa-demo/passwords.txt \
  -M smbnt -m GROUP:LOCAL -m AUTH:NTLM
```

**6. Many hosts, stop at the first valid credential on each.**

```bash
medusa -H ~/medusa-demo/hosts.txt -U ~/medusa-demo/users.txt \
  -P ~/medusa-demo/passwords.txt -M ssh -T 4 -t 4 -f -O ~/medusa-demo/found.log
```

**7. Combo file.**

```bash
medusa -C ~/medusa-demo/combo.txt -M ssh -f
```

**8. Resume and log.**

```bash
medusa -h 10.10.10.5 -U users.txt -P passwords.txt -M ssh -Z ~/medusa-demo/scan.map
```

**9. Discover module options.**

```bash
medusa -d
medusa -M smbnt -q
```

## Gotchas

- **Every attempt is a real login.** Lockout thresholds, fail2ban, and SIEM
  alerts are triggered by volume, not intent. Pace with `-t`, `-T`, `-r`, and
  `-c`.
- **`-h` is the host, not help.** Help is printed when the command line is
  invalid; `-q` prints module help only with `-M`.
- **`-e ns` adds guesses.** `n` tries an empty password and `s` tries the
  username as the password. Both are noisy and can lock accounts.
- **`-f` vs `-F`.** `-f` stops the current host; `-F` stops the entire audit.
- **Combo files are colon-separated** `host:user:password`. PwDump files
  (`user:id:lm:ntlm:::`) are auto-detected.
- **`smbnt` is the SMB module.** There is no `smb2` module in this build;
  `smbnt` covers SMBv1–3 and signing.
- **RDP does not work against Windows 2003/XP and earlier.**
- **`-s` enables SSL, not "silent".** Silence is `-b` (banner) and `-v 0`.
- **Use `netexec` for AD-aware spraying.** Medusa has no domain lockout
  awareness, BloodHound integration, or per-user bad-password counting.

## Related docs

- [`hydra`](../hydra/README.md) — the other major online brute forcer
- [`john`](../john/README.md) — offline cracking of recovered hashes
- [`hashcat`](../hashcat/README.md) — GPU/CPU offline cracking
- [`netexec`](../netexec/README.md) — AD-aware SMB/LDAP/WinRM spraying
- [`smbclient`](../smbclient/README.md) — verify a recovered SMB credential

# enum4linux-ng — SMB / NetBIOS / AD enumeration

- **Version:** 1.3.10 (next-generation rewrite)
- **Binary:** `/usr/local/bin/enum4linux-ng`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/cddmp/enum4linux-ng
- **Docs:** https://github.com/cddmp/enum4linux-ng#usage

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

enum4linux-ng is a modern rewrite of `enum4linux`. It enumerates Windows/Samba
hosts over SMB/RPC: users, groups, shares, password policy, OS info, printers,
sessions, and more. It handles null sessions, guest sessions, and credentialed
enumeration, and can export results to YAML/JSON.

## Use cases

| Situation | Flags |
|-----------|-------|
| Null-session SMB recon | default |
| Credentialed enumeration | `-u user -p pass` |
| Pass-the-hash | `-u user -H <NT>` |
| Target a specific service | `-s` (see below) |
| Export for tooling | `-oY` / `-oJ` |
| Filter noise | `-A` (all simple), `-v` verbose |

## Dependencies (this machine)

All four helper binaries are installed and on `PATH`:

```bash
smbclient /usr/bin/smbclient
rpcclient /usr/bin/rpcclient
nmblookup /usr/bin/nmblookup   # samba-common-bin
net       /usr/bin/net         # samba-common-bin
```

If a future rebuild lacks them, install `samba-common-bin`; without it,
NetBIOS and local-group enumeration degrade while SMB/RPC still works.

## Basic syntax

```bash
enum4linux-ng -A 10.10.10.5
enum4linux-ng -u user -p pass 10.10.10.5
enum4linux-ng -u user -H <NThash> 10.10.10.5
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-t, --target <ip/host>` | Target (also accepts positional) |
| `-u, --username <user>` | Username |
| `-p, --password <pass>` | Password |
| `-H, --hash <hash>` | NTLM hash (pass-the-hash) |
| `-d, --domain <domain>` | Domain |
| `-s, --services <list>` | Services to enumerate: `SMB`, `LDAP`, `NBT`, `ALL` (comma-separated) |
| `-a, --all` | All simple enumeration (no brute force) |
| `-A` | Alias for `-a` in many builds (check `-h`) |
| `-v, --verbose` | Verbose output |
| `-oY, --yaml <file>` | YAML output |
| `-oJ, --json <file>` | JSON output |
| `-w, --wordlist <file>` | Wordlist for RID brute force |
| `-R, --rid <range>` | RID range (e.g. `500-2000`) |
| `-k, --known-users <file>` | Known usernames file |
| `-L, --ldap` | LDAP enumeration |
| `-P, --port <port>` | SMB port (default 445) |
| `-m, --max-time <sec>` | Per-command timeout |
| `-l, --log <file>` | Log file |
| `--no-color` | Disable colors |
| `--debug` | Debug output |
| `-V, --version` | Version |
| `-h, --help` | Help |

> Flag names can vary slightly by release. `enum4linux-ng -h` is authoritative
> on this machine; see [`help.txt`](help.txt).

## Recipes

```bash
# Null session, all simple checks
enum4linux-ng -A 10.10.10.5

# Credentialed
enum4linux-ng -u 'CORP\user' -p 'Password123' 10.10.10.5

# Pass-the-hash
enum4linux-ng -u administrator -H aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0 10.10.10.5

# Only SMB
enum4linux-ng -s SMB -u user -p pass 10.10.10.5

# JSON output for reporting
enum4linux-ng -A -oJ enum.json 10.10.10.5

# RID brute force known range
enum4linux-ng -u user -p pass -R 500-3000 10.10.10.5
```

## What it reports

- **OS / host info:** domain, OS version, server string, SMB signing.
- **Users:** usernames, RIDs, descriptions, flags (disabled/locked), groups.
- **Groups:** local and domain groups, membership.
- **Shares:** names, comments, access hints.
- **Password policy:** min length, lockout threshold/duration, complexity.
- **Sessions / logged-on users.**
- **Printers.**
- **LDAP** (if enabled): naming contexts, users, groups.

## Gotchas

- Null sessions are disabled on modern Windows; use credentials.
- RID brute force is noisy and can trigger alerts.
- Lockout risk when password spraying — check `--pass-pol` first via another tool.
- Output is only as good as the permissions of the account used.
- Missing `nmblookup`/`net` reduces NetBIOS and local-group coverage.
- For large AD environments, pair with NetExec/BloodHound rather than RID brute force.

## References

- Source/usage: https://github.com/cddmp/enum4linux-ng
- Samba `smbclient`/`rpcclient` docs: https://www.samba.org/samba/docs/

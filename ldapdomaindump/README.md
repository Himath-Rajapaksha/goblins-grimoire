# ldapdomaindump — Active Directory LDAP domain dumper

- **Version:** 0.9.4-2 (Debian package `python3-ldapdomaindump`)
- **Binary:** `/usr/bin/ldapdomaindump` (Python entry point)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/dirkjanm/ldapdomaindump/
- **Licence:** MIT

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`ldapdomaindump` binds to a domain controller over LDAP, walks the directory,
and writes users, computers, groups, domain policy, and trusts to HTML, JSON,
and greppable text. It is the fast first-pass AD enumeration step after you
have any valid credential — or even an anonymous bind on a misconfigured
directory.

It is deliberately read-only: it queries LDAP and writes local report files.
It does not exploit, modify, or persist in the directory.

Typical assessment flow:

1. Obtain a credential (password, NT hash, or machine account).
2. Dump the domain to JSON and greppable text.
3. Parse the output to find privileged groups, kerberoastable users, old
   operating systems, and trusts.
4. Feed the interesting accounts into targeted tools such as `impacket`,
   `netexec`, or `bloodhound`.

## Output files

With all formats enabled, a successful dump writes:

| File | Contents |
|------|----------|
| `domain_users.html` / `.json` / `.grep` | User accounts and attributes |
| `domain_computers.html` / `.json` / `.grep` | Computer accounts and OS information |
| `domain_groups.html` / `.json` / `.grep` | Groups and membership |
| `domain_policy.html` / `.json` / `.grep` | Domain password and lockout policy |
| `domain_trusts.html` / `.json` / `.grep` | Trust relationships |
| `domain_users_by_group.html` / `.json` | Users grouped by group |
| `domain_computers_by_os.html` / `.json` | Computers grouped by operating system |

`--grouped-json` adds JSON for the two grouped reports. Without it only the
HTML grouped reports are written. `--no-html`, `--no-json`, and `--no-grep`
disable individual formats.

The output directory is created on the first write. Files are overwritten
without a timestamp, so use a fresh directory per dump.

## Options

```text
ldapdomaindump [-h] [-u USERNAME] [-p PASSWORD] [-at {NTLM,SIMPLE}]
               [-o DIRECTORY] [--no-html] [--no-json] [--no-grep]
               [--grouped-json] [-d DELIMITER] [-r] [-n DNS_SERVER]
               [-m]
               HOSTNAME
```

### Required

| Argument | Meaning |
|----------|---------|
| `HOSTNAME` | Hostname, IP, or `ldap://host:port` / `ldaps://host:port` connection string |

### Authentication

| Option | Meaning |
|--------|---------|
| `-u, --user USERNAME` | `DOMAIN\username`; omit for anonymous |
| `-p, --password PASSWORD` | Password or `LM:NTLM` hash; prompts if omitted |
| `-at, --authtype {NTLM,SIMPLE}` | Authentication type, default `NTLM` |

### Output

| Option | Meaning |
|--------|---------|
| `-o, --outdir DIRECTORY` | Output directory, default current directory |
| `--no-html` | Disable HTML output |
| `--no-json` | Disable JSON output |
| `--no-grep` | Disable greppable output |
| `--grouped-json` | Also write JSON for grouped reports |
| `-d, --delimiter DELIMITER` | Field delimiter for greppable output, default tab |

### Miscellaneous

| Option | Meaning |
|--------|---------|
| `-r, --resolve` | Reverse-resolve computer hostnames |
| `-n, --dns-server DNS_SERVER` | Use a custom DNS resolver instead of system DNS |
| `-m, --minimal` | Query only a minimal attribute set to limit memory use |

## Recipes

**1. Authenticated NTLM dump.**

```bash
ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump dc01.corp.example
```

**2. Pass-the-hash.**

```bash
ldapdomaindump -u 'CORP\alice' -p 'aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0' \
  -o dump dc01.corp.example
```

**3. LDAPS.**

```bash
ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump ldaps://dc01.corp.example:636
```

**4. Anonymous bind.**

```bash
ldapdomaindump -o dump dc01.corp.example
```

**5. JSON only.**

```bash
ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump --no-html --no-grep dc01.corp.example
```

**6. Greppable only with a custom delimiter.**

```bash
ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump \
  --no-html --no-json -d ',' dc01.corp.example
```

**7. Large domain with minimal attributes and custom DNS.**

```bash
ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump \
  -m -n 10.0.0.10 dc01.corp.example
```

**8. Resolve computer names for a cleaner inventory.**

```bash
ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump -r dc01.corp.example
```

**9. Summarise users from the greppable output.**

```bash
awk -F'\t' 'NR>1 {print $1}' dump/domain_users.grep | sort -u | head -50
```

**10. List privileged groups.**

```bash
grep -iE 'admin|domain admins|enterprise admins|schema admins' dump/domain_groups.grep
```

**11. Find old operating systems.**

```bash
grep -iE '2003|2008|xp|windows 7' dump/domain_computers.grep
```

**12. Query a JSON file with jq.**

```bash
jq -r '.[].attributes.sAMAccountName' dump/domain_users.json | head
```

## Exit status

| Code | Meaning |
|------|---------|
| 0 | Dump completed |
| 1 | Bind or dump failure (often a Python traceback) |
| 2 | Argument error, including a missing `HOSTNAME` |

`--version` is not implemented: argparse treats it as an unknown option and
exits 2 with the usage text.

## Quirks and limits

- **There is no `--version` flag.** It exits 2. Use `dpkg -s
  python3-ldapdomaindump` or `pip show ldapdomaindump`.
- **Connection failures print a traceback.** A refused or unreachable LDAP
  endpoint exits 1 with a Python stack trace rather than a clean error.
- **Anonymous binds usually fail.** The tool prints a warning, then either
  binds to a directory that allows anonymous reads or dies with a traceback.
- **The password prompt is `getpass`.** If `-p` is omitted, the tool prompts
  on the terminal; it cannot read a password from a pipe reliably.
- **Pass-the-hash uses `LM:NTLM`.** Supply both halves, not just the NT hash.
- **`-u` wants `DOMAIN\username`.** A UPN such as `alice@corp.example` may
  work with `SIMPLE`, but NTLM expects the down-level form.
- **`SIMPLE` sends the password in cleartext.** Use it only over LDAPS or on a
  trusted network.
- **`-o` creates the directory only when the first file is written.** A failed
  bind leaves no output directory.
- **Existing files are overwritten.** Use a new directory per dump or archive
  the previous one.
- **`--grouped-json` affects only the grouped reports.** The per-category JSON
  files are written regardless of this flag.
- **`-r` can be slow and noisy.** Reverse DNS on a large directory generates a
  lot of lookups and may hang on broken resolvers.
- **`-n` changes resolver, not the LDAP server.** It is for name resolution of
  computer accounts, not for choosing the DC.
- **`-m` trades completeness for memory.** Some attributes are omitted, so
  do not use it when you need full group membership or OS detail.
- **HTML embeds CSS inline.** The reports are self-contained but large.
- **The tool is read-only.** It does not modify the directory, but the bind
  and queries are logged by the DC.

## Related tools

- [`ldapsearch`](../ldapsearch/) — raw LDAP queries and filters.
- [`netexec`](../netexec/) — multi-protocol AD enumeration and exploitation.
- [`impacket`](../impacket/) — Kerberos, SMB, and LDAP attack tooling.
- [`rpcclient`](../rpcclient/) — MS-RPC SAMR/LSA enumeration.
- [`nbtscan`](../nbtscan/) — NetBIOS name and workgroup discovery.
- [`enum4linux-ng`](../enum4linux-ng/) — SMB and NetBIOS enumeration.
- [`smbclient`](../smbclient/) — SMB share access.

# whois — client for the WHOIS directory service (RFC 3912)

- **Version:** whois 5.6.3 (`whois --version`)
- **Binary:** `/usr/bin/whois` (Debian package `whois`)
- **Full help snapshot:** [`help.txt`](help.txt) — `whois --help`
- **Config file:** `/etc/whois.conf` — **not present on this install**
- **Upstream:** https://github.com/rfc1036/whois
- **Docs:** `man whois`, `man 5 whois.conf`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`whois` opens a plain TCP connection to port 43 of a registry's WHOIS server,
sends a single query line, and prints the free-form text that comes back. It is
a thin client: the protocol has no authentication, no encryption, no standard
response format, and no standard schema.

The client's real work is **server selection**. It guesses the authoritative
server from the object you pass, then — for thin registries — follows the
registry's referral to the registrar and queries that server too. That
recursion is why `whois example.com` can print two records from two different
servers.

```text
whois [OPTION]... OBJECT...
```

Objects can be domain names, IPv4/IPv6 addresses and netblocks, AS numbers, NIC
handles, or registry-specific keys.

## Use cases

| Situation | Query shape |
|-----------|-------------|
| Who registered a domain? | `whois example.com` |
| What is the registrar and status? | `whois example.com` (registry + registrar) |
| Who owns an IP or netblock? | `whois 203.0.113.10` |
| Which AS announces a prefix? | `whois AS64500` |
| What abuse contact applies? | `whois -b 203.0.113.0/24` |
| RIPE object inverse search | `whois -i admin-c PERSON-HANDLE` |
| Find less/more specific RIPE objects | `whois -l`, `-L`, `-m`, `-M` |
| Ask a specific server | `whois -h whois.arin.net 203.0.113.10` |
| Registry-only answer, no registrar hop | `whois --no-recursion example.com` |
| Follow IANA's referral explicitly | `whois -I example.com` |
| Get a registry's object template | `whois -t person` |

## Flag reference

### Client options (all servers)

| Flag | Meaning |
|------|---------|
| `-h HOST`, `--host HOST` | Connect to HOST instead of guessing |
| `-p PORT`, `--port PORT` | Connect to PORT (default 43) |
| `-I` | Query `whois.iana.org` first and follow its referral |
| `-H` | Hide legal disclaimers |
| `--verbose` | Explain what the client is doing |
| `--no-recursion` | Disable registry-to-registrar recursion |
| `--help` | Print help and exit |
| `--version` | Print version and exit |

### RIPE-specific options

These are sent to `whois.ripe.net` and RIPE-like servers. **Other servers
ignore them**, often silently.

| Flag | Meaning |
|------|---------|
| `-l` | One level less specific match |
| `-L` | All levels of less specific matches |
| `-m` | All one level more specific matches |
| `-M` | All levels of more specific matches |
| `-c` | Smallest match containing a `mnt-irt` attribute |
| `-x` | Exact match |
| `-b` | Brief IP ranges with abuse contact |
| `-B` | Turn off object filtering (show e-mail addresses) |
| `-G` | Turn off grouping of associated objects |
| `-d` | Return DNS reverse-delegation objects too |
| `-i ATTR[,ATTR]...` | Inverse lookup for the given attributes |
| `-T TYPE[,TYPE]...` | Only look for objects of TYPE |
| `-K` | Return primary keys only |
| `-r` | Turn off recursive contact lookups |
| `-R` | Show the local copy even if it contains a referral |
| `-a` | Also search all mirrored databases |
| `-s SOURCE[,SOURCE]...` | Search only databases mirrored from SOURCE |
| `-g SOURCE:FIRST-LAST` | Find updates between serial numbers |
| `-t TYPE` | Request the object template for TYPE |
| `-v TYPE` | Request the verbose template for TYPE |
| `-q version\|sources\|types` | Query server information |

### Environment

| Variable | Effect |
|----------|--------|
| `WHOIS_OPTIONS` | Options evaluated before the command line |
| `WHOIS_SERVER` | Fallback server when the client cannot guess |
| `LANG`, `LC_MESSAGES` | Request English from `whois.nic.ad.jp` / `whois.jprs.jp` unless Japanese |

## Recipes

**1. Domain registration.**

```bash
whois example.com
```

**2. Registry record only — no registrar hop.**

```bash
whois --no-recursion example.com
```

**3. Follow the IANA referral explicitly.**

```bash
whois -I example.com
```

**4. IP address and netblock.**

```bash
whois 203.0.113.10
whois 203.0.113.0/24
```

**5. AS number.**

```bash
whois AS64500
```

**6. Ask a specific registry.**

```bash
whois -h whois.arin.net 203.0.113.10
whois -h whois.ripe.net -- '-i admin-c ABCD-RIPE'
```

**7. RIPE inverse lookup.**

```bash
whois -i admin-c ABCD-RIPE
```

**8. Abuse contact for a range.**

```bash
whois -b 203.0.113.0/24
```

**9. Object template.**

```bash
whois -t person
```

**10. Hide disclaimers and add verbosity.**

```bash
whois -H --verbose example.com
```

**11. Bound a slow server.**

```bash
timeout 15 whois -h whois.example.net example.com
```

**12. Override the server table with a config file.**

```bash
# /etc/whois.conf (root only):
# \.example$  whois.example.net
whois example.com
```

## Gotchas

- **Plaintext and unauthenticated.** WHOIS is TCP/43 with no TLS. Assume
  anyone on the path can read the query and the answer.
- **No timeout option.** A dead or slow server hangs the client; wrap it in
  `timeout`.
- **Recursion is implicit.** `whois example.com` may query two servers. Use
  `--no-recursion` when you need the registry record specifically.
- **Thin vs thick registries.** `.com`/`.net` return a registry record that
  points at the registrar; thick registries (e.g. many ccTLDs) return the full
  record in one answer.
- **`-I` shares the whole query with IANA.** The manual warns explicitly that
  `whois.iana.org` receives your complete query.
- **RIPE flags are ignored elsewhere.** Passing `-B` to a non-RIPE server may
  do nothing; the manual calls this out as possibly unintended.
- **Options meant for the server must follow `--`.** The client otherwise
  tries to interpret them.
- **`-B` cannot un-redact what the registry removed.** GDPR and registry
  policy have stripped contact data from many gTLD records; the flag only
  disables client-side filtering.
- **`-H` changes presentation, not content.** It hides disclaimers; it does
  not change the record.
- **The config file is absent here.** This install uses the built-in server
  table only. If you create `/etc/whois.conf`, each line is a regex plus a
  server name.
- **Output is registry-specific free text.** There is no schema. Parse per
  registry, and treat missing fields as unknown rather than empty.
- **IDNs are converted to ACE.** Query `xn--…` labels, not Unicode, when you
  script it.
- **`whois` is not DNS.** It returns registration data, not A/AAAA/MX records;
  use `dig` for DNS.
- **Rate limits are real.** Registries throttle and block aggressive clients.
  Space queries and cache results.

## Related docs

- [`dnsrecon`](../dnsrecon/README.md) — DNS enumeration, which whois does not do
- [`theharvester`](../theharvester/README.md) — OSINT collection that consumes
  whois output
- [`recon-ng`](../recon-ng/README.md) — recon framework with whois modules
- [`spiderfoot`](../spiderfoot/README.md) — automated OSINT including registry
  data
- [`nmap`](../nmap/README.md) — verify that a WHOIS server is reachable
- [`traceroute`](../traceroute/README.md) — network path, not registration data

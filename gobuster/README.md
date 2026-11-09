# gobuster — directory, DNS, vhost, and bucket enumeration

- **Version:** 3.6 (`gobuster version`)
- **Binary:** `/usr/bin/gobuster` (Debian package `gobuster`)
- **Help snapshots:** [`help.txt`](help.txt) (root), [`help-dir.txt`](help-dir.txt),
  [`help-dns.txt`](help-dns.txt), [`help-vhost.txt`](help-vhost.txt),
  [`help-fuzz.txt`](help-fuzz.txt)
- **Upstream:** https://github.com/OJ/gobuster
- **Docs:** `man gobuster`; https://github.com/OJ/gobuster#readme

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

gobuster is a multi-mode brute-forcer. Each subcommand answers a different
question with the same engine: `dir` finds paths and files, `dns` finds
subdomains, `vhost` finds virtual hosts served from one address, `fuzz` replaces
`FUZZ` anywhere in a request, and `gcs`/`s3`/`tftp` enumerate cloud buckets and
TFTP files. It is smaller and simpler than ffuf: fewer matcher dimensions, no
auto-calibration, but a very predictable CLI.

The trade-off is deliberate. gobuster is fast to start and easy to script; ffuf
is more flexible when responses are noisy or the injection point is unusual.

## Use cases

| Situation | Why gobuster |
|-----------|--------------|
| First-pass content discovery | `dir` with a status blacklist is quick and readable |
| Subdomain enumeration | `dns` resolves names and can show CNAMEs/IPs |
| Shared-hosting vhosts | `vhost` finds names that only exist in web-server config |
| Cloud bucket checks | `s3`/`gcs` test bucket-name guesses |
| TFTP inventory | `tftp` enumerates files on an exposed TFTP service |
| Authenticated discovery | Cookies, headers, Basic auth, and client certs |
| Resume a long run | `--wordlist-offset` continues where a run stopped |

## Requirements & install layout

- Debian package `gobuster`; binary `/usr/bin/gobuster`. Version 3.6.
- No root and no config file. Wordlists are external.
- Defaults: 10 threads, 10s HTTP timeout, 1s DNS timeout, `dir` blacklist `404`,
  User-Agent `gobuster/3.6`.
- `-w -` reads the wordlist from stdin.
- Global flags apply to every subcommand; subcommand flags add mode-specific
  options.

## Basic syntax

```bash
gobuster dir -u https://target -w wordlist.txt
gobuster dns -d example.com -w subdomains.txt
gobuster vhost -u https://10.10.10.5 -w vhosts.txt --append-domain
gobuster fuzz -u 'https://target/?id=FUZZ' -w values.txt
```

Run `gobuster <mode> --help` for the mode's flags; the root help only lists the
subcommands and global flags.

## Flag reference

### Global flags (all modes)

| Flag | Meaning |
|------|---------|
| `-w, --wordlist <file>` | Wordlist path; `-` reads stdin |
| `-t, --threads <n>` | Concurrent threads (default 10) |
| `-o, --output <file>` | Write results to a file |
| `-q, --quiet` | Suppress banner and noise |
| `-v, --verbose` | Verbose output (errors) |
| `--delay <duration>` | Wait between requests per thread (e.g. `1500ms`) |
| `--debug` | Debug output |
| `--no-error` | Do not display errors |
| `--no-color` | Disable color |
| `-z, --no-progress` | Disable progress display |
| `-p, --pattern <file>` | Replacement patterns |
| `--wordlist-offset <n>` | Resume at a wordlist position |

### `dir` mode

| Flag | Meaning |
|------|---------|
| `-u, --url <url>` | Target URL |
| `-x, --extensions <list>` | Extensions to try (e.g. `php,html`) |
| `-X, --extensions-file <file>` | Read extensions from a file |
| `-e, --expanded` | Print full URLs |
| `-f, --add-slash` | Append `/` to each request |
| `-d, --discover-backup` | Also try backup extensions |
| `-r, --follow-redirect` | Follow redirects |
| `-k, --no-tls-validation` | Skip TLS verification |
| `-m, --method <verb>` | HTTP method (default GET) |
| `-c, --cookies <cookie>` | Cookie header |
| `-H, --headers <header>` | Header, repeatable |
| `-a, --useragent <ua>` | User-Agent (default `gobuster/3.6`) |
| `--random-agent` | Random User-Agent |
| `-U, --username <user>` / `-P, --password <pass>` | Basic auth |
| `-s, --status-codes <list>` | Positive status codes (ranges allowed) |
| `-b, --status-codes-blacklist <list>` | Negative codes (default `404`) |
| `--exclude-length <list>` | Exclude response sizes (ranges allowed) |
| `-n, --no-status` | Do not print status codes |
| `--hide-length` | Do not print body length |
| `--proxy <url>` | HTTP/SOCKS5 proxy |
| `--retry` / `--retry-attempts <n>` | Retry on timeout (default 3) |
| `--timeout <duration>` | HTTP timeout (default 10s) |

### `dns` mode

| Flag | Meaning |
|------|---------|
| `-d, --domain <domain>` | Target domain |
| `-r, --resolver <server[:port]>` | Custom DNS resolver |
| `-c, --show-cname` | Show CNAME records (not with `-i`) |
| `-i, --show-ips` | Show resolved IPs |
| `--wildcard` | Continue even when a wildcard DNS record is found |
| `--no-fqdn` | Do not append a trailing dot |
| `--timeout <duration>` | DNS timeout (default 1s) |

### `vhost` mode

| Flag | Meaning |
|------|---------|
| `-u, --url <url>` | Usually the IP address of the target |
| `--domain <domain>` | Domain to append when the URL is an IP |
| `--append-domain` | Append the URL's domain to each wordlist entry |
| `--exclude-length <list>` | Exclude response sizes |
| `-r, --follow-redirect` | Follow redirects |
| `-k, --no-tls-validation` | Skip TLS verification |
| `-H, --headers <header>` | Extra headers |
| `--timeout <duration>` | HTTP timeout (default 10s) |

### `fuzz` mode

| Flag | Meaning |
|------|---------|
| `-u, --url <url>` | URL containing `FUZZ` |
| `-B, --body <body>` | Request body containing `FUZZ` |
| `-H, --headers <header>` | Header containing `FUZZ` |
| `-m, --method <verb>` | HTTP method |
| `-b, --excludestatuscodes <list>` | Excluded status codes (ranges allowed) |
| `--exclude-length <list>` | Excluded response sizes |

### Other modes

| Mode | Purpose |
|------|---------|
| `gcs` | Enumerate Google Cloud Storage buckets |
| `s3` | Enumerate AWS S3 buckets |
| `tftp` | Enumerate files on a TFTP server |
| `completion` | Generate shell completion scripts |
| `version` | Print the version |

## Recipes

Authorized targets only.

**1. Directory discovery with extensions and full URLs.**

```bash
gobuster dir -u https://target -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,html,txt -e -t 20 -o evidence/dirs.txt
```

**2. Backup-file discovery.** `-d` appends common backup extensions to each
word.

```bash
gobuster dir -u https://target -w words.txt -d -e -t 20
```

**3. Filter by response size on a soft-404 site.** gobuster has no
auto-calibration; find the bad size with `curl` first.

```bash
curl -sS -o /dev/null -w '%{size_download}\n' https://target/nonexistent-12345
gobuster dir -u https://target -w words.txt --exclude-length 1234
```

**4. Subdomain enumeration with CNAMEs and IPs.**

```bash
gobuster dns -d example.com -w subdomains.txt -c -i -t 20 -o evidence/dns.txt
```

**5. Vhost discovery against one IP.** `--append-domain` turns `admin` into
`admin.example.com`.

```bash
gobuster vhost -u https://10.10.10.5 -w vhosts.txt --append-domain -k
```

**6. Fuzz a query parameter.**

```bash
gobuster fuzz -u 'https://target/?id=FUZZ' -w values.txt -b 404,500
```

**7. Authenticated discovery.**

```bash
gobuster dir -u https://target/admin -w words.txt \
  -c 'SESSION=abc123' -H 'X-Requested-With: XMLHttpRequest' -k
```

**8. Resume a long run.**

```bash
gobuster dir -u https://target -w big.txt -t 10 -o evidence/dirs.txt
gobuster dir -u https://target -w big.txt --wordlist-offset 50000 -o evidence/dirs-2.txt
```

## Output & pipelines

- Default output is one line per hit: status code, size, and path (or name).
- `-e` prints full URLs; `-n` and `--hide-length` trim the columns.
- `-o` writes the same lines to a file; it does not add a header.
- `-q` removes the banner, which makes the output easy to `awk`/`grep`.
- `dns` output includes the resolved name and, with `-c`/`-i`, CNAME/IP data.
- `--wordlist-offset` is a byte/line offset into the wordlist; keep the same
  wordlist and mode when resuming.

## Gotchas & operational security

- **No auto-calibration.** On a soft-404 site, `dir` reports every word as a
  hit until you set `--exclude-length` or a status blacklist.
- **`-b` overrides `-s`.** The default blacklist is `404`; if you set positive
  codes, remember the blacklist still applies.
- **`vhost` needs the IP in `-u`.** Using the hostname makes every request look
  like the same vhost; use `--domain`/`--append-domain` to build names.
- **`dns` wildcard records produce endless hits.** gobuster warns and stops;
  `--wildcard` forces it to continue, which is rarely useful.
- **Threads and delay are the only rate controls.** There is no `--rate`; use
  `-t` and `--delay`.
- **TLS errors are common on internal targets.** `-k` is usually required, but
  record that you disabled verification.
- **Extensions multiply requests.** `-x php,html,bak` triples the wordlist;
  scope it.
- **Output can include sensitive paths.** Store it in the engagement directory.

## Legal & safety

Use gobuster only on systems you own or have **explicit written authorization**
to test. Directory, DNS, and vhost enumeration generate real requests and can
trigger alerts, rate limits, or account lockouts. Cloud bucket enumeration can
cross into third-party accounts; verify ownership before testing `s3`/`gcs`
names.

## Related tools

- [`ffuf/`](../ffuf/README.md) — flexible matchers, auto-calibration, recursion.
- [`dirsearch/`](../dirsearch/README.md) — Python content discovery with reports.
- [`httpx/`](../httpx/README.md) — probe and fingerprint discovered hosts.
- [`nuclei/`](../nuclei/README.md) — template checks on discovered endpoints.
- [`subfinder/`](../subfinder/README.md) — passive subdomain sources before `dns`.
- [`wfuzz/`](../wfuzz/README.md) — another fuzzing engine for comparison.

## References

- Installed help snapshots: [`help.txt`](help.txt), [`help-dir.txt`](help-dir.txt),
  [`help-dns.txt`](help-dns.txt), [`help-vhost.txt`](help-vhost.txt),
  [`help-fuzz.txt`](help-fuzz.txt)
- Upstream README: https://github.com/OJ/gobuster
- Kali tool page: https://www.kali.org/tools/gobuster/

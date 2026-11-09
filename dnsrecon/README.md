# dnsrecon — DNS enumeration and zone reconnaissance

- **Version:** DNSRecon 1.2.0 (`dnsrecon --version`)
- **Binary:** `/usr/bin/dnsrecon` (Debian package `dnsrecon`)
- **Data:** `/usr/share/dnsrecon/dnsrecon/data/` (`namelist.txt`, `snoop.txt`,
  `subdomains-top1mil*.txt`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/darkoperator/dnsrecon
- **Author/docs:** https://www.darkoperator.com

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

dnsrecon enumerates DNS records and infrastructure for a domain. It performs
standard record enumeration, zone-transfer attempts, dictionary brute force,
reverse lookups, SRV discovery, DNSSEC zone walking, cache snooping, TLD
expansion, and optional OSINT collection from crt.sh, Bing, and Yandex. It can
write results to SQLite, XML, CSV, or JSON.

It is a **reconnaissance** tool. It does not exploit anything, but brute force
and zone-transfer attempts are visible and can be rate-limited or blocked.

## Enumeration types

| Type | What it does |
|------|--------------|
| `std` | SOA, NS, A, AAAA, MX, SRV (default) |
| `rvl` | Reverse lookup of a CIDR or IP range |
| `brt` | Brute force subdomains/hosts from a dictionary |
| `srv` | SRV record enumeration |
| `axfr` | Test all NS servers for a zone transfer |
| `bing` | Bing search for subdomains and hosts |
| `yand` | Yandex search for subdomains and hosts |
| `crt` | crt.sh certificate-transparency search |
| `snoop` | DNS cache snooping against all NS servers |
| `tld` | Remove the TLD and test against IANA TLDs |
| `zonewalk` | DNSSEC NSEC zone walk |

The standard enumeration can be extended with `-a` (AXFR), `-s` (SPF reverse
lookups), `-b` (Bing), `-y` (Yandex), `-k` (crt.sh), `-w` (deep whois), and
`-z` (DNSSEC zone walk).

## Flag reference

### Target and enumeration

| Flag | Meaning |
|------|---------|
| `-d DOMAIN` | Target domain |
| `-n NS_SERVER` | Name server to use; comma-separated list allowed |
| `-r RANGE` | IP range for reverse brute force (`first-last` or `range/bitmask`) |
| `-D DICTIONARY` | Dictionary for subdomain/host brute force |
| `-t TYPE` | Enumeration type (see table above) |
| `-a` | Also attempt AXFR during standard enumeration |
| `-s` | Reverse-lookup IPv4 ranges in SPF records |
| `-b` | Bing enumeration during standard enumeration |
| `-y` | Yandex enumeration during standard enumeration |
| `-k` | crt.sh enumeration during standard enumeration |
| `-w` | Deep whois analysis and reverse lookup |
| `-z` | DNSSEC zone walk during standard enumeration |

### Brute force and query behavior

| Flag | Meaning |
|------|---------|
| `-f` | Filter wildcard-resolving records when saving |
| `--iw` | Continue brute force even if a wildcard is found |
| `--threads NUM` | Threads for reverse/forward/brute-force/SRV |
| `--lifetime SEC` | Query timeout (default 3.0 s) |
| `--tcp` | Use TCP for queries |
| `--disable_check_recursion` | Skip recursion check on name servers |
| `--disable_check_bindversion` | Skip BIND version check |

### Output

| Flag | Meaning |
|------|---------|
| `--db FILE` | SQLite 3 database |
| `-x FILE`, `--xml FILE` | XML |
| `-c FILE`, `--csv FILE` | CSV |
| `-j FILE`, `--json FILE` | JSON |

### Misc

| Flag | Meaning |
|------|---------|
| `-V`, `--version` | Version |
| `-v`, `--verbose` | Verbose |
| `-h`, `--help` | Help |

## Recipes

**1. Standard enumeration.**

```bash
dnsrecon -d target.example -t std
```

**2. Standard plus zone transfer and crt.sh.**

```bash
dnsrecon -d target.example -t std -a -k
```

**3. Zone transfer only.**

```bash
dnsrecon -d target.example -t axfr
```

**4. Brute force with a dictionary.**

```bash
dnsrecon -d target.example -t brt \
  -D /usr/share/dnsrecon/dnsrecon/data/subdomains-top1mil-5000.txt
```

**5. Reverse lookup of a CIDR.**

```bash
dnsrecon -r 192.168.10.0/24 -t rvl
```

**6. SRV records.**

```bash
dnsrecon -d target.example -t srv
```

**7. DNSSEC zone walk.**

```bash
dnsrecon -d target.example -t zonewalk
```

**8. OSINT sources.**

```bash
dnsrecon -d target.example -t crt
dnsrecon -d target.example -t bing
dnsrecon -d target.example -t yand
```

**9. Save everything.**

```bash
dnsrecon -d target.example -t std -j ~/dnsrecon-demo/results.json \
  --db ~/dnsrecon-demo/results.db -c ~/dnsrecon-demo/results.csv
```

## Gotchas

- **Zone transfers are rare.** `-a` and `-t axfr` are worth trying, but a
  refusal is the normal result, not a failure of the tool.
- **Wildcards poison brute force.** Check for a wildcard record first; `-f`
  filters saved records, and `--iw` continues despite a wildcard.
- **`-D` is required for `-t brt`.** There is no implicit dictionary for brute
  force in this build; the data files are a starting point, not a default.
- **Bing, Yandex, and crt.sh are external and rate-limited.** Failures are often
  quota or network issues, not evidence that no subdomains exist.
- **`-z` only works on DNSSEC-signed zones with NSEC**, not NSEC3.
- **`--db` is SQLite 3.** It is not a generic "database" flag.
- **`-f` affects saving, not querying.** It does not stop wildcard responses
  from being requested.
- **Use `--tcp` when UDP is filtered or truncated.** Large responses and some
  middleboxes need it.
- **Respect the domain's DNS rate limits.** Brute force with high `--threads`
  can look like an attack.

## Related docs

- [`nmap`](../nmap/README.md) — port and service discovery on resolved hosts
- [`theharvester`](../theharvester/README.md) — OSINT email and host collection
- [`subfinder`](../subfinder/README.md) — passive subdomain enumeration
- [`httpx`](../httpx/README.md) — probe discovered hosts
- [`whois`](../whois/README.md) — registration and abuse contacts

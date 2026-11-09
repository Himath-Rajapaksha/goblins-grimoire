# DNSRecon — command scenarios for deep research

Installed build: DNSRecon 1.2.0 (Debian package `dnsrecon`). Every flag below
appears in [`help.txt`](help.txt). The shipped data files are under
`/usr/share/dnsrecon/dnsrecon/data/`.

> **Intrusive:** brute force, zone-transfer attempts, and TLD expansion generate
> real DNS traffic. Use only on domains you are authorized to test, and respect
> resolver rate limits.

## How to invoke

```text
@deep-research For an authorized external DNS assessment, compare DNSRecon 1.2.0
with subfinder, amass, and dig for standard enumeration, zone-transfer checks,
and subdomain brute force. Which method gives the best coverage per query, how
do wildcard records and NSEC3 change the result, and what evidence distinguishes
a real subdomain from a wildcard? Cite the DNSRecon source, RFC 4034/5155, and
field reports; return a phased plan with a query budget.
```

Create the demo workspace, replacing the domain with an in-scope target:

```bash
mkdir -p ~/dnsrecon-demo/evidence
printf 'target.example\n' > ~/dnsrecon-demo/domains.txt
```

## Scenario 1 — standard enumeration

The default type is `std`: SOA, NS, A, AAAA, MX, and SRV. Record the resolver
used, because results differ between recursive resolvers and authoritative
servers.

| # | Command | How to use |
|---|---------|------------|
| 1 | `dnsrecon -d target.example -t std` | Default record set |
| 2 | `dnsrecon -d target.example -t std -v` | Verbose |
| 3 | `dnsrecon -d target.example -t std -n 8.8.8.8` | Query a specific resolver |
| 4 | `dnsrecon -d target.example -t std -n ns1.target.example,ns2.target.example` | Query authoritative servers |
| 5 | `dnsrecon -d target.example -t std --tcp` | Force TCP |
| 6 | `dnsrecon -d target.example -t std --lifetime 5` | Longer query timeout |

**Research questions**
- Which records come from the recursive resolver versus the authoritative NS?
- How does `--tcp` change truncated responses?
- Which of these records are in scope for follow-on scanning?
- What does the recursion check protect against?

**Sources:** RFC 1034/1035; RFC 6891 (EDNS); DNSRecon source.
**Output:** a record table with source resolver and TTL.

## Scenario 2 — zone transfer

A zone transfer is the highest-value DNS result, and the least likely to
succeed. Try it explicitly and as part of standard enumeration.

| # | Command | How to use |
|---|---------|------------|
| 7 | `dnsrecon -d target.example -t axfr` | Test all NS servers |
| 8 | `dnsrecon -d target.example -t std -a` | Standard plus AXFR |
| 9 | `dnsrecon -d target.example -t axfr -n ns1.target.example` | One name server |
| 10 | `dnsrecon -d target.example -t std --disable_check_recursion --disable_check_bindversion` | Skip pre-checks |

**Research questions**
- Which NS servers are authoritative, and does each refuse for the same reason?
- What does a successful AXFR reveal that ordinary queries do not?
- Why would a target disable the recursion or BIND-version checks?
- How do you record a refusal as evidence without overstating it?

**Sources:** RFC 5936 (AXFR); RFC 1035; DNSRecon source.
**Output:** a per-NS transfer result and a captured zone file if successful.

## Scenario 3 — brute force and wildcards

Brute force needs a dictionary. Check for a wildcard first, or every candidate
will look valid.

| # | Command | How to use |
|---|---------|------------|
| 11 | `dnsrecon -d target.example -t brt -D /usr/share/dnsrecon/dnsrecon/data/subdomains-top1mil-5000.txt` | 5,000-word list |
| 12 | `dnsrecon -d target.example -t brt -D /usr/share/dnsrecon/dnsrecon/data/namelist.txt` | Small name list |
| 13 | `dnsrecon -d target.example -t brt -D /usr/share/dnsrecon/dnsrecon/data/subdomains-top1mil-20000.txt --threads 10` | Larger list, more threads |
| 14 | `dnsrecon -d target.example -t brt -D ~/dnsrecon-demo/words.txt -f` | Filter wildcard records when saving |
| 15 | `dnsrecon -d target.example -t brt -D ~/dnsrecon-demo/words.txt --iw` | Continue despite a wildcard |
| 16 | `dnsrecon -d target.example -t brt -D ~/dnsrecon-demo/words.txt --threads 10 --lifetime 2 -j ~/dnsrecon-demo/evidence/brt.json` | Throttled, saved |

**Research questions**
- How do you detect a wildcard record before brute force?
- Does `-f` filter queries or only saved records?
- What is the query cost of each shipped wordlist?
- When is `--iw` justified, and how do you separate real hits from wildcard
  noise?

**Sources:** DNSRecon source; wordlist line counts; RFC 4592 (wildcards).
**Output:** a wildcard baseline and a deduplicated subdomain list.

## Scenario 4 — reverse, SRV, TLD, and cache snooping

These modes answer different questions: what is in a netblock, what services a
domain advertises, what other TLDs exist, and what a resolver has cached.

| # | Command | How to use |
|---|---------|------------|
| 17 | `dnsrecon -r 192.168.10.0/24 -t rvl` | Reverse a CIDR |
| 18 | `dnsrecon -r 192.168.10.10-192.168.10.50 -t rvl` | Reverse an address range |
| 19 | `dnsrecon -d target.example -t srv` | SRV records |
| 20 | `dnsrecon -d target.example -t tld` | Test other TLDs |
| 21 | `dnsrecon -d target.example -t snoop -D /usr/share/dnsrecon/dnsrecon/data/snoop.txt` | Cache snooping |
| 22 | `dnsrecon -d target.example -t snoop -D /usr/share/dnsrecon/dnsrecon/data/snoop.txt --threads 5` | Snoop with limited threads |

**Research questions**
- Which PTR records are meaningful, and which are generic ISP names?
- Which SRV records reveal internal services or federated endpoints?
- How many queries does `-t tld` generate, and is that in scope?
- What does a cache-snoop hit prove, and what does a miss fail to prove?

**Sources:** RFC 1035 (PTR); RFC 2782 (SRV); RFC 2308 (negative caching);
DNSRecon source.
**Output:** a reverse-DNS map, SRV inventory, TLD comparison, and snoop result.

## Scenario 5 — OSINT sources and DNSSEC

crt.sh, Bing, and Yandex find names that brute force will not. DNSSEC zone
walking only works on NSEC zones.

| # | Command | How to use |
|---|---------|------------|
| 23 | `dnsrecon -d target.example -t crt` | Certificate-transparency search |
| 24 | `dnsrecon -d target.example -t bing` | Bing search |
| 25 | `dnsrecon -d target.example -t yand` | Yandex search |
| 26 | `dnsrecon -d target.example -t std -k -b -y` | Standard plus all three sources |
| 27 | `dnsrecon -d target.example -t std -w` | Deep whois and reverse lookup |
| 28 | `dnsrecon -d target.example -t std -s` | Reverse SPF ranges |
| 29 | `dnsrecon -d target.example -t zonewalk` | DNSSEC NSEC walk |
| 30 | `dnsrecon -d target.example -t std -z` | Standard plus zone walk |

**Research questions**
- Which OSINT source has the best coverage and the worst rate limit?
- How do you deduplicate names from crt.sh, Bing, and Yandex?
- Why does `zonewalk` fail on an NSEC3 zone, and what replaces it?
- What does `-w` add that a separate whois query does not?

**Sources:** RFC 4034 (NSEC); RFC 5155 (NSEC3); crt.sh API docs; Bing/Yandex
search documentation; DNSRecon source.
**Output:** a merged name list with source attribution and a DNSSEC result.

## Scenario 6 — output, database, and reporting

Store results in a format that can be queried later. JSON is easiest to merge;
SQLite is easiest to deduplicate across runs.

| # | Command | How to use |
|---|---------|------------|
| 31 | `dnsrecon -d target.example -t std -j ~/dnsrecon-demo/evidence/results.json` | JSON |
| 32 | `dnsrecon -d target.example -t std -c ~/dnsrecon-demo/evidence/results.csv` | CSV |
| 33 | `dnsrecon -d target.example -t std -x ~/dnsrecon-demo/evidence/results.xml` | XML |
| 34 | `dnsrecon -d target.example -t std --db ~/dnsrecon-demo/evidence/results.db` | SQLite |
| 35 | `sqlite3 ~/dnsrecon-demo/evidence/results.db '.tables'` | Inspect the schema |
| 36 | `jq '.[] \| select(.type=="A")' ~/dnsrecon-demo/evidence/results.json` | Filter JSON |
| 37 | `shred -u ~/dnsrecon-demo/words.txt` | Remove the demo wordlist |

**Research questions**
- Which output format preserves the record type, name, address, and source?
- How do you merge results from two runs without duplicates?
- Which fields are safe to include in a client report?
- What does the SQLite schema store that JSON does not?

**Sources:** DNSRecon source; SQLite documentation; jq manual.
**Output:** a reproducible result set with a schema note and a deduplicated
subdomain inventory.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-d`, `-n`, `-r`, `-D`, `-t` | 1–5 |
| `-a`, `-s`, `-b`, `-y`, `-k`, `-w`, `-z` | 2, 5 |
| `-f`, `--iw` | 3 |
| `--threads`, `--lifetime`, `--tcp` | 1, 3, 4 |
| `--disable_check_recursion`, `--disable_check_bindversion` | 2 |
| `--db`, `-x`, `-c`, `-j` | 6 |
| `-V`, `-v`, `-h` | 1 |

Defaults: type `std`, query lifetime 3.0 s. Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Standard enumeration | 6 |
| 2 | Zone transfer | 4 |
| 3 | Brute force and wildcards | 6 |
| 4 | Reverse, SRV, TLD, snooping | 6 |
| 5 | OSINT and DNSSEC | 8 |
| 6 | Output and reporting | 7 |
| **Total** | | **37** |

## Legal and safety

DNS enumeration is visible to the target's resolvers and may violate a
provider's acceptable-use policy if run at high volume. Use only on authorized
domains, keep `--threads` modest, and treat external OSINT sources as
rate-limited third parties. A zone-transfer refusal is not a vulnerability; do
not report it as one.

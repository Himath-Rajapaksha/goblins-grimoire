# testssl — TLS/SSL configuration and vulnerability scanner

- **Version:** 3.2.1
- **Binary:** `/usr/bin/testssl` (distribution entry point, not `testssl.sh`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/testssl/testssl.sh
- **Docs:** https://testssl.sh/ · https://testssl.sh/doc/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

testssl is a Bash + OpenSSL scanner for one TLS endpoint at a time. It walks the
handshake from the outside: accepted protocols, ciphers offered per protocol,
forward secrecy, default and preferred picks, the certificate chain, and a
catalogue of named flaws (Heartbleed, CCS, Ticketbleed, ROBOT, renegotiation,
CRIME, BREACH, POODLE, SWEET32, BEAST, LUCKY13, Winshock, FREAK, LOGJAM, DROWN,
RC4). It is diagnostic, not exploitative, but opens dozens to hundreds of
connections per target, so it is visible and can trip IDS/WAF rate rules.

## Use cases

| Situation | Why testssl |
|-----------|-------------|
| Pre-launch TLS baseline | One command covers protocols, ciphers, certificate, common flaws |
| Compliance evidence | CSV/JSON/HTML findings for a report |
| Legacy protocol hunt | Find SSLv3, TLS 1.0, TLS 1.1 still enabled |
| Certificate inventory | Chain, expiry, key size, SANs, trust problems |
| Mail/LDAP/XMPP or estate sweep | STARTTLS modes; `--file` mass testing, serial or parallel |

## Requirements & install layout

- Installed from the distribution package manager; entry point `/usr/bin/testssl`.
  The script finds its mapping/CA data relative to its install root, so do not
  copy the binary elsewhere on its own.
- Runtime needs a POSIX userland plus `bash` and `openssl`; the banner reports
  which OpenSSL is used, and TLS 1.3 coverage depends on that build.
- No root or special hardware. Outbound TCP to the target port, plus DNS unless
  `--nodns`; `--phone-out` also needs egress to CRL/OCSP responders.
- No config file: tuning options can be preset via environment variables (per
  the help header). Output files land in the cwd; `--debug` uses `/tmp`.

## Basic syntax

```bash
testssl example.com                        # HTTPS on 443, default run
testssl https://example.com:8443           # non-default port
testssl --quiet --color 0 example.com      # no banner/colour, log-safe
testssl -t smtp mail.example.com:25        # STARTTLS service
testssl -p -s -f -h example.com            # protocols + cipher classes + FS + headers
testssl --file targets.txt --mode parallel # mass run
```

`<URI>` is `host | host:port | URL | URL:port`, defaults to 443, and must be the
**last** parameter. Value-taking options also accept `=`, e.g.
`testssl -t=smtp --wide --openssl=/usr/bin/openssl <URI>`.

## Flag reference

### Informational
`--help` prints help; `-b, --banner` and `-v, --version` show banner and version;
`-V, --local [pattern]` pretty-prints the local OpenSSL cipher list, optionally
filtered by a case-insensitive word pattern.

### Target and mass testing
| Flag | Meaning |
|------|---------|
| `<URI>` | Target host/URL, last argument |
| `-t, --starttls <protocol>` | STARTTLS: ftp, smtp, lmtp, pop3, imap, xmpp, xmpp-server, telnet, ldap, nntp, sieve, postgres, mysql |
| `--xmpphost <domain>` | Domain for the XMPP `to` attribute (like SNI) |
| `--mx <domain/host>` | Test MX records high to low priority on port 25 |
| `--file, -iL <fname>` | One command line per line; accepts nmap greppable (`-oG`); implies `--warnings batch` |
| `--mode <serial\|parallel>` | Mass-testing order (`--parallel` is a shortcut) |
| `--warnings <batch\|off>` | `batch` stops on a testing error; `off` continues and skips warnings |
| `--connect-timeout` / `--openssl-timeout <s>` | Max wait for TCP connect / before an openssl connect is killed |

### Single checks (default run does everything except `-E` and `-g`)
| Flag | Meaning |
|------|---------|
| `-e, --each-cipher` | Check each local cipher remotely |
| `-E, --cipher-per-proto` | Check ciphers per protocol |
| `-s, --std, --categories` | Standard cipher categories by strength |
| `-f, --fs, --forward-secrecy` | Forward secrecy settings |
| `-p, --protocols` | TLS/SSL protocols (including SPDY/HTTP2) |
| `-g, --grease` | GREASE and size-limit implementation bugs |
| `-S, --server-defaults` | Server default picks plus certificate info |
| `-P, --server-preference` | Server picks: protocol + cipher |
| `-x, --single-cipher <pattern>` | Test ciphers matching a number or word |
| `-c, --client-simulation` | Simulate clients to see what negotiates |
| `-h, --header, --headers` | HSTS, HPKP, banners, security headers, cookies, proxy, IPv4 |

### Vulnerability checks (all of them with `-U, --vulnerable`)
| Flag | Meaning |
|------|---------|
| `-H, --heartbleed` | Heartbleed |
| `-I, --ccs, --ccs-injection` | CCS injection |
| `-T, --ticketbleed` | Ticketbleed in BigIP load balancers |
| `--BB, --robot` | ROBOT (Bleichenbacher oracle) |
| `--SI, --starttls-injection` | STARTTLS injection issues |
| `-R, --renegotiation` | Renegotiation vulnerabilities |
| `-C, --compression, --crime` | CRIME (TLS compression) |
| `-B, --breach` | BREACH (HTTP compression) |
| `-O, --poodle` | POODLE (SSL) |
| `-Z, --tls-fallback` | TLS_FALLBACK_SCSV mitigation |
| `-W, --sweet32` | SWEET32 (3DES, RC2, IDEA) |
| `-A, --beast` | BEAST |
| `-L, --lucky13` | LUCKY13 |
| `-WS, --winshock` | Winshock |
| `-F, --freak` | FREAK |
| `-J, --logjam` | LOGJAM |
| `-D, --drown` | DROWN |
| `-4, --rc4, --appelbaum` | Which RC4 ciphers are offered |

### Tuning and connectivity
| Flag | Meaning |
|------|---------|
| `-9, --full` | Implementation-bug tests plus cipher-per-protocol; help notes it "could disappear" |
| `--bugs` | Enable `s_client -bugs`, needed for some buggy F5 devices |
| `--assume-http` | If the protocol check fails, assume HTTP and enforce HTTP checks |
| `--ssl-native` | Use OpenSSL where sockets are normally used: faster but inaccurate |
| `--openssl <PATH>` | Use a specific openssl binary (default: `$PATH`, then `$RUN_DIR`) |
| `--proxy <host:port\|auto>` | Experimental proxy; `auto` reads `$http(s)_proxy` |
| `-6` | Also use IPv6 (needs supporting OpenSSL and connectivity) |
| `--ip <ip>` | Test this IP instead of resolving; `one` = first DNS answer; `proxy` = resolve via proxy |
| `-n, --nodns <min\|none>` | `none` = no DNS; `min` = A, AAAA, MX |
| `--sneaky` | Fewer traces in target logs: user agent, referer |
| `--user-agent <ua>` | Custom user agent instead of the standard one |
| `--ids-friendly` | Skip a few checks that commonly trigger IDS blocks |
| `--phone-out` | Allow external CRL download and OCSP queries |
| `--add-ca <files\|dir>` | Extra CA files (comma-separated) or a directory of `*.pem` |
| `--mtls <cert file>` | Client certificate PEM including its unencrypted key (beta) |
| `--basicauth <user:pass>` | HTTP basic authentication |
| `--reqheader <header>` | Extra HTTP request header |

### Screen and file output
| Flag | Meaning |
|------|---------|
| `--quiet` | Do not print the banner |
| `--wide` | Wide output for RC4, BEAST, FS (hex code, kx, strength, RFC name) |
| `--show-each` | With wide output, show every cipher tested, not only successes |
| `--mapping <openssl\|iana\|rfc\|no-openssl\|no-iana\|no-rfc>` | Primary cipher-name family |
| `--color <0\|1\|2\|3>` | 0 no escapes, 1 b/w escapes, 2 colour (default), 3 extra colour |
| `--colorblind` | Swap green and blue |
| `--debug <0-6>` | Debug level; debug output kept in `/tmp` |
| `--disable-rating` | Disable rating output |
| `--log, --logging` / `--logfile, -oL <file\|dir>` | Log stdout, auto-named in cwd / explicit path |
| `--json` / `--jsonfile, -oj <file\|dir>` | Flat JSON findings, auto-named / explicit |
| `--json-pretty` / `--jsonfile-pretty, -oJ <file\|dir>` | Structured JSON, auto-named / explicit |
| `--csv` / `--csvfile, -oC <file\|dir>` | CSV findings, auto-named / explicit |
| `--html` / `--htmlfile, -oH <file\|dir>` | HTML report, auto-named / explicit |
| `--out(f,F)ile, -oa, -oA <name>` | Log + JSON + CSV + HTML at once; `-oa`/`-oA` choose pretty/flat JSON |
| `--hints` | Extra hints on findings |
| `--severity <LOW\|MEDIUM\|HIGH\|CRITICAL>` | Filter lower severities out of CSV/JSON |
| `--append` / `--overwrite` | Append to an existing file without header / overwrite without warning |
| `--outprefix <prefix>` | Prepend a prefix to auto-generated filenames |

## Recipes

```bash
# Default audit, machine-readable for a report pipeline
testssl --quiet --jsonfile-pretty tls.json example.com

# Fast triage: protocols, cipher classes, forward secrecy, headers
testssl -p -s -f -h example.com

# Full vulnerability pass (noisy; many handshakes)
testssl -U --hints example.com

# STARTTLS mail server, JSON output
testssl -t smtp --quiet --jsonfile smtp.json mail.example.com:25

# Mass scan an nmap greppable file, parallel, batch warnings
testssl --file scan.gnmap --mode parallel --warnings batch

# All formats at once, severity-filtered JSON/CSV, fixed IP with hostname SNI
testssl -oA tls-audit --severity MEDIUM example.com
testssl --ip 203.0.113.10 --add-ca corp-ca.pem https://app.example.com

# IDS-conscious scan through a proxy with short timeouts
testssl --ids-friendly --sneaky --connect-timeout 10 \
  --proxy 127.0.0.1:8080 example.com
```

## Output & pipelines

- Screen output is a colour-coded table with a severity rating per finding;
  `--color 0` and `--quiet` make it log-safe.
- `--json` is a *flat* list; `--json-pretty` is structured. Both are written in
  addition to stdout, never instead of it.
- `--severity` filters only CSV/JSON, not the screen. `-oA`/`-oa` emit log,
  JSON, CSV, and HTML from one basename.
- Auto names follow `${NODE}-p${port}${YYYYMMDD-HHMM}`; `--outprefix` keeps
  batch runs sorted. Parsed findings cross-check well against `sslyze`; target
  lists come from `httpx` and go in through `--file`.

## Gotchas & operational security

- **Default run is not everything.** `-E` (cipher per protocol) and `-g`
  (GREASE) are excluded unless requested or `-9` is used.
- **There is no rate-limit flag.** The only global control is
  `--mode serial|parallel` during mass testing; `-U`, `-e`, `-E` create the most
  traffic.
- **`--ssl-native` is a trade-off**: faster, less accurate. Keep the default.
- **SNI comes from the URI**; use `--ip` to pin an address behind a load
  balancer while keeping the hostname.
- **`--phone-out` leaks target hostnames** to third-party CRL/OCSP services;
  leave it off for sensitive engagements. `--sneaky` trims only UA/referer, not
  handshake volume.
- **`--warnings batch` aborts on the first testing error**; `off` continues.
  `--append`/`--overwrite` decide repeat-run behaviour.
- **STARTTLS targets need `-t`**; a plain TLS scan of port 25 reports a broken
  handshake, not a broken server.
- Help does not document exit codes; do not gate CI on exit status alone. Parse
  JSON for the specific finding instead.

## Legal & safety

Run testssl only against systems you own or have **written authorization** to
test. Active TLS scanning opens many connections, probes vulnerability checks
(including Heartbleed and STARTTLS injection), and can be logged as an attack.
`--phone-out` contacts external services. Respect scope, rate expectations, and
change windows; never scan shared hosting or third-party load balancers you were
not authorized to touch.

## Related tools

- [`../sslyze/`](../sslyze/README.md) — targeted checks via an independent TLS
  stack and cleaner JSON automation.
- [`../httpx/`](../httpx/README.md) — build the HTTPS target list first.
- [`../nuclei/`](../nuclei/README.md) — template checks that complement TLS
  findings; [`../burp/`](../burp/README.md) and [`../zap/`](../zap/README.md)
  once the transport layer is understood.

## References

- Documentation: https://testssl.sh/doc/
- Project and release notes: https://github.com/testssl/testssl.sh
- Local truth for this build: [`help.txt`](help.txt)

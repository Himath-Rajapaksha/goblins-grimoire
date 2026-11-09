# SSLyze — fast TLS/SSL configuration scanner

- **Version:** SSLyze 6.3.1
- **Binary:** `/usr/local/bin/sslyze` (pipx)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/nabla-c0d3/sslyze
- **Docs:** https://nabla-c0d3.github.io/sslyze/documentation/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

SSLyze is a Python TLS scanner built on a custom OpenSSL wrapper. It checks
protocol support, cipher suites, certificate validity and chain, forward
secrecy, session resumption, HTTP security headers, and specific
vulnerabilities (Heartbleed, ROBOT, CCS injection, renegotiation). It is faster
and more script-friendly than testssl, and emits clean JSON through a stable
schema.

## Use cases

| Situation | Why SSLyze |
|-----------|------------|
| TLS baseline | Protocol/cipher/certificate audit in one run |
| JSON automation | Stable schema for CI and dashboards |
| Compliance checks | Mozilla modern/intermediate/old profiles |
| Certificate validation | Chain, expiry, SANs, trust store |
| Vulnerability checks | Heartbleed, ROBOT, CCS, renegotiation |
| STARTTLS | SMTP, IMAP, POP3, FTP, LDAP, RDP, postgres |
| Client certificate testing | mTLS with `--cert`/`--key` |

## Requirements & install layout

- Installed via pipx; entry point `/usr/local/bin/sslyze`.
- Python 3.13 environment; no external openssl needed for most checks.
- `--update_trust_stores` downloads fresh CA stores (network access).
- JSON output follows the documented ScanResult schema.

## Basic syntax

```bash
# default scan
sslyze example.com

# JSON output
sslyze --json_out tls.json example.com

# protocol and certificate checks
sslyze --tlsv1 --tlsv1_1 --tlsv1_2 --tlsv1_3 --certinfo example.com

# vulnerability checks
sslyze --heartbleed --robot --openssl_ccs --reneg example.com

# STARTTLS mail server
sslyze --starttls smtp mail.example.com:25

# several targets from a file
sslyze --targets_in targets.txt --json_out results.json

# compare against Mozilla profiles
sslyze --mozilla_config modern example.com
```

## Flag reference

### Input and output
| Flag | Meaning |
|------|---------|
| `target ...` | Hosts (`host`, `host:port`, `[ipv6]:port`) |
| `--targets_in <file>` | One `host:port` per line |
| `--json_out <file\\|->` | JSON output (`-` = stdout) |
| `--quiet` | No stdout (with JSON output) |
| `--update_trust_stores` | Download latest CA stores and exit |

### Trust and client certs
| Flag | Meaning |
|------|---------|
| `--cert <file>` / `--key <file>` | Client certificate chain and key |
| `--keyform <PEM\\|DER>` | Key format |
| `--pass <pass>` | Key passphrase |
| `--certinfo_ca_file <file>` | Custom CA bundle for validation |

### Connectivity
| Flag | Meaning |
|------|---------|
| `--slow_connection` | Fewer concurrent connections |
| `--https_tunnel <url>` | HTTP CONNECT proxy |
| `--starttls <proto>` | `auto, smtp, xmpp, xmpp_server, pop3, imap, ftp, ldap, rdp, postgres` |
| `--xmpp_to <host>` | XMPP `to` attribute |
| `--sni <host>` | SNI hostname |
| `--mozilla_config <modern\\|intermediate\\|old\\|disable>` | Mozilla profile check (default intermediate) |
| `--custom_tls_config <file>` | Custom Mozilla-format TLS config |

### Scan commands
| Flag | Meaning |
|------|---------|
| `--tlsv1`, `--tlsv1_1`, `--tlsv1_2`, `--tlsv1_3` | Protocol support |
| `--sslv2`, `--sslv3` | Legacy protocol support |
| `--certinfo` | Certificate chain and validity |
| `--elliptic_curves` | Supported curves |
| `--http_headers` | Security headers |
| `--compression` | TLS compression (CRIME) |
| `--early_data` | TLS 1.3 0-RTT |
| `--ems` | Extended Master Secret |
| `--fallback` | TLS_FALLBACK_SCSV |
| `--heartbleed` | Heartbleed |
| `--robot` | ROBOT |
| `--openssl_ccs` | CCS injection |
| `--reneg` | Insecure renegotiation |
| `--resum` / `--resum_attempts <n>` | Session resumption |
| `-h, --help` | Help |

With no scan command, SSLyze queues its default set of common checks.

## Recipes

```bash
# 1. Default scan with JSON
sslyze --json_out tls.json example.com

# 2. Protocol matrix only
sslyze --sslv2 --sslv3 --tlsv1 --tlsv1_1 --tlsv1_2 --tlsv1_3 example.com

# 3. Certificate deep dive
sslyze --certinfo --certinfo_ca_file /etc/ssl/certs/ca-certificates.crt example.com

# 4. Vulnerability pass
sslyze --heartbleed --robot --openssl_ccs --reneg --compression example.com

# 5. Mozilla intermediate compliance
sslyze --mozilla_config intermediate example.com

# 6. STARTTLS SMTP
sslyze --starttls smtp mail.example.com:25 --json_out smtp.json

# 7. Multiple targets, quiet JSON
sslyze --targets_in targets.txt --json_out fleet.json --quiet

# 8. mTLS endpoint
sslyze --cert client.pem --key client.key --certinfo internal.example.com:8443

# 9. Slow or rate-limited target
sslyze --slow_connection --json_out out.json example.com

# 10. Extract weak protocols from JSON
jq '.. | objects | select(.status? == "COMPLETED") | .tls_1_0_support? // empty' fleet.json
```

## Output & pipelines

- Console summary with per-target sections.
- `--json_out` follows the ScanResult schema; keys mirror the scan commands.
- JSON is easy to diff across runs and feed into dashboards.
- Results complement `testssl`; cross-check findings between the two.

## Gotchas & operational security

- Different TLS stacks can disagree; a finding is confirmed only when both
  SSLyze and testssl (or a raw client) agree.
- `--mozilla_config` queues extra scans and slows the run.
- `--slow_connection` is essential on fragile targets but multiplies runtime.
- STARTTLS needs the right protocol; a plain TLS scan of port 25 is misleading.
- The JSON schema changes across major versions; pin SSLyze when parsing.
- Client keys used with `--cert` must be handled as secrets.
- `--update_trust_stores` downloads from GitHub; do it deliberately.

## Legal & safety

Only scan systems you own or are authorized to test. TLS scanning opens many
connections and probes known vulnerabilities; it is logged and may trigger
IDS. Respect scope and change windows.

## Related tools

- [`testssl/`](../testssl/README.md) — comprehensive Bash/OpenSSL scanner
- [`httpx/`](../httpx/README.md) — target list building
- [`nuclei/`](../nuclei/README.md) — template-based checks
- [`smuggler/`](../smuggler/README.md), [`corsy/`](../corsy/README.md) — web layer

## References

- https://nabla-c0d3.github.io/sslyze/documentation/
- https://github.com/nabla-c0d3/sslyze

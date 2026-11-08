# nikto — web server vulnerability scanner

- **Version:** Nikto 2.6.1 (LW 2.5)
- **Binary:** `/usr/local/bin/nikto`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/sullo/nikto
- **Docs:** https://github.com/sullo/nikto/wiki

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Nikto is a classic, noisy web-server scanner. It checks for dangerous files,
outdated server software, default installs, misconfigurations, exposed admin
consoles, and known CGI issues. It is not a modern app scanner — no JS rendering,
limited auth handling — but it remains a fast way to catch low-hanging server
misconfigurations.

## Use cases

| Situation | Tuning |
|-----------|--------|
| Quick server hygiene check | default |
| Only interesting files | `-Tuning 1` |
| Misconfigurations/default files | `-Tuning 2` |
| Info disclosure | `-Tuning 3` |
| Injection checks | `-Tuning 4` |
| Auth bypass | `-Tuning a` |
| Software identification | `-Tuning b` |
| Avoid DoS tests | `-Tuning x 6` (all except 6) |
| Through Burp/ZAP | `-useproxy http://127.0.0.1:8080` |
| Authenticated | `-id user:pass` |
| Virtual host | `-vhost app.example.com` |

## Basic syntax

```bash
nikto -host https://example.com
nikto -h example.com -ssl -port 443
nikto -host https://example.com -output scan.html -Format htm
```

## Flag reference

### Target
| Flag | Meaning |
|------|---------|
| `-h, -host, -url` | Target host/URL |
| `-port` | Port (default 80) |
| `-ssl` | Force SSL |
| `-nossl` | Disable SSL |
| `-vhost` | Host header value |
| `-root` | Prepend a path root |
| `-ipv4` / `-ipv6` | Force IP family |

### Scan tuning
| Flag | Meaning |
|------|---------|
| `-Tuning` | Categories: 1 interesting files, 2 misconfig, 3 info disclosure, 4 injection, 5 RFI inside root, 6 DoS, 7 RFI server-wide, 8 RCE, 9 SQLi, 0 upload, a auth bypass, b software ID, c RSI, d webservice, e admin console, x exclude listed |
| `-Plugins` | Plugin list (default ALL) |
| `-Cgidirs` | CGI dirs to scan (`none`, `all`, or paths) |
| `-mutate` | Guess extra filenames: 1 all roots, 2 password files, 3 Apache user enum, 4 cgiwrap user enum |
| `-mutate-options` | Mutate config |
| `-Platform` | `nix`, `win`, `all` |
| `-404code` | Treat codes as negative (`302,301`) |
| `-404string` | Treat body matching string/regex as negative |
| `-no404` | Disable 404 guessing |
| `-maxtime` | Per-host time cap (`1h`, `60m`, `3600s`) |
| `-Pause` | Seconds between tests |
| `-timeout` | Request timeout (default 10s) |
| `-useproxy` | Proxy URL from config or CLI |

### Auth / TLS
`-id user:pass[:realm]`, `-RSAcert`, `-key`.

### Evasion
| Code | Technique |
|------|-----------|
| 1 | Random URI encoding (non-UTF8) |
| 2 | Directory self-reference (`/./`) |
| 3 | Premature URL ending |
| 4 | Prepend long random string |
| 5 | Fake parameter |
| 6 | TAB as request spacer |
| 7 | Change URL case |
| 8 | Windows `\` separator |
| A | CR as request spacer |
| B | 0x0b as request spacer |
| R | Random User-Agent |

### Output
| Flag | Meaning |
|------|---------|
| `-output` | Output file (`.` = auto-name) |
| `-Format` | `csv`, `json`, `htm`, `sql`, `sqld`, `txt`, `xml` |
| `-Save` | Save positive responses to dir |
| `-Display` | 1 redirects, 2 cookies, 3 all 200s, 4 auth URLs, D debug, E errors, P progress, S scrub, V verbose |
| `-nointeractive` | Disable interactive prompts |
| `-ask` | Update submission: yes/no/auto |

### Other
`-config`, `-dbcheck`, `-list-plugins`, `-nocheck`, `-nointeractive`,
`-nolookup`, `-noslash`, `-nosslkeepalive`, `-Option`, `-Userdbs`, `-useragent`,
`-Version`, `-Add-header`, `-check6`, `-Help`.

## Recipes

```bash
# Basic scan
nikto -host https://example.com

# Non-default port with SSL
nikto -host example.com -port 8443 -ssl

# Fast, less noisy: interesting files + misconfig only
nikto -host https://example.com -Tuning 12

# Exclude DoS tests
nikto -host https://example.com -Tuning x 6

# Through Burp
nikto -host https://example.com -useproxy http://127.0.0.1:8080

# Authenticated
nikto -host https://example.com -id admin:password

# Virtual host
nikto -host https://10.0.0.5 -vhost app.example.com

# JSON output
nikto -host https://example.com -Format json -output nikto.json

# Evasion + proxy + tuned categories
nikto -host https://example.com -Tuning 123b -evasion 1 -useproxy http://127.0.0.1:8080
```

## Output

Default is text to stdout. `-Format json -output file` gives machine-readable
results; `-Format htm` produces a shareable report; `-Save` stores raw positive
responses.

## Gotchas

- Nikto is **loud** and easily blocked; it is not stealthy.
- DoS tests (`-Tuning 6`) can disrupt fragile services — exclude on production.
- False positives are common; verify each finding.
- No JavaScript execution or modern SPA coverage.
- Respect `-maxtime` and `-Pause` on slow targets.
- Some checks are decades old; treat output as leads, not confirmed vulns.

## References

- Wiki: https://github.com/sullo/nikto/wiki
- Source: https://github.com/sullo/nikto
- Plugin docs: https://github.com/sullo/nikto/wiki/Plugins

# Goblin's Grimoire

> A field manual for the pentest and bug-bounty toolbox: every command, every
> flag, and a research workflow for deciding when to use each one.

Twenty-one offensive-security tools, documented the way they are actually used —
grouped by task, with full flag coverage and a `@deep-research` question
attached to every scenario. The point is to never run a command on guesswork:
know what it does, what it costs, and which source settles the question.

## What's inside

| Path | Contents |
|------|----------|
| `<tool>/README.md` | What the tool does, use cases, full command/flag reference, recipes, gotchas, links |
| `<tool>/SCENARIOS.md` | Command-wise scenarios grouped by task, each with the research question that decides when to use it, plus complete flag coverage |
| `<tool>/help*.txt` | Verbatim `--help` output captured from the installed version |
| [`DEEP-RESEARCH.md`](DEEP-RESEARCH.md) | The `@deep-research` workflow, prompt template, evidence rules, research-log format, and scenario index |

Help snapshots are authoritative for the exact installed build. The READMEs add
context, workflows, and copy-paste recipes that `--help` does not explain.

---

## Tool index

### Recon & asset discovery
| Tool | Directory | One-liner |
|------|-----------|-----------|
| subfinder | [`subfinder/`](subfinder/README.md) | Passive subdomain enumeration from dozens of sources |
| httpx | [`httpx/`](httpx/README.md) | Fast HTTP probing, tech detection, screenshots, filtering |
| katana | [`katana/`](katana/README.md) | Web crawler (standard + headless), JS endpoint extraction |
| arjun | [`arjun/`](arjun/README.md) | HTTP parameter discovery |

### Web application testing
| Tool | Directory | One-liner |
|------|-----------|-----------|
| burp | [`burp/`](burp/README.md) | Burp Suite Community — intercepting proxy & manual testing |
| zap | [`zap/`](zap/README.md) | OWASP ZAP — proxying, spidering, active scanning, automation |
| mitmproxy | [`mitmproxy/`](mitmproxy/README.md) | Scriptable TLS-intercepting proxy (CLI/TUI/web) |
| nikto | [`nikto/`](nikto/README.md) | Classic web server misconfiguration/vulnerability scanner |
| nuclei | [`nuclei/`](nuclei/README.md) | Template-driven vulnerability scanner |
| dalfox | [`dalfox/`](dalfox/README.md) | XSS scanner with DOM analysis, mining, WAF bypass |
| arjun | [`arjun/`](arjun/README.md) | Hidden parameter discovery |

### Exploitation
| Tool | Directory | One-liner |
|------|-----------|-----------|
| metasploit | [`metasploit/`](metasploit/README.md) | Metasploit Framework console, modules, sessions |
| msfvenom | [`metasploit-msfvenom/`](metasploit-msfvenom/README.md) | Payload generation and encoding |
| pwntools | [`pwntools/`](pwntools/README.md) | Python exploit-development framework/CLI |
| aflplusplus | [`aflplusplus/`](aflplusplus/README.md) | Coverage-guided fuzzing |

### Network / Active Directory
| Tool | Directory | One-liner |
|------|-----------|-----------|
| netexec | [`netexec/`](netexec/README.md) | Multi-protocol network exploitation (SMB/LDAP/WinRM/...) |
| enum4linux-ng | [`enum4linux-ng/`](enum4linux-ng/README.md) | SMB/NetBIOS/AD enumeration |
| wireshark | [`wireshark/`](wireshark/README.md) | Packet capture & analysis (`tshark`, `dumpcap`, GUI) |

### Code / container / supply chain
| Tool | Directory | One-liner |
|------|-----------|-----------|
| semgrep | [`semgrep/`](semgrep/README.md) | Static analysis (SAST) with community & custom rules |
| trivy | [`trivy/`](trivy/README.md) | Vulnerability/misconfig/secret scanning for images, FS, IaC |

### Privacy / anonymization
| Tool | Directory | One-liner |
|------|-----------|-----------|
| torsocks | [`torsocks/`](torsocks/README.md) | Route a single app's TCP through Tor (SOCKS5) |
| tor-browser | [`tor-browser/`](tor-browser/README.md) | Hardened Firefox preconfigured for Tor |

---

## Recommended end-to-end bug-bounty pipeline

```bash
# 1. Passive subdomain enumeration
subfinder -d example.com -all -silent -o subs.txt

# 2. Probe live HTTP services, capture status/title/tech
httpx -l subs.txt -sc -title -td -tech-detect -follow-redirects -silent -o live.txt

# 3. Crawl live hosts for URLs/endpoints (JS-aware)
katana -list live.txt -jc -jsl -d 3 -silent -o urls.txt

# 4. Discover hidden parameters
arjun -i live.txt -oT params.txt

# 5. Scan for known CVEs/misconfigs with templates
nuclei -l live.txt -severity low,medium,high,critical -o nuclei.txt

# 6. XSS-focused pass on crawled URLs
dalfox file urls.txt --silence --format jsonl -o xss.jsonl

# 7. Manual deep-dive with Burp/ZAP/mitmproxy on anything interesting
```

## Environment notes

- Paths in examples use `/home/user` and `~` as placeholders; substitute your own.
- Some binaries may be group-restricted with `dpkg-statoverride` (for example
  `afl-fuzz` and `arjun`). Run them as a member of the permitted group, or as
  root in an isolated lab.
- Live packet capture needs membership in the `wireshark` group.
- `torsocks` torifies a single process. Transparent, system-wide Tor routing is
  a separate root-level setup and is not covered here.
- Tools write config/state under `$HOME`, never to `/opt`.

## Safety & legality

Use only against systems you own or have **written authorization** to test.
Passive recon (subfinder, public data) is generally safe; active scanning
(nuclei, nikto, dalfox, ZAP active scan, netexec, Metasploit) is not.
Respect scope, rate limits, and program rules.

---

*Help snapshots captured 2026-11-09. Versions are listed in each tool's README.*

# Deep research on tool tasks

How to use the `deep-research` workflow against the tools in this knowledge
base. Every tool directory has a `SCENARIOS.md` that maps its **complete
command surface** to realistic scenarios, and each scenario carries the
research question that decides whether to trust or tune the command.

## When to use this

Reach for `@deep-research` instead of guessing when:

- a flag's behavior changed between versions,
- two tools overlap and you must pick one,
- a program's rules constrain rate, payloads, or scope,
- a benchmark or vendor claim needs independent verification,
- a failure mode (WAF block, lockout, false positive) has no obvious cause.

Do **not** use it for questions the local `help.txt` already answers. Those
snapshots are authoritative for the installed build.

## The loop

1. **Define the question** — what decision depends on the answer.
2. **Identify sources** — official docs, release notes, GitHub issues,
   independent writeups, benchmarks, program rules.
3. **Gather evidence** — read and note primary sources first.
4. **Cross-reference** — compare claims across independent sources.
5. **Identify gaps** — what the evidence does not cover.
6. **Weigh tradeoffs** — pros/cons per option for *your* constraints.
7. **Structure the output** — question, evidence, analysis, recommendation.
8. **Cite sources** — so the reasoning can be checked.

## Prompt template

```text
Use @deep-research.

Question: <one decision-shaped question>
Decision it informs: <what you will do differently>
Constraints: <version, OS, program rules, rate limits, scope>
Known local facts: <what help.txt / README already establishes>
Sources to prioritize: official docs > release notes > GitHub issues >
  independent benchmarks/writeups
Output: cited recommendation + exact command/flags + confidence + gaps
```

## Evidence rules

- Primary source or it did not happen: vendor docs, man pages, release notes,
  the project's own issue tracker.
- A blog post is a lead, not proof. Cross-check it against docs or source.
- Separate **fact** (reproduced), **opinion** (practitioner preference), and
  **speculation** (no evidence).
- Record the tool version the evidence applies to — flags drift.
- State what you could not verify. A named gap is better than false confidence.

## Recording results

Append findings to `~/tools/<tool>/RESEARCH-LOG.md`:

```markdown
## 2026-11-09 — <question>
- Sources: <urls, versions, dates>
- Finding: <one paragraph>
- Confidence: high/medium/low
- Gap: <what remains unknown>
- Applied: <command or config change>
```

## Scenario index

Every directory below has a `SCENARIOS.md`. There are 89 directories covering
88 tools: `metasploit-msfvenom` is documented alongside Metasploit rather than
counted separately.

### Recon & asset discovery

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| nmap | [`nmap/SCENARIOS.md`](nmap/SCENARIOS.md) | discovery/port/service/NSE/output |
| masscan | [`masscan/SCENARIOS.md`](masscan/SCENARIOS.md) | rate/ranges/banners/output |
| tcpdump | [`tcpdump/SCENARIOS.md`](tcpdump/SCENARIOS.md) | BPF filters/interfaces/rotation |
| whois | [`whois/SCENARIOS.md`](whois/SCENARIOS.md) | client/referral/RIPE/redaction |
| traceroute | [`traceroute/SCENARIOS.md`](traceroute/SCENARIOS.md) | UDP/ICMP/TCP methods/timing/ASN/MTU |
| dnsrecon | [`dnsrecon/SCENARIOS.md`](dnsrecon/SCENARIOS.md) | std/brt/axfr/snoop/zonewalk |
| subfinder | [`subfinder/SCENARIOS.md`](subfinder/SCENARIOS.md) | single command, all flag groups |
| httpx | [`httpx/SCENARIOS.md`](httpx/SCENARIOS.md) | probe/filter/headless/output matrix |
| katana | [`katana/SCENARIOS.md`](katana/SCENARIOS.md) | standard/JS/headless crawl matrix |
| arjun | [`arjun/SCENARIOS.md`](arjun/SCENARIOS.md) | GET/POST/JSON/XML + passive |

### OSINT & people

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| theHarvester | [`theharvester/SCENARIOS.md`](theharvester/SCENARIOS.md) | sources/limits/output |
| recon-ng | [`recon-ng/SCENARIOS.md`](recon-ng/SCENARIOS.md) | workspace/module/marketplace |
| spiderfoot | [`spiderfoot/SCENARIOS.md`](spiderfoot/SCENARIOS.md) | scan/config/module/CLI |
| ghunt | [`ghunt/SCENARIOS.md`](ghunt/SCENARIOS.md) | login/email/drive/gaia |
| holehe | [`holehe/SCENARIOS.md`](holehe/SCENARIOS.md) | email checks + only-used/csv |
| sherlock | [`sherlock/SCENARIOS.md`](sherlock/SCENARIOS.md) | username/site-list/output |
| metagoofil | [`metagoofil/SCENARIOS.md`](metagoofil/SCENARIOS.md) | search/download/extract |

### Web application testing

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| burp | [`burp/SCENARIOS.md`](burp/SCENARIOS.md) | launcher flags + tool workflows |
| zap | [`zap/SCENARIOS.md`](zap/SCENARIOS.md) | GUI/daemon/CLI/automation/api |
| mitmproxy | [`mitmproxy/SCENARIOS.md`](mitmproxy/SCENARIOS.md) | 3 frontends x 6 modes |
| nikto | [`nikto/SCENARIOS.md`](nikto/SCENARIOS.md) | tuning/evasion/output matrix |
| nuclei | [`nuclei/SCENARIOS.md`](nuclei/SCENARIOS.md) | target/template/output/DAST |
| dalfox | [`dalfox/SCENARIOS.md`](dalfox/SCENARIOS.md) | scan/server/payload/mcp |
| sqlmap | [`sqlmap/SCENARIOS.md`](sqlmap/SCENARIOS.md) | detection/enumeration/tamper/os-shell |
| ffuf | [`ffuf/SCENARIOS.md`](ffuf/SCENARIOS.md) | content/vhost/parameter fuzzing |
| gobuster | [`gobuster/SCENARIOS.md`](gobuster/SCENARIOS.md) | dir/dns/vhost/fuzz modes |
| wfuzz | [`wfuzz/SCENARIOS.md`](wfuzz/SCENARIOS.md) | payload/iterator/filter |
| dirsearch | [`dirsearch/SCENARIOS.md`](dirsearch/SCENARIOS.md) | wordlist/extensions/recursion/format |
| dirb | [`dirb/SCENARIOS.md`](dirb/SCENARIOS.md) | wordlists/extensions/recursion/auth |
| whatweb | [`whatweb/SCENARIOS.md`](whatweb/SCENARIOS.md) | aggression/plugins/output/performance |
| cewl | [`cewl/SCENARIOS.md`](cewl/SCENARIOS.md) | crawl/depth/wordlist options |

### JavaScript, CMS & secrets

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| linkfinder | [`linkfinder/SCENARIOS.md`](linkfinder/SCENARIOS.md) | single URL/file/dir modes |
| secretfinder | [`secretfinder/SCENARIOS.md`](secretfinder/SCENARIOS.md) | regex/output modes |
| cmseek | [`cmseek/SCENARIOS.md`](cmseek/SCENARIOS.md) | CMS detect + scan modes |
| joomscan | [`joomscan/SCENARIOS.md`](joomscan/SCENARIOS.md) | enumeration + vulnerability checks |
| paramspider | [`paramspider/SCENARIOS.md`](paramspider/SCENARIOS.md) | archive URL harvesting |
| x8 | [`x8/SCENARIOS.md`](x8/SCENARIOS.md) | parameter discovery modes |

### API & protocol testing

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| corsy | [`corsy/SCENARIOS.md`](corsy/SCENARIOS.md) | CORS scan options |
| smuggler | [`smuggler/SCENARIOS.md`](smuggler/SCENARIOS.md) | CL/TE detection modes |
| websocat | [`websocat/SCENARIOS.md`](websocat/SCENARIOS.md) | client/server/bridge |
| grpcurl | [`grpcurl/SCENARIOS.md`](grpcurl/SCENARIOS.md) | list/describe/invoke + reflection |
| subjs | [`subjs/SCENARIOS.md`](subjs/SCENARIOS.md) | URL/JS inventory flags |
| sourcemapper | [`sourcemapper/SCENARIOS.md`](sourcemapper/SCENARIOS.md) | url/jsurl/dir extraction |

### Network / Active Directory

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| netexec | [`netexec/SCENARIOS.md`](netexec/SCENARIOS.md) | 10 protocols x actions |
| impacket | [`impacket/SCENARIOS.md`](impacket/SCENARIOS.md) | 62 scripts: enum/kerberos/dump/exec/relay |
| smbclient | [`smbclient/SCENARIOS.md`](smbclient/SCENARIOS.md) | share access/upload/download/scripting |
| rpcclient | [`rpcclient/SCENARIOS.md`](rpcclient/SCENARIOS.md) | SAMR/LSA/SPOOLSS command groups |
| ldapsearch | [`ldapsearch/SCENARIOS.md`](ldapsearch/SCENARIOS.md) | bind/base/filter/attribute/output |
| enum4linux-ng | [`enum4linux-ng/SCENARIOS.md`](enum4linux-ng/SCENARIOS.md) | all enumeration switches |
| wireshark | [`wireshark/SCENARIOS.md`](wireshark/SCENARIOS.md) | dumpcap/tshark/GUI |
| snmp | [`snmp/SCENARIOS.md`](snmp/SCENARIOS.md) | snmpwalk/get/set/bulk/translate |
| onesixtyone | [`onesixtyone/SCENARIOS.md`](onesixtyone/SCENARIOS.md) | community-string scanning |
| nbtscan | [`nbtscan/SCENARIOS.md`](nbtscan/SCENARIOS.md) | range/verbose/script output |
| sshuttle | [`sshuttle/SCENARIOS.md`](sshuttle/SCENARIOS.md) | subnet/DNS/ssh options |
| socat | [`socat/SCENARIOS.md`](socat/SCENARIOS.md) | TCP/UDP/TLS/UNIX/PTY relays + options |
| nc | [`nc/SCENARIOS.md`](nc/SCENARIOS.md) | traditional connect/listen/scan/UDP |
| proxychains4 | [`proxychains4/SCENARIOS.md`](proxychains4/SCENARIOS.md) | chain modes/DNS/proxy auth/leak checks |
| kerbrute | [`kerbrute/SCENARIOS.md`](kerbrute/SCENARIOS.md) | userenum/passwordspray/bruteuser |

### Credentials & password attacks

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| hashcat | [`hashcat/SCENARIOS.md`](hashcat/SCENARIOS.md) | hash modes/attack modes/rules/sessions |
| john | [`john/SCENARIOS.md`](john/SCENARIOS.md) | core 1.9.0 formats/wordlists/rules/incremental |
| hydra | [`hydra/SCENARIOS.md`](hydra/SCENARIOS.md) | protocol modules/user-pass matrices/output |
| medusa | [`medusa/SCENARIOS.md`](medusa/SCENARIOS.md) | 25 modules/tasks/hosts/resume/output |

### Wireless & RF

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| hcxdumptool | [`hcxdumptool/SCENARIOS.md`](hcxdumptool/SCENARIOS.md) | capture/filter/attack modes |
| hcxtools | [`hcxtools/SCENARIOS.md`](hcxtools/SCENARIOS.md) | hcxpcapngtool/psktool/hash2cap |
| kismet | [`kismet/SCENARIOS.md`](kismet/SCENARIOS.md) | capture server + kismetdb tools |
| bully | [`bully/SCENARIOS.md`](bully/SCENARIOS.md) | WPS PIN attack options |
| eaphammer | [`eaphammer/SCENARIOS.md`](eaphammer/SCENARIOS.md) | evil-twin/EAP capture modes |
| wifiphisher | [`wifiphisher/SCENARIOS.md`](wifiphisher/SCENARIOS.md) | AP/phishing scenario modes |

### Phishing & social engineering

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| gophish | [`gophish/SCENARIOS.md`](gophish/SCENARIOS.md) | server flags + admin API |
| evilginx | [`evilginx/SCENARIOS.md`](evilginx/SCENARIOS.md) | CLI flags + console |
| set | [`set/SCENARIOS.md`](set/SCENARIOS.md) | menu paths |
| crunch | [`crunch/SCENARIOS.md`](crunch/SCENARIOS.md) | keyspace/mask/chunk/resume |

### Exploitation & fuzzing

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| metasploit | [`metasploit/SCENARIOS.md`](metasploit/SCENARIOS.md) | console flags + commands |
| msfvenom | [`metasploit-msfvenom/SCENARIOS.md`](metasploit-msfvenom/SCENARIOS.md) | list/payload/encode/format |
| pwntools | [`pwntools/SCENARIOS.md`](pwntools/SCENARIOS.md) | all 20 `pwn` subcommands |
| aflplusplus | [`aflplusplus/SCENARIOS.md`](aflplusplus/SCENARIOS.md) | fuzz/build/minimize/monitor |

### Reverse engineering, pwn & forensics

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| apktool | [`apktool/SCENARIOS.md`](apktool/SCENARIOS.md) | decode/build/install-framework |
| ropper | [`ropper/SCENARIOS.md`](ropper/SCENARIOS.md) | search/info/chain |
| ropgadget | [`ropgadget/SCENARIOS.md`](ropgadget/SCENARIOS.md) | gadget search/filter/chain |
| one-gadget | [`one-gadget/SCENARIOS.md`](one-gadget/SCENARIOS.md) | glibc one-shot gadget search |
| pwndbg | [`pwndbg/SCENARIOS.md`](pwndbg/SCENARIOS.md) | GDB command groups |
| gef | [`gef/SCENARIOS.md`](gef/SCENARIOS.md) | GDB command groups |
| volatility3 | [`volatility3/SCENARIOS.md`](volatility3/SCENARIOS.md) | plugin/history/render |

### TLS & service scanning

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| testssl | [`testssl/SCENARIOS.md`](testssl/SCENARIOS.md) | protocol/cipher/vuln/output matrix |
| sslyze | [`sslyze/SCENARIOS.md`](sslyze/SCENARIOS.md) | scan targets/json/output |

### Code / container / supply chain

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| semgrep | [`semgrep/SCENARIOS.md`](semgrep/SCENARIOS.md) | all subcommands + scan modes |
| trivy | [`trivy/SCENARIOS.md`](trivy/SCENARIOS.md) | all targets + scanners |

### Privacy / anonymization

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| torsocks | [`torsocks/SCENARIOS.md`](torsocks/SCENARIOS.md) | wrapper/shell/isolation |
| tor-browser | [`tor-browser/SCENARIOS.md`](tor-browser/SCENARIOS.md) | launcher/profile/bridges |

## Anti-patterns

- **Researching instead of reading `help.txt`.** Local truth first.
- **One blog post as the answer.** No cross-reference, no confidence.
- **Ignoring version.** A 2023 flag list may be wrong for the installed build.
- **Researching forever.** Timebox; state the gap and proceed with the safest
  reversible option.
- **Letting research widen scope.** A finding is not permission to test it.

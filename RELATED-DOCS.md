# Related documentation

Which public projects document **a tool's commands mapped to realistic use
cases**, and how does the Grimoire's format compare?

This is a `@deep-research` output, not a link dump. Every format claim below
was checked against the project's own README, site, or repository tree on
**2026-11-10**. No tools were installed to produce it.

## Research question

> Is there a public project that covers, for each tool, every command with an
> example scenario — and what should this repository borrow from it?

The decision this informs: whether the three-file Grimoire format is worth
keeping, and which external sources to use when writing the ~30 missing
`SCENARIOS.md` files.

## Short answer

No project found combines all three properties this repository has:

1. **Per-tool completeness** — the full command surface, not a highlight reel.
2. **Scenario mapping** — each command tied to a task and a decision question.
3. **Installed-build anchor** — verbatim `help.txt` from the exact version.

The closest structural match is `yaklang/hack-skills` (`SKILL.md` +
`SCENARIOS.md`), but it is topic-level and only 8 topics have a scenario file.
The closest command-level matches are WADComs, Arsenal, and the HighOn.Coffee
per-tool cheat sheets. The closest **verification** model is The Hacker Recipes,
which states its content is "verified and tested whenever possible".

**Confidence: high** for the format descriptions (primary sources read
directly). **Medium** for the negative claim that nothing else combines all
three — 14 projects were checked, not the entire internet.

## Comparison

| Project | Scope | Unit of organization | Command → scenario | Version anchor | Best use here |
|---|---|---|---|---|---|
| **Goblin's Grimoire** (this repo) | 66 installed tools | Per tool | Yes — every command, grouped by task, with a research question | Yes — `help.txt` per installed build | Baseline |
| [WADComs](https://wadcoms.github.io/) | Windows / AD offensive tools | Per tool, filtered by "what you have" | Yes — template command per tool in context | No | AD command templates; the filter UX |
| [GTFOBins](https://gtfobins.org/) | Unix-like binaries | Per binary | Yes — command per function (shell, read, write, sudo, SUID, capabilities, inherit) | Partial — version ranges noted | Privilege-escalation taxonomy |
| [LOLBAS](https://lolbas-project.github.io/) | Windows binaries | Per binary | Yes — use case + command + privilege + OS + ATT&CK + detections | No | Scenario metadata and detection notes |
| [Arsenal](https://github.com/Orange-Cyberdefense/arsenal) | Pentest commands | Per-tool cheatsheet, searchable and launchable | Yes — command + description + variables | No | Machine-usable command inventory |
| [Kali Tools](https://www.kali.org/tools/) | Kali packages | Per tool | Usage examples, not scenario-mapped | Package version | Canonical per-tool reference |
| [HighOn.Coffee](https://highon.coffee/blog/penetration-testing-tools-cheat-sheet/) | Infrastructure pentest | Mixed: one combined sheet + per-tool sheets (Nmap, SQLMap, Nikto, Subfinder, Naabu, httpx, Katana) | Yes — command + description, ordered by engagement phase | No | Writing concise per-tool sheets |
| [HackTricks](https://github.com/HackTricks-wiki/hacktricks) | Techniques and domains | Topic-first; tools appear inline | Yes — commands inside technique context | No | Technique depth and cross-links |
| [The Hacker Recipes](https://www.thehacker.recipes/) | AD, web, infra | Technique-first | Yes — commands inline, with context | No | Verification standard |
| [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) | Web vulnerabilities | Per vulnerability | Payloads, not per-tool commands | No | Payload companion |

### Also considered

- [InternalAllTheThings](https://github.com/swisskyrepo/InternalAllTheThings) — AD/internal cheatsheets, topic-first.
- [ired.team](https://www.ired.team/) — lab notes per technique, strong on artifacts and telemetry.
- [yaklang/hack-skills](https://github.com/yaklang/hack-skills) — 102 topic skills across 14 domains; 8 have a `SCENARIOS.md` companion (verified via the GitHub tree API).
- [Pentest Book](https://github.com/six2dez/pentest-book) — searchable wiki organized by phase, port, and technology.
- [Awesome Pentest](https://github.com/enaqx/awesome-pentest) — discovery index, no command detail.

## What each one actually does

### WADComs — scenario filter, not command completeness

The site describes itself as "an interactive cheat sheet, containing a curated
list of offensive security tools and their respective commands, to be used
against Windows/AD environments." Its differentiator is the entry point: you
select what you already have (password, username, service, etc.) and it
returns tools plus a template command.

**Borrow:** a short "what you have" table at the top of each `SCENARIOS.md`
(no creds / valid creds / hash / domain join / network position). It turns a
command list into a decision aid.

**Do not copy:** WADComs is deliberately a highlight reel. It does not try to
cover every flag, and it does not anchor to an installed build.

### GTFOBins — function taxonomy for privilege escalation

GTFOBins is "a curated list of Unix-like executables that can be used to
bypass local security restrictions in misconfigured systems." Each binary page
is split by **function** — Shell, File write, File read, Inherit — and each
function lists unprivileged, sudo, and SUID variants. Version constraints are
recorded where they matter (the `nmap --interactive` shell is marked 2.02 to
5.21).

**Borrow:** for privilege-escalation tools, organize scenarios by *function*
first, command second. It matches how an operator thinks under pressure.

**Do not copy:** the scope is binaries that are already on the target, not
tools an operator brings.

### LOLBAS — the metadata model

Each LOLBAS binary page carries paths, resources, detections (Sigma, Elastic,
Splunk, IOC), and then commands annotated with **use case**, **privileges
required**, **operating systems**, and **ATT&CK technique**. The Certutil page,
for example, separates Download, ADS, Encode, and Decode, each with its own
technique ID.

**Borrow:** add `Privilege`, `ATT&CK`, and `Detection` lines to scenario
entries where they apply. It makes the same command useful to both sides of an
engagement and gives readers a reason to pick one variant over another.

**Do not copy:** Windows-only, and the command set is intentionally narrow.

### Arsenal — commands as data

Arsenal calls itself "a quick inventory, reminder and launcher for pentest
commands." Cheatsheets are Markdown or reStructuredText, commands carry
descriptions and `<argument|default>` placeholders, global variables such as
`ip` prefill every command, and selecting an entry types it into the terminal.

**Borrow:** the data model. A future `SCENARIOS.md` front matter block (tool,
command, tags, variables, privilege) would make the Grimoire searchable and
launchable without changing the prose.

**Do not copy:** Arsenal has no scenario narrative and no version anchor; it is
a launcher, not a field manual.

### Kali Tools — the canonical per-tool page

Kali's tool documentation is organized by phase (Reconnaissance, Resource
Development, Initial Access, Execution, Persistence, Privilege Escalation,
Defense Evasion, Credential Access, Discovery, and so on), with one page per
package and usage examples. It is the best source for "what does the packaged
version of this tool ship with?"

**Borrow:** link the Kali Tools page from each tool `README.md` as the upstream
reference. It is stable, package-aware, and complements `help.txt`.

### HighOn.Coffee — the per-tool sheet done well

The combined Pen Testing Tools Cheat Sheet is ordered by engagement phase, and
the site also maintains dedicated sheets for Nmap, SQLMap, Nikto, Subfinder,
Naabu, httpx, and Katana. Commands are paired with a one-line description; the
combined page was last updated 2025-05-20.

**Borrow:** the command-plus-description table density. It is close to the
Grimoire's `SCENARIOS.md` tables, but without the per-command research
question.

### HackTricks — depth, not tool coverage

HackTricks is a book, not an index. Tools appear inside technique pages, so it
answers "how do I abuse this service?" better than "what are all the flags for
this tool?" It is actively maintained and available in many languages.

**Borrow:** cross-link technique pages from scenarios. Do not try to mirror it.

### The Hacker Recipes — the verification bar

The project explicitly rejects the cheat-sheet model:

> "Cheatsheets are valuable resources, but they often either overwhelm with
> complexity or fall short by not providing enough context and explanation.
> This is not a cheatsheet project. Most of the content published here is
> original content, that's verified and tested whenever possible."

**Borrow:** a `Tested:` line per scenario (date, OS, tool version, target
type). The Grimoire already anchors the *build* with `help.txt`; this would
anchor the *run*.

### PayloadsAllTheThings and InternalAllTheThings — companions, not competitors

PayloadsAllTheThings is a payload and bypass list per vulnerability, with
Burp Intruder files alongside each section. InternalAllTheThings is the AD and
internal-pentest counterpart. Neither tries to enumerate a tool's command
surface.

**Borrow:** link them from the relevant tool or scenario as payload sources.

### yaklang/hack-skills — the closest file naming

The repository has 102 deep topic skills across 14 domains, each as
`skills/{id}/SKILL.md`. Only 8 topics also have `SCENARIOS.md` (SQLi, XSS,
SSRF, SSTI, XXE, CORS, file upload, business logic), and those files are
real-world case studies and CVE notes — not per-tool command coverage.

**Borrow:** the `SKILL.md` + `SCENARIOS.md` pairing is the same instinct as
this repo. The execution differs: hack-skills is topic-first and
agent-oriented; the Grimoire is tool-first and operator-oriented.

## Gaps in the ecosystem

1. **Topic-first navigation dominates.** Most projects answer "how do I attack
   X?" — few answer "what is every documented command for tool Y?"
2. **Flags drift, anchors do not.** Almost none record the installed version
   behind a command. This is the Grimoire's strongest differentiator.
3. **Commands lack decision context.** WADComs and LOLBAS come closest, but
   most sheets do not say *why* to choose one variant over another.
4. **Coverage is uneven.** Windows/AD and Unix privesc are well served; wireless,
   OSINT, binary exploitation, cloud, and protocol-specific tooling are thin.
5. **Maintenance varies.** Community projects go stale; a per-tool page can
   look authoritative while describing an old release.

## Recommendations for this repo

1. **Keep the three-file format.** No surveyed project does per-tool command
   completeness + scenario mapping + installed-build help. It is the niche.
2. **Add a "See also" line to each tool `README.md`** pointing at the best
   external source for that tool (Kali Tools page; GTFOBins for privesc
   binaries; WADComs for AD tools; HighOn.Coffee for the tools it covers).
3. **Use the external sources as evidence, not as copy.** When writing the
   missing `SCENARIOS.md` files, verify each command against `help.txt` and
   cite the external page only for scenario framing.
4. **Prioritize the installed-but-undocumented tools** (appendix below),
   starting with `nmap`, `sqlmap`, `hashcat`, `hydra`, `impacket-*`, and the
   `aircrack-ng` suite — highest operational value and the largest scenario
   surface.
5. **Add metadata, not a new format.** `Privilege`, `ATT&CK`, and `Tested`
   lines per scenario capture the best ideas from LOLBAS and The Hacker
   Recipes without restructuring the files.

## Appendix — installed but undocumented (verified 2026-11-10)

`command -v` was used; no packages were installed. `ghidra` is present at
`/opt/ghidra/ghidra_12.1.4_PUBLIC/ghidraRun` but not on `PATH`. Impacket
ships as `impacket-*` binaries.

| Category | Tools |
|---|---|
| Recon / network | `nmap`, `masscan`, `tcpdump`, `whatweb` |
| Web | `sqlmap`, `ffuf`, `gobuster`, `dirb` |
| Credentials | `hashcat`, `john`, `hydra`, `medusa` |
| Windows / AD | `impacket-netview`, `impacket-rpcdump`, `impacket-samrdump`, `impacket-secretsdump`, `impacket-wmiexec` |
| MITM / sniffing | `bettercap`, `ettercap`, `dsniff` (+ `arpspoof`, `dnsspoof`, `macof`, `tcpkill`, `urlsnarf`, `webspy`, `mailsnarf`, `msgsnarf`, `filesnarf`) |
| Wireless | `aircrack-ng` suite, `reaver`, `pixiewps`, `wash` |
| Reverse engineering | `ghidra`, `binwalk`, `checksec` |
| Pivoting / transport | `nc`, `socat`, `proxychains4`, `openvpn` |

## Sources

Accessed 2026-11-10. Primary sources first.

- WADComs — <https://wadcoms.github.io/> and <https://github.com/WADComs/WADComs.github.io>
- GTFOBins — <https://gtfobins.org/> and <https://github.com/GTFOBins/GTFOBins.github.io> (nmap page read directly)
- LOLBAS — <https://lolbas-project.github.io/> (Certutil page read directly)
- Arsenal — <https://github.com/Orange-Cyberdefense/arsenal>
- Kali Tools — <https://www.kali.org/tools/>
- HighOn.Coffee — <https://highon.coffee/blog/penetration-testing-tools-cheat-sheet/>
- HackTricks — <https://github.com/HackTricks-wiki/hacktricks>
- The Hacker Recipes — <https://www.thehacker.recipes/> and <https://github.com/The-Hacker-Recipes/The-Hacker-Recipes>
- PayloadsAllTheThings — <https://github.com/swisskyrepo/PayloadsAllTheThings>
- InternalAllTheThings — <https://github.com/swisskyrepo/InternalAllTheThings>
- ired.team — <https://www.ired.team/>
- yaklang/hack-skills — <https://github.com/yaklang/hack-skills> (tree checked with the GitHub API)
- Pentest Book — <https://github.com/six2dez/pentest-book>
- Awesome Pentest — <https://github.com/enaqx/awesome-pentest>

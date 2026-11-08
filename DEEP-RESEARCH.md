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

| Tool | Scenarios file | Command surface |
|------|----------------|-----------------|
| subfinder | [`subfinder/SCENARIOS.md`](subfinder/SCENARIOS.md) | single command, all flag groups |
| httpx | [`httpx/SCENARIOS.md`](httpx/SCENARIOS.md) | probe/filter/headless/output matrix |
| katana | [`katana/SCENARIOS.md`](katana/SCENARIOS.md) | standard/JS/headless crawl matrix |
| arjun | [`arjun/SCENARIOS.md`](arjun/SCENARIOS.md) | GET/POST/JSON/XML + passive |
| burp | [`burp/SCENARIOS.md`](burp/SCENARIOS.md) | launcher flags + tool workflows |
| zap | [`zap/SCENARIOS.md`](zap/SCENARIOS.md) | GUI/daemon/CLI/automation/api |
| mitmproxy | [`mitmproxy/SCENARIOS.md`](mitmproxy/SCENARIOS.md) | 3 frontends x 6 modes |
| nikto | [`nikto/SCENARIOS.md`](nikto/SCENARIOS.md) | tuning/evasion/output matrix |
| nuclei | [`nuclei/SCENARIOS.md`](nuclei/SCENARIOS.md) | target/template/output/DAST |
| dalfox | [`dalfox/SCENARIOS.md`](dalfox/SCENARIOS.md) | scan/server/payload/mcp |
| netexec | [`netexec/SCENARIOS.md`](netexec/SCENARIOS.md) | 10 protocols x actions |
| enum4linux-ng | [`enum4linux-ng/SCENARIOS.md`](enum4linux-ng/SCENARIOS.md) | all enumeration switches |
| metasploit | [`metasploit/SCENARIOS.md`](metasploit/SCENARIOS.md) | console flags + commands |
| msfvenom | [`msfvenom/SCENARIOS.md`](metasploit-msfvenom/SCENARIOS.md) | list/payload/encode/format |
| wireshark | [`wireshark/SCENARIOS.md`](wireshark/SCENARIOS.md) | dumpcap/tshark/GUI |
| semgrep | [`semgrep/SCENARIOS.md`](semgrep/SCENARIOS.md) | all subcommands + scan modes |
| trivy | [`trivy/SCENARIOS.md`](trivy/SCENARIOS.md) | all targets + scanners |
| pwntools | [`pwntools/SCENARIOS.md`](pwntools/SCENARIOS.md) | all 20 `pwn` subcommands |
| aflplusplus | [`aflplusplus/SCENARIOS.md`](aflplusplus/SCENARIOS.md) | fuzz/build/minimize/monitor |
| torsocks | [`torsocks/SCENARIOS.md`](torsocks/SCENARIOS.md) | wrapper/shell/isolation |
| tor-browser | [`tor-browser/SCENARIOS.md`](tor-browser/SCENARIOS.md) | launcher/profile/bridges |

## Anti-patterns

- **Researching instead of reading `help.txt`.** Local truth first.
- **One blog post as the answer.** No cross-reference, no confidence.
- **Ignoring version.** A 2023 flag list may be wrong for the installed build.
- **Researching forever.** Timebox; state the gap and proceed with the safest
  reversible option.
- **Letting research widen scope.** A finding is not permission to test it.

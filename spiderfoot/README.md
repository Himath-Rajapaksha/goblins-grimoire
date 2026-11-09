# SpiderFoot — automated OSINT and attack-surface correlation

- **Version:** SpiderFoot 4.0.0
- **Binary:** `/usr/local/bin/spiderfoot` (`sf.py` in the install tree)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/smicallef/spiderfoot
- **Docs:** https://www.spiderfoot.net/documentation/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

SpiderFoot automates OSINT collection and correlation. A scan starts from a
target (domain, IP, email, phone, name, BSSID, etc.) and runs the modules that
can consume that target type. Each module emits typed events; other modules
consume those events, so the scan grows outward automatically. Results are
stored per scan and exposed through a web UI, CLI, and API.

## Use cases

| Situation | Why SpiderFoot |
|-----------|----------------|
| Attack-surface mapping | Automatic module chaining from one seed |
| Passive recon | 200+ modules across public sources |
| Threat intel enrichment | Correlate CVEs, leaks, and mentions |
| Repeatable scans | Scan profiles and scheduled runs |
| API integration | Feed findings into other tools |
| Black-box footprint | No credentials needed for passive scans |

## Requirements & install layout

- Installed at `/opt/spiderfoot`; wrapper `/usr/local/bin/spiderfoot`.
- Web UI default: `http://127.0.0.1:5001`.
- SQLite database `spiderfoot.db` in the install/data directory.
- API keys are configured per module in the web UI or `sf.cfg`; many modules
  work without keys.
- Python 3 virtual environment bundled in the install tree.

## Basic syntax

```bash
# launch the web UI
spiderfoot -l 127.0.0.1:5001

# list modules and event types
spiderfoot -M
spiderfoot -T

# run a scan from the CLI and print results
spiderfoot -s example.com -m sfp_dnsresolve,sfp_crt,sfp_hackertarget -o csv

# use a use-case profile
spiderfoot -s example.com -u footprint -o json

# correlate an existing scan
spiderfoot -C <scanID>

# strict mode: only modules that can consume the target directly
spiderfoot -s example.com -x
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-l <ip:port>` | Web UI listen address |
| `-s <target>` | Target for a CLI scan |
| `-m <mod1,mod2>` | Modules to enable |
| `-M, --modules` | List available modules |
| `-t <types>` | Event types to collect (auto-selects modules) |
| `-T, --types` | List available event types |
| `-u <profile>` | Module profile: `all`, `footprint`, `investigate`, `passive` |
| `-C <scanID>` | Run correlation rules against a scan |
| `-o <tab\|csv\|json>` | Output format (tab default) |
| `-H` | Omit field headers |
| `-n` | Strip newlines |
| `-r` | Include source data in tab/csv |
| `-S <len>` | Max data length to display |
| `-D <delim>` | CSV delimiter |
| `-f` | Filter to requested event types only |
| `-F <types>` | Show only these event types |
| `-x` | Strict mode (target-consuming modules only) |
| `-q` | Disable logging (hides errors) |
| `-d, --debug` | Debug output |
| `-max-threads <n>` | Concurrent modules |
| `-V, --version` | Version |

## Recipes

```bash
# 1. Start the UI and leave it running
spiderfoot -l 127.0.0.1:5001

# 2. List modules, then choose a focused set
spiderfoot -M | head -50

# 3. Passive domain scan with JSON output
spiderfoot -s example.com -u passive -o json > passive.json

# 4. Footprint scan for an external engagement
spiderfoot -s example.com -u footprint -o csv -D ';' > footprint.csv

# 5. Investigate a specific event type
spiderfoot -s example.com -t IP_ADDRESS,EMAILADDR -o tab

# 6. Strict, target-only scan
spiderfoot -s example.com -x -o json > strict.json

# 7. Re-run correlation on a stored scan
spiderfoot -C 1A2B3C4D5E6F

# 8. Limit concurrency on a small host
spiderfoot -s example.com -u footprint -max-threads 4 -o csv > out.csv
```

## Output & pipelines

- Web UI: interactive graph, scan history, per-event drill-down.
- CLI: tab/csv/json to stdout; redirect into files.
- SQLite `spiderfoot.db` for offline queries.
- REST API on the UI port (`/api/v1/...`) for automation.
- Findings feed `nuclei`, `httpx`, ticketing, and reporting tools.

## Gotchas & operational security

- `-u all` is extremely broad and slow; start with `passive` or `footprint`.
- Strict mode (`-x`) can miss useful pivots that need a second-hop target.
- Many modules require API keys; without them coverage drops sharply.
- Results contain personal data and breach records — scope and retention
  matter.
- The UI has no authentication by default; bind to localhost or put it behind
  a reverse proxy with auth.
- CLI scans write to the same database as the UI; avoid concurrent scans on a
  small host.
- Some modules are noisy outward (DNS brute force, port scans); check the
  profile before running.

## Legal & safety

Use only for authorized reconnaissance. Passive source queries still leak the
target to third parties; active modules contact the target directly and must be
covered by scope and written authorization.

## Related tools

- [`theharvester/`](../theharvester/README.md) — targeted harvesting
- [`recon-ng/`](../recon-ng/README.md) — modular framework
- [`metagoofil/`](../metagoofil/README.md) — documents
- [`sherlock/`](../sherlock/README.md), [`holehe/`](../holehe/README.md) — accounts
- [`ghunt/`](../ghunt/README.md) — Google OSINT

## References

- https://github.com/smicallef/spiderfoot
- https://www.spiderfoot.net/documentation/

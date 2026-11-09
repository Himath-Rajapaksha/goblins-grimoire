# recon-ng — modular OSINT and reconnaissance framework

- **Version:** 5.1.2
- **Binary:** `/usr/bin/recon-ng`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/lanmaster53/recon-ng
- **Docs:** https://github.com/lanmaster53/recon-ng/wiki

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

recon-ng is a Metasploit-style console for open-source intelligence. Each
module performs one task (query a data source, harvest hosts, resolve names,
scrape credentials), stores results in a per-workspace SQLite database, and can
feed other modules. Workspaces isolate engagements, `spool` records sessions,
and `marketplace` installs or removes modules.

## Use cases

| Situation | Why recon-ng |
|-----------|--------------|
| Domain reconnaissance | Chain discovery, DNS, and reporting modules |
| OSINT collection | Query public sources with API keys |
| Contact harvesting | Build emails, hosts, and profiles in one DB |
| Repeatable workflow | Resource files replay a whole session |
| Engagement isolation | One workspace per client |
| Report generation | Built-in reporting modules export HTML/JSON/CSV |

## Requirements & install layout

- Debian package 5.1.2; binary `/usr/bin/recon-ng`.
- Workspaces under `~/.recon-ng/workspaces/<name>/`.
- Modules download from the marketplace into `~/.recon-ng/modules/`; Debian
  builds may ship with the marketplace enabled.
- Many modules require API keys, set with `keys add <name> <value>`; keys are
  stored in the workspace database.
- Python 3 environment is self-contained; no pip install needed for packaged
  modules.

## Basic syntax

```bash
# start the console in a named workspace
recon-ng -w client-acme

# replay a resource file non-interactively
recon-ng -w client-acme -r script.rc

# no marketplace / no telemetry
recon-ng --no-marketplace --no-analytics -w client-acme

# stealth: disable all passive requests
recon-ng --stealth -w client-acme
```

Inside the console:

```text
marketplace search              # list available modules
marketplace install all         # install everything (large)
modules search                  # show installed modules
modules load recon/domains-hosts/hackertarget
options set SOURCE example.com
run
show hosts
back
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-w, --workspace <name>` | Load or create a workspace |
| `-r, --resource <file>` | Execute commands from a resource file |
| `--no-version` | Disable the version check |
| `--no-analytics` | Disable analytics reporting |
| `--no-marketplace` | Disable remote module management |
| `--stealth` | Disable all passive requests (implies `--no-*`) |
| `--accessible` | Use accessible output where available |
| `--version` | Print version and exit |
| `-h, --help` | Help |

### Console commands

| Command | Purpose |
|---------|---------|
| `workspaces` | List/create/select workspaces |
| `marketplace search/install/remove/info` | Manage modules |
| `modules search/load/reload` | Work with installed modules |
| `options list/set/unset` | Configure the loaded module |
| `run` | Execute the loaded module |
| `back` | Unload the module |
| `show <table>` | Inspect stored data (`hosts`, `contacts`, `domains`, ...) |
| `db insert/query/delete` | Direct database work |
| `keys add/list/remove` | Manage API keys |
| `spool start/stop` | Record console output |
| `script record/execute` | Record and replay resource files |
| `reporting` | Generate reports from the workspace |
| `exit` | Quit |

## Recipes

```bash
# 1. Create an isolated workspace
recon-ng -w acme-2026

# 2. Install a focused set of modules
#    (inside the console)
marketplace search domains-hosts
marketplace install recon/domains-hosts/hackertarget
marketplace install recon/hosts-hosts/resolve
marketplace install reporting/html

# 3. Seed a domain and pivot through modules
modules load recon/domains-hosts/hackertarget
options set SOURCE acme.example
run
modules load recon/hosts-hosts/resolve
run

# 4. Add API keys
keys add shodan_api <key>
keys add virustotal_api <key>

# 5. Inspect and export
show hosts
reporting load reporting/html
options set CREATOR "tester"
options set CUSTOMER "Acme"
options set FILENAME /evidence/acme-report.html
run

# 6. Replay everything from a resource file
recon-ng -w acme-2026 -r workflow.rc

# 7. Back up a workspace
sqlite3 ~/.recon-ng/workspaces/acme-2026/data.db '.backup /evidence/acme.db'

# 8. Start clean for the next engagement
recon-ng -w acme-2027
```

## Output & pipelines

- Per-workspace SQLite DB: `~/.recon-ng/workspaces/<name>/data.db`.
- Tables include `domains`, `hosts`, `contacts`, `credentials`, `locations`,
  `netblocks`, `ports`, `profiles`, `repositories`, `vulnerabilities`.
- `db query <SQL>` and `sqlite3` expose the raw data.
- Reporting modules write HTML/JSON/CSV/XML to a chosen path.
- Resource files (`-r`) make runs reproducible and auditable.

## Gotchas & operational security

- The marketplace pulls third-party code; review modules before installing and
  pin versions where possible.
- API keys are stored in the workspace DB — treat it as a secret.
- `--stealth` disables passive requests but not local processing.
- Module output quality varies; verify findings against a second source.
- `marketplace install all` is slow and bloats the module tree.
- Some modules break when a provider changes its HTML/API; check `modules
  search` and module info before relying on them.
- Interactive sessions expect a TTY; piping commands can loop on prompts.

## Legal & safety

Use only for authorized reconnaissance. Respect each data source's terms of
service, rate limits, and privacy law. Do not collect or retain personal data
beyond the engagement's scope.

## Related tools

- [`theharvester/`](../theharvester/README.md) — email/domain harvesting
- [`spiderfoot/`](../spiderfoot/README.md) — automated OSINT correlation
- [`sherlock/`](../sherlock/README.md), [`holehe/`](../holehe/README.md) — account discovery
- [`ghunt/`](../ghunt/README.md) — Google account OSINT
- [`metagoofil/`](../metagoofil/README.md) — document metadata

## References

- https://github.com/lanmaster53/recon-ng
- https://github.com/lanmaster53/recon-ng/wiki

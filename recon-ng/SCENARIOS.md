# recon-ng — command scenarios for deep research

Installed build: recon-ng 5.1.2. Launch flags come from `help.txt`; console
commands are documented in the recon-ng wiki and module help.

## How to invoke

```text
@deep-research For recon-ng 5.1.2 in 2026, which market modules still return
reliable domain/host/contact data without paid API keys, which require keys,
and how should a workspace be structured to avoid stale data across modules?
Cite the recon-ng wiki, module source, and provider docs; return a reproducible
resource-file workflow with fallbacks for dead modules.
```

---

## Scenario 1 — workspace and module setup

| # | Command | How to use |
|---|---------|------------|
| 1 | `recon-ng -w acme-2026` | Create an isolated workspace |
| 2 | `recon-ng --no-marketplace -w acme-2026` | Offline module set only |
| 3 | `recon-ng --stealth -w acme-2026` | Disable passive requests |
| 4 | `marketplace search` (console) | List available modules |
| 5 | `marketplace info recon/domains-hosts/hackertarget` | Read module metadata |
| 6 | `marketplace install recon/domains-hosts/hackertarget` | Install one module |

**Research questions**
- Which modules are Debian-packaged versus marketplace-only?
- How can module versions be pinned for reproducibility?

**Sources:** recon-ng wiki, marketplace metadata, Git history.
**Output:** a minimal, reviewed module set.

---

## Scenario 2 — domain and host discovery

| # | Command | How to use |
|---|---------|------------|
| 7 | `modules load recon/domains-hosts/hackertarget` | Load a discovery module |
| 8 | `options set SOURCE acme.example` | Set the seed domain |
| 9 | `run` | Execute the module |
| 10 | `show hosts` | Inspect stored hosts |
| 11 | `modules load recon/hosts-hosts/resolve` | Resolve hosts to IPs |
| 12 | `modules load recon/hosts-hosts/reverse_resolve` | Reverse DNS |
| 13 | `db query SELECT host, ip_address FROM hosts` | Direct SQL review |

**Research questions**
- Which discovery module has the best coverage per provider in 2026?
- Does `resolve` overwrite existing IPs or append?

**Sources:** module source, provider limits, DB schema.
**Output:** a host/IP table for the target domain.

---

## Scenario 3 — contacts, credentials, and profiles

| # | Command | How to use |
|---|---------|------------|
| 14 | `modules load recon/domains-contacts/whois_pocs` | WHOIS contacts |
| 15 | `modules load recon/domains-contacts/metacrawler` | Metadata contacts |
| 16 | `modules load recon/profiles-profiles/namechk` | Username profiling |
| 17 | `show contacts` | Review contacts table |
| 18 | `show profiles` | Review profiles table |
| 19 | `db query SELECT * FROM credentials` | Review captured creds |

**Research questions**
- Which contact modules comply with GDPR-style data minimisation?
- How accurate is username profiling across renamed services?

**Sources:** module source, provider Terms, privacy law.
**Output:** a scoped contact/profile dataset.

---

## Scenario 4 — API keys and rate limits

| # | Command | How to use |
|---|---------|------------|
| 20 | `keys add shodan_api <key>` | Store a provider key |
| 21 | `keys list` | List configured keys |
| 22 | `keys remove shodan_api` | Remove a key |
| 23 | `modules load recon/domains-hosts/shodan_hostname` | Key-dependent module |
| 24 | `options list` | Check required options |
| 25 | `run` | Execute with the key |

**Research questions**
- Where are keys stored, and are they encrypted at rest?
- Which free-tier quotas make a module impractical at scale?

**Sources:** recon-ng source, provider pricing pages.
**Output:** a key inventory and a quota-aware run plan.

---

## Scenario 5 — reporting, spooling, and reproducibility

| # | Command | How to use |
|---|---------|------------|
| 26 | `spool start /evidence/session.log` | Record console output |
| 27 | `script record workflow` | Record commands as a resource file |
| 28 | `script execute workflow` | Replay a recorded workflow |
| 29 | `modules load reporting/html` | Load the HTML report |
| 30 | `options set FILENAME /evidence/report.html` | Set output path |
| 31 | `run` | Generate the report |
| 32 | `recon-ng -w acme-2026 -r workflow.rc` | Non-interactive replay |
| 33 | `sqlite3 ~/.recon-ng/workspaces/acme-2026/data.db '.backup /evidence/acme.db'` | Back up the workspace |

**Research questions**
- Does `script record` capture `options set` values and keys?
- Which report format best preserves evidence provenance?

**Sources:** recon-ng wiki, report templates, evidence policy.
**Output:** a reproducible workflow, a report, and a database backup.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Launch | `-w -r --no-version --no-analytics --no-marketplace --stealth --accessible --version` | 1, 5 |
| Marketplace/modules | `search/info/install/remove`, `modules search/load` | 1, 2 |
| Data | `options`, `run`, `show`, `db query` | 2–4 |
| Evidence | `spool`, `script`, `reporting`, `sqlite3` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — setup | 6 |
| 2 — discovery | 7 |
| 3 — contacts | 6 |
| 4 — keys | 6 |
| 5 — reporting | 8 |

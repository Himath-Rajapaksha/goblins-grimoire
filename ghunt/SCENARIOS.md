# GHunt — command scenarios for deep research

Installed build: GHunt 2.3.4 (Spider Edition). Top-level commands come from
`help.txt`; per-command flags are in each subcommand's `--help`.

## How to invoke

```text
@deep-research For GHunt 2.3.4, which Google endpoints still return profile,
YouTube, Maps, and Drive data for an email in 2026, what rate limits and
blocking behaviour should be expected, and how should a sock-puppet session be
managed safely? Cite GHunt docs, issue reports, and Google terms; return a
minimal-query workflow with JSON evidence.
```

---

## Scenario 1 — authentication lifecycle

| # | Command | How to use |
|---|---------|------------|
| 1 | `ghunt login` | Import cookies and cache credentials |
| 2 | `ghunt login --clean` | Remove cached credentials |
| 3 | `ls ~/.config/ghunt` (or the documented path) | Inspect credential storage |
| 4 | `ghunt --help` | Confirm available commands |
| 5 | `ghunt <cmd> --help` | Confirm per-command flags |

**Research questions**
- Which cookies are required, and how long does a session last?
- Where exactly are credentials stored, and are they encrypted?

**Sources:** GHunt wiki, local config, source code.
**Output:** a working session and a documented teardown.

---

## Scenario 2 — email and Gaia lookups

| # | Command | How to use |
|---|---------|------------|
| 6 | `ghunt email target@gmail.com` | Full profile lookup |
| 7 | `ghunt email target@gmail.com --json target.json` | Structured output |
| 8 | `ghunt gaia <gaia-id>` | Gaia ID lookup |
| 9 | `ghunt email alias@example.com` | Non-Gmail address handling |
| 10 | `jq '.' target.json` | Review structured fields |

**Research questions**
- Which fields are returned for Gmail versus non-Gmail addresses?
- How is a Gaia ID extracted from a Google URL?

**Sources:** command output, wiki, source code.
**Output:** an identity profile with JSON evidence.

---

## Scenario 3 — Drive and linked assets

| # | Command | How to use |
|---|---------|------------|
| 11 | `ghunt drive 'https://drive.google.com/file/d/XXXX/view'` | File owner |
| 12 | `ghunt drive 'https://drive.google.com/drive/folders/XXXX'` | Folder owner |
| 13 | `ghunt spiderdal example.com` | Digital Asset Links assets |
| 14 | `ghunt gaia <id-from-drive-result>` | Pivot to profile |
| 15 | `ghunt email owner@gmail.com` | Confirm ownership |

**Research questions**
- What Drive metadata is visible without access to the file?
- Which Digital Asset Links relationships are most useful?

**Sources:** Drive API behaviour, DAL spec, GHunt source.
**Output:** asset ownership map.

---

## Scenario 4 — geolocation and Maps exposure

| # | Command | How to use |
|---|---------|------------|
| 16 | `ghunt geolocate AA:BB:CC:DD:EE:FF` | BSSID lookup |
| 17 | `ghunt email target@gmail.com` | Review Maps contributions |
| 18 | `ghunt geolocate <bssid-from-review-photo>` | Pivot from photo metadata |
| 19 | `ghunt email target@gmail.com --json maps.json` | Structured Maps data |
| 20 | `jq '.maps' maps.json` | Inspect location fields |

**Research questions**
- How precise is BSSID geolocation, and what database backs it?
- Which Maps contributions expose home or work locations?

**Sources:** Google Maps data, GHunt output, privacy research.
**Output:** a location-exposure assessment.

---

## Scenario 5 — evidence and operational safety

| # | Command | How to use |
|---|---------|------------|
| 21 | `ghunt email target@gmail.com --json /evidence/target.json` | Save evidence |
| 22 | `sha256sum /evidence/target.json` | Hash evidence |
| 23 | `ghunt login --clean` | Remove credentials |
| 24 | `rm -f /evidence/target.json` (after retention) | Destroy per policy |
| 25 | `ghunt --version` (or banner) | Record version |

**Research questions**
- What retention period applies to identity data?
- Does `--json` include fields hidden from the console view?

**Sources:** engagement policy, output comparison, privacy law.
**Output:** a minimal, hashed evidence set and a clean session.

## Flag coverage checklist

| Command | Major options | Scenario |
|---------|---------------|----------|
| login | `--clean` | 1, 5 |
| email | target, JSON/debug flags | 2, 4 |
| gaia | target ID | 2, 3 |
| drive | URL | 3 |
| geolocate | BSSID | 4 |
| spiderdal | domain | 3 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — auth | 5 |
| 2 — email/Gaia | 5 |
| 3 — Drive/assets | 5 |
| 4 — geolocation | 5 |
| 5 — evidence | 5 |

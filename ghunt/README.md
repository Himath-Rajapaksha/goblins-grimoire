# GHunt — Google account OSINT

- **Version:** GHunt 2.3.4 (Spider Edition)
- **Binary:** `/usr/local/bin/ghunt` (pipx)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/mxrch/GHunt

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

GHunt extracts public information tied to a Google account. Given an email
address or Gaia ID it can return the account's display name, profile picture,
YouTube channel, Google Maps reviews and photos, known locations, and linked
services. It works through Google's internal web APIs using a logged-in
account's cookies, so an authenticated `login` step is required.

## Use cases

| Situation | Why GHunt |
|-----------|-----------|
| Email attribution | Map an address to a real identity |
| Alias discovery | Find the Gaia ID behind a Google account |
| YouTube footprint | Retrieve the linked channel |
| Location exposure | Review Maps contributions and photos |
| Drive link analysis | Identify the owner of a shared file |
| BSSID geolocation | Approximate a Wi-Fi access point location |

## Requirements & install layout

- Installed via pipx; entry point `/usr/local/bin/ghunt`.
- Requires a Google account for authentication; credentials are cached
  locally after `login`.
- `login` accepts cookies exported from a browser (the interactive flow
  explains the required cookies).
- Google rate-limits and may temporarily block automated querying.
- No paid API key; it relies on undocumented endpoints that change.

## Basic syntax

```bash
# authenticate once and cache credentials
ghunt login

# clear cached credentials
ghunt login --clean

# look up an email address
ghunt email target@gmail.com

# resolve an email to a Gaia ID
ghunt gaia <gaia-id>

# inspect a Drive file or folder
ghunt drive <file-or-folder-url>

# geolocate a BSSID
ghunt geolocate AA:BB:CC:DD:EE:FF

# find assets from Digital Asset Links
ghunt spiderdal example.com
```

## Command reference

| Command | Purpose |
|---------|---------|
| `login` | Authenticate and cache Google cookies; `--clean` removes them |
| `email <address>` | Profile, YouTube, Maps, and linked-account data for an email |
| `gaia <id>` | Resolve a Gaia ID to account details |
| `drive <url>` | Owner and metadata for a Drive file/folder |
| `geolocate <bssid>` | Approximate location for a Wi-Fi BSSID |
| `spiderdal <domain>` | Enumerate assets via Digital Asset Links |

Common per-command options include JSON output and debug/colour controls; run
`ghunt <command> --help` for the exact flags in this build.

## Recipes

```bash
# 1. Authenticate (interactive cookie import)
ghunt login

# 2. Basic email lookup
ghunt email target@gmail.com

# 3. Save the result as JSON for parsing
ghunt email target@gmail.com --json target.json

# 4. Resolve a Gaia ID found in a URL
ghunt gaia 123456789012345678901

# 5. Identify the owner of a shared Drive file
ghunt drive 'https://drive.google.com/file/d/XXXX/view'

# 6. Approximate a BSSID location
ghunt geolocate 3C:5A:B4:11:22:33

# 7. Enumerate Digital Asset Links for a domain
ghunt spiderdal example.com

# 8. Clear credentials when finished
ghunt login --clean
```

## Output & pipelines

- Console profile with name, avatar, YouTube channel, Maps activity, and
  linked services.
- JSON output (where supported) for parsing and evidence.
- Gaia IDs link accounts across Google properties.
- Results feed `sherlock`/`holehe` correlation and reporting.

## Gotchas & operational security

- Requires a **real Google session**; use a dedicated sock-puppet account, not
  a personal one.
- Credentials are cached on disk — protect and delete them after the
  engagement.
- Google detects scraping; heavy use can suspend the sock-puppet account.
- Data can be stale or private; absence of a result is not proof.
- Undocumented endpoints break without notice; pin the version you tested.
- Results are personal data; collect the minimum and retain briefly.

## Legal & safety

Use only for authorized investigations and lawful purposes. Do not use GHunt
for stalking, harassment, or account takeover. Respect Google's terms of
service and applicable privacy law.

## Related tools

- [`holehe/`](../holehe/README.md) — which sites an email is registered on
- [`sherlock/`](../sherlock/README.md) — username footprint
- [`theharvester/`](../theharvester/README.md), [`spiderfoot/`](../spiderfoot/README.md)
- [`recon-ng/`](../recon-ng/README.md)

## References

- https://github.com/mxrch/GHunt
- https://github.com/mxrch/GHunt/wiki

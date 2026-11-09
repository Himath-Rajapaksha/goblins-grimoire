# sherlock — username search across social networks

- **Version:** 0.16.2
- **Binary:** `/usr/local/bin/sherlock` (pipx)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/sherlock-project/sherlock

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

sherlock checks a username against hundreds of social networks and websites by
requesting the profile URL for each site and looking for a "not found" marker.
It reports every site where the username appears to exist, and can export the
results to text, CSV, XLSX, or JSON. With `{?}` placeholders it also tests
common variants such as `user_name`, `user-name`, and `user.name`.

## Use cases

| Situation | Why sherlock |
|-----------|--------------|
| Username footprint | Find accounts across many platforms |
| Identity correlation | Link a handle to a person or brand |
| Brand monitoring | Check where a name is registered |
| Alias discovery | Test `_`, `-`, `.` variants |
| Evidence collection | Export results for a report |
| Pre-engagement OSINT | Build a target profile |

## Requirements & install layout

- Installed via pipx; entry point `/usr/local/bin/sherlock`.
- Site definitions ship in `data.json`; `--local` forces the bundled file,
  `--json` can load an upstream/PR version.
- No API keys.
- Network access to all listed sites; some block or rate-limit.
- `--browse` opens every hit in a browser and is best avoided in automation.

## Basic syntax

```bash
# basic username check
sherlock targetuser

# several usernames, save to a folder
sherlock -fo evidence targetuser alice bob

# single username, save to one file
sherlock -o target.txt targetuser

# export CSV and XLSX
sherlock --csv --xlsx targetuser

# test variants with {?}
sherlock 'target{?}user'

# limit to specific sites
sherlock --site GitHub --site Reddit targetuser

# include NSFW sites and show all results
sherlock --nsfw --print-all targetuser
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `USERNAMES...` | One or more usernames; `{?}` expands to `_`, `-`, `.` |
| `--version` | Version and dependency info |
| `-v, --verbose, -d, --debug` | Extra debug output |
| `-fo, --folderoutput <dir>` | Output folder for multiple usernames |
| `-o, --output <file>` | Output file for a single username |
| `--csv` | Write CSV |
| `--xlsx` | Write XLSX |
| `--site <name>` | Restrict to a site (repeatable) |
| `-p, --proxy <url>` | Proxy, e.g. `socks5://127.0.0.1:1080` |
| `--dump-response` | Print raw HTTP responses for debugging |
| `-j, --json <file>` | Load a custom/upstream site definition file |
| `--timeout <sec>` | Request timeout (default 60) |
| `--print-all` | Show misses as well as hits |
| `--print-found` | Show hits (also when exporting) |
| `--no-color` | Disable colour |
| `-b, --browse` | Open all hits in a browser |
| `-l, --local` | Force the local `data.json` |
| `--nsfw` | Include NSFW sites |
| `--txt` | Write a plain text file |
| `--ignore-exclusions` | Ignore upstream exclusions (more false positives) |

## Recipes

```bash
# 1. Plain check with colourless log output
sherlock --no-color targetuser | tee sherlock.log

# 2. Batch usernames into a folder, with CSV and TXT
sherlock -fo evidence --csv --txt targetuser alice bob

# 3. Single target, JSON-ish text and no colour
sherlock -o target.txt --no-color targetuser

# 4. Variant expansion
sherlock 'target{?}user' --print-found

# 5. Focused sites for a quick check
sherlock --site GitHub --site Reddit --site Mastodon.social targetuser

# 6. Slow/hostile network: proxy and longer timeout
sherlock -p socks5://127.0.0.1:1080 --timeout 90 targetuser

# 7. Debug one site's false negative
sherlock --site GitHub --dump-response targetuser

# 8. Ignore upstream exclusions when you need maximum coverage
sherlock --ignore-exclusions --print-all targetuser
```

## Output & pipelines

- Console lines group hits by site with the profile URL.
- `--csv`, `--xlsx`, `--txt` write alongside `-o`/`-fo`.
- Hits feed `holehe`, `ghunt`, `recon-ng`, and manual review.
- `--print-all` is useful for building a site coverage baseline.

## Gotchas & operational security

- False positives happen when a site returns HTTP 200 for missing profiles;
  always open the URL before reporting.
- False negatives happen with JavaScript-heavy sites, Cloudflare challenges,
  and 403 responses.
- The default timeout of 60s per site makes full runs slow; scope with
  `--site` or accept the wait.
- `--browse` can open hundreds of tabs — avoid in automation.
- `--nsfw` may return adult content; handle per policy.
- Requests come from your IP unless proxied and are visible to each site.
- `--ignore-exclusions` trades precision for coverage.

## Legal & safety

Use only for lawful, authorized research. Username correlation can identify
individuals; collect the minimum, verify before asserting identity, and comply
with privacy law and platform terms.

## Related tools

- [`holehe/`](../holehe/README.md) — email-side footprint
- [`ghunt/`](../ghunt/README.md) — Google account data
- [`recon-ng/`](../recon-ng/README.md), [`theharvester/`](../theharvester/README.md)
- [`spiderfoot/`](../spiderfoot/README.md)

## References

- https://github.com/sherlock-project/sherlock
- https://sherlock-project.github.io/

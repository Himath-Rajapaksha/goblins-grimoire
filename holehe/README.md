# holehe — email registration footprint across public sites

- **Version:** holehe v1.61
- **Binary:** `/usr/local/bin/holehe` (pipx)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/megadose/holehe

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

holehe tests whether an email address is registered on a large set of public
websites. For each site it uses the site's own "forgot password" or signup
flow and interprets the response: a request that reports "email already in
use" confirms a registration, while "address available" denies it. It does
not need an account on most sites and does not send mail unless password
recovery is enabled.

## Use cases

| Situation | Why holehe |
|-----------|------------|
| Email attribution | Find which services an address uses |
| Account inventory | Recover an old signup list for a target |
| Identity correlation | Link an address to social/consumer services |
| Phishing pre-check (authorized) | Confirm which services the recipient uses |
| Breach triage | See which platforms to check for exposure |
| Alias discovery | Distinguish addresses by service footprint |

## Requirements & install layout

- Installed via pipx; entry point `/usr/local/bin/holehe`.
- Pure Python; no API keys.
- Network access to many third-party sites; some are slow or rate-limited.
- Sites change their signup flows frequently, so results are probabilistic.
- Password recovery is attempted by default for sites that support it; disable
  with `-NP` to reduce side effects.

## Basic syntax

```bash
# check one address
holehe target@example.com

# show only positive results
holehe --only-used target@example.com

# check several addresses and write CSV
holehe -C target@example.com alias@example.com

# skip password-recovery requests
holehe -NP target@example.com

# increase per-site timeout on slow networks
holehe -T 20 target@example.com
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `EMAIL [EMAIL ...]` | One or more addresses to check |
| `--only-used` | Print only sites where the address exists |
| `--no-color` | Disable ANSI colour |
| `--no-clear` | Do not clear the terminal before results |
| `-NP, --no-password-recovery` | Do not trigger password-recovery flows |
| `-C, --csv` | Write a CSV file with all results |
| `-T, --timeout <sec>` | Per-request timeout (default 10) |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Full footprint, colourless, no screen clear (good for logs)
holehe --no-color --no-clear target@example.com | tee holehe.log

# 2. Only the hits
holehe --only-used target@example.com

# 3. Batch several addresses, export CSV
holehe -C -NP leader@example.com deputy@example.com

# 4. Slow network: raise timeout and disable recovery
holehe -T 30 -NP target@example.com

# 5. Strip colours and count confirmed sites
holehe --only-used --no-color target@example.com | wc -l

# 6. Merge results with other OSINT
holehe --only-used --no-color target@example.com | sort -u > used.txt

# 7. Check a role alias versus a personal address
holehe --only-used --no-color info@example.com
holehe --only-used --no-color person@example.com
```

## Output & pipelines

- Console lines: `[+] site` for registered, `[-] site` for not registered,
  `[x] site` for errors/unknown.
- `--only-used` reduces the output to positives only.
- `-C` writes `holehe_<timestamp>.csv` (or a named file in newer builds) in
  the working directory.
- Feed positives into `sherlock`, `recon-ng`, or manual verification.

## Gotchas & operational security

- Results are **inferred** from site responses; markup changes and bot
  protections cause false negatives and false positives.
- Some sites rate-limit or block after a handful of queries; the timeout and
  error lines matter.
- Password recovery may send mail to the target — use `-NP` unless the RoE
  permits it.
- The tool contacts dozens of third parties, leaking the target address to
  each; this is not stealthy.
- Some sites require a browser session and always fail in CLI mode.
- Do not treat a positive as proof of ownership; verify with a second source.

## Legal & safety

Only check addresses you are authorized to investigate. Even without password
recovery, querying signup flows is an active interaction with third-party
services and may breach their terms. Handle the resulting account map as
personal data.

## Related tools

- [`sherlock/`](../sherlock/README.md) — username-side footprint
- [`ghunt/`](../ghunt/README.md) — Google account details
- [`theharvester/`](../theharvester/README.md), [`spiderfoot/`](../spiderfoot/README.md)
- [`recon-ng/`](../recon-ng/README.md)

## References

- https://github.com/megadose/holehe
- https://pypi.org/project/holehe/

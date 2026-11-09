# WhatWeb — command scenarios for deep research

Installed build: WhatWeb 0.5.5 (Debian package `whatweb`). Every flag below
appears in [`help.txt`](help.txt). The plugin library at
`/usr/share/whatweb/plugins/` contains roughly 1,800 plugin files.

> **Intrusive at higher aggression:** `-a 3` and `-a 4` make additional requests
> per target and can trigger WAF or rate-limit controls. Use only on authorized
> targets and agree on an aggression level before a bulk scan.

## How to invoke

```text
@deep-research Compare WhatWeb 0.5.5, httpx, and Wappalyzer-style fingerprinting
for a mixed estate of CMS sites, embedded devices, and custom applications.
Which tool gives the most reliable version evidence at a given request budget,
how does WhatWeb's aggression level change the request count, and when is a
plugin hit too weak to report? Cite the WhatWeb plugin source, the httpx
documentation, and independent comparisons; return a per-target plan with an
evidence standard.
```

Create the demo inputs, replacing addresses with in-scope targets:

```bash
mkdir -p ~/whatweb-demo/evidence
printf 'https://target.example/\nhttps://app.target.example/\n' > ~/whatweb-demo/targets.txt
printf 'session=abc123\n' > ~/whatweb-demo/cookies.txt
```

## Scenario 1 — baseline fingerprint

Start at the default aggression and record exactly what one request per target
can prove. Then decide whether a more intrusive level is justified.

| # | Command | How to use |
|---|---------|------------|
| 1 | `whatweb https://target.example/` | Default stealthy scan |
| 2 | `whatweb -v https://target.example/` | Verbose plugin descriptions |
| 3 | `whatweb -a 3 https://target.example/` | Aggressive version detection |
| 4 | `whatweb --version` | Confirm the installed build |
| 5 | `whatweb --short-help` | Short usage summary |

**Research questions**
- Which plugins match at aggression 1, and which need aggression 3?
- How many requests does each level actually send to this target?
- Which reported versions are exact, and which are ranges or guesses?
- What does verbose mode add that is useful in a report?

**Sources:** WhatWeb plugin source; `man whatweb`; HTTP response headers.
**Output:** a per-level request count and a technology table with confidence.

## Scenario 2 — bulk targets and URL patterns

WhatWeb accepts CIDR, ranges, files, and stdin. URL modification lets one
target list drive a specific path on every host.

| # | Command | How to use |
|---|---------|------------|
| 6 | `whatweb --no-errors 192.168.10.0/24` | CIDR sweep |
| 7 | `whatweb --no-errors --url-prefix https:// 192.168.10.0/24` | Force HTTPS |
| 8 | `whatweb -i ~/whatweb-demo/targets.txt` | Targets from a file |
| 9 | `cat ~/whatweb-demo/targets.txt \| whatweb -i /dev/stdin` | Targets from stdin |
| 10 | `whatweb --url-suffix /robots.txt https://target.example/` | Append a path |
| 11 | `whatweb --url-pattern 'https://target.example/%insert%/robots.txt' admin api` | Insert each target into a pattern |
| 12 | `whatweb --url-prefix https:// --url-suffix /login https://target.example/` | Prefix and suffix together |

**Research questions**
- How does WhatWeb expand `x.x.x-x` and CIDR ranges?
- Does `--url-prefix` override an existing scheme or add to it?
- What happens when a target in the file is unreachable?
- How do you keep a bulk scan inside scope when redirects are enabled?

**Sources:** `man whatweb`; RFC 3986; engagement scope list.
**Output:** a normalized target list and a per-host result row.

## Scenario 3 — plugin selection and search

The default plugin set is broad. Selecting a small list makes a scan faster and
the result easier to defend.

| # | Command | How to use |
|---|---------|------------|
| 13 | `whatweb -l \| grep -i wordpress` | Find plugin names |
| 14 | `whatweb --search-plugins=wordpress` | Search plugin metadata |
| 15 | `whatweb -I=wordpress` | Detailed plugin information |
| 16 | `whatweb -p wordpress,php https://target.example/` | Run selected plugins |
| 17 | `whatweb -p + https://target.example/` | Include disabled plugins |
| 18 | `whatweb --dorks=wordpress` | Google dorks for a plugin |
| 19 | `whatweb --custom-plugin=':text=>"powered by acme"' https://target.example/` | Inline plugin |
| 20 | `whatweb -g 'WordPress\|PHP' https://target.example/` | Show only matching results |

**Research questions**
- Which plugins are disabled by default, and why?
- What does a plugin's `:text`, `:version`, `:md5`, and `:ghdb` matcher mean?
- When is an inline `--custom-plugin` better than writing a plugin file?
- How reliable is `--dorks` for finding exposed instances?

**Sources:** WhatWeb plugin development documentation; plugin source;
`--info-plugins` output.
**Output:** a selected-plugin manifest and a matcher-to-evidence table.

## Scenario 4 — HTTP behavior, auth, and proxy

Headers, redirects, cookies, and proxy settings change both reachability and
what the target reveals.

| # | Command | How to use |
|---|---------|------------|
| 21 | `whatweb -U 'Mozilla/5.0 (X11; Linux x86_64)' https://target.example/` | Custom user agent |
| 22 | `whatweb -H 'X-Forwarded-For: 127.0.0.1' https://target.example/` | Add a header |
| 23 | `whatweb --follow-redirect=never https://target.example/` | Do not follow redirects |
| 24 | `whatweb --max-redirects=3 https://target.example/` | Cap redirects |
| 25 | `whatweb -u 'alice:Summer2026!' https://target.example/` | Basic auth |
| 26 | `whatweb -c 'session=abc123' https://target.example/` | Cookie header |
| 27 | `whatweb --cookie-jar ~/whatweb-demo/cookies.txt https://target.example/` | Cookie jar |
| 28 | `whatweb --proxy 127.0.0.1:8080 --proxy-user 'proxyuser:proxypass' https://target.example/` | Authenticated proxy |

**Research questions**
- Which headers does WhatWeb send by default, and which reveal it?
- Does `--follow-redirect=never` change the plugin matches?
- When does basic auth hide the real application behind a login page?
- How do you prove the proxy is in the path?

**Sources:** RFC 7231 (HTTP semantics); RFC 6265 (cookies); WhatWeb source.
**Output:** a request/response transcript per variant and a match delta.

## Scenario 5 — performance and structured output

Bulk scans need thread and timeout control, and reports need a machine-readable
format.

| # | Command | How to use |
|---|---------|------------|
| 29 | `whatweb -t 10 --open-timeout 10 --read-timeout 20 https://target.example/` | Threads and timeouts |
| 30 | `whatweb --wait 1 https://target.example/` | One second between connections |
| 31 | `whatweb --log-json ~/whatweb-demo/evidence/results.json https://target.example/` | JSON output |
| 32 | `whatweb --log-xml ~/whatweb-demo/evidence/results.xml --log-brief ~/whatweb-demo/evidence/results.txt https://target.example/` | XML and brief text |
| 33 | `whatweb --log-sql ~/whatweb-demo/evidence/results.sql --log-sql-create ~/whatweb-demo/evidence/tables.sql https://target.example/` | SQL output |
| 34 | `whatweb --log-verbose ~/whatweb-demo/evidence/verbose.txt --log-errors ~/whatweb-demo/evidence/errors.txt https://target.example/` | Verbose and error logs |
| 35 | `whatweb -q --no-errors --colour=never https://target.example/` | Quiet, no colour |

**Research questions**
- Which log format preserves plugin names, versions, and evidence?
- How do `-t`, `--open-timeout`, and `--read-timeout` interact on a slow target?
- Does `--wait` apply per target or per connection?
- Which output format is easiest to diff between two scans?

**Sources:** `man whatweb`; WhatWeb logging source; JSON/XML/SQL specifications.
**Output:** one result set in JSON, XML, and SQL, with a schema note.

## Scenario 6 — verify and report

A plugin match is a hypothesis. Confirm the important ones with a second
request or a second tool before reporting a version.

| # | Command | How to use |
|---|---------|------------|
| 36 | `jq '.[] \| {target, plugins}' ~/whatweb-demo/evidence/results.json` | Summarize JSON |
| 37 | `grep -i 'wordpress\|php' ~/whatweb-demo/evidence/results.txt` | Quick text filter |
| 38 | `whatweb -a 3 -p wordpress https://target.example/` | Re-check one plugin |
| 39 | `curl -sS -I https://target.example/` | Independent header check |
| 40 | `shred -u ~/whatweb-demo/cookies.txt` | Remove session material |

**Research questions**
- Which reported versions survive an independent check?
- What evidence is enough to say "WordPress 6.x" rather than "WordPress"?
- How do you report a WAF block without overstating the application stack?
- What should be redacted from a fingerprint report?

**Sources:** vendor version documentation; independent HTTP response; CVE/NVD
for the version-to-vulnerability step (not WhatWeb itself).
**Output:** a verified technology table with confidence and evidence per row.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| targets, `-i`, `--url-prefix`, `--url-suffix`, `--url-pattern` | 1, 2 |
| `-a`, `-U`, `-H`, `--follow-redirect`, `--max-redirects` | 1, 4 |
| `-u`, `-c`, `--cookie-jar`, `--proxy`, `--proxy-user` | 4 |
| `-l`, `-I`, `--search-plugins`, `-p`, `-g`, `--custom-plugin`, `--dorks` | 3 |
| `-v`, `--colour`, `-q`, `--no-errors` | 1, 5 |
| `--log-brief`, `--log-verbose`, `--log-errors`, `--log-xml`, `--log-json`, `--log-json-verbose`, `--log-sql`, `--log-sql-create`, `--log-magictree`, `--log-object`, `--log-mongo-*`, `--log-elastic-*` | 5 |
| `-t`, `--open-timeout`, `--read-timeout`, `--wait` | 5 |
| `--short-help`, `--help`, `--debug`, `--version` | 1 |

Defaults: aggression 1, 25 threads, 15 s open timeout, 30 s read timeout,
10 redirects, proxy port 8080. Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Baseline fingerprint | 5 |
| 2 | Bulk targets and URL patterns | 7 |
| 3 | Plugin selection and search | 8 |
| 4 | HTTP behavior, auth, and proxy | 8 |
| 5 | Performance and structured output | 7 |
| 6 | Verify and report | 5 |
| **Total** | | **40** |

## Legal and safety

Fingerprinting sends real requests and higher aggression levels can look like
scanning. Use only on authorized targets, keep redirects inside scope, and do
not treat a version match as proof of a vulnerability. Authenticated scans
should use a dedicated, least-privilege account and delete the cookie material
afterward.

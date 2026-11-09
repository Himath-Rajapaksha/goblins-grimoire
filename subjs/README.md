# subjs — collect JavaScript URLs from web pages

- **Version:** 1.0.1
- **Binary:** `/usr/local/bin/subjs` (Go)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/lc/subjs

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

subjs takes a list of URLs — usually live hosts or pages from a reconnaissance
pipeline — requests each one, and prints the JavaScript file URLs referenced by
the returned HTML. It resolves relative script paths and emits one URL per
line, which makes the output easy to deduplicate and hand to a JavaScript
analyzer.

It is a discovery tool, not a crawler and not a JavaScript engine. It does not
execute scripts, does not follow links recursively, and does not send cookies or
arbitrary headers. Its job is the first step of a JavaScript review: find out
which bundles and vendor files the application actually loads.

## Use cases

| Situation | Why subjs |
|-----------|-----------|
| Mapping a web app's client-side surface | Enumerate bundles before reading them |
| Bug-bounty recon | Turn a host list into a JS URL list quickly |
| Secret and endpoint hunting | Feed LinkFinder, SecretFinder, or sourcemapper |
| Comparing environments | Diff JS URLs between staging and production |
| Pre-engagement inventory | Establish what client code is reachable |
| Pipeline glue | stdin/stdout works with `gau`, `httpx`, and `sort` |

## Requirements & install layout

- Single Go binary at `/usr/local/bin/subjs`; no runtime dependencies.
- No configuration file and no API key.
- Reads URLs from stdin or from `-i`; both can be used, but pick one for
  predictable ordering.
- No cookie, header, or proxy flag. Authenticated or header-gated pages will
  usually return a login page, so the extracted JS list will be incomplete.
- Outbound HTTP(S) only; the binary does not parse non-HTTP schemes.
- Concurrency and timeout are the only performance controls.

## Basic syntax

```bash
# stdin
cat urls.txt | subjs

# input file
subjs -i urls.txt

# tune concurrency and timeout
subjs -i urls.txt -c 40 -t 20

# custom user agent
subjs -i urls.txt -ua 'Mozilla/5.0 (X11; Linux x86_64)'

# version
subjs -version
```

## Flag reference

The installed build exposes five flags. There is no separate help text beyond
Go's usage block.

| Flag | Meaning |
|------|---------|
| `-c <int>` | Number of concurrent workers (default `10`) |
| `-i <file>` | Input file containing URLs; if omitted, read stdin |
| `-t <int>` | HTTP client timeout in seconds (default `15`) |
| `-ua <string>` | `User-Agent` sent with every request |
| `-version` | Print the version number and exit |

Go's flag package also accepts `-h`, which prints the usage block reproduced in
`help.txt`.

## Recipes

```bash
# 1. Basic host-to-JS conversion
cat live-hosts.txt | subjs > js-urls.txt

# 2. File input with more workers
subjs -i live-hosts.txt -c 40 -t 20 > js-urls.txt

# 3. Full recon chain: URLs from gau, then JS discovery
cat hosts.txt | gau | subjs | sort -u > js-urls.txt

# 4. Pretend to be a browser when a default Go UA is blocked
subjs -i live-hosts.txt -ua 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)' > js-urls.txt

# 5. Count unique bundles before analysis
subjs -i live-hosts.txt | sort -u | wc -l

# 6. Keep stable input and output for a repeatable run
sort -u live-hosts.txt > hosts.sorted.txt
subjs -i hosts.sorted.txt -c 20 -t 15 | sort -u > js-urls.txt
```

## Output & pipelines

- stdout: one absolute JavaScript URL per line.
- stderr: progress and errors from the HTTP client, depending on the build.
- No JSON, CSV, or status code in the output; pair with `httpx` when status
  and content type matter.
- Recommended handoffs:
  - `httpx` — verify each JS URL is live and record status/MIME.
  - `linkfinder` — extract endpoints from the downloaded JavaScript.
  - `secretfinder` — search bundles for keys and tokens.
  - `sourcemapper` — recover original sources when a `.map` file exists.
- Deduplicate with `sort -u`; the same bundle often appears on many pages.

## Gotchas & operational security

- No cookie or arbitrary-header support. Authenticated applications will yield
  only the login page's scripts unless the pages are public.
- JavaScript injected at runtime (lazy chunks, `import()` calls) is invisible
  to subjs; it only sees script references present in the initial HTML.
- The default Go user agent is often blocked or challenged; set `-ua` and
  compare results.
- A 403, WAF challenge, or redirect to a login page still exits successfully;
  check the output count, not just the exit code.
- High `-c` values can trip rate limits or IDS. Start at the default and raise
  it deliberately.
- `-t` is per request, not a global deadline; a large input with many slow
  hosts can still take a long time.
- Output is not deduplicated; always pipe through `sort -u`.
- Script URLs may include cache-busting parameters; normalize only after
  recording the original form.
- Only scan hosts in scope. Automated requests are visible in logs and may
  violate hosting terms.

## Legal & safety

Use subjs only against domains you own or are explicitly authorized to assess.
It makes real HTTP requests and will appear in server logs, CDN analytics, and
WAF telemetry. Do not point it at third-party sites, and do not use it to
enumerate applications outside an agreed scope.

## Related tools

- [`linkfinder/`](../linkfinder/README.md) — endpoints inside JavaScript
- [`secretfinder/`](../secretfinder/README.md) — secrets inside JavaScript
- [`sourcemapper/`](../sourcemapper/README.md) — recover source from maps
- [`x8/`](../x8/README.md) — parameter discovery on discovered endpoints
- [`httpx/`](../httpx/README.md) — live-host and status verification

## References

- https://github.com/lc/subjs
- https://github.com/lc/gau
- https://github.com/GerbenJavado/LinkFinder

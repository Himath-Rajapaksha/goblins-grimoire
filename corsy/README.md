# corsy — CORS misconfiguration scanner

- **Version:** 1.0-beta (banner; upstream `CHANGELOG.md` stops at 0.2-beta)
- **Binary:** `/usr/local/bin/corsy` — wrapper that runs `python3 /opt/Corsy/corsy.py`
- **Source checkout:** `/opt/Corsy`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/s0md3v/Corsy
- **Docs:** https://github.com/s0md3v/Corsy#readme

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Corsy tests whether a web endpoint's CORS policy can be abused by an
attacker-controlled origin. It sends GET requests with forged `Origin` values
and inspects the `Access-Control-Allow-Origin` (ACAO) and
`Access-Control-Allow-Credentials` (ACAC) response headers. If the server
reflects the forged origin — or accepts a look-alike, `null`, or plain-HTTP
origin — the endpoint may let a malicious site read authenticated responses.

This build runs ten checks, each tied to an entry in
`/opt/Corsy/db/details.json`:

| Check | What it proves |
|-------|----------------|
| `wildcard value` | ACAO is `*` on the baseline response (passive) |
| `third party allowed` | ACAO names a host unrelated to the target (passive) |
| `origin reflected` | Any `Origin` value is echoed back |
| `post-domain wildcard` | `target.example.com.attacker.tld` is accepted (suffix match) |
| `pre-domain wildcard` | `attacker-target.example.com` is accepted (prefix match) |
| `null origin allowed` | The literal `Origin: null` is accepted |
| `unrecognized underscore` | `target.example.com_.attacker.tld` passes the check |
| `broken parser` | A backtick in the origin breaks the parser into accepting it |
| `unescaped regex` | A dot in the allow-list regex matches another character |
| `http origin allowed` | An `http://` origin is accepted on an HTTPS target |

Each finding carries a class, description, severity, exploitation note, and the
observed ACAO/ACAC values. The scan stops at the first hit per URL.

## Use cases

| Situation | Why corsy |
|-----------|-----------|
| API recon on an authorized target | Fast, low-noise check for the common CORS mistakes |
| Bug-bounty triage | Quickly separates "reflects any origin" from "no CORS here" |
| Post-auth API review | Confirms whether credentialed cross-origin reads are possible |
| Regression testing after a CORS fix | Re-run the same URL list and diff the JSON |
| Pre-engagement scoping | Cheap way to find endpoints worth manual testing |
| Learning CORS bypass patterns | Output explains class and exploitation per finding |

Corsy is not a general web scanner and does not test other headers, methods, or
preflight behaviour in depth. Treat it as a first-pass filter.

## Requirements & install layout

- Python 3.4 or newer (the script exits early on older interpreters).
- The `requests` library; it is installed system-wide on this host.
- Git checkout at `/opt/Corsy`, invoked through the `/usr/local/bin/corsy`
  shell wrapper. No root privileges are needed to run scans.
- Per-run data file `/opt/Corsy/db/details.json` — the class/severity/
  exploitation text. It is read-only at scan time.
- No config directory, API keys, or persistent state. Output goes to stdout
  and optionally to the file given with `-o`.

## Basic syntax

```bash
corsy -u https://app.example.com/api/profile
corsy -i /home/user/scope/urls.txt -o /home/user/findings/corsy.json
cat /home/user/scope/live.txt | corsy -t 10
corsy -u https://app.example.com -q
```

`-u` and `-i` can be combined. If stdin is not a TTY, URLs are also read from
stdin (one per line).

## Flag reference

### Targeting and input

| Flag | Meaning |
|------|---------|
| `-u TARGET` | A single target URL |
| `-i INP_FILE` | File of URLs/subdomains, one per line |

URLs must begin with `http://` or `https://`; anything else is silently ignored.
With no `-u`, `-i`, or piped input the script prints `No valid URLs to test.`
and exits.

### Output

| Flag | Meaning |
|------|---------|
| `-o JSON_FILE` | Write findings as indented JSON |
| `-q` | Do not print description, severity, and exploitation tips |

`-q` does not suppress the finding class or the ACAO/ACAC values.

### Performance

| Flag | Meaning |
|------|---------|
| `-t THREADS` | Worker threads (default `2`) |
| `-d DELAY` | Delay in seconds between requests (default `0`) |

### Request headers

| Flag | Meaning |
|------|---------|
| `--headers [HEADER_DICT]` | Extra request headers |

The value is a single string with headers separated by a literal `\n`
(backslash followed by `n`), for example
`'Cookie: session=x\nAuthorization: Bearer y'`. Without a value, corsy opens
`nano` so you can type headers interactively. The supplied headers replace the
built-in browser-like defaults, and `Origin` is still overwritten per test.

### Help

| Flag | Meaning |
|------|---------|
| `-h, --help` | Show the option list and exit |

## Recipes

```bash
# Single endpoint, quiet output for a terminal
corsy -u https://app.example.com/api/profile -q

# One JSON artifact for the whole engagement
corsy -i /home/user/scope/api-urls.txt -o /home/user/findings/corsy.json

# Pipe live hosts from httpx, keep the terminal readable
httpx -l /home/user/scope/subs.txt -silent \
  | corsy -t 10 -d 1 -q -o /home/user/findings/corsy.json

# Authenticated API surface: pass session material as literal \n-separated headers
corsy -u https://app.example.com/api/account -q \
  --headers 'Cookie: session=REDACTED\nAuthorization: Bearer REDACTED'

# Gentle scan of a fragile host (serial, one second apart)
corsy -i /home/user/scope/fragile.txt -t 1 -d 1 -q

# Confirm a reported issue by hand with the same forged origin
curl -sS -D - -o /dev/null -H 'Origin: https://example.com' \
  https://app.example.com/api/profile | grep -i 'access-control-'
```

Manual confirmation matters: corsy reports a possible misconfiguration, not a
proven exploit.

## Output & pipelines

Normal output is one block per finding:

```text
 + https://app.example.com/api/profile
   - Class: origin reflected
   - Description: This host allows any origin to make requests to it.
   - Severity: high
   - Exploitation: Make requests from any domain you control.
   - ACAO Header: https://example.com
   - ACAC Header: true
```

With `-o`, results are also written as a JSON object keyed by URL:

```json
{
    "https://app.example.com/api/profile": {
        "class": "origin reflected",
        "description": "This host allows any origin to make requests to it.",
        "severity": "high",
        "exploitation": "Make requests from any domain you control.",
        "acao header": "https://example.com",
        "acac header": "true"
    }
}
```

Because the JSON is keyed by URL, one URL can hold only one finding — the first
test that matches wins. Merge results from separate runs with `python3` or `jq`
if you need a complete picture. Corsy JSON feeds report generation and can be
joined against `httpx` output by URL.

## Gotchas & operational security

- **Silent input rejection.** `http://` and `https://` prefixes are mandatory;
  bare hosts are dropped without an error per host.
- **Query strings are discarded.** The scanner rebuilds the URL from scheme,
  host, and path only. Test a query-string-specific endpoint manually.
- **One finding per URL.** Checks return at the first match, so a URL with both
  a wildcard and a reflected origin reports only the first.
- **Baseline must expose ACAO.** If the first request has no
  `Access-Control-Allow-Origin` header, that URL is skipped entirely.
- **TLS verification is off.** `requests.get(..., verify=False)` is used, which
  is convenient for staging but hides certificate problems.
- **Threads default to 2.** Raising `-t` without a delay can trip rate limits
  or WAFs; the pre-scan time estimate assumes 1.75 seconds per URL.
- **`--headers` parsing is literal.** Use single quotes and a literal `\n`;
  real newlines from `$'...'` break parsing. Supplying `--headers` with no value
  launches an interactive editor, which will hang a non-interactive pipeline.
- **No JSON on a clean run.** The file is only written when at least one
  finding exists, so a missing file can mean "no findings" rather than "failed".
- **False positives exist.** Look-alike-domain checks depend on DNS and server
  parsing quirks; always reproduce with `curl` before reporting.

## Legal & safety

Use corsy only against hosts you own or are explicitly authorized to test.
It sends active requests with attacker-shaped `Origin` headers and will appear
in access logs, WAF alerts, and SIEM dashboards. Even though the requests are
GETs and do not change server state, they are still active testing: scope them
to the program's asset list and respect rate limits. Do not point it at shared
third-party services, and do not include live session cookies in command
history or shell history — prefer environment variables or a protected file
that you delete afterwards.

## Related tools

- [httpx](../httpx/README.md) — produce the live-URL list corsy consumes.
- [katana](../katana/README.md) — crawl API and JavaScript endpoints first.
- [nuclei](../nuclei/README.md) — template coverage for CORS and other web issues.
- [burp](../burp/README.md) — manual confirmation and request replay.
- [dalfox](../dalfox/README.md) — chain a CORS finding with XSS on an allowed origin.
- [smuggler](../smuggler/README.md) — another request-layer parser flaw class.

## References

- Upstream README: https://github.com/s0md3v/Corsy
- Check implementations: `/opt/Corsy/core/tests.py`
- Finding text and severities: `/opt/Corsy/db/details.json`
- MDN CORS overview: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- PortSwigger CORS labs: https://portswigger.net/web-security/cors

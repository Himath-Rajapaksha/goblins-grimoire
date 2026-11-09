# ffuf — command scenarios for deep research

Installed build: ffuf 2.1.0-dev. Flags come from [`help.txt`](help.txt).
Commands assume an authorized target and a wordlist path that exists on the
host.

> **Intrusive:** content and credential fuzzing sends many requests and can
> lock accounts or trigger alerts. Use only within written scope.

## How to invoke

```text
@deep-research Which ffuf matcher/filter combination reliably separates real
endpoints from soft-404 noise on modern SPA and CMS targets, and how should
auto-calibration be validated before a large run? Cite the ffuf README, HTTP
status semantics, and field reports; return a calibration procedure with a
stop condition and an evidence format.
```

Prepare the demo directory and a small wordlist:

```bash
mkdir -p ~/ffuf-demo/evidence
printf 'admin\nlogin\napi\nbackup\n' > ~/ffuf-demo/words.txt
printf 'app.example.com\napi.example.com\n' > ~/ffuf-demo/vhosts.txt
```

## Scenario 1 — establish the baseline response

Before fuzzing, request a path that certainly does not exist. Its status, size,
and body are the calibration target.

| # | Command | How to use |
|---|---------|------------|
| 1 | `curl -sS -o /dev/null -w '%{http_code} %{size_download}\n' https://target/this-path-should-not-exist-12345` | Record the soft-404 shape |
| 2 | `ffuf -w ~/ffuf-demo/words.txt -u https://target/FUZZ -mc all -c` | See every response class |
| 3 | `ffuf -w ~/ffuf-demo/words.txt -u https://target/FUZZ -ac -c` | Let ffuf calibrate |
| 4 | `ffuf -w ~/ffuf-demo/words.txt -u https://target/FUZZ -mc all -fs 1234` | Manual size filter |
| 5 | `ffuf -w ~/ffuf-demo/words.txt -u https://target/FUZZ -mc all -fr 'not found'` | Manual body filter |

**Research questions**
- Does the target return a uniform soft-404, a redirect, or a real 404?
- Does `-ac` choose the same filter a human would?
- Which dimension (status, size, words, lines, regex) is most stable?

**Sources:** ffuf README; the baseline responses; server error pages.
**Output:** a documented calibration value and the command that reproduces it.

## Scenario 2 — directory and file discovery

Choose the wordlist by target type, not by size. A CMS needs CMS-specific
paths; a custom app needs a generic list plus extensions.

| # | Command | How to use |
|---|---------|------------|
| 6 | `ffuf -w words.txt -u https://target/FUZZ -ac -t 40` | Generic discovery |
| 7 | `ffuf -w words.txt -u https://target/FUZZ -e .php,.bak,.old,.zip` | Extension sweep |
| 8 | `ffuf -w words.txt -u https://target/FUZZ -D -e .php` | DirSearch-compatible mode |
| 9 | `ffuf -w words.txt -u https://target/FUZZ -recursion -recursion-depth 2 -ac` | Recursive |
| 10 | `ffuf -w words.txt -u https://target/FUZZ -mc 200,301,302,403 -o evidence/dirs.json -of json` | Durable output |

**Research questions**
- Which extensions actually exist on this stack?
- Does recursion find directories that a flat wordlist misses?
- How much does `-ac` reduce false positives compared with `-mc 200`?

**Sources:** ffuf README; application fingerprint; response bodies via `-od`.
**Output:** a result table with status, size, and a reviewed body sample.

## Scenario 3 — vhost and Host-header discovery

Vhost fuzzing finds names that resolve only through the web server's config.
The URL stays fixed; the `Host` header changes.

| # | Command | How to use |
|---|---------|------------|
| 11 | `ffuf -w vhosts.txt -u https://10.10.10.5/ -H 'Host: FUZZ' -mc 200 -fs 0` | Bare names |
| 12 | `ffuf -w vhosts.txt -u https://10.10.10.5/ -H 'Host: FUZZ.example.com' -mc all -ac` | Full domain suffix |
| 13 | `ffuf -w vhosts.txt -u http://10.10.10.5/ -H 'Host: FUZZ' -mc 301,302 -r` | Redirect-based vhosts |
| 14 | `ffuf -w vhosts.txt -u https://10.10.10.5/ -H 'Host: FUZZ' -fs 4242 -v` | Filter the default vhost |

**Research questions**
- Does the default vhost return a unique size that can be filtered?
- Are vhosts reachable over HTTP, HTTPS, or both?
- Do discovered names resolve in DNS, or only in the web server?

**Sources:** ffuf README; `curl -H 'Host: ...'` comparison; DNS lookups.
**Output:** a list of vhosts with the response that proves each one.

## Scenario 4 — parameter and value fuzzing

Parameter discovery is noisier than directory discovery because every response
may be valid. Match on reflection, error text, or a measurable difference.

| # | Command | How to use |
|---|---------|------------|
| 15 | `ffuf -w params.txt:PARAM -u 'https://target/?PARAM=test' -mr 'PARAM' -c` | Reflected parameters |
| 16 | `ffuf -w values.txt -u 'https://target/?id=FUZZ' -mc all -fs 1200` | Value fuzzing |
| 17 | `ffuf -w values.txt -u https://target/api -X POST -H 'Content-Type: application/json' -d '{"role":"FUZZ"}' -mc all -fr 'error'` | JSON body |
| 18 | `ffuf -w users.txt:USER -w passes.txt:PASS -u https://target/login -X POST -d 'user=USER&pass=PASS' -mc all -fr 'invalid' -mode clusterbomb` | Credential fuzzing (authorized only) |
| 19 | `ffuf -w params.txt:PARAM -u 'https://target/?PARAM=test' -mc all -mr 'root:\|admin'` | Interesting values |

**Research questions**
- Which parameters change the response, and how?
- Does the app reflect input, error, or silently ignore it?
- Which match dimension gives the cleanest signal?

**Sources:** ffuf README; parameter-mining methodology; response diffs.
**Output:** a parameter list with the observed behavior for each.

## Scenario 5 — authenticated and throttled runs

Authenticated fuzzing needs the exact session material. Throttling keeps a
monitored target responsive and avoids account lockout.

| # | Command | How to use |
|---|---------|------------|
| 20 | `ffuf -w words.txt -u https://target/admin/FUZZ -b 'SESSION=abc123' -ac` | Cookie auth |
| 21 | `ffuf -w words.txt -u https://target/admin/FUZZ -H 'Authorization: Bearer TOKEN' -ac` | Bearer auth |
| 22 | `ffuf -w words.txt -u https://target/FUZZ -rate 20 -p 0.2 -t 5` | Rate-limited |
| 23 | `ffuf -w words.txt -u https://target/FUZZ -timeout 20 -p 0.2 -t 5` | Slower target |
| 24 | `ffuf -w words.txt -u https://target/FUZZ -x http://127.0.0.1:8080` | Inspect via Burp |
| 25 | `ffuf -w words.txt -u https://target/FUZZ -replay-proxy http://127.0.0.1:8080` | Replay only matches |

**Research questions**
- Does the session expire mid-run, and how would that show in the results?
- Which rate keeps error rates near zero?
- Does the proxy view show the same requests ffuf reports?

**Sources:** ffuf README; Burp history; server logs (if provided).
**Output:** a run log with rate, thread count, and any 401/429 responses.

## Scenario 6 — evidence and reproducibility

A fuzzing finding needs the request, the wordlist line, and the response. Save
all three.

| # | Command | How to use |
|---|---------|------------|
| 26 | `ffuf -w words.txt -u https://target/FUZZ -ac -o evidence/run.json -of json -od evidence/bodies` | JSON + bodies |
| 27 | `ffuf -w words.txt -u https://target/FUZZ -ac -of html -o evidence/report.html` | Human-readable report |
| 28 | `ffuf -w words.txt -u https://target/FUZZ -json > evidence/run.ndjson` | Newline JSON |
| 29 | `ffuf -w words.txt -u https://target/FUZZ -maxtime 600 -or -o evidence/run.json` | Bounded, no empty file |
| 30 | `ffuf -w words.txt -u https://target/FUZZ -config ~/ffuf-demo/ffuf.conf` | Re-run from config |

**Research questions**
- Does the saved output contain the exact wordlist line that matched?
- Can a second tester reproduce the result from the JSON?
- Is the HTML report sufficient for the client, or is raw JSON needed?

**Sources:** ffuf output formats; report requirements; second-run comparison.
**Output:** an evidence bundle with JSON, matched bodies, and the command.

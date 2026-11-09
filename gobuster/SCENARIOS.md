# gobuster — command scenarios for deep research

Installed build: gobuster 3.6. Flags come from [`help.txt`](help.txt) and the
mode snapshots [`help-dir.txt`](help-dir.txt), [`help-dns.txt`](help-dns.txt),
[`help-vhost.txt`](help-vhost.txt), [`help-fuzz.txt`](help-fuzz.txt).

> **Intrusive:** enumeration sends many requests. Use only on authorized,
> in-scope systems, and respect rate limits.

## How to invoke

```text
@deep-research When does gobuster's status-blacklist model produce reliable
content-discovery results compared with ffuf's calibration model, and how
should a tester choose between dir, vhost, and fuzz modes for a given target?
Cite the gobuster README, HTTP response semantics, and field reports; return a
mode-selection decision table with a validation step.
```

Prepare the demo directory and wordlists:

```bash
mkdir -p ~/gobuster-demo/evidence
printf 'admin\nlogin\napi\nbackup\n' > ~/gobuster-demo/words.txt
printf 'www\napi\nadmin\n' > ~/gobuster-demo/subs.txt
printf 'app\napi\ninternal\n' > ~/gobuster-demo/vhosts.txt
```

## Scenario 1 — baseline and mode selection

Different questions need different modes. Directory discovery asks "what paths
exist?", vhost discovery asks "what names does this server answer?", DNS asks
"what names resolve?".

| # | Command | How to use |
|---|---------|------------|
| 1 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt` | Path discovery |
| 2 | `gobuster dns -d example.com -w ~/gobuster-demo/subs.txt` | Subdomain resolution |
| 3 | `gobuster vhost -u https://10.10.10.5 -w ~/gobuster-demo/vhosts.txt --append-domain` | Vhost discovery |
| 4 | `gobuster fuzz -u 'https://target/?id=FUZZ' -w ~/gobuster-demo/words.txt` | Generic fuzzing |
| 5 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -q -n` | Script-friendly output |

**Research questions**
- Which mode answers the engagement question with the fewest requests?
- Does a name resolve in DNS but not answer as a vhost, or vice versa?
- Which mode produces the least noise in the target's logs?

**Sources:** gobuster README; DNS records; web-server config (if available).
**Output:** a mode-selection note with the evidence for each choice.

## Scenario 2 — directory discovery and filtering

gobuster has no auto-calibration. Establish the soft-404 shape first, then
filter by size or status.

| # | Command | How to use |
|---|---------|------------|
| 6 | `curl -sS -o /dev/null -w '%{http_code} %{size_download}\n' https://target/nonexistent-12345` | Baseline |
| 7 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -t 20` | Default blacklist 404 |
| 8 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt --exclude-length 1234` | Size filter |
| 9 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -s 200,301,302,403` | Positive codes |
| 10 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -x php,html,bak -e` | Extensions |
| 11 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -d` | Backup discovery |

**Research questions**
- Does the target return a uniform soft-404?
- Which filter (size or status) is stable across paths?
- Do backup extensions reveal source or configuration files?

**Sources:** gobuster README; baseline responses; response bodies via `curl`.
**Output:** a filtered result list with the baseline value recorded.

## Scenario 3 — subdomain enumeration

DNS mode is only as good as the wordlist and resolver. A wildcard record makes
every name look valid; gobuster stops by default.

| # | Command | How to use |
|---|---------|------------|
| 12 | `gobuster dns -d example.com -w ~/gobuster-demo/subs.txt -t 20` | Basic enumeration |
| 13 | `gobuster dns -d example.com -w ~/gobuster-demo/subs.txt -c` | Show CNAMEs |
| 14 | `gobuster dns -d example.com -w ~/gobuster-demo/subs.txt -i` | Show IPs |
| 15 | `gobuster dns -d example.com -w ~/gobuster-demo/subs.txt -r 1.1.1.1` | Custom resolver |
| 16 | `gobuster dns -d example.com -w ~/gobuster-demo/subs.txt --wildcard` | Continue past wildcard |

**Research questions**
- Is there a wildcard record, and what does it resolve to?
- Which names are CNAMEs to third-party services?
- Does a custom resolver return different results?

**Sources:** gobuster README; `dig` comparison; DNS provider docs.
**Output:** a resolved-name table with CNAME/IP notes.

## Scenario 4 — vhost discovery

Vhost fuzzing keeps the URL fixed and changes the `Host` header. Use the IP in
`-u`; `--append-domain` builds the full name from each word.

| # | Command | How to use |
|---|---------|------------|
| 17 | `gobuster vhost -u https://10.10.10.5 -w ~/gobuster-demo/vhosts.txt --append-domain -k` | Full names |
| 18 | `gobuster vhost -u https://10.10.10.5 -w ~/gobuster-demo/vhosts.txt --domain example.com --append-domain` | Explicit domain |
| 19 | `gobuster vhost -u https://10.10.10.5 -w ~/gobuster-demo/vhosts.txt --exclude-length 4242` | Filter default vhost |
| 20 | `gobuster vhost -u https://10.10.10.5 -w ~/gobuster-demo/vhosts.txt -r -k` | Follow redirects |
| 21 | `curl -k -H 'Host: app.example.com' https://10.10.10.5/ -I` | Confirm one vhost manually |

**Research questions**
- Does the default vhost have a distinct size or status?
- Are discovered vhosts also in DNS?
- Does TLS SNI change the response, or only the Host header?

**Sources:** gobuster README; `curl -H 'Host: ...'`; certificate SANs.
**Output:** a vhost list with a confirming request for each.

## Scenario 5 — authenticated, throttled, and resumed runs

Long runs need session material, a rate limit, and a resume point. Record all
three so the run is reproducible.

| # | Command | How to use |
|---|---------|------------|
| 22 | `gobuster dir -u https://target/admin -w ~/gobuster-demo/words.txt -c 'SESSION=abc123'` | Cookie auth |
| 23 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -U user -P pass` | Basic auth |
| 24 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -H 'Authorization: Bearer TOKEN'` | Bearer auth |
| 25 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -t 5 --delay 500ms` | Throttled |
| 26 | `gobuster dir -u https://target -w big.txt --wordlist-offset 50000 -o evidence/resume.txt` | Resume |
| 27 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt --proxy http://127.0.0.1:8080` | Inspect via Burp |

**Research questions**
- Does the session expire during the run?
- Which thread/delay combination keeps error rates near zero?
- Does the resume offset skip or repeat entries?

**Sources:** gobuster README; Burp history; server logs.
**Output:** a run log with auth method, rate, and resume offset.

## Scenario 6 — evidence and reporting

gobuster output is plain text. Pair it with the baseline and a confirming
request so the finding is defensible.

| # | Command | How to use |
|---|---------|------------|
| 28 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -e -o evidence/dirs.txt` | Full-URL output |
| 29 | `gobuster dns -d example.com -w ~/gobuster-demo/subs.txt -c -i -o evidence/dns.txt` | DNS evidence |
| 30 | `gobuster vhost -u https://10.10.10.5 -w ~/gobuster-demo/vhosts.txt --append-domain -k -o evidence/vhosts.txt` | Vhost evidence |
| 31 | `gobuster dir -u https://target -w ~/gobuster-demo/words.txt -q -n > evidence/plain.txt` | Clean list |
| 32 | `wc -l evidence/*.txt` | Result counts |

**Research questions**
- Does the saved output contain enough context to reproduce each hit?
- Which hits need a manual confirming request?
- Are there duplicates across modes that should be merged?

**Sources:** gobuster output; confirming requests; report template.
**Output:** an evidence directory with per-mode result files and counts.

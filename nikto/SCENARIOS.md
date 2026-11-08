# nikto — command scenarios for deep research

Every nikto command form, grouped by scenario. Flags from the installed build
(`help.txt`).

## How to invoke

```text
@deep-research For nikto 2.6.1, which -Tuning classes are safe against a
production app, and how effective are the -evasion encodings against modern
WAFs? Cite Nikto docs, plugin source, and independent comparisons with nuclei;
return exact flags and a risk note per class.
```

---

## Scenario 1 — baseline scan

| # | Command | How to use |
|---|---------|------------|
| 1 | `nikto -h example.com` | Default scan on port 80 |
| 2 | `nikto -h https://example.com` | HTTPS target |
| 3 | `nikto -h example.com -port 8443 -ssl` | Non-standard TLS port |
| 4 | `nikto -h example.com -nossl` | Force plain HTTP |
| 5 | `nikto -h 10.0.0.5 -ipv4` | IPv4 only |
| 6 | `nikto -h example.com -ipv6` | IPv6 only |
| 7 | `nikto -h example.com -vhost app.example.com` | Host-header virtual host |
| 8 | `nikto -h example.com -root /app` | Prepend a path to every request |
| 9 | `nikto -h example.com -url https://example.com/app` | URL form of `-h` |

**Research questions**
- Does `-vhost` change TLS SNI as well as the Host header?
- How does `-root` interact with plugin paths that already start with `/`?

**Sources**: Nikto docs, plugin source, issues on vhost/SNI.

**Output**: a baseline report for the exact URL and port in scope.

---

## Scenario 2 — tuning the test classes

| # | Command | How to use |
|---|---------|------------|
| 10 | `nikto -h example.com -Tuning 1` | Interesting files / seen in logs |
| 11 | `nikto -h example.com -Tuning 2` | Misconfiguration / default files |
| 12 | `nikto -h example.com -Tuning 3` | Information disclosure |
| 13 | `nikto -h example.com -Tuning 4` | Injection (XSS/script/HTML) |
| 14 | `nikto -h example.com -Tuning 5` | Remote file retrieval inside web root |
| 15 | `nikto -h example.com -Tuning 6` | Denial of service |
| 16 | `nikto -h example.com -Tuning 7` | Remote file retrieval server-wide |
| 17 | `nikto -h example.com -Tuning 8` | Command execution / remote shell |
| 18 | `nikto -h example.com -Tuning 9` | SQL injection |
| 19 | `nikto -h example.com -Tuning 0` | File upload |
| 20 | `nikto -h example.com -Tuning a` | Authentication bypass |
| 21 | `nikto -h example.com -Tuning b` | Software identification |
| 22 | `nikto -h example.com -Tuning c` | Remote source inclusion |
| 23 | `nikto -h example.com -Tuning d` | WebService |
| 24 | `nikto -h example.com -Tuning e` | Administrative console |
| 25 | `nikto -h example.com -Tuning x6` | Everything except DoS |
| 26 | `nikto -h example.com -Tuning 23b` | Combine classes |
| 27 | `nikto -h example.com -Plugins 'apacheversion;cgi'` | Run only named plugins |
| 28 | `nikto -h example.com -list-plugins` | List all plugins |

**Research questions**
- Which tuning classes submit destructive or high-load requests?
- Do plugin names match the tuning classes one-to-one?

**Sources**: Nikto plugin docs, `plugins/` source, program rules on DoS.

**Output**: a tuning set justified by the engagement's allowed test types.

---

## Scenario 3 — IDS/WAF evasion

| # | Command | How to use |
|---|---------|------------|
| 29 | `nikto -h example.com -evasion 1` | Random non-UTF8 URI encoding |
| 30 | `nikto -h example.com -evasion 2` | Directory self-reference `/./` |
| 31 | `nikto -h example.com -evasion 3` | Premature URL ending |
| 32 | `nikto -h example.com -evasion 4` | Prepend a long random string |
| 33 | `nikto -h example.com -evasion 5` | Fake parameter |
| 34 | `nikto -h example.com -evasion 6` | TAB as request spacer |
| 35 | `nikto -h example.com -evasion 7` | Random URL case |
| 36 | `nikto -h example.com -evasion 8` | Windows `\` separator |
| 37 | `nikto -h example.com -evasion A` | CR as request spacer |
| 38 | `nikto -h example.com -evasion B` | 0x0b as request spacer |
| 39 | `nikto -h example.com -evasion R` | Random UA per request |
| 40 | `nikto -h example.com -evasion 1234567890abc` | Combine techniques |

**Research questions**
- Which encodings are normalized by current WAFs before matching?
- Does combining evasions increase false negatives?

**Sources**: Nikto evasion docs, WAF normalization writeups, issues.

**Output**: the minimum evasion needed to complete a scan without bypassing
scope or masking real findings.

---

## Scenario 4 — output and evidence

| # | Command | How to use |
|---|---------|------------|
| 41 | `nikto -h example.com -o report.txt` | Plain text |
| 42 | `nikto -h example.com -o report.html -Format htm` | HTML report |
| 43 | `nikto -h example.com -o report.json -Format json` | JSON for tooling |
| 44 | `nikto -h example.com -o report.csv -Format csv` | CSV |
| 45 | `nikto -h example.com -o report.xml -Format xml` | XML |
| 46 | `nikto -h example.com -o report.sql -Format sql` | Generic SQL |
| 47 | `nikto -h example.com -o report.sqld -Format sqld` | Direct DB insert (MySQL/PostgreSQL) |
| 48 | `nikto -h example.com -o .` | Auto-named output |
| 49 | `nikto -h example.com -Save /tmp/responses` | Save positive responses |
| 50 | `nikto -h example.com -Display 1` | Show redirects |
| 51 | `nikto -h example.com -Display 2` | Show cookies |
| 52 | `nikto -h example.com -Display 3` | Show all 200/OK responses |
| 53 | `nikto -h example.com -Display 4` | Show URLs requiring auth |
| 54 | `nikto -h example.com -Display E` | All HTTP errors |
| 55 | `nikto -h example.com -Display P` | Progress to stdout |
| 56 | `nikto -h example.com -Display S` | Scrub IPs/hostnames from output |
| 57 | `nikto -h example.com -Display V` | Verbose |
| 58 | `nikto -h example.com -Display D` | Debug |
| 59 | `nikto -h example.com -Display 1234EPV` | Combine display flags |

**Research questions**
- Which report format preserves evidence for a client deliverable?
- Does `-Display S` scrub only stdout or also `-o` files?

**Sources**: Nikto output docs, report-format examples, issues.

**Output**: a report format matched to the consumer, with a scrubbed variant
for sharing.

---

## Scenario 5 — authenticated and custom requests

| # | Command | How to use |
|---|---------|------------|
| 60 | `nikto -h example.com -id admin:password` | Basic auth |
| 61 | `nikto -h example.com -id admin:password:realm` | Basic auth with realm |
| 62 | `nikto -h example.com -Add-header 'Authorization: Bearer ...'` | Arbitrary header |
| 63 | `nikto -h example.com -Add-header 'X-Env: staging' -Add-header 'Accept: application/json'` | Multiple headers |
| 64 | `nikto -h example.com -key client.key -RSAcert client.crt` | Client certificate auth |
| 65 | `nikto -h example.com -useragent 'Mozilla/5.0 ...'` | Force a User-Agent |
| 66 | `nikto -h example.com -useproxy http://127.0.0.1:8080` | Route through Burp |

**Research questions**
- Are auth headers replayed on every plugin request?
- Does `-useproxy` require Burp to be in scope/configured to log?

**Sources**: Nikto auth docs, source request builder, issues.

**Output**: an authenticated scan that still respects rate and scope.

---

## Scenario 6 — runtime and safety controls

| # | Command | How to use |
|---|---------|------------|
| 67 | `nikto -h example.com -Pause 2` | Pause between tests |
| 68 | `nikto -h example.com -timeout 20` | Per-request timeout |
| 69 | `nikto -h example.com -maxtime 1h` | Total time cap |
| 70 | `nikto -h example.com -maxtime 3600s` | Same cap in seconds |
| 71 | `nikto -h example.com -followredirects` | Follow 3xx |
| 72 | `nikto -h example.com -no404` | Disable 404-page guessing |
| 73 | `nikto -h example.com -404code '302,301'` | Treat codes as negative |
| 74 | `nikto -h example.com -404string 'not found'` | Treat body string as negative |
| 75 | `nikto -h example.com -noslash` | Strip trailing slash |
| 76 | `nikto -h example.com -nolookup` | Disable DNS lookups |
| 77 | `nikto -h example.com -nocookies` | Ignore response cookies |
| 78 | `nikto -h example.com -nointeractive` | Disable interactive prompts |
| 79 | `nikto -h example.com -nocheck` | Skip update check |
| 80 | `nikto -h example.com -nosslkeepalive` | Disable TLS keep-alive |
| 81 | `nikto -h example.com -ask+ no` | Never submit updates |
| 82 | `nikto -h example.com -ask+ auto` | Auto-submit updates |
| 83 | `nikto -h example.com -Option DEFAULT_USER_AGENT=Mozilla/5.0` | Override a config option |
| 84 | `nikto -h example.com -config custom.conf` | Custom config file |
| 85 | `nikto -h example.com -Platform win` | Target platform hint (nix/win/all) |
| 86 | `nikto -h example.com -mutate 1` | All files × all root dirs |
| 87 | `nikto -h example.com -mutate 2` | Guess password-file names |
| 88 | `nikto -h example.com -mutate 3` | Apache `/~user` enumeration |
| 89 | `nikto -h example.com -mutate 4` | cgiwrap `/~user` enumeration |
| 90 | `nikto -h example.com -mutate-options 'dict=/path/users.txt'` | Mutate inputs |
| 91 | `nikto -h example.com -Cgidirs '/cgi/ /cgi-a/'` | Scan specific CGI dirs |
| 92 | `nikto -h example.com -Cgidirs none` | Skip CGI dirs |
| 93 | `nikto -h example.com -Cgidirs all` | All CGI dirs |
| 94 | `nikto -h example.com -Userdbs all` | Only user databases |
| 95 | `nikto -h example.com -Userdbs tests` | Only user test DBs |
| 96 | `nikto -check6` | Verify IPv6 connectivity |
| 97 | `nikto -dbcheck` | Check DB/config syntax |
| 98 | `nikto -Version` | Plugin and database versions |
| 99 | `nikto -Help` | This help |

**Research questions**
- Which `-mutate` modes generate the most requests and the most noise?
- `-ask+` update submission: what data leaves the host?

**Sources**: Nikto config docs, mutate docs, issues on update telemetry.

**Output**: a runtime profile with hard limits and update behavior documented.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Target | `-h`/`-host`/`-url`, `-port`, `-ssl`, `-nossl`, `-ipv4`, `-ipv6`, `-vhost`, `-root` | 1 |
| Tuning | `-Tuning`, `-Plugins`, `-list-plugins` | 2 |
| Evasion | `-evasion` | 3 |
| Output | `-output`, `-Format`, `-Save`, `-Display` | 4 |
| Auth | `-id`, `-Add-header`, `-key`, `-RSAcert`, `-useragent`, `-useproxy` | 5 |
| Runtime | `-Pause`, `-timeout`, `-maxtime`, `-followredirects`, `-no404`, `-404code`, `-404string`, `-noslash`, `-nolookup`, `-nocookies`, `-nointeractive`, `-nocheck`, `-nosslkeepalive`, `-ask+`, `-Option`, `-config`, `-Platform`, `-mutate`, `-mutate-options`, `-Cgidirs`, `-Userdbs`, `-check6`, `-dbcheck`, `-Version`, `-Help` | 6 |

Full per-flag text: [`help.txt`](help.txt).

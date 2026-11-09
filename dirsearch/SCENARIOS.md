# dirsearch — command scenarios for deep research

Installed build: Debian-packaged dirsearch. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For the packaged dirsearch on Debian 13/Python 3.13, which
filtering strategy most reliably suppresses wildcard responses, which wordlist
and extension set give the best coverage-to-noise ratio for modern apps, and
how do recursion, --crawl, and rate limits interact at scale? Cite dirsearch
docs, SecLists guidance, and field reports; return a tuned command profile.
```

---

## Scenario 1 — baseline scanning

| # | Command | How to use |
|---|---------|------------|
| 1 | `dirsearch -u https://target/` | Default scan |
| 2 | `dirsearch -u https://target/ -e php,html,js,txt` | Common extensions |
| 3 | `dirsearch -u https://target/ -w /path/list.txt` | Custom wordlist |
| 4 | `dirsearch -u https://target/ --exclude-status 404` | Drop 404s |
| 5 | `dirsearch -u https://target/ --no-color -q` | Quiet logs |

**Research questions**
- Which default wordlist does the Debian package ship, and how large is it?
- Do extensions apply to every entry or only `%EXT%` placeholders?

**Sources:** package files, `-w`, help text.
**Output:** a baseline path list.

---

## Scenario 2 — filtering wildcards

| # | Command | How to use |
|---|---------|------------|
| 6 | `dirsearch -u https://target/ --exclude-sizes 0B` | Empty responses |
| 7 | `dirsearch -u https://target/ --exclude-sizes 0B,1234B` | Known wildcard size |
| 8 | `dirsearch -u https://target/ --exclude-text "Not Found"` | Text filter |
| 9 | `dirsearch -u https://target/ --exclude-regex "error"` | Regex filter |
| 10 | `dirsearch -u https://target/ --exclude-response 404.html` | Similar-to-404 |

**Research questions**
- Which filter is most robust against dynamic error pages?
- Does `--exclude-response` compare status, body, or both?

**Sources:** docs, sample responses, output comparison.
**Output:** a low-noise finding set.

---

## Scenario 3 — recursion and crawling

| # | Command | How to use |
|---|---------|------------|
| 11 | `dirsearch -u https://target/ -r -R 2` | Bounded recursion |
| 12 | `dirsearch -u https://target/ --deep-recursive` | Recurse every depth |
| 13 | `dirsearch -u https://target/ --force-recursive` | Recurse all paths |
| 14 | `dirsearch -u https://target/ --crawl` | Crawl responses |
| 15 | `dirsearch -u https://target/ --recursion-status 200,301` | Recursion triggers |

**Research questions**
- How does `--deep-recursive` differ from `--force-recursive` in request count?
- Does `--crawl` respect robots and scope?

**Sources:** docs, source, request logs.
**Output:** a mapped directory tree without runaway requests.

---

## Scenario 4 — auth, methods, raw requests

| # | Command | How to use |
|---|---------|------------|
| 16 | `dirsearch -u https://target/ --cookie "session=abc"` | Cookie auth |
| 17 | `dirsearch -u https://target/ --auth user:pass --auth-type basic` | Basic auth |
| 18 | `dirsearch -u https://target/ --auth <jwt> --auth-type bearer` | Bearer token |
| 19 | `dirsearch --raw request.txt --scheme https -e php` | Raw request |
| 20 | `dirsearch -u https://target/ -m POST -d "a=1" -e php` | POST scan |

**Research questions**
- Which auth types are supported by the packaged build?
- How are Host and Content-Length handled in raw requests?

**Sources:** help, lab proxy, raw request tests.
**Output:** authenticated coverage of protected paths.

---

## Scenario 5 — scale and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `dirsearch -l targets.txt --max-rate 20 --threads 10` | Throttled sweep |
| 22 | `dirsearch -l targets.txt --format json -o scan.json` | JSON evidence |
| 23 | `dirsearch -u https://target/ --format sqlite -o scan.sqlite` | SQLite evidence |
| 24 | `dirsearch -u https://target/ --replay-proxy http://127.0.0.1:8080` | Replay hits |
| 25 | `sha256sum scan.json scan.sqlite` | Hash evidence |

**Research questions**
- Which report format best preserves status, size, and redirect chains?
- Does `--max-rate` apply globally or per target?

**Sources:** docs, output, proxy logs.
**Output:** a scalable scan with hashed evidence.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Dictionary | `-w -e -f -O --prefixes --suffixes -U -L -C` | 1 |
| Scan | `-r --deep-recursive --force-recursive -R -i -x --exclude-* --min/max-response-size` | 2, 3 |
| Request | `-m -d -H -F --auth --cookie --proxy --raw` | 4 |
| Output | `-o --format --log --replay-proxy` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — baseline | 5 |
| 2 — filtering | 5 |
| 3 — recursion | 5 |
| 4 — auth/raw | 5 |
| 5 — scale/evidence | 5 |

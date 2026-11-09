# hydra — command scenarios for deep research

Installed build: Hydra v9.5. Flags and the supported-service list come from
[`help.txt`](help.txt) and [`help-full.txt`](help-full.txt). The build does not
include `smb2`, `afp`, `ncp`, `oracle`, or `sapr3`.

> **High risk:** online guessing can lock accounts and trigger alerts. Check
> the lockout policy and attempt budget before the first request.

## How to invoke

```text
@deep-research How should a tester choose between password spraying and
brute-force with hydra for a given service and lockout policy, and which
parallelism, wait, and stop conditions keep the test inside a fixed attempt
budget? Cite the hydra README, service lockout documentation, and field
reports; return a decision table with an attempt-budget calculation.
```

Prepare the demo directory:

```bash
mkdir -p ~/hydra-demo/evidence
printf 'deploy\nsvc-backup\n' > ~/hydra-demo/users.txt
printf 'Spring2026!\nPassword123\n' > ~/hydra-demo/passes.txt
printf '10.10.10.5\n10.10.10.6\n' > ~/hydra-demo/targets.txt
```

Use lab accounts only; do not paste real credentials into a shared history.

## Scenario 1 — establish the policy and the attempt budget

The first step is not a hydra command; it is the lockout policy. If the policy
allows five failures, a 100-password list is out of scope.

| # | Command | How to use |
|---|---------|------------|
| 1 | `hydra -U ssh` | Show module usage and options |
| 2 | `hydra -U http-post-form` | Form module placeholders |
| 3 | `hydra -l deploy -p wrongpass -t 1 -V ssh://10.10.10.5` | One controlled failure |
| 4 | `hydra -l deploy -p wrongpass -t 1 -V rdp://10.10.10.5` | Check RDP behavior |
| 5 | `hydra -l deploy -p wrongpass -t 1 -V 10.10.10.5 http-post-form '/login:user=^USER^&pass=^PASS^:F=Invalid'` | Form failure string |

**Research questions**
- What is the exact failure response for each service?
- How many failures are allowed before lockout?
- Does the service rate-limit or tarpit after repeated failures?

**Sources:** hydra module help; identity policy docs; controlled lab tests.
**Output:** a policy note with the maximum safe attempt count.

## Scenario 2 — spraying versus brute force

Spraying tries one password across many users; brute force tries many passwords
against one user. Spraying is usually safer and finds weak seasonal passwords;
brute force is noisier and lockout-prone.

| # | Command | How to use |
|---|---------|------------|
| 6 | `hydra -L users.txt -p 'Spring2026!' -t 2 -W 5 ssh://10.10.10.5` | Spray one password |
| 7 | `hydra -L users.txt -p 'Spring2026!' -t 1 -c 10 ssh://10.10.10.5` | Slow spray |
| 8 | `hydra -l deploy -P passes.txt -t 2 -f ssh://10.10.10.5` | Brute one user |
| 9 | `hydra -L users.txt -P passes.txt -t 1 -f ssh://10.10.10.5` | Cross-product (risky) |
| 10 | `hydra -L users.txt -e nsr -t 1 ssh://10.10.10.5` | Null/same/reverse checks |

**Research questions**
- Which approach finds a valid pair within the attempt budget?
- Does `-u` change the order enough to matter for lockout?
- Are null or same-as-login passwords still present?

**Sources:** hydra README; password-policy docs; attempt logs.
**Output:** an attempt-count table and the chosen strategy.

## Scenario 3 — HTTP forms and web auth

Web forms need the exact field names, the exact failure string, and any hidden
token. A dynamic page breaks naive matching.

| # | Command | How to use |
|---|---------|------------|
| 11 | `curl -sS https://target/login \| grep -i 'form\|input'` | Inspect the form |
| 12 | `hydra -l admin -P passes.txt 10.10.10.5 http-post-form '/login:username=^USER^&password=^PASS^:F=Invalid credentials'` | POST form |
| 13 | `hydra -L users.txt -P passes.txt 10.10.10.5 http-get /admin/` | Basic auth |
| 14 | `hydra -l admin -P passes.txt -s 8443 -S 10.10.10.5 http-post-form '/login:user=^USER^&pass=^PASS^:F=error'` | HTTPS on a custom port |
| 15 | `hydra -l admin -P passes.txt -m '/login:user=^USER^&pass=^PASS^:S=Dashboard' 10.10.10.5 http-post-form` | Success-string matching |

**Research questions**
- Which failure string is stable across attempts?
- Does the form require a CSRF token or cookie?
- Does success return a redirect, a cookie, or different content?

**Sources:** hydra form module docs; Burp request; response diffs.
**Output:** a working form command and a confirmed success signal.

## Scenario 4 — service matrix

Different services need different flags and risk controls. RDP and SSH are
sensitive; databases may not have lockout at all.

| # | Command | How to use |
|---|---------|------------|
| 16 | `hydra -l root -P passes.txt -t 2 mysql://10.10.10.5` | MySQL |
| 17 | `hydra -l sa -P passes.txt -t 2 mssql://10.10.10.5` | MSSQL |
| 18 | `hydra -l postgres -P passes.txt -t 2 postgres://10.10.10.5` | PostgreSQL |
| 19 | `hydra -P communities.txt -t 4 snmp://10.10.10.5` | SNMP community strings |
| 20 | `hydra -l admin -P passes.txt -t 2 vnc://10.10.10.5` | VNC |
| 21 | `hydra -L users.txt -P passes.txt -t 1 rdp://10.10.10.5` | RDP (low parallelism) |
| 22 | `hydra -L users.txt -P passes.txt -M targets.txt -T 8 -f smb` | SMB1-era module |

**Research questions**
- Which services enforce lockout, and which only rate-limit?
- Does the module report success reliably for this service version?
- Is `smb2` required (and therefore netexec) for this target?

**Sources:** hydra module docs; service documentation; netexec comparison.
**Output:** a per-service command sheet with risk notes.

## Scenario 5 — rate control and lockout avoidance

`-t`, `-T`, `-w`, `-W`, and `-c` interact. For sensitive targets, the safest
configuration is one connection at a time with a deliberate delay.

| # | Command | How to use |
|---|---------|------------|
| 23 | `hydra -L users.txt -P passes.txt -t 1 -c 30 ssh://10.10.10.5` | One attempt per 30s |
| 24 | `hydra -L users.txt -P passes.txt -t 2 -W 10 ssh://10.10.10.5` | Per-thread wait |
| 25 | `hydra -L users.txt -p 'Spring2026!' -t 1 -W 60 ssh://10.10.10.5` | Slow spray |
| 26 | `hydra -L users.txt -P passes.txt -M targets.txt -T 2 -f ssh` | Two hosts at a time |
| 27 | `hydra -L users.txt -P passes.txt -K -M targets.txt ssh` | Skip previously failed hosts |

**Research questions**
- Which flag actually controls the interval between attempts?
- Does `-c` force single-threaded operation as documented?
- Does `-K` prevent duplicate work across runs?

**Sources:** hydra help; measured request timestamps; service logs.
**Output:** a timing table and a lockout-safe configuration.

## Scenario 6 — evidence and cleanup

Record the target, service, user list, password list, attempt count, and result.
Remove restore files and any captured credentials at engagement close.

| # | Command | How to use |
|---|---------|------------|
| 28 | `hydra -L users.txt -P passes.txt -o evidence/hydra.txt -b json ssh://10.10.10.5` | JSON evidence |
| 29 | `hydra -L users.txt -P passes.txt -o evidence/hydra.txt ssh://10.10.10.5` | Text evidence |
| 30 | `hydra -R` | Restore an interrupted session |
| 31 | `hydra -I -L users.txt -P passes.txt ssh://10.10.10.5` | Ignore stale restore |
| 32 | `shred -u hydra.restore 2>/dev/null; rm -f evidence/hydra.txt` | Cleanup (after archiving) |

**Research questions**
- Does the evidence record the exact service and port?
- Is the attempt count consistent with the budget?
- Are restore files and plaintext credentials removed at close?

**Sources:** hydra output formats; engagement retention policy.
**Output:** an evidence bundle and a cleanup checklist.

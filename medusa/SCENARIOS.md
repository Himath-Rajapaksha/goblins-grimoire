# Medusa — command scenarios for deep research

Installed build: Medusa v2.3 (Debian package `medusa`). Every flag below
appears in [`help.txt`](help.txt); module parameters are verified with
`medusa -M <module> -q` from the installed module set.

> **Intrusive and noisy:** each attempt is a real authentication request.
> Lockouts, fail2ban, and SIEM alerts are expected. Use only on authorized,
> in-scope services, with a rate and stop condition agreed in advance.

## How to invoke

```text
@deep-research Compare Medusa 2.3 and Hydra 9.5 for an authorized credential
test against SSH, HTTP basic auth, and SMB local accounts. Which tool has the
more reliable module for each protocol, how do their parallelism and retry
controls differ, and what evidence should be captured to distinguish a valid
login from a false positive? Cite the Medusa source and module help, the Hydra
documentation, and field reports; return a per-protocol command with a hard
stop condition.
```

Create the demo inputs, replacing addresses and names with in-scope targets:

```bash
mkdir -p ~/medusa-demo/evidence
printf '10.10.10.5\n10.10.10.6\n' > ~/medusa-demo/hosts.txt
printf 'alice\nbob\nsvc_sql\n' > ~/medusa-demo/users.txt
printf 'Summer2026!\nWinter2026!\nPassword123\n' > ~/medusa-demo/passwords.txt
printf '10.10.10.5:alice:Summer2026!\n' > ~/medusa-demo/combo.txt
```

## Scenario 1 — inventory modules and inputs

Start by confirming which modules this build actually has and what options each
one accepts. Medusa's module set is smaller than Hydra's and differs by
distribution.

| # | Command | How to use |
|---|---------|------------|
| 1 | `medusa -V` | Version |
| 2 | `medusa -d` | Dump all installed modules |
| 3 | `medusa -M ssh -q` | SSH module options |
| 4 | `medusa -M smbnt -q` | SMB module options |
| 5 | `medusa -M web-form -q` | Web-form module options |
| 6 | `medusa -M snmp -q` | SNMP module options |

**Research questions**
- Which modules exist here that Hydra lacks, and vice versa?
- Which modules require a parameter that is not obvious from the top-level help?
- Does the module list change between distributions or package versions?
- Which modules support hash-based authentication instead of a plaintext
  password?

**Sources:** `medusa -d`; `medusa -M <module> -q`; Medusa source tree
(`src/modules/`); Debian package file list.
**Output:** a module matrix with protocol, default port, parameter list, and
hash support.

## Scenario 2 — SSH and remote access

SSH is the most common target and the most likely to trigger lockout. Keep the
lists small, cap concurrency, and stop at the first hit.

| # | Command | How to use |
|---|---------|------------|
| 7 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M ssh -f` | One user, small list |
| 8 | `medusa -h 10.10.10.5 -U ~/medusa-demo/users.txt -p 'Summer2026!' -M ssh -t 4` | Spray one password |
| 9 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M ssh -e ns` | Also try empty and username-as-password |
| 10 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M ssh -n 2222` | Non-default port |
| 11 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M ssh -r 1 -g 5 -R 2` | Slower retry and timeout |
| 12 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M telnet` | Telnet module |
| 13 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M rdp -m DOMAIN:ACME` | RDP with a domain |
| 14 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M vnc` | VNC |

**Research questions**
- Which SSH server settings (MaxAuthTries, MaxStartups, fail2ban) change the
  safe attempt rate?
- Does `-e ns` create additional lockout risk, and when is it worth it?
- How does Medusa distinguish "auth failed" from "connection dropped"?
- Why does the RDP module exclude Windows 2003/XP and earlier?

**Sources:** OpenSSH `sshd_config`; Medusa `ssh.mod` and `rdp.mod` help;
Microsoft RDP protocol documentation.
**Output:** a paced attempt log and one confirmed-or-not credential per target.

## Scenario 3 — HTTP and web forms

HTTP basic/digest/NTLM and HTML form logins need different modules. Read the
module help before guessing parameter names.

| # | Command | How to use |
|---|---------|------------|
| 15 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M http -m DIR:/admin/ -m AUTH:BASIC -n 8080` | Basic auth on a path |
| 16 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M http -m AUTH:NTLM -m DOMAIN:ACME` | NTLM auth |
| 17 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M http -m METHOD:POST -m DIR:/api/login` | POST to a login endpoint |
| 18 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M web-form -m FORM:/login.php -m DENY-SIGNAL:'Invalid login' -m FORM-DATA:'post?username=&password=&submit=Login'` | HTML form |
| 19 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M web-form -m CUSTOM-HEADER:'Cookie: session=abc123' -m FORM:/account/login` | Form with a session cookie |
| 20 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M http -m USER-AGENT:'Mozilla/5.0' -m CUSTOM-HEADER:'X-Forwarded-For: 127.0.0.1'` | Custom headers |

**Research questions**
- How does `DENY-SIGNAL` produce false positives, and how do you verify a hit
  out of band?
- When does `http` `AUTH:automatic` choose basic versus NTLM?
- Which CSRF protections break `web-form`, and what is the safe workaround?
- How do rate limits and WAF rules change the pacing plan?

**Sources:** RFC 7617 (basic), RFC 7616 (digest), MS-NLMP (NTLM); Medusa
`http.mod` and `web-form.mod` help; OWASP authentication testing guidance.
**Output:** a request/response transcript per module and a verified hit list.

## Scenario 4 — SMB, databases, and network services

These modules need protocol-specific parameters and are more likely to lock
domain accounts. Prefer `netexec` for AD-aware spraying; use Medusa when you
need its combo-file or multi-host behavior.

| # | Command | How to use |
|---|---------|------------|
| 21 | `medusa -h 10.10.10.5 -u Administrator -P ~/medusa-demo/passwords.txt -M smbnt -m GROUP:LOCAL -m AUTH:NTLM` | Local SMB account |
| 22 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M smbnt -m GROUP:DOMAIN -m MODE:SMB2` | Domain account over SMB2 |
| 23 | `medusa -h 10.10.10.5 -u sa -P ~/medusa-demo/passwords.txt -M mssql` | MSSQL |
| 24 | `medusa -h 10.10.10.5 -u root -P ~/medusa-demo/passwords.txt -M mysql -m PASS:PASSWORD` | MySQL |
| 25 | `medusa -h 10.10.10.5 -u postgres -P ~/medusa-demo/passwords.txt -M postgres` | PostgreSQL |
| 26 | `medusa -h 10.10.10.5 -u alice -P ~/medusa-demo/passwords.txt -M ftp -m MODE:EXPLICIT -s` | Explicit FTPS |
| 27 | `medusa -h 10.10.10.5 -P ~/medusa-demo/communities.txt -M snmp -m VERSION:2C -m ACCESS:READ` | SNMP read |
| 28 | `medusa -h 10.10.10.5 -P ~/medusa-demo/communities.txt -M snmp -m ACCESS:WRITE` | SNMP write check |

**Research questions**
- What does `GROUP:LOCAL` versus `GROUP:DOMAIN` change on the wire?
- Does the SMB module support pass-the-hash through `-m PASS:HASH`?
- Which database modules log failed logins differently, and where?
- Why is the SNMP write check potentially state-changing?

**Sources:** MS-SMB2; MS-SQL/TDS; MySQL and PostgreSQL authentication
documentation; RFC 3416 (SNMP); Medusa module help.
**Output:** one verified credential per protocol and a state-change warning for
SNMP write.

## Scenario 5 — multi-host runs, pacing, and stop conditions

Medusa's advantage is parallel hosts and logins. That is also its main risk:
one command can touch an entire subnet.

| # | Command | How to use |
|---|---------|------------|
| 29 | `medusa -H ~/medusa-demo/hosts.txt -U ~/medusa-demo/users.txt -P ~/medusa-demo/passwords.txt -M ssh -T 2 -t 4 -f -O ~/medusa-demo/found.log` | Two hosts, four logins each |
| 30 | `medusa -H ~/medusa-demo/hosts.txt -U ~/medusa-demo/users.txt -P ~/medusa-demo/passwords.txt -M ssh -F -b` | Stop the whole audit at the first hit |
| 31 | `medusa -C ~/medusa-demo/combo.txt -M ssh -f` | Combo file |
| 32 | `medusa -h 10.10.10.5 -U ~/medusa-demo/users.txt -P ~/medusa-demo/passwords.txt -M ssh -L -t 4` | One username per thread |
| 33 | `medusa -h 10.10.10.5 -U ~/medusa-demo/users.txt -P ~/medusa-demo/passwords.txt -M ssh -c 1000 -g 5 -R 2 -r 2` | Socket and retry tuning |
| 34 | `medusa -h 10.10.10.5 -U ~/medusa-demo/users.txt -P ~/medusa-demo/passwords.txt -M ssh -v 4 -w 2` | Verbose and debug |
| 35 | `medusa -h 10.10.10.5 -U ~/medusa-demo/users.txt -P ~/medusa-demo/passwords.txt -M ssh -Z ~/medusa-demo/scan.map` | Resume map |
| 36 | `medusa -h 10.10.10.5 -U ~/medusa-demo/users.txt -P ~/medusa-demo/passwords.txt -M ssh -O ~/medusa-demo/audit.log` | Append audit log |

**Research questions**
- How do `-t` and `-T` multiply into actual concurrent connections?
- What is the failure mode when the target's lockout threshold is lower than
  the configured concurrency?
- What does the resume map contain, and can it be shared between runs?
- Which log fields prove a hit, and which only show an attempt?

**Sources:** `man medusa`; Medusa source (`src/medusa.c`); target service
lockout documentation.
**Output:** a concurrency worksheet, a resume map, and an audit log with
timestamps.

## Scenario 6 — verify hits and report

A "found" line is a claim, not proof. Re-authenticate out of band with a
client that does not share Medusa's parsing logic, then record the evidence.

| # | Command | How to use |
|---|---------|------------|
| 37 | `medusa -h 10.10.10.5 -u alice -p 'Summer2026!' -M ssh -f` | Single confirmation run |
| 38 | `ssh alice@10.10.10.5 'id; hostname'` | Out-of-band SSH verification |
| 39 | `smbclient -L //10.10.10.5 -U 'ACME/alice%Summer2026!'` | Out-of-band SMB verification |
| 40 | `curl -sS -u alice:'Summer2026!' -o /dev/null -w '%{http_code}\n' https://10.10.10.5/admin/` | Out-of-band HTTP verification |
| 41 | `shred -u ~/medusa-demo/passwords.txt ~/medusa-demo/combo.txt` | Destroy credential lists |

**Research questions**
- Which client gives the cleanest independent confirmation for each protocol?
- How do you avoid re-locking an account during verification?
- What should be redacted in the report, and what must be retained as evidence?
- When should a hit be reported as "valid but not authorized to use"?

**Sources:** protocol client documentation; engagement rules of engagement;
evidence-handling policy.
**Output:** a verified hit table with timestamp, protocol, account, and
verification command.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-h`, `-H`, `-u`, `-U`, `-p`, `-P`, `-C` | 2–6 |
| `-e n/s/ns` | 2 |
| `-M`, `-m` | 1–4 |
| `-n`, `-s` | 2, 4 |
| `-g`, `-r`, `-R`, `-c` | 2, 5 |
| `-t`, `-T`, `-L` | 2, 5 |
| `-f`, `-F` | 2, 5, 6 |
| `-O`, `-Z` | 5 |
| `-b`, `-q`, `-v`, `-w`, `-d`, `-V` | 1, 5 |

Defaults: connect timeout 3 s, retry sleep 3 s, socket wait 500 µs. Full text:
[`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Module and input inventory | 6 |
| 2 | SSH and remote access | 8 |
| 3 | HTTP and web forms | 6 |
| 4 | SMB, databases, network services | 8 |
| 5 | Multi-host runs and pacing | 8 |
| 6 | Verify hits and report | 5 |
| **Total** | | **41** |

## Legal and safety

Credential testing is only lawful with written authorization that names the
targets, the accounts, the rate, and the stop condition. Never run a host list
you have not verified line by line. Treat every found credential as a secret,
verify it with the least intrusive client available, and delete the input lists
at close.

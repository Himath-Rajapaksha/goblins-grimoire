# ldapdomaindump — command scenarios for deep research

Installed build: python3-ldapdomaindump 0.9.4-2, exposing
`/usr/bin/ldapdomaindump`. Every option below appears in [`help.txt`](help.txt).
The installed host has no Active Directory domain, so bind and dump behaviour
is documented from the tool's argument parser, source, and a failed bind;
output-file names come from the package source. Treat successful-dump examples
as lab plans, not observed results.

> **Scope:** querying a directory is an authenticated read of personal and
> organisational data. Only run these scenarios against a domain you own or are
> explicitly authorised to assess. Anonymous binds and password spraying are
> not authorised by a network scan alone.

> **Credential handling:** prefer an environment variable or an authentication
> file over a command line, because the password appears in shell history and
> process listings. `-p` also accepts an `LM:NTLM` hash. Never write real
> credentials into this repository.

## How to invoke

```text
@deep-research For an authorised AD assessment with ldapdomaindump 0.9.4,
explain how NTLM and SIMPLE binds differ, how pass-the-hash is supplied, how
the tool discovers the base DN and which LDAP attributes it queries, what each
output file contains, how `--minimal`, `--resolve`, `--dns-server`, and
`--delimiter` change the result, and why connection failures surface as Python
tracebacks. Cite the installed help text, the package source, and observed
output. Separate confirmed behaviour from inference and state confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/ldd-demo/evidence
cd ~/ldd-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
ldapdomaindump -h > evidence/help.txt
dpkg -s python3-ldapdomaindump | grep -E 'Version|Depends' > evidence/package.txt
```

## Scenario 1 — version, help, binaries, package provenance

| # | Command | Purpose |
|---|---------|---------|
| 1 | `ldapdomaindump -h` | Full usage and option list |
| 2 | `ldapdomaindump` | Missing `HOSTNAME` |
| 3 | `ldapdomaindump --version` | Not implemented; exits 2 |
| 4 | `ldapdomaindump -h 2>&1 \| head -5` | Usage header |
| 5 | `ldapdomaindump -h 2>&1 \| grep -c '^  -'` | Count option lines |
| 6 | `command -v ldapdomaindump` | Resolve the entry point |
| 7 | `readlink -f "$(command -v ldapdomaindump)"` | Real path |
| 8 | `file "$(command -v ldapdomaindump)"` | Python script |
| 9 | `head -3 "$(command -v ldapdomaindump)"` | Shebang and entry call |
| 10 | `dpkg -S /usr/bin/ldapdomaindump` | Owning package |
| 11 | `dpkg -s python3-ldapdomaindump \| grep -E 'Version\|Depends'` | Package provenance |
| 12 | `pip show ldapdomaindump 2>&1 \| head -6` | Upstream version metadata |
| 13 | `python3 -c 'import ldapdomaindump; print(ldapdomaindump.__file__)'` | Module path |
| 14 | `python3 -c 'import ldap3; print(ldap3.__version__)'` | LDAP library version |
| 15 | `dpkg -L python3-ldapdomaindump \| head -20` | Shipped files |
| 16 | `ldapdomaindump -h > evidence/help.txt` | Save the help text |
| 17 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which package owns the entry point?
- Does `--version` exist?
- Which LDAP library does it use?
- What exit code does a missing argument produce?

**Sources:** `help.txt`; `dpkg -S`; package source; observed output.
**Output:** a provenance record and an option inventory.

## Scenario 2 — authentication modes

| # | Command | Purpose |
|---|---------|---------|
| 18 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump dc01.corp.example` | NTLM password bind |
| 19 | `ldapdomaindump -u 'CORP\alice' -p 'LMHASH:NTHASH' -o dump dc01.corp.example` | Pass-the-hash |
| 20 | `ldapdomaindump -u 'alice@corp.example' -p 'Password123' -at SIMPLE -o dump dc01.corp.example` | SIMPLE bind |
| 21 | `ldapdomaindump -u 'CORP\alice' -o dump dc01.corp.example` | Prompt for the password |
| 22 | `ldapdomaindump -o dump dc01.corp.example` | Anonymous bind |
| 23 | `ldapdomaindump -u '' -p '' -o dump dc01.corp.example` | Empty credentials |
| 24 | `ldapdomaindump -u 'CORP\alice' -p 'wrong' -o dump dc01.corp.example` | Bad password |
| 25 | `ldapdomaindump -u 'CORP\nosuch' -p 'Password123' -o dump dc01.corp.example` | Unknown user |
| 26 | `ldapdomaindump -at BOGUS -u 'CORP\alice' -p x dc01.corp.example` | Invalid auth type |
| 27 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump ldaps://dc01.corp.example:636` | LDAPS endpoint |
| 28 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump ldap://dc01.corp.example:389` | Explicit LDAP endpoint |
| 29 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump 10.0.0.10` | Bind by IP |
| 30 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump dc01.corp.example:636` | Port without scheme |
| 31 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump dc01.corp.example > evidence/bind.log 2>&1` | Save the bind log |
| 32 | `grep -i 'bind\|error' evidence/bind.log` | Extract bind status |
| 33 | `env LDD_PASS='Password123' sh -c 'ldapdomaindump -u "CORP\\alice" -p "$LDD_PASS" -o dump dc01.corp.example'` | Keep the secret out of history |

**Research questions**
- Which modes send a cleartext password?
- Does `-p` accept an NT hash without the LM half?
- Does `SIMPLE` accept a UPN while `NTLM` does not?
- What happens with an empty username and password?

**Sources:** `help.txt`; package source; observed output.
**Output:** an authentication-mode matrix and a credential-handling checklist.

## Scenario 3 — output formats and directory layout

| # | Command | Purpose |
|---|---------|---------|
| 34 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump dc01.corp.example` | All formats |
| 35 | `ls -l dump` | Resulting files |
| 36 | `find dump -type f \| sort` | Full inventory |
| 37 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump-json --no-html --no-grep dc01.corp.example` | JSON only |
| 38 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump-grep --no-html --no-json dc01.corp.example` | Greppable only |
| 39 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump-html --no-json --no-grep dc01.corp.example` | HTML only |
| 40 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump-grouped --grouped-json dc01.corp.example` | Grouped JSON |
| 41 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump-comma --no-html --no-json -d ',' dc01.corp.example` | Comma-delimited grep |
| 42 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump-tab --no-html --no-json -d $'\t' dc01.corp.example` | Tab-delimited grep |
| 43 | `head -3 dump/domain_users.grep` | Greppable header |
| 44 | `head -3 dump-comma/domain_users.grep` | Custom delimiter |
| 45 | `jq 'length' dump/domain_users.json` | User count |
| 46 | `jq 'keys' dump/domain_users.json` | JSON structure |
| 47 | `du -sh dump dump-json dump-grep` | Size comparison |
| 48 | `stat -c '%a %n' dump/domain_users.html` | File mode |
| 49 | `grep -c '<tr>' dump/domain_users.html` | HTML row count |

**Research questions**
- Which files appear in each format combination?
- Does `--grouped-json` add new files or change existing ones?
- What delimiter does the greppable output use by default?
- How large is HTML compared with JSON and grep?

**Sources:** `help.txt`; package source; observed output on a lab domain.
**Output:** an output-file matrix and a size comparison.

## Scenario 4 — users, groups, and computers

| # | Command | Purpose |
|---|---------|---------|
| 50 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump --no-html dc01.corp.example` | Dump with JSON and grep |
| 51 | `awk -F'\t' 'NR>1 {print $1}' dump/domain_users.grep \| sort -u \| head -50` | Usernames |
| 52 | `grep -i 'admin' dump/domain_users.grep \| head -20` | Admin-like users |
| 53 | `grep -i 'service' dump/domain_users.grep \| head -20` | Service accounts |
| 54 | `grep -iE 'domain admins\|enterprise admins\|schema admins' dump/domain_groups.grep` | Privileged groups |
| 55 | `grep -i 'admin' dump/domain_groups.grep \| head -20` | Group names |
| 56 | `grep -iE '2003\|2008\|xp\|windows 7' dump/domain_computers.grep` | Old operating systems |
| 57 | `awk -F'\t' 'NR>1 {print $NF}' dump/domain_computers.grep \| sort \| uniq -c` | OS distribution |
| 58 | `jq -r '.[].attributes.sAMAccountName' dump/domain_users.json \| head` | JSON usernames |
| 59 | `jq -r '.[].attributes.memberOf[]?' dump/domain_users.json \| sort -u \| head` | Group memberships |
| 60 | `jq -r '.[].attributes.distinguishedName' dump/domain_groups.json \| head` | Group DNs |
| 61 | `jq -r '.[].attributes.operatingSystem' dump/domain_computers.json \| sort \| uniq -c` | OS counts |
| 62 | `grep -i 'trust' dump/domain_trusts.grep` | Trust relationships |
| 63 | `cat dump/domain_policy.grep` | Password and lockout policy |
| 64 | `grep -ci 'disabled' dump/domain_users.grep` | Disabled-account hints |
| 65 | `wc -l dump/domain_users.grep dump/domain_groups.grep dump/domain_computers.grep` | Record counts |

**Research questions**
- Which attribute holds the account name, and which holds the DN?
- Does the greppable output include group membership?
- Are disabled accounts marked explicitly?
- How is the domain policy represented?

**Sources:** package source; observed output on a lab domain.
**Output:** a user/group/computer inventory and privilege shortlist.

## Scenario 5 — grouping and host resolution

| # | Command | Purpose |
|---|---------|---------|
| 66 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump -r dc01.corp.example` | Reverse-resolve computers |
| 67 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump -n 10.0.0.10 dc01.corp.example` | Custom DNS resolver |
| 68 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump -r -n 10.0.0.10 dc01.corp.example` | Both |
| 69 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump -m dc01.corp.example` | Minimal attributes |
| 70 | `ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump-full dc01.corp.example` | Full attributes for comparison |
| 71 | `du -sh dump dump-full` | Memory and disk difference |
| 72 | `diff <(jq 'keys' dump/domain_users.json) <(jq 'keys' dump-full/domain_users.json)` | Attribute drift |
| 73 | `grep -c 'by_group' dump/domain_users_by_group.html` | Grouped HTML rows |
| 74 | `jq 'keys' dump/domain_users_by_group.json` | Grouped JSON keys |
| 75 | `jq 'keys' dump/domain_computers_by_os.json` | OS grouping keys |
| 76 | `time ldapdomaindump -u 'CORP\alice' -p 'Password123' -o dump-timed -r dc01.corp.example` | Resolution cost |
| 77 | `getent hosts dc01.corp.example` | System resolver baseline |
| 78 | `dig +short -x 10.0.0.10` | Reverse lookup baseline |
| 79 | `resolvectl status 2>&1 \| head` | Resolver configuration |
| 80 | `cat /etc/resolv.conf` | Resolver file |

**Research questions**
- Does `-r` change the dump or only the HTML display?
- Does `-n` affect the LDAP connection or only name resolution?
- Which attributes does `-m` drop?
- How much slower is a resolved dump on a large domain?

**Sources:** `help.txt`; package source; observed output.
**Output:** a grouping/resolution comparison and a minimal-mode delta.

## Scenario 6 — parsing and triage

| # | Command | Purpose |
|---|---------|---------|
| 81 | `jq -r '.[] \| [.attributes.sAMAccountName, .attributes.userAccountControl] \| @tsv' dump/domain_users.json \| head` | Account flags |
| 82 | `jq -r '.[] \| select(.attributes.servicePrincipalName != null) \| .attributes.sAMAccountName' dump/domain_users.json` | Kerberoastable users |
| 83 | `jq -r '.[] \| select(.attributes.adminCount == true) \| .attributes.sAMAccountName' dump/domain_users.json` | Admin-count accounts |
| 84 | `jq -r '.[] \| select(.attributes.userAccountControl \| tostring \| test("2$")) \| .attributes.sAMAccountName' dump/domain_users.json` | Disabled-account heuristic |
| 85 | `grep -i 'never' dump/domain_users.grep` | Password-never-expires hints |
| 86 | `grep -i 'unconstrained' dump/domain_computers.grep` | Delegation hints |
| 87 | `jq -r '.[].attributes.dNSHostName' dump/domain_computers.json \| sort` | Hostnames |
| 88 | `jq -r '.[].attributes.operatingSystemVersion' dump/domain_computers.json \| sort \| uniq -c` | OS versions |
| 89 | `jq -r '.[].attributes.member' dump/domain_groups.json \| head` | Group members |
| 90 | `grep -i 'krbtgt' dump/domain_users.grep` | krbtgt account |
| 91 | `grep -i 'trust' dump/domain_trusts.grep` | Trust direction and type |
| 92 | `grep -i 'lockout' dump/domain_policy.grep` | Lockout threshold |
| 93 | `grep -i 'minPwdLength\|pwdHistoryLength' dump/domain_policy.grep` | Password policy |
| 94 | `sort -t$'\t' -k2 dump/domain_users.grep \| head` | Sort by second field |
| 95 | `column -t -s$'\t' dump/domain_computers.grep \| head` | Human-readable table |

**Research questions**
- Which JSON attributes map to the greppable columns?
- Which fields identify kerberoastable and AS-REP-roastable accounts?
- Does the dump expose delegation flags directly?
- Which policy values drive a password-spray decision?

**Sources:** package source; observed output on a lab domain.
**Output:** a triage query set and a privilege shortlist.

## Scenario 7 — transport, timeouts, and failure behaviour

| # | Command | Purpose |
|---|---------|---------|
| 96 | `timeout 10 ldapdomaindump 127.0.0.1` | Unreachable local LDAP |
| 97 | `timeout 10 ldapdomaindump nosuch.invalid` | Unresolvable host |
| 98 | `timeout 10 ldapdomaindump ldap://127.0.0.1:389` | Explicit refused endpoint |
| 99 | `timeout 10 ldapdomaindump -u 'CORP\alice' -p x 127.0.0.1` | Auth against no server |
| 100 | `timeout 10 ldapdomaindump -o /root/nope 127.0.0.1` | Unwritable output directory |
| 101 | `timeout 10 ldapdomaindump -o /dev/null 127.0.0.1` | Output path is a device |
| 102 | `timeout 10 ldapdomaindump -at SIMPLE -u x -p y 127.0.0.1` | SIMPLE against no server |
| 103 | `ldapdomaindump 2>&1 \| tail -2` | Missing-host error |
| 104 | `ldapdomaindump --version 2>&1 \| tail -2` | Unknown-option error |
| 105 | `ldapdomaindump -h 2>&1 \| tail -2` | Help exits cleanly |
| 106 | `echo $?` | Exit code of the last command |
| 107 | `ldapdomaindump 127.0.0.1 > evidence/fail.log 2>&1; echo "exit=$?"` | Capture a failure |
| 108 | `grep -c 'Traceback' evidence/fail.log` | Traceback present |
| 109 | `grep -i 'anonymous' evidence/fail.log` | Anonymous warning |
| 110 | `strace -f -e trace=connect ldapdomaindump 127.0.0.1 2>&1 \| tail -5` | Connection attempt |
| 111 | `ss -tnp \| grep ':389'` | No lingering LDAP socket |

**Research questions**
- Which failures are argparse errors and which are Python exceptions?
- Does the tool time out on an unreachable host, or hang?
- Is the output directory created before or after a successful bind?
- What does a failed bind leave behind?

**Sources:** observed output; package source.
**Output:** a failure matrix and a timeout recommendation.

## Scenario 8 — cleanup and evidence handling

| # | Command | Purpose |
|---|---------|---------|
| 112 | `find dump -type f -exec sha256sum {} + > evidence/dump-sha256.txt` | Hash the evidence |
| 113 | `tar -czf evidence/dump.tar.gz dump` | Archive the dump |
| 114 | `gzip -9 dump/domain_users.json` | Compress a large file |
| 115 | `ls -l evidence` | Evidence inventory |
| 116 | `chmod 700 evidence` | Restrict access |
| 117 | `umask 077` | Future files are private |
| 118 | `shred -u dump/domain_users.grep` | Destroy a sensitive copy |
| 119 | `rm -rf dump dump-json dump-grep dump-html dump-grouped dump-comma dump-tab` | Remove lab output |
| 120 | `rm -rf dump-full dump-timed` | Remove comparison output |
| 121 | `test ! -e dump; echo "removed=$?"` | Verify cleanup |
| 122 | `history \| grep -i ldapdomaindump` | Check shell history for secrets |
| 123 | `unset LDD_PASS` | Clear the environment secret |
| 124 | `pgrep -af ldapdomaindump` | No leftover process |
| 125 | `rm -rf ~/ldd-demo` | Remove the demo tree |
| 126 | `test ! -e ~/ldd-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which files contain credentials or personal data?
- Does the tool leave temporary files outside the output directory?
- How should the dump be stored and destroyed?
- What evidence do you need before deleting the raw dump?

**Sources:** observed output; data-handling practice.
**Output:** an evidence-handling checklist and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `-h`, missing host, `--version`, entry point, package provenance | 1 |
| `-u`, `-p`, `-at NTLM`, `-at SIMPLE`, anonymous, hash, LDAPS, endpoint forms | 2 |
| `-o`, `--no-html`, `--no-json`, `--no-grep`, `--grouped-json`, `-d` | 3 |
| users, groups, computers, policy, trusts, JSON and grep parsing | 4 |
| `-r`, `-n`, `-m`, grouped reports, resolution cost | 5 |
| triage queries, kerberoastable, admin count, delegation, policy | 6 |
| unreachable host, refused port, unwritable output, traceback, exit codes | 7 |
| hashing, archiving, permissions, shredding, cleanup | 8 |

Full text: [`help.txt`](help.txt); authoritative description in the upstream
README and the `ldap3` documentation.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, binaries, package provenance | 17 |
| 2 | Authentication modes | 16 |
| 3 | Output formats and directory layout | 16 |
| 4 | Users, groups, and computers | 16 |
| 5 | Grouping and host resolution | 15 |
| 6 | Parsing and triage | 15 |
| 7 | Transport, timeouts, and failure behaviour | 16 |
| 8 | Cleanup and evidence handling | 15 |
| **Total** | | **126** |

## Legal and safety

A domain dump contains usernames, group memberships, computer names, operating
systems, and password policy — personal and organisational data. Only query a
directory you own or are explicitly authorised to assess. Anonymous binds and
credential guessing are not implied by permission to scan a network. Passwords
and hashes on the command line leak into shell history and process listings;
use an environment variable or authentication file, and clear it afterwards.
`SIMPLE` authentication sends the password in cleartext, so use LDAPS or a
trusted network. Store dumps with restrictive permissions, hash them before
deletion if they are evidence, and destroy every copy when the engagement ends.

# rpcclient — command scenarios for deep research

Installed build: Samba 4.22.11-Debian. Flags come from [`help.txt`](help.txt).
Most useful commands need valid credentials; anonymous SAMR is restricted on
modern AD.

> **Sensitive:** domain enumeration exposes account and policy data. Use only
> within written scope, and never run write commands without authorization.

## How to invoke

```text
@deep-research Which rpcclient commands give the most reliable domain
inventory on modern AD with restricted anonymous SAMR, and how should the
results be cross-checked against LDAP and netexec to avoid false conclusions?
Cite the Samba rpcclient man page, MS-SAMR/MS-LSAD, and field reports; return an
enumeration order with a confidence note per command.
```

Prepare the demo directory:

```bash
mkdir -p ~/rpcclient-demo/evidence
printf 'username=ACME/alice\npassword=Password123\ndomain=ACME\n' > ~/rpcclient-demo/auth.txt
chmod 600 ~/rpcclient-demo/auth.txt
```

Use lab credentials only.

## Scenario 1 — server and domain fingerprint

Start with the least invasive queries. `srvinfo` and `lsaquery` work with
minimal rights and establish the domain SID you need for later commands.

| # | Command | How to use |
|---|---------|------------|
| 1 | `rpcclient -U '' -N 10.10.10.5 -c 'srvinfo'` | OS/version |
| 2 | `rpcclient -U '' -N 10.10.10.5 -c 'lsaquery'` | Domain name and SID |
| 3 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'querydominfo'` | Domain info |
| 4 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'dsroledominfo'` | DC role |
| 5 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'getdcname ACME'` | DC name |

**Research questions**
- Does anonymous access return anything useful?
- What is the domain SID, and does it match LDAP?
- Is the host a DC, a member server, or a workstation?

**Sources:** rpcclient man page; `nmap smb-os-discovery`; LDAP rootDSE.
**Output:** a host/domain fingerprint with the SID.

## Scenario 2 — user and group enumeration

`enumdomusers` returns RIDs; `queryuser` and `queryusergroups` expand them.
Cross-check counts with LDAP to catch partial results.

| # | Command | How to use |
|---|---------|------------|
| 6 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'enumdomusers'` | User list |
| 7 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'enumdomgroups'` | Group list |
| 8 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'queryuser 0x1f4'` | User detail |
| 9 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'queryusergroups 0x1f4'` | Group membership |
| 10 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'querygroupmem 0x200'` | Group members |
| 11 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'enumalsgroups domain'` | Alias groups |

**Research questions**
- Does the user count match LDAP?
- Which accounts are privileged, disabled, or service accounts?
- Are there groups with unexpected members?

**Sources:** rpcclient man page; LDAP comparison; `netexec --users`.
**Output:** a user/group table with RIDs and membership.

## Scenario 3 — SID translation and trust mapping

SIDs appear in ACLs, logs, and share output. `lookupsids` turns them into
names; `lookupnames` does the reverse.

| # | Command | How to use |
|---|---------|------------|
| 12 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'lookupsids S-1-5-21-1111111111-2222222222-3333333333-1105'` | SID to name |
| 13 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'lookupnames alice'` | Name to SID |
| 14 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'getsid alice'` | SID for a name |
| 15 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'lsaenumtrustdom'` | Trusts |
| 16 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'lsaenumerateaccountrights S-1-5-21-...-1105'` | Account rights |

**Research questions**
- Do unknown SIDs map to real accounts or to deleted objects?
- Are there trusts that expand the attack surface?
- Which accounts hold sensitive rights?

**Sources:** MS-LSAD; MS-SAMR; ACL exports.
**Output:** a SID/name map and a trust list.

## Scenario 4 — shares, sessions, and policy

SRVSVC exposes shares and sessions; SAMR exposes password policy. Policy data
determines whether spraying is safe.

| # | Command | How to use |
|---|---------|------------|
| 17 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'netshareenumall'` | All shares |
| 18 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'netsharegetinfo Finance'` | Share detail |
| 19 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'netsessionenum'` | Active sessions |
| 20 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'netfileenum'` | Open files |
| 21 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'getdompwinfo'` | Password policy |
| 22 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'getusrdompwinfo 0x1f4'` | Per-user policy |

**Research questions**
- Which shares are hidden from the browse list but present in `netshareenumall`?
- What is the lockout threshold, and how does it constrain spraying?
- Do active sessions reveal privileged users or service accounts?

**Sources:** MS-SRVS; MS-SAMR; identity policy docs.
**Output:** a share/policy table and a spraying-safety note.

## Scenario 5 — authenticated variants

The same queries may work with a password, an NT hash, or Kerberos. Test only
the variants the engagement provides.

| # | Command | How to use |
|---|---------|------------|
| 23 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'enumdomusers'` | Plaintext |
| 24 | `rpcclient -U 'ACME/alice%<nthash>' --pw-nt-hash 10.10.10.5 -c 'enumdomusers'` | Pass-the-hash |
| 25 | `KRB5CCNAME=~/acme.ccache rpcclient --use-kerberos=required -U alice dc.acme.local -c 'lsaquery'` | Kerberos |
| 26 | `rpcclient -A ~/rpcclient-demo/auth.txt 10.10.10.5 -c 'enumdomusers'` | Auth file |
| 27 | `rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'getusername'` | Confirm identity |

**Research questions**
- Which auth method works where others fail?
- Does Kerberos change the visible domain data?
- Is the auth file actually used (no password in `ps`)?

**Sources:** Samba docs; Kerberos ticket cache; netexec comparison.
**Output:** an auth-method matrix with results.

## Scenario 6 — evidence and cross-checks

rpcclient output is line-oriented and easy to archive. Cross-check counts and
names against a second source before reporting.

| # | Command | How to use |
|---|---------|------------|
| 28 | `rpcclient -A ~/rpcclient-demo/auth.txt 10.10.10.5 -c 'enumdomusers' > evidence/users.txt` | User evidence |
| 29 | `rpcclient -A ~/rpcclient-demo/auth.txt 10.10.10.5 -c 'enumdomgroups' > evidence/groups.txt` | Group evidence |
| 30 | `rpcclient -A ~/rpcclient-demo/auth.txt 10.10.10.5 -c 'netshareenumall' > evidence/shares.txt` | Share evidence |
| 31 | `grep -c 'user:\[' evidence/users.txt` | Count users |
| 32 | `ldapsearch -x -H ldap://10.10.10.5 -b 'dc=acme,dc=local' '(objectClass=user)' sAMAccountName \| grep -c sAMAccountName` | LDAP count |
| 33 | `shred -u ~/rpcclient-demo/auth.txt` | Remove credentials |

**Research questions**
- Do rpcclient and LDAP report the same user count?
- Which accounts appear in one source but not the other, and why?
- Is the evidence sufficient for the report without exposing passwords?

**Sources:** rpcclient output; LDAP output; report template.
**Output:** an evidence bundle with counts and a discrepancy note.

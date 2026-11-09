# ldapsearch — command scenarios for deep research

Installed build: OpenLDAP 2.6.10. Flags come from [`help.txt`](help.txt), which
is the usage block printed by an invalid option (`ldapsearch` has no `-h` help
flag).

> **Sensitive:** directory queries expose personal data and account metadata.
> Use only within written scope and page your results.

## How to invoke

```text
@deep-research Which ldapsearch filters and paging controls give a complete,
low-noise inventory of an Active Directory domain, and how should bitwise
userAccountControl filters and recursive memberOf queries be validated against
a second source? Cite RFC 4515, MS-ADTS, the OpenLDAP man page, and field
reports; return a query order with a completeness check.
```

Prepare the demo directory:

```bash
mkdir -p ~/ldapsearch-demo/evidence
printf 'Password123\n' > ~/ldapsearch-demo/bindpw.txt
chmod 600 ~/ldapsearch-demo/bindpw.txt
```

Use lab credentials only.

## Scenario 1 — RootDSE and naming contexts

The RootDSE is the directory's self-description. It tells you the base DN, the
supported controls, and the schema location before you query anything else.

| # | Command | How to use |
|---|---------|------------|
| 1 | `ldapsearch -x -H ldap://10.10.10.5 -s base -b '' '(objectClass=*)'` | Full RootDSE |
| 2 | `ldapsearch -x -H ldap://10.10.10.5 -s base -b '' '(objectClass=*)' namingContexts` | Base DNs |
| 3 | `ldapsearch -x -H ldap://10.10.10.5 -s base -b '' '(objectClass=*)' supportedControl` | Controls |
| 4 | `ldapsearch -x -H ldap://10.10.10.5 -s base -b '' '(objectClass=*)' supportedLDAPVersion` | Protocol versions |
| 5 | `ldapsearch -x -H ldap://10.10.10.5 -s base -b '' '(objectClass=*)' dnsHostName` | AD host name |

**Research questions**
- What is the actual naming context, and does it match the assumed domain?
- Which controls (paging, sorting, DirSync) does the server support?
- Is anonymous RootDSE access allowed?

**Sources:** RFC 4512; MS-ADTS RootDSE; the output itself.
**Output:** a base-DN and capability note.

## Scenario 2 — bind and basic inventory

Test anonymous, simple, and StartTLS binds in that order. Each failure narrows
the authentication policy.

| # | Command | How to use |
|---|---------|------------|
| 6 | `ldapsearch -x -H ldap://10.10.10.5 -b 'dc=acme,dc=local' -s base '(objectClass=*)'` | Anonymous |
| 7 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' -s base '(objectClass=*)'` | Simple bind |
| 8 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -y ~/ldapsearch-demo/bindpw.txt -b 'dc=acme,dc=local' -s base '(objectClass=*)'` | Password file |
| 9 | `ldapsearch -x -H ldap://10.10.10.5 -ZZ -D 'ACME\alice' -W -b 'dc=acme,dc=local' -s base '(objectClass=*)'` | StartTLS required |
| 10 | `ldapsearch -x -H ldaps://dc.acme.local -D 'ACME\alice' -W -b 'dc=acme,dc=local' -s base '(objectClass=*)'` | LDAPS |

**Research questions**
- Which bind methods succeed, and which fail closed?
- Does `-ZZ` fail because StartTLS is unsupported or because policy rejects it?
- Does LDAPS return the same data as LDAP+StartTLS?

**Sources:** RFC 4513; server configuration; `openssl s_client` for TLS.
**Output:** a bind-method matrix.

## Scenario 3 — user inventory with paging

AD limits result sets. Paging is not optional for a complete inventory; a
search without it may look complete and be truncated.

| # | Command | How to use |
|---|---------|------------|
| 11 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(&(objectClass=user)(objectCategory=person))' sAMAccountName` | Unpaged |
| 12 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' -E pr=1000/noprompt '(&(objectClass=user)(objectCategory=person))' sAMAccountName` | Paged |
| 13 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' -E pr=1000/noprompt '(objectClass=group)' cn` | Groups |
| 14 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' -E pr=1000/noprompt '(objectClass=computer)' cn operatingSystem` | Computers |
| 15 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' -LLL -E pr=1000/noprompt '(objectClass=user)' sAMAccountName > evidence/users.ldif` | Clean LDIF |

**Research questions**
- How many users does an unpaged search return versus a paged one?
- Does the count match `rpcclient` and `netexec`?
- Are service accounts included in the person filter?

**Sources:** RFC 2696; AD size-limit docs; cross-tool counts.
**Output:** a user/group/computer count with a completeness note.

## Scenario 4 — account hygiene and roastable accounts

Bitwise filters turn `userAccountControl` into a query. Each OID and flag must
be exact; a wrong flag silently returns the wrong set.

| # | Command | How to use |
|---|---------|------------|
| 16 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(userAccountControl:1.2.840.113556.1.4.803:=2)' sAMAccountName` | Disabled |
| 17 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(userAccountControl:1.2.840.113556.1.4.803:=65536)' sAMAccountName` | Password never expires |
| 18 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(userAccountControl:1.2.840.113556.1.4.803:=4194304)' sAMAccountName` | No pre-auth (AS-REP) |
| 19 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(&(objectClass=user)(servicePrincipalName=*))' sAMAccountName servicePrincipalName` | Kerberoastable |
| 20 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(adminCount=1)' sAMAccountName` | Protected accounts |
| 21 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(msDS-KeyCredentialLink=*)' sAMAccountName` | Shadow credentials |

**Research questions**
- Which flag values are present, and do they match the policy?
- Are roastable accounts real users or service accounts?
- Does `adminCount=1` include stale ACL-protected accounts?

**Sources:** MS-ADTS userAccountControl; MS-ADTS matching rules; AD admin docs.
**Output:** a hygiene table with account counts per flag.

## Scenario 5 — group nesting and delegation

Recursive membership and delegation attributes are where AD misconfigurations
hide. Query them explicitly rather than guessing from group names.

| # | Command | How to use |
|---|---------|------------|
| 22 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(memberOf:1.2.840.113556.1.4.1941:=CN=Domain Admins,CN=Users,DC=acme,DC=local)' sAMAccountName` | Recursive DA members |
| 23 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(userAccountControl:1.2.840.113556.1.4.803:=1048576)' sAMAccountName` | Unconstrained delegation |
| 24 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(msDS-AllowedToDelegateTo=*)' sAMAccountName msDS-AllowedToDelegateTo` | Constrained delegation |
| 25 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(userAccountControl:1.2.840.113556.1.4.803:=524288)' sAMAccountName` | Protocol transition |
| 26 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' '(objectClass=trustedDomain)' cn trustDirection` | Trusts |

**Research questions**
- Does recursive membership reveal nested paths a direct query misses?
- Which accounts or computers hold delegation rights?
- Are trusts bidirectional or selective?

**Sources:** MS-ADTS; BloodHound comparison; delegation documentation.
**Output:** a delegation/trust table with the query that found each.

## Scenario 6 — evidence and reproducibility

Save the filter, base DN, and LDIF. A count without the query is not
reproducible.

| # | Command | How to use |
|---|---------|------------|
| 27 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' -LLL -E pr=1000/noprompt '(objectClass=user)' > evidence/all-users.ldif` | Full user LDIF |
| 28 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' -LLL '(objectClass=*)' > evidence/base.ldif` | Base object |
| 29 | `grep -c '^dn:' evidence/all-users.ldif` | Entry count |
| 30 | `ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W -b 'dc=acme,dc=local' -n '(objectClass=user)'` | Dry run |
| 31 | `shred -u ~/ldapsearch-demo/bindpw.txt` | Remove password file |

**Research questions**
- Does the LDIF contain the exact filter and base DN in its comments?
- Is the entry count stable across two runs?
- Are passwords and sensitive attributes excluded from the report?

**Sources:** LDIF format; report template; second-run comparison.
**Output:** an evidence bundle with LDIF, count, and query text.

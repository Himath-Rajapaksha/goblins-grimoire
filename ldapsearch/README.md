# ldapsearch — LDAP query and directory enumeration

- **Version:** OpenLDAP 2.6.10 (`ldapsearch -VV`)
- **Binary:** `/usr/bin/ldapsearch` (Debian package `ldap-utils`)
- **Full help snapshot:** [`help.txt`](help.txt) (printed by an invalid option;
  `ldapsearch` has no `-h` help flag)
- **Upstream:** https://www.openldap.org/software/man.cgi?query=ldapsearch
- **Docs:** `man ldapsearch`; RFC 4515 (filters), RFC 4511 (protocol)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

ldapsearch sends an LDAP search request to a directory server and prints the
matching entries as LDIF. It is the generic client for OpenLDAP, Active
Directory, 389 Directory Server, and any RFC 4511-compliant directory. You give
it a base DN, a scope, a filter, and a list of attributes; it returns entries,
controls, and referrals.

It is not an AD-specific tool, but it is often the most precise way to query
AD: filters can express bitwise account flags, recursive group membership, and
operational attributes that GUI tools hide.

## Use cases

| Situation | Why ldapsearch |
|-----------|---------------|
| Directory inventory | Users, groups, computers, OUs, trusts |
| AD account hygiene | Disabled, password-never-expires, no-preauth accounts |
| Kerberoast/AS-REP discovery | `servicePrincipalName=*`, `DONT_REQ_PREAUTH` |
| Group nesting | Recursive `memberOf` via matching rule `1.2.840.113556.1.4.1941` |
| RootDSE recon | Naming contexts, supported controls, schema location |
| Policy review | `msDS-PasswordSettings`, `domainPolicy` |
| Schema exploration | Attribute and class definitions |
| Automation | LDIF output parsed by scripts |

## Requirements & install layout

- Debian package `ldap-utils`; binary `/usr/bin/ldapsearch`. OpenLDAP 2.6.10.
- No root needed; network access to TCP 389 (LDAP) or 636 (LDAPS).
- Config is read from `/etc/ldap/ldap.conf` and `~/.ldaprc`; `-H` overrides the
  URI, `-o` sets libldap options.
- `-x` selects simple bind. Without it, ldapsearch attempts SASL, which often
  fails on AD without a configured mechanism.
- `ldapsearch` has **no `-h` help flag**; an invalid option prints the usage
  block captured in [`help.txt`](help.txt). Use `man ldapsearch` for the full
  manual.

## Basic syntax

```bash
ldapsearch [options] [filter [attributes...]]

ldapsearch -x -H ldap://10.10.10.5 -b 'dc=acme,dc=local' '(objectClass=*)'
ldapsearch -x -H ldap://10.10.10.5 -D 'CN=alice,CN=Users,DC=acme,DC=local' \
  -W -b 'dc=acme,dc=local' '(sAMAccountName=alice)' cn mail
ldapsearch -x -H ldaps://dc.acme.local -b 'dc=acme,dc=local' \
  '(objectClass=user)' sAMAccountName
```

The filter is RFC 4515; quote it in the shell. Attributes are a
whitespace-separated list; `*` means all user attributes, `+` all operational
attributes, and `1.1` no attributes.

## Flag reference

### Search

| Flag | Meaning |
|------|---------|
| `-b <basedn>` | Base DN for the search |
| `-s <scope>` | `base`, `one`, `sub`, or `children` |
| `-a <deref>` | Alias dereference: `never` (default), `always`, `search`, `find` |
| `-z <n>` | Size limit in entries (`none` or `max` allowed) |
| `-l <n>` | Time limit in seconds (`none` or `max` allowed) |
| `-A` | Retrieve attribute names only, no values |
| `-S <attr>` | Sort results by attribute |
| `-u` | Include user-friendly entry names |
| `-c` | Continuous mode: do not stop on errors |
| `-f <file>` | Read operations from a file |
| `-F <prefix>` | URL prefix for files (default `file:///tmp/`) |
| `-M` | Manage DSA IT control (`-MM` critical) |
| `-P <version>` | LDAP protocol version (default 3) |

### Output

| Flag | Meaning |
|------|---------|
| `-L` | LDIFv1 output |
| `-LL` | LDIF without comments |
| `-LLL` | LDIF without comments or version |
| `-t` | Write binary values to files in the temp directory |
| `-tt` | Write all values to files |
| `-T <path>` | Directory for `-t`/`-tt` output |
| `-o ldif_wrap=<width\|no>` | LDIF line wrapping |
| `-v` | Verbose diagnostics |
| `-V` / `-VV` | Version info |

### Authentication and transport

| Flag | Meaning |
|------|---------|
| `-x` | Simple authentication |
| `-D <binddn>` | Bind DN |
| `-w <passwd>` | Bind password (leaks in `ps`) |
| `-W` | Prompt for the bind password |
| `-y <file>` | Read the password from a file |
| `-H <uri>` | LDAP URI (`ldap://`, `ldaps://`, `ldapi://`) |
| `-Z` | StartTLS request |
| `-ZZ` | StartTLS required (fails if unavailable) |
| `-e <ext>` | General extension (see below) |
| `-I` | SASL interactive mode |
| `-Q` | SASL quiet mode |
| `-R <realm>` | SASL realm |
| `-U <authcid>` | SASL authentication identity |
| `-X <authzid>` | SASL authorization identity |
| `-Y <mech>` | SASL mechanism |
| `-O <props>` | SASL security properties |
| `-N` | Do not reverse-resolve the SASL host name |

### Extensions (`-E` search, `-e` general)

Search extensions:

| Extension | Meaning |
|-----------|---------|
| `pr=<size>[/prompt\|noprompt]` | Paged results (RFC 2696) |
| `sss=<attr[:OID]>...` | Server-side sorting (RFC 2891) |
| `mv=<filter>` | Matched values (RFC 3876) |
| `subentries[=true\|false]` | Subentries (RFC 3672) |
| `sync=ro[/<cookie>]` / `sync=rp[...]` | LDAP Sync (RFC 4533) |
| `vlv=<before>/<after>(/<offset>/<count>\|<value>)` | Virtual list view |
| `deref=<derefAttr:attr>...` | Dereference control |
| `dirSync=<flags>/<maxAttrCount>[/<cookie>]` | AD DirSync |
| `extendedDn=<flag>` | AD extended DN |
| `showDeleted` | AD deleted objects |
| `serverNotif` | AD server notification |
| `accountUsability` | Netscape account usability |
| `domainScope` | Domain scope |
| `!dontUseCopy` | Do not use a copy |
| `!<oid>[=:<value>\|::<b64value>]` | Generic control |

General extensions:

| Extension | Meaning |
|-----------|---------|
| `[!]assert=<filter>` | RFC 4528 assertion |
| `[!]authzid=<authzid>` | RFC 4370 proxy authz |
| `[!]bauthzid` | RFC 3829 authz identity |
| `[!]manageDSAit` | RFC 3296 |
| `[!]noop` | No-op control |
| `ppolicy` | Password policy |
| `[!]postread[=<attrs>]` / `[!]preread[=<attrs>]` | RFC 4527 |
| `[!]relax` | Relax rules |
| `[!]sessiontracking[=<user>]` | Session tracking |

## Common AD filters

| Goal | Filter |
|------|--------|
| All users | `(&(objectClass=user)(objectCategory=person))` |
| All groups | `(objectClass=group)` |
| All computers | `(objectClass=computer)` |
| Disabled accounts | `(userAccountControl:1.2.840.113556.1.4.803:=2)` |
| Password never expires | `(userAccountControl:1.2.840.113556.1.4.803:=65536)` |
| No Kerberos pre-auth (AS-REP) | `(userAccountControl:1.2.840.113556.1.4.803:=4194304)` |
| Kerberoastable | `(servicePrincipalName=*)` |
| Admin-count protected | `(adminCount=1)` |
| Unconstrained delegation | `(userAccountControl:1.2.840.113556.1.4.803:=1048576)` |
| Constrained delegation | `(msDS-AllowedToDelegateTo=*)` |
| Protocol transition | `(userAccountControl:1.2.840.113556.1.4.803:=524288)` |
| Shadow credentials | `(msDS-KeyCredentialLink=*)` |
| Recursive group membership | `(memberOf:1.2.840.113556.1.4.1941:=CN=Domain Admins,CN=Users,DC=acme,DC=local)` |
| Trusts | `(objectClass=trustedDomain)` |
| Fine-grained password policy | `(objectClass=msDS-PasswordSettings)` |

Bitwise matching rule `1.2.840.113556.1.4.803` is LDAP_MATCHING_RULE_BIT_AND;
`1.2.840.113556.1.4.1941` is LDAP_MATCHING_RULE_IN_CHAIN.

## Recipes

Authorized directories only.

**1. RootDSE recon.** No bind required on many servers.

```bash
ldapsearch -x -H ldap://10.10.10.5 -s base -b '' '(objectClass=*)' \
  namingContexts supportedControl supportedLDAPVersion
```

**2. Anonymous base search.**

```bash
ldapsearch -x -H ldap://10.10.10.5 -b 'dc=acme,dc=local' -s base '(objectClass=*)'
```

**3. Authenticated user lookup.**

```bash
ldapsearch -x -H ldap://10.10.10.5 \
  -D 'CN=alice,CN=Users,DC=acme,DC=local' -W \
  -b 'dc=acme,dc=local' '(sAMAccountName=alice)' cn mail memberOf
```

**4. Enumerate users with paging.** Without paging, AD caps the result set.

```bash
ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W \
  -b 'dc=acme,dc=local' -E pr=1000/noprompt \
  '(&(objectClass=user)(objectCategory=person))' sAMAccountName userAccountControl
```

**5. Find Kerberoastable accounts.**

```bash
ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W \
  -b 'dc=acme,dc=local' -E pr=1000/noprompt \
  '(&(objectClass=user)(servicePrincipalName=*))' \
  sAMAccountName servicePrincipalName
```

**6. Find AS-REP-roastable accounts.**

```bash
ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W \
  -b 'dc=acme,dc=local' \
  '(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))' \
  sAMAccountName
```

**7. Recursive group membership.**

```bash
ldapsearch -x -H ldap://10.10.10.5 -D 'ACME\alice' -W \
  -b 'dc=acme,dc=local' \
  '(memberOf:1.2.840.113556.1.4.1941:=CN=Domain Admins,CN=Users,DC=acme,DC=local)' \
  sAMAccountName
```

**8. LDAPS with a clean LDIF output.**

```bash
ldapsearch -x -H ldaps://dc.acme.local -D 'ACME\alice' -W \
  -b 'dc=acme,dc=local' -LLL '(objectClass=group)' cn member
```

**9. StartTLS required.**

```bash
ldapsearch -x -H ldap://10.10.10.5 -ZZ -D 'ACME\alice' -W \
  -b 'dc=acme,dc=local' '(objectClass=*)' namingContexts
```

## Output & pipelines

- Default output is LDIF with comments and a version header. `-LLL` removes
  both, which is what scripts usually want.
- Entries are separated by blank lines; attributes are `name: value` or
  `name:: base64`.
- `-t` writes binary attribute values to files and prints the path; `-T` sets
  the directory.
- `-o ldif_wrap=no` disables line wrapping, which helps `grep`/`awk`.
- `-z` is a client-side size limit; the server may impose a smaller one. Use
  `-E pr=...` to page instead of raising `-z`.
- `-c` keeps going after a failed entry; without it, the first error aborts the
  search.
- `-v` prints the filter and connection details, useful when a search returns
  nothing.

## Gotchas & operational security

- **`-h` is not help.** It is an invalid option; the usage block is printed as
  an error. Use `man ldapsearch`.
- **`-x` is usually required.** Without it, ldapsearch tries SASL and may fail
  before it ever binds.
- **Anonymous bind is often disabled.** A failure with no `-D` is normal on
  modern AD; it is not proof of a misconfiguration.
- **`-ZZ` fails hard if StartTLS is unavailable.** Try `-Z` (opportunistic) or
  `ldaps://` to distinguish policy from capability.
- **Paging changes results.** A search without `-E pr=` may silently truncate
  at the server's size limit; always page when enumerating.
- **Base64 values are not encrypted.** Decode them; they are often binary SIDs,
  certificates, or GUIDs.
- **Passwords on the command line leak.** Use `-W` or `-y`.
- **Filters are code.** A malformed filter returns an error, not a partial
  result; test with `-n` or a narrow base first.
- **Directory data is personal data.** Store LDIF in the engagement directory
  and delete at close.

## Legal & safety

Query only directories you own or have **explicit written authorization** to
access. Directory enumeration exposes personal data, credentials metadata, and
organizational structure. Do not use discovered accounts for access beyond the
agreed scope, and do not retain LDIF exports longer than necessary.

## Related tools

- [`rpcclient/`](../rpcclient/README.md) — SAMR-based enumeration for comparison.
- [`netexec/`](../netexec/README.md) — LDAP/SMB enumeration and spraying.
- [`enum4linux-ng/`](../enum4linux-ng/README.md) — broad SMB/AD enumeration.
- [`impacket/`](../impacket/README.md) — `GetADUsers`, `GetUserSPNs`,
  `GetNPUsers`, and `ldapdomaindump`-style workflows.
- [`kerbrute/`](../kerbrute/README.md) — Kerberos user enumeration.
- [`nmap/`](../nmap/README.md) — `ldap-rootdse` and `ldap-search` NSE scripts.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- OpenLDAP ldapsearch man page: https://www.openldap.org/software/man.cgi?query=ldapsearch
- RFC 4515 (LDAP search filters): https://www.rfc-editor.org/rfc/rfc4515
- RFC 2696 (paged results): https://www.rfc-editor.org/rfc/rfc2696
- AD matching rules: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/

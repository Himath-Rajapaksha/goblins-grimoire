# ghidra — software reverse-engineering suite

- **Version:** 12.1.4 PUBLIC, build 2026-Sep-21 (Ghidra
  `Ghidra/application.properties`)
- **Install:** `/opt/ghidra/ghidra_12.1.4_PUBLIC/` (not on the default
  `PATH`)
- **Launchers:** `ghidraRun` (GUI), `support/analyzeHeadless` (batch),
  `support/pyghidraRun` (CPython), `server/ghidraSvr` (shared server)
- **Help snapshot:** [`help-analyzeHeadless.txt`](help-analyzeHeadless.txt) —
  `support/analyzeHeadless -help`
- **Upstream:** https://github.com/NationalSecurityAgency/ghidra
- **Docs:** `docs/GettingStarted.md`, `support/analyzeHeadlessREADME.md`,
  `docs/GhidraAPI_javadoc.zip`, `docs/CheatSheet.html`
- **Java:** OpenJDK 21.0.13 (the launcher uses the system JDK)
- **Licence:** Apache 2.0

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Ghidra is a full software reverse-engineering environment. It imports a
binary, identifies code and data, disassembles it, builds functions and
cross-references, recovers data types, and decompiles to C-like pseudocode.
Around that core it adds a debugger, version tracking, function ID, and BSim
(behavioural similarity) for comparing large corpora.

The two surfaces matter for this repo:

- **GUI** — interactive analysis, graph views, decompiler, debugger.
- **Headless** — `analyzeHeadless` imports or reprocesses programs, runs
  analysis, and executes Java or Python scripts over them. This is the mode
  that fits repeatable, evidence-producing work.

The processor modules shipped here cover:

```text
6502 68000 8048 8051 8085 8086 AARCH64 ARM Atmel BPF CP1600 CR16 Dalvik
DATA eBPF HCS08 HCS12 Hexagon JVM Loongarch M16C M8C MC6800 MCS96 MIPS
NDS32 PA-RISC PIC PowerPC RISCV Sparc SuperH SuperH4 TI_MSP430 Toy tricore
V850 x86 Xtensa Z80
```

That includes x86/x86-64, ARM/AArch64, MIPS, PowerPC, RISC-V, SPARC,
LoongArch, Dalvik, JVM bytecode, and eBPF.

## Launch modes

| Mode | Command | Use |
|------|---------|-----|
| GUI | `/opt/ghidra/ghidra_12.1.4_PUBLIC/ghidraRun` | Interactive analysis |
| Headless | `.../support/analyzeHeadless` | Batch import, analysis, scripts |
| PyGhidra | `.../support/pyghidraRun` | CPython scripting environment |
| Server | `.../server/ghidraSvr console` | Shared multi-user repository |
| Server admin | `.../server/svrAdmin` | Users, permissions, repositories |

`pyghidraRun` needs the `pyghidra` Python module. If it is absent, the
script offers to install it from
`Ghidra/Features/PyGhidra/pypkg/dist`; the manual equivalent is:

```bash
python3 -m pip install --no-index \
  -f /opt/ghidra/ghidra_12.1.4_PUBLIC/Ghidra/Features/PyGhidra/pypkg/dist \
  pyghidra
```

Jython is **not** installed in this build. It ships as an optional extension
at `Extensions/Ghidra/ghidra_12.1.4_PUBLIC_20260921_Jython.zip`; without it,
Python scripts run through PyGhidra/CPython, not Jython.

## Headless flags

| Flag | Meaning |
|------|---------|
| `<project_location> <project_name>[/<folder>]` | Local project target |
| `ghidra://<server>[:<port>]/<repo>[/<folder>]` | Shared-server target |
| `-import <file\|dir> ...` | Import one or more files or directories |
| `-process [<project_file>]` | Reprocess programs already in the project |
| `-preScript <Script.ext> [args]` | Run before analysis; repeatable |
| `-postScript <Script.ext> [args]` | Run after analysis; repeatable |
| `-scriptPath "<p1>[;<p2>...]"` | Extra script search directories |
| `-propertiesPath "<p1>[;<p2>...]"` | Where to find script `.properties` files |
| `-scriptlog <file>` | Separate script log |
| `-log <file>` | Headless log file |
| `-overwrite` | Replace an existing program with the same name |
| `-mirror` | Mirror the import directory structure |
| `-recursive [<depth>]` | Recurse into directories and container files |
| `-readOnly` | Do not save changes to the project |
| `-deleteProject` | Delete the project when finished |
| `-noanalysis` | Import or process without auto-analysis |
| `-processor <languageID>` | Force a processor language |
| `-cspec <compilerSpecID>` | Force a compiler specification |
| `-analysisTimeoutPerFile <seconds>` | Per-file analysis deadline |
| `-keystore <path>` | PKI keystore for server auth |
| `-connect [<userID>]` | Connect to a Ghidra Server |
| `-p` | Prompt for the server password |
| `-commit ["<comment>"]` | Commit changes to the server |
| `-okToDelete` | Allow scripts to delete programs |
| `-max-cpu <n>` | Cap analysis threads |
| `-librarySearchPaths "<p1>[;<p2>...]"` | External library search paths |
| `-loader <name>` | Force a specific loader |
| `-loader-<arg> <value>` | Pass a loader-specific option |

`-import` and `-process` are mutually exclusive. `-readOnly`,
`-deleteProject`, and `-okToDelete` change what happens to the project, so
read the README section for each before using them in a batch.

## Recipes

**1. Launch the GUI.**

```bash
/opt/ghidra/ghidra_12.1.4_PUBLIC/ghidraRun
```

**2. Import and analyze one binary into a local project.**

```bash
/opt/ghidra/ghidra_12.1.4_PUBLIC/support/analyzeHeadless \
  ~/ghidra-projects demo -import /bin/ls
```

**3. Import a directory recursively without analysis.**

```bash
.../support/analyzeHeadless ~/ghidra-projects demo \
  -import ~/samples -recursive -noanalysis
```

**4. Reprocess an existing program.**

```bash
.../support/analyzeHeadless ~/ghidra-projects demo \
  -process ls -noanalysis
```

**5. Run a built-in script after analysis.**

```bash
.../support/analyzeHeadless ~/ghidra-projects demo \
  -import /bin/ls -postScript ExportFunctionInfoScript.java
```

**6. Run a script from a custom directory with arguments.**

```bash
.../support/analyzeHeadless ~/ghidra-projects demo \
  -scriptPath ~/ghidra-scripts \
  -import /bin/ls -postScript MyScript.java arg1 arg2
```

**7. Force the processor language.**

```bash
.../support/analyzeHeadless ~/ghidra-projects demo \
  -import firmware.bin -processor ARM:LE:32:Cortex
```

**8. Force a loader for a raw blob.**

```bash
.../support/analyzeHeadless ~/ghidra-projects demo \
  -import firmware.bin -loader BinaryLoader \
  -loader-imagebase 0x08000000
```

**9. Cap analysis threads and time.**

```bash
.../support/analyzeHeadless ~/ghidra-projects demo \
  -import ~/samples -recursive -max-cpu 4 -analysisTimeoutPerFile 300
```

**10. Keep the project untouched.**

```bash
.../support/analyzeHeadless ~/ghidra-projects demo \
  -process -recursive -readOnly
```

**11. Delete a throwaway project after the run.**

```bash
.../support/analyzeHeadless /tmp/ghidra-scratch scratch \
  -import /bin/ls -deleteProject
```

**12. Start a Ghidra Server console.**

```bash
/opt/ghidra/ghidra_12.1.4_PUBLIC/server/ghidraSvr console
```

**13. Add a server user and grant repository access.**

```bash
.../server/svrAdmin -add alice
.../server/svrAdmin -grant alice +w myrepo
```

**14. Run a headless job against a server repository.**

```bash
.../support/analyzeHeadless \
  ghidra://127.0.0.1:13100/myrepo -process -connect alice -p
```

**15. Launch PyGhidra.**

```bash
/opt/ghidra/ghidra_12.1.4_PUBLIC/support/pyghidraRun
```

## Gotchas

- **Ghidra is not on `PATH`.** Every command needs the full
  `/opt/ghidra/ghidra_12.1.4_PUBLIC/...` path or a symlink.
- **Headless cannot open a project that the GUI has open.** Close the GUI
  project first, or the analyzer exits with a lock error.
- **`-import` and `-process` cannot be combined.** Pick one mode per run.
- **`-noanalysis` still imports.** It skips auto-analysis, not the import;
  use it when a pre-script must set language or memory first.
- **`-processor` needs the exact language ID.** `ARM:LE:32:Cortex` is a
  language ID; `arm` is not. List IDs in the GUI or from the processor
  modules.
- **`-loader-*` options are loader-specific.** `-loader-imagebase` only
  makes sense for the binary loader; the PE and Mach-O loaders use different
  arguments.
- **`-recursive` depth means container depth, not directory depth.** The
  optional number controls nesting inside archives; directories always
  recurse when the flag is present.
- **Bulk import skips dot-files.** A file beginning with `.` is ignored
  unless it is named explicitly.
- **`-readOnly` does not mean "no writes anywhere".** It prevents project
  saves; scripts can still write external files.
- **`-deleteProject` is destructive.** It removes the project after the
  run, including any programs a script expected to keep.
- **`-scriptPath` home variables need escaping on Unix.** `$GHIDRA_HOME`
  and `$USER_HOME` must be written as `\$GHIDRA_HOME` and `\$USER_HOME`.
- **Script names include the extension.** `MyScript.java`, not `MyScript`.
  Arguments follow the script name.
- **`askXxx()` works headless, but not interactively.** Values come from a
  matching `.properties` file or from script arguments; otherwise the script
  can block or fail.
- **Jython is optional here.** A `.py` script needs PyGhidra unless the
  Jython extension is installed; the two are not interchangeable.
- **Analysis is memory-hungry.** `GHIDRA_MAXMEM` or
  `GHIDRA_GUI_MAXMEM` caps the heap; `-max-cpu` caps threads, not memory.
- **Ghidra Server defaults to TCP 13100.** Exposing it beyond a trusted
  network without TLS and strong auth is a real risk.
- **The decompiler is an aid, not ground truth.** Optimised or obfuscated
  code can produce plausible but wrong C; verify against the listing.
- **A project is a directory, not a single file.** Copy the `.rep` and
  `.gpr` together, or the project will not open.

## Related docs

- [`binwalk`](../binwalk/README.md) — find and extract the firmware first
- [`checksec`](../checksec/README.md) — hardening properties of the target
- [`pwndbg`](../pwndbg/README.md) — dynamic analysis under GDB
- [`gef`](../gef/README.md) — alternative GDB environment
- [`ropgadget`](../ropgadget/README.md) — gadget search on the same binary
- [`ropper`](../ropper/README.md) — gadget search and conversion
- [`one-gadget`](../one-gadget/README.md) — one-shot libc gadgets
- [`strace`](../strace/README.md) — runtime behaviour before static analysis
- [`volatility3`](../volatility3/README.md) — memory images and extracted code

# ghidra — command scenarios for deep research

Installed build: Ghidra 12.1.4 PUBLIC (2026-Sep-21) at
`/opt/ghidra/ghidra_12.1.4_PUBLIC/`. Every flag below appears in
[`help-analyzeHeadless.txt`](help-analyzeHeadless.txt) or
`support/analyzeHeadlessREADME.md`. Set one variable for the whole session:

```bash
export GHIDRA_HOME=/opt/ghidra/ghidra_12.1.4_PUBLIC
```

> **Scope:** Reverse-engineering a binary you do not own or have permission
> to analyse may violate copyright, contract, or law. Work on your own
> samples, licensed binaries, or an authorised engagement target, and keep
> derived artefacts inside the engagement boundary.

## How to invoke

```text
@deep-research For an authorised reverse-engineering workflow with Ghidra
12.1.4 headless mode, explain how import, loader selection, processor
language selection, auto-analysis, pre/post scripts, and project disposition
interact. Compare the headless analyzer with the GUI and PyGhidra, and
explain what `-readOnly`, `-overwrite`, `-recursive`, and `-deleteProject`
change on disk. Cite the shipped analyzeHeadlessREADME, GettingStarted.md,
the Ghidra API javadoc, and the Ghidra repository. Separate confirmed
behaviour from inference and state confidence.
```

Capture the baseline before creating projects:

```bash
mkdir -p ~/ghidra-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/ghidra-demo/evidence/run-start.txt
df -h ~ > ~/ghidra-demo/evidence/disk-before.txt
java -version > ~/ghidra-demo/evidence/java-version.txt 2>&1
```

## Scenario 1 — install layout, versions, and launch modes

Ghidra is not on `PATH`; confirm the install root and which launch mode each
script provides before running a job.

| # | Command | How to use |
|---|---------|------------|
| 1 | `ls "$GHIDRA_HOME"` | Top-level layout |
| 2 | `grep -E 'application.version\|application.build.date' "$GHIDRA_HOME/Ghidra/application.properties"` | Exact version and build |
| 3 | `java -version` | JDK the launcher will use |
| 4 | `"$GHIDRA_HOME/ghidraRun"` | GUI launch (needs a display) |
| 5 | `"$GHIDRA_HOME/support/analyzeHeadless"` | Headless usage banner |
| 6 | `"$GHIDRA_HOME/support/analyzeHeadless" -help` | Full flag summary |
| 7 | `ls "$GHIDRA_HOME/Ghidra/Processors"` | Processor modules |
| 8 | `ls "$GHIDRA_HOME/Ghidra/Features"` | Installed features |
| 9 | `ls "$GHIDRA_HOME/Extensions/Ghidra" \| grep -i jython` | Is Jython installed? |
| 10 | `ls "$GHIDRA_HOME/Ghidra/Features/PyGhidra/pypkg/dist" \| head` | PyGhidra wheels |
| 11 | `"$GHIDRA_HOME/server/ghidraSvr"` | Server usage banner |
| 12 | `"$GHIDRA_HOME/server/svrAdmin" 2>&1 \| head -20` | Server admin commands |

**Research questions**
- Which JDK does the launcher require, and which one is selected here?
- Which launch mode is appropriate for a reproducible batch job?
- Is Jython available by default, or only as an extension?
- Which processor modules cover the target architecture?
- What does the server provide that a local project does not?

**Sources:** `docs/GettingStarted.md`; `Ghidra/application.properties`;
`support/analyzeHeadlessREADME.md`; `server/svrREADME.html`.
**Output:** an install record with the chosen launch mode and JDK.

## Scenario 2 — first headless import and analysis

A project is a directory pair (`.gpr` and `.rep`). Import creates it;
analysis runs by default unless suppressed.

| # | Command | How to use |
|---|---------|------------|
| 13 | `mkdir -p ~/ghidra-projects` | Project root |
| 14 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls` | Import and analyse |
| 15 | `ls -la ~/ghidra-projects` | Confirm `.gpr` and `.rep` |
| 16 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -process ls -noanalysis` | Reprocess without analysis |
| 17 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -overwrite` | Replace the existing program |
| 18 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -log /tmp/ghidra-demo.log` | Capture the run log |
| 19 | `tail -20 /tmp/ghidra-demo.log` | Confirm the run completed |
| 20 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -scriptlog /tmp/ghidra-script.log` | Separate script log |
| 21 | `grep -iE 'analysis\|ERROR' /tmp/ghidra-demo.log \| tail -20` | Analysis and error lines |
| 22 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -process -recursive -noanalysis` | Process every program in the folder |
| 23 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -max-cpu 2` | Cap analysis threads |
| 24 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -analysisTimeoutPerFile 60` | Per-file deadline |
| 25 | `du -sh ~/ghidra-projects/demo.rep` | Project size on disk |
| 26 | `find ~/ghidra-projects -maxdepth 2 -type f \| head -20` | Project file inventory |

**Research questions**
- What does the `.rep` directory contain that the `.gpr` does not?
- Which files change when `-overwrite` is used?
- Does `-analysisTimeoutPerFile` stop analysis or mark it incomplete?
- What happens if the same program name is imported without `-overwrite`?
- How much disk does a typical small binary consume?

**Sources:** `support/analyzeHeadlessREADME.md` (`-import`, `-overwrite`,
`-log`, `-max-cpu`, `-analysisTimeoutPerFile`); Ghidra project docs.
**Output:** a project inventory with size and run-log evidence.

## Scenario 3 — corpora, recursion, wildcards, and hidden files

Bulk import is where directory layout and naming rules matter.

| # | Command | How to use |
|---|---------|------------|
| 27 | `mkdir -p ~/samples/bin` | Sample tree |
| 28 | `cp /bin/ls /bin/cat /bin/echo ~/samples/bin/` | Three benign samples |
| 29 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects corpus -import ~/samples/bin -recursive` | Recursive directory import |
| 30 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects corpus -import ~/samples/bin/*` | Shell wildcard import |
| 31 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects corpus -import ~/samples/bin -mirror` | Preserve directory structure |
| 32 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects corpus -process -recursive -noanalysis` | Reprocess the corpus |
| 33 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects corpus -import ~/samples/bin/ls -overwrite` | Replace one program |
| 34 | `touch ~/samples/.hidden` | Create a dot-file |
| 35 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects corpus -import ~/samples -recursive` | Dot-file is skipped |
| 36 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects corpus -import ~/samples/.hidden` | Explicit dot-file import |
| 37 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects corpus -import ~/samples -recursive 3` | Container recursion depth |
| 38 | `ls ~/ghidra-projects` | Project set after the runs |

**Research questions**
- Does `-recursive` with no number recurse into directories, containers, or
  both?
- What is the difference between `-mirror` and a normal import?
- Why is a dot-file skipped in bulk mode but accepted when named?
- Does the wildcard expand in the shell or inside Ghidra?
- What happens to duplicate program names in a corpus import?

**Sources:** `support/analyzeHeadlessREADME.md` (`-import`, `-recursive`,
`-mirror`, wildcards, hidden files).
**Output:** a corpus-import record with the naming and skip rules observed.

## Scenario 4 — scripts, arguments, and logs

Headless scripts are the automation surface. Create a minimal Java script
that prints program facts and its arguments:

```java
//@category Examples
import ghidra.app.script.GhidraScript;
import ghidra.program.model.listing.FunctionIterator;

public class SummaryScript extends GhidraScript {
    @Override
    public void run() throws Exception {
        int count = 0;
        FunctionIterator it = currentProgram.getFunctionManager().getFunctions(true);
        while (it.hasNext()) { it.next(); count++; }
        println("program=" + currentProgram.getName());
        println("imageBase=" + currentProgram.getImageBase());
        println("functions=" + count);
        for (String arg : getScriptArgs()) {
            println("arg=" + arg);
        }
    }
}
```

| # | Command | How to use |
|---|---------|------------|
| 39 | `mkdir -p ~/ghidra-scripts` | Script directory |
| 40 | `cat > ~/ghidra-scripts/SummaryScript.java <<'EOF'` … `EOF` | Write the script above |
| 41 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -import /bin/ls -postScript SummaryScript.java` | Run after analysis |
| 42 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -process ls -postScript SummaryScript.java` | Run on an existing program |
| 43 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -import /bin/ls -preScript SummaryScript.java -noanalysis` | Run before analysis |
| 44 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -import /bin/ls -postScript SummaryScript.java alpha beta` | Pass script arguments |
| 45 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -import /bin/ls -postScript SummaryScript.java -scriptlog /tmp/ghidra-script.log` | Capture script output separately |
| 46 | `cat /tmp/ghidra-script.log` | Read the script output |
| 47 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -postScript FixupNoReturnFunctionsScript.java` | Run a built-in script |
| 48 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -import /bin/ls -preScript SummaryScript.java -postScript SummaryScript.java` | Pre and post in one run |
| 49 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -propertiesPath ~/ghidra-scripts -import /bin/ls -postScript SummaryScript.java` | Add a properties path |
| 50 | `grep 'functions=' /tmp/ghidra-script.log` | Extract the function count |
| 51 | `head -5 ~/ghidra-scripts/SummaryScript.java` | Confirm the script source |
| 52 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -import /bin/ls -postScript SummaryScript.java one two three` | Three arguments |

**Research questions**
- Does `-preScript` run before or after the loader and language selection?
- How does a headless script obtain values from `askXxx()`?
- Which log receives `println()` output with and without `-scriptlog`?
- Are script arguments strings only, and how are spaces handled?
- What happens when a script throws an exception: is the program kept?

**Sources:** `support/analyzeHeadlessREADME.md` (Scripting, askXxx,
arguments); `docs/GhidraAPI_javadoc.zip`; built-in scripts under
`Ghidra/Features/*/ghidra_scripts/`.
**Output:** a script-run record with function counts and argument evidence.

## Scenario 5 — processor, compiler spec, and loader control

When auto-detection guesses wrong, these flags decide how bytes become
instructions and memory blocks.

| # | Command | How to use |
|---|---------|------------|
| 53 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -processor x86:LE:64:default` | Force x86-64 |
| 54 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -processor arm` | Deliberately wrong ID |
| 55 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -cspec gcc` | Force the GCC spec |
| 56 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -loader ElfLoader` | Force the ELF loader |
| 57 | `dd if=/dev/zero of=/tmp/firmware.bin bs=1024 count=4` | Raw firmware blob |
| 58 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects fw -import /tmp/firmware.bin -loader BinaryLoader -loader-imagebase 0x08000000` | Raw blob at a base address |
| 59 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects fw -import /tmp/firmware.bin -loader BinaryLoader -loader-imagebase 0x08000000 -processor ARM:LE:32:Cortex` | Raw ARM image |
| 60 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -loader ElfLoader -loader-applyRelocations true` | Loader-specific option |
| 61 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -loader PeLoader` | Wrong loader for an ELF |
| 62 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -processor x86:LE:64:default -cspec gcc` | Language and spec together |
| 63 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -loader ElfLoader -loader-imagebase 0x400000` | Override the image base |
| 64 | `grep -iE 'language\|processor\|loader' /tmp/ghidra-demo.log \| tail -20` | What was actually selected |

**Research questions**
- Where can you list the exact language IDs and compiler specs?
- Does a wrong `-processor` fail the import or analyse garbage?
- Which loader arguments apply to ELF, PE, Mach-O, and raw binary?
- Does `-loader-imagebase` change the file's recorded base or only the
  in-memory image?
- What does `-cspec` change in the decompiler output?

**Sources:** `support/analyzeHeadlessREADME.md` (loader options);
`Ghidra/Processors/*/data/languages/*.ldefs`; Ghidra loader docs.
**Output:** a language/loader decision table with the observed selection.

## Scenario 6 — project disposition and resource limits

These flags decide whether the project survives, whether changes are saved,
and how much CPU and time analysis may consume.

| # | Command | How to use |
|---|---------|------------|
| 65 | `du -sh ~/ghidra-projects/demo.rep` | Size before maintenance |
| 66 | `ls -la ~/ghidra-projects` | Project pair inventory |
| 67 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -process -recursive -readOnly` | Do not save changes |
| 68 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -process -recursive -noanalysis` | Process without analysis |
| 69 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -overwrite` | Replace a program |
| 70 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -mirror` | Mirror the source layout |
| 71 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -recursive 2` | Container depth 2 |
| 72 | `"$GHIDRA_HOME/support/analyzeHeadless" /tmp/ghidra-scratch scratch -import /bin/ls -deleteProject` | Throwaway project |
| 73 | `ls /tmp/ghidra-scratch` | Prove it was removed |
| 74 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -process -recursive -okToDelete` | Allow script-driven deletion |
| 75 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -import /bin/ls -max-cpu 1 -analysisTimeoutPerFile 30` | Strict resource caps |
| 76 | `grep -iE 'ERROR\|WARN\|lock' /tmp/ghidra-demo.log \| tail -20` | Failure signals |

**Research questions**
- Does `-readOnly` prevent temporary files as well as project saves?
- What exactly does `-okToDelete` authorise, and who can trigger it?
- Is `-deleteProject` honoured when analysis fails?
- Does `-max-cpu` bound memory as well as threads?
- What lock file blocks a second headless run on the same project?

**Sources:** `support/analyzeHeadlessREADME.md` (`-readOnly`,
`-deleteProject`, `-okToDelete`, `-max-cpu`); Ghidra project-lock docs.
**Output:** a disposition record with before/after project state.

## Scenario 7 — Ghidra Server and shared repositories

A shared server is a multi-user repository on TCP 13100 by default. Treat
it as a lab service, not something to expose casually.

| # | Command | How to use |
|---|---------|------------|
| 77 | `"$GHIDRA_HOME/server/ghidraSvr" console` | Foreground server console |
| 78 | `"$GHIDRA_HOME/server/ghidraSvr" status` | Service status |
| 79 | `"$GHIDRA_HOME/server/svrAdmin" -add alice` | Add a user |
| 80 | `"$GHIDRA_HOME/server/svrAdmin" -grant alice +w myrepo` | Grant write access |
| 81 | `"$GHIDRA_HOME/server/svrAdmin" -revoke alice myrepo` | Revoke access |
| 82 | `"$GHIDRA_HOME/support/analyzeHeadless" ghidra://127.0.0.1:13100/myrepo -process -connect alice -p` | Headless against the server |
| 83 | `"$GHIDRA_HOME/support/analyzeHeadless" ghidra://127.0.0.1:13100/myrepo -import /bin/ls -connect alice -p -commit 'import ls'` | Import and commit |
| 84 | `"$GHIDRA_HOME/support/analyzeHeadless" ghidra://127.0.0.1:13100/myrepo -process -connect alice -keystore ~/.keystore` | PKI authentication |
| 85 | `ss -ltnp \| grep 13100` | Confirm the listener |
| 86 | `grep -nE 'port\|ssl' "$GHIDRA_HOME/server/server.conf" \| head` | Server port and TLS settings |

**Research questions**
- Which authentication methods does the server support for batch use?
- What does `-commit` do that a normal local save does not?
- Is the default transport encrypted, and what has to be configured?
- What is the difference between `+r`, `+w`, and `+a` grants?
- Can a headless job run while a user has the repository open?

**Sources:** `server/svrREADME.html`; `server/server.conf`;
`support/analyzeHeadlessREADME.md` (Authentication, `-connect`, `-commit`,
`-keystore`).
**Output:** a server-access record with the auth method and permission set.

## Scenario 8 — verification, comparison, and cleanup

Prove the analysis is reproducible, then remove the projects and samples.

| # | Command | How to use |
|---|---------|------------|
| 87 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -process ls -postScript SummaryScript.java -scriptlog /tmp/ghidra-verify.log` | Re-run the summary script |
| 88 | `grep 'functions=' /tmp/ghidra-verify.log` | Function count for comparison |
| 89 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -process ls -readOnly` | Verify without saving |
| 90 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -scriptPath ~/ghidra-scripts -process ls -noanalysis -postScript SummaryScript.java` | Script on unanalysed program |
| 91 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -process ls -analysisTimeoutPerFile 10` | Short deadline |
| 92 | `"$GHIDRA_HOME/support/analyzeHeadless" ~/ghidra-projects demo -process ls -max-cpu 1` | Single-thread run |
| 93 | `"$GHIDRA_HOME/support/analyzeHeadless" /tmp/ghidra-verify verify -scriptPath ~/ghidra-scripts -import /bin/ls -postScript SummaryScript.java -log /tmp/ghidra-fresh.log` | Fresh-project comparison |
| 94 | `grep -h 'functions=' /tmp/ghidra-verify.log /tmp/ghidra-fresh.log \| sort \| uniq -c` | Compare the counts |
| 95 | `rm -rf /tmp/ghidra-scratch /tmp/ghidra-verify` | Remove scratch projects |
| 96 | `rm -rf ~/ghidra-projects/demo.gpr ~/ghidra-projects/demo.rep` | Remove the demo project |
| 97 | `ls ~/ghidra-projects` | Confirm what remains |
| 98 | `rm -rf ~/samples ~/ghidra-scripts` | Remove samples and scripts |
| 99 | `find ~/ghidra-projects -name '*.lock' -o -name '*.lock~'` | No stale locks |
| 100 | `df -h ~ \| diff - ~/ghidra-demo/evidence/disk-before.txt` | Disk back to baseline |

**Research questions**
- Is the function count stable across a fresh import and a reprocess?
- What changes when analysis is skipped but the script still runs?
- Does `-readOnly` leave the project byte-identical?
- Which artefacts remain after deleting a project?
- How do you prove no Ghidra process or lock survived the cleanup?

**Sources:** `support/analyzeHeadlessREADME.md`; Ghidra project docs;
the run logs produced above.
**Output:** a reproducibility record with two independent function counts
and a cleanup diff.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `ghidraRun`, `analyzeHeadless`, `pyghidraRun`, `ghidraSvr`, `svrAdmin` | 1 |
| `<project_location> <project_name>[/<folder>]` | 2, 3, 6, 8 |
| `ghidra://<server>[:<port>]/<repo>` | 7 |
| `-import`, `-process` | 2, 3, 4, 5, 6, 8 |
| `-preScript`, `-postScript`, script arguments | 4 |
| `-scriptPath`, `-propertiesPath`, `-scriptlog` | 4, 8 |
| `-log`, `-overwrite`, `-mirror`, `-recursive` | 2, 3, 6 |
| `-readOnly`, `-deleteProject`, `-okToDelete` | 6, 8 |
| `-noanalysis`, `-analysisTimeoutPerFile`, `-max-cpu` | 2, 3, 6, 8 |
| `-processor`, `-cspec` | 5 |
| `-loader`, `-loader-imagebase`, `-loader-applyRelocations` | 5 |
| `-keystore`, `-connect`, `-p`, `-commit` | 7 |
| `-librarySearchPaths`, `-loader-<arg>` | reference (loader/library variants) |

Full text: [`help-analyzeHeadless.txt`](help-analyzeHeadless.txt) and
`support/analyzeHeadlessREADME.md`.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Install layout, versions, launch modes | 12 |
| 2 | First headless import and analysis | 14 |
| 3 | Corpora, recursion, wildcards, hidden files | 12 |
| 4 | Scripts, arguments, and logs | 14 |
| 5 | Processor, compiler spec, loader control | 12 |
| 6 | Project disposition and resource limits | 12 |
| 7 | Ghidra Server and shared repositories | 10 |
| 8 | Verification, comparison, cleanup | 14 |
| **Total** | | **100** |

## Legal and safety

Ghidra produces derived artefacts (projects, decompiled code, exported
functions) that may contain proprietary material. Keep them inside the
engagement boundary, do not publish decompiled code you do not own, and
remember that a project is not encrypted. The Ghidra Server is a multi-user
service with its own authentication and ACLs; running it on an untrusted
network without TLS is a data-exposure risk. Delete throwaway projects and
samples when the analysis is finished.

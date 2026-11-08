# burp — command scenarios for deep research

Every Burp Suite Community launcher flag plus the GUI workflows those flags
set up. Flags from the installed build (`help.txt`). Burp's testing actions are
GUI operations, so they are listed separately and honestly labelled.

## How to invoke

```text
@deep-research For Burp Suite Community 2026.10, which Community limitations
still block scanner automation, and what is the maintained way to drive Burp
from CLI/CI without Pro? Cite PortSwigger docs, release notes, and extension
docs; return exact launch commands.
```

---

## Scenario 1 — normal launch and isolated profiles

| # | Command | How to use |
|---|---------|------------|
| 1 | `burp` | Launch with the default data directory |
| 2 | `burp --use-defaults` | Ignore saved settings; clean configuration |
| 3 | `burp --data-dir /home/user/burp-data` | Keep all state in a dedicated directory |
| 4 | `burp --use-defaults --data-dir /tmp/burp-clean` | Reproducible clean session |
| 5 | `burp --disable-extensions` | Start without third-party extensions |
| 6 | `burp --disable-auto-update` | Freeze the installed version |
| 7 | `burp --version` | Print version details |
| 8 | `burp --help` | Print launcher help |

**Research questions**
- Which settings live in the data dir versus the project file?
- Does `--use-defaults` also reset proxy listeners and CA trust?

**Sources**: PortSwigger launcher docs, release notes, support KB.

**Output**: a documented launch profile separating "clean" and "daily" state.

---

## Scenario 2 — projects and configuration

| # | Command | How to use |
|---|---------|------------|
| 9 | `burp --project-file engagement.burp` | Open or create a project file |
| 10 | `burp --project-file engagement.burp --auto-repair` | Repair a corrupted project |
| 11 | `burp --project-file engagement.burp --unpause-spider-and-scanner` | Resume paused scan activity on open |
| 12 | `burp --config-file project.json` | Load project configuration |
| 13 | `burp --config-file a.json --config-file b.json` | Layer multiple project configs |
| 14 | `burp --user-config-file user.json` | Load user-level configuration |
| 15 | `burp --user-config-file a.json --user-config-file b.json` | Layer user configs |

**Research questions**
- Precedence when `--config-file` is repeated — later wins?
- Which config keys are safe to version-control and which embed secrets?

**Sources**: PortSwigger config docs, extension config format, community
writeups on headless Burp setups.

**Output**: a versioned config set with secrets excluded.

---

## Scenario 3 — diagnostics and support

| # | Command | How to use |
|---|---------|------------|
| 16 | `burp --diagnostics` | Dump environment, memory, and install diagnostics |
| 17 | `burp --diagnostics > diag.txt` | Capture for a bug report or support ticket |

**Research questions**
- What does `--diagnostics` include that could leak paths or tokens before
  sharing it?

**Sources**: PortSwigger support guidance, diagnostics output itself.

**Output**: a redacted diagnostics file.

---

## Scenario 4 — Collaborator server mode

| # | Command | How to use |
|---|---------|------------|
| 18 | `burp --collaborator-server` | Run Burp as a Collaborator server |
| 19 | `burp --collaborator-server --collaborator-config collaborator.config` | Custom server configuration |

**Research questions**
- What are the DNS/HTTP requirements for a private Collaborator server, and
  which Burp editions can host one?
- How is the server key distributed to clients?

**Sources**: PortSwigger Collaborator docs, server config reference, community
self-hosting guides.

**Output**: a working private Collaborator endpoint or a documented reason the
Community build cannot provide one.

---

## Scenario 5 — extension development

| # | Command | How to use |
|---|---------|------------|
| 20 | `burp --developer-extension-class-name com.example.Ext` | Load a locally developed extension from the classpath |
| 21 | `burp --developer-extension-class-name com.example.Ext --disable-extensions` | Load only the named dev extension |

**Research questions**
- What classpath does Burp use for `--developer-extension-class-name`, and how
  do dependencies get resolved?
- Which Montoya API version matches the installed build?

**Sources**: Burp extension docs, Montoya API javadoc, extension repos.

**Output**: a repeatable dev-launch command and a matching API version note.

---

## Scenario 6 — GUI workflows (not CLI flags)

These are the operations the launcher sets up. They are UI paths, not command
line arguments — listed so the command forms are not confused with testing
steps.

| # | Workflow | How to use |
|---|----------|------------|
| 22 | Proxy → browser | Set browser proxy to `127.0.0.1:8080`, install Burp CA |
| 23 | Proxy → Repeater | Right-click a request → Send to Repeater (`Ctrl+R`) |
| 24 | Proxy → Intruder | Right-click → Send to Intruder (`Ctrl+I`) |
| 25 | Intruder attack types | Sniper, Battering ram, Pitchfork, Cluster bomb |
| 26 | Proxy → Scanner | Community has no active scanner; use manual + extensions |
| 27 | Collaborator client | Burp menu → Collaborator client → copy payload |
| 28 | Target → Scope | Scope include/exclude rules before any crawl |
| 29 | Extensions → BApp Store | Install/remove extensions; verify compatibility |
| 30 | Project options → Sessions | Cookie jar and macro handling for auth |
| 31 | User options → TLS | Client certificates and trust settings |

**Research questions**
- Which Repeater/Intruder behaviors differ between Community and Pro builds?
- Which BApp extensions are maintained for the installed Montoya API?

**Sources**: PortSwigger tool docs, BApp Store pages, release notes.

**Output**: a workflow cheat sheet with Community-accurate expectations.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Launch | (none), `--use-defaults`, `--data-dir` | 1 |
| Isolation | `--disable-extensions`, `--disable-auto-update` | 1 |
| Info | `--help`, `--version`, `--diagnostics` | 1, 3 |
| Project | `--project-file`, `--auto-repair`, `--unpause-spider-and-scanner` | 2 |
| Config | `--config-file`, `--user-config-file` | 2 |
| Collaborator | `--collaborator-server`, `--collaborator-config` | 4 |
| Extensions | `--developer-extension-class-name` | 5 |

Full per-flag text: [`help.txt`](help.txt).

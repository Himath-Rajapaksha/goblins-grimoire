# semgrep — command scenarios for deep research

Every semgrep command form, grouped by scenario. Subcommands from
[`help.txt`](help.txt); `scan` flags from the full manual snapshot
[`help-scan-full.txt`](help-scan-full.txt) (655 lines; the old
`help-scan.txt` only captured the `-h` failure).

## How to invoke

```text
@deep-research For Semgrep 1.184, which registry rulesets give real signal on
a modern TypeScript/React codebase, and how do I write a taint rule for
user-controlled input reaching `dangerouslySetInnerHTML`? Cite Semgrep docs,
registry rule pages, and rule-writing guides; return exact commands and rule
YAML.
```

---

## Scenario 1 — first scans and rule sources

| # | Command | How to use |
|---|---------|------------|
| 1 | `semgrep scan --config auto .` | Registry rules chosen for the repo |
| 2 | `semgrep scan --config auto` | Current directory (the default target) |
| 3 | `semgrep scan --config p/security-audit .` | Registry ruleset by name |
| 4 | `semgrep scan --config p/owasp-top-ten .` | OWASP-mapped ruleset |
| 5 | `semgrep scan --config p/secrets .` | Secrets ruleset |
| 6 | `semgrep scan --config p/default .` | Default curated set |
| 7 | `semgrep scan --config ./rules/ .` | Local rule directory |
| 8 | `semgrep scan --config rules/xss.yaml .` | One local rule file |
| 9 | `semgrep scan --config https://example.com/rules.yaml .` | Remote rule file |
| 10 | `semgrep scan --config r/python.lang.security.audit.eval-detected .` | Single registry rule |
| 11 | `semgrep scan --pattern '$X == $X' .` | Inline pattern, no rule file |
| 12 | `semgrep scan --lang python --pattern 'eval(...)' .` | Inline pattern with language |
| 13 | `semgrep scan --config auto src/ lib/` | Multiple explicit targets |
| 14 | `semgrep scan --help` | Full scan manual |
| 15 | `semgrep --help` | Command list |

**Research questions**
- Which registry rulesets are maintained versus deprecated in 2026?
- Cost and privacy of `--config auto` (it sends the project URL to the
  registry) versus a pinned local ruleset?

**Sources**: Semgrep Registry, docs on `--config`, privacy/metrics docs.

**Output**: one pinned ruleset per codebase, with an explicit note on what
leaves the host.

---

## Scenario 2 — target selection and scoping

| # | Command | How to use |
|---|---------|------------|
| 16 | `semgrep scan --config p/default src/` | Scan a subtree |
| 17 | `semgrep scan --config p/default --include '*.py' .` | Only matching paths |
| 18 | `semgrep scan --config p/default --exclude 'tests/*' .` | Skip paths |
| 19 | `semgrep scan --config p/default --exclude-rule 'python.lang.*' .` | Drop rules by pattern |
| 20 | `semgrep scan --config p/default --exclude-binary-files .` | Skip binaries |
| 21 | `semgrep scan --config p/default --no-exclude-binary-files .` | Include binaries |
| 22 | `semgrep scan --config p/default --exclude-minified-files .` | Skip minified assets |
| 23 | `semgrep scan --config p/default --no-exclude-minified-files .` | Include minified |
| 24 | `semgrep scan --config p/default --scan-unknown-extensions .` | Treat unknown files as generic text |
| 25 | `semgrep scan --config p/default --skip-unknown-extensions .` | Skip unknown file types |
| 26 | `semgrep scan --config p/default --use-git-ignore .` | Honor `.gitignore` |
| 27 | `semgrep scan --config p/default --no-git-ignore .` | Ignore `.gitignore` |
| 28 | `semgrep scan --config p/default --semgrepignore-v2 .` | New `.semgrepignore` behavior |
| 29 | `semgrep scan --config p/default --max-target-bytes 1000000 .` | Skip huge files |
| 30 | `semgrep scan --config p/default --project-root .` | Fix project root for rule path logic |
| 31 | `semgrep scan --config p/default --baseline-commit HEAD~1 .` | Only new findings since a commit |
| 32 | `semgrep scan --config p/default --novcs .` | No VCS awareness |

**Research questions**
- Precedence between `--include`, `--exclude`, `.gitignore`, and
  `.semgrepignore`?
- Does `--baseline-commit` require a clean working tree, and how does it pick
  new lines?

**Sources**: docs on ignore files, baseline scans, issues on include/exclude.

**Output**: a target scope that matches the engagement boundary exactly.

---

## Scenario 3 — output formats and severities

| # | Command | How to use |
|---|---------|------------|
| 33 | `semgrep scan --config p/default --text .` | Human text (default) |
| 34 | `semgrep scan --config p/default --json .` | JSON to stdout |
| 35 | `semgrep scan --config p/default --json-output out.json .` | JSON to a file |
| 36 | `semgrep scan --config p/default --sarif .` | SARIF to stdout |
| 37 | `semgrep scan --config p/default --sarif-output out.sarif .` | SARIF to a file |
| 38 | `semgrep scan --config p/default --emacs .` | Emacs-friendly lines |
| 39 | `semgrep scan --config p/default --emacs-output out.txt .` | Emacs output file |
| 40 | `semgrep scan --config p/default --vim .` | Vim quickfix lines |
| 41 | `semgrep scan --config p/default --vim-output out.txt .` | Vim output file |
| 42 | `semgrep scan --config p/default --gitlab-sast .` | GitLab SAST report |
| 43 | `semgrep scan --config p/default --gitlab-sast-output gl-sast.json .` | To a file |
| 44 | `semgrep scan --config p/default --gitlab-secrets .` | GitLab secrets report |
| 45 | `semgrep scan --config p/default --gitlab-secrets-output gl-secrets.json .` | To a file |
| 46 | `semgrep scan --config p/default --junit-xml .` | JUnit XML |
| 47 | `semgrep scan --config p/default --junit-xml-output junit.xml .` | To a file |
| 48 | `semgrep scan --config p/default --severity ERROR .` | Only ERROR findings |
| 49 | `semgrep scan --config p/default --severity ERROR,WARNING .` | Multiple severities |
| 50 | `semgrep scan --config p/default --files-with-matches .` | Filenames only |
| 51 | `semgrep scan --config p/default --output out.txt .` | Redirect any format |
| 52 | `semgrep scan --config p/default --quiet .` | Findings only, no progress |
| 53 | `semgrep scan --config p/default --verbose .` | Extra diagnostics |
| 54 | `semgrep scan --config p/default --debug .` | Debug logging |
| 55 | `semgrep scan --config p/default --matching-explanations .` | Explain why each match fired |
| 56 | `semgrep scan --config p/default --dataflow-traces .` | Show taint paths |
| 57 | `semgrep scan --config p/default --time .` | Per-rule timing |
| 58 | `semgrep scan --config p/default --no-time .` | Disable timing |
| 59 | `semgrep scan --config p/default --profile .` | Per-rule profiling |
| 60 | `semgrep scan --config p/default --max-lines-per-finding 5 .` | Limit context lines |
| 61 | `semgrep scan --config p/default --max-chars-per-line 120 .` | Truncate long lines |
| 62 | `semgrep scan --config p/default --max-match-context-size 10 .` | Context cap |
| 63 | `semgrep scan --config p/default --max-log-list-entries 10 .` | Limit list output |
| 64 | `semgrep scan --config p/default --force-color .` | Force ANSI |
| 65 | `semgrep scan --config p/default --no-force-color .` | Disable ANSI |
| 66 | `semgrep scan --config p/default --error .` | Exit non-zero on findings |
| 67 | `semgrep scan --config p/default --no-error .` | Always exit 0 |
| 68 | `semgrep scan --config p/default --dryrun .` | Show what would run |
| 69 | `semgrep scan --config p/default --no-dryrun .` | Explicit real run |

**Research questions**
- Which formats are stable enough for a CI gate and which are experimental?
- Exact exit-code semantics of `--error` with severities?

**Sources**: output-format docs, SARIF spec, GitLab SAST schema, issues.

**Output**: one machine format for CI plus one human format for the report.

---

## Scenario 4 — rule authoring, testing, validation

| # | Command | How to use |
|---|---------|------------|
| 70 | `semgrep scan --validate --config rules/xss.yaml` | Validate rules |
| 71 | `semgrep test rules/` | Test rules against `tests/` annotations |
| 72 | `semgrep test --config rules/xss.yaml rules/tests/` | Explicit config and tests |
| 73 | `semgrep test --json rules/` | JSON test results |
| 74 | `semgrep test --test-ignore-todo rules/` | Skip TODO cases |
| 75 | `semgrep test --no-test-ignore-todo rules/` | Fail on TODOs |
| 76 | `semgrep validate rules/` | Experimental validator |
| 77 | `semgrep validate --pro rules/` | Validate Pro-only rules |
| 78 | `semgrep scan --config rules/x.yaml --dump-ast .` | Dump ASTs while scanning |
| 79 | `semgrep scan --config rules/x.yaml --dump-command-for-core .` | Show the engine command |
| 80 | `semgrep scan --config rules/x.yaml --dump-engine-path` | Print engine path |
| 81 | `semgrep show dump-pattern python 'eval(...)'` | AST of a pattern |
| 82 | `semgrep show dump-ast python file.py` | AST of a file |
| 83 | `semgrep show dump-cst python file.py` | Concrete syntax tree |
| 84 | `semgrep show dump-config rules/x.yaml` | Internal config representation |
| 85 | `semgrep show dump-targets . rules/x.yaml` | Files that would be scanned |
| 86 | `semgrep show debug --config rules/x.yaml .` | Interactive rule debugging |
| 87 | `semgrep publish rules/x.yaml --visibility unlisted` | Upload a rule |
| 88 | `semgrep publish rules/x.yaml --visibility public --registry-id my.rule` | Publish to the registry |
| 89 | `semgrep scan --config p/default --strict .` | Fail on rule parse errors |
| 90 | `semgrep scan --config p/default --no-strict .` | Tolerate rule errors |
| 91 | `semgrep scan --config p/default --develop .` | Development mode |
| 92 | `semgrep scan --config p/default --legacy .` | Legacy engine behavior |
| 93 | `semgrep scan --config p/default --optimizations all .` | Enable all optimizations |
| 94 | `semgrep scan --config p/default --show-supported-languages` | List languages |

**Research questions**
- Required annotation format for `semgrep test` (`ruleid`, `ok`, `todoruleid`)?
- Which AST dumps are stable enough to write rules against across versions?

**Sources**: rule syntax docs, testing-rules docs, CLI reference, source.

**Output**: a tested, versioned rule file plus its regression corpus.

---

## Scenario 5 — secrets and supply chain

| # | Command | How to use |
|---|---------|------------|
| 95 | `semgrep scan --config p/secrets --secrets .` | Secret scanning |
| 96 | `semgrep scan --config p/default --secrets --secrets-timeout 30 .` | Secret scan with timeout |
| 97 | `semgrep scan --config p/default --historical-secrets .` | Scan git history |
| 98 | `semgrep scan --config p/default --no-secrets-validation .` | Do not validate found secrets |
| 99 | `semgrep ci --supply-chain` | CI supply-chain scan |
| 100 | `semgrep ci --code` | CI code scan |
| 101 | `semgrep ci --secrets` | CI secrets scan |
| 102 | `semgrep ci --audit-on` | Audit mode |
| 103 | `semgrep ci --baseline-commit HEAD~1` | Only new findings |
| 104 | `semgrep ci --dry-run` | Show what CI would do |
| 105 | `semgrep ci --subdir src/` | Scan a subdirectory in CI |
| 106 | `semgrep ci --oss-only` | OSS engine only |
| 107 | `semgrep ci --suppress-errors` | Do not fail on scan errors |
| 108 | `semgrep ci --no-suppress-errors` | Fail on errors |
| 109 | `semgrep ci --code --secrets --supply-chain` | All CI scans |

**Research questions**
- How secret validation avoids sending the secret itself to a third party?
- Which `semgrep ci` features require a Semgrep App account?

**Sources**: secrets docs, supply-chain docs, CI docs, privacy notes.

**Output**: a CI scan profile with an explicit data-handling note.

---

## Scenario 6 — performance and engine limits

| # | Command | How to use |
|---|---------|------------|
| 110 | `semgrep scan --config p/default --jobs 8 .` | Parallel jobs |
| 111 | `semgrep scan --config p/default --timeout 30 .` | Per-rule timeout |
| 112 | `semgrep scan --config p/default --timeout-threshold 3 .` | Timeout escalation |
| 113 | `semgrep scan --config p/default --max-memory 4096 .` | Memory cap |
| 114 | `semgrep scan --config p/default --interfile-timeout 60 .` | Cross-file analysis budget |
| 115 | `semgrep scan --config p/default --max-target-bytes 500000 .` | File size cap |
| 116 | `semgrep scan --config p/default --incremental-output .` | Emit incrementally |
| 117 | `semgrep scan --config p/default --disable-version-check .` | No update check |
| 118 | `semgrep scan --config p/default --enable-version-check .` | Update check on |
| 119 | `semgrep scan --config p/default --metrics off .` | Disable telemetry |
| 120 | `semgrep scan --config p/default --trace .` | Trace events |
| 121 | `semgrep scan --config p/default --no-trace .` | No trace |
| 122 | `semgrep scan --config p/default --trace-endpoint URL .` | Custom trace endpoint |
| 123 | `semgrep scan --config p/default --x-simple-profiling .` | Simple profiling data |
| 124 | `semgrep scan --config p/default --x-mem-policy` | Memory policy tuning |
| 125 | `semgrep scan --config p/default --x-run-taint-once .` | Taint optimization |
| 126 | `semgrep scan --config p/default --no-x-run-taint-once .` | Disable it |
| 127 | `semgrep scan --config p/default --x-parmap .` | Legacy parallel backend |
| 128 | `semgrep scan --config p/default --x-eio .` | Experimental Eio backend |

**Research questions**
- Which limit actually stops a runaway scan — per-rule timeout or total time?
- Does `--metrics off` fully disable network calls, or only usage metrics?

**Sources**: performance docs, metrics docs, issues on OOM/timeouts.

**Output**: a scan budget that finishes and is reproducible.

---

## Scenario 7 — Pro engine and cross-file analysis

| # | Command | How to use |
|---|---------|------------|
| 129 | `semgrep login` | Store a semgrep.dev token |
| 130 | `semgrep login --force` | Re-authenticate from scratch |
| 131 | `semgrep login --override-tty` | Non-interactive login |
| 132 | `semgrep logout` | Remove stored credentials |
| 133 | `semgrep install-semgrep-pro` | Install the Pro engine |
| 134 | `semgrep install-semgrep-pro --custom-binary /path/pro` | Custom Pro binary |
| 135 | `semgrep scan --config p/default --pro .` | Pro engine |
| 136 | `semgrep scan --config p/default --oss-only .` | Force OSS engine |
| 137 | `semgrep scan --config p/default --pro-intrafile .` | Pro intra-file only |
| 138 | `semgrep scan --config p/default --pro-languages .` | List Pro languages |
| 139 | `semgrep scan --config p/default --pro-path-sensitive .` | Path-sensitive analysis |
| 140 | `semgrep scan --config p/default --x-enable-transitive-reachability .` | Reachability analysis |
| 141 | `semgrep scan --config p/default --x-tr .` | Alias of the above |
| 142 | `semgrep scan --config p/default --x-disable-transitive-reachability .` | Disable reachability |
| 143 | `semgrep scan --config p/default --x-dependency-paths Cargo.toml .` | Explicit dependency graph |
| 144 | `semgrep scan --config p/default --x-group-taint-rules .` | Group taint rules |
| 145 | `semgrep show identity` | Print logged-in identity |
| 146 | `semgrep show deployment` | Print current deployment |
| 147 | `semgrep show identity --json` | Machine-readable identity |
| 148 | `semgrep show dump-lockfile package-lock.json` | Extracted dependencies |
| 149 | `semgrep show project-prefilter file.py` | Project-level prefilters |
| 150 | `semgrep show file-prefilter file.py` | File-level prefilters |

**Research questions**
- Exactly which analyses are Pro-only, and what degrades silently on OSS?
- What data does the Pro engine request from semgrep.dev at runtime?

**Sources**: Pro engine docs, pricing/feature matrix, source, privacy docs.

**Output**: a documented engine choice, with Pro-only findings marked.

---

## Scenario 8 — fix, ignore, and suppression

| # | Command | How to use |
|---|---------|------------|
| 151 | `semgrep scan --config p/default --autofix .` | Apply fixes (VCS required) |
| 152 | `semgrep scan --config p/default --no-autofix .` | Explicit no-fix |
| 153 | `semgrep scan --config p/default --replacement 'safe_call(...)' .` | Inline replacement for `--pattern` |
| 154 | `semgrep scan --config p/default --rewrite-rule-ids .` | Rewrite IDs to new paths |
| 155 | `semgrep scan --config p/default --no-rewrite-rule-ids .` | Keep IDs |
| 156 | `semgrep scan --config p/default --enable-nosem .` | Honor `nosem` annotations |
| 157 | `semgrep scan --config p/default --disable-nosem .` | Ignore `nosem` |
| 158 | `semgrep scan --config p/default --x-ignore-semgrepignore-files .` | Ignore `.semgrepignore` |
| 159 | `semgrep scan --config p/default --x-semgrepignore-filename custom.ignore .` | Custom ignore file |
| 160 | `semgrep scan --config p/default --allow-local-builds .` | Permit dependency builds |
| 161 | `semgrep scan --config p/default --allow-untrusted-validators .` | Allow non-semgrep.dev validators |
| 162 | `semgrep scan --config p/default --x-rule-validation full .` | Rule validation level |
| 163 | `semgrep scan --config p/default --x-no-python-schema-validation .` | Skip Python schema check |

**Research questions**
- Autofix data-loss modes and whether it ever touches files outside findings?
- How `nosem` comments are scoped and whether they are auditable in CI.

**Sources**: autofix docs, ignore docs, issues on autofix, security advisories.

**Output**: a fix policy with VCS and review requirements stated up front.

---

## Scenario 9 — editor, LSP, and MCP integration

| # | Command | How to use |
|---|---------|------------|
| 164 | `semgrep lsp` | Language server on stdio |
| 165 | `semgrep lsp --x-eio-ls` | Experimental Eio LSP |
| 166 | `semgrep mcp` | MCP server on stdio |
| 167 | `semgrep mcp --transport streamable-http --port 8000` | HTTP MCP server |
| 168 | `semgrep mcp --hook post-tool-cli-scan` | Agent post-tool hook scan |
| 169 | `semgrep mcp --hook stop-cli-scan` | Agent stop hook scan |
| 170 | `semgrep mcp --hook record-file-edit` | Record edited files |
| 171 | `semgrep mcp --hook inject-secure-defaults` | Inject secure defaults |
| 172 | `semgrep mcp --hook supply-chain-scan` | Supply-chain hook |
| 173 | `semgrep show debug --config p/default .` | Interactive debugger UI |
| 174 | `semgrep scan --config p/default --x-ls .` | Language-server mode scan |
| 175 | `semgrep scan --config p/default --x-ls-long .` | Long-form LS output |
| 176 | `semgrep scan --config p/default --x-mcp .` | Start MCP through scan |

**Research questions**
- What does the MCP server expose, and what are its trust boundaries?
- Does the LSP send code anywhere, or analyze locally?

**Sources**: LSP docs, MCP docs, source, security notes.

**Output**: an editor/agent integration that keeps code local unless chosen
otherwise.

---

## Complete flag coverage

`scan` flags are listed here grouped; every long flag from
[`help-scan-full.txt`](help-scan-full.txt) appears at least once. `--x-*`
flags are experimental and may disappear.

| Group | Flags | Scenario |
|-------|-------|----------|
| Config | `--config`/`-c`, `--pattern`/`-e`, `--lang`, `--replacement`, `--validate`, `--strict`, `--no-strict`, `--develop`, `--legacy` | 1, 4 |
| Targets | `--include`, `--exclude`, `--exclude-rule`, `--exclude-binary-files`, `--no-exclude-binary-files`, `--exclude-minified-files`, `--no-exclude-minified-files`, `--scan-unknown-extensions`, `--skip-unknown-extensions`, `--use-git-ignore`, `--no-git-ignore`, `--semgrepignore-v2`, `--max-target-bytes`, `--project-root`, `--baseline-commit`, `--novcs` | 2 |
| Output | `--text`/`--text-output`, `--json`/`--json-output`, `--sarif`/`--sarif-output`, `--emacs`/`--emacs-output`, `--vim`/`--vim-output`, `--gitlab-sast`/`--gitlab-sast-output`, `--gitlab-secrets`/`--gitlab-secrets-output`, `--junit-xml`/`--junit-xml-output`, `--output`, `--severity`, `--files-with-matches`, `--quiet`, `--verbose`, `--debug`, `--matching-explanations`, `--dataflow-traces`, `--time`, `--no-time`, `--profile`, `--max-lines-per-finding`, `--max-chars-per-line`, `--max-match-context-size`, `--max-log-list-entries`, `--force-color`, `--no-force-color`, `--error`, `--no-error`, `--dryrun`, `--no-dryrun`, `--incremental-output` | 3, 5 |
| Tests | `--test`, `--test-ignore-todo`, `--no-test-ignore-todo`, `--dump-ast`, `--dump-command-for-core`, `--dump-engine-path` | 4 |
| Secrets | `--secrets`, `--secrets-timeout`, `--historical-secrets`, `--no-secrets-validation` | 5 |
| Performance | `--jobs`, `--timeout`, `--timeout-threshold`, `--max-memory`, `--interfile-timeout`, `--disable-version-check`, `--enable-version-check`, `--metrics`, `--trace`, `--no-trace`, `--trace-endpoint`, `--optimizations`, `--show-supported-languages` | 6 |
| Pro | `--pro`, `--oss-only`, `--pro-intrafile`, `--pro-languages`, `--pro-path-sensitive` | 7 |
| Autofix/ignore | `--autofix`/`-a`, `--no-autofix`, `--rewrite-rule-ids`, `--no-rewrite-rule-ids`, `--enable-nosem`, `--disable-nosem`, `--allow-local-builds`, `--allow-untrusted-validators` | 8 |
| Experimental `--x-` | `--x-eio`, `--x-parmap`, `--x-ls`, `--x-ls-long`, `--x-mcp`, `--x-mem-policy`, `--x-rule-validation`, `--x-run-taint-once`, `--no-x-run-taint-once`, `--x-enable-transitive-reachability`, `--x-tr`, `--x-disable-transitive-reachability`, `--x-dependency-paths`, `--x-group-taint-rules`, `--x-gradle-module-attribution`, `--x-dump-symbol-analysis`, `--x-pro-naming`, `--x-simple-profiling`, `--x-ignore-semgrepignore-files`, `--x-semgrepignore-filename`, `--x-no-python-schema-validation` | 4, 6, 7, 9 |
| Misc | `--help`/`-h`, `--version` | 1 |

Subcommands: `scan`, `ci`, `install-semgrep-pro`, `login`, `logout`, `lsp`,
`publish`, `show`, `test`, `validate`, `mcp`.

Full per-flag text: [`help.txt`](help.txt), [`help-scan-full.txt`](help-scan-full.txt).

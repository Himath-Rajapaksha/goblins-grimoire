# Semgrep — static analysis (SAST)

- **Version:** 1.184.0
- **Binary:** `/usr/local/bin/semgrep`
- **Help snapshots:** [`help.txt`](help.txt), [`help-scan.txt`](help-scan.txt)
- **Docs:** https://semgrep.dev/docs/
- **Rule registry:** https://semgrep.dev/explore
- **Source:** https://github.com/semgrep/semgrep

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Semgrep is a fast, polyglot static analyzer. Rules are written as code-like
patterns (not regex), matched against ASTs across 30+ languages. It supports
taint analysis, metavariables, autofix, and CI integration. Use it to find
injection sinks, insecure crypto, hardcoded secrets, dangerous APIs, and
project-specific anti-patterns.

## Use cases

| Situation | Command |
|-----------|---------|
| Quick security scan | `semgrep scan --config auto .` |
| OWASP Top 10 rules | `--config p/owasp-top-ten` |
| Security audit ruleset | `--config p/security-audit` |
| Secrets | `--config p/secrets` |
| CI diff scan | `semgrep ci` |
| Custom rule | `--config my-rule.yaml` |
| One-off pattern | `--pattern 'eval(...)' --lang python` |
| Autofix | `--autofix` |
| SARIF output | `--sarif -o semgrep.sarif` |

## Commands

| Command | Purpose |
|---------|---------|
| `semgrep scan` | Run rules on files/directories |
| `semgrep ci` | Diff-aware CI scan (uses Semgrep AppSec Platform) |
| `semgrep login` / `logout` | Manage semgrep.dev credentials |
| `semgrep test` | Test rules against annotations |
| `semgrep validate` | Validate rule files |
| `semgrep publish` | Publish rules |
| `semgrep lsp` | Language server for IDEs |
| `semgrep mcp` | MCP server |
| `semgrep show` | Inspect supported languages, etc. |
| `semgrep install-semgrep-pro` | Install Pro Engine |

## `semgrep scan` flags (high-value)

### Rules / config
| Flag | Meaning |
|------|---------|
| `--config <ref>` | `auto`, registry pack (`p/security-audit`), file, dir, URL |
| `--config auto` | Community rules matched to the project |
| `--pattern <pat>` | Inline pattern |
| `--pattern-either <p>` | Any of several patterns |
| `--pattern-regex <re>` | Regex pattern |
| `--lang <lang>` | Language for inline patterns |
| `--exclude <glob>` | Skip paths |
| `--include <glob>` | Limit paths |
| `--exclude-rule <id>` | Skip rule IDs |
| `--severity <list>` | `INFO,WARNING,ERROR` |
| `--max-target-bytes <n>` | Skip large files |
| `--scan-unknown-extensions` | Treat unknown files as target |

### Output
| Flag | Meaning |
|------|---------|
| `--json` | JSON output |
| `--sarif` | SARIF output |
| `--junit-xml` | JUnit XML |
| `--emacs` / `--vim` | Editor formats |
| `--text` | Plain text (default) |
| `-o, --output <file>` | Write to file |
| `--quiet` | Findings only |
| `--verbose` | More detail |
| `--debug` | Debug |
| `--error` | Exit non-zero on findings |
| `--metrics <on/off/auto>` | Telemetry control |
| `--time` | Timing info |

### Behavior
| Flag | Meaning |
|------|---------|
| `--autofix` | Apply rule fixes |
| `--dryrun` | Show fixes without writing |
| `--no-rewrite-rule-ids` | Preserve rule IDs |
| `--timeout <sec>` | Per-rule timeout |
| `--max-memory <MB>` | Memory cap |
| `--jobs <n>` | Parallel jobs |
| `--optimizations <all/none>` | Rule optimizations |
| `--pro` / `--pro-intrafile` / `--pro-languages` | Pro Engine (if installed/licensed) |
| `--test` / `--validate` | Rule testing/validation |
| `--allow-unused-patterns` | Suppress unused-pattern warnings |

Run `semgrep scan --help` for the complete list; `-h` is not accepted by
`scan` (use `--help`).

## Rule syntax

```yaml
rules:
  - id: python-eval-user-input
    languages: [python]
    severity: ERROR
    message: >-
      User-controlled data reaches eval(). This can lead to remote code
      execution.
    metadata:
      category: security
      cwe: "CWE-95: Improper Neutralization of Directives in Dynamically Evaluated Code"
      owasp: "A03:2021 - Injection"
    patterns:
      - pattern: eval($X)
      - pattern-not: eval("...")
```

Taint-mode rule:

```yaml
rules:
  - id: flask-sql-injection
    languages: [python]
    severity: ERROR
    message: Untrusted input reaches a raw SQL query.
    mode: taint
    pattern-sources:
      - pattern: request.args.get(...)
      - pattern: request.form.get(...)
    pattern-sinks:
      - pattern: $DB.execute($QUERY, ...)
    pattern-sanitizers:
      - pattern: int(...)
```

## Recipes

```bash
# Quick scan with auto rules
semgrep scan --config auto .

# Security audit pack, fail on findings
semgrep scan --config p/security-audit --error .

# OWASP Top 10
semgrep scan --config p/owasp-top-ten .

# Secrets
semgrep scan --config p/secrets .

# One-off pattern
semgrep scan --pattern 'eval(...)' --lang python .

# Custom rule
semgrep scan --config rules/my-rule.yaml src/

# Autofix
semgrep scan --config rules/my-rule.yaml --autofix src/

# SARIF for GitHub
semgrep scan --config auto --sarif -o semgrep.sarif .

# JSON for tooling
semgrep scan --config auto --json -o semgrep.json .

# Exclude tests and vendored code
semgrep scan --config auto --exclude 'tests/**' --exclude 'vendor/**' .

# Validate a rule file
semgrep validate rules/my-rule.yaml

# Test rules with annotations
semgrep test rules/
```

## CI usage

```yaml
# .github/workflows/semgrep.yml
name: semgrep
on: [push, pull_request]
jobs:
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: semgrep/semgrep-action@v1
        with:
          config: p/security-audit
```

Or directly:

```bash
semgrep ci          # diff-aware, requires login/token
semgrep scan --config p/ci --error .
```

## Gotchas

- `--config auto` sends telemetry; disable with `--metrics off` and pin a pack.
- Semgrep is a **pattern/AST** tool, not a full dataflow engine in OSS mode;
  taint works but Pro adds cross-file analysis.
- Findings need triage — pattern matches can be safe in context.
- Autofix can break code; review diffs and use `--dryrun` first.
- Large monorepos: use `--jobs`, `--exclude`, and `--max-target-bytes`.
- Rules vary in quality; start with `p/security-audit` and curated packs.
- `semgrep ci` is platform-oriented; for standalone use `semgrep scan`.

## References

- Docs: https://semgrep.dev/docs/
- Rule syntax: https://semgrep.dev/docs/writing-rules/overview/
- Registry: https://semgrep.dev/explore
- Playground: https://semgrep.dev/playground
- Source: https://github.com/semgrep/semgrep

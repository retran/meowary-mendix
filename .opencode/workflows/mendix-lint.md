---
updated: 2026-04-29
tags: [mendix]
---

<role>
Mendix quality auditor. Run lint rules and best-practices report; surface naming violations, security gaps, architecture issues, and performance anti-patterns.
</role>

<summary>
Lint the Mendix project using `./mxcli lint` and generate scored reports with `./mxcli report`. Covers 30+ built-in and Starlark rules across security, quality, architecture, performance, naming, and design categories. Read-only — never modifies the project.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
| Modules to exclude | User preference | Optional |
</inputs>

<steps>

<step n="1" name="Run lint">

```bash
# Default: human-readable, grouped by module
./mxcli lint -p <project>.mpr --color

# Exclude system/generated modules
./mxcli lint -p <project>.mpr --color --exclude System --exclude Administration

# JSON output (for programmatic use)
./mxcli lint -p <project>.mpr --format json

# SARIF output (for CI / GitHub code scanning)
./mxcli lint -p <project>.mpr --format sarif > results.sarif

# List all available rules
./mxcli lint -p <project>.mpr --list-rules
```

Exit code is 1 if any **errors** are found (warnings do not fail).
<done_when>Lint output received.</done_when>
</step>

<step n="2" name="Run best-practices report">
Run after lint to get scored category breakdown. Always run both — lint gives per-issue detail, report gives the aggregate score.
```bash
# Markdown report (default) — scored 0–100 per category
./mxcli report -p <project>.mpr

# HTML report
./mxcli report -p <project>.mpr --format html --output report.html

# JSON
./mxcli report -p <project>.mpr --format json
```

Categories scored: Security, Quality, Architecture, Performance, Naming, Design (0–100 each).
<done_when>Report generated.</done_when>
</step>

<step n="3" name="Report findings">

**Built-in rules reference:**

| Rule | Category | What it checks |
|------|----------|---------------|
| MPR001 | naming | PascalCase + microflow prefixes (ACT_, SUB_, DS_, VAL_, SCH_, IVK_, BCO_, ACO_…) |
| MPR002 | quality | Empty microflows |
| MPR003 | design | Max persistent entities per domain model |
| MPR004 | quality | Validation feedback with empty message (CE0091) |
| MPR005 | quality | IMAGE widgets with no source |
| MPR006 | quality | Empty layout containers |
| MPR007 | security | Navigation pages need allowed roles (CE0557) |
| SEC001 | security | Persistent entities must have access rules |
| SEC002 | security | Password policy minimum length ≥ 8 |
| SEC003 | security | Demo users off at Production security |
| CONV011 | performance | Commit inside loop (N+1) |
| CONV012 | quality | Exclusive splits need meaningful captions |
| CONV013 | quality | External calls (REST/WS/Java) need error handling |
| CONV014 | quality | Silent error swallow (Continue) |
| ARCH001 | architecture | Cross-module data access (pages using other-module entities) |
| ARCH002 | architecture | Data changes should go through microflows |
| ARCH003 | architecture | Persistent entities need a unique business key |
| QUAL001 | quality | Cyclomatic complexity threshold |
| QUAL002 | quality | Missing documentation on entities/microflows |
| QUAL003 | quality | Microflows with >25 activities |
| QUAL004 | quality | Orphaned (unreferenced) elements |
| CONV001 | naming | Boolean attributes start with Is/Has/Can/Should/Was/Will |
| CONV002 | naming | No entity attribute defaults (use microflows) |
| CONV003 | naming | Pages end with _NewEdit/_View/_Overview/etc. |
| CONV004 | naming | Enumerations start with ENUM_ |
| CONV005 | naming | Snippets start with SNIPPET_ |
| CONV006 | security | No create/delete rights on entities (use microflows) |
| CONV007 | security | Entity access should have XPath constraints |
| CONV009 | quality | Microflows should have ≤ 15 activities |
| CONV010 | architecture | ACT_ microflows should only contain UI actions |
| CONV015 | quality | Use VAL_ microflows instead of entity validation rules |
| CONV016 | performance | No event handlers (use explicit microflow calls) |
| CONV017 | performance | No calculated attributes (use stored attributes updated by microflows) |

Custom Starlark rules in `.claude/lint-rules/*.star` (project repo path) are loaded automatically by `./mxcli lint`. Reference copies live in `.opencode/skills/mendix/write-lint-rules/examples/`. See `write-lint-rules` sub-skill for the full Starlark rule API.

Present findings grouped by severity (error → warning → info), then by module. For scored reports, show the category score table first, then the top findings.
<done_when>Findings presented; errors flagged as blockers.</done_when>
</step>

</steps>

<error_handling>
- **App not running:** Start with `./mxcli docker run -p <project>.mpr --wait`.
- **Linter builds catalog automatically** — no manual REFRESH CATALOG needed before lint.
- **Exit code 1 / CI failure:** Only lint *errors* cause exit 1; warnings do not.
</error_handling>

<contracts>
1. NEVER modify the project during lint — read-only.
2. Lint errors are blockers; warnings are advisory.
3. Always suggest `--exclude System --exclude Administration` to reduce noise from generated modules.
</contracts>

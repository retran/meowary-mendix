---
updated: 2026-04-29
tags: [mendix]
---

<role>
Git diff tool for Mendix. Compare uncommitted local changes in mxunit files against a git reference to see what Studio Pro (or mxcli) has modified.
</role>

<summary>
Compare local uncommitted changes in a Mendix MPR v2 project against a git reference using `./mxcli diff-local`. Finds changed `.mxunit` files, parses BSON, converts to MDL, and shows the diff. Requires Mendix 10.18+ (MPR v2 format with `mprcontents/` folder under git control.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| Git reference | User-provided (default: HEAD) | Optional |
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
</inputs>

<steps>

<step n="1" name="Verify prerequisites">
- MPR v2 format: Mendix 10.18+ — project stores units in `mprcontents/` folder
- Project is under git version control
- `mprcontents/` folder is tracked by git

If prerequisites not met: report to user and stop.
<done_when>Prerequisites confirmed.</done_when>
</step>

<step n="2" name="Run diff-local">

```bash
# Uncommitted changes vs HEAD (default)
./mxcli diff-local -p <project>.mpr

# Against specific commit
./mxcli diff-local -p <project>.mpr --ref HEAD~1

# Against a branch
./mxcli diff-local -p <project>.mpr --ref main

# Against a tag
./mxcli diff-local -p <project>.mpr --ref v1.0.0

# Structural summary
./mxcli diff-local -p <project>.mpr --format struct

# Side-by-side
./mxcli diff-local -p <project>.mpr --format side

# With color
./mxcli diff-local -p <project>.mpr --color
```

**How it works internally:**
1. `git diff --name-status` finds modified `.mxunit` files in `mprcontents/`
2. Reads current version from disk, git version via `git show`
3. Parses BSON from both versions
4. Converts to MDL for human-readable comparison
5. Shows diff in selected format

**Supported unit types:** entities, microflows, nanoflows, enumerations, pages, snippets, layouts, modules. Other types show a generic representation.
<done_when>Diff output received.</done_when>
</step>

<step n="3" name="Present output">
Always ends with: `Summary: N new, N modified, N deleted`

Present structural summary first, then detail for changed elements.

**Output format guide:**
- `--format struct` — best for quick overview of what changed
- `--format side` — best for large objects with subtle differences
- unified (default) — best for reviewing individual changes with +/- context

NOTE: In VS Code terminals, qualified names are **clickable links**.

**Common use cases:**
- Review changes before committing to version control
- Understand what Studio Pro modified when saving
- Audit changes between versions
- Debug issues by comparing working vs previous state
<done_when>Findings communicated.</done_when>
</step>

</steps>

<error_handling>
- **"Not MPR v2":** Requires Mendix 10.18+. If older version, this command is not available.
- **"No git repository":** Project must be under git version control.
- **"`mprcontents/` not tracked":** The folder must be committed to git for comparison to work.
- **App not running:** Start with `./mxcli docker run -p <project>.mpr --wait`.
</error_handling>

<contracts>
1. NEVER modify the project — read-only comparison.
2. Always show summary line at end of output.
3. Recommend `--format struct` as the first pass; offer detail formats on request.
</contracts>

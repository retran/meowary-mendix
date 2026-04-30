---
updated: 2026-04-29
tags: [mendix]
---

<role>
Mendix consistency checker. Run the full Mendix project validation (equivalent to Studio Pro's consistency checks) and report all CE errors and warnings.
</role>

<summary>
Validate the Mendix project using `mx check` (via `./mxcli docker check`). This is the same validation Studio Pro performs — checks CE (consistency errors), CW (warnings), and CD (deprecations). Run after significant MDL changes, before committing, or when Studio Pro reports errors. Distinct from `mendix-check-script` which validates MDL syntax only.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
</inputs>

<steps>

<step n="1" name="Verify app is running">
If not running: `./mxcli docker run -p <project>.mpr --wait`
<done_when>App running at localhost:8080.</done_when>
</step>

<step n="2" name="Run validation">

```bash
# Integrated command (auto-downloads mxbuild if not cached)
./mxcli docker check -p <project>.mpr

# Manual — after mxcli setup mxbuild
~/.mxcli/mxbuild/*/modeler/mx check <project>.mpr
```

If mxbuild is not installed, auto-download it first:
```bash
./mxcli setup mxbuild -p <project>.mpr
```

**Where mxbuild lives after setup:**

| Environment | Path |
|-------------|------|
| Dev container | `~/.mxcli/mxbuild/{version}/modeler/mx` |
| Studio Pro install | `mx` (in PATH) |

<done_when>Validation output received.</done_when>
</step>

<step n="3" name="Report results">

**Error code reference:**

| Code | Type | Action required |
|------|------|----------------|
| CE0xxx | Consistency error | **Blocker** — must fix before project can run |
| CW0xxx | Warning | Review; some can be deferred |
| CD0xxx | Deprecation | Review; plan migration |

**Example output:**
```
MyFirstModule.Customer_Overview (Page):
  CE0123: The entity 'MyFirstModule.Customer' does not exist.

MyFirstModule.SaveCustomer (Microflow):
  CW0456: Variable '$customer' is declared but never used.

Found 1 error(s) and 1 warning(s)
```

Present CE errors as explicit blockers. Group CW/CD by module. State the total count (errors / warnings / deprecations).
<done_when>Results reported; CE errors flagged as blockers.</done_when>
</step>

</steps>

<error_handling>
- **mxbuild not installed:** Run `./mxcli setup mxbuild -p <project>.mpr` to auto-download.
- **App not running:** Start with `./mxcli docker run -p <project>.mpr --wait`.
- **"Project cannot be loaded": Check that `<project>.mpr` path is correct and the file is not locked.
</error_handling>

<contracts>
1. CE errors are blockers — always flag them explicitly, never downplay.
2. `mx check` validates the full project; `./mxcli check script.mdl` validates MDL syntax — they are complementary.
3. Run after every significant MDL change set before committing to version control.
</contracts>

<next_steps>
| Condition | Action |
|-----------|--------|
| CE errors found | Fix with `mendix-write`, then re-run |
| All clear | Safe to commit or continue |
</next_steps>

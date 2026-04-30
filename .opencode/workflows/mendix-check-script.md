---
updated: 2026-04-29
tags: [mendix]
---

<role>
MDL syntax validator. Check a script for grammar errors, microflow body issues, and project reference errors before any execution.
</role>

<summary>
Validate an MDL script using `./mxcli check`. Always runs two passes: syntax + microflow body validation (no project required), then reference validation (requires project). Never execute a script that has not passed both checks.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| MDL script file path | User-provided | Required |
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required for reference check |
</inputs>

<steps>

<step n="1" name="Run syntax check">

```bash
# Syntax + microflow body validation (no project needed)
./mxcli check script.mdl
```

**What this checks:**
- MDL grammar correctness
- Proper statement structure
- Valid keywords and types
- RETURN consistency:
  - RETURN must provide a value when microflow declares a return type
  - RETURN must not provide a value on void microflows (except `RETURN empty`)
  - Scalar literals cannot be returned from entity-typed microflows
  - All code paths must end with RETURN for non-void microflows
- Variables declared inside IF/ELSE branches or ON ERROR bodies cannot be used outside that branch
- VALIDATION FEEDBACK must have a non-empty message (CE0091)

<done_when>Syntax check passes with zero errors.</done_when>
</step>

<step n="2" name="Run reference check">

```bash
# Syntax + reference validation (checks entities/modules/associations exist)
./mxcli check script.mdl -p <project>.mpr --references
```

**What this adds:**
- Module references exist
- Entity references exist
- Association references exist
- Variables declared before use (via SET, DECLARE, CREATE, RETRIEVE, CALL) — catches undeclared variable errors
- Validates all branches (IF/ELSE) and loops
- NOTE: skips references to objects created in the same script (intentional)

<done_when>Reference check passes with zero errors.</done_when>
</step>

<step n="3" name="Report results">

**On success:**
```
✓ Syntax OK (N statements)
✓ References OK
```

**On failure — example output:**
```
✗ Line 5: missing ';' at 'CREATE'
✗ Line 12: unknown type 'Strin'

statement 1 (Module.MyMicroflow): RETURN requires a value because microflow returns Boolean
statement 2 (Module.OtherFlow): microflow returns String but not all code paths have a RETURN statement
statement 3 (Module.ScopeIssue): variable '$X' is declared inside IF branch but used outside

statement 1: microflow 'Module.Name' has validation errors:
  - variable 'IsValid' is not declared. Use DECLARE IsValid: <Type> before using SET
```

Group errors by type: syntax errors first, then microflow body errors, then reference errors.

If this check is part of the `mendix-write` workflow, fix errors silently and re-run until both passes succeed. Only surface errors that require the user to clarify their intent.
<done_when>Results reported; script is either approved for execution or errors are being fixed.</done_when>
</step>

</steps>

<error_handling>
- **App not running (reference check):** Start with `./mxcli docker run -p <project>.mpr --wait`.
- **Script file not found:** Ask user for correct path.
- **Reference errors on same-script objects:** Expected — the checker intentionally skips intra-script references.
</error_handling>

<contracts>
1. ALWAYS run both syntax check AND reference check before any execution.
2. NEVER execute a script with validation errors.
3. Fix errors silently when operating within `mendix-write`; only surface clarification-needed errors to user.
</contracts>

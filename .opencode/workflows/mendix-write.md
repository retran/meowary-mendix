---
updated: 2026-04-29
tags: [mendix]
---

<role>
Mendix project builder. Translate user intent into MDL, validate it, get approval, execute it. NEVER write or execute MDL without explicit user approval. NEVER show raw MDL in chat before approval.
</role>

<summary>
Create or modify Mendix entities, microflows, pages, CRUD scaffolding, security rules, navigation, and other project elements via MDL scripts. All writes follow a strict describe → approve → write → validate → diff → execute → report cycle. Always work against the Docker-running app via `./mxcli`.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| What to create/modify | User invocation | Required |
| Codebase context | `codebases/<project>.md` | Required |
| Relevant mendix sub-skill | `.opencode/skills/mendix/<sub-skill>/SKILL.md` | Required (read before writing MDL) |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
</inputs>

<steps>

<step n="1" name="Load context">
1. Read `codebases/<project>.md` — confirm project path and module conventions.
2. Verify app is running. If not: `./mxcli docker run -p <project>.mpr --wait`.
3. Identify the relevant sub-skill (see routing table in `mendix/SKILL.md`).
4. Read that sub-skill's `SKILL.md` BEFORE writing any MDL.
<done_when>Context loaded; sub-skill read; app running.</done_when>
</step>

<step n="2" name="Gather specification">
Ask for all missing details:
- Module name
- Element name(s)
- Attributes (name, type, constraints, nullability)
- Associations (type: 1-1, 1-many, many-many; owner; delete behavior)
- Return types for microflows
- Page layout preferences
- Security / access rule requirements

DO NOT proceed to writing until specification is complete.
<done_when>Full specification gathered; no open questions.</done_when>
</step>

<step n="3" name="Describe plan" gate="HARD-GATE">
Describe what will be created in plain language — NOT raw MDL. Include:
- Elements to be created (entities, microflows, pages, etc.)
- Key attributes, parameters, or widgets
- Any existing elements that will be modified

Example:
> I will create a persistent entity `Sales.Product` with attributes: Name (String 100, required), Price (Decimal, required), StockQuantity (Integer). Then create `Sales.Product_Overview` page (Atlas_Default layout) with a DataGrid and New/Edit/Delete buttons, and `Sales.Product_NewEdit` page (PopupLayout) with a DataView form.

HARD-GATE: Wait for explicit user approval before proceeding.
<done_when>Plan described; user approved.</done_when>
</step>

<step n="4" name="Write MDL" gate="HARD-GATE">
Write the MDL script to a file silently. All non-negotiable syntax rules from the `mendix` skill apply (see `<rules>` block). Key conventions for this workflow:

**Entity conventions:**
- Position entities with `@Position(x, y)` annotation
- Persistent entities need access rules (see `manage-security` sub-skill)
- Boolean attributes must start with Is/Has/Can/Should/Was/Will
- No attribute defaults — use microflows (CONV002)

**Page conventions:**
- Overview pages: `EntityName_Overview`, layout `Atlas_Default`
- Edit/new pages: `EntityName_NewEdit`, layout `PopupLayout`
- View pages: `EntityName_View`

**Microflow conventions:**
- Naming prefixes: ACT_ (UI actions), SUB_ (sub-flows), DS_ (data source), VAL_ (validation), SCH_ (scheduled), IVK_ (invokable)
- All code paths must end with RETURN for non-void microflows
- External calls (REST/WS/Java) must have `ON ERROR` handling
<done_when>MDL file written to disk.</done_when>
</step>

<step n="5" name="Validate MDL">
Run both checks — handle errors silently:

```bash
# Pass 1: syntax + microflow body
./mxcli check script.mdl

# Pass 2: reference validation
./mxcli check script.mdl -p <project>.mpr --references
```

Fix errors in the script and re-run until both pass with zero errors. Surface to user only if clarification is needed.
<done_when>Both checks pass with zero errors.</done_when>
</step>

<step n="6" name="Preview diff" gate="SOFT-GATE">

```bash
./mxcli diff -p <project>.mpr script.mdl --format struct --color
```

Present structural summary. SOFT-GATE: if diff reveals unexpected modifications, pause and confirm with user.
<done_when>Diff shown; no unexpected changes (or user confirmed).</done_when>
</step>

<step n="7" name="Execute">

```bash
./mxcli exec script.mdl -p <project>.mpr
```

Verify success by re-running diff — expect no remaining changes:

```bash
./mxcli diff -p <project>.mpr script.mdl --format struct
```
<done_when>Script executed; post-diff shows no remaining changes.</done_when>
</step>

<step n="8" name="Report">
Report in plain language: elements created/modified (fully qualified names), any follow-up needed (e.g., add navigation items, restart app).
<done_when>Results reported.</done_when>
</step>

</steps>

<error_handling>
- **App not running:** Start with `./mxcli docker run -p <project>.mpr --wait`. Do not write or execute.
- **Validation errors:** Fix silently; only surface clarification-needed errors to user.
- **Execution fails:** Do NOT retry blindly. Diff current state, report to user.
- **User changes scope after approval:** Re-run describe step with updated plan; get approval again.
</error_handling>

<contracts>
1. NEVER show raw MDL in chat.
2. NEVER execute without passing both validation checks.
3. NEVER execute without prior explicit user approval of the plan (HARD-GATE step 3).
4. ALWAYS read the relevant sub-skill SKILL.md before writing MDL.
5. ALWAYS use `./mxcli` (never bare `mxcli`).
6. Always quote identifiers with double quotes.
7. EXTENDS before opening parenthesis.
8. No CASE/WHEN, no TRY/CATCH.
</contracts>

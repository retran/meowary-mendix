---
updated: 2026-04-29
tags: [mendix]
---

<role>
Mendix CRUD scaffolder. Generate a complete entity + overview page + NewEdit page in one workflow.
</role>

<summary>
Scaffold a full CRUD setup for a Mendix entity: persistent entity with attributes, overview page (DataGrid with New/Edit/Delete), and NewEdit page (form). Follows the same MDL write protocol as `mendix-write` — describe → approve → write → validate → execute.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| Module name | User | Required |
| Entity name and attributes | User | Required |
| Pages to generate | User (default: overview + NewEdit) | Optional |
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
</inputs>

<steps>

<step n="1" name="Load sub-skills">
Read before writing any MDL:
- `.opencode/skills/mendix/generate-domain-model/SKILL.md`
- `.opencode/skills/mendix/patterns-crud/SKILL.md`
- `.opencode/skills/mendix/overview-pages/SKILL.md`
<done_when>Sub-skills loaded.</done_when>
</step>

<step n="2" name="Gather specification">
Collect from user:
- Module name (existing or new)
- Entity name
- Attributes: name, type (String, Integer, Decimal, Boolean, DateTime, Enum, Association), constraints (NOT NULL, length), any defaults to avoid per CONV002
- Associations to other entities (if any)
- Which pages: overview only / NewEdit only / both (default: both)
- Navigation item needed? (yes/no)

Attribute types supported: `String(N)`, `Integer`, `Long`, `Decimal`, `Boolean`, `DateTime`, `AutoNumber`, `Binary`, enumeration reference, association reference.
<done_when>Full specification confirmed.</done_when>
</step>

<step n="3" name="Describe plan" gate="HARD-GATE">
Describe in plain language what will be created:

**What gets created:**
1. **Entity** — persistent entity `Module.EntityName` with listed attributes
2. **Overview page** — `Module.EntityName_Overview` (Atlas_Default layout) — DataGrid with all records, New/Edit/Delete buttons
3. **NewEdit page** — `Module.EntityName_NewEdit` (PopupLayout) — DataView form with Save/Cancel buttons
4. **Navigation snippet** (if requested)

Naming conventions:
- Pages: `EntityName_Overview`, `EntityName_NewEdit`
- Layouts: `Atlas_Default` for overview, `PopupLayout` for NewEdit

HARD-GATE: Wait for explicit approval.
<done_when>Plan described; user approved.</done_when>
</step>

<step n="4" name="Write, validate, execute">
Follow `mendix-write` steps 4–8:
1. Write MDL silently (entity first, then pages)
2. Run `./mxcli check script.mdl` and `./mxcli check script.mdl -p <project>.mpr --references`
3. Show structural diff: `./mxcli diff -p <project>.mpr script.mdl --format struct`
4. Execute: `./mxcli exec script.mdl -p <project>.mpr`
5. Verify: re-run diff to confirm no remaining changes
<done_when>CRUD scaffold created; post-diff clean.</done_when>
</step>

<step n="5" name="Report">
Report:
- Entity created (fully qualified name, attribute count)
- Pages created (fully qualified names)
- Remind user to add navigation item if not included in the script
<done_when>Results reported.</done_when>
</step>

</steps>

<contracts>
1. NEVER show raw MDL in chat.
2. NEVER execute without both validation checks passing.
3. NEVER execute without explicit user approval of the plan.
4. Always follow Atlas_Default / PopupLayout conventions.
5. No attribute defaults on the entity — use microflows (CONV002).
</contracts>

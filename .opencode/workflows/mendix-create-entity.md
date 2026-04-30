---
updated: 2026-04-29
tags: [mendix]
---

<role>
Mendix entity creator. Design and create new persistent entities with attributes, associations, and access rules via MDL.
</role>

<summary>
Create a new entity (or add attributes/associations to an existing entity) in the Mendix domain model using MDL. Follows the standard describe → approve → write → validate → execute cycle. Always reads `generate-domain-model` sub-skill before writing.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| Module name | User | Required |
| Entity name | User | Required |
| Attributes and types | User | Required |
| Associations to other entities | User | Optional |
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
</inputs>

<steps>

<step n="1" name="Load sub-skills">
Read before writing any MDL:
- `.opencode/skills/mendix/generate-domain-model/SKILL.md`
- `.opencode/skills/mendix/mdl-entities/SKILL.md`
<done_when>Sub-skills loaded.</done_when>
</step>

<step n="2" name="Gather specification">
Collect from user:
- Module name (use existing module or specify new)
- Entity name (PascalCase)
- Attributes: name, type, constraints (NOT NULL, length), nullability
- Associations: target entity, type (1-1, 1-many, many-many), owner, delete behavior
- Entity position on domain model canvas (optional — defaults to auto)

**Supported attribute types:** `String(N)`, `Integer`, `Long`, `Decimal`, `Boolean`, `DateTime`, `AutoNumber`, `Binary`, enumeration reference.

**Naming conventions:**
- Entity names: PascalCase
- Boolean attributes: must start with Is/Has/Can/Should/Was/Will (CONV001)
- No attribute defaults — use microflows (CONV002)
<done_when>Full specification confirmed.</done_when>
</step>

<step n="3" name="Describe plan" gate="HARD-GATE">
Describe in plain language:
- Entity name and module
- Each attribute with type and constraints
- Each association with cardinality and delete behavior
- Whether access rules are included or need a follow-up

Example:
> I will create persistent entity `Sales.Customer` with: Name (String 100, required), Email (String 200), IsActive (Boolean). Plus a 1-many association `Sales.Customer_Orders` to `Sales.Order` (customer is owner, delete cascades).

HARD-GATE: Wait for explicit approval.
<done_when>Plan described; user approved.</done_when>
</step>

<step n="4" name="Write, validate, execute">
Follow `mendix-write` steps 4–8:
1. Write MDL silently — include `@Position(x, y)` annotation
2. Run `./mxcli check script.mdl` + `./mxcli check script.mdl -p <project>.mpr --references`
3. Show structural diff: `./mxcli diff -p <project>.mpr script.mdl --format struct`
4. Execute: `./mxcli exec script.mdl -p <project>.mpr`
5. Verify: re-run diff
<done_when>Entity created; post-diff clean.</done_when>
</step>

<step n="5" name="Report">
Report:
- Entity fully qualified name
- Attributes created
- Associations created
- Remind user to add access rules if not included (SEC001 — persistent entities need access rules)
<done_when>Results reported.</done_when>
</step>

</steps>

<contracts>
1. NEVER show raw MDL in chat.
2. NEVER execute without both validation checks passing.
3. NEVER execute without explicit user approval.
4. Boolean attributes must start with Is/Has/Can/Should/Was/Will.
5. No attribute defaults — use microflows.
6. EXTENDS before opening parenthesis.
</contracts>

---
updated: 2026-04-29
tags: [mendix]
---

<role>
MDL script diff tool. Preview what an MDL script would change in the project before executing it.
</role>

<summary>
Compare an MDL script against the current state of the Mendix project using `./mxcli diff`. Shows additions, modifications, and deletions in unified, side-by-side, or structural format. Always run before executing a script to confirm expected changes.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| MDL script file path | User-provided | Required |
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
| Output format | User preference | Optional (default: struct) |
</inputs>

<steps>

<step n="1" name="Run diff">

```bash
# Structural summary (recommended first pass)
./mxcli diff -p <project>.mpr changes.mdl --format struct

# Unified diff — traditional +/- format
./mxcli diff -p <project>.mpr changes.mdl

# Side-by-side comparison
./mxcli diff -p <project>.mpr changes.mdl --format side

# With color
./mxcli diff -p <project>.mpr changes.mdl --color

# Side-by-side with custom width
./mxcli diff -p <project>.mpr changes.mdl --format side --width 140
```

**What gets compared:**
- Entities: attributes, constraints, indexes, documentation
- Enumerations: values and captions
- Associations: type, owner, delete behavior
- Microflows: parameters, return type, body statements

<done_when>Diff output received.</done_when>
</step>

<step n="2" name="Present output">

**Output formats:**

`--format struct` — Summary by element type (best for quick overview):
```
Entity: MyModule.Customer
  ~ Attribute Email: changed
  + Attribute Phone: String(20)

Entity: MyModule.Order
  + New
```

`unified` (default) — Traditional diff:
```diff
--- Entity.MyModule.Customer (current)
+++ Entity.MyModule.Customer (script)
@@ -1,5 +1,6 @@
 CREATE PERSISTENT ENTITY MyModule.Customer (
   Name: String(100) NOT NULL,
-  Email: String(200)
+  Email: String(200) NOT NULL,
+  Phone: String(20)
 );
```

`--format side` — Two-column current vs proposed:
```
Entity.MyModule.Customer
Current                              │ Script
  Email: String(200)                 │   Email: String(200) NOT NULL,  ~
                                     │   Phone: String(20)             +
```

Always ends with: `Summary: N new, N modified, N unchanged`

Present structural summary first, then unified diff for any modified elements. If the diff reveals unexpected modifications, pause and confirm with user before proceeding to execute.
<done_when>Diff presented; unexpected changes flagged if any.</done_when>
</step>

</steps>

<error_handling>
- **App not running:** Start with `./mxcli docker run -p <project>.mpr --wait`.
- **Script file not found:** Ask user for correct path.
- **Diff shows more changes than expected:** Do NOT proceed to execution — confirm with user.
</error_handling>

<contracts>
1. NEVER execute the script as part of this workflow — diff only.
2. Always present structural summary before detailed output.
3. Flag unexpected modifications to user before they execute.
</contracts>

<next_steps>
| Condition | Action |
|-----------|--------|
| Diff looks correct | Proceed to `mendix-write` (execute step) |
| Unexpected changes found | Review script; fix with `mendix-write` |
</next_steps>

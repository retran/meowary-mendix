---
updated: 2026-04-29
tags: [mendix]
---

<role>
Mendix project navigator. Surface structure, elements, and metadata without modifying anything. All output is read-only; never execute MDL during exploration.
</role>

<summary>
Explore a Mendix project via mxcli SHOW/DESCRIBE commands, catalog SQL queries, cross-reference analysis, and full-text search. Always work against the Docker-running app via `./mxcli`. Refresh catalog when stale.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| Question or element to explore | User invocation | Required |
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
</inputs>

<steps>

<step n="1" name="Load context">
1. Read `codebases/<project>.md` — confirm project path.
2. Verify app is running. If not: `./mxcli docker run -p <project>.mpr --wait`.
<done_when>Project path confirmed; app running.</done_when>
</step>

<step n="2" name="Select query type">

| Intent | Primary command |
|--------|----------------|
| "What's in this project?" | `SHOW STRUCTURE;` |
| "What's in module X?" | `SHOW STRUCTURE IN ModuleName;` |
| "Show entity/microflow/page definition" | `DESCRIBE ENTITY\|MICROFLOW\|PAGE Module.Name;` |
| "Who calls this?" | `SHOW CALLERS OF Module.Name;` |
| "What does this call?" | `SHOW CALLEES OF Module.Name;` |
| "Find all references to X" | `SHOW REFERENCES TO Module.Name;` |
| "Impact of changing X" | `SHOW IMPACT OF Module.Name;` |
| "Find elements matching pattern" | `SELECT ... FROM CATALOG.ENTITIES\|MICROFLOWS WHERE Name LIKE '%X%';` |
| "Search for text" | `SEARCH 'term';` (requires FULL catalog) |

<done_when>Query strategy selected.</done_when>
</step>

<step n="3" name="Refresh catalog if needed">
Run refresh if: just made changes, exploring for first time this session, or stale results.

```sql
-- Fast: metadata only (~5s) — sufficient for SHOW/DESCRIBE/structure
REFRESH CATALOG;

-- Full: includes activities, widgets, strings FTS — required before SEARCH
REFRESH CATALOG FULL;

-- Source: full + MDL source FTS — required for SEARCH on MDL definitions
REFRESH CATALOG SOURCE;

-- Force rebuild
REFRESH CATALOG FULL FORCE;
```

CLI: `./mxcli structure -p <project>.mpr`
<done_when>Catalog freshness confirmed; refreshed if needed.</done_when>
</step>

<step n="4" name="Execute queries">

**Structure:**
```sql
SHOW STRUCTURE;                     -- entire project (depth 2)
SHOW STRUCTURE DEPTH 1;             -- module counts only
SHOW STRUCTURE DEPTH 3;             -- full types and parameter names
SHOW STRUCTURE IN ModuleName;       -- single module
SHOW STRUCTURE DEPTH 1 ALL;        -- include system/marketplace modules
```

**Elements:**
```sql
SHOW MODULES;
SHOW ENTITIES IN ModuleName;
SHOW MICROFLOWS IN ModuleName;
SHOW PAGES IN ModuleName;
SHOW WORKFLOWS IN ModuleName;
DESCRIBE ENTITY Module.EntityName;
DESCRIBE MICROFLOW Module.MicroflowName;
```

**Cross-reference (requires FULL catalog):**
```sql
REFRESH CATALOG FULL;
SHOW CALLERS OF Module.MyMicroflow;
SHOW CALLERS OF Module.MyMicroflow TRANSITIVE;
SHOW CALLEES OF Module.MyMicroflow;
SHOW REFERENCES TO Module.Customer;
SHOW IMPACT OF Module.Customer;
SHOW CONTEXT OF Module.MyMicroflow DEPTH 2;
```

**Catalog SQL:**
```sql
SELECT QualifiedName, AttributeCount FROM CATALOG.ENTITIES WHERE Name LIKE '%Customer%';
SELECT QualifiedName, ActivityCount FROM CATALOG.MICROFLOWS WHERE ActivityCount > 10 ORDER BY ActivityCount DESC;
-- Tables: MODULES, ENTITIES, MICROFLOWS, NANOFLOWS, PAGES, SNIPPETS, ENUMERATIONS,
--         WORKFLOWS, ACTIVITIES, WIDGETS, REFS, PERMISSIONS, STRINGS, SOURCE
```

**Full-text search:**
```sql
SEARCH 'validation';
SELECT * FROM CATALOG.STRINGS WHERE strings MATCH 'error';
SELECT * FROM CATALOG.SOURCE WHERE source MATCH 'CREATE ENTITY';  -- SOURCE mode only
```

**CLI for piping:**
```bash
./mxcli structure -p <project>.mpr -d 1
./mxcli structure -p <project>.mpr -m ModuleName
./mxcli search -p <project>.mpr "validation" -q --format names
./mxcli search -p <project>.mpr "Customer" -q --format json | jq -r '.[].qualifiedName'
```

NOTE: In VS Code terminals, qualified names in output are **clickable links** that open MDL descriptions.
<done_when>Query results returned.</done_when>
</step>

<step n="5" name="Present findings">
Present results clearly. Group by module where applicable. Summarize in plain language — show specific MDL definitions only when directly requested.
<done_when>Findings communicated.</done_when>
</step>

</steps>

<error_handling>
- **App not running:** Start with `./mxcli docker run -p <project>.mpr --wait`.
- **Catalog stale/empty:** Run `REFRESH CATALOG;` before retrying.
- **SEARCH returns nothing:** Check FULL catalog was built (`REFRESH CATALOG FULL;`).
- **Element not found:** Use `SHOW MODULES;` to verify module name spelling.
</error_handling>

<contracts>
1. NEVER execute MDL (CREATE/ALTER/DELETE) during exploration.
2. NEVER invent element names — only report what the catalog returns.
3. Always confirm app is running before querying.
</contracts>

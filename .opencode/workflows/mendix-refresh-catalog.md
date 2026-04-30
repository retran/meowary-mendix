---
updated: 2026-04-29
tags: [mendix]
---

<role>
Mendix catalog manager. Rebuild the catalog database so queries and analysis commands have fresh metadata.
</role>

<summary>
Rebuild the mxcli catalog database for the Mendix project. Run after any project changes, before catalog queries, or when exploring a project for the first time. Three modes: fast metadata-only, full (includes activities/widgets/strings FTS), and source (full + MDL source FTS).
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| Refresh mode | User intent or upstream workflow | Required |
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr`) | Required |
</inputs>

<steps>

<step n="1" name="Select refresh mode">

| Need | Mode |
|------|------|
| After changes; before structure/describe queries | `REFRESH CATALOG;` |
| Before using SEARCH command | `REFRESH CATALOG FULL;` |
| Before searching MDL source definitions | `REFRESH CATALOG SOURCE;` |
| Cache seems stale / inconsistent results | `REFRESH CATALOG FULL FORCE;` |

<done_when>Mode selected.</done_when>
</step>

<step n="2" name="Run refresh">

```sql
-- Fast mode (metadata only, ~5 seconds)
REFRESH CATALOG;

-- Full mode (includes activities, widgets, strings FTS table)
REFRESH CATALOG FULL;

-- Source mode (full + MDL source FTS table)
REFRESH CATALOG SOURCE;

-- Force rebuild (ignores cache)
REFRESH CATALOG FULL FORCE;
```

<done_when>Refresh completed without error.</done_when>
</step>

<step n="3" name="Verify with sample query">
Run a quick verification query to confirm the catalog is populated:

```sql
SELECT COUNT(*) FROM CATALOG.ENTITIES;
SELECT COUNT(*) FROM CATALOG.MICROFLOWS;
```

<done_when>Queries return non-zero counts.</done_when>
</step>

<step n="4" name="Sample queries after refresh">

```sql
-- Find entities
SELECT Name, AttributeCount FROM CATALOG.ENTITIES WHERE ModuleName = 'Sales';

-- Find microflows
SELECT QualifiedName, ParameterCount FROM CATALOG.MICROFLOWS WHERE Name LIKE '%Customer%';

-- Find pages
SELECT Name, WidgetCount FROM CATALOG.PAGES;

-- Full-text search (requires FULL mode)
SEARCH 'validation';

-- Raw FTS on strings
SELECT * FROM CATALOG.STRINGS WHERE strings MATCH 'error';

-- Raw FTS on source (requires SOURCE mode)
SELECT * FROM CATALOG.SOURCE WHERE source MATCH 'CREATE ENTITY';
```

<done_when>Queries return expected results.</done_when>
</step>

</steps>

<error_handling>
- **App not running:** Start with `./mxcli docker run -p <project>.mpr --wait` first.
- **SEARCH returns nothing after FULL refresh:** Try `REFRESH CATALOG FULL FORCE;` to bypass cache.
- **Counts are zero:** The project may have no user modules yet. Run `SHOW MODULES;` to confirm.
</error_handling>

<contracts>
1. NEVER modify the project during catalog refresh — read-only operation.
2. Use FULL mode before any SEARCH command.
3. Use SOURCE mode before searching MDL definitions.
</contracts>

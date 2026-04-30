---
name: mendix
description: Mendix MDL authoring, project navigation, and Docker app lifecycle — entities, microflows, pages, security, navigation, OQL, REST, testing, linting, and migration. Load when working in any Mendix codebase.
compatibility: opencode
updated: 2026-04-29
---

<role>MDL authoring guide, project navigator, and Docker lifecycle operator for Mendix applications.</role>

<summary>
> Load when working in any Mendix project. Provides core conventions, Docker runtime instructions, and a sub-skill routing index. Always load the relevant sub-skill before writing MDL. The app runs via Docker — know the lifecycle commands before touching the project.
</summary>

<triggers>
Load this skill when:
- Working in any Mendix codebase (any Mendix project)
- Writing or modifying MDL scripts (entities, microflows, pages, security, navigation)
- Running or managing the app in Docker
- Linting, validating, or diffing the project
- Asking about Mendix architecture, patterns, or conventions
</triggers>

<context>
Read the active codebase file (`codebases/<project>.md`) to confirm:
- **Project file:** `<project>.mpr` — name varies per project
- **Working directory:** local repo path
- **Local binary:** `./mxcli` — NEVER bare `mxcli`
- **App URL:** `http://localhost:8080` (app) | `http://localhost:8090` (admin) — port may vary
</context>

<rules>
1. **Never show raw MDL in chat.** Describe changes as a numbered plain-language list, get explicit approval, then write the script silently and execute.
2. **Always quote identifiers** with double quotes: `"Name"`, `"Email"`. They strip automatically and prevent reserved-keyword conflicts.
3. **Always validate before executing.** Run both:
   - `./mxcli check script.mdl` (syntax + microflow body)
   - `./mxcli check script.mdl -p <project>.mpr --references` (reference check)
4. **Always use `./mxcli`** (local binary). Never bare `mxcli`.
5. **Load the relevant sub-skill FIRST** before writing any MDL, creating demo data, or doing database work.
6. **`EXTENDS` goes BEFORE the opening parenthesis.**
7. **No `CASE/WHEN`.** Use nested `IF…ELSE`.
8. **No `TRY/CATCH`.** Use `ON ERROR`.
</rules>

<docker>

## Docker Lifecycle

The app runs inside a Docker-in-Docker devcontainer. The build output (`.docker/build/`) is **volume-mounted** into the container — no Docker image rebuild needed after MDL changes.

### Start / restart

```bash
# First run or after destructive schema changes
./mxcli docker run -p <project>.mpr --wait

# Fresh start (wipes database volumes)
./mxcli docker run -p <project>.mpr --fresh --wait
```

`--wait` blocks until "Runtime successfully started".

### After MDL changes (hot reload — keeps database)

```bash
# 1. Apply MDL changes
./mxcli exec changes.mdl -p <project>.mpr

# 2. Rebuild PAD
./mxcli docker build -p <project>.mpr --skip-check

# 3. Hot reload model (~100ms, no restart)
./mxcli docker reload -p <project>.mpr --model-only
```

Or combine build + reload:
```bash
./mxcli exec changes.mdl -p <project>.mpr
./mxcli docker reload -p <project>.mpr --skip-check
```

### When to use `--fresh` vs hot reload

| Change | Command |
|--------|---------|
| Microflow / page / security / navigation | `docker reload` — keeps data |
| New entity or attribute (additive) | `docker reload` — runtime applies DDL |
| Removed entity / attribute / type change | `docker run --fresh` — destructive DDL |
| First-time setup | `docker run` |
| Database corruption or reset needed | `docker run --fresh` |

### CSS / theme only

```bash
./mxcli docker build -p <project>.mpr       # compile SCSS first
./mxcli docker reload -p <project>.mpr --css  # push to browsers instantly
```

### Monitor and manage

```bash
./mxcli docker status -p <project>.mpr
./mxcli docker logs -p <project>.mpr --follow
./mxcli docker logs -p <project>.mpr --tail 50
./mxcli docker shell -p <project>.mpr
./mxcli docker down -p <project>.mpr           # stop (keep volumes)
./mxcli docker down -p <project>.mpr --volumes  # stop + wipe database
```

### OQL queries against live runtime

```bash
./mxcli oql -p <project>.mpr "SELECT Name FROM MyModule.Customer"
./mxcli oql -p <project>.mpr --json "SELECT COUNT(*) FROM MyModule.Order"
```

### Validate project (mx check)

```bash
./mxcli docker check -p <project>.mpr
```

### Troubleshooting

| Problem | Solution |
|---------|----------|
| App not responding | `./mxcli docker logs -p <project>.mpr --tail 50` |
| Schema mismatch / DDL errors | `./mxcli docker run -p <project>.mpr --fresh --wait` |
| OQL "Action not found" | `./mxcli docker init -p <project>.mpr --force` (re-init stack) |
| Port conflict | `./mxcli docker init -p <project>.mpr --port-offset N --force` |
| mxbuild missing | `./mxcli setup mxbuild -p <project>.mpr` |

</docker>

<quick_reference>

```bash
# Execute MDL script
./mxcli exec script.mdl -p <project>.mpr

# Syntax check
./mxcli check script.mdl

# Syntax + reference check
./mxcli check script.mdl -p <project>.mpr --references

# Diff script vs project
./mxcli diff -p <project>.mpr script.mdl --format struct

# Lint
./mxcli lint -p <project>.mpr --color

# Best-practices report
./mxcli report -p <project>.mpr

# Project structure
./mxcli structure -p <project>.mpr

# Single inline query
./mxcli -p <project>.mpr -c "SHOW MODULES"
```

</quick_reference>

<sub_skills>

Load the matching sub-skill BEFORE writing any MDL for that domain.

| Task | Sub-skill(s) |
|------|-------------|
| Writing CREATE MICROFLOW, parameters, return types, decisions, loops, error handling, JavaDoc | `write-microflows` + `cheatsheet-variables` |
| Writing nanoflows (client-side, offline-capable, no server roundtrip) | `write-nanoflows` |
| Writing CREATE MICROFLOW for Save/Validate/Delete/Cancel/New/Edit/DataSource patterns | `patterns-crud` |
| Writing data processing microflows: loops, aggregates, list operations, batch processing | `patterns-data-processing` |
| Writing validation microflows that return feedback messages; client-side nanoflow variants | `validation-microflows` |
| Fixing variable declaration errors, expression errors, control flow syntax, CE codes | `cheatsheet-errors` |
| Looking up MDL variable syntax, type declarations, scope rules, common mistakes | `cheatsheet-variables` |
| Pre-flight: validating script atomicity, checking supported/unsupported statements before exec | `check-syntax` |
| Creating new pages (DYNAMICTEXT, ACTIONBUTTON, LAYOUTGRID, DATAGRID, DATAVIEW, GALLERY, etc.) | `create-page` |
| Modifying existing pages/snippets via ALTER PAGE: SET, INSERT, DROP, REPLACE | `alter-page` |
| Building overview/list pages with DataGrid and linked NewEdit forms | `overview-pages` |
| Building master-detail pages with Gallery selection binding and DataView SELECTION source | `master-detail-pages` |
| Reusing UI logic via fragments: DEFINE FRAGMENT, USE FRAGMENT, SHOW/DESCRIBE FRAGMENTS | `fragments` |
| Updating widget properties in bulk across pages and snippets (SHOW WIDGETS, UPDATE WIDGETS) | `bulk-widget-updates` |
| Building a custom pluggable widget with React + TypeScript, widget.xml, build, and install | `create-custom-widget` |
| Using third-party/custom widgets in MDL: GALLERY, COMBOBOX, .def.json, template JSON | `custom-widgets` |
| Customizing app theme: SCSS workflow, CSS reload via Docker, caveats | `theme-styling` |
| Creating or modifying domain model: persistent/non-persistent/view entities, associations, enumerations, indexes, generalization | `generate-domain-model`, `mdl-entities` |
| MDL entity syntax reference: attributes, types, constraints, access rules, ALTER ENTITY | `mdl-entities` |
| Writing OQL SELECT queries, aggregates, joins, GROUP BY for VIEW entities | `write-oql-queries` |
| XPath constraints in retrieve actions and security access rules | `xpath-constraints` |
| Configuring module roles, user roles, entity/microflow/page access rules, demo users | `manage-security` |
| Setting up navigation: home page, role-based routing, menu tree, login/not-found pages | `manage-navigation` |
| Moving documents between folders/modules, renaming, organizing project structure | `organize-project` |
| Viewing or modifying project settings: database config, startup/shutdown microflows, constants | `project-settings` |
| Integrating with external REST APIs: OpenAPI import, manual REST client, Data Transformers (JSLT) | `rest-client` |
| Generating full integration stack from JSON payload: JSON Structure → entities → Import Mapping → microflow | `rest-call-from-json` |
| Building JSON Structures, Import Mappings, Export Mappings, nested arrays, null handling | `json-structures-and-mappings` |
| Publishing or consuming OData services; sharing data between Mendix apps | `odata-data-sharing` |
| Creating/consuming business events: publish/subscribe, message definitions, PublishBusinessEvent_V2 | `business-events` |
| Connecting to external databases: CREATE DATABASE CONNECTION, query definitions, EXECUTE DATABASE QUERY | `database-connections` |
| Writing or calling Java actions from microflows; MDL CREATE JAVA ACTION with type parameters | `java-actions` |
| Connecting to RapidMiner/AnzoGraph via SPARQL: inline REST, JSLT, import mapping pipeline | `connect-rapidminer-graph` |
| Creating AI agent documents: CREATE MODEL, CREATE KNOWLEDGE BASE, CREATE CONSUMED MCP SERVICE, CREATE AGENT | `agents` |
| Searching Mendix Platform Service Registry for OData/REST/SOAP services (catalog.mendix.com) | `catalog-search` |
| Browsing and importing integration services: SHOW/DESCRIBE OData, REST, business event services | `browse-integrations` |
| Referencing System module entities: System.User, FileDocument, Image, UserRole, workflow types | `system-module` |
| Running the app in Docker: first run, rebuild after MDL changes, hot reload workflow | `run-app` |
| Full Docker lifecycle: docker run/build/up/reload/down, --fresh vs reload decision, CSS reload | `docker-workflow` |
| Querying running app via admin API (port 8090): OQL at runtime, M2EE API, admin console | `runtime-admin-api` |
| Inserting or importing demo/test data into the live database via IMPORT FROM or SQL templates | `demo-data` |
| Verifying deployed changes via OQL: confirm microflow side effects, data assertions post-deploy | `verify-with-oql` |
| End-to-end browser testing with Playwright: UI rendering, form interactions, data persistence | `test-app` |
| Microflow unit/integration testing via mxcli test: business logic, entity operations, control flow | `test-microflows` |
| Writing custom Starlark lint rules for project-specific code conventions | `write-lint-rules` |
| Full project quality assessment: lint scores, naming conventions, security, maintainability, architecture | `assess-quality` |
| Assessing a non-Mendix project for migration: tech stack, data model, business logic, UI, integrations | `assess-migration` |
| Migrating K2/Nintex: SmartObjects → entities, SmartForms → pages, workflows → microflows | `migrate-k2-nintex` |
| Migrating Oracle Forms: PL/SQL → microflows, Forms blocks → DataViews, trigger mapping | `migrate-oracle-forms` |
| Debugging BSON serialization: mxcli dump-bson, array markers, CE0463, TextTemplate fixes | `debug-bson` |

</sub_skills>

<self_review>
- [ ] Checked relevant sub-skill before writing MDL?
- [ ] All identifiers quoted with double quotes?
- [ ] Both `check` passes run before execution?
- [ ] No raw MDL shown in chat before approval?
- [ ] `./mxcli` used (not bare `mxcli`)?
- [ ] Used `docker reload` for non-destructive changes (not `docker run`)?
</self_review>

<output_rules>Output in English. Preserve verbatim CLI commands, MDL syntax, and flag names.</output_rules>



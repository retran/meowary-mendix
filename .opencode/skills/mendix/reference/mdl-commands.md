---
name: mdl-command-reference
description: Complete MDL command reference by domain — exploration, domain model, microflows, pages, security, navigation, settings, integrations, SQL, catalog, and linting rules
updated: 2026-04-30
---

# MDL Command Reference

## Exploration & Structure

| Command | Description |
|---------|-------------|
| `SHOW MODULES` | List all modules |
| `SHOW STRUCTURE [DEPTH 1|2|3] [IN Module] [ALL]` | Compact project overview at different detail levels |
| `SHOW CALLERS OF Module.Microflow` | Find what calls a microflow |
| `SHOW CALLEES OF Module.Microflow` | Find what a microflow calls |
| `SHOW REFERENCES OF Module.Entity` | Find all references to an element |
| `SHOW IMPACT OF Module.Entity` | Impact analysis for changes |
| `SHOW CONTEXT OF Module.Microflow` | Show callers + callees + references |
| `SEARCH 'keyword'` | Full-text search across all strings and source |
| `HELP [topic]` | Show all commands or help on a topic |

## Domain Model

| Command | Description |
|---------|-------------|
| `SHOW ENTITIES [IN Module]` | List entities |
| `SHOW ASSOCIATIONS [IN Module]` | List associations |
| `SHOW ENUMERATIONS [IN Module]` | List enumerations |
| `SHOW CONSTANTS [IN Module]` | List constants |
| `DESCRIBE ENTITY Module.Entity` | Show entity definition in MDL |
| `DESCRIBE ASSOCIATION Module.Assoc` | Show association definition |
| `DESCRIBE ENUMERATION Module.Enum` | Show enumeration definition |
| `CREATE MODULE ModuleName` | Create a new module |
| `CREATE PERSISTENT ENTITY ...` | Create a persistent entity with attributes |
| `CREATE NON-PERSISTENT ENTITY ...` | Create a non-persistent (transient) entity |
| `CREATE ASSOCIATION ...` | Create an association between entities |
| `CREATE ENUMERATION ...` | Create an enumeration |
| `ALTER ENTITY Module.Entity ADD ...` | Add/rename/modify/drop attributes, indexes, docs |
| `DROP ENTITY Module.Entity` | Delete an entity |
| `DROP ASSOCIATION Module.Assoc` | Delete an association |
| `DROP ENUMERATION Module.Enum` | Delete an enumeration |

## Microflows & Nanoflows

| Command | Description |
|---------|-------------|
| `SHOW MICROFLOWS [IN Module]` | List microflows |
| `SHOW NANOFLOWS [IN Module]` | List nanoflows |
| `DESCRIBE MICROFLOW Module.Flow` | Show microflow definition in MDL |
| `DESCRIBE NANOFLOW Module.Flow` | Show nanoflow definition in MDL |
| `CREATE MICROFLOW ... BEGIN ... END;` | Create a microflow with activities |
| `CREATE NANOFLOW ... BEGIN ... END;` | Create a nanoflow with activities |
| `DROP MICROFLOW Module.Flow` | Delete a microflow |
| `DROP NANOFLOW Module.Flow` | Delete a nanoflow |

## Pages & Snippets

| Command | Description |
|---------|-------------|
| `SHOW PAGES [IN Module]` | List pages |
| `SHOW SNIPPETS [IN Module]` | List snippets |
| `DESCRIBE PAGE Module.Page` | Show page definition in MDL |
| `DESCRIBE SNIPPET Module.Snippet` | Show snippet definition |
| `CREATE PAGE ... { widgets }` | Create a page with widget syntax |
| `CREATE SNIPPET ... { widgets }` | Create a reusable snippet |
| `ALTER PAGE Module.Page { ops }` | Modify page in-place (SET, INSERT, DROP, REPLACE) |
| `ALTER SNIPPET Module.Snippet { ops }` | Modify snippet in-place |
| `DROP PAGE Module.Page` | Delete a page |
| `DROP SNIPPET Module.Snippet` | Delete a snippet |

## Security

| Command | Description |
|---------|-------------|
| `SHOW PROJECT SECURITY` | Security level, admin, demo users overview |
| `SHOW MODULE ROLES [IN Module]` | Module-level roles |
| `SHOW USER ROLES` | Project-level user roles |
| `SHOW DEMO USERS` | Configured demo users |
| `SHOW ACCESS ON MICROFLOW\|PAGE\|ENTITY Mod.Name` | Role access on element |
| `SHOW SECURITY MATRIX [IN Module]` | Full access overview |
| `CREATE MODULE ROLE Mod.Role` | Create a module role |
| `CREATE USER ROLE Name (Mod.Role, ...)` | Create a user role aggregating module roles |
| `ALTER USER ROLE Name ADD\|REMOVE MODULE ROLES (...)` | Modify user role |
| `GRANT EXECUTE ON MICROFLOW Mod.MF TO Mod.Role` | Grant microflow access |
| `GRANT VIEW ON PAGE Mod.Page TO Mod.Role` | Grant page access |
| `GRANT Mod.Role ON Mod.Entity (CREATE, DELETE, READ *, WRITE *)` | Grant entity access |
| `REVOKE EXECUTE\|VIEW\|role ON element FROM role` | Revoke access |
| `ALTER PROJECT SECURITY LEVEL OFF\|PROTOTYPE\|PRODUCTION` | Set security level |
| `ALTER PROJECT SECURITY DEMO USERS ON\|OFF` | Toggle demo users |
| `CREATE DEMO USER 'name' PASSWORD 'pass' (UserRole, ...)` | Create demo user |
| `DROP MODULE ROLE\|USER ROLE\|DEMO USER ...` | Delete roles/users |

## Navigation

| Command | Description |
|---------|-------------|
| `SHOW NAVIGATION` | Summary of all profiles |
| `SHOW NAVIGATION MENU [Profile]` | Menu tree for profile or all |
| `SHOW NAVIGATION HOMES` | Home page assignments across profiles |
| `DESCRIBE NAVIGATION [Profile]` | Full MDL output (round-trippable) |
| `CREATE OR REPLACE NAVIGATION Profile ...` | Full replacement of a navigation profile |

## Project Settings

| Command | Description |
|---------|-------------|
| `SHOW SETTINGS` | Overview of all settings |
| `DESCRIBE SETTINGS` | Full MDL output (round-trippable) |
| `ALTER SETTINGS MODEL Key = Value` | AfterStartupMicroflow, HashAlgorithm, JavaVersion, etc. |
| `ALTER SETTINGS CONFIGURATION 'Name' Key = Value` | DatabaseType, DatabaseUrl, HttpPortNumber, etc. |
| `ALTER SETTINGS CONSTANT 'Name' VALUE 'val' IN CONFIGURATION 'cfg'` | Override constant per configuration |
| `ALTER SETTINGS LANGUAGE Key = Value` | DefaultLanguageCode |
| `ALTER SETTINGS WORKFLOWS Key = Value` | UserEntity, DefaultTaskParallelism |

## Business Events & Java Actions

| Command | Description |
|---------|-------------|
| `SHOW DATABASE CONNECTIONS [IN Module]` | List database connections |
| `DESCRIBE DATABASE CONNECTION Mod.Name` | Show connection definition in MDL |
| `SHOW BUSINESS EVENTS [IN Module]` | List business event services |
| `DESCRIBE BUSINESS EVENT SERVICE Mod.Name` | Full MDL output |
| `CREATE BUSINESS EVENT SERVICE ...` | Create a business event service |
| `DROP BUSINESS EVENT SERVICE Mod.Name` | Delete a service |
| `SHOW JAVA ACTIONS [IN Module]` | List Java actions |
| `DESCRIBE JAVA ACTION Mod.Name` | Full MDL output with signature |
| `CREATE JAVA ACTION ... AS $$ ... $$` | Create with inline Java code |
| `DROP JAVA ACTION Mod.Name` | Delete a Java action |

## OData

| Command | Description |
|---------|-------------|
| `SHOW ODATA CLIENTS [IN Module]` | List consumed OData services |
| `SHOW ODATA SERVICES [IN Module]` | List published OData services |
| `DESCRIBE ODATA CLIENT Mod.Name` | Full consumed OData MDL output |
| `DESCRIBE ODATA SERVICE Mod.Name` | Full published OData MDL output |
| `CREATE ODATA CLIENT ...` | Create a consumed OData service |
| `CREATE ODATA SERVICE ...` | Create a published OData service |
| `ALTER ODATA CLIENT\|SERVICE ...` | Modify an OData service |
| `DROP ODATA CLIENT\|SERVICE Mod.Name` | Delete an OData service |

## External SQL

| Command | Description |
|---------|-------------|
| `SQL CONNECT <driver> '<dsn>' AS <alias>` | Connect to external database (postgres) |
| `SQL DISCONNECT <alias>` | Close connection |
| `SQL CONNECTIONS` | List active connections (alias + driver only) |
| `SQL <alias> SHOW TABLES` | List tables via information_schema |
| `SQL <alias> DESCRIBE <table>` | Show columns, types, nullability |
| `SQL <alias> <any-sql>` | Raw SQL passthrough to external DB |

## Catalog Queries

| Command | Description |
|---------|-------------|
| `REFRESH CATALOG` | Build catalog (metadata only) |
| `REFRESH CATALOG FULL` | Full catalog with activities, widgets, cross-refs |
| `SHOW CATALOG TABLES` | List available catalog tables |
| `SELECT ... FROM CATALOG.ENTITIES WHERE ...` | SQL queries against project metadata |

Available catalog tables: `CATALOG.MODULES`, `CATALOG.ENTITIES`, `CATALOG.MICROFLOWS`, `CATALOG.PAGES`, `CATALOG.WORKFLOWS`, `CATALOG.ENUMERATIONS`, `CATALOG.ASSOCIATIONS`, `CATALOG.SNIPPETS`, `CATALOG.REFS` (requires FULL mode).

## Project Organization

| Command | Description |
|---------|-------------|
| `MOVE PAGE\|MICROFLOW\|SNIPPET\|... Mod.Name TO FOLDER 'path'` | Move element to folder |
| `MOVE PAGE Mod.Name TO Module` | Move to module root |
| `MOVE ENTITY Old.Name TO NewModule` | Move entity across modules |
| `SHOW WORKFLOWS [IN Module]` | List workflows |
| `DESCRIBE WORKFLOW Module.Workflow` | Show workflow definition |
| `SHOW WIDGETS [IN Module]` | Widget discovery (experimental) |

## Linting

```bash
./mxcli lint -p <project>.mpr              # Lint the project
./mxcli lint -p <project>.mpr --color      # With colored output
./mxcli lint -p <project>.mpr --list-rules # List available rules
./mxcli lint -p <project>.mpr --format sarif > results.sarif  # SARIF output
```

### Built-in Rules

| Rule | Category | Description |
|------|----------|-------------|
| MPR001 | quality | PascalCase naming conventions |
| MPR002 | quality | Empty microflows (no activities) |
| MPR003 | design | Domain model size (>15 persistent entities per module) |
| MPR004 | correctness | Empty validation feedback message (CE0091) |
| MPR005 | correctness | Unconfigured image widget source |
| MPR006 | correctness | Empty containers (runtime crash) |
| MPR007 | security | Navigation page without allowed role (CE0557) |
| SEC001 | security | Persistent entity without access rules |
| SEC002 | security | Weak password policy (minimum length < 8) |
| SEC003 | security | Demo users active at non-development security level |

### Bundled Starlark Rules

27 additional rules in `.claude/lint-rules/*.star`:

| Rule | Category | Description |
|------|----------|-------------|
| SEC004 | security | Guest access enabled |
| SEC005 | security | Strict mode disabled |
| SEC006 | security | PII attributes exposed without access rules |
| SEC007 | security | Anonymous unconstrained READ (DIVD-2022-00019) |
| SEC008 | security | PII entities readable without row scoping |
| SEC009 | security | Large entities missing member-level access restrictions |
| ARCH001 | architecture | Cross-module data access in pages |
| ARCH002 | architecture | Data changes should go through microflows |
| ARCH003 | architecture | Persistent entities need a unique business key |
| QUAL001 | quality | McCabe cyclomatic complexity threshold |
| QUAL002 | quality | Missing documentation on entities/microflows |
| QUAL003 | quality | Long microflows (too many activities) |
| QUAL004 | quality | Orphaned/unreferenced elements |
| DESIGN001 | design | Entity with too many attributes |
| CONV001 | naming | Boolean attributes must start with Is/Has/Can/Should/Was/Will |
| CONV002 | quality | String/numeric attributes should not have default values |
| CONV003 | naming | Pages should follow Entity_NewEdit/View/Overview naming |
| CONV004 | naming | Enumerations should be prefixed with ENUM_ |
| CONV005 | naming | Snippets should be prefixed with SNIPPET_ |
| CONV006 | security | Entity access rules should not grant Create/Delete rights |
| CONV007 | security | All persistent entity access rules need XPath constraints |
| CONV008 | security | Each module role should map to exactly one user role |
| CONV009 | quality | Microflows should have at most 15 objects |
| CONV015 | quality | Entities should not have validation rules |
| CONV016 | performance | Entities should not have event handlers |
| CONV017 | performance | Attributes should not be calculated (virtual) |

## Best Practices Report

```bash
./mxcli report -p <project>.mpr              # Markdown (default)
./mxcli report -p <project>.mpr --format json # JSON
./mxcli report -p <project>.mpr --format html # HTML
```

Scores 6 categories (Naming, Security, Quality, Architecture, Performance, Design) on a 0-100 scale.

## MDL Syntax Quick Reference

### Entity Generalization (EXTENDS)

**CRITICAL: EXTENDS goes BEFORE the opening parenthesis, not after!**

```sql
CREATE PERSISTENT ENTITY Module.ProductPhoto EXTENDS System.Image (
  PhotoCaption: String(200)
);
```

### Microflows - Supported Statements

| Statement | Syntax |
|-----------|--------|
| Variable declaration | `DECLARE $Var Type = value;` |
| Entity declaration | `DECLARE $Entity Module.Entity;` |
| List declaration | `DECLARE $List List of Module.Entity = empty;` |
| Assignment | `SET $Var = expression;` |
| Create object | `$Var = CREATE Module.Entity (Attr = value);` |
| Change object | `CHANGE $Entity (Attr = value);` |
| Commit | `COMMIT $Entity [WITH EVENTS] [REFRESH];` |
| Delete | `DELETE $Entity;` |
| Rollback | `ROLLBACK $Entity [REFRESH];` |
| Retrieve | `RETRIEVE $Var FROM Module.Entity [WHERE condition];` |
| Call microflow | `$Result = CALL MICROFLOW Module.Name (Param = $value);` |
| Call nanoflow | `$Result = CALL NANOFLOW Module.Name (Param = $value);` |
| Call Java action | `$Result = CALL JAVA ACTION Module.Name (Param = value);` |
| Show page | `SHOW PAGE Module.PageName ($Param = $value);` |
| Close page | `CLOSE PAGE;` |
| Validation | `VALIDATION FEEDBACK $Entity/Attribute MESSAGE 'message';` |
| Log | `LOG INFO\|WARNING\|ERROR [NODE 'name'] 'message';` |
| Annotation | `@annotation 'text'` (before activity) |
| Position | `@position(x, y)` (before activity) |
| Error handling | `... ON ERROR CONTINUE\|ROLLBACK\|{ handler };` |
| IF | `IF condition THEN ... [ELSE ...] END IF;` |
| LOOP | `LOOP $Item IN $List BEGIN ... END LOOP;` |
| WHILE | `WHILE condition BEGIN ... END WHILE;` |
| Return | `RETURN $value;` |

### Microflows - NOT Supported

| Unsupported | Use Instead |
|-------------|-------------|
| `CASE ... WHEN ... END CASE` | Nested `IF ... ELSE ... END IF` |
| `TRY ... CATCH` | `ON ERROR { ... }` blocks |

### Pages Syntax Summary

| Element | Syntax | Example |
|---------|--------|--------|
| Page properties | `(Key: value, ...)` | `(Title: 'Edit', Layout: Atlas_Core.Atlas_Default)` |
| Widget name | Required after type | `TEXTBOX txtName (...)` |
| Attribute binding | `Attribute: AttrName` | `TEXTBOX txt (Label: 'Name', Attribute: Name)` |
| Microflow action | `Action: MICROFLOW Name(Param: val)` | `Action: MICROFLOW Mod.ACT_Process(Order: $Order)` |
| Database source | `DataSource: DATABASE Entity` | `DATAGRID dg (DataSource: DATABASE Mod.Entity)` |
| Selection source | `DataSource: SELECTION widget` | `DATAVIEW dv (DataSource: SELECTION galleryList)` |

**Supported Widgets:** LAYOUTGRID, ROW, COLUMN, CONTAINER, TEXTBOX, TEXTAREA, CHECKBOX, RADIOBUTTONS, DATEPICKER, COMBOBOX, DYNAMICTEXT, DATAGRID, GALLERY, LISTVIEW, IMAGE, STATICIMAGE, DYNAMICIMAGE, ACTIONBUTTON, LINKBUTTON, DATAVIEW, HEADER, FOOTER, CONTROLBAR, SNIPPETCALL, NAVIGATIONLIST, CUSTOMCONTAINER.

### ALTER PAGE / ALTER SNIPPET

| Operation | Syntax |
|-----------|--------|
| Set property | `SET Caption = 'New' ON widgetName` |
| Set multiple | `SET (Caption = 'Save', ButtonStyle = Success) ON btn` |
| Page-level set | `SET Title = 'New Title'` (no ON clause) |
| Insert after | `INSERT AFTER widgetName { widgets }` |
| Insert before | `INSERT BEFORE widgetName { widgets }` |
| Drop widgets | `DROP WIDGET name1, name2` |
| Replace widget | `REPLACE widgetName WITH { widgets }` |

### Quoted Identifiers

Always quote all identifiers with double quotes — prevents reserved keyword conflicts:

```sql
CREATE PERSISTENT ENTITY Module."Customer" (
  "Name": String(200),
  "Status": String(50),
  "Create": DateTime
);
```

## MDL Script Files

Store MDL scripts in the `mdlsource/` directory:

```
mdlsource/
├── domain-model.mdl
├── microflows.mdl
└── setup.mdl
```

Execute: `./mxcli exec script.mdl -p <project>.mpr`

## Examples

### Create an Entity

```sql
/**
 * Customer entity — stores customer information.
 */
@Position(100, 100)
CREATE PERSISTENT ENTITY Sales.Customer (
  /** Customer name */
  Name: String(200) NOT NULL ERROR 'Name is required',
  /** Email address */
  Email: String(200) UNIQUE ERROR 'Email must be unique',
  Phone: String(50),
  IsActive: Boolean DEFAULT true
);
```

### Create a Microflow

```sql
/**
 * Validates a customer before saving
 * @param $Customer The customer to validate
 * @returns Boolean indicating validity
 */
CREATE MICROFLOW Sales.VAL_Customer (
  $Customer: Sales.Customer
)
RETURNS Boolean AS $IsValid
BEGIN
  DECLARE $IsValid Boolean = true;

  IF trim($Customer/Name) = '' THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Customer/Name MESSAGE 'Name is required';
  END IF;

  RETURN $IsValid;
END;
/
```

---
name: 'demo-data'
description: 'Connect to Application Database and Generate Demo Data — read DB settings, Mendix ID system, association storage modes, INSERT templates, and IMPORT FROM bulk import'
compatibility: opencode
---

<role>Mendix database data engineer — connect to the Mendix app's PostgreSQL database and insert demo data using correct IDs and association links.</role>

<summary>
> Covers reading DB settings, understanding Mendix's internal ID system, safely inserting rows with correct IDs and association links, and using the IMPORT FROM command for bulk data import.
</summary>

<triggers>
Load when:
- User asks to seed or populate the database with test/demo data
- User needs data in the app before the UI is built
- User wants to inspect the database directly (schema, row counts, etc.)
- Bulk data import that is impractical through the Mendix UI
</triggers>

<step_1_get_database_settings>

```bash
./mxcli -p <project>.mpr -c "show settings;"
# → configuration 'Default' | PostgreSql, localhost:5434, db=mxcli2-dev, http=8080

./mxcli -p <project>.mpr -c "describe settings;"
# → alter settings configuration 'Default'
#     DatabaseType = 'PostgreSql', DatabaseUrl = 'localhost:5434',
#     DatabaseName = 'mxcli2-dev', DatabaseUserName = 'mendix', DatabasePassword = 'mendix', ...
```

</step_1_get_database_settings>

<step_2_connect_to_database>

### From a devcontainer on macOS

The Mendix app's `localhost` in the project settings refers to the **Mac host**, not the devcontainer. Use `host.docker.internal` to reach it:

```bash
PGPASSWORD=mendix psql -h host.docker.internal -p 5434 -U mendix -d mxcli2-dev
```

### Useful psql commands

```bash
\dt                          # list all tables
\d tasklist$task             # describe a table

# Run a query and exit
PGPASSWORD=mendix psql -h host.docker.internal -p 5434 -U mendix -d mxcli2-dev \
  -c "select * from \"tasklist\$task\" limit 5;"
```

</step_2_connect_to_database>

<step_3_mendix_id_system>

Every Mendix object has a `bigint` ID composed of three parts:

```
| bits 63–48  | bits 47–7        | bits 6–0    |
|  entity ID  |  sequence number |  random      |
| (16 bits)   |  (41 bits)       |  (7 bits)    |
```

**Formula:** `id = (short_id::bigint << 48) | (sequence_number::bigint << 7) | (random_7bits)`

The 7-bit random suffix adds unpredictability to object IDs, preventing sequential ID enumeration attacks. Generate it with `floor(random() * 128)` in SQL.

### Look up an entity's short_id and current sequence

```sql
select e.entity_name, e.table_name, ei.short_id, ei.object_sequence,
       (ei.short_id::bigint << 48) as id_base
from mendixsystem$entityidentifier ei
join mendixsystem$entity e on e.id = ei.id
where e.entity_name = 'TaskList.Task';
```

### Decode an existing ID

```sql
select id,
       to_hex(id::bigint)                          as hex_id,
       (id::bigint >> 48)                           as entity_short_id,
       (id::bigint >> 7) & x'1ffffffffff'::bigint   as sequence_num,
       id::bigint & 127                              as random_bits
from "tasklist$task";
```

### ID generation rules

- `object_sequence` is the **next available** sequence number for that entity
- After inserting N rows, **advance** `object_sequence` by N so the running runtime does not reuse those IDs
- Each ID includes a 7-bit random suffix (0–127) for security; generate a fresh random value per row

</step_3_mendix_id_system>

<step_4_association_storage>

Query `mendixsystem$association` to see how each association is stored:

```sql
select association_name, table_name, child_column_name, storage_format
from mendixsystem$association
where table_name like 'tasklist%';
```

### Mode A — Column storage (`AssocStorage: column`)

The FK is a regular column in the **owner** entity's table. No junction table exists.

Column naming convention: `{module}${associationname}` — all lowercase, `$` separator.

```sql
insert into "tasklist$note" (id, content, author, datecreated, "tasklist$note_task", mxobjectversion)
values (
  (59::bigint << 48) | (18::bigint << 7) | floor(random() * 128)::bigint,
  'Note text', 'Alice', '2026-02-18 10:00:00',
  (50::bigint << 48) | (11::bigint << 7) | floor(random() * 128)::bigint,
  1
);
```

### Mode B — Junction table storage

Mendix creates a separate join table. Both entity IDs are stored there.

```sql
with new_note as (
  select (59::bigint << 48) | (18::bigint << 7) | floor(random() * 128)::bigint as id
)
insert into "tasklist$note" (id, content, author, datecreated)
select id, 'Note text', 'Alice', '2026-02-18 10:00:00' from new_note;

-- Then link (reuse the same id — query it back or generate in application code)
insert into "tasklist$note_task" ("tasklist$noteid", "tasklist$taskid") values
  (<the_generated_note_id>, <task_id>);
```

### Optimistic locking — `mxobjectversion`

When the project has optimistic locking enabled, every entity table gets an `mxobjectversion bigint` column.

**Always set `mxobjectversion = 1` when inserting rows directly.**

```sql
select column_name from information_schema.columns
where table_name = 'tasklist$task' and column_name = 'mxobjectversion';
```

</step_4_association_storage>

<step_5_insert_demo_data>

### Template — entity with column-storage association + optimistic locking

```sql
begin;

insert into "tasklist$note" (id, content, author, datecreated, "tasklist$note_task", mxobjectversion)
values
  ((59::bigint << 48) | (18::bigint << 7) | floor(random() * 128)::bigint,
   'First note content',  'Bob',   '2026-02-18 10:00:00',
   (50::bigint << 48) | (11::bigint << 7) | floor(random() * 128)::bigint, 1),
  ((59::bigint << 48) | (19::bigint << 7) | floor(random() * 128)::bigint,
   'Second note content', 'Alice', '2026-02-18 11:00:00',
   (50::bigint << 48) | (11::bigint << 7) | floor(random() * 128)::bigint, 1);

-- Advance Note sequence (was 18, inserted 2, now 20)
update mendixsystem$entityidentifier ei
set object_sequence = 20
from mendixsystem$entity e
where e.id = ei.id and e.entity_name = 'TaskList.Note';

commit;
```

### Template — standalone entity (no association)

```sql
begin;

insert into "tasklist$task" (id, title, taskstatus, priority, assignedto, duedate, iscompleted, estimatedhours, mxobjectversion)
values
  ((50::bigint << 48) | (11::bigint << 7) | floor(random() * 128)::bigint,
   'My demo task', 'ToDo', 'Medium', 'Alice', '2026-03-01 09:00:00', false, 4.0, 1);

-- Advance sequence (was 11, inserted 1 row)
update mendixsystem$entityidentifier ei
set object_sequence = 12
from mendixsystem$entity e
where e.id = ei.id and e.entity_name = 'TaskList.Task';

commit;
```

### INSERT column checklist

| Column | Required | Value |
|--------|----------|-------|
| `id` | Always | `(short_id::bigint << 48) \| (sequence::bigint << 7) \| random_0_127` |
| `mxobjectversion` | If column exists | `1` |
| `module$assocname` | If column-storage association | FK id of related object |
| Custom attributes | As needed | Your data |

</step_5_insert_demo_data>

<important_caveats>

### Reserved attribute names

Do NOT use these names for custom attributes:

| Reserved name | System meaning |
|---------------|----------------|
| `CreatedDate` | Auto-set on object creation |
| `ChangedDate` | Auto-set on every commit |
| `owner` | Reference to creating user |
| `ChangedBy` | Reference to last user to commit |

### New entities need a runtime sync before demo data can be inserted

When you create a new entity with `mxcli exec`, the table only appears **after the Mendix runtime starts and syncs the schema**.

Workflow:
1. Create entity with `mxcli exec`
2. Start (or restart) the Mendix runtime
3. Verify the table exists: `\dt *entityname*`
4. Insert demo data

### Sequence safety

Always update `object_sequence` in the same transaction as your inserts. To be safe, insert demo data while the runtime is stopped.

</important_caveats>

<automated_alternative_import_from>

For bulk imports from an external database, use the `import from` command:

```sql
-- Connect to external database
sql connect postgres 'postgres://user:pass@host:5432/legacydb' as source;

-- Import rows directly into Mendix app database
import from source query 'SELECT name, email, department FROM employees'
  into HRModule.Employee
  map (name as Name, email as Email, department as Department)
  batch 500;

-- Import with association linking (lookup by natural key)
import from source query 'SELECT name, email, dept_name FROM employees'
  into HR.Employee
  map (name as Name, email as Email)
  link (dept_name to Employee_Department on Name);
```

The `import` command auto-connects to the Mendix app's PostgreSQL database using project settings. Override with env vars for devcontainers/Docker:
`MXCLI_DB_TYPE`, `MXCLI_DB_HOST`, `MXCLI_DB_PORT`, `MXCLI_DB_NAME`, `MXCLI_DB_USER`, `MXCLI_DB_PASSWORD`.

Use manual INSERT when you need:
- ReferenceSet association linking
- Custom ID allocation or sequence management
- Non-standard data transformations

</automated_alternative_import_from>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

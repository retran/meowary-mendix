---
name: 'rest-call-from-json'
description: 'Generate the full Mendix integration stack from a JSON payload: JSON Structure → non-persistent entities → Import Mapping → REST CALL microflow'
compatibility: opencode
---

<role>
REST integration expert for Mendix. Covers the four-step pipeline from JSON payload to working REST call: JSON Structure, non-persistent entities, import mapping, and REST CALL microflow.
</role>

<summary>
Use this skill to generate the full stack of Mendix integration artifacts from a JSON payload: JSON Structure → Non-persistent entities → Import Mapping → microflow.

For structured APIs with reusable operations, use the **REST Client** approach instead — see `rest-client` skill.
</summary>

<triggers>
- Generate Mendix integration artifacts from a JSON payload
- Create a JSON Structure document
- Create non-persistent entities for an API response
- Create an import mapping from a JSON structure
- Write a REST CALL microflow with import mapping
</triggers>

<overview>
Four steps to integrate a REST API using inline REST CALL:

1. **CREATE JSON STRUCTURE** — store the raw payload and derive the element tree
2. **CREATE ENTITY** (non-persistent) — one per JSON object type, with attributes per JSON field
3. **CREATE IMPORT MAPPING** — link JSON structure elements to entities and attributes
4. **CREATE MICROFLOW** — inline REST CALL that invokes the import mapping
</overview>

<step1_json_structure>
```sql
create json structure Module.JSON_MyStructure
  snippet '{"key": "value", "count": 1}';
```

- The executor **formats** the snippet (pretty-print) then **refreshes** (derives element tree) automatically
- The snippet must be valid JSON; use single quotes around it in MDL
- Escape single quotes inside the snippet by doubling them: `''`
- The executor sorts JSON object keys alphabetically

**Verify** after creation:
```sql
describe json structure Module.JSON_MyStructure;
-- Should show: element tree under "-- Element tree:" comment
```
</step1_json_structure>

<step2_non_persistent_entities>
Derive one entity per JSON object type. Name them after what they represent.

```sql
create entity Module.MyRootObject (NON_PERSISTENT)
  stringField   : string
  intField      : integer
  decimalField  : decimal
  boolField     : boolean default false;

create entity Module.MyNestedObject (NON_PERSISTENT)
  name : string
  code : string;

create association Module.MyRootObject_MyNestedObject
  from Module.MyRootObject
  to Module.MyNestedObject;
```

**Rules:**
- All string fields: bare `string` (no length — unlimited)
- All number fields: `integer`, `decimal`, or `long`
- Boolean fields **require** `default true|false`
- `NON_PERSISTENT` — these entities are not stored in the database
</step2_non_persistent_entities>

<step3_import_mapping>
```sql
create import mapping Module.IMM_MyMapping
  with json structure Module.JSON_MyStructure
{
  create Module.MyRootObject {
    stringField = stringField,
    intField    = intField,
    create Module.MyRootObject_MyNestedObject/Module.MyNestedObject = nestedKey {
      name = name,
      code = code
    }
  }
};
```

**Syntax rules:**
- Root object: `create Module.Entity { ... }` — always starts with handling keyword
- Value mappings: `attributename = jsonFieldName` — entity attribute on the left, JSON field on the right
- Nested objects: `create association/entity = jsonKey { ... }` — association path + JSON key
- Object handling: `create` (default), `find` (requires KEY), `find or create`
- KEY marker: `attr = jsonField key` — marks the attribute as a matching key
- Value transforms: `attr = Module.Microflow(jsonField)` — call a microflow to transform the value
</step3_import_mapping>

<step4_rest_call_microflow>
```sql
create microflow Module.GET_MyData ()
begin
  @position(-5, 200)
  declare $baseUrl string = 'https://api.example.com';
  @position(185, 200)
  declare $endpoint string = $baseUrl + '/path';
  @position(375, 200)
  $Result = rest call get '{1}' with ({1} = $endpoint)
    header 'Accept' = 'application/json'
    timeout 300
    returns mapping Module.IMM_MyMapping as Module.MyRootObject on error rollback;
  @position(565, 200)
  log info node 'Integration' 'Retrieved result' with ();
end;
/
```

**Key points:**
- `@position` annotations control canvas layout
- The output variable name is **automatically derived** from the entity name in `as Module.MyEntity`
- Single vs list result is **automatically detected** from JSON structure root element type
- `on error rollback` — standard error handling for integration calls

**For list responses** (JSON root is an array):
```sql
  $Results = rest call get '{1}' with ({1} = $endpoint)
    header 'Accept' = 'application/json'
    timeout 300
    returns mapping Module.IMM_MyMapping as Module.MyItem on error rollback;
```
</step4_rest_call_microflow>

<import_export_mapping_in_microflows>
Instead of using `returns mapping` on a REST CALL, you can use standalone mapping actions:

### Import from mapping

```sql
-- With assignment (non-persistent entities, need the result in the flow)
$PetResponse = import from mapping Module.IMM_Pet($JsonContent);

-- Without assignment (persistent entities, just stores to DB)
import from mapping Module.IMM_Pet($JsonContent);
```

### Export to mapping

```sql
$JsonOutput = export to mapping Module.EMM_Pet($PetResponse);
```
</import_export_mapping_in_microflows>

<complete_example>
```sql
-- Step 1: JSON Structure
create json structure Integrations.JSON_BibleVerse
  snippet '{"translation":{"identifier":"web","name":"World English Bible","language":"English","language_code":"eng","license":"Public Domain"},"random_verse":{"book_id":"1SA","book":"1 Samuel","chapter":17,"verse":49,"text":"David put his hand in his bag, took a stone, and slung it."}}';

-- Step 2: Entities
create entity Integrations.BibleApiResponse (NON_PERSISTENT);

create entity Integrations.BibleTranslation (NON_PERSISTENT)
  identifier    : string
  name          : string
  language      : string
  language_code : string
  license       : string;

create entity Integrations.BibleVerse (NON_PERSISTENT)
  book_id : string
  book    : string
  chapter : integer
  verse   : integer
  text    : string;

create association Integrations.BibleApiResponse_BibleTranslation
  from Integrations.BibleApiResponse
  to Integrations.BibleTranslation;

create association Integrations.BibleApiResponse_BibleVerse
  from Integrations.BibleApiResponse
  to Integrations.BibleVerse;

-- Step 3: Import Mapping
create import mapping Integrations.IMM_BibleVerse
  with json structure Integrations.JSON_BibleVerse
{
  create Integrations.BibleApiResponse {
    create Integrations.BibleApiResponse_BibleTranslation/Integrations.BibleTranslation = translation {
      identifier    = identifier,
      language      = language,
      language_code = language_code,
      license       = license,
      name          = name
    },
    create Integrations.BibleApiResponse_BibleVerse/Integrations.BibleVerse = random_verse {
      book    = book,
      book_id = book_id,
      chapter = chapter,
      text    = text,
      verse   = verse
    }
  }
};

-- Step 4: Microflow
create microflow Integrations.GET_BibleVerse_Random ()
begin
  @position(-5, 200)
  declare $baseUrl string = 'https://bible-api.com';
  @position(185, 200)
  declare $endpoint string = $baseUrl + '/data/web/random';
  @position(375, 200)
  $Result = rest call get '{1}' with ({1} = $endpoint)
    header 'Accept' = 'application/json'
    timeout 300
    returns mapping Integrations.IMM_BibleVerse as Integrations.BibleApiResponse on error rollback;
  @position(565, 200)
  log info node 'Integration' 'Retrieved Bible verse' with ();
end;
/
```
</complete_example>

<gotchas>
| Symptom | Cause | Fix |
|---------|-------|-----|
| Studio Pro "not consistent with snippet" | JSON element tree keys not in alphabetical order | Executor sorts keys; re-derive from snippet |
| Schema elements not ticked in import mapping | JsonPath mismatch | Named object elements use `(object)\|key`, NOT `(object)\|key\|(object)` |
| Import mapping not linked in REST call | Wrong BSON field name | Use `ReturnValueMapping`, not `mapping` |
| Studio Pro shows "List of X" but mapping returns single X | `ForceSingleOccurrence` not set | Executor auto-detects from JSON structure root element type |
</gotchas>

<naming_conventions>
| Artifact | Pattern | Example |
|----------|---------|---------|
| JSON Structure | `JSON_<ApiName>` | `JSON_BibleVerse` |
| Import Mapping | `IMM_<ApiName>` | `IMM_BibleVerse` |
| Root entity | Describes the API response | `BibleApiResponse` |
| Nested entities | Describes the domain concept | `BibleVerse`, `BibleTranslation` |
| Microflow | `METHOD_Resource_Operation` | `GET_BibleVerse_Random` |
| Folder | `Private/` for mappings/structures, `Operations/` for public microflows | — |
</naming_conventions>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

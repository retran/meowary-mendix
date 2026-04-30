---
name: 'connect-rapidminer-graph'
description: 'Connecting Mendix to RapidMiner / AnzoGraph via SPARQL — inline REST call with Basic Auth, JSLT transformer, import mapping pipeline, and known gotchas'
compatibility: opencode
---

<role>MDL RapidMiner/AnzoGraph integration author — build the full pipeline from SPARQL endpoint to Mendix entity display.</role>

<summary>
> Step-by-step guide for fetching data from a RapidMiner graph mart (or any SPARQL 1.1 HTTP endpoint) and surfacing it in a Mendix app. Covers the inline REST call + JSLT transformer + import mapping pipeline, persistent entity rationale, and critical gotchas.
</summary>

<triggers>
Load when:
- An external graph database exposes a SPARQL HTTP endpoint with Basic Auth
- User wants graph results to become Mendix entities (for display, search, further processing)
- Read-only SPARQL SELECT queries that return tabular results
</triggers>

<endpoint_shape>

A RapidMiner / AnzoGraph graphmart endpoint looks like:

```
https://<host>/sparql/graphmart/<url-encoded-graphmart-uri>
```

Two things to note:
1. The graphmart URI is **URL-encoded and embedded in the path** (colons and slashes become `%3A` / `%2F`).
2. SPARQL queries are sent as the **POST body** with `content-type: application/sparql-query`, and the response is JSON when `Accept: application/sparql-results+json`.

Verify with curl first:

```bash
curl -u 'user@example.com:password' \
  -H 'Accept: application/sparql-results+json' \
  -H 'Content-Type: application/sparql-query' \
  --data-binary 'SELECT ?s WHERE { ?s a <http://example.com/Foo> } LIMIT 10' \
  'https://host/sparql/graphmart/<encoded-uri>'
```

</endpoint_shape>

<sparql_json_result_shape>

```json
{
  "head": { "vars": ["customer", "customerId", "customerName"] },
  "results": {
    "bindings": [
      {
        "customer":     {"type": "uri",     "value": "http://.../Customer/0000000"},
        "customerId":   {"type": "literal", "value": "CUST001"},
        "customerName": {"type": "literal", "value": "Global Tech Solutions Inc."}
      }
    ]
  }
}
```

Each row in `bindings` is an object of `{var: {type, value}}`. A JSLT transformer flattens this into something directly mappable into Mendix entities.

</sparql_json_result_shape>

<pipeline>

```
┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐   ┌────────────────┐
│  Inline rest call  │─▶│ data transformer   │─▶│  import mapping    │─▶│ Mendix entity  │
│  post + basic auth │   │ jslt: flatten      │   │ json → entities    │   │ (persistent)   │
│  SPARQL as body    │   │ results.bindings   │   │                    │   │                │
└────────────────────┘   └────────────────────┘   └────────────────────┘   └────────────────┘
```

**Why inline `rest call` rather than `create rest client`?**

REST Client `authentication: basic (username: '...', password: '...')` silently fails to attach the `Authorization` header when the password contains special characters (e.g. `!`). Inline `rest call ... auth basic '<user>' password '<pass>'` handles the same credentials correctly.

**Why persistent entities for the final list?**

Non-persistent `ReferenceSet` children can't be extracted as a `list` in MDL microflows, and `loop $c in $Parent/Assoc` fails at build time. Persistent entities work with `datasource: database` on a DataGrid — the standard happy path.

</pipeline>

<step_by_step_template>

### 1. Persistent target entity

```sql
@position(100, 100)
create persistent entity MyModule.Customer (
  CustomerUri:  string(500),
  CustomerId:   string(50),
  CustomerName: string(200)
);
/
```

### 2. Non-persistent wrapper (for the import mapping only)

```sql
@position(400, 100)
create non-persistent entity MyModule.CustomerImport (
  DummyAttr: string(10)
);
/

create association MyModule.CustomerImport_Customer
  from MyModule.CustomerImport
  to   MyModule.Customer
  type ReferenceSet;
/
```

### 3. Data Transformer (JSLT) — flatten SPARQL response

```sql
create data transformer MyModule.SimplifyCustomers
source json '{"head":{"vars":["customer","customerId","customerName"]},"results":{"bindings":[{"customer":{"type":"uri","value":"http://.../Customer/0"},"customerId":{"type":"literal","value":"CUST001"},"customerName":{"type":"literal","value":"Global Tech Solutions Inc."}}]}}'
{
  jslt $$
{
  "customers": [for (.results.bindings)
    {
      "customerUri":  .customer.value,
      "customerId":   .customerId.value,
      "customerName": .customerName.value
    }
  ]
}
  $$;
};
```

**JSLT notes:**
- `[for (.path.to.array) <expr>]` works for iteration.
- `.field.subfield` path access works.
- `[N]` array indexing works.
- `$var[start : end]` slice works for strings — **do not use `substring(...)`** (silently drops the field).
- `let` variables and `if/else` expressions work.
- `def fn(arg) ...` helper functions work.

### 4. JSON structure + Import Mapping

```sql
create json structure MyModule.JSON_Customers
snippet '{"customers":[{"customerUri":"http://example.com/Customer/0","customerId":"CUST001","customerName":"Global Tech Solutions Inc."}]}';

create import mapping MyModule.IMM_Customers
  with json structure MyModule.JSON_Customers
{
  create MyModule.CustomerImport {
    create MyModule.CustomerImport_Customer/MyModule.Customer = customers {
      CustomerUri  = customerUri,
      CustomerId   = customerId,
      CustomerName = customerName
    }
  }
};
```

### 5. Microflow — the actual API call

```sql
create microflow MyModule.ACT_RefreshCustomers ()
returns boolean as $success
begin
  log info node 'MyModule' '=== Refresh start ===';

  -- Clear existing persistent records (full replace)
  retrieve $Existing from MyModule.Customer;
  loop $C in $Existing begin
    delete $C;
  end loop;

  -- Inline REST CALL — NOT the REST Client (see notes)
  $RawJson = rest call post 'https://graphstudio.mendixdemo.com/sparql/graphmart/http%3A%2F%2Fcambridgesemantics.com%2FGraphmart%2F3617250aca6a40d88972c1c0de38f86a'
    header 'Accept'       = 'application/sparql-results+json'
    header 'Content-Type' = 'application/sparql-query'
    auth basic '<username>' password '<password>'
    body 'PREFIX model: <http://cambridgesemantics.com/SourceLayer/c4ce0eca2e7241f2aee13b46fbdca3f8/Model#> SELECT ?customer ?customerId ?customerName FROM <http://cambridgesemantics.com/SourceLayer/c4ce0eca2e7241f2aee13b46fbdca3f8/Model> WHERE {1} ?customer a model:ExamplePlmBom.Customer; model:ExamplePlmBom.Customer.id ?customerId; model:ExamplePlmBom.Customer.name ?customerName; {2}'
    with ({1} = '{', {2} = '}')
    timeout 60
    returns string
    on error continue;

  log info node 'MyModule' '{1}' with ({1} = 'HTTP status: ' + toString($latestHttpResponse/StatusCode));

  if $latestHttpResponse/StatusCode = 200 then
    $SimplifiedJson = transform $RawJson with MyModule.SimplifyCustomers;
    $ImportResult   = import from mapping MyModule.IMM_Customers($SimplifiedJson);
    log info node 'MyModule' '=== Done ===';
  end if;

  return true;
end;
/
```

### 6. Page

```sql
create page MyModule.Customer_Overview (
  title:  'Customers (from Graph Mart)',
  layout: Atlas_Core.Atlas_Default
) {
  dynamictext heading (content: 'Customers', rendermode: H2)
  actionbutton btnRefresh (caption: 'Refresh', action: microflow MyModule.ACT_RefreshCustomers, buttonstyle: primary)
  datagrid gridCustomers (datasource: database MyModule.Customer sort by CustomerId asc) {
    column colId   (attribute: CustomerId,   caption: 'ID')
    column colName (attribute: CustomerName, caption: 'Name')
    column colUri  (attribute: CustomerUri,  caption: 'URI')
  }
}
/
```

</step_by_step_template>

<gotchas>

### `!` in Basic Auth password → 401

REST Client `authentication: basic (...)` with a literal password containing `!` sends no auth header at runtime. Workaround: use inline `rest call ... auth basic '<user>' password '<pass>'`.

### SPARQL `{` braces in `body` templates are consumed as placeholder escapes

In `rest call ... body '...'`, a literal `{` must be passed as a placeholder value:

```sql
body '... WHERE {1} ... {2}'
with ({1} = '{', {2} = '}')
```

### JSON structure auto-detects ISO strings as DateTime

If your JSLT emits ISO 8601 timestamps and the target Mendix attribute is `string`, `create json structure` infers `datetime` and mxbuild fails with CE5015.

**Solutions:**
- Use a non-ISO sample value in the snippet (e.g. `"2026-04-13 14:00 CET"`).
- Slice/format the timestamp in JSLT so it doesn't look like ISO 8601.
- Or change the target attribute to `datetime`.

### Non-persistent child lists can't be extracted in microflows

- `return $Root/MyModule.CustomerImport_Customer` → "Error(s) in expression" at build
- `loop $c in $Root/MyModule.CustomerImport_Customer` → "The 'Iterate over' property is required"

**Solution:** Make the target entity **persistent**. Use `datasource: database MyModule.Customer` for the grid. A full replace on each refresh (delete-all-then-import) keeps data consistent.

### Rapid drop/create cycles on the same entity can corrupt the MPR

If you `drop entity X` then `create entity X` repeatedly while associations referencing `X` exist, the associations may hold the old entity GUID → mxbuild fails with `KeyNotFoundException`. Fix by dropping/recreating the broken association after the entity change.

</gotchas>

<exploring_the_graph>

```sparql
-- List all classes with counts
select distinct ?class (count(?s) as ?count)
from <http://.../model>
where { ?s a ?class }
GROUP by ?class
ORDER by desc(?count)

-- List properties used by a given class
PREFIX model: <http://.../model#>
select distinct ?property
from <http://.../model>
where {
  ?s a model:ExamplePlmBom.Customer ;
     ?property ?o .
}
ORDER by ?property
```

</exploring_the_graph>

<credential_management>

For demos, literal credentials inline in the microflow are the simplest and most reliable. For anything else, put them in a project constant and reference it from the microflow via `$ConstantName`.

**Do not** use `$ConstantName` in `create rest client ... authentication: basic (username: $C, password: $C)` — the MDL parser rejects the `$` prefix there.

</credential_management>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

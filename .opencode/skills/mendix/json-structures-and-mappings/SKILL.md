---
name: 'json-structures-and-mappings'
description: 'JSON Structures, Import Mappings & Export Mappings — domain model differences for import vs export, nested arrays, find/create handling, null values, and PE→NPE→JSON workflow'
compatibility: opencode
---

<role>MDL JSON mapping author — create JSON structures, import mappings, and export mappings with correct domain model setup.</role>

<summary>
> Covers creating and managing JSON structures, import mappings, and export mappings. Highlights the critical structural difference between import and export domain models, especially for arrays and nested objects.
</summary>

<triggers>
Load when:
- Creating JSON structures or import/export mappings
- Mapping REST API responses to Mendix entities
- Exporting entity data to JSON
- Debugging mapping errors (missing fields, wrong associations)
</triggers>

<key_concepts>

### Critical: Import and Export Need Different Domain Models

- **Import**: The child entity owns the FK to the parent (`from Child to Parent`). Arrays map directly to the item entity — no intermediate container entity needed.
- **Export**: The domain model mirrors the JSON structure. Arrays need an intermediate container entity (e.g., `Items`) plus an item entity (e.g., `ItemsItem`). The container links to the parent, the item links to the container.

</key_concepts>

<json_structures>

```sql
create json structure Module.JSON_Pet
  snippet '{"id": 1, "name": "Fido", "status": "available"}';

-- Multi-line JSON
create json structure Module.JSON_Order
  snippet $${
  "orderId": 100,
  "customer": {"name": "Alice", "email": "alice@example.com"},
  "items": [{"sku": "A1", "quantity": 2, "price": 9.99}]
}$$;

-- Custom name mapping (rename JSON fields)
create json structure Module.JSON_Pet
  snippet '{"id": 1, "name": "Fido"}'
  CUSTOM NAME map ('id' as '_id');

-- Browse
show json structures;
describe json structure Module.JSON_Pet;
drop json structure Module.JSON_Pet;
```

</json_structures>

<import_mappings>

### Domain Model for Import

Associations point FROM the child entity TO the parent:

```sql
create non-persistent entity Module.OrderResponse (OrderId: integer);
/
create non-persistent entity Module.CustomerInfo (Name: string, Email: string);
/
create non-persistent entity Module.OrderItem (Sku: string, Quantity: integer, Price: decimal);
/

-- Child entity owns the FK (FROM child TO parent)
create association Module.CustomerInfo_OrderResponse
  from Module.CustomerInfo to Module.OrderResponse;
/
create association Module.OrderItem_OrderResponse
  from Module.OrderItem to Module.OrderResponse;
/
```

### Simple Import Mapping (flat JSON)

```sql
create import mapping Module.IMM_Pet
  with json structure Module.JSON_Pet
{
  create Module.PetResponse {
    PetId = id,
    Name = name,
    status = status
  }
};
```

### Nested Import Mapping (objects and arrays)

Arrays map directly to the item entity — no intermediate container needed:

```sql
create import mapping Module.IMM_Order
  with json structure Module.JSON_Order
{
  create Module.OrderResponse {
    OrderId = orderId,
    create Module.CustomerInfo_OrderResponse/Module.CustomerInfo = customer {
      Name = name,
      Email = email
    },
    create Module.OrderItem_OrderResponse/Module.OrderItem = items {
      Sku = sku,
      Quantity = quantity,
      Price = price
    }
  }
};
```

### Object Handling

| Syntax | Meaning |
|--------|---------|
| `create Module.Entity` | Always create a new object (default) |
| `find Module.Entity` | Find by KEY attributes, ignore if not found |
| `find or create Module.Entity` | Find by KEY, create if not found |

```sql
create import mapping Module.IMM_UpsertPet
  with json structure Module.JSON_Pet
{
  find or create Module.PetResponse {
    PetId = id key,
    Name = name,
    status = status
  }
};
```

**Note**: `key` is only valid with `find` or `find or create`, not with `create`.

</import_mappings>

<export_mappings>

### Domain Model for Export

Export mappings require entities that **mirror the JSON structure**. Arrays need an intermediate container entity:

```sql
-- Root entity
create non-persistent entity Module.ExRoot (OrderId: integer);
/

-- Nested object entity (1-1 relationship, use OWNER Both)
create non-persistent entity Module.ExCustomer (Name: string, Email: string);
/

-- Array CONTAINER entity (no attributes, just links parent to items)
create non-persistent entity Module.ExItems;
/

-- Array ITEM entity
create non-persistent entity Module.ExItemsItem (Sku: string, Quantity: integer, Price: decimal);
/

create association Module.ExCustomer_ExRoot
  from Module.ExCustomer to Module.ExRoot
  owner both;   -- 1-1 for nested objects
/

create association Module.ExItems_ExRoot
  from Module.ExItems to Module.ExRoot;   -- 1-* for arrays
/

create association Module.ExItemsItem_ExItems
  from Module.ExItemsItem to Module.ExItems;   -- 1-* for array items
/
```

### Simple Export Mapping (flat JSON)

```sql
create export mapping Module.EMM_Pet
  with json structure Module.JSON_Pet
{
  Module.PetResponse {
    id = PetId,
    name = Name,
    status = status
  }
};
```

### Nested Export Mapping (objects and arrays)

Arrays have TWO levels: container entity + item entity:

```sql
create export mapping Module.EMM_Order
  with json structure Module.JSON_Order
{
  Module.ExRoot {
    orderId = OrderId,
    Module.ExCustomer_ExRoot/Module.ExCustomer as customer {
      name = Name,
      email = Email
    },
    Module.ExItems_ExRoot/Module.ExItems as items {
      Module.ExItemsItem_ExItems/Module.ExItemsItem as ItemsItem {
        sku = Sku,
        quantity = Quantity,
        price = Price
      }
    }
  }
};
```

### NULL VALUES option

```sql
create export mapping Module.EMM_Pet
  with json structure Module.JSON_Pet
  null values SendAsNil     -- or LeaveOutElement (default)
{ ... };
```

</export_mappings>

<microflow_actions>

```sql
-- Import from mapping (JSON → entities)
$PetResponse = import from mapping Module.IMM_Pet($JsonContent);

-- Without result variable (persistent entities, stores to DB)
import from mapping Module.IMM_Pet($JsonContent);

-- Export to mapping (entity → JSON)
$JsonOutput = export to mapping Module.EMM_Pet($PetResponse);
```

</microflow_actions>

<pe_to_npe_workflow>

When the source data is in persistent entities (PE) in the database, the typical workflow is:

1. **Retrieve** persistent data from the database
2. **Build NPE tree** in a microflow: create NPE objects, set attributes, link via associations to match the JSON structure
3. **Export to mapping** to serialize the NPE tree to JSON

**Shortcut with View Entities**: OQL-backed view entities can retrieve data directly into the export-ready structure, reducing the microflow to a single retrieve + export step.

</pe_to_npe_workflow>

<browse>

```sql
show import mappings [in module];
show export mappings [in module];
describe import mapping Module.Name;
describe export mapping Module.Name;
drop import mapping Module.Name;
drop export mapping Module.Name;
```

</browse>

<common_mistakes>

| Mistake | Fix |
|---------|-----|
| Reusing import domain model for export | Export needs separate entities mirroring JSON structure |
| Association direction wrong | Always FROM child TO parent (child owns FK) |
| Using `owner default` for 1-1 nested objects in export | Use `owner both` for 1-1 relationships |
| Missing array container entity in export | Arrays need Container + Item entities |
| Using `key` with `create` handling | `key` only valid with `find` or `find or create` |
| Arrays in import with container entity | Import arrays map directly to item entity, no container |

</common_mistakes>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

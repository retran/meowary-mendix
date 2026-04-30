---
name: 'generate-domain-model'
description: 'Creating Mendix Domain Model MDL Scripts — entities, associations, enumerations, view entities, indexes, generalization, event handlers, ALTER ENTITY, positioning, and documentation'
compatibility: opencode
---

<role>MDL domain model author — generate well-documented, correctly positioned entity structures with associations, enumerations, and view entities.</role>

<summary>
> Full reference for generating Mendix domain model scripts. Covers modules, enumerations, persistent/non-persistent/view entities, indexes, generalization, auditing attributes, event handlers, associations, calculated attributes, ALTER ENTITY, and positioning guidelines.
</summary>

<triggers>
Load when:
- User asks to create a domain model for a specific use case
- User wants to generate entities, associations, and enumerations
- User requests a complete e-commerce, HR, CRM, or other business domain model
- Needing ALTER ENTITY for incremental modifications
</triggers>

<critical_rules>

1. **All CREATE statements MUST have JavaDoc-style documentation** (`/** ... */`)
2. **All entities MUST have `@position(x, y)` annotation**
3. **EXTENDS goes BEFORE the opening parenthesis** — `create persistent entity Module.Photo extends System.Image (...)`
4. **INDEX syntax goes AFTER the closing parenthesis, with NO comma before**
5. **Non-persistent entities cannot have validation rules** (`not null error`, `unique error`) — only `default` values
6. **Large files (300+ lines) MUST use `-- MARK: Section Name` comments** (at least 3 for 300+ lines)
7. **Always quote all identifiers** with double quotes to avoid reserved keyword conflicts

</critical_rules>

<module_creation>

```sql
/**
 * Module for financial transaction management
 *
 * @since 1.0.0
 */
create module Finance;
```

</module_creation>

<mark_comments>

Use `-- MARK: Section Name` comments for navigation in large files:

```sql
-- MARK: ENUMERATIONS
-- MARK: CORE ENTITIES
-- MARK: - Core Entities (Persistent)
-- MARK: ASSOCIATIONS
-- MARK: VIEW ENTITIES
-- MARK: MICROFLOWS
```

</mark_comments>

<enumerations>

```sql
/**
 * Transaction type classification
 *
 * @since 1.0.0
 */
create enumeration Module.TransactionType (
  INCOME 'Income',
  EXPENSE 'Expense'
);
```

</enumerations>

<entities>

### Persistent Entity

```sql
/**
 * Entity description
 *
 * @since 1.0.0
 * @see Module.RelatedEntity
 */
@position(100, 100)
create persistent entity Module.EntityName (
  /** Unique identifier */
  Id: long not null error 'ID is required' unique error 'ID must be unique',
  /** Attribute description */
  attributename: string(200) not null error 'Attribute name is required',
  Amount: decimal,
  CreationDate: date,
  IsActive: boolean not null error 'IsActive flag is required' default true,
  status: enumeration(Module.StatusEnum) not null error 'Status is required'
);
```

### Entity Indexes

**INDEX syntax goes AFTER closing parenthesis, NO comma before first INDEX:**

```sql
create persistent entity Module.Transaction (
  TransactionDate: datetime not null,
  status: enumeration(Module.Status) not null,
  Amount: decimal not null,
  IsRecurring: boolean default false
)
index (TransactionDate desc)
index (status, TransactionDate)
index (IsRecurring);
```

### Entity Generalization (EXTENDS)

```sql
-- CORRECT: EXTENDS before (
create persistent entity Module.ProductPhoto extends System.Image (
  PhotoCaption: string(200),
  SortOrder: integer default 0
);

create persistent entity Module.Attachment extends System.FileDocument (
  AttachmentDescription: string(500)
);

-- WRONG: parse error
create persistent entity Module.Photo (PhotoCaption: string(200)) extends System.Image;
```

### System Attributes (Auditing)

```sql
create persistent entity Sales.Order (
  OrderNumber: autonumber,
  TotalAmount: decimal not null,
  status: enumeration(Sales.OrderStatus) not null,
  owner: autoowner,
  ChangedBy: autochangedby,
  CreatedDate: autocreateddate,
  ChangedDate: autochangeddate
);
```

| Pseudo-Type | System Attribute | Set When |
|-------------|-----------------|----------|
| `autoowner` | `System.owner` (→ System.User) | Object created |
| `autochangedby` | `System.changedBy` (→ System.User) | Every commit |
| `autocreateddate` | `CreatedDate` (DateTime) | Object created |
| `autochangeddate` | `ChangedDate` (DateTime) | Every commit |

### Non-Persistent Entity

**Cannot have validation rules** (`not null error`, `unique error`):

```sql
@position(200, 100)
create non-persistent entity Module.TemporaryData (
  SessionId: string(100),
  data: string(1000),
  IsActive: boolean default false
);
```

### View Entity (with OQL)

```sql
@position(300, 500)
create view entity Module.ViewName (
  Attribute1: type,
  Attribute2: type
) as (
  select
    e.Id as Id,
    e.Name as Name,
    e.Amount as Amount
  from Module.Entity as e
  where e.IsActive = true
);
```

**Enumeration Comparisons in OQL**: Use the enumeration **value** (identifier), not the caption:
```sql
where e.Status != 'CANCELLED'   -- Correct: uses enum value
where e.Status != 'Cancelled'   -- Wrong: this is the caption
```

### Entity Event Handlers

```sql
create persistent entity Sales.Order (
  Total: decimal,
  status: string(50)
)
on before commit call Sales.ACT_ValidateOrder raise error
on after create call Sales.ACT_InitDefaults;

-- Add via ALTER ENTITY
alter entity Sales.Order
  add event handler on before delete call Sales.ACT_CheckCanDelete raise error;

-- Drop via ALTER ENTITY
alter entity Sales.Order
  drop event handler on before commit;
```

### Calculated Attributes

Only supported on **persistent entities**:

```sql
@position(100, 100)
create persistent entity Module.OrderLine (
  UnitPrice: decimal not null,
  Quantity: integer not null,
  TotalPrice: decimal calculated by Module.CalcTotalPrice
);
```

</entities>

<data_types>

| Type | Example | Description |
|------|---------|-------------|
| `string(length)` | `string(200)` | Text field with max length |
| `integer` | `integer` | 32-bit integer |
| `long` | `long` | 64-bit integer (use for IDs) |
| `decimal` | `decimal` | Decimal number |
| `boolean` | `boolean` | True/false (auto-defaults to `false`) |
| `datetime` | `datetime` | Date and time |
| `date` | `date` | Date only |
| `binary` | `binary` | Binary data |
| `autonumber` | `autonumber default 1` | Auto-incrementing number |
| `enumeration(Module.Enum)` | `enumeration(Shop.Status)` | Enumeration reference |

</data_types>

<associations>

**CRITICAL: Association Directionality**

Associations are defined **FROM the entity that contains the foreign key TO the entity that is referenced**.

```sql
-- ✅ CORRECT: Transaction stores the account reference (foreign key)
create association Finance.Transaction_Account
from Finance.Transaction to Finance.Account
type reference;

-- ✅ One-to-Many: each order knows its customer
create association Sales.Order_Customer
from Sales.Order to Sales.Customer
type reference;

-- ✅ Many-to-Many
create association Sales.Order_Products
from Sales.Order to Sales.Product
type ReferenceSet
owner both;

-- ✅ Self-reference MUST use owner default (not owner both)
create association Module.Category_ParentCategory
from Module.Category to Module.Category
type reference
owner default;
```

**Full Syntax:**

```sql
/**
 * Association description
 *
 * @since 1.0.0
 */
create association Module.EntityWithFK_ReferencedEntity
from Module.EntityWithFK to Module.ReferencedEntity
type reference
owner default
delete_behavior DELETE_BUT_KEEP_REFERENCES
comment 'Additional documentation';
```

**Delete Behaviors:** `DELETE_AND_REFERENCES`, `DELETE_BUT_KEEP_REFERENCES`, `DELETE_IF_NO_REFERENCES`, `cascade`, `prevent`

**Naming Convention:** `{FromEntity}_{ToEntity}` (e.g., `Order_Customer`, `Transaction_Account`)

</associations>

<alter_entity>

```sql
-- Add attribute
alter entity Module.Customer
  add attribute PhoneNumber: string(20);

-- Add multiple attributes
alter entity Module.Order
  add attribute VATRate: decimal
  add attribute VATAmount: decimal;

-- Rename attribute (preserves data)
alter entity Module.Order
  rename attribute CreatedDate to OrderDate;

-- Drop attribute
alter entity Module.Product
  drop attribute LegacyCode;

-- Modify attribute type
alter entity Module.Customer
  modify attribute Address: string(500);

-- Add index
alter entity Module.Customer
  add index idx_email (Email asc);

-- Reposition entity on domain model canvas
alter entity Module.Customer
  set position (100, 200);
```

**Supported operations:** ADD ATTRIBUTE, RENAME ATTRIBUTE, MODIFY ATTRIBUTE, DROP ATTRIBUTE, SET DOCUMENTATION, SET COMMENT, ADD INDEX, DROP INDEX, SET POSITION.

</alter_entity>

<entity_positioning>

**Layout rules for readable domain models:**

- **Horizontal spacing:** 350px between columns (x = 50, 400, 750, 1100, ...)
- **Vertical spacing:** calculate per-column: `y = previous_y + 50 + (previous_entity_attribute_count * 20)`
- Entity header is ~40px, each attribute adds ~20px of height, plus ~50px padding
- Place related entities in the same column or adjacent columns so associations are short

```sql
@position(50, 50)      -- Top-left: Core entity
create persistent entity Module.Customer (...);

@position(400, 50)     -- Same row: Related entity
create persistent entity Module.Address (...);

@position(50, 250)     -- Below: Dependent entity
create persistent entity Module.Order (...);
```

</entity_positioning>

<script_structure>

```sql
-- ============================================================================
-- Domain Model Name
-- ============================================================================

-- MARK: ENUMERATIONS

create enumeration Module.Enum1 (...);

-- MARK: CORE ENTITIES

-- MARK: - Entity Group 1

create persistent entity Module.Entity1 (...);

-- MARK: VIEW ENTITIES

create view entity Module.View1 as ...;

-- MARK: ASSOCIATIONS

create association Module.Assoc1 ...;
```

</script_structure>

<checklist>

- [ ] All entities have JavaDoc documentation
- [ ] All attributes have inline comments
- [ ] All associations have descriptions
- [ ] Position annotations on all entities
- [ ] MARK comments for files 300+ lines (at least 3 sections)
- [ ] All identifiers quoted with double quotes
- [ ] No duplicate names
- [ ] Valid OQL queries in view entities
- [ ] Required fields marked with NOT NULL (persistent entities only)
- [ ] Validation error messages added for NOT NULL and UNIQUE constraints
- [ ] IDs marked with NOT NULL UNIQUE
- [ ] Self-referencing associations use `owner default`
- [ ] EXTENDS before `(` on generalization

</checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

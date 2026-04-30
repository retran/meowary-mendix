---
name: 'migrate-oracle-forms'
description: 'Migrate Oracle Forms applications to Mendix: PL/SQL to microflows, Forms blocks to DataViews, trigger mapping, and validation patterns'
compatibility: opencode
---

<role>
Migration expert for Oracle Forms to Mendix. Covers PL/SQL mapping, UI component translation, trigger mapping, and validation patterns.
</role>

<summary>
Comprehensive guidance for migrating Oracle Forms (.fmb) applications to Mendix using MDL. Covers data model mapping, PL/SQL to microflow translation, UI component mapping, and script organization.
</summary>

<triggers>
- Converting Oracle Forms (.fmb) applications to Mendix
- Translating PL/SQL logic to Mendix microflows
- Mapping Oracle Forms UI elements to Mendix widgets
- Planning a migration strategy for legacy Oracle Forms systems
</triggers>

<migration_overview>
Oracle Forms migration to Mendix involves:
1. **Data Model**: Oracle tables → Mendix entities
2. **Business Logic**: PL/SQL triggers/procedures → Mendix microflows
3. **User Interface**: Forms blocks/items → Mendix pages/widgets
4. **Navigation**: Form canvases → Mendix page navigation
</migration_overview>

<reserved_word_conflicts>
Most common words (`check`, `text`, `format`, `value`, `type`, `index`, `status`, `select`, etc.) now work **unquoted** as attribute names in MDL. Only structural keywords (`create`, `delete`, `begin`, `end`, `return`, `entity`, `module`) need quoting.

| Oracle Forms Field | Recommended Mendix Name | Notes |
|-------------------|-------------------------|-------|
| `check` | `check` or `CheckStatus` | Works unquoted |
| `text` | `text` or `TextContent` | Works unquoted |
| `format` | `format` or `FormatType` | Works unquoted |
| `value` | `value` or `FieldValue` | Works unquoted |
| `Name` | `Name` or `ItemName` | Works unquoted (not a keyword) |
| `type` | `type` or `ItemType` | Works unquoted |
| `create` | `"create"` or `CreatedBy` | **Requires quoting** (structural keyword) |
| `delete` | `"delete"` or `IsDeleted` | **Requires quoting** (structural keyword) |

```mdl
create persistent entity MyModule.FormField (
  check: boolean default false,  -- Works unquoted
  text: string(500),             -- Works unquoted
  format: string(50),            -- Works unquoted
  CheckFlag: boolean default false  -- Renamed alternative (also fine)
);
```
</reserved_word_conflicts>

<script_organization>
MDL scripts execute statements sequentially. Items created in one statement can be referenced in subsequent statements within the **same script execution**.

### Recommended Script Structure

```mdl
-- ============================================
-- PHASE 1: Domain Model (Entities & Associations)
-- ============================================

create persistent entity MyModule.Customer (
  CustomerCode: string(50),
  CustomerName: string(200),
  Email: string(200),
  IsActive: boolean default true
);

create persistent entity MyModule.Order (
  OrderNumber: string(50),
  OrderDate: datetime,
  TotalAmount: decimal
);

create association MyModule.Order_Customer (
  MyModule.Order [*] -> MyModule.Customer [1]
);
/

-- ============================================
-- PHASE 2: Microflows (Business Logic)
-- ============================================

/**
 * Validates and saves a customer record
 * Replaces Oracle Forms POST-INSERT/POST-UPDATE triggers
 */
create microflow MyModule.ACT_Customer_Save ($Customer: MyModule.Customer)
returns boolean as $success
begin
  declare $success boolean = false;

  -- Validation (replaces WHEN-VALIDATE-ITEM)
  if $Customer/CustomerCode = empty then
    validation feedback $Customer/CustomerCode message 'Customer code is required';
    return false;
  end if;

  commit $Customer with events;
  set $success = true;
  return $success;
end;
/

-- ============================================
-- PHASE 3: Pages (User Interface)
-- ============================================

-- Pages can reference microflows created above
```
</script_organization>

<validation_feedback>
**CRITICAL**: VALIDATION FEEDBACK requires an attribute path, not just a message.

**Wrong:**
```mdl
validation feedback 'Customer code is required';  -- Missing attribute!
```

**Correct:**
```mdl
-- Syntax: VALIDATION FEEDBACK $entity/attribute MESSAGE 'message'
validation feedback $Customer/CustomerCode message 'Customer code is required';
validation feedback $Order/OrderDate message 'Order date cannot be in the future';
```

### Mapping Oracle Forms Validation

| Oracle Forms | Mendix MDL |
|--------------|------------|
| `when-VALIDATE-item` trigger | `if ... validation feedback` in microflow |
| `raise FORM_TRIGGER_FAILURE` | `validation feedback` + `return false` |
| `message('error text')` | `validation feedback $entity/attribute message 'error text'` |

### Complete Validation Pattern

```mdl
/**
 * Validates order before save
 * Replaces Oracle Forms WHEN-VALIDATE-RECORD trigger
 */
create microflow MyModule.ACT_Order_Validate ($Order: MyModule.Order)
returns boolean as $IsValid
begin
  declare $IsValid boolean = true;

  if $Order/OrderNumber = empty then
    validation feedback $Order/OrderNumber message 'Order number is required';
    set $IsValid = false;
  end if;

  if $Order/OrderDate > [%CurrentDateTime%] then
    validation feedback $Order/OrderDate message 'Order date cannot be in the future';
    set $IsValid = false;
  end if;

  if $Order/TotalAmount < 0 then
    validation feedback $Order/TotalAmount message 'Total amount cannot be negative';
    set $IsValid = false;
  end if;

  return $IsValid;
end;
/
```
</validation_feedback>

<plsql_to_microflow_mapping>
### Data Manipulation

| Oracle PL/SQL | Mendix MDL |
|---------------|------------|
| `insert into table ...` | `$var = create Module.Entity (...)` |
| `update table set ...` | `change $var (...)` + `commit $var` |
| `delete from table ...` | `delete $var` |
| `select ... into ...` | `retrieve $var from Module.Entity where ...` |
| `commit` | `commit $var` |
| `rollback` | Built-in with error handlers |

### Control Flow

| Oracle PL/SQL | Mendix MDL |
|---------------|------------|
| `if ... then ... elsif ... else ... end if` | `if ... then ... else ... end if` |
| `for ... loop ... end loop` | `loop $item in $list begin ... end loop` |
| `while ... loop ... end loop` | Not directly supported; use recursive microflow |
| `CURSOR` | `retrieve $list from ...` then `loop` |
| `EXCEPTION when ... then` | `on error { ... }` |

### Example: PL/SQL to MDL

**Oracle PL/SQL:**
```sql
declare
  v_count NUMBER := 0;
  v_total NUMBER := 0;
begin
  for rec in (select * from orders where status = 'PENDING') loop
    v_count := v_count + 1;
    v_total := v_total + rec.amount;
    update orders set status = 'PROCESSED' where id = rec.id;
  end loop;
  commit;
EXCEPTION
  when OTHERS then
    rollback;
    raise;
end;
```

**Mendix MDL:**
```mdl
create microflow MyModule.ACT_ProcessPendingOrders ()
returns string as $Result
begin
  declare $count integer = 0;
  declare $Total decimal = 0;
  declare $Result string = '';

  retrieve $OrderList from MyModule.Order
    where status = 'PENDING';

  loop $Order in $OrderList
  begin
    set $count = $count + 1;
    set $Total = $Total + $Order/Amount;

    change $Order (status = 'PROCESSED');
    commit $Order on error {
      log error 'Failed to process order: ' + $Order/OrderNumber;
    };
  end loop;

  log info 'Processed ' + toString($count) + ' orders, total: ' + toString($Total);
  set $Result = 'Processed ' + toString($count) + ' orders';
  return $Result;
end;
/
```
</plsql_to_microflow_mapping>

<ui_component_mapping>
### Oracle Forms Items to Mendix Widgets

| Oracle Forms Item | Mendix Widget | MDL Syntax |
|-------------------|---------------|------------|
| Text Item | Text Input | `INPUT 'name' attribute attr` |
| Display Item | Text | `text 'content'` |
| Check Box | Check Box | `checkbox 'name' attribute attr` |
| Radio Group | Radio Buttons | `RADIO 'name' attribute attr` |
| List Item (LOV) | Drop-down | `dropdown 'name' attribute attr` |
| Push Button | Button | `button 'name' on CLICK ...` |
| Tab Canvas | Tab Container | `TAB_CONTAINER (TAB 'name' (...))` |

### Oracle Forms Blocks to Mendix DataViews

```mdl
-- Single-record block
dataview source $Customer (
  INPUT 'Code' attribute CustomerCode,
  INPUT 'Name' attribute CustomerName
)

-- Multi-record block (tabular)
datagrid source $OrderList (
  column 'OrderNumber' attribute OrderNumber,
  column 'OrderDate' attribute OrderDate,
  column 'Amount' attribute TotalAmount
)
```
</ui_component_mapping>

<trigger_mapping>
| Oracle Forms Trigger | Mendix Implementation |
|---------------------|----------------------|
| `when-NEW-FORM-INSTANCE` | Page load microflow (data source) |
| `when-NEW-RECORD-INSTANCE` | OnChange microflow on data source |
| `when-VALIDATE-item` | OnChange microflow or validation in save |
| `when-VALIDATE-RECORD` | Validation microflow before save |
| `post-query` | Microflow data source with transformation |
| `PRE-insert` / `PRE-update` | Before commit event handler |
| `post-insert` / `post-update` | After commit event handler |
| `key-commit` | Save button action microflow |
| `on-error` | `on error { ... }` blocks |
</trigger_mapping>

<migration_checklist>
Before starting migration:
- [ ] Export Oracle Forms XML (.xml) or use Forms2XML utility
- [ ] Document all triggers and their purposes
- [ ] Map database tables to Mendix entities
- [ ] Identify LOVs and map to enumerations
- [ ] Check for structural keyword conflicts (Create, Delete, Begin, End, Return)

During migration:
- [ ] Create entities first (Phase 1)
- [ ] Create microflows second (Phase 2)
- [ ] Create pages last (Phase 3) - they can reference microflows
- [ ] Test validation patterns thoroughly
- [ ] Use `validation feedback $entity/attribute message 'message'` for all validations

After migration:
- [ ] Run `mxcli check script.mdl -p app.mpr --references`
- [ ] Open in Mendix Studio Pro to verify
- [ ] Test all validation scenarios
- [ ] Verify master-detail relationships work correctly
</migration_checklist>

<common_migration_errors>
| Error | Cause | Fix |
|-------|-------|-----|
| "Parse error: mismatched input 'Create'" | Structural keyword as attribute | Use `"create"` (quoted) or rename |
| "microflow not found" | Referenced before created | Move microflow definition before page |
| "page not found" | Referenced before created | Move page definition earlier |
| "VALIDATION FEEDBACK requires attribute" | Missing attribute path | Use `validation feedback $entity/attribute message 'msg'` |
| CE0117 "Error in expression" | Missing module prefix | Use fully qualified names |
</common_migration_errors>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

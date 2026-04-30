---
name: 'patterns-crud'
description: 'Standard CRUD microflow patterns for Mendix: Save, Validate, Delete, Cancel, New, Edit, and DataSource microflows with naming conventions'
compatibility: opencode
---

<role>
CRUD microflow pattern expert for Mendix. Covers Save, Validate, Delete, Cancel, New, Edit, and DataSource patterns with naming conventions and best practices.
</role>

<summary>
Standard patterns for Create, Read, Update, Delete operations on entities.
</summary>

<triggers>
- Write CRUD action microflows for an entity
- Create validation microflows
- Implement save, delete, cancel, or new patterns
- Set up data source microflows for grids and lists
</triggers>

<naming_conventions>
| Prefix | Purpose | Example |
|--------|---------|---------|
| `ACT_` | Action microflow (page button) | `ACT_Customer_Save` |
| `VAL_` | Validation microflow | `VAL_Customer_Save` |
| `DS_` | Data source microflow | `DS_Customer_GetAll` |
| `SUB_` | Sub-microflow (internal) | `SUB_Customer_SendEmail` |
</naming_conventions>

<save_pattern>
```mdl
/**
 * Save action for Customer NewEdit page
 * Validates, commits, and closes the page
 *
 * @param $Customer The customer to save
 * @returns true if saved successfully
 */
create microflow Module.ACT_Customer_Save (
  $Customer: Module.Customer
)
returns boolean
begin
  -- Validate first
  declare $IsValid boolean = true;
  $IsValid = call microflow Module.VAL_Customer_Save($Customer = $Customer);

  if $IsValid then
    commit $Customer with events;
    close page;
  end if;

  return $IsValid;
end;
/
```
</save_pattern>

<validation_pattern>
```mdl
/**
 * Validate Customer before save
 *
 * @param $Customer The customer to validate
 * @returns true if valid
 */
create microflow Module.VAL_Customer_Save (
  $Customer: Module.Customer
)
returns boolean
begin
  declare $IsValid boolean = true;

  -- Required field validation
  if $Customer/Name = empty then
    validation feedback $Customer/Name message 'Name is required';
    set $IsValid = false;
  end if;

  if $Customer/Email = empty then
    validation feedback $Customer/Email message 'Email is required';
    set $IsValid = false;
  end if;

  -- Business rule validation
  if $Customer/CreditLimit < 0 then
    validation feedback $Customer/CreditLimit message 'Credit limit cannot be negative';
    set $IsValid = false;
  end if;

  return $IsValid;
end;
/
```
</validation_pattern>

<delete_pattern>
```mdl
/**
 * Delete a customer
 * Called after user confirms deletion
 *
 * @param $Customer The customer to delete
 * @returns true if deleted
 */
create microflow Module.ACT_Customer_Delete (
  $Customer: Module.Customer
)
returns boolean
begin
  delete $Customer;
  close page;
  return true;
end;
/
```
</delete_pattern>

<cancel_pattern>
```mdl
/**
 * Cancel editing and close page
 * Discards uncommitted changes
 *
 * @param $Customer The customer being edited
 * @returns true
 */
create microflow Module.ACT_Customer_Cancel (
  $Customer: Module.Customer
)
returns boolean
begin
  rollback $Customer;
  close page;
  return true;
end;
/
```
</cancel_pattern>

<create_new_pattern>
```mdl
/**
 * Create new customer and open edit page
 *
 * @returns true
 */
create microflow Module.ACT_Customer_New ()
returns boolean
begin
  declare $NewCustomer as Module.Customer;

  $NewCustomer = create Module.Customer (
    IsActive = true,
    CreatedDate = [%CurrentDateTime%]
  );

  show page Module.Customer_NewEdit ($Customer = $NewCustomer);
  return true;
end;
/
```
</create_new_pattern>

<data_source_pattern>
```mdl
/**
 * Get all active customers
 * Used as data source for Customer overview
 *
 * @returns List of active customers
 */
create microflow Module.DS_Customer_GetActive ()
returns list of Module.Customer
begin
  declare $Customers list of Module.Customer = empty;

  retrieve $Customers from Module.Customer
    where IsActive = true;

  return $Customers;
end;
/
```
</data_source_pattern>

<edit_pattern>
```mdl
/**
 * Open customer for editing
 *
 * @param $Customer The customer to edit
 * @returns true
 */
create microflow Module.ACT_Customer_Edit (
  $Customer: Module.Customer
)
returns boolean
begin
  show page Module.Customer_NewEdit ($Customer = $Customer);
  return true;
end;
/
```
</edit_pattern>

<complete_crud_set>
For a typical entity, create these microflows:

| Microflow | Purpose | Parameters |
|-----------|---------|------------|
| `ACT_Entity_New` | Create new | None |
| `ACT_Entity_Edit` | Open for edit | `$entity` |
| `ACT_Entity_Save` | Save changes | `$entity` |
| `VAL_Entity_Save` | Validate | `$entity` |
| `ACT_Entity_Delete` | Delete | `$entity` |
| `ACT_Entity_Cancel` | Cancel edit | `$entity` |
| `DS_Entity_GetAll` | List all | None |
</complete_crud_set>

<best_practices>
1. **Always validate before commit**: Call VAL_ microflow in ACT_Save
2. **Use WITH EVENTS**: `commit $entity with events` triggers event handlers
3. **Close page on success**: Use `close page` after successful save/delete
4. **Rollback on cancel**: Use `rollback $entity` to discard changes
5. **Initialize defaults**: Set default values in ACT_New microflow
6. **Return Boolean**: All action microflows should return success status
</best_practices>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

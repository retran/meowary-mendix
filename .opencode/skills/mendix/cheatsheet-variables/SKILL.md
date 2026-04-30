---
name: 'cheatsheet-variables'
description: 'MDL Variable Cheatsheet — declaration syntax for all types, key rules, common mistakes, special values, parameter vs variable, and scope'
compatibility: opencode
---

<role>MDL variable reference — quick lookup for variable declarations, rules, and scope in microflows.</role>

<summary>
> Quick reference for variable declarations in MDL microflows. Covers all types (string, integer, boolean, decimal, datetime, entity, list), key rules, common mistakes, special values, and variable scope.
</summary>

<triggers>
Load when:
- Writing or debugging MDL microflow variable declarations
- Getting CE0053 "Selected type is not allowed" errors
- Confused about AS keyword, list syntax, or initialization requirements
</triggers>

<declaration_syntax>

| Type | Syntax | Example |
|------|--------|---------|
| String | `declare $name string = 'value';` | `declare $message string = '';` |
| Integer | `declare $name integer = 0;` | `declare $count integer = 0;` |
| Boolean | `declare $name boolean = true;` | `declare $IsValid boolean = true;` |
| Decimal | `declare $name decimal = 0.0;` | `declare $Amount decimal = 0;` |
| DateTime | `declare $name datetime = [%CurrentDateTime%];` | `declare $Now datetime = [%CurrentDateTime%];` |
| Entity | `declare $name as Module.Entity;` | `declare $Customer as Sales.Customer;` |
| List | `declare $name list of Module.Entity = empty;` | `declare $Orders list of Sales.Order = empty;` |

</declaration_syntax>

<key_rules>

1. **Primitives**: Use `declare $var type = value;` (initialization required)
2. **Entities**: Use `declare $var as Module.Entity;` (use AS keyword, no initialization)
3. **Lists**: Use `declare $var list of Module.Entity = empty;`
4. **SET requires DECLARE**: Always declare variables before using SET
5. **Parameters are pre-declared**: Microflow parameters don't need DECLARE

</key_rules>

<common_mistakes>

### Entity Declaration

```mdl
-- WRONG: Missing AS keyword
declare $Product Module.Product = empty;

-- CORRECT: Use AS for entity types
declare $Product as Module.Product;
```

### SET Without DECLARE

```mdl
-- WRONG: Variable not declared
if $value > 10 then
  set $message = 'High';  -- ERROR!
end if;

-- CORRECT: Declare first
declare $message string = '';
if $value > 10 then
  set $message = 'High';
end if;
```

### List Declaration

```mdl
-- WRONG: Missing 'of' keyword
declare $Items list Module.Item = empty;

-- CORRECT: Use 'list of'
declare $Items list of Module.Item = empty;
```

</common_mistakes>

<special_values>

| Value | Usage |
|-------|-------|
| `empty` | Null/empty value for any type |
| `[%CurrentDateTime%]` | Current date and time |
| `[%CurrentUser%]` | Currently logged in user object |
| `true` / `false` | Boolean literals |

</special_values>

<parameter_vs_variable>

```mdl
create microflow Module.Example (
  $Input: string,              -- Parameter: auto-declared
  $entity: Module.Customer     -- Parameter: auto-declared
)
returns boolean
begin
  -- Parameters $Input and $Entity are already available

  declare $Result boolean = true;  -- Local variable: must declare
  declare $Temp as Module.Order;   -- Local entity: must declare

  return $Result;
end;
/
```

</parameter_vs_variable>

<variable_scope>

- Parameters: Available throughout the microflow
- DECLARE variables: Available from declaration point forward
- Loop variables: Only available inside the loop body

```mdl
loop $item in $ItemList
begin
  -- $item is available here (derived from list type)
  set $count = $count + 1;
end loop;
-- $item is NOT available here
```

</variable_scope>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

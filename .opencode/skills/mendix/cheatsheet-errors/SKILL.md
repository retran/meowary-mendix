---
name: 'cheatsheet-errors'
description: 'MDL Common Errors Cheatsheet — quick fixes for variable, expression, control flow, syntax, and reference errors with Studio Pro error code reference'
compatibility: opencode
---

<role>MDL error reference — quick lookup for common MDL syntax and runtime errors with corrected code examples.</role>

<summary>
> Quick fixes for common MDL syntax errors. Covers variable declaration errors, expression errors, control flow errors, syntax errors, reference errors, and a Studio Pro error code table.
</summary>

<triggers>
Load when:
- An MDL script produces a syntax or compile error
- Debugging CE0053, CE0104, CE0105, CE0117, or CW0094 errors
- Looking up the correct syntax for a specific pattern
</triggers>

<variable_errors>

### "Variable 'X' is not declared"

**Problem**: Using SET on a variable that wasn't declared.

```mdl
-- WRONG
if $value > 10 then
  set $IsValid = false;  -- ERROR: $IsValid not declared
end if;
```

**Fix**: Add DECLARE before SET.

```mdl
-- CORRECT
declare $IsValid boolean = true;
if $value > 10 then
  set $IsValid = false;
end if;
```

### "Selected type is not allowed" (CE0053)

**Problem**: Wrong syntax for entity type declaration.

```mdl
-- WRONG: Missing AS keyword
declare $Product Module.Product = empty;
```

**Fix**: Use AS keyword for entity types.

```mdl
-- CORRECT
declare $Product as Module.Product;
```

</variable_errors>

<expression_errors>

### "Error in expression" (CE0117)

**Problem**: Using unqualified association name in XPath.

```mdl
-- WRONG: Missing module qualification
set $Name = $Order/Customer/Name;
```

**Fix**: Use fully qualified association name.

```mdl
-- CORRECT: Module.AssociationName
set $Name = $Order/Shop.Order_Customer/Name;
```

### "Type mismatch" in enum comparison

**Problem**: Comparing enumeration with string literal.

```mdl
-- WRONG: String literal instead of enum value
if $task/status = 'Completed' then
```

**Fix**: Use qualified enumeration value.

```mdl
-- CORRECT: Module.Enumeration.Value
if $task/status = Module.TaskStatus.Completed then
```

</expression_errors>

<control_flow_errors>

### "Activity cannot be the last object" (CE0105)

**Problem**: Missing RETURN statement.

```mdl
-- WRONG: No RETURN
begin
  declare $Result boolean = true;
  log info 'Done';
  -- Missing RETURN!
end;
```

**Fix**: Add RETURN statement.

```mdl
-- CORRECT
begin
  declare $Result boolean = true;
  log info 'Done';
  return $Result;
end;
```

### "Action activity is unreachable" (CE0104)

**Problem**: Code after RETURN statement.

```mdl
-- WRONG: Code after RETURN
if $value < 0 then
  return false;
  log info 'Negative';  -- Unreachable!
end if;
```

**Fix**: Move code before RETURN.

```mdl
-- CORRECT
if $value < 0 then
  log info 'Negative';
  return false;
end if;
```

</control_flow_errors>

<syntax_errors>

### Division operator

```mdl
-- WRONG: Using / for division
set $average = $Total / $count;

-- CORRECT: Use 'div' keyword
set $average = $Total div $count;
```

### Missing END IF / END LOOP

```mdl
-- WRONG: Missing END IF
if $value > 0 then
  set $Positive = true;
-- Missing END IF!

-- CORRECT
if $value > 0 then
  set $Positive = true;
end if;
```

### Missing semicolons

```mdl
-- WRONG: Missing semicolon
declare $count integer = 0
set $count = 1

-- CORRECT
declare $count integer = 0;
set $count = 1;
```

</syntax_errors>

<reference_errors>

### "Module not found"

**Problem**: Using non-existent module name.

**Fix**: Check module exists with `show modules`.

### "Entity not found"

**Problem**: Using non-existent entity name.

**Fix**:
1. Check entity exists: `show entities in ModuleName`
2. Use fully qualified name: `Module.EntityName`

### "Microflow not found"

**Problem**: Calling non-existent microflow.

**Fix**:
1. Check microflow exists: `show microflows in ModuleName`
2. Use fully qualified name: `Module.MicroflowName`

</reference_errors>

<error_code_reference>

| Code | Message | Common Cause |
|------|---------|--------------|
| CE0053 | Selected type is not allowed | Missing AS for entity type |
| CE0104 | Action activity is unreachable | Code after RETURN |
| CE0105 | Must end with end event | Missing RETURN |
| CE0117 | Error in expression | Unqualified association path |
| CW0094 | Variable never used | Unused parameter/variable |

</error_code_reference>

<quick_validation_checklist>

Before executing MDL:

- [ ] All entity types use `declare $var as Module.Entity`
- [ ] All SET targets have prior DECLARE
- [ ] Association paths are qualified: `$var/Module.Assoc/attr`
- [ ] Enum comparisons use `Module.Enum.Value`
- [ ] Every flow path ends with RETURN
- [ ] Division uses `div` not `/`
- [ ] All statements end with `;`
- [ ] IF/LOOP properly closed with END IF/END LOOP

</quick_validation_checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>

---
name: 'fragments'
description: 'Mendix Fragments — DEFINE FRAGMENT, USE FRAGMENT with prefix, SHOW/DESCRIBE FRAGMENTS, common patterns, and validation checklist'
compatibility: opencode
---

<role>MDL fragment author — define and reuse widget groups across multiple pages in the same script.</role>

<summary>
> Fragments are script-scoped, transient widget groups. Defined once and reused in multiple pages/snippets within the same script execution. Not persisted in the MPR file.
</summary>

<triggers>
Load when:
- Defining reusable widget groups with `define fragment`
- Inserting fragments into pages or snippets with `use fragment`
- Building multiple pages that share common widget patterns (footers, form fields, buttons)
- Avoiding copy-paste of repeated widget structures across pages
</triggers>

<overview>

Fragments are **script-scoped, transient** widget groups:
- Defined once, reused in multiple pages/snippets within the same script
- **Not persisted** in the MPR file — they exist only during script execution
- Widgets are deep-cloned on expansion (each USE gets independent copies)
- Optional prefix support to avoid name conflicts when using the same fragment multiple times

</overview>

<syntax_reference>

### DEFINE FRAGMENT

```mdl
define fragment SaveCancelFooter as {
  footer footer1 {
    actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
    actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
  }
};
```

Multiple top-level widgets:

```mdl
define fragment CustomerFields as {
  textbox txtName (label: 'Name', attribute: Name)
  textbox txtEmail (label: 'Email', attribute: Email)
  textbox txtPhone (label: 'Phone', attribute: Phone)
};
```

### USE FRAGMENT

```mdl
create page Module.CustomerEdit
(
  params: { $Customer: Module.Customer },
  title: 'Edit Customer',
  layout: Atlas_Core.PopupLayout
)
{
  dataview dvCustomer (datasource: $Customer) {
    use fragment CustomerFields
    use fragment SaveCancelFooter
  }
};
```

With prefix (avoids name conflicts):

```mdl
use fragment SaveCancelFooter as order_
-- Creates: order_footer1, order_btnSave, order_btnCancel
```

### SHOW / DESCRIBE FRAGMENTS

```mdl
show fragments;
-- Lists all defined fragments with widget counts

describe fragment SaveCancelFooter;
-- Outputs the full MDL definition
```

</syntax_reference>

<common_patterns>

### Pattern 1: Standard CRUD Footer

```mdl
define fragment CrudFooter as {
  footer footer1 {
    actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
    actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
  }
};

create page Module.Customer_Edit (...) {
  dataview dv (datasource: $Customer) {
    textbox txtName (label: 'Name', attribute: Name)
    use fragment CrudFooter
  }
};

create page Module.Order_Edit (...) {
  dataview dv (datasource: $Order) {
    textbox txtNumber (label: 'Order #', attribute: Number)
    use fragment CrudFooter
  }
};
```

### Pattern 2: Form Field Groups

```mdl
define fragment AddressFields as {
  textbox txtStreet (label: 'Street', attribute: Street)
  textbox txtCity (label: 'City', attribute: City)
  textbox txtZip (label: 'Zip Code', attribute: ZipCode)
  textbox txtCountry (label: 'Country', attribute: Country)
};

create page Module.Customer_Edit (...) {
  dataview dv (datasource: $Customer) {
    textbox txtName (label: 'Name', attribute: Name)
    use fragment AddressFields
    use fragment CrudFooter
  }
};
```

### Pattern 3: Same Fragment with Prefix

```mdl
define fragment ActionButtons as {
  actionbutton btnApprove (caption: 'Approve', action: save_changes, buttonstyle: success)
  actionbutton btnReject (caption: 'Reject', action: cancel_changes, buttonstyle: danger)
};

create page Module.DualPanel (...) {
  layoutgrid lg {
    row row1 {
      column col1 (desktopwidth: 6) {
        use fragment ActionButtons as left_
      }
      column col2 (desktopwidth: 6) {
        use fragment ActionButtons as right_
      }
    }
  }
};
```

</common_patterns>

<common_mistakes>

### Duplicate Fragment Names

```mdl
-- WRONG: Defining the same fragment name twice causes an error
define fragment footer as { ... };
define fragment footer as { ... };  -- Error: fragment "Footer" already defined
```

### Missing Fragment

```mdl
-- WRONG: Using a fragment that hasn't been defined
create page Module.MyPage (...) {
  use fragment NonExistent   -- Error: fragment "NonExistent" not found
};
```

### Name Conflicts Without Prefix

```mdl
-- WRONG: Using same fragment twice without prefix creates duplicate widget names
use fragment footer
use fragment footer   -- Widget name "footer1" already exists!

-- CORRECT: Use prefix for uniqueness
use fragment footer as first_
use fragment footer as second_
```

### Fragment Order

```mdl
-- WRONG: Using a fragment before defining it
create page Module.MyPage (...) {
  use fragment footer   -- Error: fragment "Footer" not found
};
define fragment footer as { ... };

-- CORRECT: Define before use
define fragment footer as { ... };
create page Module.MyPage (...) {
  use fragment footer   -- OK
};
```

</common_mistakes>

<validation_checklist>

- [ ] All `define fragment` statements appear before their `use fragment` references
- [ ] No duplicate fragment names in the script
- [ ] Prefix used when the same fragment appears multiple times on one page
- [ ] Fragment widget names don't conflict with other widgets on the page
- [ ] All widgets inside fragments use valid syntax (same as page bodies)

</validation_checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
